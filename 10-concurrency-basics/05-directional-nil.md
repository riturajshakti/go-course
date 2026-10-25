[← 10.4 Closing, range over a Channel, and the comma-ok Receive](04-closing-range.md) · [Course Index](../README.md) · [10.6 select, default, and Timeouts →](06-select.md)

# 10.5 — Directional Channels and nil Channels

## What You'll Learn

- `chan<- T` sends only, `<-chan T` receives only; narrowing at a function boundary is free documentation
- The conversion is one-way: bidirectional narrows to directional, never back
- A nil channel blocks forever on both send and receive — and never panics, so it hides
- That blocking is the feature: setting a channel to `nil` disables its `select` case

## Coming From Another Language

| Other | Go |
|---|---|
| Rust `mpsc::Sender<T>` / `Receiver<T>` — distinct types | `chan<- T` / `<-chan T` — views of one type |
| Rust: a `Receiver` cannot be cloned | a `<-chan T` can be copied freely; receivers share the stream |
| TS `ReadableStream` vs `WritableStream` | the same split, enforced by the compiler |
| Python: no direction in the type; convention only | the compiler rejects a wrong-direction operation |
| A `None` queue in Python raises `AttributeError` | a nil channel **silently blocks forever** |

The arrow in the type is the arrow in the operation. `chan<- T` reads "send into the
channel", `<-chan T` reads "receive out of the channel". Unlike Rust's `Sender`/`Receiver`
these are not separate objects with separate ownership — they are restricted views of one
channel value, and you can make as many of either as you like.

The nil-channel rule is the one with no analogue anywhere else: receiving from or sending
to a nil channel is not an error, it blocks forever. That makes an uninitialized channel
field a hang rather than a crash, and it also makes nil the cleanest way to turn a
`select` case off.

## Narrowing at the function boundary

```go
package main

import "fmt"

func produce(out chan<- int, n int) {
  for i := 1; i <= n; i++ {
    out <- i
  }
  close(out)
}

func consume(in <-chan int) int {
  sum := 0
  for v := range in {
    sum += v
  }
  return sum
}

func main() {
  ch := make(chan int)
  go produce(ch, 5)
  fmt.Println(consume(ch))
}
```

Output:

```
15
```

`main` holds a bidirectional `chan int` and passes it to both functions; the conversion to
`chan<- int` and `<-chan int` is implicit. The signatures now state the contract: `produce`
writes and closes, `consume` reads. Nobody has to read either body to know that, and
nobody can get it wrong — `consume` cannot send, and cannot `close`.

Idiomatic: always narrow channel parameters. A bidirectional `chan T` in a signature
should make you ask why the function needs both halves.

## The conversion does not go backwards

```go
package main

import "fmt"

func main() {
  bidi := make(chan int, 1)

  var send chan<- int = bidi
  var recv <-chan int = bidi

  send <- 42
  fmt.Println(<-recv)

  var back chan int = send
  fmt.Println(back)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:14:23: cannot use send (variable of type chan<- int) as chan int value in variable declaration
```

Once narrowed, a channel cannot be widened — not by assignment, and not by a type
conversion. The restriction is permanent for that value, which is what makes it a real
guarantee rather than a hint. (An `unsafe` or `reflect` route exists; it is not something
to do.)

## The wrong direction is a compile error, not a runtime one

```go
package main

import "fmt"

func main() {
  ch := make(chan int, 1)
  var send chan<- int = ch
  send <- 1
  fmt.Println(<-send)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:17: invalid operation: cannot receive from send-only channel chan<- int send (variable of type chan<- int)
```

And the one that matters most — you cannot close what you can only read from:

```go
package main

func main() {
  ch := make(chan int, 1)
  var recv <-chan int = ch
  <-recv
  close(recv)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:9: invalid operation: cannot close receive-only channel recv (variable of type <-chan int)
```

This turns the "the sender closes, never the receiver" rule from
[10.4](04-closing-range.md) into something the compiler enforces. Hand consumers a
`<-chan T` and the `close of closed channel` panic becomes unreachable.

## The generator: returning <-chan T

A function that spawns a producer and returns the read end is the standard Go generator.

```go
package main

import "fmt"

func squares(n int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for i := 1; i <= n; i++ {
      out <- i * i
    }
  }()
  return out
}

func main() {
  for v := range squares(6) {
    fmt.Print(v, " ")
  }
  fmt.Println()
}
```

Output:

```
1 4 9 16 25 36 
```

Three things make this the canonical shape. `out` is created and closed inside, so
ownership of `close` never leaves. The return type is `<-chan int`, so the caller cannot
close it or send into it. And `defer close(out)` is the first statement of the goroutine,
so the close happens on every exit path — this is what stops the caller's `range` from
hanging.

