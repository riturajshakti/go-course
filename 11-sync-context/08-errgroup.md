[← 11.7 context Values and Propagation](07-context-values.md) · [Course Index](../README.md) · [12.1 Worker Pools →](../12-concurrency-patterns/01-worker-pools.md)

# 11.8 — errgroup

## What You'll Learn

- `errgroup.Group` is `WaitGroup` plus one error — replacing ~20 lines of channel plumbing
- `errgroup.WithContext` cancels every sibling when the first goroutine fails: the reason to use it
- `SetLimit` bounds concurrency and makes `Go` **block**; `TryGo` reports instead of blocking
- Panics are *not* recovered, and only the *first* error survives — both need deliberate handling

## Coming From Another Language

| Other | Go |
|---|---|
| `Promise.all` (rejects on first failure, no cancellation) | `errgroup.WithContext` — and it *does* cancel the rest |
| `Promise.allSettled` (every result) | not errgroup; a mutex plus `errors.Join` |
| Python `asyncio.gather(..., return_exceptions=False)` | `errgroup.Group.Wait` |
| Python `asyncio.TaskGroup` (3.11+), cancels siblings | the closest analogue, including the cancellation |
| Rust `tokio::try_join!` / `JoinSet` | `errgroup` + `SetLimit` |
| `Promise.all` auto-rejects on a thrown exception | a panic in a goroutine **kills the process** |

One difference dominates: `Promise.all` rejecting does not stop the other promises — they
keep running and their results are discarded. `errgroup.WithContext` hands you a context
cancelled the instant any goroutine returns an error, so the siblings can actually
*stop*, release their connections and stop burning CPU. That is the feature you are
buying. The other asymmetry: in JS a throw inside a `Promise.all` task becomes a
rejection, whereas in Go an unrecovered panic in any goroutine takes down the whole
process, errgroup included.

## Install it

`errgroup` is not in the standard library; it lives in the `x/sync` repo. `go get
golang.org/x/sync/errgroup` resolved the version every output below was captured with:

```
go: added golang.org/x/sync v0.24.0
```

```go
import "golang.org/x/sync/errgroup"
```

## First, the manual version

This is what errgroup replaces. Fan out three fetches, collect the results, surface an
error.

```go
package main

import (
  "errors"
  "fmt"
  "slices"
  "sync"
)

func fetch(name string) (string, error) {
  if name == "billing" {
    return "", errors.New("billing: 503 unavailable")
  }
  return name + ": ok", nil
}

func main() {
  names := []string{"users", "billing", "catalog"}

  results := make([]string, len(names))
  errCh := make(chan error, len(names))
  var wg sync.WaitGroup
  for i, n := range names {
    wg.Go(func() {
      r, err := fetch(n)
      if err != nil {
        errCh <- err
        return
      }
      results[i] = r
    })
  }
  wg.Wait()
  close(errCh)

  var firstErr error
  for err := range errCh {
    if firstErr == nil {
      firstErr = err
    }
  }
  got := slices.Clone(results)
  slices.Sort(got)
  fmt.Println("results:", got)
  fmt.Println("err:    ", firstErr)
}
```

Output:

```
results: [ catalog: ok users: ok]
err:     billing: 503 unavailable
```

Count what you had to get right: a buffered error channel sized to the goroutine count
(unbuffered deadlocks, under-sized blocks a goroutine forever), `close` it after `Wait`,
drain it for the first error, and write results by index to avoid the shared-slice race
from [11.5](05-race-detector.md). Twenty lines of ceremony around three lines of work —
and it still does not cancel the surviving fetches when `billing` fails.

## Group.Go and Wait

```go
package main

import (
  "errors"
  "fmt"
  "slices"

  "golang.org/x/sync/errgroup"
)

func fetch(name string) (string, error) {
  if name == "billing" {
    return "", errors.New("billing: 503 unavailable")
  }
  return name + ": ok", nil
}

func main() {
  names := []string{"users", "billing", "catalog"}
  results := make([]string, len(names))

  var g errgroup.Group
  for i, n := range names {
    g.Go(func() error {
      r, err := fetch(n)
      if err != nil {
        return err
      }
      results[i] = r
      return nil
    })
  }
  err := g.Wait()

  got := slices.Clone(results)
  slices.Sort(got)
  fmt.Println("results:", got)
  fmt.Println("err:    ", err)

  var empty errgroup.Group
  fmt.Println("Wait on an empty group:", empty.Wait())

  var ok errgroup.Group
  ok.Go(func() error { return nil })
  ok.Go(func() error { return nil })
  fmt.Println("Wait with no errors:", ok.Wait())
}
```

