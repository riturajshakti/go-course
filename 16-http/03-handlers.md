[← 16.2 ServeMux Routing: Methods and Wildcards](02-servemux-routing.md) · [Course Index](../README.md) · [16.4 Request Parsing: Path, Query, Headers, Body →](04-request-parsing.md)

# 16.3 — Handlers, HandlerFunc, and the Interface

## What You'll Learn

- `http.Handler` is a one-method interface, and `http.HandlerFunc` is a *named func type with a method on it* — the clearest example of 4.6 in the standard library
- The two idiomatic dependency-injection shapes: a **struct with `ServeHTTP`** and a **closure over dependencies**, both returned as `http.Handler` from a constructor
- `ResponseWriter`'s real dynamic type, the optional interfaces it may implement, and why `http.ResponseController` replaced asserting on them
- A handler panic sends the client **no response at all** — not a 500 — which is the entire motivation for recovery middleware

## Coming From Another Language

| Other | Go |
|---|---|
| Express: a handler is just a function | a handler is an **interface**; a function becomes one by conversion |
| Express `app.use(injectDeps)` / DI containers | dependencies are **struct fields** or captured closure variables |
| Python class-based views (`View.as_view()`) | a struct with a `ServeHTTP` method — the same idea, no framework needed |
| Rust `tower::Service` | `http.Handler`, minus the async and the generics |
| Express: an uncaught throw gives a 500 by default | an uncaught panic gives **no response**; you must add middleware |
| `res.statusCode` is readable | `ResponseWriter` is write-only; you cannot read the status back |

## The interface, and the func adapter

Two declarations, and they are the whole design:

```go
type Handler interface {
  ServeHTTP(ResponseWriter, *Request)
}

type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
  f(w, r)
}
```

`HandlerFunc` is a **named function type** with a method defined on it
([4.6](../04-types-structs/06-named-types.md)). Its `ServeHTTP` calls the function it *is*.
That is the entire adapter: it exists only so an ordinary function satisfies a one-method
interface ([5.1](../05-interfaces/01-implicit-satisfaction.md)). Nothing is wrapped,
allocated, or boxed beyond the interface value itself.

This is why `http.HandlerFunc(myFunc)` is a **type conversion**, not a function call — and
why you can define your own `ServeHTTP` on any type at all.

## Two ways to carry dependencies

Every real handler needs a logger, a store, a config. Go has no DI container; it has struct
fields and closures, and both are idiomatic. Here they are side by side, each returned from
a constructor as `http.Handler`.

```go
package main

import (
  "fmt"
  "io"
  "log/slog"
  "net/http"
  "net/http/httptest"
  "os"
  "strings"
  "sync"
)

type Store struct {
  mu    sync.RWMutex
  items map[string]string
}

func (s *Store) Get(k string) (string, bool) {
  s.mu.RLock()
  defer s.mu.RUnlock()
  v, ok := s.items[k]
  return v, ok
}

type ItemHandler struct {
  store *Store
  log   *slog.Logger
}

func (h *ItemHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  id := r.PathValue("id")
  h.log.Info("lookup", "id", id)
  v, ok := h.store.Get(id)
  if !ok {
    http.Error(w, "not found", http.StatusNotFound)
    return
  }
  fmt.Fprintln(w, v)
}

func NewItemHandler(s *Store, log *slog.Logger) http.Handler {
  return &ItemHandler{store: s, log: log}
}

func closureHandler(s *Store) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    v, ok := s.Get(r.PathValue("id"))
    fmt.Fprintf(w, "closure: %q ok=%v\n", v, ok)
  })
}

var log = slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
  ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
    if a.Key == slog.TimeKey {
      return slog.Attr{}
    }
    return a
  },
}))

func main() {
  store := &Store{items: map[string]string{"a": "apple", "b": "banana"}}

  mux := http.NewServeMux()
  mux.Handle("GET /struct/{id}", NewItemHandler(store, log))
  mux.Handle("GET /closure/{id}", closureHandler(store))

  srv := httptest.NewServer(mux)
  defer srv.Close()

  for _, p := range []string{"/struct/a", "/struct/zzz", "/closure/b"} {
    resp, _ := http.Get(srv.URL + p)
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-14s %-16s %q\n", p, resp.Status, strings.TrimSpace(string(b)))
  }

  var h http.Handler = http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {})
  fmt.Printf("\nHandlerFunc dynamic type: %T\n", h)
  fmt.Printf("*ItemHandler implements http.Handler: %v\n", func() bool {
    var _ http.Handler = (*ItemHandler)(nil)
    return true
  }())
}
```

