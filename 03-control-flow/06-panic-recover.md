[← 3.5 defer: Order, Arguments, and Gotchas](05-defer.md) · [Course Index](../README.md) · [4.1 Pointers and the Absence of Pointer Arithmetic →](../04-types-structs/01-pointers.md)

# 3.6 — panic and recover

## What You'll Learn

- `panic(any)` unwinds, runs every `defer` on the way, prints a traceback, and exits with status 2
- `recover()` returns non-nil **only** when called directly from a deferred function in the panicking frame
- A panic in a goroutine cannot be recovered by its parent and kills the whole process
- Panic is for programmer bugs; return an `error` for anything a caller could reasonably handle

## Coming From Another Language

| Other | Go |
|---|---|
| JS `throw` / `catch` | `panic` / `recover` — but not for control flow |
| Python `raise ValueError` for bad input | return an `error`; panic is for bugs |
| Python bare `except:` at a server boundary | `defer func(){ recover() }()` in middleware |
| Rust `panic!` + `catch_unwind` | the closest analogue, same discouragement |
| JS unhandled rejection logs and continues | an unrecovered goroutine panic kills the process |
| Go `error` | the default; `panic` is the exception, not the norm |

## What a panic looks like

```go
package main

import "fmt"

func level3() { panic("something went very wrong") }
func level2() { level3() }
func level1() { level2() }

func main() {
  fmt.Println("before")
  level1()
  fmt.Println("after")
}
```

This program panics on purpose. Output:

```
before
panic: something went very wrong

goroutine 1 [running]:
main.level3(...)
	/tmp/scratch/main.go:5
main.level2(...)
	/tmp/scratch/main.go:6
main.level1(...)
	/tmp/scratch/main.go:7
main.main()
	/tmp/scratch/main.go:11 +0x6c
exit status 2
```

`after` is never printed. The traceback is innermost-first. `(...)` means the
arguments were elided because those frames were inlined; `+0x6c` is a byte offset
into the frame's machine code and differs between builds. Exit status is always 2
for a panic.

## panic takes any value

The signature is `func panic(v any)`. Convention is a `string` or an `error`.

```go
package main

import "fmt"

type myErr struct{ code int }

func (e myErr) Error() string { return fmt.Sprintf("code %d", e.code) }

func try(f func()) {
  defer func() {
    r := recover()
    fmt.Printf("recovered %#v (type %T)\n", r, r)
  }()
  f()
}

func main() {
  try(func() { panic("a string") })
  try(func() { panic(42) })
  try(func() { panic(myErr{404}) })
  try(func() { panic(nil) })
  try(func() { panic([]int{1, 2}) })
}
```

Output:

```
recovered "a string" (type string)
recovered 42 (type int)
recovered main.myErr{code:404} (type main.myErr)
recovered &runtime.PanicNilError{_:[0]*runtime.PanicNilError{}} (type *runtime.PanicNilError)
recovered []int{1, 2} (type []int)
```

Note `panic(nil)`: since Go 1.21 it is converted to a `*runtime.PanicNilError`,
so `recover() != nil` is now a reliable "did we panic" test. Under older versions
`panic(nil)` was recoverable but indistinguishable from no panic at all.

## Runtime panics

Most panics you will see are not written by you. These are the real messages:

```go
package main

import "fmt"

type T struct{ n int }

func catch(name string, f func()) {
  defer func() {
    if r := recover(); r != nil {
      fmt.Printf("%-22s %T | %v\n", name, r, r)
    }
  }()
  f()
}

func main() {
  catch("nil deref", func() {
    var p *T
    _ = p.n
  })
  catch("index out of range", func() {
    xs := []int{1, 2, 3}
    i := 5
    _ = xs[i]
  })
  catch("slice bounds", func() {
    xs := []int{1, 2, 3}
    i := 9
    _ = xs[:i]
  })
  catch("nil map write", func() {
    var m map[string]int
    m["k"] = 1
  })
  catch("type assertion", func() {
    var v any = "str"
    _ = v.(int)
  })
  catch("integer divide by zero", func() {
    a, b := 1, 0
    _ = a / b
  })
  catch("close of closed chan", func() {
    ch := make(chan int)
    close(ch)
    close(ch)
  })
  catch("send on closed chan", func() {
    ch := make(chan int, 1)
    close(ch)
    ch <- 1
  })
  catch("nil func call", func() {
    var f func()
    f()
  })
  catch("explicit panic", func() {
    panic("programmer error")
  })
}
```

