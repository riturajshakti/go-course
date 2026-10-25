[← 10.2 Unbuffered Channels](02-unbuffered-channels.md) · [Course Index](../README.md) · [10.4 Closing, range over a Channel, and the comma-ok Receive →](04-closing-range.md)

# 10.3 — Buffered Channels

## What You'll Learn

- `make(chan T, n)` gives a channel a queue of `n` slots; `len` is how many are occupied, `cap` is `n`
- A send blocks only when the buffer is full; a receive blocks only when it is empty
- A buffer that is "big enough" because of timing is a latent deadlock, not a fix
- `make(chan struct{}, n)` is Go's semaphore, and you can prove the bound with an atomic counter

## Coming From Another Language

| Other | Go |
|---|---|
| `asyncio.Queue(maxsize=4)` | `make(chan T, 4)` — same bounded-queue semantics |
| `queue.Queue()` with no maxsize — unbounded | there is no unbounded channel; pick an `n` |
| `tokio::sync::mpsc::channel(4)` | the closest match, bound and all |
| JS — no bounded queue in the language; you write one | first-class, one argument to `make` |
| `asyncio.Semaphore(3)` | `make(chan struct{}, 3)`, acquire by sending |

A buffered channel is the only queue primitive Go gives you, and it is always bounded.
There is no unbounded variant, which is deliberate: an unbounded queue turns a slow
consumer into an out-of-memory crash instead of backpressure. Picking `n` is picking how
much burst you absorb before the producer is forced to wait.

## len and cap as the buffer fills

```go
package main

import "fmt"

func main() {
  ch := make(chan string, 3)

  ch <- "a"
  ch <- "b"
  fmt.Println(len(ch), cap(ch))

  ch <- "c"
  fmt.Println(len(ch), cap(ch))

  fmt.Println(<-ch)
  fmt.Println(len(ch), cap(ch))
  fmt.Println(<-ch, <-ch)
  fmt.Println(len(ch), cap(ch))
}
```

Output:

```
2 3
3 3
a
2 3
b c
0 3
```

`cap` never changes. `len` is the number of values sitting in the buffer right now.
Values come out in FIFO order — `a`, then `b`, then `c`.

Note that `len` and `cap` on a channel mean something different from `len` and `cap` on a
slice. On a slice `len` is the data you can index; on a channel `len` is a *momentary*
queue depth that another goroutine may change the instant after you read it.

## An unbuffered channel has capacity zero

```go
package main

import "fmt"

func main() {
  u := make(chan int)
  b := make(chan int, 2)
  fmt.Println(len(u), cap(u))
  fmt.Println(len(b), cap(b))
  b <- 1
  fmt.Println(len(b), cap(b))
}
```

Output:

```
0 0
0 2
1 2
```

`cap(u) == 0` is the whole difference. With zero slots there is nowhere to park a value,
so a send on `u` must find a receiver.

## Deadlock on a full buffer

Sending into a full buffer with nobody draining it blocks exactly like an unbuffered send.

```go
package main

import "fmt"

func main() {
  ch := make(chan int, 2)
  ch <- 1
  ch <- 2
  fmt.Println("buffer full, len =", len(ch))
  ch <- 3
  fmt.Println("never printed")
}
```

This deadlocks on purpose. Output:

```
buffer full, len = 2
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/tmp/scratch/main.go:10 +0xb0
exit status 2
```

The buffer bought you two sends, not infinite sends. Notice the goroutine state the
runtime prints: `[chan send]`. It is the same state as the unbuffered deadlock in
[10.2](02-unbuffered-channels.md) — a full buffer and a zero-capacity channel block the
sender identically.

## Receiving from an empty buffer blocks

```go
package main

import "fmt"

func main() {
  ch := make(chan int, 1)

  ch <- 1
  fmt.Println(<-ch)

  ch <- 2
  fmt.Println(<-ch)
  fmt.Println("len", len(ch), "cap", cap(ch))
}
```

Output:

```
1
2
len 0 cap 1
```

A capacity-1 channel used this way is a mailbox: one value in flight, hand it off, reuse
the slot. It is the most common buffer size in real code, because it lets a producer get
one step ahead of a consumer without any extra coordination.

## Decoupling a bursty producer

The buffer's real job is to let a sender finish and move on without waiting for a
receiver to show up. Three goroutines each produce one value and exit immediately; `main`
collects at its leisure.

```go
package main

import "fmt"

func main() {
  results := make(chan string, 3)

  go func() { results <- "from A" }()
  go func() { results <- "from B" }()
  go func() { results <- "from C" }()

  got := make([]string, 0, 3)
  for range 3 {
    got = append(got, <-results)
  }
  fmt.Println(len(got), cap(results), len(results))
}
```

