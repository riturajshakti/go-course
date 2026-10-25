[← 10.3 Buffered Channels](03-buffered-channels.md) · [Course Index](../README.md) · [10.5 Directional Channels and nil Channels →](05-directional-nil.md)

# 10.4 — Closing, range over a Channel, and the comma-ok Receive

## What You'll Learn

- A closed channel yields buffered values first, then the zero value immediately and forever
- `v, ok := <-ch` is the only way to tell "a real zero" from "the channel is closed"
- Sending on, closing twice, or closing a nil channel all panic — three distinct messages
- `close` is a **broadcast**: one close wakes every blocked receiver at once

## Coming From Another Language

| Other | Go |
|---|---|
| JS `for await (const x of stream)` ends on the stream's end event | `for v := range ch` ends on `close(ch)` |
| Python generator raises `StopIteration` | the receive just returns `ok == false` |
| `asyncio.Queue` has no close; you push a sentinel | `close(ch)` is built in, and it broadcasts |
| Rust: dropping all senders closes the channel | you must call `close` explicitly; nothing is automatic |
| `AbortController.signal` — many listeners, one abort | `close(done)` — many receivers, one close |

Closing is not "destroying" the channel. The channel stays perfectly usable for receives;
what `close` changes is that a receive can no longer block. That one property is what
makes `close` the cheapest broadcast in the language: `n` goroutines blocked on `<-done`
all become runnable from a single `close(done)`, with no loop, no counter, and no
per-listener registration.

## close, then drain, then zeros forever

```go
package main

import "fmt"

func main() {
  ch := make(chan int, 2)
  ch <- 10
  ch <- 20
  close(ch)

  fmt.Println(<-ch)
  fmt.Println(<-ch)
  fmt.Println(<-ch)
  fmt.Println(<-ch)
  fmt.Println(len(ch), cap(ch))
}
```

Output:

```
10
20
0
0
0 2
```

Buffered values survive the close and come out in order. After the buffer empties, every
further receive returns the element type's zero value and returns it *immediately* — no
blocking, no panic, no end. You could receive a billion times.

This is why a receiver cannot detect the end of a stream from the value alone: `0` here is
indistinguishable from a real `0` that a sender put in.

## The comma-ok receive

```go
package main

import "fmt"

func main() {
  ch := make(chan string, 1)
  ch <- "payload"
  close(ch)

  v, ok := <-ch
  fmt.Printf("%q %v\n", v, ok)

  v, ok = <-ch
  fmt.Printf("%q %v\n", v, ok)
}
```

Output:

```
"payload" true
"" false
```

`ok` is `true` when the value came from a send, `false` when it is a synthetic zero from a
drained closed channel. This is the central idiom of the lesson — everything else in Go
that needs to distinguish "end of stream" from "a zero-valued element" is built on it.

Note `ok == true` for a value received *after* the close, as long as it was buffered. `ok`
reports where the value came from, not whether the channel is open.

## for range over a channel

`for v := range ch` is the comma-ok receive in a loop, exiting when `ok` goes false.

```go
package main

import "fmt"

func main() {
  ch := make(chan int)

  go func() {
    for i := 1; i <= 3; i++ {
      ch <- i
    }
    close(ch)
  }()

  for v := range ch {
    fmt.Println("got", v)
  }
  fmt.Println("range ended")

  v, ok := <-ch
  fmt.Println(v, ok)
}
```

Output:

```
got 1
got 2
got 3
range ended
0 false
```

Deterministic: each unbuffered send blocks until the loop takes it, so the values arrive
in order. The channel is still readable after the range ends — ranging does not consume
the channel, it consumes the values.

A channel `range` has exactly one loop variable. There is no index, because unlike a slice
there is no position to report.

## range without a close hangs

```go
package main

import "fmt"

func main() {
  ch := make(chan int)

  go func() {
    for i := 1; i <= 2; i++ {
      ch <- i
    }
  }()

  for v := range ch {
    fmt.Println("got", v)
  }
  fmt.Println("never printed")
}
```

This deadlocks on purpose. Output:

```
got 1
got 2
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
	/tmp/scratch/main.go:14 +0xdc
exit status 2
```

The sender finished and exited without closing, so the `range` is waiting for a value that
can never come. Forgetting the `close` on a producer goroutine is the single most common
way to hang a Go program.

