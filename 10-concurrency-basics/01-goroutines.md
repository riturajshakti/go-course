[← 9.5 When Generics Are the Wrong Tool](../09-generics/05-when-not-to.md) · [Course Index](../README.md) · [10.2 Unbuffered Channels →](02-unbuffered-channels.md)

# 10.1 — Goroutines and the Scheduler

## What You'll Learn

- `go f()` starts a goroutine; it returns nothing, so you have no handle, no ID, and no way to cancel it
- Goroutines cost about 2 KB of stack, so 100,000 of them start in under a tenth of a second
- When `main` returns the process exits immediately and every other goroutine is killed mid-flight
- An unrecovered panic in any goroutine takes down the whole process

## Coming From Another Language

| Other | Go |
|---|---|
| JS `const p = f()` — a promise you can `await`, chain, or race | `go f()` — a statement, not an expression; it evaluates to nothing |
| JS/Python `async def` colours every caller | no `async` keyword; any function can run in a goroutine |
| Python `t = asyncio.create_task(c)`; `t.cancel()` | no task object and no `cancel()`; cancellation is something you build |
| Rust `tokio::spawn(fut)` returns a `JoinHandle` | nothing is returned; you wait with a channel or `sync.WaitGroup` |
| Blocking the event loop stalls everything | a blocking syscall parks one goroutine; the rest keep running |
| `threading.Thread` — ~8 MB stack, hundreds max | ~2 KB stack, hundreds of thousands is routine |

Lead with the thing that trips everyone up: a goroutine is **not** a future. `go f()`
is a statement. It produces no value, so there is nothing to `await`, nothing to
`.then()`, nothing to cancel, and no result to collect. Every mechanism you are used to
getting for free — waiting, returning a value, propagating an error, cancelling — you
build yourself out of channels (this module) or `sync` primitives (module 11).

The flip side is that there is no function colouring and no event loop you can starve.
A goroutine that calls a blocking syscall blocks only itself; the runtime moves the other
goroutines to a different OS thread. There is no `async` version of `os.ReadFile`, because
the plain one is already safe to call from concurrent code.

## go f()

```go
package main

import "fmt"

func greet(name string) {
  fmt.Println("hello,", name)
}

func main() {
  go greet("goroutine")
  fmt.Println("hello, main")
}
```

Output:

```
hello, main
```

The greeting never prints. `go greet(...)` returned instantly, `main` printed its line
and returned, and a returning `main` exits the process without waiting for anything.
The goroutine was real, it was scheduled, and it was killed before it reached `Println`.

The fix needs a channel — there is no other built-in way to wait:

```go
package main

import "fmt"

func greet(name string, done chan<- struct{}) {
  fmt.Println("hello,", name)
  close(done)
}

func main() {
  done := make(chan struct{})
  go greet("goroutine", done)
  fmt.Println("hello, main")
  <-done
}
```

Output:

```
hello, main
hello, goroutine
```

That ordering is now guaranteed: `main` prints before it blocks on `<-done`, and the
goroutine prints before it closes `done`. Channel handoff is what makes concurrent output
deterministic, and most of this module is that idea applied in different shapes.

The arguments to `go f(a, b)` are evaluated **immediately**, in the calling goroutine, at
the `go` statement. Only the call itself is deferred.

## The forms a go statement can take

```go
package main

import "fmt"

type counter struct{ n int }

func (c *counter) bump(out chan<- int) { out <- c.n + 1 }

func main() {
  out := make(chan int, 3)

  go func() { out <- 1 }()

  f := func(v int) { out <- v }
  go f(2)

  c := &counter{n: 2}
  go c.bump(out)

  sum := 0
  for i := 0; i < 3; i++ {
    sum += <-out
  }
  fmt.Println("sum:", sum)
}
```

Output:

```
sum: 6
```

A `go` statement needs a function **call**. `go f` is a compile error, and so is
`go x + 1`. An immediately-invoked closure — `go func(){ ... }()` — is by far the most
common form.

## Goroutines are cheap

Start a hundred thousand of them, wait until every one has reported in, and measure.

```go
package main

import (
  "fmt"
  "runtime"
  "time"
)

func main() {
  const n = 100_000

  var before runtime.MemStats
  runtime.ReadMemStats(&before)

  block := make(chan struct{})
  ready := make(chan struct{}, n)
  start := time.Now()
  for i := 0; i < n; i++ {
    go func() {
      ready <- struct{}{}
      <-block
    }()
  }
  for i := 0; i < n; i++ {
    <-ready
  }
  elapsed := time.Since(start)

  var after runtime.MemStats
  runtime.ReadMemStats(&after)

  fmt.Println("goroutines alive:", runtime.NumGoroutine())
  fmt.Printf("time to start and sync %d: %v\n", n, elapsed.Round(time.Millisecond))
  fmt.Println("stack in use:", after.StackInuse/1024, "KB")

  close(block)
  fmt.Println("done")
}
```

