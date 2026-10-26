[← 11.5 The Race Detector](05-race-detector.md) · [Course Index](../README.md) · [11.7 context Values and Propagation →](07-context-values.md)

# 11.6 — context: Cancellation and Timeouts

## What You'll Learn

- The four-method `Context` interface, and why `Done()` returning a channel is the whole design
- Every cancel func **must** be called — `go vet`'s `lostcancel` check proves it, and a leak is measurable
- `ctx.Err()` distinguishes `Canceled` from `DeadlineExceeded`; `context.Cause` gives you *why*
- Cancellation propagates down the tree, never up, and the shortest deadline always wins

## Coming From Another Language

| Other | Go |
|---|---|
| JS `AbortController` + `AbortSignal` | `context.WithCancel` returning `(ctx, cancel)` |
| `AbortSignal.timeout(ms)` | `context.WithTimeout` |
| Python `asyncio.CancelledError` raised *into* the coroutine | nothing is raised; you must *select* on `ctx.Done()` |
| Python `asyncio.wait_for(coro, timeout)` | `WithTimeout` plus a `select` you write yourself |
| Rust `tokio::select!` with a `CancellationToken` | `select` with `<-ctx.Done()` |
| Thread cancellation / `Thread.interrupt()` | does not exist — goroutines are never killed from outside |

The one thing to internalize: Go cannot interrupt a goroutine. There is no
`goroutine.kill()`, no exception injected at an await point, no preemption you can
trigger. A context is a *broadcast channel plus a deadline plus a bag of values*, and
cancellation works only because every participant voluntarily checks `ctx.Done()`. A
goroutine that ignores the context runs to completion no matter what you cancel, which
makes "did I pass the context all the way down, and does every blocking step select on
it" the central question of every Go concurrency review. The flip side is that
cancellation is explicit and traceable: you can read the code and see exactly where it
takes effect.

## The Context interface

Four methods, and `context.Context` is just an interface:

```go
type Context interface {
  Deadline() (deadline time.Time, ok bool)
  Done() <-chan struct{}
  Err() error
  Value(key any) any
}
```

`Background()` is the root: never cancelled, no deadline, no values. `TODO()` is
identical at runtime and exists purely as a marker that a context should be plumbed
through here but has not been yet.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "time"
)

