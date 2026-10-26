[← 15.8 go generate and Workspaces](../15-tooling/08-generate-workspaces.md) · [Course Index](../README.md) · [16.2 ServeMux Routing: Methods and Wildcards →](02-servemux-routing.md)

# 16.1 — Your First net/http Server

## What You'll Learn

- Every request runs in its **own goroutine** — the single biggest difference from Node, and the source of your first data race
- `WriteHeader` must come before any `Write`, and `Header().Set` before `WriteHeader` — both rules fail *silently* or with a log line, never with an error
- Why you should always construct an `http.Server` instead of calling `ListenAndServe` with timeouts left at zero
- Why `http.HandleFunc` and `DefaultServeMux` are a production hazard

## Coming From Another Language

| Other | Go |
|---|---|
| Node: one event loop, your handler never runs concurrently with itself | one goroutine **per request**; shared state needs a mutex from day one |
| Express `res.status(201).json(x)` — order enforced by the API | `w.WriteHeader(201)` then `w.Write(...)`; wrong order is a *log line*, not an error |
| Express/FastAPI ship a framework; routing is the framework's job | `net/http` is the framework; `chi` adds routing sugar, not a runtime (16.7) |
| `app.listen(8080)` with sane library defaults | `http.ListenAndServe` has **no timeouts at all** — you must set them |
| Python WSGI/ASGI: returning a value *is* the response | you *write* to a `ResponseWriter`; there is no return value |
| Flask `@app.route` global decorator registry | `http.HandleFunc` is the same global registry — and that is why you avoid it |

## The minimal server

Three lines of real work. `http.HandleFunc` registers a function against the package-global
mux, `ListenAndServe` binds the port and blocks forever.

```go
package main

import (
  "fmt"
  "log"
  "net/http"
)

func main() {
  http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello from %s %s\n", r.Method, r.URL.Path)
  })
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

`ListenAndServe` only ever returns an error, so wrapping it in `log.Fatal` is the idiom.
The `nil` second argument means "use `DefaultServeMux`" — which we will stop doing by the
end of this lesson.

A blocking server cannot be run inline in a course, so from here on every example drives
the *same handler* through `net/http/httptest`, which starts a real server on a real
loopback port and makes real requests to it. Nothing below is simulated.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
)

func hello(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "hello from %s %s\n", r.Method, r.URL.Path)
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/hello", hello)

  srv := httptest.NewServer(mux)
  defer srv.Close()

  resp, err := http.Get(srv.URL + "/hello")
  if err != nil {
    panic(err)
  }
  body, _ := io.ReadAll(resp.Body)
  resp.Body.Close()

  fmt.Println("status:", resp.Status)
  fmt.Println("content-type:", resp.Header.Get("Content-Type"))
  fmt.Println("content-length:", resp.Header.Get("Content-Length"))
  fmt.Printf("body: %q\n", body)
}
```

Output:

```
status: 200 OK
content-type: text/plain; charset=utf-8
content-length: 22
body: "hello from GET /hello\n"
```

Nobody set `Content-Type` or `Content-Length`. Go sniffed the first 512 bytes of the body
to guess the type, and buffered the (small) response so it could compute the length. Both
are conveniences you will override deliberately in 16.5.

`srv.URL` is `http://127.0.0.1:<port>` with a kernel-assigned port, so it differs on every
run — never hardcode it, always read it from the server.

## ResponseWriter and *http.Request

The handler signature is the whole API surface:

```go
func(w http.ResponseWriter, r *http.Request)
```

`*http.Request` is a struct you read. `http.ResponseWriter` is an **interface** you write:

```go
type ResponseWriter interface {
  Header() Header
  Write([]byte) (int, error)
  WriteHeader(statusCode int)
}
```

Three methods, and the order you call them in is load-bearing. Note what is *missing*: no
`Status()`, no `Body()`. A `ResponseWriter` is write-only — you cannot ask it what status
you set. That omission drives the wrapper pattern in 16.6.

## Every request runs in its own goroutine

`http.Server` accepts a connection and calls your handler on a **new goroutine**. Two
requests arriving together run your handler *simultaneously*, on different cores. If you
came from Node, where a handler is never concurrent with itself, this is the single most
important sentence in the module.

The consequence: any state a handler touches that outlives the request is shared mutable
state and needs synchronization ([11.2](../11-sync-context/02-mutex.md)).

