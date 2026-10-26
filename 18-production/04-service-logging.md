[← 18.3 Authentication with JWT](03-auth-jwt.md) · [Course Index](../README.md) · [18.5 Graceful Shutdown →](05-graceful-shutdown.md)

# 18.4 — Structured Logging in a Service

## What You'll Learn

- The production handler: JSON to stdout, level from config, and a `slog.LevelVar` you can flip at runtime without a restart
- The per-request logger carried in the context, set once by middleware, used by every layer
- `slog.LogValuer` for automatic redaction — the most valuable twenty lines in this lesson
- Logging an error once at the boundary, and testing log output with a `bytes.Buffer`

## Coming From Another Language

| Other | Go |
|---|---|
| `winston`/`pino` configured once and imported everywhere | a `*slog.Logger` value you construct and **pass**; the global is a fallback, not the design |
| `logging.getLogger(__name__)` per-module hierarchy | no hierarchy; `logger.With(...)` derives a child with extra attrs |
| `logger.info(f"user {id} did {thing}")` | `logger.Info("did thing", "user_id", id)` — the message is a constant, the data is fields |
| a request-scoped logger from async-local storage | `context.Context` carries it explicitly (11.7) |
| `tracing`/`log` crate macros | plain function calls; no macro magic, so `AddSource` costs a `runtime.Callers` |

[13.12](../13-stdlib/12-slog.md) covers the `slog` API — levels, `Attr`, handlers, the
`Handler` interface. This lesson is about the four or five decisions you make once per service.

## The production handler

```go
package main

import (
  "fmt"
  "log/slog"
  "os"
)

func stripTime(groups []string, a slog.Attr) slog.Attr {
  if a.Key == slog.TimeKey && len(groups) == 0 {
    return slog.Attr{}
  }
  return a
}

func NewLogger(w *os.File, level *slog.LevelVar, format string, addSource bool) *slog.Logger {
  opts := &slog.HandlerOptions{
    Level:       level,
    AddSource:   addSource,
    ReplaceAttr: stripTime,
  }
  var h slog.Handler
  if format == "text" {
    h = slog.NewTextHandler(w, opts)
  } else {
    h = slog.NewJSONHandler(w, opts)
  }
  return slog.New(h).With("service", "orders-api", "version", "1.4.2")
}

func main() {
  level := new(slog.LevelVar)
  level.Set(slog.LevelInfo)
  log := NewLogger(os.Stdout, level, "json", false)

  log.Debug("cache lookup", "key", "user:42", "hit", false)
  log.Info("server started", "addr", ":8080")
  log.Warn("connection pool near capacity", "in_use", 18, "max", 20)
  log.Error("payment provider unreachable", "provider", "stripe", "attempt", 3)

  fmt.Println("\n--- flipping the level to DEBUG at runtime, no restart ---")
  level.Set(slog.LevelDebug)
  log.Debug("cache lookup", "key", "user:42", "hit", false)
  log.Info("level changed", "from", "INFO", "to", level.Level().String())

  fmt.Println("\n--- back to WARN: Info and Debug disappear ---")
  level.Set(slog.LevelWarn)
  log.Debug("invisible")
  log.Info("also invisible")
  log.Warn("still visible", "reason", "level is WARN")

  fmt.Println("\n--- text format, same logger construction ---")
  level.Set(slog.LevelInfo)
  NewLogger(os.Stdout, level, "text", false).Info("server started", "addr", ":8080")
}
```

`ReplaceAttr` drops the `time` attribute so this output is byte-reproducible — in production
you keep it, and your log platform indexes it. Note the `len(groups) == 0` guard: without it,
an attribute *named* `time` inside a group would also be dropped.

Output:

```
{"level":"INFO","msg":"server started","service":"orders-api","version":"1.4.2","addr":":8080"}
{"level":"WARN","msg":"connection pool near capacity","service":"orders-api","version":"1.4.2","in_use":18,"max":20}
{"level":"ERROR","msg":"payment provider unreachable","service":"orders-api","version":"1.4.2","provider":"stripe","attempt":3}

--- flipping the level to DEBUG at runtime, no restart ---
{"level":"DEBUG","msg":"cache lookup","service":"orders-api","version":"1.4.2","key":"user:42","hit":false}
{"level":"INFO","msg":"level changed","service":"orders-api","version":"1.4.2","from":"INFO","to":"DEBUG"}

--- back to WARN: Info and Debug disappear ---
{"level":"WARN","msg":"still visible","service":"orders-api","version":"1.4.2","reason":"level is WARN"}

--- text format, same logger construction ---
level=INFO msg="server started" service=orders-api version=1.4.2 addr=:8080
```

The decisions encoded there:

