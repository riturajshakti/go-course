[← 16.11 Testing Handlers with httptest](11-httptest.md) · [Course Index](../README.md) · [17.1 database/sql and the Driver Model →](../17-database/01-database-sql.md)

# 16.12 — Context Propagation Through Handlers

## What You'll Learn

- `r.Context()` is cancelled when the client disconnects **and** when the handler returns — both demonstrated for real
- The trap: a goroutine spawned from a handler that keeps using `r.Context()`, and `context.WithoutCancel` as the Go 1.21+ fix
- What `context.WithTimeout`, `http.TimeoutHandler` and `Server.ReadTimeout` each actually bound
- The payoff: cancelling an inbound request genuinely aborts the outbound HTTP call it triggered, across two real servers

## Coming From Another Language

| Other | Go |
|---|---|
| Node `req.on('close')` / `AbortController` | `<-r.Context().Done()` — one mechanism for both |
| `AbortSignal` passed implicitly through async calls | `ctx` is an **explicit first parameter** on every function |
| Express: a disconnected client is silently ignored | the request context is cancelled; your handler can stop working |
| Python `asyncio.CancelledError` raised into the coroutine | nothing is raised; you must *check* `ctx.Done()` or `ctx.Err()` |
| A thread-local request object | no goroutine-local storage; the context argument is the only channel |

This lesson closes the module by connecting it to [11.6](../11-sync-context/06-context-cancel.md)
and [11.7](../11-sync-context/07-context-values.md), whose conventions — an unexported
`struct{}` key type, typed getter/setter helpers, a logger derived from the context — are
reused verbatim here.

## The request context lifecycle

`r.Context()` is created by the server per request and cancelled when **either** of two
things happens:

1. the client disconnects, or
2. **the handler returns**

The second is the one people are unaware of, and it is the source of a specific class of bug.
Both, demonstrated:

```go
package main

import (
  "context"
  "fmt"
  "net/http"
  "net/http/httptest"
  "sort"
  "sync"
  "time"
)

func main() {
  var wg sync.WaitGroup

  fmt.Println("=== 1. client disconnects mid-request ===")
  done := make(chan string, 1)
  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    fmt.Printf("  handler start: ctx.Err()=%v\n", ctx.Err())
    select {
    case <-time.After(5 * time.Second):
      done <- "work finished"
    case <-ctx.Done():
      done <- fmt.Sprintf("cancelled: %v", ctx.Err())
    }
  }))
  defer srv.Close()

  cctx, cancel := context.WithCancel(context.Background())
  req, _ := http.NewRequestWithContext(cctx, "GET", srv.URL, nil)
  wg.Add(1)
  go func() {
    defer wg.Done()
    _, err := http.DefaultClient.Do(req)
    fmt.Printf("  client Do err: %v\n", err != nil)
  }()
  time.Sleep(80 * time.Millisecond)
  cancel()
  fmt.Printf("  handler observed -> %s\n", <-done)
  wg.Wait()

  fmt.Println("\n=== 2. ctx is cancelled when the handler RETURNS ===")
  observed := make(chan string, 1)
  srv2 := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    go func() {
      time.Sleep(100 * time.Millisecond)
      observed <- fmt.Sprintf("background goroutine sees ctx.Err()=%v", ctx.Err())
    }()
    fmt.Fprint(w, "handler returned immediately")
  }))
  defer srv2.Close()
  resp, _ := http.Get(srv2.URL)
  resp.Body.Close()
  fmt.Printf("  response received, handler has returned\n")
  fmt.Printf("  %s\n", <-observed)

  fmt.Println("\n=== 3. context.WithoutCancel detaches the background work ===")
  results := make(chan string, 2)
  srv3 := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    reqCtx := r.Context()
    detached := context.WithoutCancel(reqCtx)

    go func() {
      time.Sleep(100 * time.Millisecond)
      results <- fmt.Sprintf("  r.Context():                Err()=%v  value=%v",
        reqCtx.Err(), reqCtx.Value(tenantKey{}))
    }()
    go func() {
      time.Sleep(100 * time.Millisecond)
      bctx, bcancel := context.WithTimeout(detached, 5*time.Second)
      defer bcancel()
      results <- fmt.Sprintf("  context.WithoutCancel(ctx): Err()=%v  value=%v  (own deadline usable: %v)",
        detached.Err(), detached.Value(tenantKey{}), bctx.Err() == nil)
    }()
    fmt.Fprint(w, "ok")
  }))
  defer srv3.Close()

  withTenant := func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      ctx := context.WithValue(r.Context(), tenantKey{}, "acme-corp")
      next.ServeHTTP(w, r.WithContext(ctx))
    })
  }
  srv3.Config.Handler = withTenant(srv3.Config.Handler)
  resp, _ = http.Get(srv3.URL)
  resp.Body.Close()
  two := []string{<-results, <-results}
  sort.Strings(two)
  for _, line := range two {
    fmt.Println(line)
  }
}

type tenantKey struct{}
```

