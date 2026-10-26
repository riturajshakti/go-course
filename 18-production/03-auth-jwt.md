[← 18.2 Configuration via Environment](02-config.md) · [Course Index](../README.md) · [18.4 Structured Logging in a Service →](04-service-logging.md)

# 18.3 — Authentication with JWT

## What You'll Learn

- Creating and validating a token with `github.com/golang-jwt/jwt/v5`, reproducibly, with a fixed key and fixed times
- The **algorithm confusion** attack, demonstrated for real: a forged `admin=true` token accepted by a keyfunc that forgets to check `alg`
- Every validation failure captured with `errors.Is` — expired, not-yet-valid, bad signature, malformed, wrong audience, wrong issuer, missing `exp`
- Auth middleware returning 401 vs 403 correctly, plus access/refresh rotation and `jti` revocation

## Coming From Another Language

| Other | Go |
|---|---|
| `jsonwebtoken.verify(tok, secret)` | `jwt.ParseWithClaims(tok, &claims, keyfunc, opts...)` — the keyfunc is where you check `alg` |
| `jwt.decode(tok, options={"verify_signature": False})` | `jwt.NewParser(jwt.WithoutClaimsValidation())`, and you almost never want it |
| `req.user` set by Passport | `context.WithValue` + a typed getter (11.7) |
| a framework guard/decorator per route | a `func(http.Handler) http.Handler` (16.6) |
| `exp` checked by the library with no clock control | `jwt.WithTimeFunc` lets you inject a clock, which is why this lesson's output is reproducible |

## What a JWT is

A JWT is three base64url segments joined by dots: a header naming the signing algorithm, a
payload of JSON claims, and a signature over the first two. It is **signed, not encrypted** —
anyone holding the token reads every claim with `base64 -d`, so a payload must never contain a
password, API key or card number. The signature buys *integrity*: the server verifies the claims
it issued have not been altered, without storing them.

```go
package main

import (
  "encoding/base64"
  "fmt"
  "strings"
  "time"

  "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
  Email string   `json:"email"`
  Roles []string `json:"roles"`
  jwt.RegisteredClaims
}

var key = []byte("test-signing-key-at-least-32-bytes!!")

var issued = time.Date(2026, 1, 1, 12, 0, 0, 0, time.UTC)

func main() {
  claims := Claims{
    Email: "ada@example.com",
    Roles: []string{"user", "admin"},
    RegisteredClaims: jwt.RegisteredClaims{
      Issuer:    "orders-api",
      Subject:   "user-42",
      Audience:  jwt.ClaimStrings{"orders-api"},
      ExpiresAt: jwt.NewNumericDate(issued.Add(15 * time.Minute)),
      NotBefore: jwt.NewNumericDate(issued),
      IssuedAt:  jwt.NewNumericDate(issued),
      ID:        "jti-0001",
    },
  }

  tok := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
  signed, err := tok.SignedString(key)
  if err != nil {
    panic(err)
  }
  fmt.Println("token:")
  fmt.Println(signed)
  fmt.Println("length:", len(signed), "bytes")

  parts := strings.Split(signed, ".")
  fmt.Println("dot-separated parts:", len(parts))
  h, _ := base64.RawURLEncoding.DecodeString(parts[0])
  p, _ := base64.RawURLEncoding.DecodeString(parts[1])
  fmt.Printf("\ndecoded header:  %s\n", h)
  fmt.Printf("decoded payload: %s\n", p)
  fmt.Printf("signature:       %s\n", parts[2])
}
```

Because the key, the claims and the timestamps are all fixed, HMAC-SHA256 is deterministic
and this token string is byte-for-byte reproducible:

Output:

```
token:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFkYUBleGFtcGxlLmNvbSIsInJvbGVzIjpbInVzZXIiLCJhZG1pbiJdLCJpc3MiOiJvcmRlcnMtYXBpIiwic3ViIjoidXNlci00MiIsImF1ZCI6WyJvcmRlcnMtYXBpIl0sImV4cCI6MTc2NzI2OTcwMCwibmJmIjoxNzY3MjY4ODAwLCJpYXQiOjE3NjcyNjg4MDAsImp0aSI6Imp0aS0wMDAxIn0.fYA_0gWkaIWKXkub59f6QCFFdHUrIeyXtSZmnONBHTc
length: 316 bytes
dot-separated parts: 3

decoded header:  {"alg":"HS256","typ":"JWT"}
decoded payload: {"email":"ada@example.com","roles":["user","admin"],"iss":"orders-api","sub":"user-42","aud":["orders-api"],"exp":1767269700,"nbf":1767268800,"iat":1767268800,"jti":"jti-0001"}
signature:       fYA_0gWkaIWKXkub59f6QCFFdHUrIeyXtSZmnONBHTc
```

The payload came back as plain JSON from a `base64` decode. That is the property to internalise.

## Registered claims and custom claims

`jwt.RegisteredClaims` is a struct of the RFC 7519 standard claims; **embed** it in your own
type and the JSON tags compose:

| Claim | Field | Meaning |
|---|---|---|
| `iss` | `Issuer` | who minted it; validate it |
| `sub` | `Subject` | the user ID — the one claim you can always rely on |
| `aud` | `Audience` (`ClaimStrings`) | which service(s) may accept it; validate it |
| `exp` | `ExpiresAt` (`*NumericDate`) | expiry; **require** it |
| `nbf` | `NotBefore` | not valid before |
| `iat` | `IssuedAt` | issue time; useful for "force logout before T" |
| `jti` | `ID` | unique token ID — the handle you revoke by |