**JSON to stdout, not to a file.** Containers log to stdout and the platform collects it;
writing a file means log rotation, disk-full outages and a volume mount you did not need. Text
for local development, JSON everywhere else — one `LogFormat` config value
([18.2](02-config.md)), not an `if env == "production"`.

**`*slog.LevelVar` as the `Level`, not a bare `slog.Level`.** `LevelVar` is a mutable,
atomic level holder, so `level.Set(slog.LevelDebug)` takes effect on the next log call across
every goroutine. Wire it to an admin endpoint and you can turn on debug logging during an
incident without a deploy:

```go
mux.HandleFunc("PUT /admin/log-level", func(w http.ResponseWriter, r *http.Request) {
  var l slog.Level
  if err := l.UnmarshalText([]byte(r.FormValue("level"))); err != nil {
    http.Error(w, "bad level", http.StatusBadRequest)
    return
  }
  level.Set(l)
  w.WriteHeader(http.StatusNoContent)
})
```

Put that behind auth. It is also the one piece of state worth exposing in your readiness body
so you can see which pods are chatty.

**`AddSource` conditionally.** It attaches `source:{file,line,function}` to every record,
which costs a `runtime.Callers` per call — fine at `Error` volume, measurable at `Debug`
volume in a hot loop. Enable it in development and in staging; in production enable it only if
you actually use it, because the file and line of the log call is usually less useful than the
error-wrapping chain ([8.2](../08-errors/02-wrapping.md)) already in the message.

**Base attributes via `.With(...)` at construction.** `service` and `version` are on every
line, which is what lets you query one service's logs out of a shared index. Add `env`,
`hostname` and the git commit here too; `.With` computes the attrs once rather than per call.

## Global logger or passed logger

`slog.SetDefault(log)` makes `slog.Info(...)` work package-wide and also redirects the
standard `log` package through your handler, which is worth doing solely to catch
`net/http`'s own error output.

The recommendation, stated plainly: **call `slog.SetDefault` once in `main`, and still pass
the logger explicitly.** The default is a safety net for library code and for the places a
dependency cannot reach your struct; your own code takes a `*slog.Logger` so a test can hand
it a buffer and so no package depends on global initialisation order.

```go
func main() {
  level := new(slog.LevelVar)
  log := NewLogger(os.Stdout, level, cfg.LogFormat, cfg.Env != "production")
  slog.SetDefault(log)

  svc := order.NewService(repo, log.With("component", "order"))
}
```

Handlers are the exception: threading a logger into every handler signature is noise, and the
request logger must be request-scoped anyway. That is what the context is for.

## The per-request logger

Middleware builds a logger carrying the request's identity, puts it in the context, and every
layer below pulls it out. One pattern, three lines of plumbing, and now every log line in the
request's lifetime is correlated.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "io"
  "log/slog"
  "net/http"
  "net/http/httptest"
  "os"
  "sync/atomic"
  "time"
)

type loggerKey struct{}

func WithLogger(ctx context.Context, l *slog.Logger) context.Context {
  return context.WithValue(ctx, loggerKey{}, l)
}

func LoggerFrom(ctx context.Context) *slog.Logger {
  if l, ok := ctx.Value(loggerKey{}).(*slog.Logger); ok {
    return l
  }
  return slog.Default()
}

type recorder struct {
  http.ResponseWriter
  status int
  bytes  int
}

func (rw *recorder) WriteHeader(code int) {
  if rw.status == 0 {
    rw.status = code
    rw.ResponseWriter.WriteHeader(code)
  }
}

func (rw *recorder) Write(b []byte) (int, error) {
  if rw.status == 0 {
    rw.status = http.StatusOK
  }
  n, err := rw.ResponseWriter.Write(b)
  rw.bytes += n
  return n, err
}

func (rw *recorder) Unwrap() http.ResponseWriter { return rw.ResponseWriter }

type reqIDKey struct{}

func RequestIDFrom(ctx context.Context) string {
  id, _ := ctx.Value(reqIDKey{}).(string)
  return id
}

func RequestID(gen func() string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      id := r.Header.Get("X-Request-Id")
      if id == "" {
        id = gen()
      }
      w.Header().Set("X-Request-Id", id)
      next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), reqIDKey{}, id)))
    })
  }
}

func RequestLogger(base *slog.Logger, clock func() time.Duration) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      reqLog := base.With(
        "request_id", RequestIDFrom(r.Context()),
        "method", r.Method,
        "path", r.URL.Path,
      )
      ctx := WithLogger(r.Context(), reqLog)
      rw := &recorder{ResponseWriter: w}
      next.ServeHTTP(rw, r.WithContext(ctx))

      lvl := slog.LevelInfo
      switch {
      case rw.status >= 500:
        lvl = slog.LevelError
      case rw.status >= 400:
        lvl = slog.LevelWarn
      }
      reqLog.Log(ctx, lvl, "request",
        "status", rw.status,
        "bytes", rw.bytes,
        "duration_ms", clock().Milliseconds(),
        "remote_ip", "203.0.113.7",
        "user_agent", r.UserAgent(),
      )
    })
  }
}