This is a generator, not a promise: nothing is computed until a receiver takes a value,
because the unbuffered send blocks. Note too that it is not cancellable as written — if
the caller stops ranging early, the producer goroutine blocks on its next send forever.
Fixing that needs a `done` channel and `select` ([10.6](06-select.md)); pipelines with
proper cancellation are
[12.3](../12-concurrency-patterns/03-pipelines.md).

## nil channels block forever

```go
package main

import "fmt"

func main() {
  var ch chan int
  fmt.Println("nil channel:", ch == nil, len(ch), cap(ch))
  ch <- 1
  fmt.Println("never printed")
}
```

This deadlocks on purpose. Output:

```
nil channel: true 0 0
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
main.main()
	/tmp/scratch/main.go:8 +0xc8
exit status 2
```

The runtime names the state exactly: `chan send (nil chan)`. Receiving is the same — a
`<-ch` on a nil channel parks the goroutine with no timeout and no wakeup. There is no
panic, which is the dangerous part: compare `close` of a nil channel, which panics
immediately ([10.4](04-closing-range.md)).

Note `len` and `cap` return 0 without complaint, so neither one can be used to detect nil.
Only `ch == nil` can.

## The real-world version: a forgotten make

The zero value of a struct field of channel type is nil. A constructor that forgets one
field produces an object that hangs on first use rather than failing.

```go
package main

import "fmt"

type Server struct {
  events chan string
}

func main() {
  s := &Server{}
  fmt.Println("s.events is nil:", s.events == nil)
  s.events <- "boot"
  fmt.Println("never printed")
}
```

This deadlocks on purpose. Output:

```
s.events is nil: true
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send (nil chan)]:
main.main()
	/tmp/scratch/main.go:12 +0x74
exit status 2
```

`&Server{}` compiles, `go vet` is silent, and in a program with any other live goroutine
there is no fatal error either — just one goroutine stuck forever. Channels must be made
in a constructor; a struct with channel fields should not have a usable zero value.

## Why a closed channel cannot be the "off" switch

Before the payoff, the problem it solves. A closed channel is *permanently ready*, so a
`select` case reading a drained closed channel fires immediately, every time, forever.

```go
package main

import "fmt"

func main() {
  a := make(chan string, 1)
  b := make(chan string, 1)
  a <- "only value"
  close(a)
  close(b)

  spins := 0
  for spins < 5 {
    select {
    case v, ok := <-a:
      spins++
      fmt.Printf("a: %q ok=%v\n", v, ok)
    case v, ok := <-b:
      spins++
      fmt.Printf("b: %q ok=%v\n", v, ok)
    }
  }
  fmt.Println("a closed channel is always ready; it spins forever without the nil trick")
}
```

Which case wins each spin is random, so the interleaving differs every run. Three real
runs:

```
a: "only value" ok=true
a: "" ok=false
a: "" ok=false
a: "" ok=false
b: "" ok=false
a closed channel is always ready; it spins forever without the nil trick
```

```
a: "only value" ok=true
a: "" ok=false
b: "" ok=false
a: "" ok=false
a: "" ok=false
a closed channel is always ready; it spins forever without the nil trick
```

```
b: "" ok=false
b: "" ok=false
b: "" ok=false
a: "only value" ok=true
b: "" ok=false
a closed channel is always ready; it spins forever without the nil trick
```

The `spins < 5` bound is only there so the program terminates; without it this is a busy
loop that burns a core producing zero values. So "drop the case when the channel closes"
cannot be done by closing — it has to be done by making the channel unselectable.

## Setting a channel to nil to disable a select case

A nil channel in a `select` case is never ready, so the case is effectively deleted. That
is the whole mechanism, and it turns the merge-until-both-closed pattern into eight lines.

```go
package main

import "fmt"

func gen(prefix string, n int) <-chan string {
  out := make(chan string)
  go func() {
    defer close(out)
    for i := 1; i <= n; i++ {
      out <- fmt.Sprintf("%s%d", prefix, i)
    }
  }()
  return out
}

func merge(a, b <-chan string) <-chan string {
  out := make(chan string)
  go func() {
    defer close(out)
    for a != nil || b != nil {
      select {
      case v, ok := <-a:
        if !ok {
          a = nil
          continue
        }
        out <- v
      case v, ok := <-b:
        if !ok {
          b = nil
          continue
        }
        out <- v
      }
    }
  }()
  return out
}

func main() {
  counts := map[string]int{}
  total := 0
  for v := range merge(gen("a", 3), gen("b", 2)) {
    counts[v]++
    total++
  }
  fmt.Println("total:", total)
  for _, k := range []string{"a1", "a2", "a3", "b1", "b2"} {
    fmt.Println(k, counts[k])
  }
}
```

Output:

```
total: 5
a1 1
a2 1
a3 1
b1 1
b2 1
```

The *order* values come out of `merge` varies, which is why the program counts instead of
printing a sequence. What is deterministic, and what is actually being asserted, is that
all five values arrive exactly once and the loop terminates.

