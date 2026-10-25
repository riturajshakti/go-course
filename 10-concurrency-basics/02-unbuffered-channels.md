[← 10.1 Goroutines and the Scheduler](01-goroutines.md) · [Course Index](../README.md) · [10.3 Buffered Channels →](03-buffered-channels.md)

# 10.2 — Unbuffered Channels

## What You'll Learn

- An unbuffered channel is a **rendezvous**, not a queue: the send blocks until a receiver arrives, and vice versa
- That blocking is what makes concurrent output deterministic — a handoff orders both goroutines
- A send with no receiver (or a receive with no sender) is a `fatal error: all goroutines are asleep - deadlock!`
- `close(done)` on a `chan struct{}` is the idiomatic zero-cost signal

## Coming From Another Language

| Other | Go |
|---|---|
| JS promise — settles once, caches the value, many `await`s see it | a channel value is consumed by exactly **one** receiver |
| A promise can be awaited after it resolved | an unbuffered send that nobody is waiting for just blocks |
| `asyncio.Queue(maxsize=0)` — unbounded | `make(chan T)` — capacity **zero**, every send blocks |
| `asyncio.Queue(maxsize=1)` | `make(chan T, 1)` — see [10.3](03-buffered-channels.md) |
| Rust `tokio::sync::oneshot` | the closest analogue to a result channel, but reusable |
| Rust `std::sync::mpsc::sync_channel(0)` | the exact same rendezvous semantics |

A channel is not a promise. A promise is a *memoized* value: it resolves once, and every
consumer that awaits it gets the same result, forever. A channel is a *conduit*: each
value sent is received by exactly one receiver and then it is gone. Sending twice means
two receives. Two goroutines receiving from one channel split the values between them,
they do not each get a copy.

And `make(chan T)` with no size is not "unbounded". It is capacity **zero**. There is
nowhere to put a value, so a send cannot complete until a receiver is standing there to
take it directly. That is the single most important fact in this lesson.

## make, send, receive

```go
package main

import "fmt"

func main() {
  ch := make(chan string)
  go func() {
    ch <- "ping"
  }()
  v := <-ch
  fmt.Println(v)
  fmt.Println(len(ch), cap(ch))
}
```

Output:

```
ping
0 0
```

`cap(ch)` is 0 — that is what "unbuffered" means. `len(ch)` on an unbuffered channel is
always 0 too, because a value never rests inside it.

The three operations:

| Form | Meaning |
|---|---|
| `ch <- v` | send `v`; blocks until a receiver takes it |
| `v := <-ch` | receive into `v`; blocks until a sender offers one |
| `<-ch` | receive and discard the value; still blocks |

The arrow always points in the direction the data moves, which is why the send has the
channel on the left and the receive has it on the right.

## The rendezvous proves ordering

Because both sides must meet, a channel handoff imposes an order on two goroutines that
otherwise race. A ping-pong makes that order visible, and it is identical on every run.

```go
package main

import "fmt"

func main() {
  ping := make(chan int)
  pong := make(chan int)

  go func() {
    for i := 0; i < 3; i++ {
      v := <-ping
      fmt.Println("B received", v)
      pong <- v * 10
    }
  }()

  for i := 1; i <= 3; i++ {
    fmt.Println("A sending", i)
    ping <- i
    fmt.Println("A received", <-pong)
  }
}
```

Output (stable across runs):

```
A sending 1
B received 1
A received 10
A sending 2
B received 2
A received 20
A sending 3
B received 3
A received 30
```

Nothing in here uses a lock, an atomic, or a sleep, and yet the two goroutines strictly
alternate. `A` cannot print "A received" until `B` has sent on `pong`, and `B` cannot
reach that send until it has received from `ping`. The channels are the schedule.

Contrast with [10.1](01-goroutines.md), where five goroutines printed in a different order
on every run. The difference is entirely the handoff.

## Deadlock: nobody receives

```go
package main

import "fmt"

func main() {
  ch := make(chan int)
  fmt.Println("sending")
  ch <- 42
  fmt.Println("unreachable")
}
```

This deadlocks on purpose. Output:

```
sending
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/tmp/scratch/main.go:8 +0x70
exit status 2
```

This is a **runtime** detection, not a compile-time one, and it is a `fatal error`, not a
`panic`: it cannot be recovered. The runtime notices that every single goroutine is
blocked on something no other goroutine can supply, so no progress is possible ever
again, and it dumps every goroutine's state. The `[chan send]` in the header tells you
exactly what goroutine 1 is stuck on.

Important limitation: Go only reports this when **all** goroutines are asleep. One
goroutine blocked forever while others keep working is a goroutine leak, and the runtime
says nothing at all. That asymmetry is the subject of
[12.5 Deadlocks, Leaks, and How to Spot Them](../12-concurrency-patterns/05-deadlocks-leaks.md).

### Send and receive on the same goroutine

Even with both operations present, one goroutine cannot rendezvous with itself.

```go
package main

import "fmt"

func main() {
  ch := make(chan int)
  ch <- 1
  fmt.Println(<-ch)
}
```