The `slog` time attribute is stripped via `ReplaceAttr` so the output is reproducible; in
production you keep it.

Output:

```
level=INFO msg=lookup id=a
/struct/a      200 OK           "apple"
level=INFO msg=lookup id=zzz
/struct/zzz    404 Not Found    "not found"
/closure/b     200 OK           "closure: \"banana\" ok=true"

HandlerFunc dynamic type: http.HandlerFunc
*ItemHandler implements http.Handler: true
```

The server log lines interleave *before* their client lines because the server goroutine
runs first — a real ordering, not an artifact.

**When to use which.** The struct form scales: it gives each dependency a name, lets you
add helper methods (`h.respond`, `h.decode`), groups several related routes on one receiver,
and is trivially testable by constructing the struct with fakes
([14.8](../14-testing/08-test-doubles.md)). The closure form is shorter and perfect for one
route with one dependency. Both hide the dependencies behind `http.Handler`, so the mux —
and your tests — need to know nothing about them.

Idiomatic: `func NewXHandler(deps...) http.Handler`. Returning the *interface* rather than
the concrete type keeps callers from depending on your internals and makes the
constructor-plus-middleware composition in 16.6 uniform.

Note `*ItemHandler` is the receiver, not `ItemHandler`, so it is the *pointer* that
satisfies `http.Handler` ([4.4](../04-types-structs/04-methods.md)). Returning
`&ItemHandler{...}` is what makes `var _ http.Handler = (*ItemHandler)(nil)` the right
compile-time assertion.

## Handle vs HandleFunc, package-level vs method

Four registration functions that people routinely confuse:

| Call | Takes | Registers on |
|---|---|---|
| `http.Handle(pat, h)` | an `http.Handler` | `DefaultServeMux` — avoid |
| `http.HandleFunc(pat, f)` | a plain `func(w, r)` | `DefaultServeMux` — avoid |
| `mux.Handle(pat, h)` | an `http.Handler` | your mux |
| `mux.HandleFunc(pat, f)` | a plain `func(w, r)` | your mux |

`HandleFunc` is literally `Handle(pat, HandlerFunc(f))`. Use `Handle` when you have a
handler value (a struct, a middleware-wrapped chain, `http.FileServer`) and `HandleFunc`
when you have a bare function. Always the method form, never the package form (16.1).

## Composition: handlers wrapping handlers

Because a handler is an interface value, a function that takes one and returns another
composes trivially. That is the middleware shape in full, and 16.6 is entirely about it:

```go
func withHeader(key, val string, next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set(key, val)
    next.ServeHTTP(w, r)
  })
}
```

The standard library ships several wrappers of exactly this shape already —
`http.StripPrefix`, `http.TimeoutHandler`, `http.MaxBytesHandler`, `http.RedirectHandler`.
They are not a framework feature; they are just functions returning `http.Handler`.

## ResponseWriter's real type and its optional interfaces

`http.ResponseWriter` has three methods, but the concrete value behind it has more. The
convention in `net/http` is **optional interfaces**: type-assert to see whether a capability
is available. Here is what is really there, on both a live server and a recorder.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "time"
)

func probe(w http.ResponseWriter, r *http.Request) {
  fmt.Printf("dynamic type: %T\n", w)
  _, flusher := w.(http.Flusher)
  _, hijacker := w.(http.Hijacker)
  _, readerFrom := w.(io.ReaderFrom)
  _, pusher := w.(http.Pusher)
  fmt.Printf("  http.Flusher:  %v\n", flusher)
  fmt.Printf("  http.Hijacker: %v\n", hijacker)
  fmt.Printf("  io.ReaderFrom: %v\n", readerFrom)
  fmt.Printf("  http.Pusher:   %v\n", pusher)

  rc := http.NewResponseController(w)
  fmt.Printf("  rc.Flush() err:            %v\n", rc.Flush())
  fmt.Printf("  rc.SetWriteDeadline err:   %v\n", rc.SetWriteDeadline(time.Time{}))
  fmt.Printf("  rc.EnableFullDuplex err:   %v\n", rc.EnableFullDuplex())
  fmt.Fprintln(w, "probed")
}

