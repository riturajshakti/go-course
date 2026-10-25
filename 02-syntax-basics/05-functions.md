[← 2.4 Constants, Typed vs Untyped, and iota](04-constants-iota.md) · [Course Index](../README.md) · [2.6 Variadic Functions and Closures →](06-variadic-closures.md)

# 2.5 — Functions, Multiple Returns, and Named Results

## What You'll Learn

- Multiple return values and the `(value, error)` convention that replaces exceptions
- Named results exist mainly so `defer` can modify them
- You must account for every return value or the code does not compile
- No default parameters, no overloading, no optional arguments

## Coming From Another Language

| Other | Go |
|---|---|
| JS `[a, b] = f()` destructuring a tuple | `a, b := f()` — a real multi-value return, not an array |
| JS `function f(a, b = 1)` | no default parameters; use an options struct |
| Python `def f(*, timeout=5)` | no keyword arguments |
| Python exceptions | returned `error` values; `panic` is for bugs only |
| Rust `Result<T, E>` + `?` | `(T, error)` + a 3-line `if err != nil` at every call |
| TS overloads | not supported; one name, one signature |

## Parameters and returns

```go
package main

import (
  "errors"
  "fmt"
  "strconv"
)

func add(a int, b int) int { return a + b }

func add2(a, b int) int { return a + b }

func noReturn(s string) { fmt.Println("got", s) }

func divmod(a, b int) (int, int) { return a / b, a % b }

func parse(s string) (int, error) {
  n, err := strconv.Atoi(s)
  if err != nil {
    return 0, fmt.Errorf("parse %q: %w", s, err)
  }
  return n, nil
}

var ErrNotFound = errors.New("not found")

func lookup(k string) (string, error) {
  m := map[string]string{"a": "alpha"}
  v, ok := m[k]
  if !ok {
    return "", ErrNotFound
  }
  return v, nil
}

func main() {
  fmt.Println(add(1, 2), add2(3, 4))
  noReturn("hi")

  q, r := divmod(17, 5)
  fmt.Println(q, r)

  n, err := parse("42")
  fmt.Println(n, err)
  n, err = parse("4x")
  fmt.Println(n, err)
  fmt.Println(errors.Unwrap(err))

  v, err := lookup("a")
  fmt.Println(v, err)
  v, err = lookup("z")
  fmt.Println(v, err, errors.Is(err, ErrNotFound))
}
```

Output:

```
3 7
got hi
3 2
42 <nil>
0 parse "4x": strconv.Atoi: parsing "4x": invalid syntax
strconv.Atoi: parsing "4x": invalid syntax
alpha <nil>
 not found true
```

`func add2(a, b int)` collapses consecutive same-typed parameters. The type comes *after* the name, and the return type after the parameter list — read declarations left to right, unlike C.

Two conventions do most of the work in Go code:

- `(T, error)` — the error is always **last**, and it is `nil` on success. The value is conventionally the zero value when the error is non-nil, and you must not read it.
- `(T, bool)` — the "comma ok" form, for lookups where absence is not an error: map reads, type assertions, and channel receives.

## Every return value must be accounted for

```go
package main

import (
  "fmt"
  "strconv"
)

func main() {
  n := strconv.Atoi("42")
  fmt.Println(n)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:8: assignment mismatch: 1 variable but strconv.Atoi returns 2 values
```

The count must match exactly in both directions:

```go
package main

import "fmt"

func one() int { return 1 }

func main() {
  a, b := one()
  fmt.Println(a, b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:11: assignment mismatch: 2 variables but one returns 1 value
```

And a multi-value call cannot appear inside a larger expression:

```go
package main

import (
  "fmt"
  "strconv"
)

func main() {
  fmt.Println(strconv.Atoi("1") + 1)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:15: multiple-value strconv.Atoi("1") (value of type (int, error)) in single-value context
```

The one exception: a multi-value call may be the *sole* argument to a function whose parameters match, as in `fmt.Println(divmod(17, 5))`.

To discard a value, name it `_`. There is no way to call a two-value function and silently ignore the second — the compiler makes the discard visible at every call site, which is the whole point.

```go
package main

import (
  "fmt"
  "strconv"
)

func divmod(a, b int) (int, int) { return a / b, a % b }

func main() {
  n, _ := strconv.Atoi("42")
  fmt.Println(n)

  _, err := strconv.Atoi("4x")
  fmt.Println(err)

  fmt.Println(divmod(17, 5))

  strconv.Atoi("ignored")
}
```

Output:

```
42
strconv.Atoi: parsing "4x": invalid syntax
3 2
```

The bare `strconv.Atoi("ignored")` as a statement compiles — a function call may always be used as a statement and its results dropped wholesale. `go vet` does not flag it, but `errcheck` and `golangci-lint` do (see 15.3).

## Named results

Naming the results declares them as variables, initialized to their zero values, scoped to the whole function.