var ErrNotFound = errors.New("resource not found")

func main() {
  log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      if a.Key == slog.TimeKey && len(g) == 0 {
        return slog.Attr{}
      }
      return a
    },
  })).With("service", "orders-api")

  var n atomic.Int64
  gen := func() string { return fmt.Sprintf("req-%03d", n.Add(1)) }
  fixedDuration := func() time.Duration { return 12 * time.Millisecond }

  mux := http.NewServeMux()
  mux.HandleFunc("GET /orders/{id}", func(w http.ResponseWriter, r *http.Request) {
    l := LoggerFrom(r.Context())
    l.Debug("loading order", "order_id", r.PathValue("id"))
    l.Info("order served", "order_id", r.PathValue("id"), "total_cents", 4999)
    io.WriteString(w, `{"id":"`+r.PathValue("id")+`"}`)
  })
  mux.HandleFunc("GET /missing", func(w http.ResponseWriter, r *http.Request) {
    err := fmt.Errorf("loading order 999 from table orders: %w", ErrNotFound)
    LoggerFrom(r.Context()).Warn("request rejected", "internal_error", err.Error())
    http.Error(w, `{"error":{"code":"not_found","trace_id":"`+RequestIDFrom(r.Context())+`"}}`,
      http.StatusNotFound)
  })
  mux.HandleFunc("GET /broken", func(w http.ResponseWriter, r *http.Request) {
    err := fmt.Errorf("query postgres://app:hunter2@db.internal:5432/prod: connection refused")
    LoggerFrom(r.Context()).Error("request failed", "internal_error", err.Error())
    http.Error(w, `{"error":{"code":"internal_error","trace_id":"`+RequestIDFrom(r.Context())+`"}}`,
      http.StatusInternalServerError)
  })

  handler := RequestID(gen)(RequestLogger(log, fixedDuration)(mux))
  srv := httptest.NewServer(handler)
  srv.Config.ErrorLog = slog.NewLogLogger(slog.NewTextHandler(io.Discard, nil), slog.LevelError)
  defer srv.Close()

  for _, p := range []string{"/orders/7", "/missing", "/broken"} {
    req, _ := http.NewRequest("GET", srv.URL+p, nil)
    req.Header.Set("User-Agent", "course-client/1.0")
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      fmt.Println("client err", err)
      continue
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("CLIENT %-10s %d  X-Request-Id=%s  body=%s\n",
      p, resp.StatusCode, resp.Header.Get("X-Request-Id"), b)
  }
}
```

The request ID generator and the duration are **injected** so the output is reproducible; in
production they are a UUID and `time.Since(start)`.

Output:

```
{"level":"DEBUG","msg":"loading order","service":"orders-api","request_id":"req-001","method":"GET","path":"/orders/7","order_id":"7"}
{"level":"INFO","msg":"order served","service":"orders-api","request_id":"req-001","method":"GET","path":"/orders/7","order_id":"7","total_cents":4999}
{"level":"INFO","msg":"request","service":"orders-api","request_id":"req-001","method":"GET","path":"/orders/7","status":200,"bytes":10,"duration_ms":12,"remote_ip":"203.0.113.7","user_agent":"course-client/1.0"}
CLIENT /orders/7  200  X-Request-Id=req-001  body={"id":"7"}
{"level":"WARN","msg":"request rejected","service":"orders-api","request_id":"req-002","method":"GET","path":"/missing","internal_error":"loading order 999 from table orders: resource not found"}
{"level":"WARN","msg":"request","service":"orders-api","request_id":"req-002","method":"GET","path":"/missing","status":404,"bytes":52,"duration_ms":12,"remote_ip":"203.0.113.7","user_agent":"course-client/1.0"}
CLIENT /missing   404  X-Request-Id=req-002  body={"error":{"code":"not_found","trace_id":"req-002"}}

