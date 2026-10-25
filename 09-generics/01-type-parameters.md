[← 8.6 Error Handling Strategy for Libraries vs Services](../08-errors/06-strategy.md) · [Course Index](../README.md) · [9.2 Constraints, comparable, and cmp.Ordered →](02-constraints.md)

# 9.1 — Type Parameters on Functions

## What You'll Learn

- The `[T any]` bracket syntax, multiple type parameters, and one constraint per group
- Explicit instantiation `F[int](x)` versus inferred `F(x)`
- `var zero T` — why `T{}` and `return nil` do not compile
- A type parameter carries no runtime type: you cannot type-switch on `T`, only on `any(v)`

## Coming From Another Language

| Other | Go |
|---|---|
| TS `function f<T>(x: T)` — angle brackets, erased | `func f[T any](x T)` — square brackets, compiled per shape |
| TS `T = {}` default type params | no defaults; omit a param and inference must find it |
| Rust `fn f<T: Display>` | `func f[T fmt.Stringer]` — constraint goes inside the brackets |
| Rust `T::default()` | `var zero T` — no trait to call, just the zero value |
| TS `T extends A ? B : C` conditional types | nothing like it; constraints are interfaces, not computations |
| Rust turbofish `f::<i32>(x)` | `f[int](x)` — same idea, less punctuation |

Go has no variance, no specialization, and no operator overloading. Those three
absences shape everything in this module: you cannot write `a + b` on a `T`
unless the constraint restricts `T` to types that already have `+`, and you
cannot write a faster version of a generic function for one specific type.

## The bracket syntax

Type parameters go in square brackets between the function name and the
parameter list.

```go
package main

import "fmt"

func Identity[T any](v T) T { return v }

func Pair[T, U any](a T, b U) string {
  return fmt.Sprintf("%v/%T %v/%T", a, a, b, b)
}

func Keys[K comparable, V any](m map[K]V) []K {
  out := make([]K, 0, len(m))
  for k := range m {
    out = append(out, k)
  }
  return out
}

func main() {
  fmt.Println(Identity(42), Identity("hi"), Identity[float64](3))
  fmt.Println(Pair(1, "a"))
  fmt.Println(len(Keys(map[string]int{"a": 1, "b": 2})))
}
```

Output:

```
42 hi 3
1/int a/string
2
```

Three forms to notice. `[T, U any]` groups two parameters under one constraint,
exactly like `func(a, b int)`. `[K comparable, V any]` gives each its own.
`Identity[float64](3)` instantiates explicitly, which is why the untyped
constant `3` became a `float64` and printed as `3` rather than `3.0` — `%v` on a
whole float omits the decimal.

`any` is a predeclared alias for `interface{}`, added in Go 1.18 alongside
generics. They are the same type, not two types that happen to match.

## Type parameters in the return type

`T` is in scope for the whole signature, including results.

```go
package main

import "fmt"

func Find[T comparable](xs []T, want T) (T, bool) {
  for _, x := range xs {
    if x == want {
      return x, true
    }
  }
  var zero T
  return zero, false
}

func Fresh[T any]() *T {
  p := new(T)
  return p
}

func main() {
  fmt.Println(Find([]int{1, 2, 3}, 9))
  fmt.Println(Find([]string{"a"}, "z"))
  fmt.Printf("%v %v\n", *Fresh[int](), *Fresh[[]byte]() == nil)
  type P struct {
    X, Y int
  }
  fmt.Println(*Fresh[P]())
}
```

Output:

```
0 false
 false
0 true
{0 0}
```

The second line's leading space is the empty string zero value. `new(T)`
allocates and zeroes a `T` and returns `*T`; it is the only way to get a pointer
to a fresh `T`, since you cannot write `&T{}`.

## You cannot write T{}

Rust has `T::default()`, TS has nothing to construct from a type param either —
but Go's failure mode is a specific compile error worth recognising.

```go
package main

import "fmt"

func zeroOf[T any]() T {
  return T{}
}

func main() {
  fmt.Println(zeroOf[int]())
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:10: invalid composite literal type T (no common underlying type)
```

`T{}` would need `T` to be a struct, slice, or map type. `any` admits `int`, so
there is no composite-literal syntax that could work. Returning `nil` fails for
the mirror-image reason.

```go
package main

import "fmt"

func find[T any](xs []T, ok func(T) bool) T {
  for _, x := range xs {
    if ok(x) {
      return x
    }
  }
  return nil
}

func main() {
  fmt.Println(find([]int{1, 2}, func(i int) bool { return i > 5 }))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:11:10: cannot use nil as T value in return statement
```