func main() {
  srv := httptest.NewServer(http.HandlerFunc(probe))
  defer srv.Close()
  resp, _ := http.Get(srv.URL)
  b, _ := io.ReadAll(resp.Body)
  resp.Body.Close()
  fmt.Printf("client got %s %q\n", resp.Status, b)

  fmt.Println("\n--- through httptest.NewRecorder instead ---")
  rec := httptest.NewRecorder()
  probe(rec, httptest.NewRequest("GET", "/", nil))
  fmt.Printf("recorder code=%d flushed=%v body=%q\n", rec.Code, rec.Flushed, rec.Body.String())
}
```

Output:

```
dynamic type: *http.response
  http.Flusher:  true
  http.Hijacker: true
  io.ReaderFrom: true
  http.Pusher:   false
  rc.Flush() err:            <nil>
  rc.SetWriteDeadline err:   <nil>
  rc.EnableFullDuplex err:   <nil>
client got 200 OK "probed\n"

--- through httptest.NewRecorder instead ---
dynamic type: *httptest.ResponseRecorder
  http.Flusher:  true
  http.Hijacker: false
  io.ReaderFrom: false
  http.Pusher:   false
  rc.Flush() err:            <nil>
  rc.SetWriteDeadline err:   feature not supported
  rc.EnableFullDuplex err:   feature not supported
recorder code=200 flushed=true body="probed\n"
```

The dynamic type is the unexported `*http.response` on a real server and
`*httptest.ResponseRecorder` in a direct test — and they support *different* optional
interfaces. That difference is exactly why type-asserting is fragile.

| Optional interface | Method | Purpose |
|---|---|---|
| `http.Flusher` | `Flush()` | push buffered bytes to the client now (SSE, streaming) |
| `http.Hijacker` | `Hijack() (net.Conn, *bufio.ReadWriter, error)` | take over the raw TCP connection (WebSocket upgrade) |
| `io.ReaderFrom` | `ReadFrom(io.Reader)` | zero-copy `sendfile` path used by `io.Copy` |
| `http.Pusher` | `Push(...)` | HTTP/2 server push; `false` here because this is HTTP/1.1 |
| `http.CloseNotifier` | `CloseNotify()` | **deprecated** — use `r.Context()` (16.12) |

`http.NewResponseController(w)` (Go 1.20+) is the modern answer. It does the assertions for
you, **unwraps** any middleware wrapper that implements `Unwrap() http.ResponseWriter`, and
returns a plain `error` — `"feature not supported"` — instead of panicking or silently doing
nothing when the capability is absent.

| `*ResponseController` method | Replaces |
|---|---|
| `Flush() error` | `w.(http.Flusher).Flush()` |
| `Hijack() (net.Conn, *bufio.ReadWriter, error)` | `w.(http.Hijacker).Hijack()` |
| `SetReadDeadline(time.Time) error` | nothing — previously impossible per-request |
| `SetWriteDeadline(time.Time) error` | nothing — the only way past `Server.WriteTimeout` |
| `EnableFullDuplex() error` | nothing — allows reading the body while writing the response |

`SetWriteDeadline` is the important one. `Server.WriteTimeout` (16.1) is a flat per-request
cap; a streaming endpoint needs to extend it per chunk, and `ResponseController` is the only
way to do that. Passing the zero `time.Time` removes the deadline entirely.

Idiomatic: use `http.NewResponseController(w)`. Reach for a raw type assertion only for
`http.Pusher`, which the controller does not cover.

## Streaming a response with Flush

A complete server-sent-events handler, driven by a real client that reads the stream as it
arrives. Flushing is what turns a buffered response into a chunked one.

```go
package main

import (
  "bufio"
  "fmt"
  "net/http"
  "net/http/httptest"
  "time"
)

func stream(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "text/event-stream")
  w.Header().Set("Cache-Control", "no-cache")
  rc := http.NewResponseController(w)

  for i := 1; i <= 3; i++ {
    if err := rc.SetWriteDeadline(time.Now().Add(5 * time.Second)); err != nil {
      fmt.Println("server: deadline unsupported:", err)
    }
    if _, err := fmt.Fprintf(w, "data: tick %d\n\n", i); err != nil {
      fmt.Println("server: write failed, client gone:", err)
      return
    }
    if err := rc.Flush(); err != nil {
      fmt.Println("server: flush failed:", err)
      return
    }
  }
}