{"level":"ERROR","msg":"request failed","service":"orders-api","request_id":"req-003","method":"GET","path":"/broken","internal_error":"query postgres://app:hunter2@db.internal:5432/prod: connection refused"}
{"level":"ERROR","msg":"request","service":"orders-api","request_id":"req-003","method":"GET","path":"/broken","status":500,"bytes":57,"duration_ms":12,"remote_ip":"203.0.113.7","user_agent":"course-client/1.0"}
CLIENT /broken    500  X-Request-Id=req-003  body={"error":{"code":"internal_error","trace_id":"req-003"}}
```

**Every line carries `request_id`, `method` and `path` without the handler passing them.**
`base.With(...)` returns a derived logger; `WithLogger` puts it in the context; `LoggerFrom`
retrieves it. A handler, a service and a repository all call `LoggerFrom(ctx)` and all produce
correlated lines. The `*slog.Logger` is immutable and safe to share, so there is no race.

**`LoggerFrom` falls back to `slog.Default()`.** That is deliberate: a code path reached
without the middleware (a background job, a test) logs to the default handler instead of
nil-panicking. Returning a working logger rather than `(logger, bool)` is the right trade-off
here, because a missing logger must never break a request.

**The access log level tracks the status.** 500 is `ERROR`, 4xx is `WARN`, 2xx is `INFO` — the
same split as 16.9, so an alert on `level=ERROR` fires on your bugs and not on clients sending
bad input.

**The 500's internal error stays internal.** The log line has
`postgres://app:hunter2@db.internal:5432/prod: connection refused`; the client's body is
`{"error":{"code":"internal_error","trace_id":"req-003"}}`. `trace_id` in the response equals
`request_id` in the log, which is the entire correlation mechanism — the user quotes
`req-003` in a ticket and you grep one line. This is 16.9's rule, now with the logging half
visible.

## What to log, and at which level

| Level | Means | Real examples |
|---|---|---|
| `Debug` | diagnostic detail, off in production | `cache miss`, `query plan chosen`, `retry 2 of 5`, parsed request params |
| `Info` | a thing happened that you would want in an audit | `server started`, `order created`, `user logged in`, `migration applied`, the access log |
| `Warn` | degraded but handled; look at it tomorrow | `pool near capacity`, `fell back to cache`, `rate limit hit`, `deprecated endpoint used`, 4xx |
| `Error` | **someone should look at this**; it is a bug or an outage | `database unreachable`, `panic recovered`, `payment provider 5xx after retries`, 500s |

The rule that keeps the signal usable: **`Error` means a human should investigate.** A client
sending malformed JSON is not an error on your side — it is `Warn` at most, and arguably
nothing at all beyond the access log. The failure mode of ignoring this is that your `Error`
stream fills with routine client misbehaviour, everyone stops reading it, and the real outage
scrolls past unnoticed. If you cannot write the sentence "when this fires, do X", it is not
`Error`.

A corollary for retries: log the *final* failure at `Error` and the intermediate attempts at
`Debug`. Logging every attempt at `Error` multiplies one incident by your retry count.

## What never to log

Passwords (even wrong ones — users type their password into the username field), tokens and
API keys, session IDs, full `Authorization` headers, card numbers, government IDs, full request
and response bodies, and email addresses or names beyond what your privacy policy allows.

The problem is that "remember not to log it" fails. Someone adds a convenient
`log.Info("user", "user", u)` and the whole struct lands in your log index, where it is
retained for ninety days and replicated to a vendor. The fix is to make it **structurally
impossible**: implement `slog.LogValuer` on the types, so the redaction travels with the data.

```go
package main

import (
  "fmt"
  "log/slog"
  "os"
  "strings"
)

type Secret string

func (s Secret) LogValue() slog.Value {
  if s == "" {
    return slog.StringValue("")
  }
  return slog.StringValue(fmt.Sprintf("[REDACTED %d chars]", len(s)))
}

func (s Secret) String() string { return "[REDACTED]" }

type Email string

func (e Email) LogValue() slog.Value {
  name, domain, ok := strings.Cut(string(e), "@")
  if !ok {
    return slog.StringValue("[invalid-email]")
  }
  if len(name) <= 1 {
    return slog.StringValue("*@" + domain)
  }
  return slog.StringValue(name[:1] + strings.Repeat("*", len(name)-1) + "@" + domain)
}

type CardNumber string

func (c CardNumber) LogValue() slog.Value {
  s := string(c)
  if len(s) < 4 {
    return slog.StringValue("****")
  }
  return slog.StringValue("**** **** **** " + s[len(s)-4:])
}

type User struct {
  ID           int
  Email        Email
  PasswordHash Secret
  APIToken     Secret
  Card         CardNumber
  Plan         string
}

func (u User) LogValue() slog.Value {
  return slog.GroupValue(
    slog.Int("id", u.ID),
    slog.Any("email", u.Email),
    slog.String("plan", u.Plan),
  )
}

type UnsafeUser struct {
  ID           int
  Email        string
  PasswordHash string
  APIToken     string
  Card         string
  Plan         string
}

func main() {
  log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      if a.Key == slog.TimeKey && len(g) == 0 {
        return slog.Attr{}
      }
      return a
    },
  }))

  u := User{
    ID:           42,
    Email:        "ada.lovelace@example.com",
    PasswordHash: "$2a$10$N9qo8uLOickgx2ZMRZoMye",
    APIToken:     "sk_live_51H8xQ2abcdef",
    Card:         "4242424242424242",
    Plan:         "enterprise",
  }

  fmt.Println("=== the naive version: everything leaks ===")
  bad := UnsafeUser{42, "ada.lovelace@example.com", "$2a$10$N9qo8uLOickgx2ZMRZoMye",
    "sk_live_51H8xQ2abcdef", "4242424242424242", "enterprise"}
  log.Info("user loaded", "user", bad)

  fmt.Println("\n=== with LogValue on the struct: only safe fields ===")
  log.Info("user loaded", "user", u)

  fmt.Println("\n=== the typed fields redact themselves wherever they appear ===")
  log.Info("auth attempt",
    "email", u.Email,
    "token", u.APIToken,
    "card", u.Card,
    "hash", u.PasswordHash)

  fmt.Println("\n=== and inside an error, and in fmt output, because String() is defined too ===")
  log.Error("charge failed", "err", fmt.Errorf("declined for %v on %v", u.APIToken, u.Card))
  fmt.Printf("fmt.Printf: token=%v card=%v\n", u.APIToken, u.Card)

  fmt.Println("\n=== edge cases ===")
  log.Info("edges",
    "empty_secret", Secret(""),
    "short_secret", Secret("x"),
    "one_char_local", Email("a@example.com"),
    "not_an_email", Email("nonsense"),
    "short_card", CardNumber("42"))
}
```