Output:

```
results: [ catalog: ok users: ok]
err:     billing: 503 unavailable
```

Same behaviour, the error channel gone. `Group`'s zero value is usable, `Go` takes a
`func() error`, and `Wait` blocks until every goroutine returns, then returns one error
or nil (nil immediately on an empty group). Note the empty first element in `results`:
index 1 was never written because `billing` failed. errgroup does not clean up partial
results — the `err != nil` branch should normally discard the whole batch.

## Only the first error survives

```go
package main

import (
  "errors"
  "fmt"
  "sync/atomic"
  "time"

  "golang.org/x/sync/errgroup"
)

var errFirst = errors.New("first failure")
var errSecond = errors.New("second failure")

func main() {
  var g errgroup.Group
  var ran atomic.Int64

  g.Go(func() error {
    ran.Add(1)
    return errFirst
  })
  g.Go(func() error {
    ran.Add(1)
    time.Sleep(50 * time.Millisecond)
    return errSecond
  })
  g.Go(func() error {
    ran.Add(1)
    time.Sleep(50 * time.Millisecond)
    return nil
  })

  err := g.Wait()
  fmt.Println("goroutines that ran:", ran.Load())
  fmt.Println("Wait returned:", err)
  fmt.Println("is errFirst: ", errors.Is(err, errFirst))
  fmt.Println("is errSecond:", errors.Is(err, errSecond))
  fmt.Println("Wait again:", g.Wait())
}
```

Output:

```
goroutines that ran: 3
Wait returned: first failure
is errFirst:  true
is errSecond: false
Wait again: first failure
```

All three goroutines ran to completion — errgroup interrupts nothing — but only
`errFirst` is returned. `errSecond` is **discarded silently**: no log line, no way to
recover it. "First" means first to *return* an error, which under real concurrency is a
race; it is deterministic here only because the other two sleep 50ms. `Wait` is
idempotent; calling `Go` after `Wait` returned is not allowed. For all the errors, see
the `errors.Join` section below.

## WithContext: the reason to use errgroup

`errgroup.WithContext(parent)` returns a group *and* a derived context. The first
goroutine to return a non-nil error cancels that context, so every sibling that selects
on it can stop immediately.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "slices"
  "sync"
  "time"

  "golang.org/x/sync/errgroup"
)

func fetch(ctx context.Context, name string, d time.Duration, fail bool) (string, error) {
  if fail {
    return "", fmt.Errorf("%s: 503 unavailable", name)
  }
  select {
  case <-time.After(d):
    return name + ": ok", nil
  case <-ctx.Done():
    return "", fmt.Errorf("%s: abandoned: %w", name, ctx.Err())
  }
}