The counts are stable; the wall-clock time varies by a few tens of milliseconds between
runs. Output on this machine (darwin/arm64, 12 cores, Go 1.27.1):

```
goroutines alive: 100001
time to start and sync 100000: 71ms
stack in use: 200544 KB
done
```

200 MB of stack for 100,000 goroutines is almost exactly 2 KB each — the initial
goroutine stack size. It grows on demand (the runtime copies the stack to a bigger block
when it overflows) and shrinks during GC, so idle goroutines stay tiny. Compare 8 MB of
reserved address space for an OS thread: 100,000 pthreads is not something you attempt.

`runtime.NumGoroutine` counts live goroutines including `main`, hence 100001.

## Goroutines vs OS threads vs async tasks

| | OS thread | JS promise / Python task / Rust future | Goroutine |
|---|---|---|---|
| Initial stack | 512 KB – 8 MB reserved | heap frame, ~hundreds of bytes | 2 KB, growable |
| Practical count | hundreds | millions | hundreds of thousands |
| Created by | kernel (`clone`, `pthread_create`) | your runtime, in user space | Go runtime, in user space |
| Scheduled by | kernel, preemptively | cooperatively, only at `await` | Go runtime, M:N, with async preemption |
| Blocking I/O | blocks the thread | blocks the whole loop | parks the goroutine only |
| Colouring | none | `async`/`await` infects callers | none |
| Handle returned | thread ID / handle | task/promise object | nothing |
| Cancellation | signals, `pthread_cancel` | `task.cancel()`, `AbortController` | none built in; use `context` |
| Identity | has an ID | has an object | no accessible ID |

The row that matters most for a day-to-day mental model is "blocking I/O". In Node or
asyncio, a synchronous `read()` is a bug that freezes the process. In Go it is the normal
way to write code — the runtime detects that the goroutine entered a blocking syscall,
detaches the OS thread it was running on, and schedules other goroutines onto a different
thread.

## GOMAXPROCS and NumCPU

`runtime.NumCPU` reports the cores the process can use. `GOMAXPROCS` is how many
goroutines may execute Go code *simultaneously*; it defaults to `NumCPU`.

```go
package main

import (
  "fmt"
  "runtime"
)

func main() {
  fmt.Println("NumCPU:", runtime.NumCPU())
  fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
  runtime.GOMAXPROCS(1)
  fmt.Println("after setting to 1:", runtime.GOMAXPROCS(0))
}
```

Output on this machine (12-core Apple silicon, darwin/arm64):

```
NumCPU: 12
GOMAXPROCS: 12
after setting to 1: 1
```

Calling `runtime.GOMAXPROCS(n)` with `n > 0` sets the value and returns the previous one;
`GOMAXPROCS(0)` just reads it. The environment variable `GOMAXPROCS=1 go run .` does the
same thing without touching your code, which is the preferred way to test whether a bug
is a real race or an artifact of parallelism.

`GOMAXPROCS` does not cap the number of goroutines, and it does not cap OS threads.
Goroutines blocked in syscalls get their own threads beyond this limit. Since Go 1.25 the
default is also container-aware: in a cgroup with a CPU quota, the runtime derives
`GOMAXPROCS` from the quota rather than from the host's core count.

## The scheduler in one paragraph

Go runs an M:N scheduler: M goroutines (G) multiplexed onto N OS threads (M) through a
fixed number of logical processors (P), where N of those Ps equals `GOMAXPROCS`. Each P
owns a local run queue of runnable goroutines; when a P's queue empties it **steals** half
the work from a random other P's queue, which is why load spreads without a central lock.
A goroutine yields at function calls (the compiler inserts a stack-bound check that
doubles as a preemption point), and — since Go 1.14 — the runtime can also preempt
**asynchronously**, by signalling the thread, so a tight loop with no calls in it no
longer wedges a P forever. Blocking syscalls hand the M off: the P is detached and
rescheduled onto a fresh thread, so a goroutine sitting in `read()` costs you a thread,
not a core.

That is the whole model you need. You do not tune it, and `runtime.Gosched()` is almost
never the answer.

## Interleaving really is nondeterministic

When two goroutines print without coordinating, the order is whatever the scheduler
chose. This is the one place in the module where the output genuinely differs run to run,
so here it is, three times.

```go
package main

import "fmt"

func main() {
  done := make(chan struct{})
  for i := 1; i <= 5; i++ {
    go func() {
      fmt.Print(i, " ")
      done <- struct{}{}
    }()
  }
  for k := 0; k < 5; k++ {
    <-done
  }
  fmt.Println()
}
```

Three consecutive runs:

```
5 2 1 3 4
```