`aud` is `jwt.ClaimStrings` because the JSON may be a string *or* an array; it unmarshals both.
`*NumericDate` wraps Unix seconds, so use `jwt.NewNumericDate(t)`.

Embed it, do not make it a field: `jwt.RegisteredClaims` has the `GetExpirationTime`/`GetIssuer`
methods satisfying `jwt.Claims`, and promotion ([4.5](../04-types-structs/05-embedding.md)) gives
them to your type for free.

## Signing methods

| Method | Keys | Use when |
|---|---|---|
| `HS256` (HMAC-SHA256) | one shared secret for sign and verify | a single service signs and verifies its own tokens |
| `RS256` (RSA-SHA256) | private key signs, public key verifies | a separate auth service signs; many services verify with the public key |
| `ES256` (ECDSA P-256) | same split as RS256 | same as RS256, with much smaller signatures and keys |
| `EdDSA` (Ed25519) | same split | modern choice when every verifier supports it |
| `none` | — | never |

The decision is one question: does anything other than the signer need to verify? If no,
HS256, because a shared secret is simpler to rotate than a keypair. If yes, go asymmetric so
verifiers can hold only the public key and a compromised verifier cannot mint tokens.

## The algorithm confusion attack

This is the vulnerability the keyfunc exists to prevent. A server verifies RS256 tokens with an
RSA public key, which is **not secret** — it is published at a JWKS endpoint. An attacker takes
that public key, uses it as the *HMAC secret* to sign a token with `alg: HS256`, and sends it. A
keyfunc that returns the key without checking `t.Method` hands the HMAC verifier the public-key
bytes — exactly the secret the attacker used.

```go
package main

import (
  "crypto/rand"
  "crypto/rsa"
  "crypto/x509"
  "encoding/pem"
  "fmt"

  "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
  Admin bool `json:"admin"`
  jwt.RegisteredClaims
}

func main() {
  priv, err := rsa.GenerateKey(rand.Reader, 2048)
  if err != nil {
    panic(err)
  }
  pubDER, err := x509.MarshalPKIXPublicKey(&priv.PublicKey)
  if err != nil {
    panic(err)
  }
  pubPEM := pem.EncodeToMemory(&pem.Block{Type: "PUBLIC KEY", Bytes: pubDER})
  fmt.Printf("server's PUBLIC key is %d bytes of PEM, and is not a secret\n", len(pubPEM))

  legit, err := jwt.NewWithClaims(jwt.SigningMethodRS256, Claims{
    Admin:            false,
    RegisteredClaims: jwt.RegisteredClaims{Subject: "user-42"},
  }).SignedString(priv)
  if err != nil {
    panic(err)
  }

  forged, err := jwt.NewWithClaims(jwt.SigningMethodHS256, Claims{
    Admin:            true,
    RegisteredClaims: jwt.RegisteredClaims{Subject: "user-42"},
  }).SignedString(pubPEM)
  if err != nil {
    panic(err)
  }

  vulnerable := func(t *jwt.Token) (any, error) {
    return pubPEM, nil
  }
  safe := func(t *jwt.Token) (any, error) {
    if _, ok := t.Method.(*jwt.SigningMethodRSA); !ok {
      return nil, fmt.Errorf("unexpected signing method %q", t.Method.Alg())
    }
    return &priv.PublicKey, nil
  }

  run := func(label string, tok string, kf jwt.Keyfunc, opts ...jwt.ParserOption) {
    var c Claims
    _, err := jwt.ParseWithClaims(tok, &c, kf, opts...)
    if err != nil {
      fmt.Printf("%-42s REJECTED %v\n", label, err)
      return
    }
    fmt.Printf("%-42s ACCEPTED sub=%s admin=%t\n", label, c.Subject, c.Admin)
  }

  fmt.Println("\n--- VULNERABLE keyfunc: returns the key without checking alg ---")
  run("legit RS256 token", legit, vulnerable)
  run("FORGED HS256 token (admin=true)", forged, vulnerable)

  fmt.Println("\n--- FIXED: keyfunc asserts the signing method type ---")
  run("legit RS256 token", legit, safe)
  run("FORGED HS256 token (admin=true)", forged, safe)

  fmt.Println("\n--- also fixed by WithValidMethods, even with the bad keyfunc ---")
  run("legit RS256 token", legit, vulnerable, jwt.WithValidMethods([]string{"RS256"}))
  run("FORGED HS256 token (admin=true)", forged, vulnerable, jwt.WithValidMethods([]string{"RS256"}))

  fmt.Println("\n--- alg=none is rejected outright by jwt/v5 ---")
  noneTok, err := jwt.NewWithClaims(jwt.SigningMethodNone, Claims{
    Admin:            true,
    RegisteredClaims: jwt.RegisteredClaims{Subject: "user-42"},
  }).SignedString(jwt.UnsafeAllowNoneSignatureType)
  if err != nil {
    fmt.Println("could not even mint:", err)
  } else {
    fmt.Println("minted:", noneTok)
    run("alg=none token", noneTok, safe)
  }
}
```

The RSA key is generated fresh each run, so the token strings vary; the decisions do not.

Output:

```
server's PUBLIC key is 451 bytes of PEM, and is not a secret

--- VULNERABLE keyfunc: returns the key without checking alg ---
legit RS256 token                          REJECTED token signature is invalid: key is of invalid type: RSA verify expects *rsa.PublicKey
FORGED HS256 token (admin=true)            ACCEPTED sub=user-42 admin=true

--- FIXED: keyfunc asserts the signing method type ---
legit RS256 token                          ACCEPTED sub=user-42 admin=false
FORGED HS256 token (admin=true)            REJECTED token is unverifiable: error while executing keyfunc: unexpected signing method "HS256"

--- also fixed by WithValidMethods, even with the bad keyfunc ---
legit RS256 token                          REJECTED token signature is invalid: key is of invalid type: RSA verify expects *rsa.PublicKey
FORGED HS256 token (admin=true)            REJECTED token signature is invalid: signing method HS256 is invalid

--- alg=none is rejected outright by jwt/v5 ---
minted: eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJhZG1pbiI6dHJ1ZSwic3ViIjoidXNlci00MiJ9.
alg=none token                             REJECTED token is unverifiable: error while executing keyfunc: unexpected signing method "none"
```

Line four is a full privilege escalation: `ACCEPTED sub=user-42 admin=true`, from a token the
attacker signed with a key they downloaded from a public endpoint.

The rule, stated once: **the keyfunc must assert on `t.Method` before returning a key**, because
the `alg` header is attacker-controlled input and the keyfunc is the only place the library asks
you what to do with it. `jwt.WithValidMethods` is a second independent layer — use **both**; they
fail differently and defence in depth costs one line. Note that jwt/v5 requires the explicit
`jwt.UnsafeAllowNoneSignatureType` sentinel even to *mint* an `alg=none` token, and a
method-checking keyfunc rejects it like anything else.

## Validation options and every failure mode

`jwt.ParseWithClaims` takes parser options. These are the ones that matter:

| Option | Effect |
|---|---|
| `WithValidMethods([]string{"HS256"})` | reject any other `alg` before the keyfunc result is used |
| `WithIssuer("orders-api")` | require `iss` to match exactly |
| `WithAudience("orders-api")` | require `aud` to contain this value |
| `WithExpirationRequired()` | a token with no `exp` is invalid (it is optional in the RFC) |
| `WithLeeway(30*time.Second)` | tolerate that much clock skew on `exp`/`nbf`/`iat` |
| `WithIssuedAt()` | also validate `iat` is not in the future |
| `WithTimeFunc(f)` | inject the clock — makes expiry tests deterministic |
| `WithJSONNumber()` | decode numbers as `json.Number` instead of `float64` |

Every failure, with the sentinel it matches:

```go
package main

import (
  "crypto/rand"
  "crypto/rsa"
  "errors"
  "fmt"
  "time"

  "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
  Email string `json:"email"`
  jwt.RegisteredClaims
}

var (
  key      = []byte("test-signing-key-at-least-32-bytes!!")
  otherKey = []byte("a-completely-different-signing-key!!")
  now      = time.Date(2026, 1, 1, 12, 0, 0, 0, time.UTC)
)

func mint(method jwt.SigningMethod, k any, mutate func(*Claims)) string {
  c := Claims{
    Email: "ada@example.com",
    RegisteredClaims: jwt.RegisteredClaims{
      Issuer:    "orders-api",
      Subject:   "user-42",
      Audience:  jwt.ClaimStrings{"orders-api"},
      ExpiresAt: jwt.NewNumericDate(now.Add(15 * time.Minute)),
      NotBefore: jwt.NewNumericDate(now.Add(-time.Minute)),
      IssuedAt:  jwt.NewNumericDate(now),
      ID:        "jti-0001",
    },
  }
  if mutate != nil {
    mutate(&c)
  }
  s, err := jwt.NewWithClaims(method, c).SignedString(k)
  if err != nil {
    panic(err)
  }
  return s
}

func parse(tokenStr string, opts ...jwt.ParserOption) (*Claims, error) {
  keyfunc := func(t *jwt.Token) (any, error) {
    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
      return nil, fmt.Errorf("unexpected signing method %q", t.Method.Alg())
    }
    return key, nil
  }
  all := append([]jwt.ParserOption{
    jwt.WithValidMethods([]string{"HS256"}),
    jwt.WithIssuer("orders-api"),
    jwt.WithAudience("orders-api"),
    jwt.WithExpirationRequired(),
    jwt.WithTimeFunc(func() time.Time { return now }),
  }, opts...)

  var c Claims
  _, err := jwt.ParseWithClaims(tokenStr, &c, keyfunc, all...)
  if err != nil {
    return nil, err
  }
  return &c, nil
}

var sentinels = []struct {
  name string
  err  error
}{
  {"ErrTokenExpired", jwt.ErrTokenExpired},
  {"ErrTokenNotValidYet", jwt.ErrTokenNotValidYet},
  {"ErrTokenSignatureInvalid", jwt.ErrTokenSignatureInvalid},
  {"ErrTokenMalformed", jwt.ErrTokenMalformed},
  {"ErrTokenInvalidAudience", jwt.ErrTokenInvalidAudience},
  {"ErrTokenInvalidIssuer", jwt.ErrTokenInvalidIssuer},
  {"ErrTokenInvalidClaims", jwt.ErrTokenInvalidClaims},
}

func report(label, tokenStr string, opts ...jwt.ParserOption) {
  c, err := parse(tokenStr, opts...)
  if err == nil {
    fmt.Printf("%-22s OK   sub=%s email=%s jti=%s\n", label, c.Subject, c.Email, c.ID)
    return
  }
  fmt.Printf("%-22s FAIL %v\n", label, err)
  var matched []string
  for _, s := range sentinels {
    if errors.Is(err, s.err) {
      matched = append(matched, s.name)
    }
  }
  fmt.Printf("%-22s      errors.Is -> %v\n", "", matched)
}

func main() {
  rsaKey, err := rsa.GenerateKey(rand.Reader, 2048)
  if err != nil {
    panic(err)
  }

  report("valid", mint(jwt.SigningMethodHS256, key, nil))

  report("expired", mint(jwt.SigningMethodHS256, key, func(c *Claims) {
    c.ExpiresAt = jwt.NewNumericDate(now.Add(-time.Minute))
  }))

  report("not yet valid", mint(jwt.SigningMethodHS256, key, func(c *Claims) {
    c.NotBefore = jwt.NewNumericDate(now.Add(time.Hour))
  }))

  report("wrong signature", mint(jwt.SigningMethodHS256, otherKey, nil))

  report("malformed", "not.a.jwt")
  report("two segments", "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ4In0")

  report("wrong audience", mint(jwt.SigningMethodHS256, key, func(c *Claims) {
    c.Audience = jwt.ClaimStrings{"billing-api"}
  }))

  report("wrong issuer", mint(jwt.SigningMethodHS256, key, func(c *Claims) {
    c.Issuer = "evil-api"
  }))

  report("RS256 token", mint(jwt.SigningMethodRS256, rsaKey, nil))

  fmt.Println("\n=== leeway lets a slightly-expired token through ===")
  slightlyExpired := mint(jwt.SigningMethodHS256, key, func(c *Claims) {
    c.ExpiresAt = jwt.NewNumericDate(now.Add(-20 * time.Second))
  })
  report("no leeway", slightlyExpired)
  report("30s leeway", slightlyExpired, jwt.WithLeeway(30*time.Second))
}
```