Output:

```
=== the naive version: everything leaks ===
{"level":"INFO","msg":"user loaded","user":{"ID":42,"Email":"ada.lovelace@example.com","PasswordHash":"$2a$10$N9qo8uLOickgx2ZMRZoMye","APIToken":"sk_live_51H8xQ2abcdef","Card":"4242424242424242","Plan":"enterprise"}}

=== with LogValue on the struct: only safe fields ===
{"level":"INFO","msg":"user loaded","user":{"id":42,"email":"a***********@example.com","plan":"enterprise"}}

=== the typed fields redact themselves wherever they appear ===
{"level":"INFO","msg":"auth attempt","email":"a***********@example.com","token":"[REDACTED 21 chars]","card":"**** **** **** 4242","hash":"[REDACTED 29 chars]"}

=== and inside an error, and in fmt output, because String() is defined too ===
{"level":"ERROR","msg":"charge failed","err":"declined for [REDACTED] on 4242424242424242"}
fmt.Printf: token=[REDACTED] card=4242424242424242

=== edge cases ===
{"level":"INFO","msg":"edges","empty_secret":"","short_secret":"[REDACTED 1 chars]","one_char_local":"*@example.com","not_an_email":"[invalid-email]","short_card":"****"}
```

Compare lines 2 and 4 of that output. The `UnsafeUser` dumped a bcrypt hash, a live API token
and a full card number into the log index. The `User` with `LogValue()` emitted `id`, a masked
`email` and `plan` — because `LogValue` returns a `slog.GroupValue` listing exactly the fields
you allow, so a field added to the struct later is **invisible by default**. That default is
the whole point: an allowlist fails safe, a denylist does not.

The mechanism: `slog` checks every value for the one-method interface

```go
type LogValuer interface {
  LogValue() Value
}
```

and calls it, recursively, before formatting. So `Secret`, `Email` and `CardNumber` redact
themselves in *any* log line in the program, in any order of attributes, written by anyone —
including code written next year by someone who has never read this lesson. Declare the field
as `Secret` in your struct and the problem is solved once.

### The leak this example actually caught

Look at the error line: `declined for [REDACTED] on 4242424242424242`. The token is redacted,
the **card number is not**. `fmt.Errorf("%v")` does not know about `LogValuer` — it looks for
`fmt.Stringer`. `Secret` has a `String()` method, `CardNumber` did not, so the card leaked
through the error and through `fmt.Printf` alike.

Adding the missing method fixes it:

```go
func (c CardNumber) String() string { return c.LogValue().String() }
```

Output:

```
{"level":"ERROR","msg":"charge failed","err":"declined for [REDACTED] on **** **** **** 4242"}
fmt.Printf: token=[REDACTED] card=**** **** **** 4242
```

So the rule is **implement both**: `LogValue()` for `slog`, `String()` for `fmt` and for every
`%v` in an error message, with `String()` usually delegating to `LogValue()`. A sensitive type
with only one of them has a hole, and the hole is in error messages, which is exactly where
sensitive values tend to end up.

One more caution: `LogValue()` is also called for the `Debug` level, so keep it cheap and
allocation-light, and never let it panic — a panicking `LogValue` takes down the log call.

