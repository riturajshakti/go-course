[← 13.4 time: Durations, Instants, and Formatting](04-time.md) · [Course Index](../README.md) · [13.6 encoding/json: Marshal and Unmarshal →](06-json.md)

# 13.5 — time: Timers, Tickers, and After

## What You'll Learn

- `Sleep`, `After`, `NewTimer`, `NewTicker`, `Tick`, `AfterFunc` and when each is right
- That the classic "`time.After` leaks in a loop" advice is **obsolete** on modern Go for a discarded timer — measured here, not recalled
- That ticks are **dropped** when the receiver is slow, demonstrated with real counts
- Debounce and throttle from scratch, and why `context.WithTimeout` beats both for cancellation

## Coming From Another Language

| Other | Go |
|---|---|
| JS `setTimeout(f, 1000)` | `time.AfterFunc(time.Second, f)` — runs in its own goroutine |
| JS `setInterval` / `clearInterval` | `time.NewTicker` / `tk.Stop()` |
| JS timers queued on one event loop | each timer fires on a real goroutine; handlers can overlap |
| Python `threading.Timer` | `time.AfterFunc`, far cheaper |
| Rust `tokio::time::sleep(..).await` | `<-time.After(d)` in a `select` |
| `await Promise.race([work, timeout])` | `select { case <-work: ; case <-time.After(d): }` |

The Go-specific shape here is that a timer is a **channel**, so a timeout is not a special
construct but an ordinary `select` case. That composes with everything: a timeout, a
cancellation and a result are three cases in one `select`, with no callback nesting. The
part worth measuring rather than memorizing is the lifecycle. Go 1.23 changed timers so
that an unreferenced timer becomes garbage-collectable and timer channels became
unbuffered, which invalidated a decade of "always `Stop()` your `time.After`" advice. The
numbers below are from this machine on go1.27.1, not from the old folklore.

## Sleep, After, and the select timeout

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  fmt.Println("--- time.Sleep ---")
  start := time.Now()
  time.Sleep(20 * time.Millisecond)
  fmt.Println("slept at least 20ms:", time.Since(start) >= 20*time.Millisecond)
  start = time.Now()
  time.Sleep(-1)
  fmt.Println("negative Sleep returns immediately:", time.Since(start) < 5*time.Millisecond)

  fmt.Println("\n--- time.After in a select ---")
  done := make(chan string)
  go func() {
    time.Sleep(10 * time.Millisecond)
    done <- "work finished"
  }()
  select {
  case msg := <-done:
    fmt.Println("got:", msg)
  case <-time.After(200 * time.Millisecond):
    fmt.Println("timed out")
  }

  slow := make(chan string)
  select {
  case msg := <-slow:
    fmt.Println("got:", msg)
  case <-time.After(20 * time.Millisecond):
    fmt.Println("timed out as expected")
  }

  fmt.Println("\n--- NewTimer ---")
  t := time.NewTimer(20 * time.Millisecond)
  fmt.Println("channel cap:", cap(t.C))
  v := <-t.C
  fmt.Println("timer fired, value is a time.Time:", !v.IsZero())
  fmt.Println("Stop after firing:", t.Stop())

  fmt.Println("\n--- Reset ---")
  t2 := time.NewTimer(time.Hour)
  fmt.Println("Stop before firing:", t2.Stop())
  fmt.Println("Reset to 20ms:", t2.Reset(20*time.Millisecond))
  <-t2.C
  fmt.Println("fired after Reset: true")

  t3 := time.NewTimer(10 * time.Millisecond)
  time.Sleep(30 * time.Millisecond)
  fmt.Println("Reset on an already-fired timer:", t3.Reset(20*time.Millisecond))
  <-t3.C
  fmt.Println("fired again: true")
}
```

This program asserts about timings rather than printing them, since the exact durations
vary per run. Output:

```
--- time.Sleep ---
slept at least 20ms: true
negative Sleep returns immediately: true

--- time.After in a select ---
got: work finished
timed out as expected

--- NewTimer ---
channel cap: 0
timer fired, value is a time.Time: true
Stop after firing: false