```go
package main

import (
  "errors"
  "fmt"
  "strconv"
)

func split(sum int) (x, y int) {
  x = sum * 4 / 9
  y = sum - x
  return
}

func zeroed() (n int, err error) {
  return
}

func safeParse(s string) (n int, err error) {
  defer func() {
    if r := recover(); r != nil {
      err = fmt.Errorf("recovered: %v", r)
    }
  }()
  if s == "boom" {
    panic("bad input")
  }
  return strconv.Atoi(s)
}

func annotate(path string) (err error) {
  defer func() {
    if err != nil {
      err = fmt.Errorf("open %s: %w", path, err)
    }
  }()
  return errors.New("permission denied")
}

func main() {
  fmt.Println(split(18))
  fmt.Println(zeroed())
  fmt.Println(safeParse("12"))
  fmt.Println(safeParse("x"))
  fmt.Println(safeParse("boom"))
  fmt.Println(annotate("/etc/shadow"))
}
```

Output:

```
8 10
0 <nil>
12 <nil>
0 strconv.Atoi: parsing "x": invalid syntax
0 recovered: bad input
open /etc/shadow: permission denied
```

The two functions at the bottom are the real reason named results exist. `safeParse` recovers from a panic and turns it into an error; `annotate` wraps whatever error the body returned. Neither is expressible with anonymous results, because the deferred closure needs a *name* to assign to.

Note that `return strconv.Atoi(s)` works even with named results — the returned values are assigned to `n` and `err` before the defers run.

## A deferred function can only modify a named result

```go
package main

import "fmt"

func counted() (n int) {
  defer func() { n *= 10 }()
  n = 5
  return n
}

func notCounted() int {
  n := 5
  defer func() { n *= 10 }()
  return n
}

func main() {
  fmt.Println(counted(), notCounted())
}
```

Output:

```
50 5
```

Identical bodies, different signatures, different answers. In `notCounted` the `return n` copies `n` into the (anonymous) result slot first; the defer then multiplies a local that nobody reads. In `counted`, `n` *is* the result slot, so the defer's write is what the caller sees. Full treatment of defer ordering in 3.5.

## Naked return

A bare `return` in a function with named results returns their current values. It is what `split` above uses.

Idiomatic: acceptable in a short function (under about ten lines) where the names document the results, and in the `recover`/wrap pattern. Avoid it in anything longer — the reader has to scan the whole body to learn what is returned, and inserting an assignment in the middle silently changes the result. Prefer `return x, y` explicitly even when the results are named.

## Missing return

Go requires the function to end in a terminating statement; it does not reason about exhaustive conditions.

```go
package main

func sign(n int) string {
  if n > 0 {
    return "+"
  } else if n < 0 {
    return "-"
  }
}

func main() {
  _ = sign(1)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:1: missing return
```

An `if/else if` with no final `else` is not terminating even when it covers every case. Drop the `else` and return at the end — which is the shape Go wants anyway.

## No default parameters

```go
package main

import "fmt"

func greet(name string, greeting string = "hello") string {
  return greeting + " " + name
}

func main() {
  fmt.Println(greet("go"))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:5:41: syntax error: unexpected = in parameter list; possibly missing comma or )
```

The replacement is a struct of options, where the zero value is the default. The caller names only what it changes.

```go
package main

import (
  "fmt"
  "time"
)

type DialOptions struct {
  Timeout time.Duration
  Retries int
  Verbose bool
}

func Dial(addr string, opts DialOptions) string {
  if opts.Timeout == 0 {
    opts.Timeout = 5 * time.Second
  }
  return fmt.Sprintf("%s timeout=%v retries=%d verbose=%v",
    addr, opts.Timeout, opts.Retries, opts.Verbose)
}

func main() {
  fmt.Println(Dial("db:5432", DialOptions{}))
  fmt.Println(Dial("db:5432", DialOptions{Retries: 3}))
  fmt.Println(Dial("db:5432", DialOptions{Timeout: time.Second, Verbose: true}))
}
```

Output:

```
db:5432 timeout=5s retries=0 verbose=false
db:5432 timeout=5s retries=3 verbose=false
db:5432 timeout=1s retries=0 verbose=true
```

The weakness is visible on the first line: you cannot distinguish "caller wants no timeout" from "caller said nothing", because both are `0`. When that matters, use a pointer field (`*time.Duration`) or the functional options pattern, which is the subject of 4.8.

## No overloading

```go
package main

import "fmt"

func describe(n int) string { return "int" }

func describe(s string) string { return "string" }

func main() {
  fmt.Println(describe(1))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:6: describe redeclared in this block
	./main.go:5:6: other declaration of describe
```

One identifier, one declaration, no exceptions. The stdlib's answer is distinct names: `strconv.Itoa` / `FormatInt` / `FormatFloat`, `fmt.Print` / `Printf` / `Println`. Since 1.18, generics cover the cases where the bodies really are identical (module 09).

## Functions are values

