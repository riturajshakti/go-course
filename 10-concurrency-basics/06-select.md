[← 10.5 Directional Channels and nil Channels](05-directional-nil.md) · [Course Index](../README.md) · [11.1 sync.WaitGroup →](../11-sync-context/01-waitgroup.md)

# 10.6 — select, default, and Timeouts

## What You'll Learn

- `select` blocks until one case is ready, and picks **uniformly at random** among several ready cases
- `default` makes it non-blocking: a try-send or try-receive that never parks
- `time.After` in a `select` is the whole timeout idiom; `time.NewTimer` + `Reset` is the hot-loop version
- `break` inside `select` leaves only the `select` — breaking the loop needs a label

## Coming From Another Language

| Other | Go |
|---|---|
| `Promise.race([a, b])` | `select` — but it waits on channel *operations*, not values |
| `asyncio.wait(fs, return_when=FIRST_COMPLETED)` | the nearest match; `select` also handles *sends* |
| `tokio::select!` | almost identical, random branch choice included |
| `Promise.race` leaves the loser running | `select` leaves the loser's channel untouched; nothing is cancelled |
| `AbortSignal.timeout(ms)` | `case <-time.After(ms)` |

`select` is not `Promise.race`. A race takes a set of futures and settles on the first one
to finish; `select` takes a set of *channel operations* — receives and sends, mixed — and
commits to exactly one of them. The others do not happen at all: no value is consumed, no
value is sent, nothing is cancelled. That is why `select` can express a timeout without
leaking a consumed message, and why a `select` with a send case is something `Promise.race`
has no equivalent for.

## select blocks until a case is ready

```go
package main

import "fmt"

func main() {
  fast := make(chan string, 1)
  slow := make(chan string)

  fast <- "fast is ready"

  select {
  case v := <-fast:
    fmt.Println("picked:", v)
  case v := <-slow:
    fmt.Println("picked:", v)
  }
}
```

Output:

```
picked: fast is ready
```

Only one case is ready, so the choice is forced. `slow` is never touched — no receiver is
left registered on it.

## Random choice among ready cases

When more than one case is ready, Go picks uniformly at random. Not in source order, not
round-robin. This loop makes both cases ready on every iteration and tallies 1000 picks.

```go
package main

import "fmt"

func main() {
  a := make(chan int, 1)
  b := make(chan int, 1)

  counts := map[string]int{}
  for range 1000 {
    a <- 1
    b <- 2
    select {
    case <-a:
      counts["a"]++
      <-b
    case <-b:
      counts["b"]++
      <-a
    }
  }
  fmt.Println("a:", counts["a"], "b:", counts["b"])
  fmt.Println("total:", counts["a"]+counts["b"])
}
```

Output:

```
a: 524 b: 476
total: 1000
```

The total is exact; the split varies by roughly ±30 per run (five runs here gave a-counts
of 524, 513, 497, 485, 491). The aggregate is the point: close to 50/50, which is what
"uniform" buys you. Writing the readier channel first does not prioritise it. If you need
priority, you need a nested `select` with a `default`, not case ordering.

## default makes it non-blocking

```go
package main

import "fmt"

func main() {
  ch := make(chan int, 1)

  select {
  case v := <-ch:
    fmt.Println("received", v)
  default:
    fmt.Println("nothing to receive")
  }

  select {
  case ch <- 7:
    fmt.Println("sent 7")
  default:
    fmt.Println("buffer full, send skipped")
  }

  select {
  case ch <- 8:
    fmt.Println("sent 8")
  default:
    fmt.Println("buffer full, send skipped")
  }

  fmt.Println("len", len(ch), "value", <-ch)
}
```

Output:

```
nothing to receive
sent 7
buffer full, send skipped
len 1 value 7
```

With a `default`, a `select` never blocks: if nothing else is ready the default runs
immediately. That gives you a try-receive (first block) and a try-send (second and third).
This is the honest way to ask "is there anything there?" — unlike `len(ch) > 0`, the test
and the operation are a single atomic step.

## select{} blocks forever

```go
package main

import "fmt"

func main() {
  fmt.Println("about to block forever")
  select {}
}
```

This deadlocks on purpose. Output:

```
about to block forever
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [select (no cases)]:
main.main()
	/tmp/scratch/main.go:7 +0x50
exit status 2
```

An empty `select` has no case that can ever be ready. It appears deliberately in servers
as the last line of `main` to park the main goroutine while worker goroutines run — in
that case other goroutines are alive, so the deadlock detector stays quiet.

## Timeouts with time.After