The two goroutines in part 3 finish in a nondeterministic order, so their lines are sorted
before printing.

Output:

```
=== 1. client disconnects mid-request ===
  handler start: ctx.Err()=<nil>
  client Do err: true
  handler observed -> cancelled: context canceled

=== 2. ctx is cancelled when the handler RETURNS ===
  response received, handler has returned
  background goroutine sees ctx.Err()=context canceled

=== 3. context.WithoutCancel detaches the background work ===
  context.WithoutCancel(ctx): Err()=<nil>  value=acme-corp  (own deadline usable: true)
  r.Context():                Err()=context canceled  value=acme-corp
```

Three results, each worth its own paragraph.

**Part 1 — the client disconnect.** The handler was asleep for 5 seconds; the client gave up
at 80ms; the handler's `ctx.Done()` fired with `context canceled`. That is free abort
propagation: a user closing a browser tab stops your server doing work nobody will read. It
only works if your handler *checks*. A handler doing `time.Sleep(5*time.Second)` instead of a
`select` would sit there regardless, because **nothing is raised into a goroutine** — Go has
no cancellation exception.

**Part 2 — the handler-return cancellation, and the trap.** The handler returned immediately,
and 100ms later the background goroutine found `ctx.Err() == context canceled`. Nothing went
wrong for the *client* — the response was fine. But the goroutine's context is dead, so if it
had tried to do I/O with it, every call would fail instantly with `context canceled`. This is
the bug: you add a "fire and forget" audit write or a metrics push to a handler, it works in
development because your fake is a no-op, and in production it silently never happens.

**Part 3 — `context.WithoutCancel` (Go 1.21+, verified on 1.27.1) is the fix.** The detached
context's `Err()` is `nil` after the handler returned, **and it still carries the values** —
`value=acme-corp` on both lines. That combination is exactly what detached work needs: the
tenant ID and request ID for logging, without the cancellation. Note the final clause: you can
layer a *new* deadline on it with `context.WithTimeout(detached, 5*time.Second)`, which you
must, or you have created an unbounded background operation.

The correct shape for background work from a handler:

```go
func (h *Handler) create(w http.ResponseWriter, r *http.Request) {
  // ... do the request's real work with r.Context() ...

  auditCtx, cancel := context.WithTimeout(context.WithoutCancel(r.Context()), 5*time.Second)
  h.wg.Go(func() {
    defer cancel()
    if err := h.audit.Record(auditCtx, event); err != nil {
      logger(auditCtx).Error("audit write failed", "err", err)
    }
  })
  writeJSON(w, http.StatusCreated, created)
}
```

The `h.wg` is a `sync.WaitGroup` owned by the handler struct, waited on at shutdown
([18.5](../18-production/05-graceful-shutdown.md)) — otherwise `srv.Shutdown` returns while
your audit writes are still in flight and the process exits mid-write. A goroutine spawned
from a handler with no owner is a leak
([12.5](../12-concurrency-patterns/05-deadlocks-leaks.md)).

## r.WithContext: the request is immutable by convention

There is no `r.SetContext`. To add a value you build a shallow copy:

```go
ctx := context.WithValue(r.Context(), tenantKey{}, tenant)
next.ServeHTTP(w, r.WithContext(ctx))
```

`r.WithContext(ctx)` returns a new `*http.Request` sharing everything except the context.
Mutating `r` in place would be a data race against anything else holding the request — and
the middleware above you *is* holding it. The pattern is identical for a deadline:

```go
ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
defer cancel()
r = r.WithContext(ctx)
```

`defer cancel()` is not optional. Skipping it leaks the timer and the context's goroutine
until the deadline fires; `go vet`'s `lostcancel` check catches the obvious cases.

## Pass ctx into every downstream call

The rule is mechanical: **every function that does I/O or can block takes `ctx context.Context`
as its first parameter.**