You only got the `fatal error` because *every* goroutine was asleep. If anything else in
the process were still running — an HTTP server, a ticker — this receiver would hang
silently forever. That is a goroutine leak, covered in
[12.5](../12-concurrency-patterns/05-deadlocks-leaks.md).

## Send on a closed channel panics

```go
package main

import "fmt"

func main() {
  ch := make(chan int, 1)
  close(ch)
  fmt.Println("closed")
  ch <- 1
}
```

This program panics on purpose. Output:

```
closed
panic: send on closed channel

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:9 +0x74
exit status 2
```

There is no way to test for this. `len`, `cap`, and the comma-ok receive all tell you
nothing about whether a channel is closed from the sending side, and there is no
`isClosed(ch)`. The only defence is structural: make the ownership of `close` unambiguous.

## Close of a closed channel panics

```go
package main

import "fmt"

func main() {
  ch := make(chan int)
  close(ch)
  fmt.Println("closed once")
  close(ch)
}
```

This program panics on purpose. Output:

```
closed once
panic: close of closed channel

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:9 +0x6c
exit status 2
```

## Close of a nil channel panics

```go
package main

import "fmt"

func main() {
  var ch chan int
  fmt.Println(ch == nil, len(ch), cap(ch))
  close(ch)
}
```

This program panics on purpose. Output:

```
true 0 0
panic: close of nil channel

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:8 +0xa4
exit status 2
```

`len` and `cap` on a nil channel are both 0 and do not panic, which is exactly why the
nil case slips through — see [10.5](05-directional-nil.md).

## The sender closes, never the receiver

Three distinct panics, one rule: **the goroutine that sends is the goroutine that
closes**. A receiver that closes can cause a `send on closed channel` panic in a sender it
has no control over, and a second closer causes `close of closed channel`. Keeping `close`
with the sender makes both impossible by construction.

The type system helps here. A function given a `<-chan T` physically cannot call `close`
on it — the compiler rejects it ([10.5](05-directional-nil.md)).

## Multiple senders: WaitGroup, then close

With more than one sender, no single sender knows it was the last. Put the `close` in a
goroutine that waits for all of them.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
)

func main() {
  out := make(chan string)

  var wg sync.WaitGroup
  for id := 1; id <= 3; id++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      for n := 1; n <= 2; n++ {
        out <- fmt.Sprintf("sender%d-msg%d", id, n)
      }
    }()
  }

  go func() {
    wg.Wait()
    close(out)
  }()

  var got []string
  for v := range out {
    got = append(got, v)
  }
  sort.Strings(got)
  for _, v := range got {
    fmt.Println(v)
  }
  fmt.Println("count:", len(got))
}
```

Output:

```
sender1-msg1
sender1-msg2
sender2-msg1
sender2-msg2
sender3-msg1
sender3-msg2
count: 6
```

The arrival order genuinely varies, so the program sorts before printing. The `count: 6`
is what is actually being asserted: every message got through and the range terminated.

Two details carry the whole pattern. The `wg.Wait()` is in its own goroutine, because
calling it in `main` before the `range` would deadlock — the senders cannot finish until
something drains `out`. And `wg.Add(1)` is before the `go`, never inside it, or `Wait`
can return before a goroutine has registered. `sync.WaitGroup` is
[11.1](../11-sync-context/01-waitgroup.md).

Idiomatic: one sender closes directly, often with `defer close(out)` as the first line of
the producer. Many senders means a `WaitGroup` and a dedicated closer goroutine.

## close as broadcast

This is the property that makes `close` worth more than everything else in this lesson.
Four goroutines block on `<-quit`. One `close` releases all four — no loop over listeners,
no counting, no bookkeeping.

```go
package main

import (
  "fmt"
  "runtime"
  "sort"
  "sync"
)