## Structured fields, not interpolated strings

```go
package main

import (
  "fmt"
  "log/slog"
  "os"
)

func main() {
  log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      if a.Key == slog.TimeKey && len(g) == 0 {
        return slog.Attr{}
      }
      return a
    },
  }))

  fmt.Println("=== interpolated string: one unqueryable field ===")
  userID, from, to, ms := 42, "free", "enterprise", 1203
  log.Info(fmt.Sprintf("user %d upgraded from %s to %s in %dms", userID, from, to, ms))

  fmt.Println("\n=== structured: every value is a field ===")
  log.Info("plan upgraded",
    "user_id", userID,
    "from_plan", from,
    "to_plan", to,
    "duration_ms", ms)

  fmt.Println("\n=== slog.Group nests related fields ===")
  log.Info("request",
    slog.Group("http",
      slog.String("method", "POST"),
      slog.String("route", "/orders/{id}"),
      slog.Int("status", 201),
    ),
    slog.Group("db",
      slog.Int("queries", 3),
      slog.Int("duration_ms", 8),
    ),
    slog.String("request_id", "req-001"),
  )

  fmt.Println("\n=== WithGroup applies to everything after it ===")
  dbLog := log.WithGroup("db").With("pool", "primary")
  dbLog.Info("slow query", "duration_ms", 1840, "table", "orders")

  fmt.Println("\n=== trace correlation is just two more attrs ===")
  log.Info("order created",
    "trace_id", "4bf92f3577b34da6a3ce929d0e0e4736",
    "span_id", "00f067aa0ba902b7",
    "order_id", "ord_7")
}
```

Output:

```
=== interpolated string: one unqueryable field ===
{"level":"INFO","msg":"user 42 upgraded from free to enterprise in 1203ms"}

=== structured: every value is a field ===
{"level":"INFO","msg":"plan upgraded","user_id":42,"from_plan":"free","to_plan":"enterprise","duration_ms":1203}

=== slog.Group nests related fields ===
{"level":"INFO","msg":"request","http":{"method":"POST","route":"/orders/{id}","status":201},"db":{"queries":3,"duration_ms":8},"request_id":"req-001"}

=== WithGroup applies to everything after it ===
{"level":"INFO","msg":"slow query","db":{"pool":"primary","duration_ms":1840,"table":"orders"}}

=== trace correlation is just two more attrs ===
{"level":"INFO","msg":"order created","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"00f067aa0ba902b7","order_id":"ord_7"}
```

The consequence is a query. With the first line, finding slow upgrades means a regex over
`msg` and hoping nobody reworded it. With the second it is
`to_plan:"enterprise" AND duration_ms > 1000`, and `avg(duration_ms) by from_plan` is a
dashboard. Keep `msg` a **constant string** — it is the group-by key — and put every variable
in a field.

`slog.Group` namespaces related attrs so `http.status` cannot collide with a database
`status`; `WithGroup` does the same for every attr a derived logger emits afterwards.

**OpenTelemetry correlation** is just `trace_id` and `span_id` as attributes, pulled from the
span in the context (`trace.SpanContextFromContext(ctx)`) in the same middleware that builds
the request logger — then a log line links to a trace and back.

## Log the error once, at the boundary

16.9's rule, restated from the logging side: the repository wraps with `%w` and returns, the
service wraps and returns, and the **one** place that calls `logger.Error` is the handler
adapter or the request-logging middleware. The wrapping chain already contains everything each
layer would have logged:

```
"creating order for user 42: charging card: upstream 503 from stripe: context deadline exceeded"
```

One line, full causal chain, one severity, one `request_id`. The alternative — each layer
logging as the error passes — gives four lines at three severities with no shared identifier,
and your incident review turns into archaeology.

## Canonical log lines

The access-log line above is a **canonical log line**: one rich record per request carrying
everything about it — status, duration, route, user ID, query count, cache hits, bytes. The
case for it is that one row answers almost every operational question, needs no join, and
costs you one log event per request instead of twenty. The case against is that you lose the
*sequence* of what happened inside the request, so a hang halfway through shows up only as a
large `duration_ms` with no indication of where. The practical answer is both, split by level:
a canonical `Info` line always, plus `Debug` lines for the steps that you enable when you need
them. Build the canonical line by accumulating attributes into a per-request struct and
emitting it once in the middleware's deferred tail.

**Sampling** is the other half of volume control: for a high-traffic endpoint, log every error
and every slow request but only 1-in-N successes, which is a `rand.IntN(n) == 0` check or a
`Handler` wrapper that drops records — and remember to record the sample rate as a field so
counts can be scaled back up.

## Testing log output

Hand the handler a `bytes.Buffer`, parse the JSON, assert on fields. This is the only honest
way to test that an error is logged and that a secret is not.