```go
func (r *Repository) FindUser(ctx context.Context, id int) (*User, error)
func (s *Service) Report(ctx context.Context) (string, error)
func (c *APIClient) GetItem(ctx context.Context, id int) (Item, error)
```

And every such function passes it on:

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)   // 16.10
rows, err := db.QueryContext(ctx, "SELECT ...", args...)                // 17.x
```

Note the `Context` suffix on the `database/sql` methods — `db.Query` exists and uses
`context.Background()`, which is why module 17 uses `QueryContext`/`ExecContext`
exclusively. Same for `pgx`. A single non-context call anywhere in the chain breaks
cancellation for everything below it, and nothing will tell you.

## What each timeout actually bounds

Four different mechanisms, routinely confused:

| Mechanism | Bounds | On expiry | Can the handler see it? |
|---|---|---|---|
| `Server.ReadHeaderTimeout` | accept → end of request headers | connection killed, **no HTTP response** | no — the handler never ran |
| `Server.ReadTimeout` | accept → end of request **body** | connection killed, no response | no |
| `Server.WriteTimeout` | headers read → response fully written | connection killed mid-response | only via a `Write` error |
| `http.TimeoutHandler(h, d, msg)` | handler **execution** | cancels `r.Context()`, writes **503** + `msg` | yes, `ctx.Done()` |
| `context.WithTimeout` in the handler | whatever you pass it to | your code decides the status | yes, you own it |
| `Client.Timeout` (outbound) | the whole outbound exchange | `*url.Error`, `Timeout()` true | n/a — it is your client |

The practical division of labour: **`Server.*Timeout` protects the server from slow clients**
(16.1) and produces no HTTP response at all. **`context.WithTimeout` inside the handler
protects the client from slow dependencies** and lets you choose 504 vs 503 vs a degraded
200. `http.TimeoutHandler` sits in between — convenient as blanket middleware, but it buffers
the whole response (so it breaks streaming) and cannot kill the handler goroutine (16.6).

Three facts that follow, and surprise people:

- **No timeout can stop a running goroutine.** Go has no goroutine kill. Every timeout is
  advisory: it cancels a context, and only code that checks the context stops.
- `Server.WriteTimeout` is why a long-polling or SSE endpoint needs either `WriteTimeout: 0`
  or `http.ResponseController.SetWriteDeadline` per chunk (16.3).
- A per-request `context.WithTimeout` deadline composes with the inbound one: the *earliest*
  deadline in the chain wins, so a 2s handler timeout inside a request whose client gave up
  after 100ms is cancelled at 100ms.

## Request ID and a logger through the context

Reusing 11.7's conventions exactly — an unexported `struct{}` key, a typed setter and getter,
and a logger *derived* from the context rather than stored in it:

```go
type reqIDKey struct{}

func WithRequestID(ctx context.Context, id string) context.Context {
  return context.WithValue(ctx, reqIDKey{}, id)
}

func RequestID(ctx context.Context) string {
  id, _ := ctx.Value(reqIDKey{}).(string)
  return id
}