This deadlocks on purpose. Output:

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/tmp/scratch/main.go:7 +0x38
exit status 2
```

Line 7 is `ch <- 1`. The receive on the next line is unreachable, because the send never
completes. This compiles cleanly and is a classic first-day mistake. The fix is either a
goroutine on one side, or a buffer of 1 ([10.3](03-buffered-channels.md)).

### Deadlock with multiple blocked goroutines

The dump lists every goroutine, which is what makes it useful.

```go
package main

import "fmt"

func produce(ch chan int, v int) {
  ch <- v
  fmt.Println("produced", v)
}

func main() {
  ch := make(chan int)
  done := make(chan struct{})
  for i := 1; i <= 3; i++ {
    go produce(ch, i)
  }
  fmt.Println("main never receives")
  <-done
}
```

This deadlocks on purpose. Output (goroutines 35 and 36 are identical to 34 and have been
truncated here):

```
main never receives
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
	/tmp/scratch/main.go:17 +0xf4

goroutine 34 [chan send]:
main.produce(0x0?, 0x1)
	/tmp/scratch/main.go:6 +0x24
created by main.main in goroutine 1
	/tmp/scratch/main.go:14 +0x58
exit status 2
```

Read it as a list of suspects: goroutine 1 wants a receive on `done` that will never
come, and three producers want a send on `ch` that nobody will take. `created by
main.main` tells you where each goroutine was started, which is usually more useful than
where it is blocked.

## An unbuffered channel as a signal

When you only care *that* something happened, not what the value was, use
`chan struct{}` and `close` it. `struct{}` occupies zero bytes, and a closed channel
makes every receive return immediately — so one `close` can wake any number of waiters.

```go
package main

import "fmt"

func main() {
  done := make(chan struct{})
  go func() {
    defer close(done)
    fmt.Println("step 1")
    fmt.Println("step 2")
  }()
  <-done
  fmt.Println("main continues")

  fmt.Println(<-done)
  fmt.Println(<-done)
}
```

Output:

```
step 1
step 2
main continues
{}
{}
```

Receiving from the closed channel returns the zero value of `struct{}` — printed as `{}`
— immediately, forever. That is why `close` is a broadcast and a single `done <-
struct{}{}` is not: a send reaches exactly one waiter.

Idiomatic: `done := make(chan struct{})` plus `defer close(done)`. Use `chan bool` only
when the boolean actually carries information. Full treatment of close semantics in
[10.4](04-closing-range.md).

## A channel as a function's return value

The cleanest way to get *one* result out of a goroutine is to return the channel it will
be delivered on.

```go
package main

import (
  "fmt"
  "strings"
)

type result struct {
  n   int
  err error
}

func work(in string) <-chan result {
  out := make(chan result)
  go func() {
    if in == "" {
      out <- result{err: fmt.Errorf("empty input")}
      return
    }
    out <- result{n: len(strings.Fields(in))}
  }()
  return out
}

func main() {
  r := <-work("the quick brown fox")
  fmt.Println(r.n, r.err)
  r = <-work("")
  fmt.Println(r.n, r.err)
}
```

Output:

```
4 <nil>
0 empty input
```

Two things to notice. The error travels **in** the value, because a channel carries one
type — there is no second channel for errors and no `reject` path. And the return type is
`<-chan result`, receive-only, so the caller cannot send into it or close it; that is
[10.5](05-directional-nil.md).

This is Go's nearest thing to a promise, and the difference is instructive: calling
`work` twice gives you two independent channels, and each result can be received exactly
once. There is no caching, no `.then`, and no error-to-rejection mapping.

## Channels are reference types

`make(chan T)` returns a reference to a runtime structure. Copying the variable, passing
it to a function, or storing it in a struct all share the same channel — you never need a
`*chan T`.

```go
package main

import "fmt"

func fill(ch chan int) {
  ch <- 7
}

func main() {
  a := make(chan int)
  b := a
  fmt.Println(a == b)
  go fill(a)
  fmt.Println(<-b)

  var nilCh chan int
  fmt.Println(nilCh == nil, nilCh)
}
```

Output:

```
true
7
true <nil>
```

`fill` sent on `a` and `main` received on `b`, because they are the same channel. `==`
compares channel identity, not contents.

The last line is the one to remember: the **zero value of a channel type is nil**. A
`chan T` field in a struct you did not initialize is nil, and operating on a nil channel
blocks forever rather than panicking — which turns a forgotten `make` into a silent hang.
That, plus the one genuinely useful thing nil channels are good for, is
[10.5](05-directional-nil.md).

## fmt.Println is not a synchronization point

`fmt.Println` writes to `os.Stdout` with an internal lock, so concurrent calls do not
interleave *within* a line and the program is race-free. But the lock says nothing about
which goroutine gets there first. Print order from unsynchronized goroutines is
scheduler order, and using prints to "prove" an ordering is how people convince
themselves of concurrency facts that are not true. If you need an order, create it with a
handoff, as the ping-pong above does.

## A complete worker example

Three workers, four jobs, results collected and sorted by job ID so the output is
deterministic even though the work was distributed unpredictably.

```go
package main