`time.After(d)` returns a `<-chan Time` that delivers one value after `d`. Put it in a
`select` and you have a timeout.

```go
package main

import (
  "fmt"
  "time"
)

func work(d time.Duration) <-chan string {
  out := make(chan string, 1)
  go func() {
    time.Sleep(d)
    out <- fmt.Sprintf("done after %v", d)
  }()
  return out
}

func main() {
  for _, d := range []time.Duration{20 * time.Millisecond, 200 * time.Millisecond} {
    select {
    case res := <-work(d):
      fmt.Println("result:", res)
    case <-time.After(100 * time.Millisecond):
      fmt.Printf("timeout waiting on %v job\n", d)
    }
  }
}
```

Output:

```
result: done after 20ms
timeout waiting on 200ms job
```

The `time.Sleep` inside `work` is standing in for real latency; timing *is* the lesson
here, which is the one case where sleeping is the right tool. The 20ms vs 200ms gap
against a 100ms timeout is wide enough that the output is stable.

Note what the timeout does and does not do. It stops *waiting*. The 200ms goroutine keeps
running, completes, and sends into its capacity-1 buffer with nobody listening. Had `out`
been unbuffered, that goroutine would have blocked on the send forever — a leak. `select`
cancels nothing; cancellation is a signal you propagate, which is
[11.6](../11-sync-context/06-context-cancel.md).

## The time.After cost in a hot loop

The classic advice is that `time.After` leaks until the timer fires. On Go 1.27 that is
no longer true for a *discarded* timer, but it is still true for a *retained* one.

```go
package main

import (
  "fmt"
  "runtime"
  "time"
)

func heapKB() uint64 {
  runtime.GC()
  var m runtime.MemStats
  runtime.ReadMemStats(&m)
  return m.HeapAlloc / 1024
}

func main() {
  fmt.Println("go version:", runtime.Version())

  base := heapKB()

  held := make([]<-chan time.Time, 0, 100_000)
  for range 100_000 {
    held = append(held, time.After(time.Hour))
  }
  fmt.Printf("100k retained time.After: %d KB over baseline\n", heapKB()-base)
  runtime.KeepAlive(held)
}
```

Output:

```
go version: go1.27.1
100k retained time.After: 25021 KB over baseline
```

25 MB for 100,000 one-hour timers nobody will ever read. Now the same count discarded each
iteration, plus the reuse pattern:

```go
package main

import (
  "fmt"
  "runtime"
  "time"
)

func heapKB() uint64 {
  runtime.GC()
  var m runtime.MemStats
  runtime.ReadMemStats(&m)
  return m.HeapAlloc / 1024
}

func main() {
  base := heapKB()

  for range 100_000 {
    select {
    case <-time.After(time.Hour):
    default:
    }
  }
  fmt.Printf("100k discarded time.After: %d KB over baseline\n", heapKB()-base)

  timer := time.NewTimer(time.Hour)
  defer timer.Stop()
  for range 100_000 {
    timer.Stop()
    timer.Reset(time.Hour)
    select {
    case <-timer.C:
    default:
    }
  }
  fmt.Printf("plus 100k timer reuses:    %d KB over baseline\n", heapKB()-base)
}
```

Output:

```
100k discarded time.After: 11 KB over baseline
plus 100k timer reuses:    13 KB over baseline
```

Those two numbers wobble by a few KB run to run (repeated runs gave 11, 10 and 5 KB for
the first line), and the 25 MB figure moves by a few KB too; what matters is the three
orders of magnitude between them, not the digits. Since Go 1.23 an unreferenced timer is garbage-collectable, so the "leak"
only bites when something holds the channel — a `select` case in a loop that keeps the
value alive, or a slice like the one above.

Even so, `time.NewTimer` + `Reset` is the right shape for a tight loop: it allocates one
timer instead of asking the runtime to register and collect a hundred thousand. Note the
`timer.Stop()` before `Reset` and the drain, because `Reset` on a timer that has already
fired leaves a stale value in `timer.C`.

`time.Tick` has no `Stop` at all — it returns only the channel, so the ticker it creates
can never be stopped and genuinely does leak for the life of the process. Use it only for
a ticker that should live as long as the program; everywhere else use `time.NewTicker` with
`defer ticker.Stop()`. Timers and tickers in depth: [13.5](../13-stdlib/05-timers.md).

## Per-iteration vs whole-operation timeouts