func main() {
  g, ctx := errgroup.WithContext(context.Background())

  var mu sync.Mutex
  var outcomes []string
  record := func(s string) {
    mu.Lock()
    outcomes = append(outcomes, s)
    mu.Unlock()
  }

  g.Go(func() error {
    _, err := fetch(ctx, "billing", 0, true)
    record("billing returned an error")
    return err
  })
  for _, n := range []string{"users", "catalog"} {
    g.Go(func() error {
      _, err := fetch(ctx, n, time.Hour, false)
      record(fmt.Sprintf("%s: %v", n, err))
      return err
    })
  }

  err := g.Wait()
  slices.Sort(outcomes)
  for _, o := range outcomes {
    fmt.Println(o)
  }
  fmt.Println("Wait returned:", err)
  fmt.Println("ctx after Wait:", ctx.Err())
  fmt.Println("cancelled by Canceled, not DeadlineExceeded:", errors.Is(ctx.Err(), context.Canceled))

  g2, ctx2 := errgroup.WithContext(context.Background())
  g2.Go(func() error { return nil })
  fmt.Println("all-success Wait:", g2.Wait())
  fmt.Println("ctx cancelled anyway when Wait returns:", ctx2.Err())
}
```

Output:

```
billing returned an error
catalog: catalog: abandoned: context canceled
users: users: abandoned: context canceled
Wait returned: billing: 503 unavailable
ctx after Wait: context canceled
cancelled by Canceled, not DeadlineExceeded: true
all-success Wait: <nil>
ctx cancelled anyway when Wait returns: context canceled
```

`users` and `catalog` were each waiting an *hour*. `billing` failed instantly, and both
of them returned within microseconds with `context canceled`. Without this, the group
would block for an hour on work whose result is already being thrown away.

Three details the run pins down. Siblings see `context.Canceled`, not
`DeadlineExceeded`, so a sibling failure is a cancellation, not a timeout
([11.6](06-context-cancel.md)). `Wait` cancels the context on the way out **even when
nothing failed**, so never use the returned context for work that outlives `Wait`. And
errgroup uses `WithCancelCause` internally, so `context.Cause(ctx)` gives you the failing
goroutine's error — handy for logging inside a sibling.

The contract this creates: every goroutine in a `WithContext` group should accept the
group's `ctx` and select on `ctx.Done()` at every blocking point. A goroutine that
ignores it keeps running, and `Wait` keeps waiting for it.

## SetLimit and TryGo

`SetLimit(n)` caps the number of goroutines running at once. Call it before the first
`Go`, or it panics.

```go
package main

import (
  "fmt"
  "sync/atomic"
  "time"

  "golang.org/x/sync/errgroup"
)

func main() {
  var g errgroup.Group
  g.SetLimit(3)

  var running, peak atomic.Int64
  for range 20 {
    g.Go(func() error {
      n := running.Add(1)
      for {
        p := peak.Load()
        if n <= p || peak.CompareAndSwap(p, n) {
          break
        }
      }
      time.Sleep(5 * time.Millisecond)
      running.Add(-1)
      return nil
    })
  }
  fmt.Println("Wait:", g.Wait())
  fmt.Println("peak concurrency never exceeded 3:", peak.Load() <= 3)
  fmt.Println("peak:", peak.Load())

  var t errgroup.Group
  t.SetLimit(2)
  block := make(chan struct{})
  accepted, rejected := 0, 0
  for range 5 {
    if t.TryGo(func() error { <-block; return nil }) {
      accepted++
    } else {
      rejected++
    }
  }
  fmt.Println("TryGo accepted:", accepted, "rejected:", rejected)
  close(block)
  fmt.Println("Wait:", t.Wait())

  var u errgroup.Group
  fmt.Println("TryGo with no limit always succeeds:", u.TryGo(func() error { return nil }))
  u.Wait()

  var n errgroup.Group
  n.SetLimit(-1)
  fmt.Println("SetLimit(-1) means unlimited; TryGo:", n.TryGo(func() error { return nil }))
  n.Wait()
}
```

Output:

```
Wait: <nil>
peak concurrency never exceeded 3: true
peak: 3
TryGo accepted: 2 rejected: 3
Wait: <nil>
TryGo with no limit always succeeds: true
SetLimit(-1) means unlimited; TryGo: true
```

Both verified on `x/sync v0.24.0`. `SetLimit(-1)` (or never calling it) means unlimited,
and `TryGo` on an unlimited group always succeeds.

The critical behaviour: **`Go` blocks when the limit is reached.** It does not queue, it
does not spawn anyway — the calling goroutine waits for a slot.

```go
package main

import (
  "fmt"
  "time"

  "golang.org/x/sync/errgroup"
)

func main() {
  var g errgroup.Group
  g.SetLimit(2)

  start := time.Now()
  for i := range 6 {
    g.Go(func() error {
      time.Sleep(40 * time.Millisecond)
      return nil
    })
    fmt.Printf("Go(%d) returned at %v\n", i, time.Since(start).Round(20*time.Millisecond))
  }
  g.Wait()
  fmt.Println("total:", time.Since(start).Round(20*time.Millisecond))
}
```

Output — reproducible across runs because every job sleeps exactly 40ms:

```
Go(0) returned at 0s
Go(1) returned at 0s
Go(2) returned at 40ms
Go(3) returned at 40ms
Go(4) returned at 80ms
Go(5) returned at 80ms
total: 120ms
```

Six jobs, limit 2, 40ms each: three waves, 120ms total, with `Go` returning late for
jobs 2-5 because it was waiting. So `Go` is a *backpressure* point — the submitting loop
paces itself, which is what makes errgroup a usable worker pool — and calling `Go` from
inside a group goroutine can deadlock, since all N slots can end up held by waiters.

`TryGo` is the non-blocking alternative, returning false instead of waiting: a shed-load
policy ("drop it rather than queue it"). The rejected work is simply *not run*, so a
false return is your signal to handle it some other way.

## Panics are not recovered

```go
package main

