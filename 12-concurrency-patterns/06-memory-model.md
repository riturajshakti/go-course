[← 12.5 Deadlocks, Leaks, and How to Spot Them](05-deadlocks-leaks.md) · [Course Index](../README.md) · [13.1 os: Files, Args, and Environment →](../13-stdlib/01-os.md)

# 12.6 — The Go Memory Model in Practice

## What You'll Learn

- Without synchronization, one goroutine's writes may never become visible to another
- The six happens-before rules you actually use, each in three lines of code
- Why `var done bool` plus a spin loop is broken even though it usually appears to work
- Go's atomics are sequentially consistent, so there is no relaxed-ordering footgun

## Coming From Another Language

| Other | Go |
|---|---|
| JS: one thread, so visibility is never a question | real threads; a write needs a happens-before edge to be seen |
| Python: the GIL makes most races benign | no GIL; a torn or stale read is real |
| Rust: `Send`/`Sync` are compiler-enforced | nothing is enforced; the race detector is your only check |
| C++ `memory_order_relaxed` | no such thing — Go atomics are always sequentially consistent |
| Java `volatile` | `sync/atomic` types, or just use a channel |

The memory model answers one question: **when is a write by one goroutine guaranteed to be
visible to a read by another?** The answer is never by default. Without a synchronizing
operation between them the compiler may keep a value in a register, reorder independent
writes, and the CPU may make writes visible to other cores out of order. This lesson is the
short list of operations that create that guarantee, and the single rule that follows from
it.

## The problem: a flag-plus-data publication

The classic shape — write the data, then set a flag; spin on the flag, then read the data.

```go
package main

import "fmt"

type config struct {
  host string
  port int
}

var (
  shared *config
  ready  bool
)

func publish() {
  shared = &config{host: "db.internal", port: 5432}
  ready = true
}

func main() {
  done := make(chan struct{})
  go func() {
    defer close(done)
    for !ready {
    }
    fmt.Printf("consumer saw %s:%d\n", shared.host, shared.port)
  }()

  publish()
  <-done
}
```

Output:

```
consumer saw db.internal:5432
```

It printed the right answer, which is the trap. On this machine (darwin/arm64) the loop
happens to observe the write, and a broken program that works is far worse than one that
crashes. So rather than pretending to demonstrate a visible reordering, here is the race
detector (11.5) proving it is a genuine race:

```bash
go run -race .
```

Output (the first two of four reported races; `Found 4 data race(s)`, `exit status 66`):

```
==================
WARNING: DATA RACE
Read at 0x0001023923c8 by goroutine 7:
  main.main.func1()
      /tmp/scratch/main.go:24 +0x84

Previous write at 0x0001023923c8 by main goroutine:
  main.publish()
      /tmp/scratch/main.go:17 +0x134
  main.main()
      /tmp/scratch/main.go:29 +0x94

Goroutine 7 (running) created at:
  main.main()
      /tmp/scratch/main.go:22 +0x90
==================
==================

==================
```

Two distinct races: one on `ready` (read at line 24, written at line 17) and one on
`shared` (read at line 26, written at line 16). The detector does not care that the output
looked correct — it reports that two goroutines touched the same address with no
happens-before edge between them, which is the definition of a data race and means the
program has **no defined behaviour at all**.

## Why the spin loop is broken

`for !ready {}` looks like it must eventually see `ready = true`. Three things can stop it:

1. **The compiler may hoist the load.** `ready` is not written in the loop, so a compiler
   may legally read it once into a register and spin on the register forever.
2. **The CPU may reorder the two writes.** `publish` writes `shared` then `ready`; nothing
   forces that order to be *observed* in that order, so a consumer can see `ready == true`
   while `shared` is still nil — and then panic on `shared.host`.
3. **Writes may not propagate promptly.** Without a barrier there is no guarantee about
   *when* another core sees the store.

Point 2 is the dangerous one: the failure is a nil dereference in the consumer, nowhere
near the code that caused it.

## The two fixes

A channel send/receive, or an atomic. Both create the missing happens-before edge, and both
carry the data rather than a flag pointing at it.

