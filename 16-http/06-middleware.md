[← 16.5 JSON Requests and Responses](05-json-api.md) · [Course Index](../README.md) · [16.7 chi: Routing, Groups, and Middleware →](07-chi.md)

# 16.6 — Middleware from Scratch

## What You'll Learn

- `func(http.Handler) http.Handler` is the entire convention — no framework, no registry, no `next()` callback
- The `ResponseWriter` wrapper that captures status and bytes, and the `Unwrap` method that keeps streaming alive behind it
- Recovery middleware that turns a panic into a real 500 — and proof that it cannot save a response that already started
- Composition order, a `Chain` helper, short-circuiting, and the real interleaved trace

## Coming From Another Language

| Other | Go |
|---|---|
| Express `(req, res, next) => {}` and you call `next()` | `func(next http.Handler) http.Handler`; you call `next.ServeHTTP(w, r)` |
| Express `app.use(mw)` mutates a global stack | you *compose functions*; there is no stack to mutate |
| `next(err)` propagates an error to an error handler | no error channel — return early, or use the pattern in 16.9 |
| Express: throwing in a handler hits the default error handler | nothing is caught; **you** write the recovery middleware |
| Django/ASGI middleware classes with `__call__` | a closure returning `http.HandlerFunc` |
| Rust `tower::Layer` | the same idea, without the type gymnastics |

## The shape, from first principles

A middleware needs to run code before and/or after a handler, and must itself be usable
wherever a handler is. Since `http.Handler` is an interface, the signature writes itself:

```go
func Middleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    // before
    next.ServeHTTP(w, r)
    // after
  })
}
```

That is the whole convention, and because it is just a function type, every Go router and
every third-party middleware package interoperates. Give it a name so signatures read
better:

```go
type Middleware func(http.Handler) http.Handler
```

A middleware needing configuration becomes a function *returning* a middleware:

```go
func RequireHeader(name string) Middleware {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      if r.Header.Get(name) == "" {
        http.Error(w, "missing "+name, http.StatusBadRequest)
        return
      }
      next.ServeHTTP(w, r)
    })
  }
}
```

Not calling `next.ServeHTTP` **short-circuits** the chain. That is how auth and rate limits
reject a request, and there is no special API for it — you just return.

## Order of composition

Nesting `A(B(C(h)))` means A is outermost: its "before" runs first and its "after" runs
last. Here is the real interleaving, plus a `Chain` helper that preserves the reading order,
plus a short-circuit.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
)

func mw(name string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      fmt.Printf("-> %s before\n", name)
      next.ServeHTTP(w, r)
      fmt.Printf("<- %s after\n", name)
    })
  }
}

type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, mws ...Middleware) http.Handler {
  for i := len(mws) - 1; i >= 0; i-- {
    h = mws[i](h)
  }
  return h
}

func main() {
  final := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Println("   == handler runs ==")
    io.WriteString(w, "done")
  })

  fmt.Println("--- nested by hand: A(B(C(h))) ---")
  nested := mw("A")(mw("B")(mw("C")(final)))
  rec := httptest.NewRecorder()
  nested.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))
  fmt.Printf("body=%q\n\n", rec.Body.String())

  fmt.Println("--- Chain(h, A, B, C) -- same order ---")
  chained := Chain(final, mw("A"), mw("B"), mw("C"))
  rec = httptest.NewRecorder()
  chained.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))
  fmt.Printf("body=%q\n", rec.Body.String())

  fmt.Println("\n--- short-circuiting middleware stops the chain ---")
  block := func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      fmt.Println("-> BLOCK: rejecting, not calling next")
      http.Error(w, "forbidden", http.StatusForbidden)
    })
  }
  sc := Chain(final, mw("A"), block, mw("C"))
  rec = httptest.NewRecorder()
  sc.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))
  fmt.Printf("status=%d body=%q\n", rec.Code, rec.Body.String())
}
```

Output:

```
--- nested by hand: A(B(C(h))) ---
-> A before
-> B before
-> C before
   == handler runs ==