Output:

```
valid                  OK   sub=user-42 email=ada@example.com jti=jti-0001
expired                FAIL token has invalid claims: token is expired
                            errors.Is -> [ErrTokenExpired ErrTokenInvalidClaims]
not yet valid          FAIL token has invalid claims: token is not valid yet
                            errors.Is -> [ErrTokenNotValidYet ErrTokenInvalidClaims]
wrong signature        FAIL token signature is invalid: signature is invalid
                            errors.Is -> [ErrTokenSignatureInvalid]
malformed              FAIL token is malformed: could not JSON decode header: invalid character '\x9e' looking for beginning of value
                            errors.Is -> [ErrTokenMalformed]
two segments           FAIL token is malformed: token contains an invalid number of segments
                            errors.Is -> [ErrTokenMalformed]
wrong audience         FAIL token has invalid claims: token has invalid audience
                            errors.Is -> [ErrTokenInvalidAudience ErrTokenInvalidClaims]
wrong issuer           FAIL token has invalid claims: token has invalid issuer
                            errors.Is -> [ErrTokenInvalidIssuer ErrTokenInvalidClaims]
RS256 token            FAIL token signature is invalid: signing method RS256 is invalid
                            errors.Is -> [ErrTokenSignatureInvalid]

=== leeway lets a slightly-expired token through ===
no leeway              FAIL token has invalid claims: token is expired
                            errors.Is -> [ErrTokenExpired ErrTokenInvalidClaims]
30s leeway             OK   sub=user-42 email=ada@example.com jti=jti-0001
```

jwt/v5 wraps with `%w` ([8.3](../08-errors/03-is-as-unwrap.md)), so every claim failure matches
both a **specific** sentinel and the **general** `ErrTokenInvalidClaims` — branch on
`ErrTokenExpired` to tell a client "refresh" while treating everything else as a flat 401.
`errors.Is`, never string matching.

`WithTimeFunc` makes the whole validation path a pure function, which is why these lines are reproducible; use it in every test involving expiry.

**Clock skew** is real — a token minted on a host 5 seconds ahead of the verifier is
`not valid yet` the instant it is issued. `WithLeeway(30*time.Second)` is a reasonable default;
the last two lines show the same token rejected without it and accepted with it. Do not set it
large, since it also extends the life of expired tokens.

## Auth middleware, end to end

16.6's middleware shape, 11.7's context-key convention, 16.9's error-body shape, and the
401/403 distinction people get wrong. **401 means I do not know who you are**; **403 means I know
who you are and you may not do this.** Sending 403 for a missing token makes clients retry
instead of logging in; sending 401 for an insufficient role makes them log in again pointlessly.