--- Reset ---
Stop before firing: true
Reset to 20ms: false
fired after Reset: true
Reset on an already-fired timer: true
fired again: true
```

| Call | Returns | Stoppable | Fires |
|---|---|---|---|
| `Sleep(d)` | nothing | no | blocks the goroutine |
| `After(d)` | `<-chan Time` | **no handle** | once |
| `NewTimer(d)` | `*Timer` (has `.C`) | `Stop`, `Reset` | once |
| `AfterFunc(d, f)` | `*Timer` (no usable `.C`) | `Stop`, `Reset` | once, in a new goroutine |
| `Tick(d)` | `<-chan Time` | **no handle** | repeatedly, forever |
| `NewTicker(d)` | `*Ticker` (has `.C`) | `Stop`, `Reset` | repeatedly |

`cap(t.C)` is **0**: timer channels are unbuffered as of Go 1.23. Before that they held one
element, which is what made stale-value draining necessary.

`time.Sleep` with a non-positive duration returns immediately rather than blocking or
panicking. `time.Sleep` is also the one timer call that is not cancellable — in any code
that might need to stop early, use `select` with `time.After` or a context instead.

Idiomatic: `time.After` in a `select`, `time.Sleep` only in tests and `main`.

## Stop and Reset, and what their return values mean

`Stop` and `Reset` both report whether the timer was **active** (scheduled but not yet
fired) at the moment of the call. That is subtler than "did it work".

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  t1 := time.AfterFunc(time.Hour, func() {})
  fmt.Println("Stop before firing:", t1.Stop())
  fmt.Println("Stop again:        ", t1.Stop())

  t2 := time.AfterFunc(5*time.Millisecond, func() {})
  time.Sleep(40 * time.Millisecond)
  fmt.Println("Stop after firing: ", t2.Stop())
}
```

Output:

```
Stop before firing: true
Stop again:         false
Stop after firing:  false
```

For `AfterFunc` this is genuinely useful: `Stop() == false` means the function already ran
(or is running), `true` means you prevented it. That is how you implement "cancel unless it
already happened" without a flag of your own.

### The old drain idiom is no longer needed

Pre-1.23, resetting a fired timer could leave a stale value buffered in `t.C`, and the
canonical workaround was `if !t.Stop() { <-t.C }`. On Go 1.23+ both `Stop` and `Reset`
clear any pending value, so that is unnecessary:

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  fmt.Println("--- Reset drains a pending value ---")
  t := time.NewTimer(5 * time.Millisecond)
  time.Sleep(30 * time.Millisecond)
  fmt.Println("timer has fired; len(t.C):", len(t.C), "cap:", cap(t.C))

  t.Reset(time.Hour)
  select {
  case v := <-t.C:
    fmt.Println("STALE VALUE received immediately:", v)
  case <-time.After(30 * time.Millisecond):
    fmt.Println("no stale value: Reset cleared the channel")
  }

  fmt.Println("\n--- Stop also clears a pending value ---")
  t3 := time.NewTimer(5 * time.Millisecond)
  time.Sleep(30 * time.Millisecond)
  t3.Stop()
  select {
  case <-t3.C:
    fmt.Println("value still present after Stop")
  default:
    fmt.Println("no value after Stop")
  }
}
```

Output:

```
--- Reset drains a pending value ---
timer has fired; len(t.C): 0 cap: 0
no stale value: Reset cleared the channel

--- Stop also clears a pending value ---
no value after Stop
```

The correct Reset protocol on a modern toolchain is therefore just:

```go
t.Reset(d)
```

Two caveats remain. `Reset` is still only safe to call from the goroutine that receives
from `t.C`, or while you know no other goroutine is in a `<-t.C`; the channel is
synchronized but your logic about "which deadline am I waiting on" is not. And if you must
support Go 1.22 or older, keep the drain, because there the stale value is real.

One measured detail on go1.27.1 worth knowing: `Stop()` on a `NewTimer` whose value you
already *received* returns `false` (the `NewTimer` section above), but `Stop()` on one that
fired and was **never received** returns `true` — reproducibly, five times out of five.
Because the unbuffered channel means an undelivered firing is still "pending", `Stop`
genuinely did cancel something. So do not read `Stop() == false` as "it already fired" for
a plain `NewTimer`; that inference is only reliable for `AfterFunc`, where it means the
callback ran.

## Does time.After leak in a loop?

The advice you will find everywhere is that `time.After` in a loop leaks, because the
timer stays in the runtime heap until it fires. Since Go 1.23 an unreferenced timer is
garbage-collectable. Here is the measurement rather than the claim:

```go
package main