func main() {
  bg := context.Background()
  d, ok := bg.Deadline()
  fmt.Println("Background Deadline:", d.IsZero(), ok)
  fmt.Println("Background Done is nil:", bg.Done() == nil)
  fmt.Println("Background Err:", bg.Err())
  fmt.Println("Background Value:", bg.Value("anything"))
  fmt.Println("Background String:", bg)
  fmt.Println("TODO String:", context.TODO())

  ctx, cancel := context.WithCancel(bg)
  fmt.Println("before cancel, Err:", ctx.Err())
  cancel()
  <-ctx.Done()
  fmt.Println("after cancel, Err:", ctx.Err())
  fmt.Println("is context.Canceled:", errors.Is(ctx.Err(), context.Canceled))
  cancel()
  cancel()
  fmt.Println("cancel is idempotent, Err still:", ctx.Err())

  tctx, tcancel := context.WithTimeout(bg, 50*time.Millisecond)
  defer tcancel()
  start := time.Now()
  <-tctx.Done()
  fmt.Println("timeout fired after ~50ms:", time.Since(start) >= 50*time.Millisecond)
  fmt.Println("Err:", tctx.Err())
  fmt.Println("is DeadlineExceeded:", errors.Is(tctx.Err(), context.DeadlineExceeded))
  fmt.Println("is Canceled:", errors.Is(tctx.Err(), context.Canceled))
  dl, ok := tctx.Deadline()
  fmt.Println("Deadline set:", ok, "in the past now:", dl.Before(time.Now()))

  dctx, dcancel := context.WithDeadline(bg, time.Now().Add(-time.Second))
  defer dcancel()
  fmt.Println("already-past deadline, Err:", dctx.Err())
}
```

Output:

```
Background Deadline: true false
Background Done is nil: true
Background Err: <nil>
Background Value: <nil>
Background String: context.Background
TODO String: context.TODO
before cancel, Err: <nil>
after cancel, Err: context canceled
is context.Canceled: true
cancel is idempotent, Err still: context canceled
timeout fired after ~50ms: true
Err: context deadline exceeded
is DeadlineExceeded: true
is Canceled: false
Deadline set: true in the past now: true
already-past deadline, Err: context deadline exceeded
```

Eight facts worth extracting from that run.

`Background().Done()` returns a **nil channel**. Receiving from nil blocks forever, which
is exactly right: a context that is never cancelled should make `<-ctx.Done()` a case
that never fires. It also means `select` with `<-ctx.Done()` is always safe to write,
even on `Background()`.

`Err()` is `nil` until cancellation, then one of exactly two sentinel values:
`context.Canceled` or `context.DeadlineExceeded`. Compare with `errors.Is`, never with
`==` on a wrapped error, because by the time an error reaches your handler it has usually
been wrapped several times.

Calling `cancel()` three times is fine — it is idempotent and safe from any goroutine.
And `WithDeadline` with a time already in the past returns an already-cancelled context,
with no timer involved.

Idiomatic: `context.Background()` at the top of `main` and in tests; `r.Context()` in an
HTTP handler; `context.TODO()` only as a temporary marker you intend to remove.

## The cancel func must be called

`WithCancel`, `WithTimeout` and `WithDeadline` all register the new context with its
parent. Not calling `cancel` leaves it registered until the *parent* is cancelled — and
if the parent is `Background()`, that is forever.

```go
package main

import (
  "context"
  "fmt"
  "os"
  "runtime"
  "time"
)

func spawn(withCancel bool) {
  for range 1000 {
    ctx, cancel := context.WithCancel(context.Background())
    go func() {
      <-ctx.Done()
    }()
    if withCancel {
      cancel()
    }
  }
}

func main() {
  withCancel := len(os.Args) > 1 && os.Args[1] == "cancel"
  base := runtime.NumGoroutine()
  spawn(withCancel)
  time.Sleep(200 * time.Millisecond)
  fmt.Println("cancel called:", withCancel)
  fmt.Println("leaked goroutines:", runtime.NumGoroutine()-base)
}
```

`go vet .` on this (the `cancel` variable is assigned but only used on one path):

```
main.go:13:5: the cancel function is not used on all paths (possible context leak)
main.go:21:1: this return statement may be reached without using the cancel var defined on line 13
```

`go run .`:

```
cancel called: false
leaked goroutines: 1000
```

and `go run . cancel`:

```
cancel called: true
leaked goroutines: 0
```

1000 goroutines parked forever on `<-ctx.Done()`, plus 1000 contexts still chained to
`Background()`. The `lostcancel` analyzer is part of the default `go vet` suite, so this
is caught by `go test` too. The more common spelling of the bug trips it even harder:

```go
func leaky() context.Context {
  ctx, _ := context.WithCancel(context.Background())
  return ctx
}
```

```
main.go:9:8: the cancel function returned by context.WithCancel should be called, not discarded, to avoid a context leak
```

Idiomatic: `defer cancel()` on the very next line, every time. It is correct even when
the context finishes by timing out — cancelling an already-cancelled context is a no-op
that releases the timer and unregisters from the parent.

## Cancellation propagates to children, never to parents

```go
package main

import (
  "context"
  "fmt"
  "slices"
  "sync"
  "time"
)