```go
package main

import (
  "context"
  "encoding/json"
  "errors"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "slices"
  "strings"
  "sync"
  "time"

  "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
  Email string   `json:"email"`
  Roles []string `json:"roles"`
  jwt.RegisteredClaims
}

type userKey struct{}

func UserFrom(ctx context.Context) (*Claims, bool) {
  c, ok := ctx.Value(userKey{}).(*Claims)
  return c, ok
}

type Denylist struct {
  mu  sync.RWMutex
  ids map[string]time.Time
}

func NewDenylist() *Denylist { return &Denylist{ids: map[string]time.Time{}} }

func (d *Denylist) Revoke(jti string, until time.Time) {
  d.mu.Lock()
  defer d.mu.Unlock()
  d.ids[jti] = until
}

func (d *Denylist) Revoked(jti string, now time.Time) bool {
  d.mu.RLock()
  defer d.mu.RUnlock()
  until, ok := d.ids[jti]
  return ok && now.Before(until)
}

type Authenticator struct {
  Secret   []byte
  Issuer   string
  Audience string
  Now      func() time.Time
  Deny     *Denylist
}

var ErrRevoked = errors.New("token has been revoked")

func (a *Authenticator) Parse(raw string) (*Claims, error) {
  keyfunc := func(t *jwt.Token) (any, error) {
    if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
      return nil, fmt.Errorf("unexpected signing method %q", t.Method.Alg())
    }
    return a.Secret, nil
  }
  var c Claims
  _, err := jwt.ParseWithClaims(raw, &c, keyfunc,
    jwt.WithValidMethods([]string{"HS256"}),
    jwt.WithIssuer(a.Issuer),
    jwt.WithAudience(a.Audience),
    jwt.WithExpirationRequired(),
    jwt.WithLeeway(30*time.Second),
    jwt.WithTimeFunc(a.Now),
  )
  if err != nil {
    return nil, err
  }
  if c.ID != "" && a.Deny.Revoked(c.ID, a.Now()) {
    return nil, ErrRevoked
  }
  return &c, nil
}

func writeJSONErr(w http.ResponseWriter, status int, code, msg string) {
  w.Header().Set("Content-Type", "application/json")
  if status == http.StatusUnauthorized {
    w.Header().Set("WWW-Authenticate", `Bearer realm="api"`)
  }
  w.WriteHeader(status)
  json.NewEncoder(w).Encode(map[string]any{
    "error": map[string]string{"code": code, "message": msg},
  })
}

func (a *Authenticator) Middleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    h := r.Header.Get("Authorization")
    if h == "" {
      writeJSONErr(w, http.StatusUnauthorized, "missing_token", "Authentication is required.")
      return
    }
    scheme, raw, found := strings.Cut(h, " ")
    if !found || !strings.EqualFold(scheme, "Bearer") || raw == "" {
      writeJSONErr(w, http.StatusUnauthorized, "invalid_auth_header",
        "Authorization header must be: Bearer <token>")
      return
    }
    claims, err := a.Parse(strings.TrimSpace(raw))
    if err != nil {
      code := "invalid_token"
      switch {
      case errors.Is(err, jwt.ErrTokenExpired):
        code = "token_expired"
      case errors.Is(err, jwt.ErrTokenNotValidYet):
        code = "token_not_yet_valid"
      case errors.Is(err, jwt.ErrTokenMalformed):
        code = "token_malformed"
      case errors.Is(err, jwt.ErrTokenSignatureInvalid):
        code = "token_signature_invalid"
      case errors.Is(err, ErrRevoked):
        code = "token_revoked"
      }
      writeJSONErr(w, http.StatusUnauthorized, code, "The access token is not valid.")
      return
    }
    ctx := context.WithValue(r.Context(), userKey{}, claims)
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}

func RequireRole(role string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      c, ok := UserFrom(r.Context())
      if !ok {
        writeJSONErr(w, http.StatusUnauthorized, "missing_token", "Authentication is required.")
        return
      }
      if !slices.Contains(c.Roles, role) {
        writeJSONErr(w, http.StatusForbidden, "forbidden",
          "You do not have permission to do that.")
        return
      }
      next.ServeHTTP(w, r)
    })
  }
}

var (
  secret = []byte("test-signing-key-at-least-32-bytes!!")
  wrong  = []byte("a-completely-different-signing-key!!")
  now    = time.Date(2026, 1, 1, 12, 0, 0, 0, time.UTC)
)

func mint(k []byte, roles []string, jti string, exp, nbf time.Time) string {
  c := Claims{
    Email: "ada@example.com",
    Roles: roles,
    RegisteredClaims: jwt.RegisteredClaims{
      Issuer:    "orders-api",
      Subject:   "user-42",
      Audience:  jwt.ClaimStrings{"orders-api"},
      ExpiresAt: jwt.NewNumericDate(exp),
      NotBefore: jwt.NewNumericDate(nbf),
      IssuedAt:  jwt.NewNumericDate(now),
      ID:        jti,
    },
  }
  s, err := jwt.NewWithClaims(jwt.SigningMethodHS256, c).SignedString(k)
  if err != nil {
    panic(err)
  }
  return s
}

func main() {
  deny := NewDenylist()
  auth := &Authenticator{
    Secret:   secret,
    Issuer:   "orders-api",
    Audience: "orders-api",
    Now:      func() time.Time { return now },
    Deny:     deny,
  }

  me := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    c, _ := UserFrom(r.Context())
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]any{
      "sub": c.Subject, "email": c.Email, "roles": c.Roles,
    })
  })
  admin := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, `{"ok":"admin area"}`)
  })

  mux := http.NewServeMux()
  mux.Handle("GET /me", auth.Middleware(me))
  mux.Handle("GET /admin", auth.Middleware(RequireRole("admin")(admin)))

  srv := httptest.NewServer(mux)
  defer srv.Close()

  valid := mint(secret, []string{"user"}, "jti-1", now.Add(15*time.Minute), now.Add(-time.Minute))
  adminTok := mint(secret, []string{"user", "admin"}, "jti-2", now.Add(15*time.Minute), now.Add(-time.Minute))
  expired := mint(secret, []string{"user"}, "jti-3", now.Add(-time.Hour), now.Add(-2*time.Hour))
  future := mint(secret, []string{"user"}, "jti-4", now.Add(2*time.Hour), now.Add(time.Hour))
  badsig := mint(wrong, []string{"user", "admin"}, "jti-5", now.Add(15*time.Minute), now.Add(-time.Minute))
  revoked := mint(secret, []string{"user"}, "jti-6", now.Add(15*time.Minute), now.Add(-time.Minute))
  deny.Revoke("jti-6", now.Add(time.Hour))

  type tc struct {
    label, path, header string
  }
  cases := []tc{
    {"no header", "/me", ""},
    {"wrong scheme", "/me", "Basic YWRtaW46eA=="},
    {"Bearer with no token", "/me", "Bearer"},
    {"malformed token", "/me", "Bearer not.a.jwt"},
    {"valid token", "/me", "Bearer " + valid},
    {"lowercase bearer", "/me", "bearer " + valid},
    {"expired token", "/me", "Bearer " + expired},
    {"not yet valid", "/me", "Bearer " + future},
    {"wrong signature", "/me", "Bearer " + badsig},
    {"revoked jti", "/me", "Bearer " + revoked},
    {"user hits /admin", "/admin", "Bearer " + valid},
    {"admin hits /admin", "/admin", "Bearer " + adminTok},
    {"no header on /admin", "/admin", ""},
  }

  for _, c := range cases {
    req, _ := http.NewRequest("GET", srv.URL+c.path, nil)
    if c.header != "" {
      req.Header.Set("Authorization", c.header)
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      fmt.Printf("%-20s client err %v\n", c.label, err)
      continue
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    wa := resp.Header.Get("WWW-Authenticate")
    fmt.Printf("%-20s %-8s %d  %s\n", c.label, c.path, resp.StatusCode, strings.TrimSpace(string(b)))
    if wa != "" {
      fmt.Printf("%-20s          WWW-Authenticate: %s\n", "", wa)
    }
  }
}
```