```go
package main

import (
  "bytes"
  "encoding/json"
  "log/slog"
  "strings"
  "testing"
)

func newTestLogger() (*slog.Logger, *bytes.Buffer) {
  var buf bytes.Buffer
  h := slog.NewJSONHandler(&buf, &slog.HandlerOptions{
    Level: slog.LevelDebug,
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      if a.Key == slog.TimeKey && len(g) == 0 {
        return slog.Attr{}
      }
      return a
    },
  })
  return slog.New(h), &buf
}

type record struct {
  Level   string         `json:"level"`
  Msg     string         `json:"msg"`
  Account map[string]any `json:"account"`
  Amount  int            `json:"amount_cents"`
  Reason  string         `json:"reason"`
}

func parseLines(t *testing.T, buf *bytes.Buffer) []record {
  t.Helper()
  var out []record
  for _, line := range strings.Split(strings.TrimSpace(buf.String()), "\n") {
    if line == "" {
      continue
    }
    var r record
    if err := json.Unmarshal([]byte(line), &r); err != nil {
      t.Fatalf("log line is not valid JSON: %v\n%s", err, line)
    }
    out = append(out, r)
  }
  return out
}

var acct = Account{ID: 42, Token: "sk_live_51H8xQ2abcdef", Plan: "enterprise",
  Password: "$2a$10$N9qo8uLOickgx2ZMRZoMye"}

func TestChargeLogsInfoOnSuccess(t *testing.T) {
  log, buf := newTestLogger()
  if err := ChargeAccount(log, acct, 4999); err != nil {
    t.Fatalf("ChargeAccount = %v", err)
  }
  lines := parseLines(t, buf)
  if len(lines) != 1 {
    t.Fatalf("got %d log lines, want 1:\n%s", len(lines), buf)
  }
  r := lines[0]
  if r.Level != "INFO" {
    t.Errorf("level = %q, want INFO", r.Level)
  }
  if r.Msg != "charging account" {
    t.Errorf("msg = %q", r.Msg)
  }
  if r.Amount != 4999 {
    t.Errorf("amount_cents = %d, want 4999", r.Amount)
  }
  if got := r.Account["id"]; got != float64(42) {
    t.Errorf("account.id = %v, want 42", got)
  }
  if _, present := r.Account["Token"]; present {
    t.Error("account group contains Token")
  }
}

func TestChargeLogsWarnOnRejection(t *testing.T) {
  log, buf := newTestLogger()
  if err := ChargeAccount(log, acct, 0); err == nil {
    t.Fatal("ChargeAccount(0) succeeded, want error")
  }
  lines := parseLines(t, buf)
  if len(lines) != 2 {
    t.Fatalf("got %d log lines, want 2:\n%s", len(lines), buf)
  }
  if lines[1].Level != "WARN" {
    t.Errorf("level = %q, want WARN", lines[1].Level)
  }
  if lines[1].Reason != "non-positive amount" {
    t.Errorf("reason = %q", lines[1].Reason)
  }
}

func TestNoSecretReachesTheLog(t *testing.T) {
  log, buf := newTestLogger()
  ChargeAccount(log, acct, 4999)
  ChargeAccount(log, acct, -1)
  for _, secret := range []string{"sk_live_51H8xQ2abcdef", "$2a$10$N9qo8uLOickgx2ZMRZoMye"} {
    if strings.Contains(buf.String(), secret) {
      t.Errorf("log leaked %q:\n%s", secret, buf)
    }
  }
}
```

The code under test:

```go
package main

import (
  "fmt"
  "log/slog"
)

type Secret string

func (s Secret) LogValue() slog.Value {
  if s == "" {
    return slog.StringValue("")
  }
  return slog.StringValue(fmt.Sprintf("[REDACTED %d chars]", len(s)))
}

func (s Secret) String() string { return "[REDACTED]" }

type Account struct {
  ID       int
  Token    Secret
  Plan     string
  Password Secret
}

func (a Account) LogValue() slog.Value {
  return slog.GroupValue(
    slog.Int("id", a.ID),
    slog.String("plan", a.Plan),
  )
}

func ChargeAccount(log *slog.Logger, a Account, cents int) error {
  log.Info("charging account", "account", a, "amount_cents", cents)
  if cents <= 0 {
    log.Warn("rejected charge", "account", a, "amount_cents", cents,
      "reason", "non-positive amount")
    return fmt.Errorf("amount must be positive, got %d", cents)
  }
  return nil
}

func main() {}
```

`go test -v` (the elapsed time varies run to run):

Output:

```
=== RUN   TestChargeLogsInfoOnSuccess
--- PASS: TestChargeLogsInfoOnSuccess (0.00s)
=== RUN   TestChargeLogsWarnOnRejection
--- PASS: TestChargeLogsWarnOnRejection (0.00s)
=== RUN   TestNoSecretReachesTheLog
--- PASS: TestNoSecretReachesTheLog (0.00s)
PASS
ok  	example.com/scratch	0.528s
```

Three things that make these tests useful rather than brittle. The handler writes to a buffer,
so the function under test takes a `*slog.Logger` parameter — which is the argument for passing
the logger rather than reaching for the global. `ReplaceAttr` strips `time`, so assertions do
not have to tolerate a timestamp. And `TestNoSecretReachesTheLog` asserts on the **raw buffer
text** rather than on parsed fields, because a leak could appear anywhere — in `msg`, in a
nested group, in an error string — and a substring check over the whole output catches all of
them.

Numbers come back from `encoding/json` as `float64` into a `map[string]any`, hence
`got != float64(42)` ([13.6](../13-stdlib/06-json.md)).

## Log aggregation, in one line

Your platform (Loki, Elasticsearch, Datadog, CloudWatch) ingests the JSON from stdout, indexes
the fields, and charges you by volume and retention — which is why `Debug` is off in production
and why one canonical line per request beats twenty small ones.

## Quick Reference

| Piece | Call |
|---|---|
| JSON handler | `slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{...})` |
| runtime-adjustable level | `v := new(slog.LevelVar); v.Set(slog.LevelDebug)`; pass `v` as `Level` |
| drop/rewrite an attr | `HandlerOptions.ReplaceAttr func([]string, Attr) Attr` |
| file and line | `HandlerOptions.AddSource: true` |
| base attrs | `log.With("service", "api", "version", v)` |
| nested attrs | `slog.Group("http", slog.Int("status", 200))` or `log.WithGroup("db")` |
| set the global and redirect `log` | `slog.SetDefault(log)` |
| bridge to `*log.Logger` | `slog.NewLogLogger(handler, level)` |
| dynamic level | `log.Log(ctx, lvl, "msg", ...)` |
| automatic redaction | implement `LogValue() slog.Value` **and** `String() string` |
| typed attrs (no `any` boxing) | `slog.String`, `slog.Int`, `slog.Duration`, `slog.Time`, `slog.Any` |

| Level | Fires when | Alert on it? |
|---|---|---|
| `Debug` | diagnostics; off in production | no |
| `Info` | expected events worth auditing | no |
| `Warn` | degraded but handled; 4xx | trend it |
| `Error` | a human must investigate; 5xx | yes |

## Common Mistakes

**Interpolating values into `msg`.** The field becomes unqueryable and the message stops being
a usable group-by key; pass values as attributes.

**Logging a whole struct without `LogValue`.** Tokens, hashes and card numbers land in your
log index; implement `slog.LogValuer` with an allowlist of fields.

**Implementing `LogValue` but not `String`.** The value still leaks through `fmt.Errorf("%v")`
into error messages; implement both.

**Logging the same error at every layer.** One failure becomes four uncorrelated lines; wrap
with `%w` and log once at the boundary.

**Using `Error` for client mistakes.** The `Error` stream fills with routine 4xx and nobody
reads it any more; 4xx is `Warn` at most.

**A fixed `slog.Level` instead of a `*slog.LevelVar`.** You then need a deploy to turn on debug
logging during an incident.

**Reaching for `slog.Default()` inside business logic.** Tests cannot capture the output and
there is no request correlation; take a `*slog.Logger` or read it from the context.

**Writing logs to a file.** You have signed up for rotation, disk-full outages and a volume
mount; write JSON to stdout.

## Exercises

1. Build the canonical-log-line middleware: accumulate attributes into a per-request struct
   (status, duration, route, user ID, DB query count, cache hits) and emit exactly one `Info`
   record per request. Prove with a test that a request touching three layers produces one
   line.
2. Write a `slog.Handler` wrapper that samples: pass every record at `Warn` and above, and
   1-in-10 at `Info`, adding a `sample_rate` attribute. Verify the ratio over 1000 records.
3. Add `LogValue()` and `String()` to a `PhoneNumber` type and an `IPAddress` type (mask the
   last octet), then write the test that asserts no unmasked form of either appears in 50
   generated log lines.
4. Implement `PUT /admin/log-level` behind the auth middleware from 18.3, and prove with
   `httptest` that a `Debug` line is absent before the call and present after.
5. Write a handler wrapper that adds `trace_id`/`span_id` from an OpenTelemetry span in the
   context when one is present and omits the fields when it is not. Test both paths without
   running a collector.

---

[← 18.3 Authentication with JWT](03-auth-jwt.md) · [Course Index](../README.md) · [18.5 Graceful Shutdown →](05-graceful-shutdown.md)