Output:

```
3 3 0
```

With `cap(results) == 3`, none of the three senders ever blocks, so all three goroutines
are gone before `main` reads its first value. With an unbuffered channel the same program
still works, but each sender parks until `main` arrives to take its value.

The order the three values arrive in varies run to run, which is why this program prints a
count rather than the values.

## Known-count fan-in

When you know exactly how many results are coming, size the buffer to the count and no
sender can ever block. Collect into a slice and sort to make the output deterministic.

```go
package main

import (
  "fmt"
  "sort"
)

func main() {
  jobs := []int{5, 2, 9, 1, 7}
  results := make(chan int, len(jobs))

  for _, j := range jobs {
    go func() { results <- j * j }()
  }

  squares := make([]int, 0, len(jobs))
  for range jobs {
    squares = append(squares, <-results)
  }
  sort.Ints(squares)
  fmt.Println(squares)
}
```

Output:

```
[1 4 25 49 81]
```

This relies on Go 1.22+ loop-variable semantics: each iteration gets its own `j`. Under
Go 1.21 all five goroutines would share one `j` — see [10.1](01-goroutines.md).

Idiomatic: when the number of results is known up front, `make(chan T, n)` plus `n`
receives. No `WaitGroup` needed, because `n` successful receives already proves `n`
sends happened.

## A buffer that only works because of timing

Size the buffer one short of the number of senders and the program still looks correct.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  out := make(chan int, 3)
  var wg sync.WaitGroup

  for i := 1; i <= 4; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      out <- i
    }()
  }

  wg.Wait()
  close(out)
  for v := range out {
    fmt.Println(v)
  }
}
```

This deadlocks on purpose. Output (goroutine dump truncated — the real output lists the
other blocked senders too; the goroutine numbers and addresses differ every run):

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
sync.runtime_SemacquireWaitGroup(0x104b3c230?, 0xc0?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/sema.go:114 +0x38
sync.(*WaitGroup).Wait(0x69a8aad96020)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:206 +0xa4
main.main()
	/tmp/scratch/main.go:20 +0xd8

goroutine 21 [chan send]:
main.main.func1()
	/tmp/scratch/main.go:16 +0x48
created by main.main in goroutine 1
	/tmp/scratch/main.go:14 +0x60
exit status 2
```

Four senders, three slots. The fourth blocks on its send, so its `wg.Done` never runs, so
`wg.Wait` never returns, so nothing ever drains the buffer. Change the loop bound to 3 and
it passes; change it back to 4 and it hangs. A buffer sized to the happy path is a latent
deadlock waiting for load to change.

The fix is not a bigger buffer. It is to drain concurrently — receive in one goroutine
while the senders send, and close the channel after `wg.Wait` in a separate goroutine.
That is the pattern in [10.4](04-closing-range.md).

## Non-blocking sends: dropping instead of blocking

`select` with a `default` turns a send into a try-send. Covered properly in
[10.6](06-select.md), but it is the natural companion to a buffer.

```go
package main

import "fmt"

func main() {
  events := make(chan string, 2)

  dropped := 0
  for _, e := range []string{"e1", "e2", "e3", "e4"} {
    select {
    case events <- e:
      fmt.Println("queued", e)
    default:
      dropped++
      fmt.Println("dropped", e)
    }
  }

  close(events)
  for e := range events {
    fmt.Println("handled", e)
  }
  fmt.Println("dropped total:", dropped)
}
```

Output:

```
queued e1
queued e2
dropped e3
dropped e4
handled e1
handled e2
dropped total: 2
```

This is how a metrics or logging pipeline avoids letting a slow sink stall the request
path: bounded buffer, drop on overflow, count the drops.

## The buffered-channel semaphore

A buffer of `n` is a counting semaphore of `n`: sending acquires a slot, receiving
releases it. Here 20 goroutines contend for 3 slots, and an atomic high-water counter
proves the bound was never exceeded.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
  "time"
)