Putting `time.After` inside the loop gives each iteration its own fresh deadline, which
means a stream of slow-but-not-quite-timing-out items can run forever. Creating the timeout
channel *once*, outside the loop, bounds the whole operation.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  ticker := time.NewTicker(10 * time.Millisecond)
  defer ticker.Stop()

  done := make(chan struct{})
  go func() {
    time.Sleep(55 * time.Millisecond)
    close(done)
  }()

  ticks := 0
  for {
    select {
    case <-ticker.C:
      ticks++
    case <-done:
      fmt.Println("stopped after at least 5 ticks:", ticks >= 5)
      return
    }
  }
}
```

Output:

```
stopped after at least 5 ticks: true
```

`done` is created once, so it bounds the entire loop no matter how many ticks arrive. The
exact tick count varies with scheduling, which is why the program asserts a lower bound
instead of printing it. Swap `case <-done` for `case <-time.After(55*time.Millisecond)`
and the deadline resets on every tick — the loop never ends.

## break leaves the select, not the loop

```go
package main

import "fmt"

func bareBreak(jobs chan int) (sum, rounds int) {
  for {
    rounds++
    if rounds > 6 {
      return sum, rounds
    }
    select {
    case v, ok := <-jobs:
      if !ok {
        break
      }
      sum += v
    }
  }
}

func labelledBreak(jobs chan int) (sum, rounds int) {
loop:
  for {
    rounds++
    select {
    case v, ok := <-jobs:
      if !ok {
        break loop
      }
      sum += v
    }
  }
  return sum, rounds
}

func filled() chan int {
  ch := make(chan int, 3)
  ch <- 1
  ch <- 2
  ch <- 3
  close(ch)
  return ch
}

func main() {
  s, r := bareBreak(filled())
  fmt.Printf("bare break:      sum=%d rounds=%d (bailed out on a counter)\n", s, r)

  s, r = labelledBreak(filled())
  fmt.Printf("break loop:      sum=%d rounds=%d\n", s, r)
}
```

Output:

```
bare break:      sum=6 rounds=7 (bailed out on a counter)
break loop:      sum=6 rounds=4
```

`select` is a `break`-able statement in its own right, so a bare `break` in a case just
exits the `select` and the `for` starts another round — forever, against a closed channel.
Both versions compute the right sum; only the labelled one terminates on its own. The
counter in `bareBreak` exists purely so the demo returns. Labels and `break`/`continue`
targeting are [3.4](../03-control-flow/04-labels-goto.md).

Idiomatic: `break loop` with a label, or `return` from the function.

## nil channels disable cases

A nil channel case is never ready, so assigning `nil` deletes it from the `select`. That
is the merge-until-both-closed pattern, built completely in
[10.5](05-directional-nil.md). The reason it belongs here too: it is the only way to turn a
case off, because closing a channel makes its case *permanently ready* and spins.

## The canonical worker: work, timeout, cancellation

One `select` handling all three, with a nested `select` for the per-job deadline.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
  "time"
)

type job struct {
  id   int
  cost time.Duration
}

type result struct {
  id     int
  status string
}

func worker(jobs <-chan job, out chan<- result, quit <-chan struct{}, wg *sync.WaitGroup) {
  defer wg.Done()
  for {
    select {
    case j, ok := <-jobs:
      if !ok {
        return
      }
      done := make(chan struct{})
      go func() {
        time.Sleep(j.cost)
        close(done)
      }()
      select {
      case <-done:
        out <- result{j.id, "ok"}
      case <-time.After(40 * time.Millisecond):
        out <- result{j.id, "timeout"}
      case <-quit:
        out <- result{j.id, "cancelled"}
        return
      }
    case <-quit:
      return
    }
  }
}

func run(cancelAfter time.Duration) []result {
  jobs := make(chan job, 4)
  out := make(chan result, 4)
  quit := make(chan struct{})

  for _, j := range []job{{1, time.Millisecond}, {2, 200 * time.Millisecond}, {3, time.Millisecond}, {4, 200 * time.Millisecond}} {
    jobs <- j
  }
  close(jobs)

  var wg sync.WaitGroup
  wg.Add(1)
  go worker(jobs, out, quit, &wg)

  if cancelAfter > 0 {
    go func() {
      time.Sleep(cancelAfter)
      close(quit)
    }()
  }

  wg.Wait()
  close(out)

  var rs []result
  for r := range out {
    rs = append(rs, r)
  }
  sort.Slice(rs, func(i, j int) bool { return rs[i].id < rs[j].id })
  return rs
}

func main() {
  fmt.Println("no cancellation:")
  for _, r := range run(0) {
    fmt.Printf("  job %d: %s\n", r.id, r.status)
  }

  fmt.Println("cancelled after 20ms:")
  for _, r := range run(20 * time.Millisecond) {
    fmt.Printf("  job %d: %s\n", r.id, r.status)
  }
}
```