import (
  "fmt"
  "sort"
  "strings"
)

type job struct {
  id   int
  text string
}

type outcome struct {
  id    int
  words int
}

func worker(jobs <-chan job, results chan<- outcome) {
  for j := range jobs {
    results <- outcome{id: j.id, words: len(strings.Fields(j.text))}
  }
}

func main() {
  jobs := make(chan job)
  results := make(chan outcome)

  for w := 0; w < 3; w++ {
    go worker(jobs, results)
  }

  inputs := []string{"one", "one two", "one two three", "a b c d"}
  go func() {
    for i, s := range inputs {
      jobs <- job{id: i, text: s}
    }
    close(jobs)
  }()

  var got []outcome
  for range inputs {
    got = append(got, <-results)
  }
  sort.Slice(got, func(i, j int) bool { return got[i].id < got[j].id })
  for _, o := range got {
    fmt.Printf("job %d: %d words\n", o.id, o.words)
  }
}
```

Output:

```
job 0: 1 words
job 1: 2 words
job 2: 3 words
job 3: 4 words
```

The pattern to internalize: **distribute unpredictably, collect deterministically.** Which
worker handled which job varies per run; the output does not, because the main goroutine
knows it expects exactly `len(inputs)` results and sorts them. `go run -race .` reports no
races — the only shared state is the two channels, and channel operations are
synchronization points. Worker pools in depth:
[12.1](../12-concurrency-patterns/01-worker-pools.md).

Note the feeding goroutine. Sending four jobs from `main` and *then* receiving would
deadlock on the first send once the workers were busy, because an unbuffered send cannot
complete until someone receives. Producer in a goroutine, consumer in `main`, is the
standard shape.

## Cost of the rendezvous

Every unbuffered send is a scheduling event: park the sender, wake the receiver, hand the
value over directly. For a hot pipeline that is measurable overhead, and a small buffer
lets a producer run ahead instead of stopping on every item — which is
[10.3 Buffered Channels](03-buffered-channels.md).

## Quick Reference

| Expression | Behaviour on an unbuffered channel |
|---|---|
| `make(chan T)` | capacity 0; every send needs a waiting receiver |
| `ch <- v` | blocks until a receiver takes `v` |
| `v := <-ch` | blocks until a sender offers a value |
| `<-ch` | receive, discard the value, still blocks |
| `len(ch)` | always 0 |
| `cap(ch)` | always 0 |
| `a == b` | true if the same channel; `ch == nil` tests the zero value |
| `close(done)` | wakes every current and future receiver at once |

| Situation | Result |
|---|---|
| Send, no receiver anywhere | `fatal error: all goroutines are asleep - deadlock!` |
| Receive, no sender anywhere | same fatal error |
| Send then receive, same goroutine | deadlock on the send |
| Send, receiver blocked elsewhere forever | goroutine leak; no error reported |
| `chan struct{}` + `close` | broadcast signal, zero bytes per value |

## Common Mistakes

**Thinking `make(chan T)` is an unbounded queue.** Its capacity is zero, so the first
send blocks until someone receives; add a size if you want buffering.

**Sending and receiving in the same goroutine.** It deadlocks on the send; move one side
into a goroutine or use `make(chan T, 1)`.

**Expecting several receivers to each see the same sent value.** A value goes to exactly
one receiver — to broadcast, `close` the channel instead.

**Using `done <- struct{}{}` to signal N waiters.** Only one wakes up; `close(done)` wakes
all of them.

**Filling a channel from `main` before starting the consumer.** The first send blocks
forever; run the producer in a goroutine.

**Reading print order as a guarantee.** `fmt.Println` locks stdout but imposes no
ordering; sequence the goroutines with a channel if order matters.

## Exercises

1. Write `func alternate(n int)` that makes two goroutines print `odd 1`, `even 2`,
   `odd 3` … up to `n`, strictly alternating, using two unbuffered channels and no
   `sync` package. Confirm the output is identical on 20 runs.
2. Implement `func first(a, b <-chan int) int` that returns whichever channel produces a
   value first — without `select`. Explain why this is awkward, then note how
   [10.6](06-select.md) solves it in three lines.
3. Take the deadlock example with the three blocked producers and fix it three different
   ways: add a receiver loop, add a buffer, and give the producers an escape hatch. For
   each, say what happens to the values that were in flight.
4. Build `func pipeline(nums []int) <-chan int` as three stages (generate, square, sum)
   wired with unbuffered channels, each stage a goroutine. Print the final sum.
5. Write a program that leaks exactly one goroutine blocked on a channel send, and show
   with `runtime.NumGoroutine` that it is still there at the end of `main`, while the
   deadlock detector never fires. Explain why it does not.

---

[← 10.1 Goroutines and the Scheduler](01-goroutines.md) · [Course Index](../README.md) · [10.3 Buffered Channels →](03-buffered-channels.md)