Output:

```
nil deref              runtime.errorString | runtime error: invalid memory address or nil pointer dereference
index out of range     runtime.boundsError | runtime error: index out of range [5] with length 3
slice bounds           runtime.boundsError | runtime error: slice bounds out of range [:9] with capacity 3
nil map write          runtime.plainError | assignment to entry in nil map
type assertion         *runtime.TypeAssertionError | interface conversion: interface {} is string, not int
integer divide by zero runtime.errorString | runtime error: integer divide by zero
close of closed chan   runtime.plainError | close of closed channel
send on closed chan    runtime.plainError | send on closed channel
nil func call          runtime.errorString | runtime error: invalid memory address or nil pointer dereference
explicit panic         string | programmer error
```

All of those types satisfy `error`, so `recover().(error)` works for runtime
panics — but not for `panic("a string")`, which is why the usual shape is a type
switch on the recovered value.

One asymmetry worth knowing: an out-of-range *constant* index into an **array**
is caught at compile time, because the length is part of the type.

```
./main.go:7:18: invalid argument: index 5 out of bounds [0:3]
```

The same constant index into a **slice** is not — `len` is a runtime property, so
`xs[5]` on a three-element slice compiles and panics.

| Operation | Panic text |
|---|---|
| `p.field` on nil `*T` | `runtime error: invalid memory address or nil pointer dereference` |
| `xs[i]`, `i >= len(xs)` | `runtime error: index out of range [i] with length n` |
| `xs[:i]`, `i > cap(xs)` | `runtime error: slice bounds out of range [:i] with capacity n` |
| write to nil map | `assignment to entry in nil map` |
| failed `v.(T)` | `interface conversion: interface {} is X, not T` |
| `a / 0` | `runtime error: integer divide by zero` |
| `close(ch)` twice | `close of closed channel` |
| `close(nilCh)` | `close of nil channel` |
| send on closed channel | `send on closed channel` |
| `make([]T, n)`, `n < 0` | `runtime error: makeslice: len out of range` |
| calling a nil func value | `runtime error: invalid memory address or nil pointer dereference` |

Reading a missing key from a nil map returns the zero value — only *writing*
panics.

## recover() only works in a deferred call

Called anywhere else, including from a normal function invoked by the deferred
function, it returns `nil`.

```go
package main

import "fmt"

func notDeferred() {
  r := recover()
  fmt.Println("recover() outside a deferred call returns:", r)
}

func main() {
  defer func() {
    fmt.Println("the real recover:", recover())
  }()
  notDeferred()
  panic("boom")
}
```

Output:

```
recover() outside a deferred call returns: <nil>
the real recover: boom
```

It also only reaches one level. Once a frame recovers, the panic is over; a
`recover` higher up the stack sees nothing.

```go
package main

import "fmt"

func inner() {
  defer func() {
    fmt.Println("inner's defer, recover() =", recover())
  }()
  panic("deep")
}

func main() {
  defer func() {
    fmt.Println("main's defer, recover() =", recover())
  }()
  inner()
  fmt.Println("main continued normally")
}
```

Output:

```
inner's defer, recover() = deep
main continued normally
main's defer, recover() = <nil>
```

After `inner` recovers, execution resumes in `main` right after the call — not at
the `panic`. The recovering function returns normally.

## Re-panicking

Recover, inspect, and panic again if you cannot handle it. The traceback records
both.

```go
package main

import "fmt"

func main() {
  defer func() {
    if r := recover(); r != nil {
      fmt.Println("inspected:", r)
      panic(fmt.Sprintf("re-panicking: %v", r))
    }
  }()
  panic("original")
}
```

This program panics on purpose. Output:

```
inspected: original
panic: original [recovered]
	panic: re-panicking: original

goroutine 1 [running]:
main.main.func1()
	/tmp/scratch/main.go:9 +0xa8
panic({0x1003404b8?, 0x10032c010?})
	/usr/local/go/src/runtime/panic.go:859 +0x120
main.main()
	/tmp/scratch/main.go:12 +0x48
exit status 2
```

The `panic: original [recovered]` line shows the first panic was caught; the
indented line is the new one. The hex values are runtime pointers and vary per
run.

## Converting panic to error at a boundary

This is the one place `recover` is genuinely idiomatic: at the edge of a package
or a request, turning a bug into an error instead of taking down the process.