import (
  "fmt"

  "golang.org/x/sync/errgroup"
)

func main() {
  var g errgroup.Group
  g.Go(func() error {
    var m map[string]int
    m["boom"] = 1
    return nil
  })
  fmt.Println("Wait:", g.Wait())
}
```

This program crashes on purpose. Output:

```
panic: assignment to entry in nil map

goroutine 19 [running]:
main.main.func1()
	/tmp/scratch/main.go:13 +0x34
golang.org/x/sync/errgroup.(*Group).Go.func1()
	/Users/raj/go/pkg/mod/golang.org/x/sync@v0.24.0/errgroup/errgroup.go:93 +0x4c
created by golang.org/x/sync/errgroup.(*Group).Go in goroutine 1
	/Users/raj/go/pkg/mod/golang.org/x/sync@v0.24.0/errgroup/errgroup.go:78 +0x98
exit status 2
```

The `Wait:` line never prints. `Wait` never returns, because the process is gone — a
panic in any goroutine terminates the whole program, and errgroup adds no `recover`. The
goroutine number varies per run.

This surprises people coming from `Promise.all`, where a throw becomes a rejection. If
your group runs code that might panic — user-supplied callbacks, a parser, anything
reflective — wrap it:

```go
package main

import (
  "errors"
  "fmt"

  "golang.org/x/sync/errgroup"
)

type PanicError struct {
  Value any
}

func (p PanicError) Error() string { return fmt.Sprintf("panic: %v", p.Value) }

func safeGo(g *errgroup.Group, f func() error) {
  g.Go(func() (err error) {
    defer func() {
      if r := recover(); r != nil {
        err = PanicError{Value: r}
      }
    }()
    return f()
  })
}

func main() {
  var g errgroup.Group
  safeGo(&g, func() error {
    var m map[string]int
    m["boom"] = 1
    return nil
  })
  safeGo(&g, func() error { return nil })

  err := g.Wait()
  fmt.Println("Wait:", err)
  var pe PanicError
  fmt.Println("errors.As PanicError:", errors.As(err, &pe))
  fmt.Println("recovered value:", pe.Value)
}
```

Output:

```
Wait: panic: assignment to entry in nil map
errors.As PanicError: true
recovered value: assignment to entry in nil map
```

The named return `err` is what makes this work: `recover()` runs after the body has
unwound, and assigning to the named result replaces the return value. Do it deliberately
rather than reflexively — a panic usually means a bug, and turning every bug into an
error hides it. The case for the wrapper is a request-scoped group where one bad request
should not kill the process.

## Collecting all errors instead of the first

errgroup will not do this. Return `nil` from every goroutine, collect the errors
yourself, and `errors.Join` them.

```go
package main

import (
  "errors"
  "fmt"
  "slices"
  "strings"
  "sync"

  "golang.org/x/sync/errgroup"
)

func check(name string) error {
  if name == "users" {
    return nil
  }
  return fmt.Errorf("%s: failed", name)
}