func logger(ctx context.Context) *slog.Logger {
  return base.With("request_id", RequestID(ctx))
}
```

Deriving the logger means every layer gets the request ID in its log lines without any layer
knowing about HTTP. Storing a pre-built `*slog.Logger` in the context is also legitimate
(and cheaper if you attach many attributes); 11.7 compares the two.

Context values cross middleware boundaries the same way they cross function boundaries —
`r.WithContext` in each middleware layers one more value onto the chain, and a handler reads
all of them. The chain is per-request, so `req-001` and `req-002` never see each other's
values.

## The payoff: inbound cancellation aborting an outbound call

Everything in this module, in one program. A handler → service → repository chain
(16.3), a request-ID middleware (16.6), an HTTP client with context (16.10), two real
`httptest` servers (16.11), and context cancellation threading all the way through.

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

type reqIDKey struct{}

func WithRequestID(ctx context.Context, id string) context.Context {
  return context.WithValue(ctx, reqIDKey{}, id)
}

func RequestID(ctx context.Context) string {
  id, _ := ctx.Value(reqIDKey{}).(string)
  return id
}

var base = slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
  ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
    if a.Key == slog.TimeKey {
      return slog.Attr{}
    }
    return a
  },
}))

func logger(ctx context.Context) *slog.Logger {
  return base.With("request_id", RequestID(ctx))
}

func requestIDMiddleware(gen func() string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      id := gen()
      w.Header().Set("X-Request-Id", id)
      next.ServeHTTP(w, r.WithContext(WithRequestID(r.Context(), id)))
    })
  }
}

type Upstream struct {
  baseURL string
  http    *http.Client
}

func (u *Upstream) Fetch(ctx context.Context, path string) (string, error) {
  logger(ctx).Info("outbound request", "layer", "repository", "path", path)
  req, err := http.NewRequestWithContext(ctx, http.MethodGet, u.baseURL+path, nil)
  if err != nil {
    return "", err
  }
  resp, err := u.http.Do(req)
  if err != nil {
    logger(ctx).Warn("outbound request aborted", "layer", "repository",
      "cause", classify(err))
    return "", fmt.Errorf("upstream %s: %w", path, err)
  }
  defer func() { io.Copy(io.Discard, resp.Body); resp.Body.Close() }()
  b, _ := io.ReadAll(resp.Body)
  return string(b), nil
}

func classify(err error) string {
  switch {
  case errors.Is(err, context.Canceled):
    return "context.Canceled"
  case errors.Is(err, context.DeadlineExceeded):
    return "context.DeadlineExceeded"
  }
  return "other"
}

type Service struct{ up *Upstream }

func (s *Service) Report(ctx context.Context) (string, error) {
  logger(ctx).Info("building report", "layer", "service")
  return s.up.Fetch(ctx, "/slow")
}

func handler(svc *Service, perRequestTimeout time.Duration) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    if perRequestTimeout > 0 {
      var cancel context.CancelFunc
      ctx, cancel = context.WithTimeout(ctx, perRequestTimeout)
      defer cancel()
    }
    logger(ctx).Info("handler start", "layer", "handler", "path", r.URL.Path)

    out, err := svc.Report(ctx)
    if err != nil {
      status := http.StatusBadGateway
      if errors.Is(err, context.DeadlineExceeded) {
        status = http.StatusGatewayTimeout
      } else if errors.Is(err, context.Canceled) {
        status = 499
      }
      logger(ctx).Error("handler failed", "layer", "handler",
        "status", status, "cause", classify(err))
      http.Error(w, "upstream problem", status)
      return
    }
    fmt.Fprintf(w, "report: %s", out)
  })
}

func main() {
  var upstreamSaw atomic.Int32
  upstreamDone := make(chan string, 4)
  upstream := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    upstreamSaw.Add(1)
    select {
    case <-time.After(3 * time.Second):
      upstreamDone <- "upstream completed its 3s work"
      io.WriteString(w, "SLOW-DATA")
    case <-r.Context().Done():
      upstreamDone <- fmt.Sprintf("upstream ABORTED: %v", r.Context().Err())
    }
  }))
  defer upstream.Close()

  var n atomic.Int64
  gen := func() string { return fmt.Sprintf("req-%03d", n.Add(1)) }
  up := &Upstream{baseURL: upstream.URL, http: upstream.Client()}
  svc := &Service{up: up}

  fmt.Println("### A. inbound client cancels -> outbound aborted")
  ours := httptest.NewServer(requestIDMiddleware(gen)(handler(svc, 0)))
  defer ours.Close()

  cctx, cancel := context.WithCancel(context.Background())
  req, _ := http.NewRequestWithContext(cctx, "GET", ours.URL+"/report", nil)
  go func() {
    time.Sleep(100 * time.Millisecond)
    cancel()
  }()
  _, err := http.DefaultClient.Do(req)
  fmt.Printf("  inbound client saw an error: %v\n", err != nil)
  fmt.Printf("  %s\n", <-upstreamDone)

  fmt.Println("\n### B. per-request timeout -> 504, outbound aborted")
  timed := httptest.NewServer(requestIDMiddleware(gen)(handler(svc, 120*time.Millisecond)))
  defer timed.Close()
  resp, err := http.Get(timed.URL + "/report")
  if err != nil {
    panic(err)
  }
  b, _ := io.ReadAll(resp.Body)
  resp.Body.Close()
  fmt.Printf("  inbound client got: %s %q\n", resp.Status, b)
  fmt.Printf("  %s\n", <-upstreamDone)
  fmt.Printf("  upstream was called %d times total\n", upstreamSaw.Load())
}
```

Output (clean under `go run -race .`; the `slog` time attribute is stripped and the request-ID
generator is injected, so this is reproducible):