Idiomatic: `var zero T` then `return zero`. There is no shorter form, and every
generic function with an early-out return will contain this line.

## No runtime type information

`T` exists at compile time. At runtime the value is just a value, so you cannot
type-switch on it.

```go
package main

import "fmt"

func describe[T any](v T) string {
  switch v.(type) {
  case int:
    return "int"
  default:
    return "other"
  }
}

func main() {
  fmt.Println(describe(1))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:10: cannot use type switch on type parameter value v (variable of type T constrained by any)
```

Convert to `any` first and the switch is legal — but notice what you have done:
you threw away the static type to recover a dynamic one. If you find yourself
reaching for this, the function probably wanted an interface parameter, not a
type parameter (9.5).

## Calling methods on a T

A method call on `T` is legal only when the constraint declares it.

```go
package main

import "fmt"

type Stringish interface {
  String() string
}

func Describe[T any](v T) string {
  switch x := any(v).(type) {
  case int:
    return fmt.Sprintf("int %d", x)
  case string:
    return "string " + x
  default:
    return fmt.Sprintf("other %T", x)
  }
}

func Labels[T Stringish](xs []T) []string {
  out := make([]string, len(xs))
  for i, x := range xs {
    out[i] = x.String()
  }
  return out
}

type ID int

func (i ID) String() string { return fmt.Sprintf("#%d", int(i)) }

func Drain[T any](ch <-chan T) []T {
  var out []T
  for v := range ch {
    out = append(out, v)
  }
  return out
}

func Apply[T any](f func(T) T, v T, n int) T {
  if n == 0 {
    return v
  }
  return Apply(f, f(v), n-1)
}

func Twice[T any](f func(T) T, v T) T { return f(f(v)) }

func Identity[T any](v T) T { return v }

func main() {
  fmt.Println(Describe(1), "|", Describe("a"), "|", Describe(1.5))
  fmt.Println(Labels([]ID{1, 2, 3}))

  ch := make(chan string, 3)
  ch <- "a"
  ch <- "b"
  close(ch)
  fmt.Println(Drain(ch))

  fmt.Println(Apply(func(n int) int { return n * 2 }, 1, 8))

  inc := Identity[int]
  fmt.Println(inc(5))
  fmt.Println(Twice(func(s string) string { return s + "!" }, "go"))

  var e any = 3
  var f interface{} = 3
  fmt.Println(e == f)
}
```

Output:

```
int 1 | string a | other float64
[#1 #2 #3]
[a b]
256
5
go!!
true
```

`Drain` takes a `<-chan T`: type parameters work in channel, slice, map, and
function types wherever a type is allowed. `Apply` recurses with its type
parameter intact — each recursive call is the same instantiation, so there is no
depth explosion.

## Generic function values require instantiation

`inc := Identity[int]` above produced a plain `func(int) int`. Dropping the
brackets does not.

```go
package main

import "fmt"

func Identity[T any](v T) T { return v }

func main() {
  f := Identity
  fmt.Println(f(3))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:8: cannot use generic function Identity without instantiation
```

There is no such thing as a value of generic function type. A `func` value has
one concrete signature, so you must pick the instantiation before you can store
it, pass it, or put it in a slice.

## Map, Filter, Reduce

The motivating trio, and the clearest demonstration of two type parameters where
the second comes only from the callback.

```go
package main

import (
  "fmt"
  "strings"
)

func Map[T, U any](xs []T, f func(T) U) []U {
  out := make([]U, 0, len(xs))
  for _, x := range xs {
    out = append(out, f(x))
  }
  return out
}

func Filter[T any](xs []T, keep func(T) bool) []T {
  var out []T
  for _, x := range xs {
    if keep(x) {
      out = append(out, x)
    }
  }
  return out
}

func Reduce[T, A any](xs []T, init A, f func(A, T) A) A {
  acc := init
  for _, x := range xs {
    acc = f(acc, x)
  }
  return acc
}

func main() {
  words := []string{"go", "rust", "ts", "python"}
  fmt.Println(Map(words, strings.ToUpper))
  fmt.Println(Map(words, func(s string) int { return len(s) }))
  fmt.Println(Filter(words, func(s string) bool { return len(s) > 2 }))
  fmt.Println(Reduce(words, 0, func(n int, s string) int { return n + len(s) }))
  fmt.Println(Reduce(words, "", func(a, s string) string { return a + s[:1] }))
}
```

Output:

```
[GO RUST TS PYTHON]
[2 4 2 6]
[rust python]
14
grtp
```

The standard library deliberately does not ship these — `slices` has `SortFunc`,
`IndexFunc`, and `DeleteFunc` but no `Map` — and 9.5 covers why.