func main() {
  names := []string{"users", "billing", "catalog", "search"}

  var g errgroup.Group
  for _, n := range names {
    g.Go(func() error { return check(n) })
  }
  fmt.Println("errgroup reports exactly one error:", g.Wait() != nil)

  var mu sync.Mutex
  var all []error
  var g2 errgroup.Group
  for _, n := range names {
    g2.Go(func() error {
      if err := check(n); err != nil {
        mu.Lock()
        all = append(all, err)
        mu.Unlock()
      }
      return nil
    })
  }
  fmt.Println("g2.Wait:", g2.Wait())

  slices.SortFunc(all, func(a, b error) int { return strings.Compare(a.Error(), b.Error()) })
  joined := errors.Join(all...)
  fmt.Println("collected", len(all), "errors:")
  fmt.Println(joined)

  var multi interface{ Unwrap() []error }
  fmt.Println("Join result unwraps to a slice:", errors.As(joined, &multi))
  fmt.Println("unwrapped count:", len(multi.Unwrap()))
}
```

Output:

```
errgroup reports exactly one error: true
g2.Wait: <nil>
collected 3 errors:
billing: failed
catalog: failed
search: failed
Join result unwraps to a slice: true
unwrapped count: 3
```

Which specific error the first group returns varies per run, so the output only asserts
that there was one. The second group returns `nil` from every goroutine — that is the
whole trick — appends real errors under a mutex, sorts them for determinism, and joins
them. `errors.Join` renders one error per line and keeps every one reachable via
`errors.Is`, `errors.As`, and the `Unwrap() []error` interface.

The trade: you lose sibling cancellation, because errgroup only cancels on a non-nil
return. For both — all errors *and* cancellation — return a sentinel from the first
failure only, or cancel a `WithCancelCause` context yourself.

## errgroup as a worker pool

`SetLimit` plus indexed writes plus a `ctx.Done()` check is a complete bounded worker
pool in about fifteen lines.

```go
package main

import (
  "context"
  "fmt"
  "slices"
  "sync"

  "golang.org/x/sync/errgroup"
)

func process(id int) (int, error) {
  if id == 13 {
    return 0, fmt.Errorf("item %d: corrupt", id)
  }
  return id * id, nil
}

func indexed(ids []int) ([]int, error) {
  out := make([]int, len(ids))
  g, ctx := errgroup.WithContext(context.Background())
  g.SetLimit(4)
  for i, id := range ids {
    g.Go(func() error {
      select {
      case <-ctx.Done():
        return ctx.Err()
      default:
      }
      v, err := process(id)
      if err != nil {
        return err
      }
      out[i] = v
      return nil
    })
  }
  if err := g.Wait(); err != nil {
    return nil, err
  }
  return out, nil
}

func appendBroken(ids []int) []int {
  var out []int
  var g errgroup.Group
  g.SetLimit(4)
  for _, id := range ids {
    g.Go(func() error {
      v, err := process(id)
      if err != nil {
        return nil
      }
      out = append(out, v)
      return nil
    })
  }
  g.Wait()
  return out
}

func appendSafe(ids []int) []int {
  var mu sync.Mutex
  var out []int
  var g errgroup.Group
  g.SetLimit(4)
  for _, id := range ids {
    g.Go(func() error {
      v, err := process(id)
      if err != nil {
        return nil
      }
      mu.Lock()
      out = append(out, v)
      mu.Unlock()
      return nil
    })
  }
  g.Wait()
  slices.Sort(out)
  return out
}

func main() {
  ids := []int{1, 2, 3, 4, 5}
  out, err := indexed(ids)
  fmt.Println("indexed:", out, err)

  _, err = indexed([]int{1, 13, 3})
  fmt.Println("with a bad item:", err)

  fmt.Println("append without a lock, len:", len(appendBroken(ids)), "- and racy under -race")
  fmt.Println("append under a lock:", appendSafe(ids))
}
```

Output:

```
indexed: [1 4 9 16 25] <nil>
with a bad item: item 13: corrupt
append without a lock, len: 5 - and racy under -race
append under a lock: [1 4 9 16 25]
```

`indexed` is the pattern to copy: results by index into a preallocated slice (no lock,
order preserved), a `ctx.Done()` check so queued items skip their work after a failure,
and `SetLimit(4)` for backpressure. The channel-based pool with explicit workers and a
jobs channel — and when to prefer it — is
[12.1](../12-concurrency-patterns/01-worker-pools.md).

`appendBroken` is the mistake: `go run -race .` reports a data race on the slice header
in `appendBroken.func1`, even though it happened to produce 5 elements on this run.
`SetLimit` does not make appends safe — it bounds concurrency, it does not serialize.

The other classic mistake is the loop variable. On Go 1.22+ this is fine:


```go
for i, id := range ids {
  g.Go(func() error { out[i] = id * id; return nil })
}
```

On Go 1.21 and below, every goroutine captures the *same* `i` and `id` and the result is
garbage — a genuine data race, demonstrated under `-race` in
[11.5](05-race-detector.md). If your `go.mod` says `go 1.21` or lower, write
`i, id := i, id` inside the loop.

## An HTTP handler fanning out to backends

The canonical production use: a dashboard endpoint that needs three upstreams, in
parallel, where any failure should abandon the rest and return 502.

```go
package main