```go
package main

import (
  "errors"
  "fmt"
)

func parse(s string) (n int, err error) {
  defer func() {
    if r := recover(); r != nil {
      switch v := r.(type) {
      case error:
        err = fmt.Errorf("parse %q: %w", s, v)
      default:
        err = fmt.Errorf("parse %q: %v", s, v)
      }
    }
  }()
  if s == "" {
    panic(errors.New("empty input"))
  }
  if s == "bad" {
    panic("unparseable")
  }
  return len(s), nil
}

func main() {
  for _, s := range []string{"hello", "", "bad"} {
    n, err := parse(s)
    fmt.Printf("%-7q n=%d err=%v\n", s, n, err)
  }
}
```

Output:

```
"hello" n=5 err=<nil>
""      n=0 err=parse "": empty input
"bad"   n=0 err=parse "bad": unparseable
```

The named result `err` is essential — see [3.5](05-defer.md). The `error` case
uses `%w` so `errors.Is` still works through the boundary.

The HTTP version is the same shape wrapped around a handler:

```go
func recoverMW(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    defer func() {
      if rec := recover(); rec != nil {
        http.Error(w, "internal server error", http.StatusInternalServerError)
      }
    }()
    next.ServeHTTP(w, r)
  })
}
```

Driven with `httptest.NewServer(recoverMW(mux))` against an `/ok` and a `/boom`
route, the requests return in order:

```
/ok 200 OK
/boom 500 Internal Server Error
/ok 200 OK
```

The server survives the panicking handler. Middleware properly in
[16.6 Middleware from Scratch](../16-http/06-middleware.md).

`net/http` actually recovers handler panics itself and closes the connection, but
it logs a stack trace and returns nothing useful to the client. Your own
middleware gives you a real 500, structured logging, and the chance to attach a
request ID.

## A panic in a goroutine kills the program

This is the most important rule in this lesson. `recover` is per-goroutine. A
`defer`/`recover` in the parent does nothing for a child.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  defer func() {
    fmt.Println("parent recover:", recover())
  }()
  go func() {
    panic("panic in a goroutine")
  }()
  time.Sleep(100 * time.Millisecond)
  fmt.Println("never printed")
}
```

This program panics on purpose. Output:

```
panic: panic in a goroutine

goroutine 19 [running]:
main.main.func2()
	/tmp/scratch/main.go:13 +0x2c
created by main.main in goroutine 1
	/tmp/scratch/main.go:12 +0x40
exit status 2
```

The goroutine number varies per run. Neither the parent's `defer` nor
`fmt.Println` ran — the whole process is gone. Every long-lived goroutine you
spawn needs its own recover:

```go
package main

import (
  "fmt"
  "sync"
)

func safeGo(wg *sync.WaitGroup, f func()) {
  wg.Add(1)
  go func() {
    defer wg.Done()
    defer func() {
      if r := recover(); r != nil {
        fmt.Println("worker recovered:", r)
      }
    }()
    f()
  }()
}

func main() {
  var wg sync.WaitGroup
  safeGo(&wg, func() { panic("worker 1 died") })
  safeGo(&wg, func() { fmt.Println("worker 2 fine") })
  wg.Wait()
  fmt.Println("main survived")
}
```

Output:

```
worker 2 fine
worker recovered: worker 1 died
main survived
```

The two workers run concurrently, so the first two lines may swap. Note the defer
ordering: `wg.Done` is registered first so it runs *last*, after the recover, and
therefore cannot be skipped by a panic in the recovering closure.

## Capturing the stack

`recover()` gives you the value, not the stack. Grab the stack inside the
recovering defer, while the frames are still live.

```go
package main

import (
  "fmt"
  "runtime/debug"
)

func deep() {
  defer func() {
    if r := recover(); r != nil {
      fmt.Println("recovered:", r)
      fmt.Println("--- debug.Stack() ---")
      fmt.Print(string(debug.Stack()))
    }
  }()
  panic("inspect me")
}

func main() {
  deep()
}
```

Output:

```
recovered: inspect me
--- debug.Stack() ---
goroutine 1 [running]:
runtime/debug.Stack()
	/usr/local/go/src/runtime/debug/stack.go:26 +0x64
main.deep.func1()
	/tmp/scratch/main.go:13 +0x90
panic({0x100bec4d8?, 0x100bd8010?})
	/usr/local/go/src/runtime/panic.go:859 +0x120
main.deep()
	/tmp/scratch/main.go:16 +0x48
main.main()
	/tmp/scratch/main.go:20 +0x1c