```go
package main

import (
  "fmt"
  "net/http"
  "net/http/httptest"
  "sync"
)

func main() {
  var counter int
  racy := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    counter++
    fmt.Fprintln(w, counter)
  })

  srv := httptest.NewServer(racy)
  defer srv.Close()

  var wg sync.WaitGroup
  for range 200 {
    wg.Go(func() {
      resp, err := http.Get(srv.URL)
      if err != nil {
        return
      }
      resp.Body.Close()
    })
  }
  wg.Wait()
  fmt.Println("200 requests sent, counter =", counter)
}
```

This program has a data race on purpose. Under `go run -race .` the detector fires
(traceback trimmed, scratch path normalized):

```
==================
WARNING: DATA RACE
Read at 0x00c00019e1c8 by goroutine 280:
  main.main.func1()
      /tmp/scratch/main.go:13 +0x34
  net/http.HandlerFunc.ServeHTTP()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/net/http/server.go:2338 +0x48
  net/http.serverHandler.ServeHTTP()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/net/http/server.go:3413 +0x268
  net/http.(*conn).serve()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/net/http/server.go:2137 +0xa84

Previous write at 0x00c00019e1c8 by goroutine 277:
  main.main.func1()
      /tmp/scratch/main.go:13 +0x44
  net/http.HandlerFunc.ServeHTTP()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/net/http/server.go:2338 +0x48
```

Line 13 is `counter++`. The goroutine numbers and the address vary per run; the stack
shape — your handler called from `net/http.(*conn).serve`, once per connection — does not.
`counter` also ends below 200 because increments are lost.

The fix is a mutex or an atomic, exactly as in module 11:

```go
var mu sync.Mutex
var counter int
// in the handler:
mu.Lock()
counter++
n := counter
mu.Unlock()
fmt.Fprintln(w, n)
```

Idiomatic: give the handler its dependencies as struct fields and make each one
concurrency-safe, rather than reaching for package-level variables (16.3).

## Writing a response: Write, io.WriteString, fmt.Fprintf

`ResponseWriter` is an `io.Writer`, so everything in the `io`/`fmt` toolbox works on it.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
)

func main() {
  mux := http.NewServeMux()

  mux.HandleFunc("/bytes", func(w http.ResponseWriter, r *http.Request) {
    n, err := w.Write([]byte("raw bytes"))
    fmt.Println("Write returned:", n, err)
  })
  mux.HandleFunc("/string", func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "io.WriteString")
  })
  mux.HandleFunc("/printf", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "formatted %d %s", 42, "args")
  })
  mux.HandleFunc("/empty", func(w http.ResponseWriter, r *http.Request) {})
  mux.HandleFunc("/nocontent", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusNoContent)
  })
  mux.HandleFunc("/json-sniff", func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, `{"ok":true}`)
  })

  srv := httptest.NewServer(mux)
  defer srv.Close()

  for _, p := range []string{"/bytes", "/string", "/printf", "/empty", "/nocontent", "/json-sniff"} {
    resp, err := http.Get(srv.URL + p)
    if err != nil {
      panic(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-12s %s len=%-2d ct=%-30q body=%q\n",
      p, resp.Status, len(b), resp.Header.Get("Content-Type"), b)
  }
}
```

Output:

```
Write returned: 9 <nil>
/bytes       200 OK len=9  ct="text/plain; charset=utf-8"    body="raw bytes"
/string      200 OK len=14 ct="text/plain; charset=utf-8"    body="io.WriteString"
/printf      200 OK len=17 ct="text/plain; charset=utf-8"    body="formatted 42 args"
/empty       200 OK len=0  ct=""                             body=""
/nocontent   204 No Content len=0  ct=""                             body=""
/json-sniff  200 OK len=11 ct="text/plain; charset=utf-8"    body="{\"ok\":true}"
```

Four things worth naming.

**`/empty` returns 200, not an error.** A handler that writes nothing still produces a
valid `200 OK` with an empty body. The status is written implicitly the moment the handler
returns, if it was not written already.

**`Write` returns `(int, error)`.** Almost nobody checks it, and that is defensible: the
only realistic error is the client having hung up, at which point there is nothing useful
to do but return. Do not ignore it in a *streaming* handler, though (16.3).

**`/json-sniff` got `text/plain`.** Content sniffing looked at `{"ok":true}` and did not
recognize it as JSON. Serving an API means setting `Content-Type` yourself, every time.

**`/nocontent` sent no Content-Type.** Go suppresses the sniffing and the body for 204 and
304, per the spec.

## WriteHeader, implicit 200, and the ordering rules

Three rules, all of which fail quietly:

1. `WriteHeader(code)` sets the status. Call it **once**, **before** any `Write`.
2. The first `Write` implicitly calls `WriteHeader(200)` if you have not.
3. Mutating `Header()` after the headers are sent is a **silent no-op**.

Here are all three breaking, in one real run. `httptest.Server` exposes its underlying
`*http.Server` as `srv.Config`, so we can point the server's `ErrorLog` at stdout and
capture the log it emits.

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

func writeFirst(w http.ResponseWriter, r *http.Request) {
  io.WriteString(w, "body first\n")
  w.WriteHeader(http.StatusTeapot)
  w.Header().Set("X-Late", "ignored")
  io.WriteString(w, "more body\n")
}

func correct(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "text/plain; charset=utf-8")
  w.Header().Set("X-Early", "kept")
  w.WriteHeader(http.StatusCreated)
  io.WriteString(w, "created\n")
}

func main() {
  log.SetFlags(0)
  log.SetOutput(os.Stdout)

  mux := http.NewServeMux()
  mux.HandleFunc("/wrong", writeFirst)
  mux.HandleFunc("/right", correct)

  srv := httptest.NewServer(mux)
  srv.Config.ErrorLog = log.New(os.Stdout, "server log: ", 0)
  defer srv.Close()

  for _, path := range []string{"/wrong", "/right"} {
    resp, err := http.Get(srv.URL + path)
    if err != nil {
      panic(err)
    }
    body, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%s -> %s  X-Early=%q X-Late=%q\n", path, resp.Status,
      resp.Header.Get("X-Early"), resp.Header.Get("X-Late"))
    fmt.Printf("  body: %q\n", body)
  }
}
```