```go
package main

import (
  "fmt"
  "sort"
  "strings"
)

type Transform func(string) string

func apply(xs []string, f Transform) []string {
  out := make([]string, len(xs))
  for i, s := range xs {
    out[i] = f(s)
  }
  return out
}

func compose(f, g Transform) Transform {
  return func(s string) string { return g(f(s)) }
}

func main() {
  var f Transform = strings.ToUpper
  fmt.Println(f("go"), f == nil)

  var g Transform
  fmt.Println(g == nil)

  words := []string{"go", "rust"}
  fmt.Println(apply(words, strings.ToUpper))
  fmt.Println(apply(words, compose(strings.ToUpper, func(s string) string { return s + "!" })))

  nums := []int{3, 1, 2}
  sort.Slice(nums, func(i, j int) bool { return nums[i] < nums[j] })
  fmt.Println(nums)

  fns := map[string]Transform{
    "up":   strings.ToUpper,
    "trim": strings.TrimSpace,
  }
  fmt.Printf("%q %q\n", fns["up"]("x"), fns["trim"]("  y  "))
}
```

Output:

```
GO false
true
[GO RUST]
[GO! RUST!]
[1 2 3]
"X" "y"
```

A `func` type's zero value is `nil`, and calling a nil func panics. `strings.ToUpper` is assignable to `Transform` because a defined func type's underlying signature is what matters, not the name. Func values can go in slices, maps, struct fields, and channels — but they are not comparable to each other (2.3), only to `nil`.

## Recursion, including mutual

```go
package main

import "fmt"

func fib(n int) int {
  if n < 2 {
    return n
  }
  return fib(n-1) + fib(n-2)
}

func isEven(n int) bool {
  if n == 0 {
    return true
  }
  return isOdd(n - 1)
}

func isOdd(n int) bool {
  if n == 0 {
    return false
  }
  return isEven(n - 1)
}

func main() {
  fmt.Println(fib(10))
  fmt.Println(isEven(10), isOdd(10))
}
```

Output:

```
55
true false
```

Declaration order does not matter at package level, so `isEven` can call `isOdd` declared below it with no forward declaration. Go does **not** guarantee tail-call optimization, so deep recursion grows the stack — though goroutine stacks start at 8 KB and grow to a 1 GB default limit, so the depths that break in other languages usually work here. A recursive *anonymous* function needs a `var f func(int) int` declared before the assignment, because `f := func(...)` has not bound `f` yet inside the body.

## Quick Reference

| Form | Meaning |
|---|---|
| `func f(a int, b string) bool` | one type each |
| `func f(a, b int) bool` | shared type |
| `func f() (int, error)` | multiple results |
| `func f() (n int, err error)` | named results |
| `func f()` | no result |
| `return` | naked return; named results only |
| `return a, b` | explicit |
| `type F func(int) string` | a function type |
| `var f F = g` | a function value |
| `_, err := f()` | discard the first result |
| `f()` as a statement | discard all results |

| Convention | Shape |
|---|---|
| fallible operation | `(T, error)`, error last, `nil` on success |
| lookup where absence is normal | `(T, bool)`, the "comma ok" form |
| mutating in place | return nothing, take a pointer or slice |
| builder/chaining | return the receiver |
| optional arguments | an options struct (here) or functional options (4.8) |

| Not in Go | Use instead |
|---|---|
| default parameters | options struct, or a second function |
| keyword arguments | options struct with field names |
| overloading | distinct names, or generics |
| exceptions | returned `error` |
| tuples | multiple return values |
| guaranteed tail calls | a loop |

## Common Mistakes

**Reading the value when `err != nil`.** It is conventionally the zero value and carries no information; return early instead.

**Using named results everywhere because they look self-documenting.** They enable naked returns, which obscure long functions; name them when a `defer` must modify them.

**Expecting `if/else if` covering all cases to satisfy the compiler.** It still reports `missing return`; end the function with an unconditional `return`.

**Calling a function and ignoring its error by just not assigning it.** It compiles silently; use `_ =` to make the choice explicit, and run `errcheck`.

**Reaching for an options struct when a required argument is involved.** Required arguments belong in the signature; options are for the ones with sensible defaults.

## Exercises

1. Write `func stat(path string) (size int64, mode fs.FileMode, err error)` wrapping `os.Stat`, and call it three ways: all three results, discarding `mode`, and discarding everything but `err`. Confirm each compiles.
2. Write a `func mustParse(s string) int` that panics, and a `func tryParse(s string) (n int, err error)` that recovers from `mustParse` and converts the panic into an error. The recovery must use a named result.
3. Write a `func middleware(next http.HandlerFunc) http.HandlerFunc` that logs the duration of each call. Do it without closures first (you will find you cannot), then with one.
4. Implement `func apply[T, U any](xs []T, f func(T) U) []U` and use it to map `[]string` to `[]int` lengths. Then write the non-generic version for one concrete pair and compare the call sites.
5. Convert this signature to an options struct: `func NewServer(addr string, readTimeout, writeTimeout time.Duration, maxConns int, tls bool) *Server`. Then convert it again to functional options and write one sentence on which you would prefer for a library and which for internal code.

---

[← 2.4 Constants, Typed vs Untyped, and iota](04-constants-iota.md) · [Course Index](../README.md) · [2.6 Variadic Functions and Closures →](06-variadic-closures.md)