func main() {
  quit := make(chan struct{})
  ready := make(chan int, 4)
  results := make(chan string, 4)
  var wg sync.WaitGroup

  for id := 1; id <= 4; id++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      ready <- id
      <-quit
      results <- fmt.Sprintf("worker %d: saw quit", id)
    }()
  }

  for range 4 {
    <-ready
  }
  fmt.Println("goroutines before close:", runtime.NumGoroutine())

  close(quit)
  wg.Wait()
  close(results)

  var lines []string
  for r := range results {
    lines = append(lines, r)
  }
  sort.Strings(lines)
  for _, l := range lines {
    fmt.Println(l)
  }
  fmt.Println("goroutines after:", runtime.NumGoroutine())
}
```

Output:

```
goroutines before close: 5
worker 1: saw quit
worker 2: saw quit
worker 3: saw quit
worker 4: saw quit
goroutines after: 1
```

No `time.Sleep` anywhere: the four `<-ready` receives prove all four workers have reached
their `<-quit` before `main` closes it, and `runtime.NumGoroutine()` confirms 5 (four
workers plus `main`). This is why the output is byte-identical every run.

Sending would not work here. A send on `quit` is taken by exactly one receiver, so waking
four workers would require four sends — and you would need to know there were four.
`close` needs no count, which is precisely why it scales to "however many goroutines
happen to be listening".

`chan struct{}` is the conventional type for this: you never send a value, so the element
type should carry none. `context.Context` is this pattern with a parent/child tree and a
cancellation reason attached, in
[11.6](../11-sync-context/06-context-cancel.md).

## Draining after close

A `range` over a closed channel still yields whatever is buffered, which is how you
collect results after the producers are done. Both earlier examples do exactly that:
`close(results)` then `for r := range results`. Because the channel is closed, the range
terminates on its own instead of blocking after the last value.

The trap is the reverse: closing a channel whose buffer still has values and then
*not* draining. The values are simply discarded when the channel is collected. Closing
does not flush anything anywhere.

## Quick Reference

| Operation | Open channel | Closed channel | nil channel |
|---|---|---|---|
| `ch <- v` | blocks or buffers | **panics** | blocks forever |
| `<-ch` | blocks or yields | zero value, no block | blocks forever |
| `v, ok := <-ch` | `ok == true` | drains, then `ok == false` | blocks forever |
| `for range ch` | loops | drains then exits | blocks forever |
| `close(ch)` | closes | **panics** | **panics** |
| `len`/`cap` | fine | fine | `0`, `0` |

| Panic text | Cause |
|---|---|
| `send on closed channel` | a sender raced or outlived the close |
| `close of closed channel` | two goroutines both own the close |
| `close of nil channel` | the channel was never `make`d |

| Shape | Who closes |
|---|---|
| one producer | the producer, usually `defer close(out)` |
| `n` producers | a goroutine running `wg.Wait(); close(out)` |
| a quit/done signal | whoever decides to stop; receivers never close |
| a receive-only `<-chan T` | nobody — the compiler forbids it |

## Common Mistakes

**Testing `v != 0` or `v != ""` to detect the end of a stream.** A sender can send a real
zero; use `v, ok := <-ch`.

**Forgetting `close` in a producer goroutine.** Every `range` over that channel hangs
forever; put `defer close(out)` at the top of the producer.

**Letting the receiver close the channel.** It panics the sender; close belongs to the
sender, and `<-chan T` parameters make that structural.

**Calling `wg.Wait()` in `main` before ranging over the output channel.** The senders
block on an undrained channel and nothing finishes; run `wg.Wait(); close(out)` in its own
goroutine.

**Sending on a done channel instead of closing it.** Only one receiver wakes up; `close`
is the broadcast and the only thing that scales to `n` listeners.

## Exercises

1. Write a producer that sends five `int`s where the third one is `0`, close the channel,
   and have the receiver print every value with its `ok`. Show that the real `0` and the
   post-close `0` are distinguishable only by `ok`.
2. Build a fan-in of three producers over one unbuffered channel with the
   `wg.Wait(); close(out)` pattern. Then deliberately move `wg.Add(1)` inside the
   goroutine and show the output it produces.
3. Recreate all three close-related panics in one program, each recovered with `defer` +
   `recover`, and print the three panic values. Compare them with the uncaught text in
   this lesson.
4. Implement `func first[T any](chans ...<-chan T) (T, bool)` that returns the first value
   from whichever channel produces one, using `close` on a shared `done` channel to stop
   the losers. (You will need `select` from [10.6](06-select.md).)
5. Write a start-gate: 100 goroutines that all block on `<-start`, then measure with
   `time.Since` how long after `close(start)` the last one runs. Compare with sending 100
   values on an unbuffered channel instead.

---

[← 10.3 Buffered Channels](03-buffered-channels.md) · [Course Index](../README.md) · [10.5 Directional Channels and nil Channels →](05-directional-nil.md)
