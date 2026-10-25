[← 3.4 goto, Labels, break, and continue](04-labels-goto.md) · [Course Index](../README.md) · [3.6 panic and recover →](06-panic-recover.md)

# 3.5 — defer: Order, Arguments, and Gotchas

## What You'll Learn

- Deferred calls run LIFO when the **function** returns — not when the block ends
- Arguments are evaluated at `defer` time, not at call time; the closure form defers evaluation too
- A `defer` in a loop accumulates until the function exits, which is how file handles leak
- A deferred closure can modify a **named** result, which is the standard error-wrapping idiom

## Coming From Another Language

| Other | Go |
|---|---|
| JS `try { } finally { }` | `defer` — but scoped to the function, not a block |
| Python `with open(f) as h:` | `h, _ := os.Open(f); defer h.Close()` |
| Python `try/finally` | `defer` |
| Rust `Drop` on scope exit | `defer` on function exit; no automatic destructors |
| C++ RAII / scope guards | `defer` is the explicit, manual version |
| JS `finally` runs on throw | `defer` runs on `panic` too |

## LIFO order

```go
package main

import "fmt"

func main() {
  defer fmt.Println("first deferred, runs last")
  defer fmt.Println("second deferred")
  defer fmt.Println("third deferred, runs first")
  fmt.Println("body")
}
```

Output:

```
body
third deferred, runs first
second deferred
first deferred, runs last
```

A stack, not a queue. This is exactly what you want when the deferred calls are
releases: the last thing acquired is the first thing freed.

## Arguments are evaluated at defer time

This is the number one `defer` surprise. The function value and all its arguments
are evaluated when the `defer` statement executes; only the *call* is postponed.

```go
package main

import "fmt"

func main() {
  i := 0
  defer fmt.Println("captured at defer time:", i)
  defer func() { fmt.Println("captured at call time:", i) }()
  i = 42
  fmt.Println("body set i to", i)
}
```

Output:

```
body set i to 42
captured at call time: 42
captured at defer time: 0
```

The first `defer` froze `0`. The second deferred a closure that takes no
arguments and reads `i` when it finally runs, so it sees `42`. Idiomatic: wrap in
`func() { ... }()` whenever the value must be current at exit.

The same split in a loop:

```go
package main

import "fmt"

func main() {
  for i := range 3 {
    defer fmt.Println("eager arg:", i)
  }
  for i := range 3 {
    defer func() { fmt.Println("closure:", i) }()
  }
}
```

Output:

```
closure: 2
closure: 1
closure: 0
eager arg: 2
eager arg: 1
eager arg: 0
```

Both print 2, 1, 0 here — the eager form because each `defer` captured that
iteration's value, and the closure form because loop variables are per-iteration
since Go 1.22 (see [3.2](02-for-loops.md)). Under `go 1.21` or lower the closure
version would print `3 3 3`.

The timing version of the same trap:

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  start := time.Now()
  defer fmt.Println("evaluated at defer time, always ~0:", time.Since(start).Round(time.Millisecond))
  defer func() {
    fmt.Println("evaluated at call time:", time.Since(start).Round(10*time.Millisecond))
  }()
  time.Sleep(50 * time.Millisecond)
}
```

Output:

```
evaluated at call time: 50ms
evaluated at defer time, always ~0: 0s
```

`go vet` catches this specific case:

```
main.go:10:59: call to time.Since is not deferred
```

Idiomatic: `defer func() { log.Printf("took %v", time.Since(start)) }()`.

## defer runs on panic

The deferred calls of every frame being unwound run before the program dies.

```go
package main

import "fmt"

func risky() {
  defer fmt.Println("defer ran despite the panic")
  panic("boom")
}

func main() {
  risky()
}
```

This program panics on purpose. Output:

```
defer ran despite the panic
panic: boom

goroutine 1 [running]:
main.risky()
	/tmp/scratch/main.go:7 +0x60
main.main()
	/tmp/scratch/main.go:11 +0x1c
exit status 2
```

Runtime panics — nil dereference, index out of range — are ordinary panics and
also run defers.

## defer does not run on os.Exit

```go
package main

import (
  "fmt"
  "os"
)

func main() {
  defer fmt.Println("never printed")
  fmt.Println("about to exit")
  os.Exit(3)
}
```

Output:

```
about to exit
```

The process exits with status 3 and the `defer` is skipped. This is why
`log.Fatal` (which calls `os.Exit(1)`) is a poor choice anywhere below `main` —
it silently discards every pending cleanup in the whole program.

Fatal runtime errors are not panics and also skip defers:

```go
package main

import "fmt"