```go
package main

import (
  "fmt"
  "sync/atomic"
)

type config struct {
  host string
  port int
}

func channelFix() {
  ch := make(chan *config)
  done := make(chan struct{})
  go func() {
    defer close(done)
    c := <-ch
    fmt.Printf("channel:  %s:%d\n", c.host, c.port)
  }()
  ch <- &config{host: "db.internal", port: 5432}
  <-done
}

func atomicFix() {
  var shared atomic.Pointer[config]
  done := make(chan struct{})
  go func() {
    defer close(done)
    for {
      if c := shared.Load(); c != nil {
        fmt.Printf("atomic:   %s:%d\n", c.host, c.port)
        return
      }
    }
  }()
  shared.Store(&config{host: "db.internal", port: 5432})
  <-done
}

func main() {
  channelFix()
  atomicFix()
}
```

Output:

```
channel:  db.internal:5432
atomic:   db.internal:5432
```

`go run -race .` is clean. The channel version is the one to write: the value travels
*through* the synchronizing operation, so there is no separate flag to get wrong.
`atomic.Pointer[T]` is the right tool when a value is published once and read on a hot path
by many goroutines — `Load` returning non-nil guarantees everything written before the
`Store` is visible.

## The six happens-before rules

These are the ones that come up in real code. Each demo below is race-clean under
`-race` and prints a deterministic result.

```go
package main

import (
  "fmt"
  "sync"
)

func sendBeforeReceive() {
  var data []int
  ch := make(chan struct{})
  go func() {
    data = append(data, 1, 2, 3)
    ch <- struct{}{}
  }()
  <-ch
  fmt.Println("1. send happens-before receive completes:", data)
}

func closeBeforeZero() {
  var data string
  ch := make(chan struct{})
  go func() {
    data = "written before close"
    close(ch)
  }()
  <-ch
  fmt.Println("2. close happens-before a zero-value receive:", data)
}

func unlockBeforeLock() {
  var mu sync.Mutex
  var n int
  mu.Lock()
  go func() {
    n = 42
    mu.Unlock()
  }()
  mu.Lock()
  fmt.Println("3. Unlock happens-before the next Lock:", n)
  mu.Unlock()
}

func onceBeforeReturn() {
  var once sync.Once
  var cfg string
  var wg sync.WaitGroup
  results := make([]string, 4)
  for i := 0; i < 4; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      once.Do(func() { cfg = "initialized" })
      results[i] = cfg
    }()
  }
  wg.Wait()
  fmt.Println("4. once.Do completion happens-before any Do returning:", results)
}

func createBeforeRun() {
  msg := "set before go"
  done := make(chan string)
  go func() {
    done <- msg
  }()
  fmt.Println("5. goroutine creation happens-before it runs:", <-done)
}

func exitGuaranteesNothing() {
  var n int
  done := make(chan struct{})
  go func() {
    n = 7
    close(done)
  }()
  <-done
  fmt.Println("6. a goroutine's exit alone guarantees nothing; close(done) does:", n)
}

func main() {
  sendBeforeReceive()
  closeBeforeZero()
  unlockBeforeLock()
  onceBeforeReturn()
  createBeforeRun()
  exitGuaranteesNothing()
}
```

Output:

```
1. send happens-before receive completes: [1 2 3]
2. close happens-before a zero-value receive: written before close
3. Unlock happens-before the next Lock: 42
4. once.Do completion happens-before any Do returning: [initialized initialized initialized initialized]
5. goroutine creation happens-before it runs: set before go
6. a goroutine's exit alone guarantees nothing; close(done) does: 7
```

| Rule | Guarantee |
|---|---|
| **channel send → receive** | a send on a channel happens-before the corresponding receive *completes* |
| **close → zero receive** | `close(ch)` happens-before a receive that returns the zero value |
| **Unlock → Lock** | `mu.Unlock()` happens-before any subsequent `mu.Lock()` returning |
| **once.Do** | the completion of `f` in `once.Do(f)` happens-before any `Do(f)` call returns |
| **go statement** | the `go` statement happens-before the goroutine's first instruction |
| **goroutine exit** | **no guarantee.** A goroutine finishing happens-before *nothing* |

Rule 6 is the one people get wrong. A goroutine's exit synchronizes with nothing, so you
cannot "wait" for it by sleeping, by checking a flag it set, or by reasoning that it must be
done by now. You need an explicit edge: a `close(done)`, a `wg.Wait()`, or a receive from a
channel it sent on. That is why `exitGuaranteesNothing` above is correct — not because the
goroutine returned, but because of the `close(done)`.

Two corollaries worth stating. On an **unbuffered** channel the receive also happens-before
the send *completes*, which is why an unbuffered send is a rendezvous. On a **buffered**
channel of capacity C, the receive of the kth value happens-before the send of the
(k+C)th value completes — the mechanism behind backpressure in 12.3.