import (
  "fmt"
  "runtime"
  "time"
)

func heapKB() uint64 {
  runtime.GC()
  runtime.GC()
  var ms runtime.MemStats
  runtime.ReadMemStats(&ms)
  return ms.HeapAlloc / 1024
}

func main() {
  const n = 200000

  base := heapKB()
  fmt.Printf("baseline heap: %d KB\n", base)

  for i := 0; i < n; i++ {
    _ = time.After(time.Hour)
  }
  discarded := heapKB()
  fmt.Printf("after %d DISCARDED time.After(1h): %d KB (delta %d KB)\n",
    n, discarded, int64(discarded)-int64(base))

  retained := make([]<-chan time.Time, 0, n)
  for i := 0; i < n; i++ {
    retained = append(retained, time.After(time.Hour))
  }
  held := heapKB()
  fmt.Printf("after %d RETAINED time.After(1h):  %d KB (delta %d KB)\n",
    n, held, int64(held)-int64(base))
  fmt.Println("retained len:", len(retained))

  runtime.KeepAlive(retained)
  retained = nil
  after := heapKB()
  fmt.Printf("after dropping the slice:           %d KB (delta %d KB)\n",
    after, int64(after)-int64(base))
  fmt.Println("go version:", runtime.Version())
}
```

Heap figures vary a little between runs; the orders of magnitude do not. Output:

```
baseline heap: 225 KB
after 200000 DISCARDED time.After(1h): 231 KB (delta 6 KB)
after 200000 RETAINED time.After(1h):  50268 KB (delta 50043 KB)
retained len: 200000
after dropping the slice:           262 KB (delta 37 KB)
go version: go1.27.1
```

Two hundred thousand one-hour timers, created and discarded, cost **6 KB**. The same
timers held in a slice cost **50 MB**, and dropping the slice returns the memory. So on
go1.27.1:

- `for { select { case <-work: ; case <-time.After(d): } }` does **not** leak. Each
  iteration's channel becomes unreachable and is collected.
- What costs memory is *retaining* the channel, which is what a long-lived struct field or
  a growing slice does.
- The pre-1.23 advice to hoist a `*time.Timer` out of the loop and `Reset` it is now a
  micro-optimization (it avoids an allocation per iteration), not a leak fix.

`time.Tick` is the genuine exception, because it returns only a channel with no `Stop`:

```go
package main

import (
  "fmt"
  "runtime"
  "sync/atomic"
  "time"
)

func main() {
  fmt.Println("--- an unstopped ticker keeps firing after GC ---")
  var ticks int64
  func() {
    tk := time.NewTicker(5 * time.Millisecond)
    go func() {
      for range tk.C {
        atomic.AddInt64(&ticks, 1)
      }
    }()
  }()
  time.Sleep(60 * time.Millisecond)
  runtime.GC()
  runtime.GC()
  time.Sleep(60 * time.Millisecond)
  fmt.Println("ticks still arriving after GC:", atomic.LoadInt64(&ticks) > 5)

  fmt.Println("\n--- time.Tick gives you no handle at all ---")
  ch := time.Tick(5 * time.Millisecond)
  var tickTicks int64
  go func() {
    for range ch {
      atomic.AddInt64(&tickTicks, 1)
    }
  }()
  time.Sleep(60 * time.Millisecond)
  fmt.Println("ticks received:", atomic.LoadInt64(&tickTicks) > 5)

  fmt.Println("\n--- a stopped ticker stops ---")
  tk2 := time.NewTicker(5 * time.Millisecond)
  var stopped int64
  go func() {
    for range tk2.C {
      atomic.AddInt64(&stopped, 1)
    }
  }()
  time.Sleep(40 * time.Millisecond)
  tk2.Stop()
  before := atomic.LoadInt64(&stopped)
  time.Sleep(60 * time.Millisecond)
  after := atomic.LoadInt64(&stopped)
  fmt.Println("ticks before Stop > 0:", before > 0)
  fmt.Println("no further ticks after Stop:", after == before)
}
```

Output:

```
--- an unstopped ticker keeps firing after GC ---
ticks still arriving after GC: true

--- time.Tick gives you no handle at all ---
ticks received: true