func main() {
  defer fmt.Println("this defer never runs")
  fmt.Println("blocking forever")
  ch := make(chan int)
  <-ch
}
```

This program fails on purpose. Output:

```
blocking forever
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
	/tmp/scratch/main.go:9 +0x9c
exit status 2
```

Note `fatal error:`, not `panic:` — these cannot be recovered either.

## defer is function-scoped, so loops accumulate

A `defer` inside a `for` does not run at the end of the iteration. It stacks up
until the function returns. With file handles that is a leak.

```go
package main

import (
  "fmt"
  "os"
  "path/filepath"
)

func leaky(dir string, names []string) error {
  for _, n := range names {
    f, err := os.Open(filepath.Join(dir, n))
    if err != nil {
      return err
    }
    defer f.Close()
    fmt.Println("opened", n)
  }
  fmt.Println("all files still open here")
  return nil
}

func main() {
  dir, _ := os.MkdirTemp("", "deferdemo")
  defer os.RemoveAll(dir)
  names := []string{"a.txt", "b.txt", "c.txt"}
  for _, n := range names {
    os.WriteFile(filepath.Join(dir, n), []byte(n), 0o644)
  }
  leaky(dir, names)
}
```

Output:

```
opened a.txt
opened b.txt
opened c.txt
all files still open here
```

Three files is harmless; three hundred thousand hits the per-process file
descriptor limit and every `os.Open` starts returning `too many open files`.

Two fixes. Extract the body into a named function taking one path, so each
`defer f.Close()` fires on that function's return. Or wrap the body in an
immediately-invoked `func() { ... }()`. Idiomatic: the named function — the
closure form cannot return an error to the outer loop without extra plumbing.

## defer modifying a named result

A deferred closure runs *after* the return value has been set but *before* the
caller sees it, so it can change a named result. This is how error wrapping and
panic-to-error conversion are written.

```go
package main

import (
  "errors"
  "fmt"
)

func work(n int) (err error) {
  defer func() {
    if err != nil {
      err = fmt.Errorf("work(%d): %w", n, err)
    }
  }()
  if n < 0 {
    return errors.New("negative input")
  }
  return nil
}

func main() {
  fmt.Println(work(1))
  fmt.Println(work(-1))
}
```

Output:

```
<nil>
work(-1): negative input
```

Change the signature to `func work(n int) error` with a local `var err error` and
the wrapping silently stops working: `return err` copies into the anonymous
result slot, and the defer then mutates a local the caller cannot see. The result
**must** be named.

The mechanism works for any result type: `func f() (n int) { defer func(){ n *= 2 }(); return 5 }`
returns 10, while the same body with an unnamed `int` result returns 5.

## Method calls: the receiver is captured too

The receiver is just another argument, so a value receiver is copied at `defer`
time. A pointer receiver captures the pointer, and the pointee is read at call
time.

```go
package main

import "fmt"

type counter struct{ n int }

func (c counter) showVal()  { fmt.Println("value receiver:", c.n) }
func (c *counter) showPtr() { fmt.Println("pointer receiver:", c.n) }

func main() {
  c := counter{n: 1}
  defer c.showVal()
  defer c.showPtr()
  c.n = 99
  fmt.Println("body set n to", c.n)
}
```

Output:

```
body set n to 99
pointer receiver: 99
value receiver: 1
```

`defer c.showVal()` snapshotted the whole struct. `defer c.showPtr()` is
`defer (&c).showPtr()`, so it sees the update.

## Deferring a nil function panics at call time

The `defer` statement itself succeeds; the nil call blows up on the way out.

```go
package main

import "fmt"

func main() {
  var f func()
  defer f()
  fmt.Println("body ran fine")
}
```

This program panics on purpose. Output:

```
body ran fine
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=0x104d0f598]

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:9 +0x68
exit status 2
```

The `pc` address varies per build. The practical consequence: a `defer cleanup()`
where `cleanup` is an optional callback needs a nil check, or `defer func() { if
cleanup != nil { cleanup() } }()`.

## defer order with recover

Defers registered *after* the one holding the `recover` run first, because LIFO.
Defers registered *before* it run after the panic has been neutralised.

```go
package main

import "fmt"

func main() {
  defer fmt.Println("outermost defer, runs last")
  defer func() {
    if r := recover(); r != nil {
      fmt.Println("recovered:", r)
    }
  }()
  defer fmt.Println("defer registered after the recover, runs before it")
  panic("kaboom")
}
```

Output:

```
defer registered after the recover, runs before it
recovered: kaboom
outermost defer, runs last
```

Register the recovering defer first (so it is outermost) when you want it to be
the last line of defence. Details in [3.6 panic and recover](06-panic-recover.md).

## The two idioms you will write constantly

```go
package main

import (
  "fmt"
  "sync"
)