`sync.WaitGroup`, `errgroup`, and `context` cancellation all build on these same edges;
`wg.Done()` happens-before the `wg.Wait()` it releases.

## Atomics are sequentially consistent

Go has no `memory_order_relaxed`, no acquire/release variants, and no fences: every
operation in `sync/atomic` behaves as a sequentially consistent operation, as the memory
model document states explicitly. That removes the entire class of C++/Rust bugs where a
relaxed load silently buys you no ordering — the cost is that Go's atomics can be slightly
slower than a hand-tuned relaxed atomic, which is almost never the bottleneck.

What atomics do **not** give you is multi-word atomicity. `atomic.Int64` protects one
64-bit value; two related atomics can still be read in an inconsistent combination. If an
invariant spans two fields, you need a mutex or a single `atomic.Pointer` to an immutable
struct.

## The conclusion

The practical takeaway is short, and deliberately boring:

1. **Use channels or mutexes.** They are fast enough, they carry the data with the
   synchronization, and they are what the next reader expects.
2. **Use `sync/atomic` only for a single word** — a counter, a flag, one pointer to an
   immutable value. The moment two atomics must agree, switch to a mutex.
3. **Never invent your own lock-free data structure.** Correct lock-free code requires
   reasoning the race detector cannot verify and reviewers cannot check. The stdlib's
   `sync.Map` and `sync.Pool` exist so you do not have to.
4. **Run `-race` in CI.** It is the only tool that finds these bugs, and it only finds
   races on code paths your tests actually execute (11.5).

The authoritative reference is the [Go Memory Model](https://go.dev/ref/mem), which is
short and worth reading once in full. Its own advice closes the matter: "If you must read
the rest of this document to understand the behavior of your program, you are being too
clever. Don't be clever."

## Quick Reference

| Operation | Creates a happens-before edge? |
|---|---|
| channel send / receive | yes |
| `close(ch)` / receive of zero value | yes |
| `mu.Unlock()` / next `mu.Lock()` | yes |
| `once.Do(f)` completion / any `Do` return | yes |
| `go f()` statement / `f`'s first instruction | yes |
| `wg.Done()` / `wg.Wait()` returning | yes |
| `atomic.Store` / `atomic.Load` seeing it | yes |
| a goroutine returning | **no** |
| `time.Sleep` | **no** |
| a plain `bool` flag | **no** |

| Need | Use |
|---|---|
| hand a value to another goroutine | a channel |
| guard a multi-field invariant | `sync.Mutex` |
| a counter across goroutines | `atomic.Int64` |
| publish one immutable value, read often | `atomic.Pointer[T]` |
| one-time initialization | `sync.Once` |
| anything lock-free and clever | don't |

## Common Mistakes

**Using a plain `bool` as a ready flag.** It creates no happens-before edge, so the reader
may spin forever or see the flag before the data. Use a channel or an atomic.

**Assuming a goroutine's return publishes its writes.** It publishes nothing. Synchronize
with `close(done)`, `wg.Wait()`, or a channel receive.

**`time.Sleep` as synchronization.** It orders nothing and only makes the race rarer, which
means it will surface in production instead of in tests.

**Reading two atomics and assuming a consistent snapshot.** Each is individually atomic;
together they are not. One mutex, or one `atomic.Pointer` to an immutable struct.

**Concluding there is no race because the output looked right.** As the first example
shows, a racy program frequently prints the correct answer. `-race` is the arbiter.

## Exercises

1. Run the flag-plus-data example 1000 times without `-race` and count how often it prints
   the wrong result or panics on a nil `shared`. Then add `GOMAXPROCS=1` and re-run.
   Explain the difference.
2. Rewrite the publication with `sync.RWMutex` and compare its `-race` output and timing
   against the `atomic.Pointer` version under 100 concurrent readers.
3. Build a struct with two `atomic.Int64` fields that must sum to 100. Write a reader that
   observes a sum that is not 100, then fix it with a single `atomic.Pointer` to an
   immutable struct.
4. Take rule 6 and write a program that *looks* like it waits for a goroutine (a sleep long
   enough to "obviously" suffice) and show `-race` reporting it anyway.
5. Read the `sync.Once` implementation in the standard library. Identify the atomic load and
   the mutex, and explain which of this lesson's rules makes the fast path safe.

---

[← 12.5 Deadlocks, Leaks, and How to Spot Them](05-deadlocks-leaks.md) · [Course Index](../README.md) · [13.1 os: Files, Args, and Environment →](../13-stdlib/01-os.md)
