[← 12.3 Pipelines and Cancellation](03-pipelines.md) · [Course Index](../README.md) · [12.5 Deadlocks, Leaks, and How to Spot Them →](05-deadlocks-leaks.md)

# 12.4 — Rate Limiting and Semaphores

## What You'll Learn

- Concurrency limiting and rate limiting are different controls — most bugs here are conflating them
- A counting semaphore from a buffered channel, and `golang.org/x/sync/semaphore` for weighted acquires
- `golang.org/x/time/rate`: `Wait`, `Allow`, `Reserve`, and what `burst` actually means
- Retries with exponential backoff and jitter, written so a context can cancel mid-sleep

## Coming From Another Language

| Other | Go |
|---|---|
| JS `p-limit(5)` | buffered-channel semaphore, or `errgroup.SetLimit(5)` |
| JS `bottleneck`, `p-throttle` | `golang.org/x/time/rate` |
| Python `asyncio.Semaphore(5)` | `make(chan struct{}, 5)` or `semaphore.NewWeighted(5)` |
| Python `ratelimit` decorator | `limiter.Wait(ctx)` at the top of the function |
| Rust `tokio::sync::Semaphore` | `x/sync/semaphore` — same weighted-permit model, same `acquire_many` |

Both controls bound load, and they bound *different* load. A semaphore limits how many
operations are in flight **at once**; a rate limiter limits how many start **per unit
time**. Neither implies the other, and real clients usually need both — this lesson builds
each one, proves its bound with real measurements, then combines them.

## Concurrency limiting vs rate limiting

| | Concurrency limit | Rate limit |
|---|---|---|
| Bounds | simultaneous in-flight operations | operations started per second |
| Unit | a count (5 at a time) | a rate (5 per second) |
| Protects | your memory, FDs, connection pool | the remote service's quota |
| Tool | buffered channel, `x/sync/semaphore`, `errgroup.SetLimit` | `x/time/rate` |
| If the op is slow | fewer ops/sec as a side effect | unchanged — you just start more at once |
| If the op is fast | unbounded ops/sec | still capped |

The trap: "limit to 5 concurrent" does **not** limit your request rate. If each call takes
1ms, 5 concurrent is 5000 requests/second and you will be throttled. Conversely "limit to
20/sec" does not bound concurrency: if each call takes 10 seconds, you accumulate 200
in-flight calls. A 429-returning API needs a rate limiter; your own file descriptors need a
semaphore.

## A semaphore from a buffered channel

Acquire is a send, release is a receive, and the buffer size is the limit. An atomic
high-water mark proves the bound holds.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
  "time"
)

type gauge struct {
  cur atomic.Int64
  max atomic.Int64
}

func (g *gauge) enter() {
  cur := g.cur.Add(1)
  for {
    old := g.max.Load()
    if cur <= old || g.max.CompareAndSwap(old, cur) {
      return
    }
  }
}

func (g *gauge) leave() { g.cur.Add(-1) }

func main() {
  const limit = 3
  sem := make(chan struct{}, limit)

  var g gauge
  out := make([]int, 12)

  var wg sync.WaitGroup
  for i := 0; i < 12; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()

      sem <- struct{}{}
      defer func() { <-sem }()

      g.enter()
      defer g.leave()

      time.Sleep(20 * time.Millisecond)
      out[i] = i * i
    }()
  }
  wg.Wait()

  fmt.Println("results:", out)
  fmt.Println("max concurrent:", g.max.Load(), "limit:", limit)
  fmt.Println("in flight at end:", g.cur.Load())
}
```

Output:

```
results: [0 1 4 9 16 25 36 49 64 81 100 121]
max concurrent: 3 limit: 3
in flight at end: 0
```

Three details carry the whole pattern:

- `defer func() { <-sem }()` immediately after the acquire, so a panic still releases.
- The acquire is *inside* the goroutine here, so all 12 goroutines exist but only 3 work at
  a time. Put `sem <- struct{}{}` *before* the `go` (as 12.1 does) to bound goroutines
  too. Bounding the work is usually what you want; bounding goroutines matters when each
  carries a large stack or a held resource.
- `out[i] = i * i` writes to distinct indices, so `-race` is clean with no mutex.

Idiomatic: this, for a plain "at most N at once" with no weights. It needs no dependency
and is four lines. There is no `TryAcquire` though — for that, `select` with a `default`.

## x/sync/semaphore: weights and context

`golang.org/x/sync/semaphore` adds three things the channel cannot do: weighted acquires,
a context-aware `Acquire` that can be cancelled while waiting, and `TryAcquire`. Weights
are the real reason to reach for it — bound a *resource* like memory rather than a count.

Pinned version: `golang.org/x/sync v0.24.0`.

```go
package main