--- a stopped ticker stops ---
ticks before Stop > 0: true
no further ticks after Stop: true
```

A ticker with a live receiver is reachable *through* that receiver, so GC cannot help:
the goroutine ranging over `tk.C` keeps everything alive, and the ticker keeps firing
forever. `time.Tick` makes that permanent by giving you nothing to call `Stop` on. It is
documented as appropriate only for a ticker that must live for the program's lifetime.

Idiomatic: `tk := time.NewTicker(d); defer tk.Stop()`. Use `time.Tick` only in `main` or a
package-level ticker that genuinely never stops.

`time.AfterFunc` behaves differently again — the runtime keeps it alive until it fires,
because the callback is an observable effect:

```go
package main

import (
  "fmt"
  "runtime"
  "sync/atomic"
  "time"
)

var fired int64

func main() {
  func() {
    time.AfterFunc(20*time.Millisecond, func() { atomic.AddInt64(&fired, 1) })
  }()
  runtime.GC()
  runtime.GC()
  time.Sleep(60 * time.Millisecond)
  fmt.Println("unreferenced AfterFunc still fired:", atomic.LoadInt64(&fired) == 1)
}
```

Output:

```
unreferenced AfterFunc still fired: true
```

So `AfterFunc` is fire-and-forget by design: you can drop the `*Timer` and the function
still runs. Keep the handle only if you might want to `Stop` it.

## Tickers drop ticks

A ticker never queues. If the receiver is busy when a tick is due, that tick is
**discarded** — the ticker is explicitly designed to drop rather than fall behind.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  const period = 10 * time.Millisecond
  const handler = 50 * time.Millisecond
  const run = 500 * time.Millisecond

  tk := time.NewTicker(period)
  defer tk.Stop()

  start := time.Now()
  ticks := 0
  deadline := time.After(run)

loop:
  for {
    select {
    case <-tk.C:
      ticks++
      time.Sleep(handler)
    case <-deadline:
      break loop
    }
  }
  elapsed := time.Since(start)

  fmt.Printf("period=%v handler=%v ran for ~%v\n", period, handler, run)
  fmt.Println("ticks if none were dropped:", int(run/period))
  fmt.Println("ticks actually received:   ", ticks)
  fmt.Println("dropped:", int(run/period)-ticks > 20)
  fmt.Println("received roughly elapsed/handler:",
    ticks >= int(elapsed/handler)-2 && ticks <= int(elapsed/handler)+2)
  fmt.Println("ticker channel capacity:", cap(tk.C))
}
```

The exact tick count varies by a tick or two per run. Output:

```
period=10ms handler=50ms ran for ~500ms
ticks if none were dropped: 50
ticks actually received:    11
dropped: true
received roughly elapsed/handler: true
ticker channel capacity: 0
```

A 10ms ticker with a 50ms handler over half a second should deliver 50 ticks and delivers
**11** — one per handler execution, not one per period. The rate is set by the *slowest*
of the two, and nothing tells you ticks were lost.

Consequences worth internalizing. A ticker is not a scheduler: if you need "exactly N
executions", count them yourself, and if work must not be skipped, hand each tick to a
worker pool (12.1) instead of doing the work inline. For a "once per period, drift-free"
schedule, compute the next absolute deadline and use a `Timer`, since `Ticker` drifts by
the handler's duration whenever it overruns.

## Timeout patterns

```go
package main

import (
  "errors"
  "fmt"
  "time"
)

var errTimeout = errors.New("operation timed out")

func withTimeout[T any](d time.Duration, op func() (T, error)) (T, error) {
  type result struct {
    v   T
    err error
  }
  ch := make(chan result, 1)
  go func() {
    v, err := op()
    ch <- result{v, err}
  }()

  select {
  case r := <-ch:
    return r.v, r.err
  case <-time.After(d):
    var zero T
    return zero, errTimeout
  }
}

func main() {
  v, err := withTimeout(200*time.Millisecond, func() (string, error) {
    time.Sleep(10 * time.Millisecond)
    return "fast enough", nil
  })
  fmt.Printf("%q err=%v\n", v, err)

  v, err = withTimeout(20*time.Millisecond, func() (string, error) {
    time.Sleep(200 * time.Millisecond)
    return "too slow", nil
  })
  fmt.Printf("%q err=%v is errTimeout=%v\n", v, err, errors.Is(err, errTimeout))

  v, err = withTimeout(200*time.Millisecond, func() (string, error) {
    return "", errors.New("op failed")
  })
  fmt.Printf("%q err=%v\n", v, err)
}
```