<- C after
<- B after
<- A after
body="done"

--- Chain(h, A, B, C) -- same order ---
-> A before
-> B before
-> C before
   == handler runs ==
<- C after
<- B after
<- A after
body="done"

--- short-circuiting middleware stops the chain ---
-> A before
-> BLOCK: rejecting, not calling next
<- A after
status=403 body="forbidden\n"
```

It is a **stack**: before-code runs outside-in, after-code runs inside-out. The `Chain`
helper iterates backwards precisely so that `Chain(h, A, B, C)` reads in the same order as
`A(B(C(h)))` executes — which is the order you want when the list gets long.

Note the short-circuit trace: `BLOCK` never called `next`, so `C` and the handler never ran,
but `A`'s after-code still did. Outer middleware always gets its "after" phase, which is
exactly what makes logging and recovery work.

**The order that is correct for a real service**, outermost first:

```go
handler := Chain(router,
  Recoverer,        // 1. outermost: must catch panics from everything below
  RequestID(gen),   // 2. so every log line below has an ID
  Logger,           // 3. inside RequestID, so it can log the ID
  RealIP,           // 4. fix r.RemoteAddr before anyone reads it
  Timeout(5*time.Second),
  CORS(origins),    // 5. must answer preflights before auth rejects them
  Authenticate,     // 6. after CORS, before anything business-specific
  RateLimit,
)
```

Two orderings people get wrong: **`Recoverer` outside `Logger`** (otherwise a panic in the
logger itself is unhandled, and a recovered panic never gets logged with its request), and
**`CORS` outside auth** (otherwise the browser's unauthenticated `OPTIONS` preflight gets a
401 and the real request is never sent — a maddening bug to diagnose).

## The ResponseWriter wrapper, and Unwrap

A logger needs the status code and the byte count, and `ResponseWriter` exposes neither
(16.3). The canonical fix is a wrapper — and the canonical *bug* is writing it without
`Unwrap`.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
)

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

type brokenRecorder struct {
  http.ResponseWriter
  status int
}

func (rw *brokenRecorder) WriteHeader(c int) { rw.status = c; rw.ResponseWriter.WriteHeader(c) }

func main() {
  probe := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    _, isFlusher := w.(http.Flusher)
    _, isHijacker := w.(http.Hijacker)
    rc := http.NewResponseController(w)
    fmt.Printf("  seen by handler: %T\n", w)
    fmt.Printf("    w.(http.Flusher)  = %v\n", isFlusher)
    fmt.Printf("    w.(http.Hijacker) = %v\n", isHijacker)
    fmt.Printf("    ResponseController.Flush() = %v\n", rc.Flush())
    io.WriteString(w, "ok")
  })

  good := func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      rw := &recorder{ResponseWriter: w}
      next.ServeHTTP(rw, r)
      fmt.Printf("  logged: status=%d bytes=%d\n", rw.status, rw.bytes)
    })
  }
  broken := func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      next.ServeHTTP(&brokenRecorder{ResponseWriter: w}, r)
    })
  }

  mux := http.NewServeMux()
  mux.Handle("/good", good(probe))
  mux.Handle("/broken", broken(probe))
  mux.Handle("/bare", probe)

  srv := httptest.NewServer(mux)
  defer srv.Close()

  for _, p := range []string{"/bare", "/good", "/broken"} {
    fmt.Printf("=== %s\n", p)
    resp, _ := http.Get(srv.URL + p)
    io.ReadAll(resp.Body)
    resp.Body.Close()
  }

  fmt.Println("\n=== implicit 200 captured correctly ===")
  implicit := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "no WriteHeader call")
  })
  rec := httptest.NewRecorder()
  rw := &recorder{ResponseWriter: rec}
  implicit.ServeHTTP(rw, httptest.NewRequest("GET", "/", nil))
  fmt.Printf("recorded status=%d bytes=%d\n", rw.status, rw.bytes)
}
```

Output:

```
=== /bare
  seen by handler: *http.response
    w.(http.Flusher)  = true
    w.(http.Hijacker) = true
    ResponseController.Flush() = <nil>
=== /good
  seen by handler: *main.recorder
    w.(http.Flusher)  = false
    w.(http.Hijacker) = false
    ResponseController.Flush() = <nil>
  logged: status=200 bytes=2
=== /broken
  seen by handler: *main.brokenRecorder
    w.(http.Flusher)  = false
    w.(http.Hijacker) = false
    ResponseController.Flush() = feature not supported

=== implicit 200 captured correctly ===
recorded status=200 bytes=19
```

This is the whole problem and the whole fix in one capture.

**Both wrappers break the type assertions.** `/bare` sees `*http.response` with `Flusher`
and `Hijacker` true; behind either middleware both are `false`. Embedding
`http.ResponseWriter` gives the wrapper only the interface's three methods, so any handler
doing `w.(http.Flusher)` stops streaming the moment you add logging. That bug is invisible
in tests that do not check for it, and it only manifests as "our SSE endpoint buffers in
production".

**`ResponseController` saves the one with `Unwrap`.** `/good` flushes fine (`<nil>`);
`/broken` gets `feature not supported`. `http.NewResponseController` walks the
`Unwrap() http.ResponseWriter` chain to find the real writer. Four lines of boilerplate per
wrapper.

**The implicit 200 is captured.** `status=200` for a handler that never called
`WriteHeader`, because `Write` defaults it. Without that branch you log `0` for every
successful response.

**`WriteHeader` guards against a double call.** `if rw.status == 0` means a second
`WriteHeader` is swallowed by the wrapper rather than reaching the real writer, which
suppresses the `superfluous response.WriteHeader` log (16.1) — pick whether you want that;
some prefer to let it through so the bug stays visible.

A logging middleware on top of that wrapper:

```go
func Logger(log *slog.Logger) Middleware {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      start := time.Now()
      rw := &recorder{ResponseWriter: w}
      next.ServeHTTP(rw, r)
      log.Info("request",
        "method", r.Method,
        "path", r.URL.Path,
        "status", rw.status,
        "bytes", rw.bytes,
        "duration_ms", time.Since(start).Milliseconds(),
        "request_id", RequestIDFrom(r.Context()))
    })
  }
}
```

Log the matched **pattern** rather than the raw path if you feed this to metrics —
`mux.Handler(r)` gives it to you (16.2) and it keeps label cardinality bounded.

## Recovery middleware

Turn a panic into a 500 instead of a dropped connection (16.3).