func main() {
  srv := httptest.NewServer(http.HandlerFunc(stream))
  defer srv.Close()

  resp, err := http.Get(srv.URL)
  if err != nil {
    panic(err)
  }
  defer resp.Body.Close()
  fmt.Println("Transfer-Encoding:", resp.TransferEncoding)
  fmt.Println("Content-Length:", resp.ContentLength)

  sc := bufio.NewScanner(resp.Body)
  for sc.Scan() {
    if line := sc.Text(); line != "" {
      fmt.Printf("client received: %q\n", line)
    }
  }
}
```

Output:

```
Transfer-Encoding: [chunked]
Content-Length: -1
client received: "data: tick 1"
client received: "data: tick 2"
client received: "data: tick 3"
```

`Transfer-Encoding: chunked` and `Content-Length: -1` (meaning "unknown") are the
observable consequence of flushing: Go can no longer buffer the whole body to compute a
length, so it streams. Without the `Flush` calls the client would see one `Content-Length`
response arriving all at once.

This is also the one case where you **must** check the error from `Write`: in a long-lived
stream the client disconnecting is the normal termination condition, and the write error is
how you learn about it. (`r.Context()` cancellation is the other way — 16.12.)

## You cannot read the status back

`ResponseWriter` has no `Status()` method. A logging middleware that wants to record the
status code has to intercept `WriteHeader`:

```go
type statusRecorder struct {
  http.ResponseWriter
  status int
  bytes  int
}

func (sr *statusRecorder) WriteHeader(code int) {
  sr.status = code
  sr.ResponseWriter.WriteHeader(code)
}

func (sr *statusRecorder) Write(b []byte) (int, error) {
  if sr.status == 0 {
    sr.status = http.StatusOK
  }
  n, err := sr.ResponseWriter.Write(b)
  sr.bytes += n
  return n, err
}

func (sr *statusRecorder) Unwrap() http.ResponseWriter { return sr.ResponseWriter }
```

Embedding `http.ResponseWriter` promotes `Header()` for free
([4.5](../04-types-structs/05-embedding.md)) and overrides only the two methods that matter.
The `status == 0` branch in `Write` is essential — an implicit 200 never calls
`WriteHeader`, so without it you would log `0`.

The `Unwrap` method is what lets `http.NewResponseController` see through your wrapper to
the real `Flusher` and `Hijacker` underneath. Embedding alone does *not* preserve them: the
wrapper's method set is `ResponseWriter`'s three methods plus your overrides, so a
`w.(http.Flusher)` assertion against the wrapper **fails**, and a streaming handler behind
your middleware silently stops streaming. 16.6 builds this out properly.

## A handler panic sends no response

Go recovers a handler panic so the *process* survives, but the request is simply abandoned.
This is nothing like Express giving you a 500.

```go
package main

import (
  "fmt"
  "io"
  "log"
  "net/http"
  "net/http/httptest"
  "os"
)

func panicBefore(w http.ResponseWriter, r *http.Request) {
  panic("boom before any write")
}

func panicAfter(w http.ResponseWriter, r *http.Request) {
  w.WriteHeader(http.StatusOK)
  io.WriteString(w, `{"partial":true`)
  panic("boom after a partial write")
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/panic-before", panicBefore)
  mux.HandleFunc("/panic-after", panicAfter)
  mux.HandleFunc("/after", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "server still alive")
  })

  srv := httptest.NewServer(mux)
  srv.Config.ErrorLog = log.New(os.Stdout, "SERVERLOG: ", 0)
  defer srv.Close()

  for _, p := range []string{"/panic-before", "/panic-after", "/after"} {
    resp, err := http.Get(srv.URL + p)
    if err != nil {
      fmt.Printf("%-15s client error: %v\n", p, err)
      continue
    }
    b, readErr := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-15s %s body=%q readErr=%v\n", p, resp.Status, b, readErr)
  }
}
```

This program panics twice on purpose. Output, with the two stack traces removed (each is
about 18 lines of `net/http` internals) and the ports being kernel-assigned and different
every run:

```
SERVERLOG: http: panic serving 127.0.0.1:56597: boom before any write
/panic-before   client error: Get "http://127.0.0.1:56596/panic-before": EOF
SERVERLOG: http: panic serving 127.0.0.1:56598: boom after a partial write
/panic-after    client error: Get "http://127.0.0.1:56596/panic-after": EOF
/after          200 OK body="server still alive" readErr=<nil>
```

The first line of each stack trace, verbatim, is:

```
SERVERLOG: http: panic serving 127.0.0.1:56597: boom before any write
goroutine 22 [running]:
net/http.(*conn).serve.func1()
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/net/http/server.go:1939 +0xac
panic({0x100b11590?, 0x100ab80e8?})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/panic.go:859 +0x120
main.panicBefore({0x5c50a7d24100?, 0x5c50a7d2a104?}, 0x5c50a7e81a98?)
	/tmp/scratch/main.go:13 +0x2c