Read the loop condition and the two assignments together. `a = nil` removes that case from
the next `select`, so the loop stops re-firing on the drained channel and keeps serving
`b`. When both are nil, `for a != nil || b != nil` ends and `defer close(out)` terminates
the caller's range. Without the nil assignment this is the busy-spin from the previous
section; without the loop condition it never exits.

Note the local parameters `a` and `b` are being reassigned — that is fine, they are copies
of the caller's channel values, and setting one to nil does not affect the caller.
Fan-in in general is
[12.2](../12-concurrency-patterns/02-fan-out-fan-in.md).

## Channels are comparable

```go
package main

import "fmt"

func main() {
  a := make(chan int)
  b := a
  c := make(chan int)
  var n chan int

  fmt.Println(a == b, a == c, n == nil)

  seen := map[chan int]string{a: "a", c: "c"}
  fmt.Println(seen[b], seen[c])
}
```

Output:

```
true false true
a c
```

`==` compares channel identity, not contents: two channels are equal when they are the
same channel. Because they are comparable they are valid map keys and valid struct fields
for a comparable struct. Directional views of one channel compare equal to each other too.

## Channel of channels: request and reply

Putting a reply channel inside the request is how you get a response back to the right
caller, and it is where `chan chan T` stops looking exotic.

```go
package main

import "fmt"

type request struct {
  n     int
  reply chan int
}

func server(reqs <-chan request) {
  for r := range reqs {
    r.reply <- r.n * 2
  }
}

func main() {
  reqs := make(chan request)
  go server(reqs)

  for _, n := range []int{1, 5, 9} {
    reply := make(chan int)
    reqs <- request{n: n, reply: reply}
    fmt.Println(n, "->", <-reply)
  }
  close(reqs)
}
```

Output:

```
1 -> 2
5 -> 10
9 -> 18
```

Fully deterministic: each iteration blocks on `<-reply` before sending the next request.
One fresh reply channel per request means the server never has to track who asked, and no
reply can be delivered to the wrong caller. A buffered `make(chan int, 1)` for the reply
would let the server finish and move on even if the caller abandoned the request — the
usual choice in a real service.

## Quick Reference

| Type | Send | Receive | `close` | Note |
|---|---|---|---|---|
| `chan T` | yes | yes | yes | what `make` returns |
| `chan<- T` | yes | **compile error** | yes | a sender's view |
| `<-chan T` | **compile error** | yes | **compile error** | a receiver's view |

| Conversion | Allowed |
|---|---|
| `chan T` → `chan<- T` | yes, implicit |
| `chan T` → `<-chan T` | yes, implicit |
| `chan<- T` → `chan T` | no |
| `<-chan T` → `chan T` | no |
| `chan<- T` ↔ `<-chan T` | no |

| On a nil channel | Result |
|---|---|
| `ch <- v` | blocks forever |
| `<-ch` | blocks forever |
| `close(ch)` | **panics** |
| `len(ch)`, `cap(ch)` | `0`, `0` |
| a `select` case on it | never ready — the case is disabled |

## Common Mistakes

**Declaring `var ch chan T` and using it.** That is a nil channel and it blocks forever
with no panic; `ch := make(chan T)`.

**Leaving a channel struct field unset in a constructor.** The zero value is nil, so the
first send hangs the caller; make every channel field explicitly.

**Taking a bidirectional `chan T` parameter out of habit.** Narrow it to `chan<- T` or
`<-chan T` so the signature states the direction and the compiler enforces it.

**Closing a channel to remove its `select` case.** A closed channel is always ready and
you get a busy loop; set the variable to `nil` instead.

**Expecting a nil channel to panic like a nil map write.** It does not — it blocks, which
is far harder to find; only `close` of nil panics.

## Exercises

1. Write `func split(in <-chan int) (evens, odds <-chan int)` that fans one input into two
   output channels, closing both when the input closes. Prove the return types prevent the
   caller from closing them.
2. Take the `merge` function and extend it to `merge(chans ...<-chan string) <-chan string`
   for an arbitrary number of inputs. Explain why the nil-disabling trick does not scale
   to `n` cases in a static `select`, and what you use instead.
3. Build a `Server` struct with three channel fields and a `NewServer` constructor. Write
   a test that constructs it with `&Server{}` and shows the hang, bounding the test with
   `time.After` so it fails instead of blocking the suite.
4. Write a request/reply server where each reply channel is buffered with capacity 1, and
   a caller that abandons its request without reading the reply. Show with
   `runtime.NumGoroutine` that the server goroutine is not leaked, then change the reply
   channel to unbuffered and show that it is.
5. Use channel comparability: build a `map[<-chan int]string` naming several generator
   channels, merge them, and report which source each value came from. Explain what you
   had to change about `merge` to make that possible.

---

[← 10.4 Closing, range over a Channel, and the comma-ok Receive](04-closing-range.md) · [Course Index](../README.md) · [10.6 select, default, and Timeouts →](06-select.md)