func main() {
  parent, cancelParent := context.WithCancel(context.Background())
  defer cancelParent()
  childA, cancelA := context.WithCancel(parent)
  defer cancelA()
  childB, cancelB := context.WithTimeout(parent, time.Hour)
  defer cancelB()
  grand, cancelG := context.WithCancel(childA)
  defer cancelG()

  cancelA()
  fmt.Println("after cancelling childA:")
  fmt.Println("  childA:    ", childA.Err())
  fmt.Println("  grandchild:", grand.Err())
  fmt.Println("  childB:    ", childB.Err())
  fmt.Println("  parent:    ", parent.Err())

  var mu sync.Mutex
  var stopped []string
  var wg sync.WaitGroup
  wg.Go(func() {
    <-childB.Done()
    mu.Lock()
    stopped = append(stopped, fmt.Sprintf("childB: %v", childB.Err()))
    mu.Unlock()
  })
  wg.Go(func() {
    <-parent.Done()
    mu.Lock()
    stopped = append(stopped, fmt.Sprintf("parent: %v", parent.Err()))
    mu.Unlock()
  })

  cancelParent()
  wg.Wait()
  slices.Sort(stopped)
  fmt.Println("after cancelling the parent:")
  for _, s := range stopped {
    fmt.Println("  " + s)
  }
}
```

Output:

```
after cancelling childA:
  childA:     context canceled
  grandchild: context canceled
  childB:     <nil>
  parent:     <nil>
after cancelling the parent:
  childB: context canceled
  parent: context canceled
```

Deterministic, because `cancel()` closes the `Done` channels of the whole subtree before
it returns. Cancelling `childA` took its grandchild down and left its sibling and its
parent untouched; cancelling the parent took everything down. This is the mechanism that
makes "cancel the request" a single call at the top of a handler no matter how deep the
call tree goes.

Note `childB` was created with `WithTimeout(parent, time.Hour)` and still reports
`context canceled`, not `context deadline exceeded` — the parent's cancellation reached
it long before its own timer.

## Err: Canceled vs DeadlineExceeded

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()

if err := doWork(ctx); err != nil {
  switch {
  case errors.Is(err, context.DeadlineExceeded):
    // we ran out of time: a 504 to the client, retry maybe
  case errors.Is(err, context.Canceled):
    // somebody gave up: the client disconnected, or a sibling failed
  default:
    // a real error
  }
}
```

The distinction is operationally important and people routinely collapse it.
`DeadlineExceeded` means *your* budget expired — that is a timeout to alert on, and the
work may be worth retrying. `Canceled` means *somebody upstream stopped caring* — the
client hung up, or an errgroup sibling failed ([11.8](08-errgroup.md)) — and retrying is
pointless. Collapsing both into "timeout" produces dashboards full of phantom timeouts
every time a user closes a tab.

`http.Client` wraps these in a `*url.Error`, so `errors.Is` is mandatory:

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "net/url"
  "strings"
  "time"
)

func main() {
  gone := make(chan error, 4)
  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    d := 10 * time.Millisecond
    if r.URL.Query().Get("slow") == "1" {
      d = 500 * time.Millisecond
    }
    select {
    case <-time.After(d):
      fmt.Fprint(w, "payload")
    case <-r.Context().Done():
      gone <- r.Context().Err()
    }
  }))
  defer srv.Close()

  fetch := func(ctx context.Context, q string) (string, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, srv.URL+q, nil)
    if err != nil {
      return "", err
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      return "", err
    }
    defer resp.Body.Close()
    b, err := io.ReadAll(resp.Body)
    return string(b), err
  }

  clean := func(err error) string {
    return strings.Replace(err.Error(), srv.URL, "http://127.0.0.1:PORT", 1)
  }

  body, err := fetch(context.Background(), "")
  fmt.Printf("fast request: %q err=%v\n", body, err)

  ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
  defer cancel()
  _, err = fetch(ctx, "?slow=1")
  fmt.Println("slow request err:", clean(err))
  fmt.Println("  errors.Is DeadlineExceeded:", errors.Is(err, context.DeadlineExceeded))
  var uerr *url.Error
  fmt.Println("  wrapped in *url.Error:", errors.As(err, &uerr), "Timeout():", uerr.Timeout())
  fmt.Println("  server saw:", <-gone)

  cctx, ccancel := context.WithCancel(context.Background())
  go func() {
    time.Sleep(30 * time.Millisecond)
    ccancel()
  }()
  _, err = fetch(cctx, "?slow=1")
  fmt.Println("cancelled request err:", clean(err))
  fmt.Println("  errors.Is Canceled:", errors.Is(err, context.Canceled))
  fmt.Println("  server saw:", <-gone)
}
```

Output (the server's real port varies per run, so it is replaced with `PORT` before
printing; everything else is verbatim):

```
fast request: "payload" err=<nil>
slow request err: Get "http://127.0.0.1:PORT?slow=1": context deadline exceeded
  errors.Is DeadlineExceeded: true
  wrapped in *url.Error: true Timeout(): true
  server saw: context canceled