type store struct {
  mu sync.Mutex
  m  map[string]int
}

func (s *store) set(k string, v int) {
  s.mu.Lock()
  defer s.mu.Unlock()
  s.m[k] = v
}

func main() {
  s := &store{m: map[string]int{}}
  var wg sync.WaitGroup
  for i := range 4 {
    wg.Add(1)
    go func() {
      defer wg.Done()
      s.set(fmt.Sprint("k", i), i)
    }()
  }
  wg.Wait()
  fmt.Println(len(s.m))
}
```

Output:

```
4
```

`defer mu.Unlock()` immediately after `Lock()`, and `defer wg.Done()` as the
first line of the goroutine, so neither can be skipped by an early return or a
panic. The one caveat: `defer Unlock` holds the lock for the whole function, so
if the function does slow work after the critical section, unlock explicitly
instead.

For `Close`, note that discarding the error is only safe on a read. On a write,
`Close` can report a failed flush, so capture it with a named result:

```go
func write(path string, b []byte) (err error) {
  f, err := os.Create(path)
  if err != nil {
    return err
  }
  defer func() {
    if cerr := f.Close(); cerr != nil && err == nil {
      err = cerr
    }
  }()
  _, err = f.Write(b)
  return err
}
```

## Cost

Since Go 1.14 most `defer`s are open-coded by the compiler and cost roughly a
direct call; a `defer` in a loop or behind a conditional falls back to the
heap-allocated path and is slower. Do not avoid `defer` on performance grounds
until a profile says so.

## Quick Reference

| Behaviour | Detail |
|---|---|
| when it runs | when the enclosing **function** returns, not at end of block |
| order | LIFO |
| argument evaluation | at `defer` time |
| receiver evaluation | at `defer` time; value receivers are copied |
| runs on `return` | yes, after the result is set |
| runs on `panic` | yes, during unwinding |
| runs on runtime panic | yes |
| runs on `os.Exit` | **no** |
| runs on `log.Fatal` | **no** (it calls `os.Exit`) |
| runs on fatal runtime error (deadlock, OOM) | **no** |
| runs on `runtime.Goexit` | yes |
| can modify a result | only a **named** result |
| deferring a nil func | panics when the defer fires |

| Idiom | Shape |
|---|---|
| mutex | `mu.Lock(); defer mu.Unlock()` |
| waitgroup | `defer wg.Done()` as the goroutine's first line |
| file read | `defer f.Close()` |
| file write | `defer func(){ if c := f.Close(); c != nil && err == nil { err = c } }()` |
| wrap the error | `func f() (err error) { defer func(){ if err != nil { err = fmt.Errorf("f: %w", err) } }() }` |
| current value at exit | `defer func() { use(x) }()` |
| per-iteration cleanup | extract the loop body into a function |
| timing | `defer func() { log.Print(time.Since(start)) }()` |

## Common Mistakes

**Expecting `defer` to run at the end of a loop iteration or `if` block.** It is
function-scoped; extract the body into its own function.

**Writing `defer fmt.Println(x)` and expecting the final value of `x`.** The
argument is frozen at `defer` time; wrap it in a closure.

**Writing `defer log.Printf("took %v", time.Since(start))`.** `time.Since` runs
immediately and logs zero; `go vet` reports `call to time.Since is not deferred`.

**Mutating `err` in a deferred closure when the result is unnamed.** The caller
never sees it; name the result `(err error)`.

**Relying on `defer` after `os.Exit` or `log.Fatal`.** Neither unwinds; return an
error up to `main` and exit there.

## Exercises

1. Write a function with four defers that print 1, 2, 3, 4 in the body order and
   predict the output before running it. Then make it print 1, 2, 3, 4 at exit.
2. Open 2000 files in a loop with `defer f.Close()` inside the loop and see what
   error `os.Open` eventually returns. Lower the limit with `ulimit -n 64` first
   to hit it faster. Fix it and confirm the error disappears.
3. Write `func readAll(path string) (data []byte, err error)` that wraps any
   error with the path using a deferred closure, and prove with `errors.Is` that
   the original `fs.ErrNotExist` still unwraps.
4. Write a `func withTx(db *sql.DB, fn func(*sql.Tx) error) (err error)` that
   commits on success and rolls back on both error and panic, using exactly one
   `defer`. (No real database needed — stub the types.)
5. Benchmark a function with a `defer mu.Unlock()` against one that unlocks
   explicitly, at 1 and at 100 iterations of a loop containing the defer. Explain
   the difference in terms of open-coded versus heap-allocated defers.

---

[← 3.4 goto, Labels, break, and continue](04-labels-goto.md) · [Course Index](../README.md) · [3.6 panic and recover →](06-panic-recover.md)