import (
  "context"
  "fmt"
  "sort"
  "sync"
  "sync/atomic"
  "time"

  "golang.org/x/sync/semaphore"
)

type task struct {
  name string
  mib  int64
}

func main() {
  const budgetMiB = 100
  sem := semaphore.NewWeighted(budgetMiB)
  ctx := context.Background()

  tasks := []task{
    {"thumbnail-a", 10}, {"thumbnail-b", 10}, {"thumbnail-c", 10},
    {"video-a", 60}, {"video-b", 60},
    {"thumbnail-d", 10}, {"thumbnail-e", 10},
  }

  var inUse, maxUse atomic.Int64
  var mu sync.Mutex
  var order []string

  var wg sync.WaitGroup
  for _, t := range tasks {
    wg.Add(1)
    go func() {
      defer wg.Done()
      if err := sem.Acquire(ctx, t.mib); err != nil {
        return
      }
      defer sem.Release(t.mib)

      cur := inUse.Add(t.mib)
      for {
        old := maxUse.Load()
        if cur <= old || maxUse.CompareAndSwap(old, cur) {
          break
        }
      }
      time.Sleep(20 * time.Millisecond)
      inUse.Add(-t.mib)

      mu.Lock()
      order = append(order, t.name)
      mu.Unlock()
    }()
  }
  wg.Wait()

  sort.Strings(order)
  fmt.Println("completed:", order)
  fmt.Println("peak <= budget:", maxUse.Load() <= budgetMiB, "- peak never exceeds", budgetMiB, "MiB")
  fmt.Println("held at end:", inUse.Load())

  if sem.TryAcquire(budgetMiB) {
    fmt.Println("TryAcquire(100) after all released: true")
    sem.Release(budgetMiB)
  }
  if !sem.TryAcquire(budgetMiB + 1) {
    fmt.Println("TryAcquire(101) exceeds the budget: false")
  }

  sem.Acquire(ctx, budgetMiB)
  tctx, cancel := context.WithTimeout(context.Background(), 30*time.Millisecond)
  defer cancel()
  err := sem.Acquire(tctx, 1)
  fmt.Println("Acquire(1) while full, 30ms timeout:", err)
  sem.Release(budgetMiB)
}
```

Output:

```
completed: [thumbnail-a thumbnail-b thumbnail-c thumbnail-d thumbnail-e video-a video-b]
peak <= budget: true - peak never exceeds 100 MiB
held at end: 0
TryAcquire(100) after all released: true
TryAcquire(101) exceeds the budget: false
Acquire(1) while full, 30ms timeout: context deadline exceeded
```

The peak itself varies run to run (90 or 100 MiB depending on which tasks win the
semaphore first), so the program asserts the invariant instead of printing the figure. Two
60 MiB videos can never run together; five 10 MiB thumbnails can.

`Acquire` returns `ctx.Err()` if the context is done before a permit frees up — that is
the behaviour a channel semaphore cannot give you without wrapping the send in a `select`.
Note that `Acquire(n)` where `n` exceeds the total weight blocks forever rather than
erroring, so validate your weights.

## A naive rate limiter from a ticker

Receiving from a ticker before each call paces them exactly.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  tick := time.NewTicker(50 * time.Millisecond)
  defer tick.Stop()

  start := time.Now()
  for i := 1; i <= 5; i++ {
    <-tick.C
    fmt.Printf("request %d at %v\n", i, time.Since(start).Round(10*time.Millisecond))
  }
}
```