cancelled request err: Get "http://127.0.0.1:PORT?slow=1": context canceled
  errors.Is Canceled: true
  server saw: context canceled
```

Three things this proves against a real server. The transport actually aborts the
in-flight request — the handler's `r.Context()` fires, so the *server* learns the client
left and can stop doing expensive work. The error is wrapped in `*url.Error` whose
`Timeout()` reports true for the deadline case. And from the server's side both look like
`context canceled`, because a closed connection is a cancellation regardless of why the
client closed it.

Idiomatic: always `http.NewRequestWithContext`. `http.Client.Timeout` covers the whole
request including the body read but cannot be shortened per call or cancelled early;
a context can do both and composes with the rest of the request's budget.

## Timeouts compose: the shorter deadline wins

```go
package main

import (
  "context"
  "fmt"
  "time"
)

func main() {
  outer, c1 := context.WithTimeout(context.Background(), 100*time.Millisecond)
  defer c1()
  inner, c2 := context.WithTimeout(outer, 30*time.Millisecond)
  defer c2()

  od, _ := outer.Deadline()
  id, _ := inner.Deadline()
  fmt.Println("inner deadline is earlier:", id.Before(od))

  start := time.Now()
  <-inner.Done()
  fmt.Println("inner fired at ~30ms:", time.Since(start).Round(10*time.Millisecond))
  fmt.Println("outer still live:", outer.Err())

  wider, c3 := context.WithTimeout(context.Background(), 30*time.Millisecond)
  defer c3()
  child, c4 := context.WithTimeout(wider, time.Hour)
  defer c4()

  wd, _ := wider.Deadline()
  cd, _ := child.Deadline()
  fmt.Println("child asked for 1h but inherits the parent's deadline:", cd.Equal(wd))

  start = time.Now()
  <-child.Done()
  fmt.Println("child fired at ~30ms:", time.Since(start).Round(10*time.Millisecond))
  fmt.Println("child Err:", child.Err())
}
```

Output:

```
inner deadline is earlier: true
inner fired at ~30ms: 30ms
outer still live: <nil>
child asked for 1h but inherits the parent's deadline: true
child fired at ~30ms: 30ms
child Err: context deadline exceeded
```

The second half is the one people get wrong: `WithTimeout(parent, time.Hour)` on a parent
with 30ms left does **not** extend anything. `WithDeadline` explicitly keeps the earlier
of the two deadlines, so the returned context's `Deadline()` equals the parent's. A
deadline is a budget that can only shrink as it travels down the call tree, which is
exactly the property you want: an inner retry loop cannot vote itself more time than the
request has.

To genuinely escape a parent's deadline you need `context.WithoutCancel`, below.

## WithoutCancel and AfterFunc

`WithoutCancel` (Go 1.21) returns a context that keeps the parent's *values* but none of
its cancellation or deadline. `AfterFunc` (Go 1.21) runs a function in its own goroutine
when a context is done. Both verified on Go 1.27.1.

```go
package main

import (
  "context"
  "fmt"
  "time"
)