Output:

```
no cancellation:
  job 1: ok
  job 2: timeout
  job 3: ok
  job 4: timeout
cancelled after 20ms:
  job 1: ok
  job 2: cancelled
```

Race-clean under `go run -race .`, and identical across repeated runs — the 1ms vs 200ms
costs are far enough either side of the 40ms deadline, and 20ms is comfortably inside
job 2's 200ms, that scheduling jitter cannot flip a verdict.

Four things to take from the shape. The outer `select` is the worker's whole life: take
work, or quit. `case j, ok := <-jobs` with the `return` on `!ok` is how a worker retires
when the producer closes. The inner `select` is the per-job deadline, and it carries the
`quit` case too so cancellation lands even mid-job. And `quit` is only ever closed, never
sent on, so one close stops every worker ([10.4](04-closing-range.md)).

In real code the `quit chan struct{}` becomes a `context.Context` and `case <-quit`
becomes `case <-ctx.Done()`, which adds a deadline, a cause, and propagation to children
for free — [11.6](../11-sync-context/06-context-cancel.md). Deliberate data races and the
`-race` detector itself are [11.5](../11-sync-context/05-race-detector.md).

## Quick Reference

| Form | Behaviour |
|---|---|
| `select { case <-a: ... case <-b: ... }` | blocks until one is ready; random if both |
| `select { case a <- v: ... }` | send cases work the same as receive cases |
| `select { ... default: ... }` | never blocks; `default` runs if nothing is ready |
| `select {}` | blocks forever |
| `case v, ok := <-ch:` | comma-ok works in a case |
| `case <-time.After(d):` | timeout |
| `case <-ticker.C:` | periodic |
| a nil channel in a case | never ready — the case is disabled |
| a closed channel in a case | always ready — fires immediately, forever |

| Timer API | Stoppable | Use for |
|---|---|---|
| `time.After(d)` | no | a one-shot timeout, not in a hot loop |
| `time.NewTimer(d)` | `Stop`, `Reset` | a timeout in a loop; reuse one timer |
| `time.NewTicker(d)` | `Stop` | periodic work; always `defer Stop()` |
| `time.Tick(d)` | **no** | only a ticker meant to outlive the program |

| Need | Shape |
|---|---|
| wait for whichever comes first | plain `select` |
| try without blocking | `select` + `default` |
| time-bound a wait | `case <-time.After(d)` |
| bound a whole loop | one timeout channel made *outside* the loop |
| stop every worker at once | `close(quit)` and a `case <-quit` |
| drop a case permanently | set its channel variable to `nil` |
| leave the loop from a case | `break label` or `return` |

## Common Mistakes

**Expecting the first case to win when several are ready.** The choice is uniformly
random; order expresses nothing.

**Writing `break` in a `select` case to exit the enclosing loop.** It only exits the
`select`; use `break label` or `return`.

**Putting `time.After` inside the loop for a whole-operation deadline.** Each iteration
gets a fresh deadline and the loop can run forever; create the timeout channel once
outside.

**Calling `timer.Reset` without stopping and draining first.** A stale value may already
be in `timer.C` and the next receive returns instantly.

**Treating a timeout as cancellation.** `select` only stops *waiting*; the goroutine keeps
running, so give it a `quit` channel or a context, and buffer its result channel.

## Exercises

1. Reproduce the 1000-iteration distribution test with three ready cases instead of two,
   and confirm each lands near 333.
2. Write `func tryRecv[T any](ch <-chan T) (T, bool)` and `func trySend[T any](ch chan<- T, v T) bool`
   using `select`/`default`, then show both returning `false` against an empty and a full
   buffered channel.
3. Build a priority `select`: given `high` and `low` channels, always drain `high` first
   and only take from `low` when `high` is empty. Prove it with 100 values interleaved on
   both channels.
4. Write a retry loop with a whole-operation budget of 500ms that retries a failing
   operation every 50ms using one `time.NewTimer` and one deadline channel, and prints how
   many attempts fitted in the budget.
5. Take the canonical worker and replace `quit chan struct{}` with a `context.Context`
   from `context.WithTimeout`. Show the same two outputs, then explain what the context
   version gives you that the channel version does not.

---

[← 10.5 Directional Channels and nil Channels](05-directional-nil.md) · [Course Index](../README.md) · [11.1 sync.WaitGroup →](../11-sync-context/01-waitgroup.md)