Output:

```
no header            /me      401  {"error":{"code":"missing_token","message":"Authentication is required."}}
                              WWW-Authenticate: Bearer realm="api"
wrong scheme         /me      401  {"error":{"code":"invalid_auth_header","message":"Authorization header must be: Bearer \u003ctoken\u003e"}}
                              WWW-Authenticate: Bearer realm="api"
Bearer with no token /me      401  {"error":{"code":"invalid_auth_header","message":"Authorization header must be: Bearer \u003ctoken\u003e"}}
                              WWW-Authenticate: Bearer realm="api"
malformed token      /me      401  {"error":{"code":"token_malformed","message":"The access token is not valid."}}
                              WWW-Authenticate: Bearer realm="api"
valid token          /me      200  {"email":"ada@example.com","roles":["user"],"sub":"user-42"}
lowercase bearer     /me      200  {"email":"ada@example.com","roles":["user"],"sub":"user-42"}
expired token        /me      401  {"error":{"code":"token_expired","message":"The access token is not valid."}}
                              WWW-Authenticate: Bearer realm="api"
not yet valid        /me      401  {"error":{"code":"token_not_yet_valid","message":"The access token is not valid."}}
                              WWW-Authenticate: Bearer realm="api"
wrong signature      /me      401  {"error":{"code":"token_signature_invalid","message":"The access token is not valid."}}
                              WWW-Authenticate: Bearer realm="api"
revoked jti          /me      401  {"error":{"code":"token_revoked","message":"The access token is not valid."}}
                              WWW-Authenticate: Bearer realm="api"
user hits /admin     /admin   403  {"error":{"code":"forbidden","message":"You do not have permission to do that."}}
admin hits /admin    /admin   200  {"ok":"admin area"}
no header on /admin  /admin   401  {"error":{"code":"missing_token","message":"Authentication is required."}}
                              WWW-Authenticate: Bearer realm="api"
```

Five details in that output earn their place:

**401 for every authentication failure, 403 only for the role check.** `user hits /admin` is
403 — correctly authenticated, insufficiently authorised. `no header on /admin` is 401, because
`auth.Middleware` short-circuits before `RequireRole` runs. The middleware **order** is what
makes that right: authenticate outside, authorise inside.

**The `code` is machine-readable, the `message` is not.** `token_expired` tells a client to
refresh; `invalid_token` tells it to log in again. Both carry the same vague `message`, which is
deliberate — a detailed 401 body is an oracle telling an attacker whether the signature was
wrong (keep probing) or the token merely expired (the format is right).

**`WWW-Authenticate: Bearer` on every 401, never on the 403** (RFC 9110: the header advertises
*how* to authenticate, meaningless once you are). **`strings.EqualFold`** so `bearer` works —
the scheme is case-insensitive and real clients send all three casings.

**`strings.Cut` on the single space**, not `Split` or `TrimPrefix("Bearer ")` — `Cut` handles
the no-space case (`"Bearer"` alone) without a bounds check, which is the
`Bearer with no token` row.

**The context key is `userKey{}`, an unexported empty struct type**, not a string: a string key
can collide with another package's, and `userKey{}` is unforgeable from outside the package
([11.7](../11-sync-context/07-context-values.md)). `UserFrom` returns `(*Claims, bool)` so a
handler reached without the middleware cannot nil-panic, and `RequireRole`'s 401-on-no-claims
branch fails closed if someone wires it without `Middleware`.