Output:

```
"fast enough" err=<nil>
"" err=operation timed out is errTimeout=true
"" err=op failed
```

The buffered `chan result` with capacity 1 is load-bearing: on the timeout path nobody
ever receives, and an unbuffered channel would block the worker goroutine forever. This
is the leak that `time.After` is innocent of and people blame it for. See 10.6 for the
`select` mechanics.

Note what this *cannot* do: the operation keeps running after the timeout. There is no way
to interrupt a goroutine from outside, which is exactly why `context` exists.

## context.WithTimeout is the better answer

A timeout that cannot cancel the work is half a timeout. `context.WithTimeout` propagates
the deadline to everything downstream, so the HTTP request, the database query and your
own loops all stop.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "time"
)

func main() {
  ctx, cancel := context.WithTimeout(context.Background(), 20*time.Millisecond)
  defer cancel()
  work := make(chan struct{})
  select {
  case <-work:
    fmt.Println("work done")
  case <-ctx.Done():
    fmt.Println("ctx.Done:", ctx.Err())
    fmt.Println("is DeadlineExceeded:", errors.Is(ctx.Err(), context.DeadlineExceeded))
  }
  dl, ok := ctx.Deadline()
  fmt.Println("has a deadline:", ok, "in the past now:", dl.Before(time.Now()))

  ctx2, cancel2 := context.WithTimeout(context.Background(), time.Hour)
  cancel2()
  fmt.Println("cancelled early:", ctx2.Err(), errors.Is(ctx2.Err(), context.Canceled))
}
```

Output:

```
ctx.Done: context deadline exceeded
is DeadlineExceeded: true
has a deadline: true in the past now: true
cancelled early: context canceled true
```

`ctx.Done()` is a channel, so it drops into the same `select` as a timer — but unlike
`time.After` it is *shared*, so every function that received the context stops at the same
instant. `ctx.Err()` then tells you which of deadline or cancellation happened.

Idiomatic: `time.After` for a local, uncancellable wait; `context.WithTimeout` for
anything that calls into other code. See 11.6.

## Debounce and throttle

Debounce: run only after the input has been quiet for a window. Throttle: run at most once
per window. `AfterFunc` plus `Stop` gives you both.

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

type debouncer struct {
  mu    sync.Mutex
  timer *time.Timer
  wait  time.Duration
}

func newDebouncer(wait time.Duration) *debouncer {
  return &debouncer{wait: wait}
}

func (d *debouncer) Trigger(f func()) {
  d.mu.Lock()
  defer d.mu.Unlock()
  if d.timer != nil {
    d.timer.Stop()
  }
  d.timer = time.AfterFunc(d.wait, f)
}

func (d *debouncer) Stop() {
  d.mu.Lock()
  defer d.mu.Unlock()
  if d.timer != nil {
    d.timer.Stop()
  }
}

type throttler struct {
  mu   sync.Mutex
  last time.Time
  min  time.Duration
}

func newThrottler(min time.Duration) *throttler {
  return &throttler{min: min}
}

func (t *throttler) Allow() bool {
  t.mu.Lock()
  defer t.mu.Unlock()
  now := time.Now()
  if now.Sub(t.last) < t.min {
    return false
  }
  t.last = now
  return true
}

func main() {
  fmt.Println("--- debounce: 10 rapid triggers, one call ---")
  var mu sync.Mutex
  calls := 0
  d := newDebouncer(40 * time.Millisecond)
  for i := 0; i < 10; i++ {
    d.Trigger(func() { mu.Lock(); calls++; mu.Unlock() })
    time.Sleep(5 * time.Millisecond)
  }
  time.Sleep(100 * time.Millisecond)
  mu.Lock()
  fmt.Println("calls after 10 triggers 5ms apart:", calls)
  mu.Unlock()

  fmt.Println("\n--- debounce: triggers spaced beyond the window ---")
  mu.Lock()
  calls = 0
  mu.Unlock()
  for i := 0; i < 3; i++ {
    d.Trigger(func() { mu.Lock(); calls++; mu.Unlock() })
    time.Sleep(60 * time.Millisecond)
  }
  time.Sleep(80 * time.Millisecond)
  mu.Lock()
  fmt.Println("calls after 3 triggers 60ms apart:", calls)
  mu.Unlock()
  d.Stop()

  fmt.Println("\n--- throttle: at most one per 30ms ---")
  th := newThrottler(30 * time.Millisecond)
  allowed := 0
  for i := 0; i < 20; i++ {
    if th.Allow() {
      allowed++
    }
    time.Sleep(5 * time.Millisecond)
  }
  fmt.Println("20 attempts 5ms apart over ~100ms")
  fmt.Println("allowed between 3 and 5:", allowed >= 3 && allowed <= 5)
}
```