Output:

```
server log: http: superfluous response.WriteHeader call from main.writeFirst (main.go:14)
/wrong -> 200 OK  X-Early="" X-Late=""
  body: "body first\nmore body\n"
/right -> 201 Created  X-Early="kept" X-Late=""
  body: "created\n"
```

That one log line is the error message you will meet most often in your first month of Go
HTTP work: **`http: superfluous response.WriteHeader call`**. It names the function and
the line, which is the only reason it is tolerable.

Read the `/wrong` line carefully. The handler *asked* for 418. The client got **200**. The
`WriteHeader(418)` on line 14 came after a `Write`, so the status was already on the wire
and the call was discarded — with a log line, not an error, not a panic. `X-Late` is gone
for the same reason: `Header()` still returns a live, mutable map after the headers are
flushed, so setting a key succeeds and changes nothing anyone will ever see. Both body
writes *did* land, so the response is a plausible-looking 200 that silently lost its
status and its header.

The `/right` line is the shape to internalize: **headers, then status, then body**.

Idiomatic: set every header you care about, call `WriteHeader` exactly once, then write.
If a handler can fail, decide the status *before* writing a single byte.

## Always construct an http.Server

`http.ListenAndServe(addr, handler)` is a convenience wrapper that builds an
`&http.Server{Addr: addr, Handler: handler}` — with every timeout at zero.

```go
package main

import (
  "fmt"
  "net/http"
  "time"
)

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "ok")
  })

  srv := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       15 * time.Second,
    WriteTimeout:      15 * time.Second,
    IdleTimeout:       60 * time.Second,
    MaxHeaderBytes:    1 << 20,
  }

  def := &http.Server{}
  fmt.Println("defaults on a zero-value http.Server:")
  fmt.Println("  ReadHeaderTimeout:", def.ReadHeaderTimeout)
  fmt.Println("  ReadTimeout:      ", def.ReadTimeout)
  fmt.Println("  WriteTimeout:     ", def.WriteTimeout)
  fmt.Println("  IdleTimeout:      ", def.IdleTimeout)
  fmt.Println("  MaxHeaderBytes:   ", def.MaxHeaderBytes, "(0 means DefaultMaxHeaderBytes)")
  fmt.Println("  http.DefaultMaxHeaderBytes:", http.DefaultMaxHeaderBytes)
  fmt.Println()
  fmt.Println("configured server:")
  fmt.Println("  ReadHeaderTimeout:", srv.ReadHeaderTimeout)
  fmt.Println("  IdleTimeout:      ", srv.IdleTimeout)
  fmt.Println("  MaxHeaderBytes:   ", srv.MaxHeaderBytes)
}
```