```
### A. inbound client cancels -> outbound aborted
level=INFO msg="handler start" request_id=req-001 layer=handler path=/report
level=INFO msg="building report" request_id=req-001 layer=service
level=INFO msg="outbound request" request_id=req-001 layer=repository path=/slow
  inbound client saw an error: true
level=WARN msg="outbound request aborted" request_id=req-001 layer=repository cause=context.Canceled
level=ERROR msg="handler failed" request_id=req-001 layer=handler status=499 cause=context.Canceled
  upstream ABORTED: context canceled

### B. per-request timeout -> 504, outbound aborted
level=INFO msg="handler start" request_id=req-002 layer=handler path=/report
level=INFO msg="building report" request_id=req-002 layer=service
level=INFO msg="outbound request" request_id=req-002 layer=repository path=/slow
level=WARN msg="outbound request aborted" request_id=req-002 layer=repository cause=context.DeadlineExceeded
level=ERROR msg="handler failed" request_id=req-002 layer=handler status=504 cause=context.DeadlineExceeded
  inbound client got: 504 Gateway Timeout "upstream problem\n"
  upstream ABORTED: context canceled
  upstream was called 2 times total
```

Six things in that output, and together they are the whole module.

**Case A: a cancelled inbound request aborted the outbound one.** The user gave up after
100ms. The upstream — a *different process* in a real deployment — printed
`upstream ABORTED: context canceled` instead of spending 3 seconds producing a report nobody
would read. Nobody wrote cancellation-propagation code. It happened because `r.Context()`
was passed to the service, which passed it to the repository, which passed it to
`http.NewRequestWithContext`. Every link is one argument.

**The request ID reached all three layers** — `req-001` on the handler, service, and
repository lines — and the repository does not import `net/http` for the request type, only
for the client. The layers stay decoupled while the diagnostics stay joined, which is 11.7's
payoff realized.

**Case B: the per-request timeout produced a proper 504.** The handler's own
`context.WithTimeout(r.Context(), 120ms)` fired, the error came back as
`context.DeadlineExceeded`, and `errors.Is` distinguished it from `context.Canceled` so the
handler could choose 504 rather than 499. The inbound client got a real HTTP response — which
`Server.ReadTimeout` could never have given it.

**`errors.Is` sees cancellation through the wrapping.** The repository wrapped with
`fmt.Errorf("upstream %s: %w", path, err)`, and the client had already wrapped it in a
`*url.Error`. Two layers of wrapping, and `errors.Is(err, context.DeadlineExceeded)` still
works ([8.3](../08-errors/03-is-as-unwrap.md)).

**499 for a client disconnect.** It is a non-standard nginx code, and it is the convention for
"the client went away so there is nobody to send a status to". Nothing receives it; it exists
to keep your metrics honest — a disconnect is not a 500, and counting it as one will have you
chasing phantom outages.

**The upstream was called exactly twice** — once per inbound request. No retry fired, because
a cancelled or deadline-exceeded context is not a retryable failure (16.10).

## Server shutdown and in-flight contexts

`srv.Shutdown(ctx)` stops accepting new connections and waits for in-flight handlers to
return. It does **not** cancel their request contexts — a handler mid-work keeps its live
context and finishes normally, which is the point of a graceful shutdown. If `Shutdown`'s own
context expires first, it returns `context.DeadlineExceeded` and you decide whether to call
`srv.Close()` and drop the remaining connections. The interaction with detached
`WithoutCancel` goroutines is the reason they need an owner — `Shutdown` knows nothing about
them. Full treatment in [18.5](../18-production/05-graceful-shutdown.md).

## A checklist for correct context handling

1. Every I/O or blocking function takes `ctx context.Context` as its **first** parameter.
2. Handlers start from `r.Context()` — never `context.Background()`.
3. Use the `Context` variants of every stdlib call: `NewRequestWithContext`, `QueryContext`,
   `ExecContext`.
4. `defer cancel()` immediately after every `WithCancel`/`WithTimeout`/`WithDeadline`.
5. Add values with `r.WithContext(ctx)` in middleware; never mutate `r`.
6. Context keys are unexported `struct{}` types with typed getter/setter helpers (11.7).
7. Context carries **request-scoped metadata only** — request ID, trace ID, user, tenant.
   Never config, never a database handle, never an optional parameter.
8. Background work from a handler uses `context.WithoutCancel` **plus its own
   `WithTimeout`**, and is owned by a `WaitGroup` you wait on at shutdown.