## Access and refresh tokens

The core tension: a JWT is valid until it expires because validation is stateless, so a stolen
token cannot be taken back. Short expiry limits the damage and also means logging in every 15
minutes. The two-token split resolves it:

| | Access token | Refresh token |
|---|---|---|
| Lifetime | 5-15 minutes | 7-30 days |
| Sent with | every API request | only to `POST /auth/refresh` |
| Validated | statelessly, by signature | against **server-side state** |
| Audience | the API | the auth endpoint only (`aud: "refresh"`) |
| On theft | expires on its own, quickly | must be revoked, which is why it is stateful |

The trade-off, plainly: **you do not escape server-side state, you shrink it.** Instead of a
session lookup per request, you do one per refresh — hundreds of times less often. Refuse to
store refresh-token state and you have no logout and no revocation, and a stolen token works
for its full lifetime.

**Rotation** is the other half. Every refresh issues a *new* refresh token and invalidates the
old one, so if an attacker steals one and uses it, the legitimate user's next refresh presents
the now-invalid old token — and that reuse is a **detectable signal** that the token was stolen,
at which point you revoke the whole family and force a re-login. Storing
`(token_hash, user_id, family_id, expires_at, used_at)` is enough, and the handler is a
`ConsumeRefresh` that returns `ErrAlreadyUsed` plus a `RevokeFamily` on that branch.

Store a **hash** of the refresh token for the same reason you hash passwords: a database leak
must not yield usable credentials.

## Logout and revocation by jti

Logout deletes the refresh token server-side, stopping renewal; the already-issued access token
stays cryptographically valid for its remaining minutes. If that is unacceptable — an admin
console, a banking app — add the `Denylist` from the middleware above, keyed by `jti`; the
`revoked jti` row in that output is it doing its job.

In production the denylist is Redis with a TTL equal to the token's remaining lifetime, so it
stays small — you only need to remember a revoked `jti` until it would have expired anyway. For
a fleet-wide "log everyone out", a single `tokens_valid_after` timestamp per user compared
against `iat` is cheaper than an entry per token.

Note that a denylist is a lookup on every request — the statelessness you were trying to avoid.
Which is the honest segue to the next point.

## Password hashing

Never store a password. Store a slow, salted hash.

```go
package main

import (
  "fmt"
  "strings"

  "golang.org/x/crypto/bcrypt"
)

func main() {
  password := []byte("correct horse battery staple")

  hash, err := bcrypt.GenerateFromPassword(password, bcrypt.DefaultCost)
  if err != nil {
    panic(err)
  }
  fmt.Println("stored hash length:", len(hash), "bytes")
  parts := strings.Split(string(hash), "$")
  fmt.Printf("format: $%s$%s$<22-char salt + 31-char digest>\n", parts[1], parts[2])
  cost, _ := bcrypt.Cost(hash)
  fmt.Println("cost embedded in hash:", cost)

  fmt.Println("\ncorrect password:", bcrypt.CompareHashAndPassword(hash, password))
  err = bcrypt.CompareHashAndPassword(hash, []byte("Correct horse battery staple"))
  fmt.Println("wrong password:  ", err)
  fmt.Println("errors.Is mismatch sentinel:", err == bcrypt.ErrMismatchedHashAndPassword)

  again, _ := bcrypt.GenerateFromPassword(password, bcrypt.DefaultCost)
  fmt.Println("\nsame password hashed twice is identical:", string(hash) == string(again))
  fmt.Println("both verify:",
    bcrypt.CompareHashAndPassword(hash, password) == nil,
    bcrypt.CompareHashAndPassword(again, password) == nil)

  _, err = bcrypt.GenerateFromPassword(make([]byte, 100), bcrypt.DefaultCost)
  fmt.Println("\n100-byte password:", err)
  fmt.Println("MinCost, DefaultCost, MaxCost =",
    bcrypt.MinCost, bcrypt.DefaultCost, bcrypt.MaxCost)
}
```

Output:

```
stored hash length: 60 bytes
format: $2a$10$<22-char salt + 31-char digest>
cost embedded in hash: 10

correct password: <nil>
wrong password:   crypto/bcrypt: hashedPassword is not the hash of the given password
errors.Is mismatch sentinel: true

same password hashed twice is identical: false
both verify: true true

100-byte password: bcrypt: password length exceeds 72 bytes
MinCost, DefaultCost, MaxCost = 4 10 31
```

Everything you need is in those lines. The hash is **60 bytes**, so size the column `char(60)`.
The **salt is inside the hash**, which is why the same password hashes to different strings and
why you never manage salts. The **cost is inside the hash too**, so you can raise `DefaultCost`
later and re-hash on next login without breaking old hashes — check `bcrypt.Cost(hash)` after a
successful compare and upgrade if it is below target. And bcrypt **truncates at 72 bytes**; Go
errors rather than truncating silently, so pre-hash with SHA-256 for longer passphrases.

Cost 10 is roughly 50-80ms per hash — that is the point, and also a denial-of-service surface, so
rate-limit your login endpoint. `argon2id` (`golang.org/x/crypto/argon2`) is the stronger modern
choice, memory-hard so GPUs help an attacker less, at the cost of tuning three parameters
instead of one; bcrypt remains perfectly defensible.