Output:

```
defaults on a zero-value http.Server:
  ReadHeaderTimeout: 0s
  ReadTimeout:       0s
  WriteTimeout:      0s
  IdleTimeout:       0s
  MaxHeaderBytes:    0 (0 means DefaultMaxHeaderBytes)
  http.DefaultMaxHeaderBytes: 1048576

configured server:
  ReadHeaderTimeout: 5s
  IdleTimeout:       1m0s
  MaxHeaderBytes:    1048576
```

Zero means *no deadline*. Every one of those zeros is a denial-of-service vector:

| Field | What it bounds | The attack when it is 0 |
|---|---|---|
| `ReadHeaderTimeout` | accept → end of request headers | **Slowloris**: open 10k connections, dribble one header byte per minute, never finish. Each holds a goroutine and an fd forever. |
| `ReadTimeout` | accept → end of request **body** | Slow-POST: advertise `Content-Length: 10000000`, send 1 byte/s. Covers headers too. |
| `WriteTimeout` | end of headers read → end of response write | Slow-read: request a large response, then stop reading. Your write blocks indefinitely. |
| `IdleTimeout` | between requests on a keep-alive connection | Idle keep-alives accumulate without bound, each pinning a goroutine. Falls back to `ReadTimeout` if unset. |
| `MaxHeaderBytes` | total size of request headers | A single request with a 2 GB header line. The 1 MiB default is applied when 0, so this one is actually safe by default. |

Because `ReadTimeout` bounds the *whole* request including the body, you cannot set it
aggressively on an endpoint that accepts large uploads — so the near-universal
recommendation is a short `ReadHeaderTimeout` (which cannot conflict with anything) plus a
generous `ReadTimeout`, and per-handler deadlines via `context` or
`http.ResponseController` for the rest (16.3, 16.12).

`WriteTimeout` is also why you cannot implement long-polling or server-sent events without
either setting it to 0 for that route or extending the deadline with
`http.ResponseController` (16.3).

Idiomatic: never call `http.ListenAndServe` in production code. Build an `http.Server`,
set the five fields above, call `srv.ListenAndServe()`, and shut it down with
`srv.Shutdown(ctx)` — graceful shutdown is its own lesson ([18.5](../18-production/05-graceful-shutdown.md)).

## DefaultServeMux, and why not to use it

`http.HandleFunc`, `http.Handle`, and `ListenAndServe(addr, nil)` all operate on one
package-level `*ServeMux` called `DefaultServeMux`. It is a **global mutable registry**,
and any package in your dependency tree can register a route on it from an `init`
function. You will not see it in your code, your router, or your tests.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
)