Output:

```
request 1 at 50ms
request 2 at 100ms
request 3 at 150ms
request 4 at 200ms
request 5 at 250ms
```

Correct and dependency-free, with three real limitations: no burst (the first request waits
a full tick even if the system has been idle for an hour), no way to ask "would this be
allowed?" without blocking, and no context — you cannot abandon the wait. `time.Tick`
(rather than `NewTicker`) is worse still: it never releases its ticker, which is a leak
(12.5). Use `NewTicker` with `defer Stop()`.

## x/time/rate: the token bucket

`golang.org/x/time/rate` implements a token bucket: tokens refill at `r` per second into a
bucket of capacity `burst`, and each operation takes one. An idle limiter accumulates up to
`burst` tokens, so a quiet period buys you a burst of `burst` immediate calls, after which
you are paced at exactly `r`. That single sentence is the whole model.

Pinned version: `golang.org/x/time v0.16.0`.

```go
package main

import (
  "context"
  "fmt"
  "time"

  "golang.org/x/time/rate"
)

func main() {
  fmt.Println("Wait, 20/s with burst 3:")
  lim := rate.NewLimiter(20, 3)
  ctx := context.Background()
  start := time.Now()
  for i := 1; i <= 6; i++ {
    lim.Wait(ctx)
    fmt.Printf("  call %d at %v\n", i, time.Since(start).Round(10*time.Millisecond))
  }

  fmt.Println("Allow, 20/s with burst 3, called 5 times immediately:")
  lim2 := rate.NewLimiter(20, 3)
  for i := 1; i <= 5; i++ {
    fmt.Printf("  call %d allowed: %v\n", i, lim2.Allow())
  }

  fmt.Println("Reserve tells you how long you would wait:")
  lim3 := rate.NewLimiter(20, 1)
  r1 := lim3.Reserve()
  fmt.Printf("  first  reservation ok=%v delay=%v\n", r1.OK(), r1.Delay().Round(10*time.Millisecond))
  r2 := lim3.Reserve()
  fmt.Printf("  second reservation ok=%v delay=%v\n", r2.OK(), r2.Delay().Round(10*time.Millisecond))
  r2.Cancel()

  fmt.Println("Limit and Burst:", lim.Limit(), lim.Burst())
}
```

Output:

```
Wait, 20/s with burst 3:
  call 1 at 0s
  call 2 at 0s
  call 3 at 0s
  call 4 at 50ms
  call 5 at 100ms
  call 6 at 150ms
Allow, 20/s with burst 3, called 5 times immediately:
  call 1 allowed: true
  call 2 allowed: true
  call 3 allowed: true
  call 4 allowed: false
  call 5 allowed: false
Reserve tells you how long you would wait:
  first  reservation ok=true delay=0s
  second reservation ok=true delay=50ms
Limit and Burst: 20 3
```

Calls 1-3 are free — that is the burst draining. From call 4 on, 20/s means one per 50ms
exactly. The three methods differ only in what they do when no token is available:

| Method | No token available | Use for |
|---|---|---|
| `Wait(ctx)` | blocks until one, or returns `ctx.Err()` | your own outbound calls — you want the work done |
| `Allow()` | returns `false` immediately | inbound requests — reject with 429 |
| `Reserve()` | reserves a future token, `Delay()` says when | you need to decide based on the wait; `Cancel()` returns it |

`Reserve` without `Cancel` on an abandoned path consumes the token anyway, which silently
lowers your effective rate.

## Combining both limits over real HTTP

A throttled client needs the rate limiter and the semaphore, in that order: pace the
starts, then bound the overlap. The handler sleeps 120ms, longer than the 50ms pacing, so
both limits actually bind.