```

Read the client lines again, because this is the part people get wrong. Both panicking
requests produced **`EOF`** — not a 500, not a 502, *no HTTP response at all*. Go's
recovery closes the connection. Even `/panic-after`, which had already written a `200 OK`
status line and a partial JSON body, gave the client an `EOF`: a truncated response is
indistinguishable from a network failure, so `io.ReadAll` never even ran.

Three conclusions:

1. **The process survives.** `/after` returned 200, so one panicking handler does not take
   the server down. Go recovers at the connection level.
2. **The client gets nothing useful.** No status code means every client — browser, mobile
   app, service mesh, your own Go client — sees a transport error. Your monitoring sees
   connection resets, not 5xx, so your error-rate dashboards miss it entirely.
3. **A partial write cannot be undone.** Once `/panic-after` wrote its status, no middleware
   could have turned it into a clean 500 either. Recovery middleware can only save you if
   nothing has been written yet.

That is the case for recovery middleware, built in full in 16.6: catch the panic, log it
with the stack and the request ID, and write a `500` with a safe body — which works
precisely because it runs before anything else has written.

`http.ErrAbortHandler` is the one panic value you should treat specially. Panicking with it
aborts the response *without* logging a stack trace:

```go
mux.HandleFunc("/abort", func(w http.ResponseWriter, r *http.Request) {
  panic(http.ErrAbortHandler)
})
```

Output (the port varies per run):

```
/abort -> err=Get "http://127.0.0.1:56607/abort": EOF resp=false
```

No `SERVERLOG:` line at all — compare with the two panics above. It is how `http.ReverseProxy`
abandons a response it has decided is unsalvageable. Your recovery middleware must
**re-panic** on `http.ErrAbortHandler` rather than converting it to a 500, or you break that
contract.

## http.Error and http.ServeContent

`http.Error(w, msg, code)` is the quick error path: plain text, `nosniff`, status, message.
`http.ServeContent` is the generated-content counterpart to `ServeFile`, and it gives you
`Range` and conditional-request handling on data that never touched a disk.

```go
mux.HandleFunc("/content", func(w http.ResponseWriter, r *http.Request) {
  body := strings.NewReader("generated report line 1\ngenerated report line 2\n")
  modtime := time.Date(2024, 3, 1, 12, 0, 0, 0, time.UTC)
  http.ServeContent(w, r, "report.txt", modtime, body)
})
```

Driven with a plain `GET`, then a `Range` header, then an `If-Modified-Since`:

Output:

```
/content -> 200 OK ct="text/plain; charset=utf-8" len=48 lastmod="Fri, 01 Mar 2024 12:00:00 GMT" etag=""
  body="generated report line 1\ngenerated report line 2\n"
  with Range: 206 Partial Content cr="bytes 10-21/48" body="report line "
  with If-Modified-Since: 304 Not Modified