func init() {
  http.HandleFunc("/debug/secret", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "internal build info nobody meant to expose")
  })
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("GET /api/ping", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "pong")
  })

  global := httptest.NewServer(nil)
  defer global.Close()
  own := httptest.NewServer(mux)
  defer own.Close()

  for _, tc := range []struct{ name, url string }{
    {"DefaultServeMux", global.URL + "/debug/secret"},
    {"own mux", own.URL + "/debug/secret"},
  } {
    resp, _ := http.Get(tc.url)
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-16s %s %q\n", tc.name, resp.Status, b)
  }
}
```

Output:

```
DefaultServeMux  200 OK "internal build info nobody meant to expose\n"
own mux          404 Not Found "404 page not found\n"
```

That `init` stands in for a real one: importing `net/http/pprof` registers four handlers
under `/debug/pprof/` on `DefaultServeMux` as a side effect of the import, and
`expvar` registers `/debug/vars`. Serve `DefaultServeMux` on a public port and you have
published your heap profile and every exported variable. The own-mux server returns 404
for the same path, because nothing can reach into a mux you constructed.

Idiomatic: `mux := http.NewServeMux()` always; pass it as `Handler` explicitly; use
`mux.Handle`/`mux.HandleFunc`, never the package-level `http.Handle`/`http.HandleFunc`.

## Quick Reference

| Function / field | Signature | Note |
|---|---|---|
| `http.ListenAndServe` | `(addr string, h Handler) error` | convenience only; no timeouts. Always returns non-nil |
| `http.ListenAndServeTLS` | `(addr, certFile, keyFile string, h Handler) error` | same caveat |
| `http.NewServeMux` | `() *ServeMux` | your own registry — use this |
| `http.HandleFunc` | `(pattern string, f func(ResponseWriter, *Request))` | registers on `DefaultServeMux`; avoid |
| `(*Server).ListenAndServe` | `() error` | returns `http.ErrServerClosed` after `Shutdown` |
| `(*Server).Shutdown` | `(ctx) error` | graceful; see 18.5 |
| `httptest.NewServer` | `(h Handler) *Server` | real listener on 127.0.0.1, random port |

| ResponseWriter method | Effect |
|---|---|
| `Header() http.Header` | the mutable response header map; edits after flush are silently dropped |
| `WriteHeader(code int)` | sends status + headers; second call logs `superfluous` and is ignored |
| `Write([]byte) (int, error)` | implicitly `WriteHeader(200)` first; error means the client went away |

| `http.Server` field | Default | Set it to |
|---|---|---|
| `ReadHeaderTimeout` | 0 (none) | 5s — blocks Slowloris with no downside |
| `ReadTimeout` | 0 (none) | 15-30s; longer for upload routes |
| `WriteTimeout` | 0 (none) | 15-30s; 0 for SSE/long-poll routes |
| `IdleTimeout` | falls back to `ReadTimeout` | 60-120s |
| `MaxHeaderBytes` | 1 MiB | leave it, or lower it |

## Common Mistakes

**Calling `WriteHeader` after writing the body.** The status is already on the wire, so
your code logs `superfluous response.WriteHeader call` and the client gets 200 — decide
the status before writing a single byte.

**Setting a header after the first `Write`.** `Header().Set` still succeeds and still
mutates the map, but the headers are gone, so the change is invisible; set every header
first.

**Using `http.ListenAndServe` in production.** Every timeout is zero, so one slow client
can pin a goroutine and a file descriptor forever; construct an `http.Server`.

**Registering on `DefaultServeMux`.** Any imported package can add routes to it from
`init` — `net/http/pprof` really does — so build your own `ServeMux` and pass it as
`Handler`.

**Mutating a package-level variable from a handler.** Handlers run concurrently on
separate goroutines, so that is a data race the race detector will find; hold a mutex or
use `sync/atomic`.

**Assuming `Content-Type` is right.** Go sniffs it from the first 512 bytes and guesses
`text/plain` for JSON; set it explicitly on every API response.

## Exercises

1. Write a server with one handler that sleeps 200ms and returns the value of a
   package-level request counter. Drive 500 concurrent requests at it through
   `httptest.NewServer` under `-race`, confirm the detector fires, then fix it two ways —
   `sync.Mutex` and `atomic.Int64` — and confirm both are race-clean and end at exactly 500.
2. Write a handler that calls `w.WriteHeader(201)`, then `w.WriteHeader(500)`, then writes
   a body. Capture the real server log with `srv.Config.ErrorLog` and state which status
   the client actually received and why.
3. Build a handler that sets `Content-Type: application/json`, writes `{"a":1}`, and *then*
   tries to set `Content-Length` and `X-Total-Count`. Print the response headers the client
   sees and explain exactly which of the three headers arrived.
4. Start two `httptest` servers over the same mux, one via `httptest.NewServer(mux)` and one
   via `httptest.NewServer(nil)` after importing `net/http/pprof` for its side effect
   (`_ "net/http/pprof"`). Request `/debug/pprof/heap` on both and print both statuses.
5. Construct an `http.Server` with `ReadHeaderTimeout: 100 * time.Millisecond` on a manual
   `net.Listen` listener. Connect with `net.Dial`, send `GET / HTTP/1.1\r\n` and nothing
   else, and capture what the server sends back and when. Then set the field to 0 and show
   the connection hangs instead.

---

[← 15.8 go generate and Workspaces](../15-tooling/08-generate-workspaces.md) · [Course Index](../README.md) · [16.2 ServeMux Routing: Methods and Wildcards →](02-servemux-routing.md)
