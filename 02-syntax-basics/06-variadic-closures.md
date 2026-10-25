[← 2.5 Functions, Multiple Returns, and Named Results](05-functions.md) · [Course Index](../README.md) · [2.7 Comments, Doc Comments, and go doc →](07-comments-docs.md)

# 2.6 — Variadic Functions and Closures

## What You'll Learn

- A variadic parameter *is* a slice, so `f(xs...)` lets the callee mutate your array
- Closures capture variables by reference, not by value
- Go 1.22 changed loop variables to per-iteration, fixing the classic closure bug
- Zero variadic arguments give a `nil` slice; `f([]T{}...)` gives an empty non-nil one

## Coming From Another Language

| Other | Go |
|---|---|
| JS `...args` → a real Array | `nums ...int` → a real `[]int` |
| JS `f(...arr)` spread | `f(arr...)` — suffix, and it **aliases** `arr` |
| Python `*args` → a tuple (immutable) | a slice (mutable, and shared with the caller) |
| Python `**kwargs` | no equivalent; use an options struct |
| JS closures capture `var` per-function, `let` per-iteration | Go captures by reference; loop vars are per-iteration since 1.22 |
| Rust `move` closures | no `move`; capture a copy by introducing a new variable |

## Variadic basics

```go
package main

import "fmt"

func sum(nums ...int) int {
  t := 0
  for _, n := range nums {
    t += n
  }
  return t
}

func describe(nums ...int) {
  fmt.Printf("%T len=%d cap=%d nil=%v %v\n", nums, len(nums), cap(nums), nums == nil, nums)
}

func logf(level string, format string, args ...any) string {
  return level + ": " + fmt.Sprintf(format, args...)
}

func main() {
  fmt.Println(sum())
  fmt.Println(sum(1))
  fmt.Println(sum(1, 2, 3))

  xs := []int{4, 5, 6}
  fmt.Println(sum(xs...))

  describe()
  describe(1, 2)
  describe(xs...)
  describe([]int{}...)
  describe([]int(nil)...)

  fmt.Println(logf("INFO", "user %s id=%d", "ada", 7))
  fmt.Println(logf("WARN", "no args"))
}
```

Output:

```
0
1
6
15
[]int len=0 cap=0 nil=true []
[]int len=2 cap=2 nil=false [1 2]
[]int len=3 cap=3 nil=false [4 5 6]
[]int len=0 cap=0 nil=false []
[]int len=0 cap=0 nil=true []
INFO: user ada id=7
WARN: no args
```

Inside the function, `nums` is an ordinary `[]int` — `len`, `cap`, `range`, indexing, and `append` all work. With no arguments it is `nil`, not an empty slice, which is almost always harmless because `len(nil) == 0` and `range nil` iterates zero times. The distinction shows up only if you check `== nil` or marshal it to JSON (`null` vs `[]`).

`logf` shows the common relay pattern: a `...any` parameter passed straight through with `args...`. `fmt.Printf`, `fmt.Println`, `log.Printf`, `slog.Info`, and `append` are all variadic.

A fixed parameter may precede the variadic one, which must be last.

## The variadic parameter aliases the caller's slice

This is the gotcha. `f(xs...)` does **not** copy — the callee's `nums` is a slice header pointing at `xs`'s array.

```go
package main

import "fmt"

func zeroFirst(nums ...int) {
  if len(nums) > 0 {
    nums[0] = 0
  }
}

func appendIn(nums ...int) []int {
  return append(nums, 999)
}

func main() {
  xs := []int{1, 2, 3}
  zeroFirst(xs...)
  fmt.Println("after zeroFirst(xs...):", xs)

  ys := []int{1, 2, 3}
  zeroFirst(ys[0], ys[1], ys[2])
  fmt.Println("after spelling out args:", ys)

  zs := make([]int, 3, 8)
  copy(zs, []int{1, 2, 3})
  got := appendIn(zs...)
  fmt.Println(zs, len(zs), cap(zs), got)
  fmt.Println(zs[:4])
}
```

Output:

```
after zeroFirst(xs...): [0 2 3]
after spelling out args: [1 2 3]
[1 2 3] 3 8 [1 2 3 999]
[1 2 3 999]
```

Two distinct surprises here:

1. `zeroFirst(xs...)` mutated the caller's slice. The same call with the arguments spelled out built a fresh array, so `ys` is untouched. The *syntax of the call site* decides whether the callee can write through to you.
2. `appendIn(zs...)` appended into `zs`'s spare capacity. `len(zs)` is still 3 so the 999 is invisible through `zs`, but re-slicing to `zs[:4]` exposes it.

If a variadic function may retain or mutate its arguments, copy inside: `nums = slices.Clone(nums)`. Full treatment of aliasing in 7.4.

## What the compiler rejects

```go
package main

func bad(nums ...int, label string) {}

func main() {
  bad(1, "x")
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:3:15: can only use ... with final parameter
```