func main() {
  req, cancelReq := context.WithCancel(context.Background())
  audit := context.WithoutCancel(req)

  cancelReq()
  fmt.Println("request ctx Err:", req.Err())
  fmt.Println("audit ctx Err:  ", audit.Err())
  fmt.Println("audit Done is nil:", audit.Done() == nil)
  d, ok := audit.Deadline()
  fmt.Println("audit inherits no deadline:", d.IsZero(), ok)

  done := make(chan string, 2)

  c1, cc1 := context.WithCancel(context.Background())
  stop1 := context.AfterFunc(c1, func() { done <- "AfterFunc ran on cancel" })
  cc1()
  fmt.Println(<-done)
  fmt.Println("stop after it already ran:", stop1())

  c2, cc2 := context.WithCancel(context.Background())
  stop2 := context.AfterFunc(c2, func() { done <- "should not appear" })
  fmt.Println("stop before cancel:", stop2())
  cc2()
  time.Sleep(50 * time.Millisecond)
  fmt.Println("pending callbacks:", len(done))

  c3, cc3 := context.WithTimeout(context.Background(), 20*time.Millisecond)
  defer cc3()
  context.AfterFunc(c3, func() { done <- fmt.Sprintf("AfterFunc ran on timeout: %v", c3.Err()) })
  fmt.Println(<-done)

  already, ca := context.WithCancel(context.Background())
  ca()
  context.AfterFunc(already, func() { done <- "AfterFunc on an already-cancelled ctx still runs" })
  fmt.Println(<-done)
}
```

Output:

```
request ctx Err: context canceled
audit ctx Err:   <nil>
audit Done is nil: true
audit inherits no deadline: true false
AfterFunc ran on cancel
stop after it already ran: false
stop before cancel: true
pending callbacks: 0
AfterFunc ran on timeout: context deadline exceeded
AfterFunc on an already-cancelled ctx still runs
```

`WithoutCancel` exists for the one real case: detaching work that must outlive the
request that triggered it — flushing an audit record, writing a metric, finishing a
background upload — while still carrying the request ID and trace ID for logging
([11.7](07-context-values.md)). Note `Done()` on the detached context is nil again, so it
is genuinely uncancellable; give it its own timeout with `WithTimeout(audit, 5*time.Second)`
or you have created an unbounded operation.

`AfterFunc` returns a `stop func() bool`: true if it successfully prevented the call,
false if the function has already started or already run. The last line is the subtle
one — registering on an *already-cancelled* context still runs the function, immediately,
in a new goroutine. That makes `AfterFunc` safe to use without a "is it already done"
check, and it is how you attach cleanup to a cancellation without parking a goroutine on
`<-ctx.Done()` yourself.

## WithCancelCause and context.Cause

`Err()` only ever tells you *which of two things* happened. `WithCancelCause` (Go 1.20)
lets the canceller attach a real error, retrievable with `context.Cause`. Verified on
Go 1.27.1, along with `WithDeadlineCause` and `WithTimeoutCause` (Go 1.21).

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "time"
)

var ErrQuotaExceeded = errors.New("quota exceeded")

func main() {
  ctx, cancel := context.WithCancelCause(context.Background())
  fmt.Println("Cause before cancel:", context.Cause(ctx))
  cancel(fmt.Errorf("tenant acme: %w", ErrQuotaExceeded))
  <-ctx.Done()
  fmt.Println("Err:  ", ctx.Err())
  fmt.Println("Cause:", context.Cause(ctx))
  fmt.Println("errors.Is(Cause, ErrQuotaExceeded):", errors.Is(context.Cause(ctx), ErrQuotaExceeded))
  cancel(errors.New("a later cause is ignored"))
  fmt.Println("Cause after a second cancel:", context.Cause(ctx))

  nctx, ncancel := context.WithCancelCause(context.Background())
  ncancel(nil)
  fmt.Println("cancel(nil) gives Cause:", context.Cause(nctx))

  tctx, tcancel := context.WithTimeout(context.Background(), 20*time.Millisecond)
  defer tcancel()
  <-tctx.Done()
  fmt.Println("timeout Err:  ", tctx.Err())
  fmt.Println("timeout Cause:", context.Cause(tctx))

  dctx, dcancel := context.WithDeadlineCause(context.Background(),
    time.Now().Add(20*time.Millisecond), errors.New("upstream SLA is 20ms"))
  defer dcancel()
  <-dctx.Done()
  fmt.Println("WithDeadlineCause Err:  ", dctx.Err())
  fmt.Println("WithDeadlineCause Cause:", context.Cause(dctx))

  tcctx, tccancel := context.WithTimeoutCause(context.Background(),
    20*time.Millisecond, errors.New("search backend budget"))
  defer tccancel()
  <-tcctx.Done()
  fmt.Println("WithTimeoutCause Cause:", context.Cause(tcctx))
}
```