import (
  "context"
  "encoding/json"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
  "time"

  "golang.org/x/sync/errgroup"
)

type Dashboard struct {
  Profile string `json:"profile"`
  Orders  string `json:"orders"`
  Credits string `json:"credits"`
}

func backend(name string, delay time.Duration, status int) *httptest.Server {
  return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    select {
    case <-time.After(delay):
    case <-r.Context().Done():
      return
    }
    if status != http.StatusOK {
      w.WriteHeader(status)
      fmt.Fprintf(w, "%s is down", name)
      return
    }
    fmt.Fprintf(w, "%s-data", name)
  }))
}

func get(ctx context.Context, name, url string) (string, error) {
  req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
  if err != nil {
    return "", err
  }
  resp, err := http.DefaultClient.Do(req)
  if err != nil {
    return "", fmt.Errorf("%s: %w", name, err)
  }
  defer resp.Body.Close()
  b, err := io.ReadAll(resp.Body)
  if err != nil {
    return "", fmt.Errorf("%s: %w", name, err)
  }
  if resp.StatusCode != http.StatusOK {
    return "", fmt.Errorf("%s: status %d: %s", name, resp.StatusCode, b)
  }
  return string(b), nil
}

func handler(profileURL, ordersURL, creditsURL string) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    g, ctx := errgroup.WithContext(r.Context())
    var d Dashboard

    g.Go(func() (err error) { d.Profile, err = get(ctx, "profile", profileURL); return })
    g.Go(func() (err error) { d.Orders, err = get(ctx, "orders", ordersURL); return })
    g.Go(func() (err error) { d.Credits, err = get(ctx, "credits", creditsURL); return })

    if err := g.Wait(); err != nil {
      http.Error(w, "upstream failure: "+err.Error(), http.StatusBadGateway)
      return
    }
    json.NewEncoder(w).Encode(d)
  })
}