```

A fixed `modtime` makes that reproducible. The name argument is used only to guess
`Content-Type` from the extension — the file need not exist. You get `206 Partial Content`
with a correct `Content-Range`, and `304 Not Modified` for free, from an `io.ReadSeeker` over
a string. Writing that by hand is many dozens of lines of RFC 9110 range parsing.

## Quick Reference

| Type / function | Signature | Note |
|---|---|---|
| `http.Handler` | `interface { ServeHTTP(ResponseWriter, *Request) }` | the only abstraction in the package |
| `http.HandlerFunc` | `func(ResponseWriter, *Request)` | named func type with `ServeHTTP`; conversion, not a call |
| `http.NewResponseController` | `(w ResponseWriter) *ResponseController` | 1.20+; unwraps middleware, returns errors |
| `http.Error` | `(w, msg string, code int)` | text + `nosniff` + status |
| `http.ServeContent` | `(w, r, name string, modtime time.Time, c io.ReadSeeker)` | `Range` + `If-Modified-Since` for free |
| `http.ErrAbortHandler` | `error` | panic with it to abort silently; re-panic it in recovery middleware |
| `http.TimeoutHandler` | `(h Handler, dt time.Duration, msg string) Handler` | stdlib middleware (16.6) |
| `http.MaxBytesHandler` | `(h Handler, n int64) Handler` | body cap as middleware (16.4) |

| Dependency-injection shape | Use when |
|---|---|
| struct + `ServeHTTP` method | several routes share deps; you want helper methods; **default for a service** |
| closure returning `http.HandlerFunc` | one route, one or two deps |
| package-level `var db *sql.DB` | never — untestable, and a race waiting to happen |

| Panic in a handler | Result |
|---|---|
| any value | stack trace to `Server.ErrorLog`, connection closed, client sees a transport error |
| `http.ErrAbortHandler` | connection closed, **no** log line |
| caught by recovery middleware before any write | you can send a real 500 (16.6) |
| caught after a partial write | unrecoverable; the response is already truncated |

## Common Mistakes

**Expecting a panic to produce a 500.** Go closes the connection and the client sees an
`EOF`, so your dashboards show resets rather than 5xx — add recovery middleware (16.6).

**Converting `http.ErrAbortHandler` into a 500 in recovery middleware.** That breaks the
deliberate-abort contract `ReverseProxy` relies on; check for it with `errors.Is` and
re-panic.

**Wrapping `ResponseWriter` by embedding and nothing else.** The wrapper no longer satisfies
`http.Flusher` or `http.Hijacker`, so streaming and WebSocket upgrades silently break behind
your middleware; add an `Unwrap() http.ResponseWriter` method.

**Forgetting the implicit 200 in a status-capturing wrapper.** A handler that only calls
`Write` never calls `WriteHeader`, so your recorded status stays `0` unless `Write` defaults
it to 200.

**Storing dependencies in package-level variables.** Two tests in the same package then
fight over them, and nothing is injectable; put them in the handler struct.

**Using a value receiver for a handler with a mutex field.** `func (h ItemHandler) ServeHTTP`
copies the struct — and the `sync.RWMutex` with it — on every request, so the locking
protects nothing; use a pointer receiver.

## Exercises

1. Implement `http.Handler` on a struct holding a `*slog.Logger` and a `map[string]int`
   guarded by a `sync.RWMutex`. Give it three routes via three methods on the same receiver
   and a `Routes() http.Handler` method that returns a configured mux. Verify under `-race`
   with 100 concurrent requests.
2. Write the `statusRecorder` from this lesson and prove with a test that (a) it records 404
   from an explicit `WriteHeader`, (b) it records 200 from a handler that only calls
   `Write`, and (c) `http.NewResponseController` can still `Flush` through it *only* when
   `Unwrap` is present — delete the method and watch it fail.
3. Build a handler whose `ResponseWriter` you assert to `http.Hijacker`, take over the
   connection, and write a raw HTTP/1.1 response by hand with `bufio.Writer`. Confirm with a
   client that the status and body arrive.
4. Write an SSE handler that streams one event per 100ms for 2 seconds, and a client that
   cancels after 500ms. Capture the exact error your `Write` or `Flush` returns when the
   client goes away, and compare it to what `r.Context().Err()` says at that moment.
5. Write a recovery middleware that handles `http.ErrAbortHandler` correctly. Prove with
   three routes — a normal panic, an `ErrAbortHandler` panic, and a panic after a partial
   write — that you get a clean 500, a silent abort, and a truncated response respectively.

---

[← 16.2 ServeMux Routing: Methods and Wildcards](02-servemux-routing.md) · [Course Index](../README.md) · [16.4 Request Parsing: Path, Query, Headers, Body →](04-request-parsing.md)