9. Long loops check `ctx.Err()` or select on `ctx.Done()` — a timeout cannot interrupt code
   that never looks.
10. Distinguish `context.Canceled` (client gave up → 499, do not retry, do not alert) from
    `context.DeadlineExceeded` (your dependency was slow → 504, maybe alert) with `errors.Is`.
11. Do not retry a request whose context is already done.
12. Test cancellation explicitly (16.11) — it is the path that is never exercised by hand.

## Quick Reference

| Call | Note |
|---|---|
| `r.Context()` | cancelled on client disconnect **and** on handler return |
| `r.WithContext(ctx)` | shallow copy; the only way to attach a context |
| `context.WithTimeout(parent, d)` | earliest deadline in the chain wins; `defer cancel()` |
| `context.WithoutCancel(parent)` | 1.21+; keeps **values**, drops cancellation |
| `context.WithCancelCause(parent)` | 1.20+; `context.Cause(ctx)` gives your own error |
| `context.AfterFunc(ctx, f)` | 1.21+; run `f` when `ctx` is done |
| `ctx.Err()` | `nil`, `context.Canceled`, or `context.DeadlineExceeded` |
| `ctx.Done()` | a channel closed on cancellation; select on it |

| Symptom | Cause |
|---|---|
| background work silently never runs | it used `r.Context()` after the handler returned |
| `context canceled` on every call in a goroutine | same |
| cancellation ignored by a dependency | a non-`Context` stdlib call somewhere in the chain |
| handler keeps burning CPU after a timeout | nothing checks `ctx.Done()` |
| 500s spiking during a deploy | `context.Canceled` counted as a server error |
| timer leak reported by pprof | a missing `defer cancel()` |

## Common Mistakes

**Using `r.Context()` in a goroutine that outlives the handler.** It is cancelled the moment
the handler returns, so the work silently fails; use `context.WithoutCancel` plus its own
timeout.

**Spawning a goroutine from a handler with no owner.** `srv.Shutdown` cannot wait for it and
the process exits mid-write; register it with a `WaitGroup` on the handler struct.

**Calling `context.Background()` inside a handler.** You discard cancellation and every
context value, so a disconnected client's work runs to completion and your logs lose the
request ID.

**Forgetting `defer cancel()`.** The timer and the context's goroutine leak until the deadline;
`go vet`'s `lostcancel` catches the simple cases.

**Using `db.Query` instead of `db.QueryContext`.** One non-context call breaks cancellation for
everything below it, with no warning.

**Expecting a timeout to stop a running goroutine.** Nothing is raised into a goroutine in Go;
only code that checks `ctx.Done()` or `ctx.Err()` stops.

**Treating `context.Canceled` as a server error.** A client disconnect is not a 500, and
counting it as one makes you chase outages that never happened.

**Mutating `r` to pass data down.** `*http.Request` is shared with the middleware above you, so
that is a race; build a copy with `r.WithContext(ctx)`.

## Exercises

1. Write a handler that runs three independent upstream calls concurrently with
   `errgroup.WithContext` ([11.8](../11-sync-context/08-errgroup.md)) against three
   `httptest` servers, one of which fails. Prove with real output that the other two were
   cancelled, and report how long the handler took versus the slowest server.
2. Build the audit-goroutine bug on purpose: a handler that writes an audit record in a
   goroutine using `r.Context()`. Capture the real failure, then fix it with
   `context.WithoutCancel` plus a 5s timeout and a `WaitGroup`, and show the record is written
   even when the client disconnects.
3. Compare all four timeout mechanisms against one slow handler: `Server.ReadTimeout`,
   `http.TimeoutHandler`, a handler-level `context.WithTimeout`, and `Client.Timeout` on the
   caller. Capture what the client receives in each case and fill in the table from this
   lesson from your own measurements.
4. Use `context.WithCancelCause` and `context.Cause` so a handler can distinguish "the client
   disconnected" from "the circuit breaker opened" from "the deadline passed", and map each to
   a different status. Show all three real responses.
5. Instrument the three-layer chain from this lesson with `context.AfterFunc` to emit a
   `request_abandoned` metric whenever a request context is cancelled before the handler
   finishes. Verify it fires on a disconnect and does **not** fire on a successful request.

---

[← 16.11 Testing Handlers with httptest](11-httptest.md) · [Course Index](../README.md) · [17.1 database/sql and the Driver Model →](../17-database/01-database-sql.md)