func main() {
  profile := backend("profile", 10*time.Millisecond, http.StatusOK)
  orders := backend("orders", 20*time.Millisecond, http.StatusOK)
  credits := backend("credits", 15*time.Millisecond, http.StatusOK)
  broken := backend("credits", 0, http.StatusInternalServerError)
  defer profile.Close()
  defer orders.Close()
  defer credits.Close()
  defer broken.Close()

  for _, c := range []struct{ label, credits string }{
    {"all healthy", credits.URL},
    {"credits down", broken.URL},
  } {
    srv := httptest.NewServer(handler(profile.URL, orders.URL, c.credits))
    start := time.Now()
    resp, err := http.Get(srv.URL)
    if err != nil {
      panic(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    srv.Close()
    fmt.Printf("%s: status=%d elapsed=%v\n", c.label, resp.StatusCode, time.Since(start).Round(10*time.Millisecond))
    fmt.Println("  " + strings.TrimSpace(string(b)))
  }
}
```

Output:

```
all healthy: status=200 elapsed=20ms
  {"profile":"profile-data","orders":"orders-data","credits":"credits-data"}
credits down: status=502 elapsed=0s
  upstream failure: credits: status 500: credits is down
```

Reproducible across runs. The 20ms is the proof of parallelism: the three backends take
10ms, 20ms and 15ms, so sequential would be 45ms and the max is 20ms. The failure case
returns in ~0ms rather than waiting out the 20ms `orders` call, because the group context
cancelled the in-flight requests — and `http.NewRequestWithContext` means that
cancellation really does abort the HTTP calls ([11.6](06-context-cancel.md)).

Two things make the struct writes safe without a lock: each goroutine writes a
**different field** (distinct memory, per [11.5](05-race-detector.md)), and nothing reads
`d` until after `Wait`. `go run -race .` confirms it; two goroutines on the *same* field,
or a read of `d` before `Wait`, would both race. The
`func() (err error) { d.X, err = get(...); return }` form is a neat idiom — a named
return assigns the field and propagates the error in one line.

## Choosing between the four

| Tool | Gives you | Does not give you | Use when |
|---|---|---|---|
| [`sync.WaitGroup`](01-waitgroup.md) | wait for N goroutines | errors, cancellation, limits | fire-and-forget work that cannot fail |
| `errgroup.Group` | wait + the first error | all errors, panic safety | N tasks, any failure aborts the batch |
| `errgroup` + `SetLimit` | the above + bounded concurrency | a queue, per-item retries | many tasks, limited resource (connections, API quota) |
| channels + `select` | streaming, pipelines, per-item control | error aggregation for free | results consumed as they arrive, or stages feeding stages |
| `semaphore.Weighted` | *weighted* limits (3 units for a big job) | waiting, error collection | heterogeneous task costs |

If the function can return an error, `errgroup` over `WaitGroup`. If results must be
processed as they arrive, channels. If tasks have different costs,
`golang.org/x/sync/semaphore`. `errgroup` is the default for request-scoped fan-out,
which is most of what a server does.

## Quick Reference

| API | Signature | Note |
|---|---|---|
| `errgroup.Group` | struct, zero value usable | no sibling cancellation without `WithContext` |
| `errgroup.WithContext(ctx)` | `(*Group, context.Context)` | ctx cancelled on first error **and** on `Wait` return |
| `g.Go(f)` | `func(func() error)` | **blocks** when `SetLimit` is reached |
| `g.TryGo(f)` | `func(func() error) bool` | false if no slot free; the work is not run |
| `g.Wait()` | `error` | the first non-nil error, or nil; idempotent |
| `g.SetLimit(n)` | `func(int)` | call before the first `Go`; `-1` = unlimited; panics if goroutines are active |

| Behaviour | Fact |
|---|---|
| multiple errors | only the first non-nil return is kept; the rest are discarded |
| `Wait` on an empty group | returns nil immediately |
| `Wait` called twice | same error both times |
| `Go` after `Wait` returned | not allowed; reuse is undefined |
| panic in a goroutine | **not recovered** — the process dies |
| context after `Wait` | always cancelled, even on full success |
| `context.Cause(groupCtx)` | the failing goroutine's error |

## Common Mistakes

**Using a bare `errgroup.Group` and expecting siblings to stop.** Only
`errgroup.WithContext` cancels; a plain `Group` lets every goroutine run to completion.

**Ignoring the group's context inside the goroutines.** Cancellation is cooperative;
without a `ctx.Done()` case at every blocking point, `Wait` waits out the slowest task.

**Assuming you get every error.** Only the first non-nil return survives; collect them
under a mutex and `errors.Join` if you need all of them.

**Expecting a panic to come back as an error.** It takes down the process; if the work
can panic, wrap the body in a `recover` that assigns a named `err` return.

**Appending to a shared slice from group goroutines.** `SetLimit` bounds concurrency
but does not serialize; write by index, or hold a mutex.

**Calling `SetLimit` after the first `Go`, or `Go` from inside a group goroutine.** The
first panics; the second deadlocks once every slot is held by a waiter.

## Exercises

1. Build `FetchAll(ctx, urls []string) ([]Result, error)` with `errgroup.WithContext`
   and `SetLimit(8)`, writing by index. Prove with `httptest` that one 500 cancels the
   rest, and that elapsed time is the slowest *surviving* request.
2. Write the same fan-out twice — `WaitGroup` + an error channel, then `errgroup` — and
   compare line count, allocations (`-benchmem`), and behaviour when the second of five
   tasks fails.
3. Implement `AllErrors(tasks []func() error) error` running every task with a limit of
   4, collecting *every* failure into an `errors.Join`. Make `errors.Is` work for each
   sentinel, and add sibling cancellation without losing any error.
4. Add panic recovery to an errgroup wrapper so a panicking task becomes an error
   carrying the recovered value *and* `debug.Stack()`. Verify the context still cancels
   siblings and that a second task's real error is not lost.
5. Compare `errgroup` + `SetLimit(4)` against `golang.org/x/sync/semaphore` with weights
   1-4 on a mix of cheap and expensive tasks. Report wall time and peak concurrency for
   both, and say which you would ship.

---

[← 11.7 context Values and Propagation](07-context-values.md) · [Course Index](../README.md) · [12.1 Worker Pools →](../12-concurrency-patterns/01-worker-pools.md)