## Where the client stores the token, in one line

An `httpOnly; Secure; SameSite=Strict` cookie is immune to XSS theft but needs CSRF protection,
while `localStorage` needs no CSRF protection but any XSS on your origin exfiltrates the token —
so cookie plus CSRF token for a browser app, and `Authorization: Bearer` from memory for a
native or server-to-server client.

## Sessions are often simpler

For a monolith with its own database, a random opaque session ID in an httpOnly cookie plus a
`sessions` table is less code, revocable instantly, has no claim-staleness problem, and costs one
indexed lookup per request. JWTs pay off when several services must verify tokens without calling
the auth service, which is not most applications. Choose them for a reason, not by default.

## Quick Reference

| Function | Purpose |
|---|---|
| `jwt.NewWithClaims(method, claims)` | build an unsigned `*Token` |
| `tok.SignedString(key)` | `(string, error)` — the compact serialization |
| `jwt.ParseWithClaims(s, &claims, keyfunc, opts...)` | parse, verify signature, validate claims |
| `jwt.Parse(s, keyfunc, opts...)` | same, into `jwt.MapClaims` |
| `jwt.NewNumericDate(t)` | `*NumericDate` for `exp`/`nbf`/`iat` |
| `jwt.ClaimStrings{...}` | `aud`, which may be a string or an array in JSON |
| `jwt.NewParser(opts...)` | a reusable parser instead of per-call options; `bcrypt.Cost(hash)` reads a hash's cost |

| Signing method | Key type to sign | Key type to verify |
|---|---|---|
| `jwt.SigningMethodHS256` | `[]byte` | `[]byte` (same) |
| `jwt.SigningMethodRS256` | `*rsa.PrivateKey` | `*rsa.PublicKey` |
| `jwt.SigningMethodES256` | `*ecdsa.PrivateKey` | `*ecdsa.PublicKey` |
| `jwt.SigningMethodEdDSA` | `ed25519.PrivateKey` | `ed25519.PublicKey` |

| Sentinel | Meaning |
|---|---|
| `jwt.ErrTokenMalformed` | not three base64url segments, or undecodable JSON |
| `jwt.ErrTokenSignatureInvalid` | signature mismatch, or `alg` not in `WithValidMethods` |
| `jwt.ErrTokenUnverifiable` | the keyfunc returned an error |
| `jwt.ErrTokenExpired` | `exp` in the past |
| `jwt.ErrTokenNotValidYet` | `nbf` in the future |
| `jwt.ErrTokenInvalidAudience` / `InvalidIssuer` | `aud`/`iss` mismatch |
| `jwt.ErrTokenRequiredClaimMissing` | e.g. no `exp` under `WithExpirationRequired` |
| `jwt.ErrTokenInvalidClaims` | the umbrella all claim failures also match |

| Status | Send when |
|---|---|
| **401** + `WWW-Authenticate: Bearer` | missing, malformed, expired, bad-signature or revoked token |
| **403** (no `WWW-Authenticate`) | valid token, insufficient role or scope |

## Common Mistakes

**A keyfunc that returns the key without checking `t.Method`.** An attacker signs an HS256
token with your public RSA key and becomes an admin; assert the method type and also pass
`jwt.WithValidMethods`.

**Putting anything sensitive in the payload.** It is base64, not encryption — anyone with the
token reads every claim.

**Returning 403 for a missing or expired token.** Clients then retry forever instead of
refreshing or logging in; 401 means unauthenticated.

**Trusting claims without validating `iss` and `aud`.** A token minted by your staging
environment, or by a different service sharing the secret, is otherwise accepted.

**Comparing errors with `==` or by string.** jwt/v5 wraps everything; use `errors.Is` against
the sentinels.

**Assuming a JWT can be revoked.** It cannot, by construction — add a `jti` denylist or keep
access-token lifetimes short enough that you do not care.

**No `WithExpirationRequired()`.** `exp` is optional in the RFC, so a token with no `exp`
validates forever.

**Storing passwords with SHA-256, with or without a salt.** It is far too fast; use bcrypt or
argon2id.

## Exercises

1. Build `POST /auth/login`, `POST /auth/refresh` and `POST /auth/logout` with bcrypt password
   checking, a 10-minute access token and a 14-day rotating refresh token in an in-memory
   store. Drive all three through `httptest` and show a refresh-reuse attempt revoking the
   family.
2. Switch the middleware to RS256 with a keypair generated at startup, serve a JWKS document
   at `/.well-known/jwks.json`, and write a second process that fetches the JWKS and validates
   a token with it. Then re-run the algorithm-confusion attack against your own keyfunc and
   confirm it is rejected.
3. Add scope-based authorisation: claims carry `scopes: ["orders:read", "orders:write"]` and
   `RequireScope("orders:write")` returns 403 on a mismatch. Table-test all four combinations
   of (has scope, has role).
4. Write a test that asserts a token is rejected one second after `exp` and accepted one second
   before, using `jwt.WithTimeFunc`, with no `time.Sleep` anywhere.
5. Implement `tokens_valid_after` per user: store a timestamp, reject any token whose `iat` is
   before it, and expose an admin endpoint that bumps it. Compare the request-path cost against
   the `jti` denylist and write down which you would ship.

---

[← 18.2 Configuration via Environment](02-config.md) · [Course Index](../README.md) · [18.4 Structured Logging in a Service →](04-service-logging.md)