Output:

```
Cause before cancel: <nil>
Err:   context canceled
Cause: tenant acme: quota exceeded
errors.Is(Cause, ErrQuotaExceeded): true
Cause after a second cancel: tenant acme: quota exceeded
cancel(nil) gives Cause: context canceled
timeout Err:   context deadline exceeded
timeout Cause: context deadline exceeded
WithDeadlineCause Err:   context deadline exceeded
WithDeadlineCause Cause: upstream SLA is 20ms
WithTimeoutCause Cause: search backend budget
```

This is genuinely useful and underused. `Err()` stays `context canceled` — code that
switches on it keeps working — while `Cause` carries the domain error all the way down,
wrapping intact so `errors.Is` and `errors.As` work on it. The first cause wins; later
`cancel(err)` calls are ignored, the same way `Err()` is latched.

Four rules the run confirms: `Cause` returns `nil` before cancellation; `cancel(nil)`
falls back to `context.Canceled`; a plain `WithTimeout` has `Cause == Err ==
DeadlineExceeded`; and `WithTimeoutCause` / `WithDeadlineCause` let you say *whose*
budget expired, which is the difference between a useful and a useless timeout log.

Idiomatic: `WithCancelCause` whenever the reason for cancelling is information a caller
would want. `errgroup.WithContext` uses exactly this mechanism internally — the context
it hands you reports the failing goroutine's error as its `Cause`.

## A cancellable worker

The shape every long-running goroutine should have: `ctx` first parameter, a `select` on
`ctx.Done()` at every point it would otherwise block, and `ctx.Err()` wrapped into the
returned error.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "sync"
  "time"
)

func worker(ctx context.Context, id int, jobs <-chan int, results chan<- string) error {
  for {
    select {
    case <-ctx.Done():
      return fmt.Errorf("worker %d stopping: %w", id, ctx.Err())
    case j, ok := <-jobs:
      if !ok {
        return nil
      }
      select {
      case <-ctx.Done():
        return fmt.Errorf("worker %d stopping: %w", id, ctx.Err())
      case results <- fmt.Sprintf("job %d by worker %d", j, id):
      }
    }
  }
}