## Generic methods: new in Go 1.27

Through Go 1.26, a method could not introduce its own type parameter. That
restriction is lifted in 1.27, so this compiles:

```go
package main

import "fmt"

type Plain struct{ n int }

func (p Plain) Twice[U any](f func(int) U) (U, U) {
  return f(p.n), f(p.n)
}

func main() {
  a, c := Plain{5}.Twice(func(i int) float64 { return float64(i) / 2 })
  fmt.Println(a, c)
}
```

Output:

```
2.5 2.5
```

Set a lower language version in `go.mod` and the old error comes back, which is
what you will see in any codebase that has not bumped:

```
# example.com/scratch
./main.go:7:22: generic method requires go1.27 or later (-lang was set to go1.24; check go.mod)
```

Two limits remain. An interface method still cannot have type parameters:

```go
package main

type Mapper interface {
  MapTo[U any](f func(int) U) U
}

func main() {}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:4:8: interface method must have no type parameters
./main.go:4:28: undefined: U
```

And a generic method, like a generic function, has no uninstantiated value form:

```go
package main

import "fmt"

type Plain struct{ n int }

func (p Plain) Twice[U any](f func(int) U) U { return f(p.n) }

func main() {
  m := Plain{1}.Twice
  fmt.Println(m)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:8: cannot use generic function Plain{…}.Twice without instantiation
```

Write `m := Plain{1}.Twice[string]` and it works. 9.3 uses generic methods to
build a `Result[T].Map` that chains like Rust's.

## Instantiation cost

The compiler generates one copy of a generic function per *GCShape* — roughly,
per distinct memory layout — so all pointer types share a single instantiation
and reach their type information through a hidden dictionary argument. `Sum[int]`
and `Sum[float64]` get real specialised code; `Sum[*A]` and `Sum[*B]` share one
body. 9.5 benchmarks what that costs.

## Quick Reference

| Form | Meaning |
|---|---|
| `func F[T any](v T) T` | one type parameter, unconstrained |
| `func F[T, U any](a T, b U)` | two parameters sharing a constraint |
| `func F[K comparable, V any](m map[K]V)` | one constraint per group |
| `F(x)` | inferred instantiation |
| `F[int](x)` | explicit instantiation |
| `f := F[int]` | a concrete function value |
| `any` | alias for `interface{}` |
| `var zero T` | the only way to produce a zero `T` |
| `new(T)` → `*T` | a pointer to a fresh zero `T` |
| `any(v).(type)` | the only way to switch on a `T`'s dynamic type |
| `func (r R) M[U any](...)` | generic method, Go 1.27+ |

| Not allowed | Instead |
|---|---|
| `T{}` | `var zero T` |
| `return nil` for `T` | `var zero T; return zero` |
| `switch v.(type)` on `T` | `switch any(v).(type)` |
| `f := F` (generic) | `f := F[int]` |
| interface method with type params | a generic function taking the interface |
| specialising `F` for one `T` | branch on `any(v).(type)` inside `F` |

## Common Mistakes

**Writing `T{}` or `return nil` to produce an empty value.** Declare `var zero T`
and return that.

**Type-switching on a type parameter directly.** Convert with `any(v)` first —
and reconsider whether you wanted an interface parameter.

**Assigning a generic function to a variable without brackets.** Instantiate it:
`f := F[int]`.

**Calling a method on `T` that the constraint does not declare.** Add the method
to the constraint interface, or take an interface parameter instead.

**Assuming `[T, U any]` means two independent constraints.** It means both share
`any`; write `[T any, U comparable]` when they differ.

## Exercises

1. Write `func Zip[A, B any](as []A, bs []B) []struct{ A A; B B }` truncating to
   the shorter input. Call it with mismatched lengths both ways and print the
   result.
2. Write `func First[T any](xs []T, pred func(T) bool) (T, bool)` and prove with
   `%#v` that the failure case returns the true zero value for a struct `T`, a
   pointer `T`, and a map `T`.
3. Implement `func Must[T any](v T, err error) T` that panics on a non-nil error.
   Capture the real panic text when it fires.
4. Try to write a generic function that returns `len(v)` for any `T`. Capture the
   compile error, then fix it two ways: with a type set constraint, and with
   `any(v).(type)`. State which you would ship.
5. Write a `Memoize[K comparable, V any](f func(K) V) func(K) V`. Then try to
   store the uninstantiated `Memoize` in a slice and capture the error.

---

[← 8.6 Error Handling Strategy for Libraries vs Services](../08-errors/06-strategy.md) · [Course Index](../README.md) · [9.2 Constraints, comparable, and cmp.Ordered →](02-constraints.md)