```
2 3 4 1 5
```

```
5 3 4 2 1
```

Never write a test or a lesson that depends on this order. If you need an order, create
it with a channel handoff, or collect the results and sort them — which is what the next
example does.

## Loop variables: Go 1.22 changed this

Before Go 1.22 the loop variable of a `for` statement was **one** variable reused across
iterations, so every closure captured the same `i` and usually saw its final value. Since
Go 1.22 each iteration gets a fresh variable. Results are collected and sorted so the
output is deterministic.

```go
package main

import (
  "fmt"
  "sort"
)

func main() {
  got := make(chan int, 3)
  for i := 1; i <= 3; i++ {
    go func() {
      got <- i
    }()
  }
  var out []int
  for k := 0; k < 3; k++ {
    out = append(out, <-got)
  }
  sort.Ints(out)
  fmt.Println(out)
}
```

With `go 1.27.1` in `go.mod`. Output:

```
[1 2 3]
```

The semantics are selected by the `go` line in `go.mod`, not by the compiler version. Put
the identical file in a module whose `go.mod` says `go 1.21` and the same Go 1.27.1
toolchain compiles it with the old rules:

```
module example.com/scratch

go 1.21
```

Output:

```
[4 4 4]
```

Three goroutines, all reading the one shared `i`, all of them scheduled after the loop
had already incremented it past the bound to 4. This is the single most notorious Go bug
of the pre-1.22 era, and this is why you will still see the old workaround in older code:

```go
package main

import (
  "fmt"
  "sort"
)

func main() {
  got := make(chan int, 3)
  for i := 1; i <= 3; i++ {
    go func(i int) {
      got <- i
    }(i)
  }
  var out []int
  for k := 0; k < 3; k++ {
    out = append(out, <-got)
  }
  sort.Ints(out)
  fmt.Println(out)
}
```

Passing `i` as a parameter copies it at the `go` statement. Under `go 1.21` this also
prints `[1 2 3]`. Idiomatic today: rely on per-iteration scoping and do not pass the
variable in. Keep the parameter form only in code that must build with an old `go`
directive.

## A panic in any goroutine kills the process

This is the biggest behavioural difference from a rejected promise. An unhandled
rejection in Node logs a warning. An uncaught panic in a goroutine terminates the program.

```go
package main

import "fmt"

func main() {
  defer fmt.Println("main deferred: does not run")
  done := make(chan struct{})
  go func() {
    panic("boom in goroutine")
  }()
  <-done
}
```

This program panics on purpose. Output:

```
panic: boom in goroutine

goroutine 19 [running]:
main.main.func1()
	/tmp/scratch/main.go:9 +0x2c
created by main.main in goroutine 1
	/tmp/scratch/main.go:8 +0x74
exit status 2
```

Note two things. `main`'s deferred `Println` did not run — the runtime tears the process
down without unwinding other goroutines' stacks. And `recover()` in `main` cannot help:
`recover` only sees panics on its own goroutine's stack. There is no cross-goroutine
`try/catch`.

The only fix is a `defer`/`recover` **inside** the goroutine:

```go
package main

import "fmt"

func safely(fn func()) {
  defer func() {
    if r := recover(); r != nil {
      fmt.Println("goroutine recovered:", r)
    }
  }()
  fn()
}

func main() {
  done := make(chan struct{})
  go func() {
    defer close(done)
    safely(func() { panic("boom in goroutine") })
  }()
  <-done
  fmt.Println("main survived")
}
```

Output:

```
goroutine recovered: boom in goroutine
main survived
```

Idiomatic: any goroutine you start that runs arbitrary or request-scoped code — a worker
pool task, an HTTP handler you spun off, a plugin callback — gets its own recover at the
top. `net/http` already does this per request, which is why one bad handler does not kill
your server.

## Counting goroutines

`runtime.NumGoroutine` is the cheapest leak check you have. The polling loop below is
deliberately crude; `runtime.Gosched()` yields the current P so the finishing goroutines
get scheduled.

```go
package main

import (
  "fmt"
  "runtime"
)

func main() {
  fmt.Println("at start:", runtime.NumGoroutine())

  block := make(chan struct{})
  ready := make(chan struct{})
  for i := 0; i < 5; i++ {
    go func() {
      ready <- struct{}{}
      <-block
    }()
  }
  for i := 0; i < 5; i++ {
    <-ready
  }
  fmt.Println("after 5 go statements:", runtime.NumGoroutine())

  close(block)
  for runtime.NumGoroutine() > 1 {
    runtime.Gosched()
  }
  fmt.Println("after they finish:", runtime.NumGoroutine())
}
```

Output:

```
at start: 1
after 5 go statements: 6
after they finish: 1
```