Output:

```
--- debounce: 10 rapid triggers, one call ---
calls after 10 triggers 5ms apart: 1

--- debounce: triggers spaced beyond the window ---
calls after 3 triggers 60ms apart: 3

--- throttle: at most one per 30ms ---
20 attempts 5ms apart over ~100ms
allowed between 3 and 5: true
```

Ten triggers 5ms apart collapse to **one** call; three triggers 60ms apart (outside the
40ms window) each get through. The throttle's count is asserted as a range rather than a
number, because scheduling jitter moves it between 3 and 5.

The mutex is not optional. `Trigger` is called concurrently in real code — from an fsnotify
handler or an HTTP endpoint — and `Stop`-then-`AfterFunc` is two operations that must be
atomic together, or two rapid triggers can both schedule.

The callback runs on its own goroutine, so anything it touches needs its own
synchronization — which is why `calls` has a mutex here too.

For rate limiting a request path rather than a callback, `golang.org/x/time/rate` and the
token-bucket patterns in 12.4 are the better tool; this `throttler` is for coalescing
local events.

## Testing code that sleeps

A test that waits a real second is a test nobody runs. Inject the clock.

```go
package main

import (
  "errors"
  "fmt"
  "time"
)

type Clock interface {
  Now() time.Time
  After(d time.Duration) <-chan time.Time
  Sleep(d time.Duration)
}

type realClock struct{}

func (realClock) Now() time.Time                         { return time.Now() }
func (realClock) After(d time.Duration) <-chan time.Time { return time.After(d) }
func (realClock) Sleep(d time.Duration)                  { time.Sleep(d) }

type fakeClock struct {
  now   time.Time
  slept []time.Duration
}

func (f *fakeClock) Now() time.Time { return f.now }
func (f *fakeClock) After(d time.Duration) <-chan time.Time {
  ch := make(chan time.Time, 1)
  f.now = f.now.Add(d)
  ch <- f.now
  return ch
}
func (f *fakeClock) Sleep(d time.Duration) {
  f.slept = append(f.slept, d)
  f.now = f.now.Add(d)
}

func retry(c Clock, attempts int, base time.Duration, op func(int) error) error {
  var err error
  for i := 0; i < attempts; i++ {
    if err = op(i); err == nil {
      return nil
    }
    if i < attempts-1 {
      c.Sleep(base << i)
    }
  }
  return err
}

func main() {
  fake := &fakeClock{now: time.Date(2026, 3, 7, 15, 0, 0, 0, time.UTC)}
  start := time.Now()
  err := retry(fake, 4, time.Hour, func(i int) error {
    if i < 3 {
      return errors.New("boom")
    }
    return nil
  })
  fmt.Println("err:", err)
  fmt.Println("slept:", fake.slept)
  fmt.Println("fake clock advanced to:", fake.now.Format(time.RFC3339))
  fmt.Println("real wall time spent < 50ms:", time.Since(start) < 50*time.Millisecond)
  var _ Clock = realClock{}
}
```

Output:

```
err: <nil>
slept: [1h0m0s 2h0m0s 4h0m0s]
fake clock advanced to: 2026-03-07T22:00:00Z
real wall time spent < 50ms: true
```

Seven hours of exponential backoff, asserted exactly, in under 50 milliseconds of real
time. The `slept` slice is the assertion that matters — it proves the backoff schedule is
1h, 2h, 4h rather than just "it eventually succeeded".

Keep the interface as small as the code actually needs; a three-method `Clock` is usually
enough, and production code passes `realClock{}`. For anything more elaborate (fake timers
that fire in order, a controllable "advance by d") use a library rather than growing this.