func main() {
  const limit = 3
  sem := make(chan struct{}, limit)

  var running, maxRunning atomic.Int64
  var wg sync.WaitGroup

  for i := range 20 {
    wg.Add(1)
    go func() {
      defer wg.Done()
      sem <- struct{}{}
      defer func() { <-sem }()

      n := running.Add(1)
      for {
        old := maxRunning.Load()
        if n <= old || maxRunning.CompareAndSwap(old, n) {
          break
        }
      }
      time.Sleep(5 * time.Millisecond)
      running.Add(-1)
      _ = i
    }()
  }
  wg.Wait()

  fmt.Println("limit:", limit)
  fmt.Println("max concurrent observed:", maxRunning.Load())
  fmt.Println("still running:", running.Load())
}
```

Output:

```
limit: 3
max concurrent observed: 3
still running: 0
```

The `time.Sleep` here is only there to make the critical sections overlap long enough to
observe; it is not a synchronization primitive and the bound does not depend on it. The
`sem <- struct{}{}` / `<-sem` pair is what enforces the limit. Run with `-race`: clean.

`struct{}` is the right element type — it occupies zero bytes, so the buffer is pure
bookkeeping. `atomic.Int64` and the CAS loop come up again in
[11.4](../11-sync-context/04-atomic.md); rate limiting and more semaphore variants are
[12.4](../12-concurrency-patterns/04-rate-limiting.md).

Note the `defer func() { <-sem }()` rather than `defer <-sem`: `defer` takes a function
call, and a receive expression is not one.

## Unbuffered vs buffered: how to choose

| You want | Use | Why |
|---|---|---|
| A handoff that proves both sides met | `make(chan T)` | the send cannot complete without a receiver |
| A done or quit signal | `make(chan struct{})` | you only ever close it; capacity is irrelevant |
| A single result from one goroutine | `make(chan T, 1)` | the sender can finish and exit before you read |
| `n` results, `n` known | `make(chan T, n)` | no sender ever blocks, no `WaitGroup` needed |
| Absorb a bursty producer | `make(chan T, burst)` | backpressure kicks in past `burst` |
| Drop under overload | buffer + `select`/`default` | bounded memory, countable drops |
| Limit concurrency to `n` | `make(chan struct{}, n)` | a counting semaphore |
| Unknown, unbounded volume | still pick an `n` | there is no unbounded channel, by design |

Idiomatic: start unbuffered. Add a buffer when you can name the number — the result count,
the burst size, the concurrency limit. A buffer size chosen by "that seemed to fix it" is
the bug in the latent-deadlock section above.

## Quick Reference

| Expression | Meaning |
|---|---|
| `make(chan T)` | capacity 0; every send is a rendezvous |
| `make(chan T, n)` | capacity `n`; sends 1..n do not block |
| `len(ch)` | values currently buffered (0 for unbuffered) |
| `cap(ch)` | buffer size, fixed at `make` time |
| `ch <- v` | blocks while `len(ch) == cap(ch)` |
| `<-ch` | blocks while `len(ch) == 0` |
| `make(chan struct{}, n)` | counting semaphore of `n` |
| `select { case ch <- v: ... default: ... }` | non-blocking send |

| Situation | Sender blocks? | Receiver blocks? |
|---|---|---|
| unbuffered, no peer | yes | yes |
| buffered, `len < cap` | no | — |
| buffered, `len == cap` | yes | — |
| buffered, `len == 0` | — | yes |
| buffered, `len > 0` | — | no |

## Common Mistakes

**Treating `make(chan T, n)` as an unbounded queue.** Past `n` the sender blocks like any
other; size `n` deliberately or add `select`/`default` to drop.

**Sizing the buffer by trial and error until the hang stops.** A buffer that works only
for the current goroutine count is a deadlock under load; drain concurrently instead.

**Branching on `len(ch)` to decide whether to send.** It is stale the moment you read it;
use `select` with `default` for a real non-blocking send.

**Writing `defer <-sem` to release a semaphore slot.** `defer` requires a function call;
write `defer func() { <-sem }()`.

**Assuming a buffered send gives you a happens-before edge with the receiver.** It does
not — only the matching receive does; see
[12.6](../12-concurrency-patterns/06-memory-model.md).

## Exercises

1. Write a program with `ch := make(chan int, 4)` that prints `len` and `cap` after each
   of six sends, using `select`/`default` so it reports the two sends that were refused
   rather than deadlocking.
2. Build a bounded worker pool using only a buffered channel as the semaphore (no
   `WaitGroup` for the limit) that processes 50 jobs with at most 4 in flight, and prove
   the bound with `atomic.Int64` exactly as the lesson does.
3. Take the four-senders-three-slots deadlock and fix it two ways: once by sizing the
   buffer correctly, once by draining in a separate goroutine. Explain which one survives
   the loop bound changing to 400.
4. Measure the cost: benchmark 1,000,000 sends and receives over `make(chan int)` versus
   `make(chan int, 1024)` with `go test -bench . -benchmem`, and explain the gap.
5. Implement a drop-oldest channel: a `send` that, when the buffer is full, discards the
   oldest buffered value to make room for the new one. Use `select` with `default` and a
   receive, and explain the race you have to tolerate.

---

[← 10.2 Unbuffered Channels](02-unbuffered-channels.md) · [Course Index](../README.md) · [10.4 Closing, range over a Channel, and the comma-ok Receive →](04-closing-range.md)