A slice spread cannot be mixed with extra arguments:

```go
package main

import "fmt"

func sum(nums ...int) int { return len(nums) }

func main() {
  xs := []int{1, 2}
  fmt.Println(sum(xs..., 3))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:26: syntax error: unexpected literal 3, expected )
```

Write `sum(append(xs, 3)...)` — noting that `append` may mutate `xs` — or `sum(append(slices.Clone(xs), 3)...)` if it must not.

Forgetting the `...` is a type error, which is the compiler saving you:

```go
package main

func sum(nums ...int) int { return len(nums) }

func main() {
  xs := []int{1, 2}
  _ = sum(xs)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:11: cannot use xs (variable of type []int) as int value in argument to sum
```

And `...any` is not satisfied by `[]string`, because `[]string` and `[]any` have different memory layouts:

```go
package main

import "fmt"

func main() {
  xs := []string{"a", "b"}
  fmt.Println(xs...)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:15: cannot use xs (variable of type []string) as []any value in argument to fmt.Println
```

Converting requires a loop that copies each element into an `[]any`.

## Closures capture by reference

```go
package main

import "fmt"

func main() {
  n := 1
  show := func() { fmt.Println("n is", n) }
  show()
  n = 42
  show()

  xs := []int{1, 2, 3}
  grow := func() { xs = append(xs, 4) }
  grow()
  fmt.Println(xs)

  byValue := func(v int) func() { return func() { fmt.Println("captured", v) } }
  f := byValue(n)
  n = 7
  f()
}
```

Output:

```
n is 1
n is 42
[1 2 3 4]
captured 42
```

`show` sees the *current* `n`, not the value at creation time, and `grow` can reassign the caller's `xs` — a closure can do what a function parameter cannot. Captured variables are moved to the heap (escape analysis; see 15.6), so they outlive the enclosing function.

To capture a *copy*, pass the value as a parameter, as `byValue` does. There is no `move` keyword.

## Loop variables: Go 1.22 changed this

Before Go 1.22 there was one loop variable per *loop*, shared by every iteration, so closures created in the body all saw the final value. This was the single most reported Go gotcha. Since 1.22 each iteration gets its own variable.

```go
package main

import "fmt"

func main() {
  var fns []func() int
  for i := 0; i < 3; i++ {
    fns = append(fns, func() int { return i })
  }
  for _, f := range fns {
    fmt.Print(f(), " ")
  }
  fmt.Println()

  var fns2 []func() string
  for _, s := range []string{"a", "b", "c"} {
    fns2 = append(fns2, func() string { return s })
  }
  for _, f := range fns2 {
    fmt.Print(f(), " ")
  }
  fmt.Println()

  var fns3 []func() int
  for i := range 3 {
    j := i
    fns3 = append(fns3, func() int { return j })
  }
  for _, f := range fns3 {
    fmt.Print(f(), " ")
  }
  fmt.Println()

  var fns4 []func() int
  k := 0
  for k = 0; k < 3; k++ {
    fns4 = append(fns4, func() int { return k })
  }
  for _, f := range fns4 {
    fmt.Print(f(), " ")
  }
  fmt.Println()
}
```

Output on Go 1.27.1:

```
0 1 2 
a b c 
0 1 2 
3 3 3 
```

The first three lines all print the distinct values. The third shows the old `j := i` workaround, now redundant but harmless.

The fourth line is the exception worth memorizing: `k` is declared **outside** the loop and merely assigned by the `for` clause, so there is still only one variable, and all three closures print `3` — the value that failed the condition. Per-iteration scoping applies only to variables the `for` statement itself declares.

The behaviour is keyed to the `go` directive in `go.mod`, not the toolchain version, so you can see both on one machine. The same first loop, compiled under `go 1.21`:

```
3 3 3 
```

and under `go 1.22`:

```
0 1 2 
```

So a module that has not bumped its `go` line still has the old semantics even on Go 1.27. If you maintain pre-1.22 code, or read it, assume the shared variable. `go vet` has a `loopclosure` analyzer that catches the most dangerous residual cases.

## Closures as counters and generators

```go
package main

import "fmt"

func counter() func() int {
  n := 0
  return func() int {
    n++
    return n
  }
}

func counterPair() (func() int, func()) {
  n := 0
  inc := func() int { n++; return n }
  reset := func() { n = 0 }
  return inc, reset
}

func accumulator() func(int) int {
  total := 0
  return func(d int) int {
    total += d
    return total
  }
}

func fibGen() func() int {
  a, b := 0, 1
  return func() int {
    r := a
    a, b = b, a+b
    return r
  }
}

func main() {
  c1 := counter()
  c2 := counter()
  fmt.Println(c1(), c1(), c1(), c2())

  inc, reset := counterPair()
  fmt.Println(inc(), inc())
  reset()
  fmt.Println(inc())

  acc := accumulator()
  fmt.Println(acc(10), acc(5), acc(-3))

  next := fibGen()
  for range 8 {
    fmt.Print(next(), " ")
  }
  fmt.Println()

  x := 1
  func() { x = 99 }()
  fmt.Println(x)

  msg := func(s string) string { return "<" + s + ">" }("hi")
  fmt.Println(msg)
}
```