```go
package main

import (
  "context"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "sort"
  "sync"
  "sync/atomic"
  "time"

  "golang.org/x/time/rate"
)

func main() {
  var inFlight, maxInFlight atomic.Int64
  var mu sync.Mutex
  var arrivals []time.Duration
  var srvStart time.Time

  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    cur := inFlight.Add(1)
    for {
      old := maxInFlight.Load()
      if cur <= old || maxInFlight.CompareAndSwap(old, cur) {
        break
      }
    }
    mu.Lock()
    arrivals = append(arrivals, time.Since(srvStart))
    mu.Unlock()

    time.Sleep(120 * time.Millisecond)
    inFlight.Add(-1)
    fmt.Fprintf(w, "ok %s", r.URL.Path)
  }))
  defer srv.Close()

  lim := rate.NewLimiter(20, 1)
  sem := make(chan struct{}, 2)
  ctx := context.Background()

  srvStart = time.Now()
  var wg sync.WaitGroup
  bodies := make([]string, 8)
  for i := 0; i < 8; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()

      if err := lim.Wait(ctx); err != nil {
        return
      }
      sem <- struct{}{}
      defer func() { <-sem }()

      resp, err := http.Get(fmt.Sprintf("%s/r%d", srv.URL, i))
      if err != nil {
        bodies[i] = "error: " + err.Error()
        return
      }
      defer resp.Body.Close()
      b, _ := io.ReadAll(resp.Body)
      bodies[i] = string(b)
    }()
  }
  wg.Wait()

  sort.Strings(bodies)
  fmt.Println("responses:", bodies)
  fmt.Println("max concurrent at the server:", maxInFlight.Load(), "<= semaphore 2")

  mu.Lock()
  sort.Slice(arrivals, func(i, j int) bool { return arrivals[i] < arrivals[j] })
  minGap := time.Hour
  for i := 1; i < len(arrivals); i++ {
    if g := arrivals[i] - arrivals[i-1]; g < minGap {
      minGap = g
    }
  }
  span := arrivals[len(arrivals)-1]
  mu.Unlock()

  fmt.Println("requests:", len(arrivals))
  fmt.Println("smallest gap between arrivals >= 45ms:", minGap >= 45*time.Millisecond)
  fmt.Println("8 requests at 20/s span >= 350ms:", span >= 350*time.Millisecond)
}
```

Output:

```
responses: [ok /r0 ok /r1 ok /r2 ok /r3 ok /r4 ok /r5 ok /r6 ok /r7]
max concurrent at the server: 2 <= semaphore 2
requests: 8
smallest gap between arrivals >= 45ms: true
8 requests at 20/s span >= 350ms: true
```

A real `httptest.NewServer`, so these are real HTTP round-trips over loopback, measured at
the *server*. Both bounds are asserted rather than printed because the exact arrival
timestamps jitter by a few milliseconds.

Order matters: `lim.Wait(ctx)` before `sem <- struct{}{}`. The other way round, a goroutine
holds a concurrency slot while waiting out its rate delay, so your effective concurrency
collapses to the rate. Acquire the slow, fair control first.

## Per-client rate limiting

For inbound requests, one limiter per API key in a mutex-guarded map.

```go
package main

import (
  "fmt"
  "sync"

  "golang.org/x/time/rate"
)

type perClient struct {
  mu    sync.Mutex
  lims  map[string]*rate.Limiter
  r     rate.Limit
  burst int
}

func newPerClient(r rate.Limit, burst int) *perClient {
  return &perClient{lims: map[string]*rate.Limiter{}, r: r, burst: burst}
}

func (p *perClient) get(key string) *rate.Limiter {
  p.mu.Lock()
  defer p.mu.Unlock()
  l, ok := p.lims[key]
  if !ok {
    l = rate.NewLimiter(p.r, p.burst)
    p.lims[key] = l
  }
  return l
}

func main() {
  p := newPerClient(10, 2)

  for _, client := range []string{"alice", "bob"} {
    for i := 1; i <= 4; i++ {
      fmt.Printf("%s request %d allowed: %v\n", client, i, p.get(client).Allow())
    }
  }
  fmt.Println("limiters tracked:", len(p.lims))
}
```

Output:

```
alice request 1 allowed: true
alice request 2 allowed: true
alice request 3 allowed: false
alice request 4 allowed: false
bob request 1 allowed: true
bob request 2 allowed: true
bob request 3 allowed: false
bob request 4 allowed: false
limiters tracked: 2
```