```

`debug.Stack()` returns `[]byte` for logging; `debug.PrintStack()` writes it
straight to stderr. The GOROOT paths and hex offsets differ on your machine.

Related: `runtime.Goexit()` terminates the calling goroutine after running its
defers, without panicking and without affecting other goroutines. Called from
`main` it is a fatal error. It is what `t.FailNow`/`t.Fatal` use under the hood;
you will not call it directly.

## When to panic

Panic only when the program is already broken: a precondition no correct caller
could violate, failed initialisation at startup (missing config, bad regexp
literal, unparseable embedded template), or a `default:` branch in a switch over
your own enum that should have been exhaustive.

Return an `error` for everything else — bad user input, a missing file, a network
timeout, a failed parse of data you did not author. A library that panics on bad
input forces every caller to wrap it in a recover. Error strategy in full is
[Module 08 — Errors](../08-errors/01-error-interface.md).

The stdlib convention: a `Must`-prefixed function panics, and its non-`Must` twin
returns an error — `regexp.MustCompile` / `regexp.Compile`,
`template.Must` / `template.New(...).Parse`. Use `Must` only in package-level
`var` initialisers, where a failure means the binary is unusable anyway.

## Quick Reference

| Function | Signature | Notes |
|---|---|---|
| `panic` | `func panic(v any)` | unwinds, runs defers, prints traceback, exit status 2 |
| `recover` | `func recover() any` | non-nil only in a deferred func of the panicking frame |
| `runtime.Goexit` | `func Goexit()` | ends this goroutine, runs its defers, no panic |
| `debug.Stack` | `func Stack() []byte` | the current goroutine's stack |
| `debug.PrintStack` | `func PrintStack()` | writes the stack to stderr |
| `debug.SetPanicOnFault` | `func SetPanicOnFault(bool) bool` | turn memory faults into recoverable panics |

| Situation | `recover()` returns |
|---|---|
| deferred func, frame is panicking | the panic value |
| deferred func, no panic | `nil` |
| called directly in the function body | `nil` |
| called in a func *called by* the deferred func | `nil` |
| after a nested frame already recovered | `nil` |
| `panic(nil)` | `*runtime.PanicNilError` (Go 1.21+) |
| panic in another goroutine | `nil` — unrecoverable, process dies |
| fatal runtime error (deadlock, OOM, stack overflow) | not a panic; unrecoverable |

| Exit status | Cause |
|---|---|
| 0 | normal return from `main` |
| 1 | `os.Exit(1)`, `log.Fatal` |
| 2 | unrecovered panic, fatal runtime error |
| n | `os.Exit(n)` |

## Common Mistakes

**Calling `recover()` directly in a function body instead of in a `defer`.** It
returns `nil`; it must be called by the deferred function itself.

**Assuming a parent's `recover` protects a goroutine it spawned.** It does not —
the process dies; give every spawned goroutine its own deferred recover.

**Using `panic`/`recover` as exceptions for expected failures.** Return an
`error`; panic is for bugs the caller cannot do anything about.

**Recovering and swallowing without logging.** You have hidden a real bug; log
the value and `debug.Stack()` at minimum.

**Wrapping the panic into an unnamed result.** The caller sees the zero value;
declare the result as `(err error)`.

## Exercises

1. Write a function that panics three frames deep and recovers at the top. Print
   the recovered value's dynamic type with `%T` for a `string` panic, an `error`
   panic, and an index-out-of-range panic, and note which satisfy `error`.
2. Write `func SafeDiv(a, b int) (q int, err error)` that recovers the
   divide-by-zero panic and returns an error instead. Then rewrite it with an
   explicit `if b == 0` check and say which you would ship.
3. Spawn 10 goroutines where one panics. Observe the process dying, then wrap
   each in a recovering helper and confirm the other nine finish. Report the exit
   code both times with `echo $?`.
4. Build the recover middleware, add `slog` logging of `debug.Stack()`, and write
   an `httptest` test asserting a 500 and that the server is still serving
   afterwards.
5. Write a `MustParseConfig(s string) Config` that panics and a
   `ParseConfig(s string) (Config, error)` that does not, with the former
   implemented in terms of the latter. Explain in one sentence when each belongs
   in a library's public API.

---

[← 3.5 defer: Order, Arguments, and Gotchas](05-defer.md) · [Course Index](../README.md) · [4.1 Pointers and the Absence of Pointer Arithmetic →](../04-types-structs/01-pointers.md)