Output:

```
1 2 3 1
1 2
1
10 15 12
0 1 1 2 3 5 8 13 
99
<hi>
```

Each call to `counter()` creates a fresh `n`, so `c1` and `c2` are independent — that is the object-without-a-struct pattern. `counterPair` returns two closures over the *same* `n`, which is shared mutable state: safe here, a data race the moment two goroutines call it.

The last two are the immediately-invoked form. `func() { ... }()` runs at once, useful for scoping a few statements or for a `defer func(){...}()`. Note the trailing `()` — without it you have a func value, not a call, and `go vet` will not tell you.

## Closures and goroutines

Since 1.22 the common case is finally correct without a workaround.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var wg sync.WaitGroup
  results := make([]int, 5)
  for i := range 5 {
    wg.Add(1)
    go func() {
      defer wg.Done()
      results[i] = i * i
    }()
  }
  wg.Wait()
  fmt.Println(results)
}
```

Output:

```
[0 1 4 9 16]
```

This is also clean under `go run -race .`, because each goroutine writes a distinct index. Under `go 1.21` semantics every goroutine would share one `i` and the result would be a race with nondeterministic output.

Per-iteration loop variables fix the *capture*, not concurrency in general: any closure that writes shared state from several goroutines still needs a mutex or a channel. Module 10 onward.

## Quick Reference

| Form | Meaning |
|---|---|
| `func f(xs ...T)` | variadic; `xs` is a `[]T`, must be the last parameter |
| `func f(a T, xs ...U)` | fixed parameters may precede |
| `f()` | `xs` is `nil`, `len` 0 |
| `f(a, b)` | `xs` is a new `[]T{a, b}` |
| `f(s...)` | `xs` aliases `s`; no copy |
| `f([]T{}...)` | `xs` is non-nil with `len` 0 |
| `f(append(s, v)...)` | append then spread; may mutate `s` |
| `func() { ... }()` | immediately invoked |
| `func() T { ... }` as a value | a closure over the enclosing scope |

| Variadic in the stdlib | Signature |
|---|---|
| `append` | `append(s []T, v ...T) []T` |
| `fmt.Println` | `(a ...any) (int, error)` |
| `fmt.Printf` | `(format string, a ...any) (int, error)` |
| `errors.Join` | `(errs ...error) error` |
| `slices.Concat` | `(s ...[]T) []T` |
| `max`, `min` (builtins) | `(x T, y ...T) T` |

| Loop variable semantics | `go` directive in go.mod |
|---|---|
| one variable per loop (closures see the last value) | `go 1.21` or lower |
| one variable per iteration | `go 1.22` or higher |
| one variable per loop, always | the variable is declared outside the `for` |

## Common Mistakes

**Passing `xs...` to a function that mutates or appends.** The callee shares your array; clone it on either side.

**Checking `len(args) == 0` to mean "caller passed nothing".** `f([]T{}...)` also gives zero length; compare `args == nil` if the distinction matters.

**Assuming the pre-1.22 loop-variable workaround is still needed.** `j := i` is now redundant for variables the `for` declares — but still required when the variable is declared outside it.

**Capturing a variable in a closure and expecting the value at creation time.** Capture is by reference; pass it as a parameter to snapshot it.

**Writing `go func() { ... }` without the trailing `()`.** That is a syntax error in a `go` statement, but inside a slice literal it silently stores the function instead of calling it.

## Exercises

1. Write `func joinAll(sep string, parts ...string) string` without `strings.Join`. Call it with zero, one, and many arguments, and with a spread slice. Print `len`, `cap`, and `== nil` of the parameter in each case.
2. Write a variadic function that is documented as not retaining its arguments, then write a test that proves it: mutate the caller's slice afterward and assert the function's stored copy did not change.
3. Build a `func rateLimited(n int, f func()) func() bool` closure that lets `f` run at most `n` times and reports whether it ran. Then make it safe for concurrent use and verify with `go test -race`.
4. Reproduce the pre-1.22 behaviour on your own machine: create a module with `go 1.21` in `go.mod`, run a loop that appends three closures, then change the line to `go 1.22` and run again. Record both outputs.
5. Write a `func memoize(f func(int) int) func(int) int` using a captured map, apply it to a naive recursive `fib`, and compare the call counts for `fib(30)` with and without it. Then explain why the recursive calls inside `fib` are *not* memoized and fix it.

---

[← 2.5 Functions, Multiple Returns, and Named Results](05-functions.md) · [Course Index](../README.md) · [2.7 Comments, Doc Comments, and go doc →](07-comments-docs.md)