```go
package main

import (
  "errors"
  "fmt"
  "io"
  "log/slog"
  "net/http"
  "net/http/httptest"
  "os"
  "strings"
)

var log = slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
  ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
    if a.Key == slog.TimeKey {
      return slog.Attr{}
    }
    return a
  },
}))

func Recoverer(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    defer func() {
      rec := recover()
      if rec == nil {
        return
      }
      if err, ok := rec.(error); ok && errors.Is(err, http.ErrAbortHandler) {
        panic(rec)
      }
      log.Error("panic recovered",
        "panic", fmt.Sprint(rec),
        "method", r.Method,
        "path", r.URL.Path,
        "stack_frames", "omitted for reproducibility")
      w.Header().Set("Content-Type", "application/json")
      w.WriteHeader(http.StatusInternalServerError)
      io.WriteString(w, `{"error":"internal server error"}`)
    }()
    next.ServeHTTP(w, r)
  })
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/nil-map", func(w http.ResponseWriter, r *http.Request) {
    var m map[string]int
    m["x"] = 1
  })
  mux.HandleFunc("/string-panic", func(w http.ResponseWriter, r *http.Request) {
    panic("deliberate failure")
  })
  mux.HandleFunc("/abort", func(w http.ResponseWriter, r *http.Request) {
    panic(http.ErrAbortHandler)
  })
  mux.HandleFunc("/after-write", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    io.WriteString(w, `{"partial":`)
    panic("too late")
  })
  mux.HandleFunc("/fine", func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "all good")
  })

  srv := httptest.NewServer(Recoverer(mux))
  srv.Config.ErrorLog = slog.NewLogLogger(
    slog.NewTextHandler(io.Discard, nil), slog.LevelError)
  defer srv.Close()

  for _, p := range []string{"/fine", "/nil-map", "/string-panic", "/after-write", "/abort"} {
    resp, err := http.Get(srv.URL + p)
    if err != nil {
      fmt.Printf("%-15s CLIENT ERR: %v\n", p, strings.Replace(err.Error(), srv.URL, "SRV", 1))
      continue
    }
    b, readErr := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-15s %-28s body=%q readErr=%v\n", p, resp.Status, b, readErr)
  }
}
```

The stack is deliberately not logged here because its frame addresses vary every run; in
production you log `string(debug.Stack())`. The server's own `ErrorLog` is sent to
`io.Discard` so only our middleware's output appears.

Output:

```
/fine           200 OK                       body="all good" readErr=<nil>
level=ERROR msg="panic recovered" panic="assignment to entry in nil map" method=GET path=/nil-map stack_frames="omitted for reproducibility"
/nil-map        500 Internal Server Error    body="{\"error\":\"internal server error\"}" readErr=<nil>
level=ERROR msg="panic recovered" panic="deliberate failure" method=GET path=/string-panic stack_frames="omitted for reproducibility"
/string-panic   500 Internal Server Error    body="{\"error\":\"internal server error\"}" readErr=<nil>
level=ERROR msg="panic recovered" panic="too late" method=GET path=/after-write stack_frames="omitted for reproducibility"
/after-write    200 OK                       body="{\"partial\":{\"error\":\"internal server error\"}" readErr=<nil>
/abort          CLIENT ERR: Get "SRV/abort": EOF
```

Four distinct outcomes, all real:

**`/nil-map` and `/string-panic` became clean 500s** with a JSON body and a log line. The
panic value is captured whether it is a runtime error or a plain string, because `recover()`
returns `any`.

**`/after-write` is the one to stare at.** The response is
`{"partial":{"error":"internal server error"}` — status **200**, and a body that is the
handler's truncated JSON with the error object jammed onto the end. It is not valid JSON and
it is not a 500. The handler had already sent its status (16.1), so `WriteHeader(500)` in the
middleware was a no-op and `Write` simply appended. **Recovery middleware cannot fix a
response that has already started.** The lesson for handler authors: decide your status and
do your fallible work *before* you write anything.

**`/abort` re-panicked**, so the connection closed with no log and no 500, preserving the
`http.ErrAbortHandler` contract. Without that check, `httputil.ReverseProxy`'s deliberate
aborts would turn into bogus 500s in your logs.

Note the `errors.Is` check requires the type assertion to `error` first, because `recover()`
returns `any` and `errors.Is` takes an `error`.

## Request ID, CORS, basic auth, and a timeout

Four more middlewares, all in one real run. The ID generator is **injected** so the output is
reproducible; in production you pass a UUID or random-hex generator.

```go
package main

import (
  "context"
  "crypto/subtle"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
  "sync/atomic"
  "time"
)

type reqIDKey struct{}

func RequestID(gen func() string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      id := r.Header.Get("X-Request-Id")
      if id == "" {
        id = gen()
      }
      w.Header().Set("X-Request-Id", id)
      ctx := context.WithValue(r.Context(), reqIDKey{}, id)
      next.ServeHTTP(w, r.WithContext(ctx))
    })
  }
}

func RequestIDFrom(ctx context.Context) string {
  id, _ := ctx.Value(reqIDKey{}).(string)
  return id
}

func CORS(allowed []string) func(http.Handler) http.Handler {
  allow := func(origin string) bool {
    for _, a := range allowed {
      if a == origin {
        return true
      }
    }
    return false
  }
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      origin := r.Header.Get("Origin")
      if origin != "" && allow(origin) {
        w.Header().Set("Access-Control-Allow-Origin", origin)
        w.Header().Set("Vary", "Origin")
        w.Header().Set("Access-Control-Allow-Credentials", "true")
      }
      if r.Method == http.MethodOptions && r.Header.Get("Access-Control-Request-Method") != "" {
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Access-Control-Max-Age", "86400")
        w.WriteHeader(http.StatusNoContent)
        return
      }
      next.ServeHTTP(w, r)
    })
  }
}

func BasicAuth(realm, user, pass string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      u, p, ok := r.BasicAuth()
      userOK := subtle.ConstantTimeCompare([]byte(u), []byte(user)) == 1
      passOK := subtle.ConstantTimeCompare([]byte(p), []byte(pass)) == 1
      if !ok || !userOK || !passOK {
        w.Header().Set("WWW-Authenticate", fmt.Sprintf("Basic realm=%q, charset=\"UTF-8\"", realm))
        http.Error(w, "unauthorized", http.StatusUnauthorized)
        return
      }
      next.ServeHTTP(w, r)
    })
  }
}

func main() {
  var n atomic.Int64
  fixedGen := func() string { return fmt.Sprintf("req-%03d", n.Add(1)) }

  echo := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "handler saw request_id=%q", RequestIDFrom(r.Context()))
  })
  slow := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    select {
    case <-time.After(500 * time.Millisecond):
      io.WriteString(w, "finished slowly")
    case <-r.Context().Done():
      fmt.Println("  slow handler: ctx done:", r.Context().Err())
    }
  })

  mux := http.NewServeMux()
  mux.Handle("/echo", echo)
  mux.Handle("/slow", http.TimeoutHandler(slow, 80*time.Millisecond, "request timed out"))
  mux.Handle("/private", BasicAuth("api", "admin", "s3cret")(echo))

  srv := httptest.NewServer(RequestID(fixedGen)(CORS([]string{"https://app.example.com"})(mux)))
  defer srv.Close()

  show := func(label string, req *http.Request) {
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      fmt.Printf("%-26s client err %v\n", label, err)
      return
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-26s %s\n", label, resp.Status)
    fmt.Printf("   X-Request-Id=%q\n", resp.Header.Get("X-Request-Id"))
    if o := resp.Header.Get("Access-Control-Allow-Origin"); o != "" {
      fmt.Printf("   ACAO=%q Vary=%q\n", o, resp.Header.Get("Vary"))
    }
    if m := resp.Header.Get("Access-Control-Allow-Methods"); m != "" {
      fmt.Printf("   ACAM=%q ACAH=%q MaxAge=%q\n", m,
        resp.Header.Get("Access-Control-Allow-Headers"),
        resp.Header.Get("Access-Control-Max-Age"))
    }
    if wa := resp.Header.Get("WWW-Authenticate"); wa != "" {
      fmt.Printf("   WWW-Authenticate=%q\n", wa)
    }
    if len(b) > 0 {
      fmt.Printf("   body=%q\n", strings.TrimSpace(string(b)))
    }
  }

  r1, _ := http.NewRequest("GET", srv.URL+"/echo", nil)
  show("plain GET", r1)

  r2, _ := http.NewRequest("GET", srv.URL+"/echo", nil)
  r2.Header.Set("X-Request-Id", "client-supplied-id")
  show("client-supplied id", r2)

  r3, _ := http.NewRequest("GET", srv.URL+"/echo", nil)
  r3.Header.Set("Origin", "https://app.example.com")
  show("allowed Origin", r3)

  r4, _ := http.NewRequest("GET", srv.URL+"/echo", nil)
  r4.Header.Set("Origin", "https://evil.example.com")
  show("disallowed Origin", r4)

  r5, _ := http.NewRequest("OPTIONS", srv.URL+"/echo", nil)
  r5.Header.Set("Origin", "https://app.example.com")
  r5.Header.Set("Access-Control-Request-Method", "POST")
  show("CORS preflight", r5)

  r6, _ := http.NewRequest("GET", srv.URL+"/private", nil)
  show("no credentials", r6)

  r7, _ := http.NewRequest("GET", srv.URL+"/private", nil)
  r7.SetBasicAuth("admin", "wrong")
  show("wrong password", r7)

  r8, _ := http.NewRequest("GET", srv.URL+"/private", nil)
  r8.SetBasicAuth("admin", "s3cret")
  show("correct credentials", r8)

  r9, _ := http.NewRequest("GET", srv.URL+"/slow", nil)
  show("TimeoutHandler (80ms)", r9)
}
```

Output:

```
plain GET                  200 OK
   X-Request-Id="req-001"
   body="handler saw request_id=\"req-001\""
client-supplied id         200 OK
   X-Request-Id="client-supplied-id"
   body="handler saw request_id=\"client-supplied-id\""
allowed Origin             200 OK
   X-Request-Id="req-002"
   ACAO="https://app.example.com" Vary="Origin"
   body="handler saw request_id=\"req-002\""
disallowed Origin          200 OK
   X-Request-Id="req-003"
   body="handler saw request_id=\"req-003\""
CORS preflight             204 No Content
   X-Request-Id="req-004"
   ACAO="https://app.example.com" Vary="Origin"
   ACAM="GET, POST, PUT, DELETE" ACAH="Content-Type, Authorization" MaxAge="86400"
no credentials             401 Unauthorized
   X-Request-Id="req-005"
   WWW-Authenticate="Basic realm=\"api\", charset=\"UTF-8\""
   body="unauthorized"
wrong password             401 Unauthorized
   X-Request-Id="req-006"
   WWW-Authenticate="Basic realm=\"api\", charset=\"UTF-8\""
   body="unauthorized"
correct credentials        200 OK
   X-Request-Id="req-007"
   body="handler saw request_id=\"req-007\""
  slow handler: ctx done: context deadline exceeded
TimeoutHandler (80ms)      503 Service Unavailable
   X-Request-Id="req-008"
   body="request timed out"
```

**Request ID.** It goes in the context (via `r.WithContext`, because `*http.Request` is
immutable by convention — [11.7](../11-sync-context/07-context-values.md)) *and* in the
response header, so a user reporting a failure can quote the ID straight from their browser's
network tab. Honouring an inbound `X-Request-Id` preserves a trace ID across a service
boundary — note the security trade-off: it is client-controlled, so validate its length and
charset before logging it, or an attacker injects newlines into your logs.

**CORS.** The disallowed origin gets **200 with no `Access-Control-Allow-Origin`** — that is
correct. CORS is enforced by the *browser*, not the server; the server's job is to omit the
header, and the browser then refuses to hand the response to JavaScript. `Vary: Origin` is
mandatory when the allowed origin is echoed back, or a shared cache will serve one origin's
response to another. The preflight is only answered when the method is `OPTIONS` **and**
`Access-Control-Request-Method` is present — a bare `OPTIONS` is not a preflight.

**Basic auth** uses `subtle.ConstantTimeCompare` so response time does not leak how many
leading characters matched. Both comparisons run unconditionally for the same reason. The
`WWW-Authenticate` header is what makes a browser show its credential prompt; omit it on a
JSON API where you do not want that.

**`http.TimeoutHandler`** returned 503 with the message, and the handler's own
`r.Context().Done()` fired with `context deadline exceeded`. Its limits are specific and
worth knowing:

| | What it bounds | What happens on expiry |
|---|---|---|
| `http.TimeoutHandler(h, d, msg)` | handler execution | cancels `r.Context()`, writes **503** + `msg` — but only if the handler has not written yet |
| `context.WithTimeout` inside the handler | whatever you select on | you choose the status (504/503) and the body |
| `Server.ReadTimeout` / `WriteTimeout` | socket I/O, not handler logic | the connection is killed; no HTTP response |

`TimeoutHandler` does **not** stop the handler goroutine — Go cannot kill a goroutine. The
handler keeps running until it notices `r.Context()` is done, which is why handlers must
select on it or pass it to every downstream call (16.12). It also buffers the whole response
in memory so it can discard a partial write, which makes it unsuitable in front of a
streaming endpoint. For per-route control inside a handler, `context.WithTimeout` is usually
the better tool.

## Gzip, in sketch

```go
type gzipWriter struct {
  http.ResponseWriter
  gz *gzip.Writer
}

func (g *gzipWriter) Write(b []byte) (int, error) { return g.gz.Write(b) }
func (g *gzipWriter) Unwrap() http.ResponseWriter { return g.ResponseWriter }

func Gzip(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if !strings.Contains(r.Header.Get("Accept-Encoding"), "gzip") {
      next.ServeHTTP(w, r)
      return
    }
    w.Header().Set("Content-Encoding", "gzip")
    w.Header().Add("Vary", "Accept-Encoding")
    w.Header().Del("Content-Length")
    gz := gzip.NewWriter(w)
    defer gz.Close()
    next.ServeHTTP(&gzipWriter{ResponseWriter: w, gz: gz}, r)
  })
}
```

Driven with and without `Accept-Encoding: gzip`, using a client with
`DisableCompression: true` so Go does not transparently decode it:

Output:

```
Accept-Encoding=""     -> 200 OK
  Content-Encoding="" Vary="" Content-Length="900"
  raw bytes received: 900 (uncompressed payload is 900)
Accept-Encoding="gzip" -> 200 OK
  Content-Encoding="gzip" Vary="Accept-Encoding" Content-Length="73"
  raw bytes received: 73 (uncompressed payload is 900)
  decompressed to 900 bytes, matches payload: true
```

900 bytes down to 73. The three pitfalls, all visible above:

**`Accept-Encoding` must be checked.** Gzipping unconditionally breaks any client that did
not ask.

**`Content-Length` must be deleted.** The handler's length describes the *uncompressed* body;
leave it and the client truncates or hangs. Here Go recomputed 73 because the response was
small enough to buffer.

**`Vary: Accept-Encoding` is mandatory.** Without it a cache serves gzipped bytes to a
client that cannot decode them.

This sketch is deliberately incomplete for production: it compresses tiny responses (where
gzip makes them *bigger*), compresses already-compressed types like JPEG, and its
`defer gz.Close()` runs after any `Flush`, breaking SSE. Use a maintained package or chi's
`middleware.Compress` (16.7), which handles the content-type allowlist and a minimum size.

## Conditional middleware per route

Because middleware is just function application, applying it to one route is trivial:

```go
mux := http.NewServeMux()
mux.Handle("GET /public", publicHandler)
mux.Handle("GET /private", Authenticate(privateHandler))
mux.Handle("POST /upload", Chain(uploadHandler,
  Authenticate,
  MaxBody(10<<20),
))

handler := Chain(mux, Recoverer, RequestID(gen), Logger(log))
```

Global middleware wraps the mux; per-route middleware wraps the handler before registration.
That is the stdlib answer, and its limitation is that there is no way to say "these twelve
routes share this middleware" without repeating yourself or building a second mux —
which is exactly the gap chi's `Group` and `Route` fill (16.7).

## Testing middleware in isolation

A middleware is a function from handler to handler, so test it with a stub handler and
`httptest.NewRecorder` — no server, no network:

```go
func TestRequestIDSetsHeader(t *testing.T) {
  var seen string
  stub := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    seen = RequestIDFrom(r.Context())
  })
  h := RequestID(func() string { return "fixed-id" })(stub)

  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))

  if seen != "fixed-id" {
    t.Errorf("handler saw %q, want %q", seen, "fixed-id")
  }
  if got := rec.Header().Get("X-Request-Id"); got != "fixed-id" {
    t.Errorf("header = %q, want %q", got, "fixed-id")
  }
}
```

The stub handler is both the assertion target and the proof that `next` was called. For a
short-circuiting middleware, assert the stub was **not** called. 16.11 covers the full
toolkit.

## Quick Reference

| Concept | Shape |
|---|---|
| middleware | `func(http.Handler) http.Handler` |
| configurable middleware | `func(opts) func(http.Handler) http.Handler` |
| call the next handler | `next.ServeHTTP(w, r)` |
| short-circuit | write a response and `return` without calling `next` |
| add a context value | `next.ServeHTTP(w, r.WithContext(ctx))` |
| chain | `Chain(h, A, B, C)` ≡ `A(B(C(h)))`; A is outermost |

| Middleware | Key detail |
|---|---|
| Recoverer | outermost; re-panic on `http.ErrAbortHandler`; cannot fix a started response |
| RequestID | context **and** response header; validate a client-supplied value |
| Logger | needs a `ResponseWriter` wrapper with `Unwrap`; log the pattern, not the path |
| Timeout | `http.TimeoutHandler` buffers and cannot kill the goroutine |
| CORS | outside auth; `Vary: Origin`; preflight needs `Access-Control-Request-Method` |
| BasicAuth | `subtle.ConstantTimeCompare`; `WWW-Authenticate` triggers the browser prompt |
| Gzip | check `Accept-Encoding`, delete `Content-Length`, add `Vary` |
| MaxBody | `http.MaxBytesHandler(h, n)` — already in the stdlib |

| Stdlib middleware | Signature |
|---|---|
| `http.StripPrefix` | `(prefix string, h Handler) Handler` |
| `http.TimeoutHandler` | `(h Handler, dt time.Duration, msg string) Handler` |
| `http.MaxBytesHandler` | `(h Handler, n int64) Handler` |
| `http.AllowQuerySemicolons` | `(h Handler) Handler` |

## Common Mistakes

**Wrapping `ResponseWriter` without an `Unwrap` method.** Every handler below loses
`Flusher` and `Hijacker`, so streaming and WebSocket upgrades break silently; add
`func (w *wrapper) Unwrap() http.ResponseWriter`.

**Defaulting the captured status to 0.** A handler that only calls `Write` never calls
`WriteHeader`, so your logs show `status=0` for every success; default to 200 inside `Write`.

**Putting `Recoverer` inside `Logger`.** A recovered panic then never gets logged with its
request, and a panic in the logger is unhandled; `Recoverer` goes outermost.

**Putting `CORS` inside auth.** The browser's unauthenticated `OPTIONS` preflight gets a 401
and the real request never happens; CORS must answer preflights first.

**Expecting `TimeoutHandler` to stop the handler.** Go cannot kill a goroutine, so the work
continues until the handler itself checks `r.Context().Done()`.

**Converting `http.ErrAbortHandler` into a 500.** That breaks the deliberate-abort contract
`ReverseProxy` depends on; check it with `errors.Is` and re-panic.

**Mutating `r` in place to pass data down.** Build a new request with
`r.WithContext(ctx)`; mutating the shared `*http.Request` is a race.

## Exercises

1. Build the full stack — Recoverer, RequestID, Logger, Timeout, CORS — with a `Chain`
   helper, and write one test that asserts the exact order by having each middleware append
   its name to a slice in the context. Verify the before/after interleaving matches this
   lesson's trace.
2. Write a `ResponseWriter` wrapper that also implements `http.Flusher` by delegating, and
   compare it against the `Unwrap` approach. Prove both let an SSE handler stream, then show
   which one still works when a *second* wrapper is added on top.
3. Write a rate-limiting middleware keyed by client IP using `golang.org/x/time/rate`
   ([12.4](../12-concurrency-patterns/04-rate-limiting.md)), returning 429 with a
   `Retry-After` header. Test it with 10 rapid requests and show the real boundary.
4. Make a recovery middleware that logs `string(debug.Stack())` and a request ID, and
   returns a 500 whose body contains *only* the request ID. Verify the internal detail never
   appears in the response for a panic containing a database connection string.
5. Write a middleware that measures and logs the time spent in the *handler* versus the time
   spent in middleware below it, by recording timestamps on both sides of
   `next.ServeHTTP`. Run it with a 3-deep chain and report the real numbers.

---

[← 16.5 JSON Requests and Responses](05-json-api.md) · [Course Index](../README.md) · [16.7 chi: Routing, Groups, and Middleware →](07-chi.md)