Alice's traffic does not consume Bob's budget. The missing piece is eviction: this map
grows forever, one entry per key ever seen, which is an unbounded memory leak on a public
endpoint — store a `lastSeen time.Time` alongside each limiter and sweep idle entries, or
use an LRU.

## Retries with exponential backoff and jitter

Backoff doubles the wait per attempt; jitter randomises it so N clients retrying after the
same outage do not synchronise into a thundering herd. The sleep must be a `select` on
`ctx.Done()`, or a cancelled request keeps sleeping.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "math/rand"
  "net/http"
  "net/http/httptest"
  "sync/atomic"
  "time"
)

func retry(ctx context.Context, attempts int, base, maxBackoff time.Duration, rng *rand.Rand, f func(int) error) error {
  var last error
  for a := 0; a < attempts; a++ {
    if err := f(a); err == nil {
      return nil
    } else {
      last = err
    }
    if a == attempts-1 {
      break
    }

    backoff := base << a
    if backoff > maxBackoff {
      backoff = maxBackoff
    }
    sleep := time.Duration(rng.Int63n(int64(backoff)))

    t := time.NewTimer(sleep)
    select {
    case <-t.C:
    case <-ctx.Done():
      t.Stop()
      return errors.Join(last, ctx.Err())
    }
  }
  return fmt.Errorf("after %d attempts: %w", attempts, last)
}

func main() {
  var hits atomic.Int64
  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if hits.Add(1) < 3 {
      http.Error(w, "unavailable", http.StatusServiceUnavailable)
      return
    }
    fmt.Fprint(w, "ok")
  }))
  defer srv.Close()

  rng := rand.New(rand.NewSource(1))
  ctx := context.Background()

  start := time.Now()
  err := retry(ctx, 5, 20*time.Millisecond, 500*time.Millisecond, rng, func(a int) error {
    resp, err := http.Get(srv.URL)
    if err != nil {
      return err
    }
    defer resp.Body.Close()
    fmt.Printf("  attempt %d -> %s at %v\n", a, resp.Status, time.Since(start).Round(5*time.Millisecond))
    if resp.StatusCode >= 500 {
      return fmt.Errorf("status %s", resp.Status)
    }
    return nil
  })
  fmt.Println("result:", err, "- server hits:", hits.Load())

  fmt.Println("all attempts fail, context cancelled mid-backoff:")
  cctx, cancel := context.WithTimeout(context.Background(), 60*time.Millisecond)
  defer cancel()
  err = retry(cctx, 10, 50*time.Millisecond, time.Second, rand.New(rand.NewSource(2)), func(a int) error {
    return errors.New("always down")
  })
  fmt.Println("  err:", err)
  fmt.Println("  is context.DeadlineExceeded:", errors.Is(err, context.DeadlineExceeded))
}
```

Output:

```
  attempt 0 -> 503 Service Unavailable at 0s
  attempt 1 -> 503 Service Unavailable at 10ms
  attempt 2 -> 200 OK at 15ms
result: <nil> - server hits: 3
all attempts fail, context cancelled mid-backoff:
  err: always down
context deadline exceeded
  is context.DeadlineExceeded: true