func main() {
  ctx, cancel := context.WithTimeout(context.Background(), 60*time.Millisecond)
  defer cancel()

  jobs := make(chan int)
  results := make(chan string, 100)
  var mu sync.Mutex
  var errs []error

  var wg sync.WaitGroup
  for id := range 3 {
    wg.Go(func() {
      err := worker(ctx, id, jobs, results)
      mu.Lock()
      errs = append(errs, err)
      mu.Unlock()
    })
  }

  produced := 0
feed:
  for i := 1; ; i++ {
    select {
    case <-ctx.Done():
      break feed
    case jobs <- i:
      produced++
      time.Sleep(10 * time.Millisecond)
    }
  }
  wg.Wait()
  close(results)

  completed := 0
  for range results {
    completed++
  }

  allDeadline := true
  for _, err := range errs {
    if !errors.Is(err, context.DeadlineExceeded) {
      allDeadline = false
    }
  }

  fmt.Println("workers that returned:", len(errs))
  fmt.Println("every error unwraps to DeadlineExceeded:", allDeadline)
  fmt.Println("completed == produced:", completed == produced)
  fmt.Println("one sample error:", errors.Unwrap(errs[0]))
}
```

Output:

```
workers that returned: 3
every error unwraps to DeadlineExceeded: true
completed == produced: true
one sample error: context deadline exceeded
```

The assertions are deterministic even though the exact job count is not: all three
workers return, every error unwraps to `DeadlineExceeded` via `%w`, and no job is
reported complete that was not produced. The job count itself is deliberately not printed
because it depends on how many 10ms ticks fit in the 60ms budget on the day.

The detail that is easy to miss: the **inner** `select` around `results <- ...`. Without
it, a worker that has a result to deliver blocks forever on a full channel after the
consumer gave up — a goroutine leak that no amount of outer `ctx.Done()` checking fixes.
Every blocking operation needs the `ctx.Done()` case, not just the first one.

## A goroutine that ignores ctx is a leak

```go
func bad(ctx context.Context) {
  go func() {
    time.Sleep(time.Hour)
    publish()
  }()
}
```

Cancelling `ctx` does nothing here. There is no mechanism by which Go interrupts that
sleep; the goroutine, its stack, and everything it references stay alive for an hour.
Scaled up by one request, this is how a service with a steady request rate and a
flat-looking heap profile dies at 3am. `time.Sleep` becomes
`select { case <-time.After(d): case <-ctx.Done(): return }`, a channel receive gets a
`ctx.Done()` case, and a library call gets its `...Context` variant. Diagnosing the ones
you missed — goroutine dumps, `runtime.NumGoroutine`, pprof — is
[12.5](../12-concurrency-patterns/05-deadlocks-leaks.md).

## Conventions

**`ctx` is the first parameter, always, and it is named `ctx`.**

```go
func (s *Service) Fetch(ctx context.Context, id int) (*Order, error)
func Query(ctx context.Context, db *sql.DB, q string) (*Rows, error)
```

Not second, not last, not optional, and not `context.Context` by value in a variadic.
This is one of the few genuinely rigid conventions in Go, and the whole ecosystem relies
on it: every `...Context` function in the standard library follows it, so a reviewer can
tell at a glance whether a call site is cancellable.

**Never store a context in a struct.**

```go
type Service struct {
  ctx context.Context
}
```

A context is scoped to one operation; a struct usually outlives many. Stuffing it in a
field means every method shares one deadline set by whoever constructed the service,
`WithValue` additions leak between unrelated calls, and the cancellation story becomes
unreadable. Pass it as an argument.

The one exception: a struct that *is* a single request or operation — `http.Request`
carries one, and so does a job you hand to a worker. If the struct's lifetime is exactly
the operation's lifetime, a `ctx` field is fine. Everything else, pass it.

**Never pass a nil context.** Use `context.Background()` or `context.TODO()`. A nil
`Context` panics on the first method call.

**Do not create a context for work that cannot be cancelled** just to satisfy a
signature — that is what `context.Background()` is for at the call site.

## Quick Reference

| Constructor | Returns | Note |
|---|---|---|
| `context.Background()` | `Context` | the root; never cancelled, `Done()` is nil |
| `context.TODO()` | `Context` | identical at runtime; a marker to fix later |
| `context.WithCancel(parent)` | `(Context, CancelFunc)` | cancel manually; `Err` is `Canceled` |
| `context.WithCancelCause(parent)` | `(Context, CancelCauseFunc)` | `cancel(err)`; read with `context.Cause` |
| `context.WithTimeout(parent, d)` | `(Context, CancelFunc)` | relative; `WithDeadline(parent, now+d)` |
| `context.WithTimeoutCause(parent, d, err)` | `(Context, CancelFunc)` | Go 1.21+ |
| `context.WithDeadline(parent, t)` | `(Context, CancelFunc)` | absolute; keeps the *earlier* deadline |
| `context.WithDeadlineCause(parent, t, err)` | `(Context, CancelFunc)` | Go 1.21+ |
| `context.WithoutCancel(parent)` | `Context` | keeps values, drops cancellation and deadline |
| `context.WithValue(parent, k, v)` | `Context` | see [11.7](07-context-values.md) |
| `context.AfterFunc(ctx, f)` | `stop func() bool` | runs `f` in a goroutine when `ctx` is done |

| Method / function | Returns | Note |
|---|---|---|
| `ctx.Done()` | `<-chan struct{}` | closed on cancellation; **nil** if uncancellable |
| `ctx.Err()` | `error` | `nil`, `context.Canceled`, or `context.DeadlineExceeded` |
| `ctx.Deadline()` | `(time.Time, bool)` | `ok` is false when there is no deadline |
| `ctx.Value(k)` | `any` | `nil` when absent |
| `context.Cause(ctx)` | `error` | the cause, or `ctx.Err()`, or `nil` if live |

| Need | Use |
|---|---|
| stop work on a signal you control | `WithCancel` |
| stop work after a duration | `WithTimeout` |
| stop work at an absolute instant (an SLA boundary) | `WithDeadline` |
| tell the callee *why* you cancelled | `WithCancelCause` + `context.Cause` |
| run cleanup when a context ends | `AfterFunc` |
| let background work outlive the request | `WithoutCancel` (plus its own timeout) |
| cancel a sibling when one of N fails | [`errgroup.WithContext`](08-errgroup.md) |

## Common Mistakes

**Dropping the cancel func.** `ctx, _ := context.WithCancel(...)` leaks the context and
anything parked on it; `defer cancel()` on the next line, and let `go vet`'s `lostcancel`
check hold you to it.

**Treating `Canceled` and `DeadlineExceeded` as the same thing.** One means your budget
expired, the other means somebody upstream gave up; distinguish them with `errors.Is` and
only alert on the first.

**Comparing errors with `==`.** `http.Client` wraps context errors in `*url.Error`, so
`err == context.DeadlineExceeded` is false; use `errors.Is`.

**Only checking `ctx.Done()` at the top of the loop.** Every blocking operation inside
needs its own `ctx.Done()` case, including the channel send that delivers the result.

**Storing a context in a long-lived struct.** One operation's deadline then applies to
every method; pass it as the first parameter instead.

**Expecting `WithTimeout(parent, time.Hour)` to extend the parent's deadline.** The
earlier deadline always wins; `WithoutCancel` is the only way to detach.

## Exercises

1. Write `FetchFirst(ctx, urls []string) (string, error)` that fires all requests in
   parallel, returns the first 200 response and cancels the rest. Prove with
   `runtime.NumGoroutine` that nothing is left behind, and that a cancelled `ctx` returns
   `Canceled` without issuing any request.
2. Build a retry loop with exponential backoff that respects the context: it must not
   sleep past the deadline, and it must return the *last* real error rather than
   `DeadlineExceeded` when a real error is available. Test it with `WithTimeout(ctx, 0)`.
3. Implement graceful shutdown in a server: `signal.NotifyContext` for SIGINT,
   `srv.Shutdown(shutdownCtx)` with a 10s budget, and `WithoutCancel` for an
   audit-log flush that must survive. Verify with a slow handler that it drains.
4. Take the cancellable worker in this lesson and remove the inner `select` around the
   result send. Write a test with `runtime.NumGoroutine` that fails because of the leak,
   then put it back.
5. Use `WithCancelCause` to build a `Group` where each goroutine's failure cancels the
   rest *and* every surviving goroutine can read which sibling failed via
   `context.Cause`. Compare your API with `errgroup`'s in [11.8](08-errgroup.md).

---

[← 11.5 The Race Detector](05-race-detector.md) · [Course Index](../README.md) · [11.7 context Values and Propagation →](07-context-values.md)