## The monotonic clock and timers

Timers are scheduled against the runtime's monotonic clock, not the wall clock. If NTP
steps the system clock forward an hour, a `time.After(2*time.Hour)` still fires two hours
from now, and a `NewTicker(time.Minute)` still ticks every minute. That is almost always
what you want, and it is why "run at 09:00 local" cannot be implemented as a sleep —
compute the duration until the next 09:00 with `AddDate` (13.4) and re-compute it after
each firing.

## Quick Reference

| Need | Call |
|---|---|
| Block for a fixed time | `time.Sleep(d)` |
| Timeout one `select` | `case <-time.After(d):` |
| Timeout a call chain | `context.WithTimeout(ctx, d)` |
| A reusable one-shot | `t := time.NewTimer(d)`, `t.Reset(d)` |
| Run a function later | `time.AfterFunc(d, f)` |
| Cancel a scheduled function | `if t.Stop() { /* prevented */ }` |
| Repeat forever | `tk := time.NewTicker(d); defer tk.Stop()` |
| Repeat for the program's life | `time.Tick(d)` |
| Debounce | `Stop` the old `AfterFunc`, schedule a new one |
| Throttle | compare `time.Since(last)` against the window |
| Test timing logic | inject a `Clock` interface |

| Trap | Reality |
|---|---|
| "`time.After` leaks in a loop" | obsolete since 1.23 for a **discarded** timer |
| Retaining a timer channel in a struct | *this* is what keeps the timer alive |
| `time.Tick` in a function | no `Stop`; genuinely unreclaimable |
| `NewTicker` without `defer tk.Stop()` | fires forever while a receiver exists |
| Expecting every tick | ticks are dropped when the handler is slow |
| `if !t.Stop() { <-t.C }` on 1.23+ | unnecessary; can block |
| Branching on `Stop()` for a `NewTimer` | only meaningful for `AfterFunc` |
| Unbuffered result channel in a timeout | the worker goroutine leaks on the timeout path |
| `time.Sleep` for cancellable work | it cannot be interrupted; use `select` |
| `cap(timer.C) == 1` | it is 0 since Go 1.23 |

## Common Mistakes

**Using `time.Tick` anywhere but `main`.** It returns no handle, so the ticker can never be
stopped; use `time.NewTicker` with `defer tk.Stop()`.

**Forgetting the result channel's buffer in a timeout.** On the timeout path nobody
receives, so an unbuffered channel pins the worker goroutine forever.

**Assuming a ticker delivers one tick per period.** A slow handler silently drops ticks;
if every interval must be processed, hand them to a worker pool.

**Keeping the old `if !t.Stop() { <-t.C }` drain.** On Go 1.23+ `Stop` and `Reset` clear
the channel, and the extra receive can block.

**Hoisting a `*time.Timer` out of a loop to "fix a leak".** There is no leak for a
discarded `time.After`; do it for the allocation saving, if you measured one.

**Calling `Reset` from a different goroutine than the one receiving.** The channel is safe
but your notion of which deadline is pending is not.

## Exercises

1. Write `func pollUntil(ctx context.Context, every time.Duration, check func() (bool, error)) error`
   with a `Ticker`, returning on the first `true`, on a `check` error, or on
   `ctx.Done()`. Test all three exits with a 1ms period.
2. Implement a drift-free scheduler: run a job at fixed absolute instants (t0, t0+d,
   t0+2d, …) using a `Timer` and `time.Until`, so a slow job does not push later runs
   late. Compare its instants to a `Ticker`'s under an artificially slow handler.
3. Extend the `debouncer` with a `maxWait` so the callback runs at least that often even
   under continuous triggering. Prove both the quiet-period and max-wait paths fire.
4. Reproduce the "retained timer" memory result for `context.WithTimeout`: measure
   `HeapAlloc` for 100,000 contexts with and without calling `cancel()`, and explain the
   difference.
5. Write `func raceFirst[T any](ctx context.Context, fns ...func(context.Context) (T, error)) (T, error)`
   returning the first success and cancelling the rest. Assert with a counter that the
   losers observed cancellation.

---

[← 13.4 time: Durations, Instants, and Formatting](04-time.md) · [Course Index](../README.md) · [13.6 encoding/json: Marshal and Unmarshal →](06-json.md)