```

The RNG is seeded with a constant so the timings above are reproducible; in production seed
it properly or use the global `rand` functions. The pieces that matter:

- `base << a` doubles per attempt, clamped at `maxBackoff`, so you cannot sleep for an hour.
- `rng.Int63n(int64(backoff))` is **full jitter** — uniform in `[0, backoff)`. This is the
  variant AWS measured as best for reducing contention. "Equal jitter"
  (`backoff/2 + rand(backoff/2)`) is the other reasonable choice.
- `time.NewTimer` plus `select`, with `t.Stop()` on the cancel path. A bare `time.Sleep`
  cannot be cancelled and would hold the goroutine for the full backoff.
- Only the last attempt's error is wrapped, with the attempt count, so `errors.Is` still
  works through it (8.3).

**Retry only idempotent operations, and only retryable failures.** A 400 will fail
identically five times; retrying a non-idempotent POST can double-charge someone. In Go
that means checking the status code or `errors.Is` before deciding to retry, not retrying
every non-nil error.

A **circuit breaker** is the next control up — after N consecutive failures, stop calling
for a cooldown so you stop hammering a service that is already down.
`github.com/sony/gobreaker` is the usual choice; the logic is a state machine over a
failure counter, not something to hand-roll.

## Quick Reference

| Tool | Signature | Bounds |
|---|---|---|
| buffered channel | `sem := make(chan struct{}, n)` · `sem <- struct{}{}` / `<-sem` | concurrency, unweighted |
| `x/sync/semaphore` | `semaphore.NewWeighted(n)` · `Acquire(ctx, w)` / `Release(w)` / `TryAcquire(w)` | concurrency, weighted, cancellable |
| `errgroup.SetLimit` | `g.SetLimit(n)` then `g.Go(f)` | concurrency over a fixed task set |
| `x/time/rate` | `rate.NewLimiter(r, burst)` · `Wait(ctx)` / `Allow()` / `Reserve()` | rate, with burst |
| `time.NewTicker` | `<-tick.C` before each op | rate, no burst, no context |

| Situation | Reach for |
|---|---|
| at most N of a fixed slice of tasks, first error aborts | `errgroup.SetLimit(n)` |
| at most N at once, no weights, no deps | buffered-channel semaphore |
| bound memory/CPU where tasks cost differently | `semaphore.NewWeighted` + weights |
| respect a remote API's requests/second | `rate.NewLimiter(r, burst)` + `Wait` |
| reject inbound traffic over quota | `rate.Limiter.Allow()` → 429 |
| per-tenant quotas | map of limiters + eviction |
| a flaky remote | retry with backoff + full jitter, idempotent ops only |
| a remote that is down | circuit breaker (`sony/gobreaker`) |

## Common Mistakes

**Using a concurrency limit to protect a rate-limited API.** Five concurrent 1ms calls is
5000/sec. You need `x/time/rate`; the semaphore bounds something else entirely.

**Acquiring the semaphore before waiting on the rate limiter.** The goroutine holds a slot
while it sleeps out its rate delay, so concurrency collapses to the rate. Rate limiter
first.

**Releasing a semaphore without `defer`.** Any early `return` or panic on the path leaks a
permit permanently, and the limit silently ratchets down to zero.

**`time.Sleep(backoff)` in a retry loop.** It cannot be cancelled, so a cancelled request
still occupies a goroutine for the full backoff. Use `time.NewTimer` in a `select` with
`ctx.Done()`.

**A per-client limiter map with no eviction.** One entry per key ever seen, forever. Track
`lastSeen` and sweep, or use an LRU.

**Retrying without jitter.** Every client that saw the same outage retries at the same
instant and re-creates the overload you were backing off from.

## Exercises

1. Replace the buffered-channel semaphore with a `TryAcquire`-style non-blocking version
   using `select` with `default`, and report how many of 12 tasks were skipped rather than
   queued.
2. Swap the order of `lim.Wait(ctx)` and `sem <- struct{}{}` in the HTTP example and measure
   the total wall time of all 8 requests both ways. Explain the difference from the limits
   alone.
3. Add eviction to `perClient`: store `lastSeen`, run a sweeper goroutine that drops
   limiters idle for more than 1 minute, and prove with `len(p.lims)` that it shrinks.
   Make sure the sweeper can be stopped.
4. Write a `rate.Limiter`-based `http.Handler` middleware that returns 429 with a
   `Retry-After` header computed from `Reserve().Delay()`, and test it with `httptest`.
   Remember to `Cancel()` the reservation you reject.
5. Measure the thundering herd: 50 goroutines retrying a failing endpoint, once with full
   jitter and once with none. Record the arrival timestamps at an `httptest` server and
   compare the peak concurrent arrivals in each case.

---

[← 12.3 Pipelines and Cancellation](03-pipelines.md) · [Course Index](../README.md) · [12.5 Deadlocks, Leaks, and How to Spot Them →](05-deadlocks-leaks.md)