A goroutine count that climbs monotonically in a long-running server is a **goroutine
leak**: goroutines blocked forever on a channel nobody will ever touch. They are never
collected, because a blocked goroutine is still reachable. Covered properly in
[12.5 Deadlocks, Leaks, and How to Spot Them](../12-concurrency-patterns/05-deadlocks-leaks.md).

## No ID, no handle, no kill

A `pthread` has a thread ID. A `tokio` task has a `JoinHandle` with `.abort()`. An
asyncio task has `.cancel()`. A goroutine has none of this, by design:

- There is no exported `runtime.GoroutineID()`. The ID in a panic trace is for humans.
- `go f()` returns nothing, so there is nothing to store, pass around, or join on.
- There is no `kill`. You cannot stop a goroutine from outside. It stops when its
  function returns, and the only way to influence that is to tell it to return — a
  `done` channel ([10.4](04-closing-range.md)) or a `context.Context`
  ([11.6 context: Cancellation and Timeouts](../11-sync-context/06-context-cancel.md)).

The absence of goroutine-local storage is deliberate too: it forces you to pass state
explicitly, which is why `context.Context` is the first parameter of so many Go functions.

Waiting for one goroutine takes a channel, as shown above. Waiting for *N* goroutines
with a channel means counting receives, which is tedious and easy to get wrong — that is
exactly the job of [11.1 sync.WaitGroup](../11-sync-context/01-waitgroup.md).

Every example in this module is race-free and passes `go run -race .`; deliberate data
races and the race detector itself are
[11.5 The Race Detector](../11-sync-context/05-race-detector.md).

## Quick Reference

| Syntax / call | Meaning |
|---|---|
| `go f(a, b)` | run `f` concurrently; `a`, `b` evaluated now, in this goroutine |
| `go func() { ... }()` | the common form; closes over surrounding variables |
| `go f` | compile error — a `go` statement requires a call |
| `runtime.NumGoroutine()` | live goroutines, including the caller |
| `runtime.NumCPU()` | usable CPU cores |
| `runtime.GOMAXPROCS(0)` | read the current limit on parallel Go execution |
| `runtime.GOMAXPROCS(n)` | set it to `n`, return the previous value |
| `runtime.Gosched()` | yield the processor; rarely needed |
| `GOMAXPROCS=1 go run .` | force single-threaded execution, no code change |
| `go run -race .` | build with the race detector |

| Property | Goroutine |
|---|---|
| Initial stack | 2 KB, grows and shrinks automatically |
| Return value of `go` | none |
| Accessible ID | none |
| Cancellation | none built in; use a channel or `context` |
| Unrecovered panic | terminates the whole process |
| `main` returns | all other goroutines die immediately |

## Common Mistakes

**Treating `go f()` like `await f()` or `tokio::spawn`.** It returns nothing — to get a
result or wait for completion you need a channel or a `sync.WaitGroup`.

**Letting `main` return while goroutines are still working.** The process exits and they
are killed silently; block on a channel or `WaitGroup` before returning.

**Putting `recover()` in `main` to catch a goroutine's panic.** `recover` only sees its
own goroutine's stack; defer a recover inside each goroutine you start.

**Assuming print order from concurrent goroutines.** It is scheduler-dependent; sequence
the work with channels or collect and sort before printing.

**Raising `GOMAXPROCS` to run more goroutines.** It limits parallel execution, not the
goroutine count; you can run 100,000 goroutines with `GOMAXPROCS=1`.

**Spawning a goroutine per item with no exit path.** If it blocks on a channel nobody
reads, it leaks for the lifetime of the process.

## Exercises

1. Write a program that starts three goroutines, each printing its index, such that the
   output is `1 2 3` on **every** run. Use only channels — no `time.Sleep`, no `sync`.
2. Measure the cost of a goroutine yourself: start `n` goroutines that each block on a
   channel, for `n` in 1e3, 1e4, 1e5, 1e6, and print `StackInuse` from
   `runtime.ReadMemStats` divided by `n`. Explain why the per-goroutine figure is not
   constant.
3. Create a module with `go 1.21` in `go.mod` and one with `go 1.27.1`, put the same
   loop-variable-capturing program in both, and confirm the outputs differ. Then change
   only the `go` line in the 1.21 module and watch the behaviour flip.
4. Write a `func Go(fn func()) <-chan error` that runs `fn` in a goroutine, converts any
   panic into an `error` on the returned channel, and sends `nil` on success. Verify both
   paths.
5. Write a tight loop with no function calls in it (`for { x++ }`) in one goroutine while
   another prints every 100 ms. Run with `GOMAXPROCS=1`. Explain why the printer still
   runs, and what would have happened before Go 1.14.

---

[← 9.5 When Generics Are the Wrong Tool](../09-generics/05-when-not-to.md) · [Course Index](../README.md) · [10.2 Unbuffered Channels →](02-unbuffered-channels.md)
