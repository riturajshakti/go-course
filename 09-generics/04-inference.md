[← 9.3 Generic Types and Methods](03-generic-types.md) · [Course Index](../README.md) · [9.5 When Generics Are the Wrong Tool →](05-when-not-to.md)

# 9.4 — Type Inference and Its Limits

## What You'll Learn

- Inference works from arguments, and only from arguments
- The #1 failure: a type parameter that appears only in the return type
- Partial instantiation must supply a *prefix* of the parameters
- Untyped constants infer `int`, not `float64` — and integer division follows
- When `T` becomes the concrete type and when it becomes the interface

## Coming From Another Language

| Other | Go |
|---|---|
| TS infers from return context: `const x: string[] = f()` | Go never looks at the assignment target |
| TS `f<string>()` to pin a return-only param | `f[string]()` — same fix, mandatory more often |
| Rust `let x: Vec<i32> = v.collect()` | impossible; Go has no return-type inference |
| Rust `v.collect::<Vec<_>>()` turbofish | `Collect[[]int](seq)` — same necessity |
| TS `f(null)` infers `null`/`any` | `f(nil)` cannot infer at all |
| Python/TS `f(1)` with float params | `f(1)` infers `int`; `1/2` is `0` |

The governing rule: Go's inference is *bottom-up from the argument list only*.
It never consults the type of the variable you are assigning to, never the
surrounding function's return type, and never a later statement. If the argument
list does not pin a parameter, you must write it.

## Inference from arguments

The common case, and the reason generic code reads well most of the time.

```go
package main

import "fmt"

type Pair[K comparable, V any] struct {
  K K
  V V
}

func MakePair[K comparable, V any](k K, v V) Pair[K, V] { return Pair[K, V]{k, v} }

func Sum[T ~int | ~float64](xs []T) T {
  var t T
  for _, x := range xs {
    t += x
  }
  return t
}

func main() {
  p := MakePair("a", 1)
  fmt.Printf("%v %T\n", p, p)
  fmt.Println(Sum([]float64{1, 2}))
  var xs = []int{1, 2}
  fmt.Println(Sum(xs))
  type MyInt int
  fmt.Println(Sum([]MyInt{1, 2}))
}
```

Output:

```
{a 1} main.Pair[string,int]
3
3
3
```

Note `%T` prints the instantiated type with its arguments: `main.Pair[string,int]`.
That notation is what you will see in panic messages and `go vet` output, so it
is worth recognising.

## The #1 failure: a parameter only in the return type

```go
package main

import "fmt"

func Zero[T any]() T {
  var zero T
  return zero
}

func main() {
  fmt.Println(Zero())
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:11:19: in call to Zero, cannot infer T (declared at ./main.go:5:11)
```

`Zero()` has no arguments, so there is nothing to infer from. The fix is always
explicit instantiation — `Zero[int]()`. This bites hardest on constructors
(`NewStack[string]()`), on decoders (`DecodeInto[Config](data)`), and on anything
shaped like `collect`.

Even assigning to a typed variable does not help:

```go
package main

import "fmt"

func Zero[T any]() T {
  var zero T
  return zero
}

func main() {
  var n int = Zero()
  fmt.Println(n)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:11:19: in call to Zero, cannot infer T (declared at ./main.go:5:11)
```

Identical error. TypeScript would have inferred `string` from
`const x: string = f()`; Go does not look there at all.

## Partial instantiation takes a prefix

You may supply some parameters and let the rest be inferred — but only in order,
left to right, with no gaps.

```go
package main

import "fmt"

func Make[T any, U any]() (T, U) {
  var t T
  var u U
  return t, u
}

func main() {
  fmt.Println(Make[int]())
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:12:24: in call to Make[int], cannot infer U (declared at ./main.go:5:18)
```

`T` was accepted, `U` was not inferable. And there is no syntax for skipping a
parameter — a hole in the list is a parse error:

```go
package main

import "fmt"

func Convert[T, U any](v T, f func(T) U) U { return f(v) }

func main() {
  fmt.Println(Convert[, string](1, func(i int) string { return "x" }))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:23: syntax error: unexpected comma, expected expression
```

So if you need to pin the *second* parameter, you must write the first as well.
Practical consequence: order your type parameters with the ones callers are most
likely to need to pin first. `Map[T, U]` is the wrong way round if callers
usually need to pin `U`; `Reduce[T, A any]` works because `A` is almost always
inferred from `init`.

## Untyped constants infer int

```go
package main

import "fmt"

func Sum[T ~int | ~float64](xs ...T) T {
  var t T
  for _, x := range xs {
    t += x
  }
  return t
}

func Half[T ~int | ~float64](v T) T { return v / 2 }

func main() {
  fmt.Printf("%v %T\n", Sum(1, 2, 3), Sum(1, 2, 3))
  fmt.Printf("%v %T\n", Half(7), Half(7))
  fmt.Printf("%v %T\n", Half[float64](7), Half[float64](7))
  fmt.Printf("%v %T\n", Half(7.0), Half(7.0))
  fmt.Printf("%v %T\n", Sum(1, 2, 3.5), Sum(1, 2, 3.5))
}
```

Output:

```
6 int
3 int
3.5 float64
3.5 float64
6.5 float64
```

Line two is the surprise. `Half(7)` gave `3`, not `3.5`, because an untyped
integer constant infers `int` and `7 / 2` is integer division. Two fixes, both
shown: instantiate explicitly as `Half[float64](7)`, or write the literal as
`7.0` so its default type is `float64`.

The last line shows the mixed case: when untyped constants of different kinds
appear in one `T`, Go takes the widest default type among them, so `1, 2, 3.5`
all become `float64`. Idiomatic: in numeric generic code called with literals,
instantiate explicitly. It costs seven characters and removes a whole class of
bug.

## nil cannot be inferred

```go
package main

import "fmt"

func Wrap[T any](v T) []T { return []T{v} }

func main() {
  fmt.Println(Wrap(nil))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:19: in call to Wrap, cannot infer T (declared at ./main.go:5:11)
```

`nil` is untyped and has no default type, so it contributes nothing.
`Wrap[*int](nil)` and `Wrap[error](nil)` both work.

## One T, two different concrete types

When a single `T` appears in several parameters, every argument must agree.

```go
package main

import "fmt"

func Equal[T comparable](a, b T) bool { return a == b }

func main() {
  fmt.Println(Equal(1, "one"))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:24: in call to Equal, mismatched types untyped int and untyped string (cannot infer T)
```

With *typed* arguments the error reads differently and is more informative,
because inference succeeded on the first argument and then failed to match:

```go
package main

import "fmt"

func Equal[T comparable](a, b T) bool { return a == b }

func main() {
  var i int32 = 1
  var j int64 = 1
  fmt.Println(Equal(i, j))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:24: in call to Equal, type int64 of j does not match inferred type int32 for T
```

Note "inferred type int32" — the first argument won. There is no widening, no
numeric promotion; convert explicitly at the call site.

## Inference from a function-typed argument

Both parameters can come from one callback, and the argument order matters for
the error messages you get.

```go
package main

import "fmt"

func Map[T, U any](xs []T, f func(T) U) []U {
  out := make([]U, len(xs))
  for i, x := range xs {
    out[i] = f(x)
  }
  return out
}

func Apply[T, U any](f func(T) U, xs []T) []U { return Map(xs, f) }

func double(i int) int { return i * 2 }

func main() {
  fmt.Println(Map([]int{1, 2}, func(x int) string { return fmt.Sprint(x) }))
  fmt.Println(Apply(double, []int{1, 2}))
  fmt.Println(Apply(func(s string) int { return len(s) }, []string{"ab"}))
  var f func(int) bool = func(i int) bool { return i > 1 }
  fmt.Println(Map([]int{1, 2}, f))
  fmt.Println(Map[int, float64]([]int{1, 2}, func(x int) float64 { return float64(x) / 2 }))
}
```

Output:

```
[1 2]
[2 4]
[2]
[false true]
[0.5 1]
```

All five work. The closure's parameter types must be written out, because they
are what the inference reads — you cannot leave them bare and hope `T` flows in:

```go
package main

import "fmt"

func Map[T, U any](xs []T, f func(T) U) []U {
  out := make([]U, len(xs))
  for i, x := range xs {
    out[i] = f(x)
  }
  return out
}

func main() {
  fmt.Println(Map([]int{1, 2}, func(x) string { return fmt.Sprint(x) }))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:14:37: undefined: x
```

That error is baffling until you realise `func(x) string` parsed `x` as a *type
name*, not a parameter name. Go has no contextual typing for closure parameters,
which is the single biggest ergonomic gap versus TypeScript's
`xs.map(x => ...)`.

## A variadic function will not fit

`fmt.Sprint` looks like it should work as a `func(T) U` and does not.

```go
package main

import "fmt"

func Apply[T, U any](f func(T) U, xs []T) []U {
  out := make([]U, len(xs))
  for i, x := range xs {
    out[i] = f(x)
  }
  return out
}

func main() {
  fmt.Println(Apply(fmt.Sprint, []int{1, 2}))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:14:21: in call to Apply, type func(a ...any) string of fmt.Sprint does not match func(T) U (cannot infer T and U)
```

A variadic signature is not assignable to a one-parameter one. Wrap it:
`func(i int) string { return fmt.Sprint(i) }`.

## Method values infer fine; method expressions do not

```go
package main

import (
  "fmt"
  "strings"
)

func Map[T, U any](xs []T, f func(T) U) []U {
  out := make([]U, len(xs))
  for i, x := range xs {
    out[i] = f(x)
  }
  return out
}

type Prefixer struct{ p string }

func (pr Prefixer) Add(s string) string { return pr.p + s }

func main() {
  pr := Prefixer{">> "}
  fmt.Println(Map([]string{"a", "b"}, pr.Add))
  fmt.Println(Map([]string{" x "}, strings.TrimSpace))
}
```

Output:

```
[>> a >> b]
[x]
```

A *method value* `pr.Add` is already a bound `func(string) string`. A *method
expression* `Prefixer.Add` is not — it takes the receiver as its first
parameter:

```
# m09gen
./main.go:23:33: in call to Map, type func(pr Prefixer, s string) string of Prefixer.Add does not match inferred type func(string) U for func(T) U
```

## Constraint type inference

A constraint with a core type lets the compiler infer a parameter that never
appears in an argument position directly.

```go
package main

import "fmt"

func Rev[S ~[]E, E any](s S) S {
  out := make(S, len(s))
  for i, v := range s {
    out[len(s)-1-i] = v
  }
  return out
}

type IDs []int

func main() {
  r := Rev(IDs{1, 2, 3})
  fmt.Printf("%v %T\n", r, r)
  p := Rev([]string{"a", "b"})
  fmt.Printf("%v %T\n", p, p)
}
```

Output:

```
[3 2 1] main.IDs
[b a] []string
```

`S` comes from the argument; `E` then comes from `S`'s constraint `~[]E`. This
second step is constraint type inference, and it is what makes
`slices.Clone(myNamedSlice)` return the named type rather than a plain slice.

## Generic types infer nothing

There is no inference for a type instantiation, in any Go version. `var s Stack`
is an error (9.3); the zero value and the constructor are your two options.

```go
package main

import "fmt"

type Stack[T any] struct{ items []T }

func NewStack[T any](items ...T) *Stack[T] { return &Stack[T]{items} }

func main() {
  a := NewStack(1, 2, 3)
  fmt.Println(a.items)
  b := NewStack[string]()
  fmt.Println(b.items == nil)
  c := Stack[int]{items: []int{1}}
  fmt.Println(c.items)
}
```

Output:

```
[1 2 3]
true
[1]
```

`NewStack(1, 2, 3)` infers from arguments like any function. `NewStack[string]()`
is the return-only case and must be explicit. A composite literal always names
its instantiation.

## T concrete versus T interface

When a type parameter is constrained by an interface and you pass a value, `T`
becomes the *static* type of the argument. Pass a concrete value and `T` is
concrete; pass an interface variable and `T` is the interface.

```go
package main

import (
  "fmt"
  "reflect"
)

type Shouter interface{ Shout() string }

type Dog struct{ Name string }

func (d Dog) Shout() string { return d.Name + "!" }

func Report[T Shouter](x T) {
  var zero T
  fmt.Printf("T = %v   zero = %#v\n", reflect.TypeOf(&zero).Elem(), zero)
}

func main() {
  Report(Dog{"rex"})
  var s Shouter = Dog{"rex"}
  Report(s)
  Report[Shouter](Dog{"rex"})
}
```

Output:

```
T = main.Dog   zero = main.Dog{Name:""}
T = main.Shouter   zero = <nil>
T = main.Shouter   zero = <nil>
```

This matters for three reasons. First, the zero value differs — `Dog{}` versus
`nil` — which changes what your early-return paths produce. Second, when `T` is
concrete the method call is direct; when `T` is the interface it goes through the
itable. Third, `reflect.TypeOf(&zero).Elem()` is the trick for seeing what the
compiler chose; `%T` on the argument shows the dynamic type and will lie to you
here.

Note `reflect` is needed only to *observe* this. The behaviour applies with or
without it.

## Debugging table

The error texts, verbatim, and what to do.

| Error | Cause | Fix |
|---|---|---|
| `cannot infer T (declared at ...)` | `T` appears only in the return type, or the call has no arguments | instantiate: `F[int](...)` |
| `cannot infer T` on `F(nil)` | `nil` is untyped and has no default | `F[*T](nil)` or `F[error](nil)` |
| `cannot infer U (declared at ...)` after `F[int]` | partial instantiation stopped short | supply `U` too, or make it inferable |
| `syntax error: unexpected comma, expected expression` | tried to skip a parameter: `F[, string]` | no such syntax; write the prefix |
| `mismatched types untyped int and untyped string (cannot infer T)` | two untyped constants of different kinds for one `T` | convert one, or split into two parameters |
| `type int64 of j does not match inferred type int32 for T` | first argument pinned `T`, second disagrees | convert at the call site; no implicit widening |
| `type func(a ...any) string of fmt.Sprint does not match func(T) U` | variadic signature for a fixed-arity parameter | wrap in a closure |
| `type func(pr P, s string) string of P.Add does not match` | method *expression*, not a method value | use `instance.Add` |
| `undefined: x` inside `func(x) string` | closure parameter written without a type | write `func(x int) string` |
| `X does not satisfy C (possibly missing ~ for int in C)` | named type against an exact type set | add `~` to the constraint (9.2) |
| `cannot use generic type S[T any] without instantiation` | bare generic type name | write `S[int]` |
| `cannot use generic function F without instantiation` | generic function used as a value | write `F[int]` |

## When to instantiate explicitly anyway

Inference succeeding does not mean you should rely on it. Write the brackets
when:

- the argument is an untyped constant and `T` is numeric (`Half[float64](7)`)
- the call is in a return statement and the reader cannot see the argument types
- there are three or more type parameters
- you are passing `nil`, an empty literal, or a `make` call
- the inferred type would be a named type and you wanted the underlying one, or
  vice versa

Idiomatic: let inference work for one or two obvious parameters, and write them
out as soon as a reviewer would have to think.

## Quick Reference

| Inference source | Works? |
|---|---|
| ordinary argument's type | yes — the main case |
| untyped constant's default type | yes, but `1` is `int` |
| a function-typed argument's signature | yes, for both `T` and `U` |
| a method value `x.M` | yes |
| a method expression `T.M` | no — receiver is an extra parameter |
| constraint core type (`~[]E`) | yes — constraint type inference |
| the return type | never |
| the assignment target's type | never |
| `nil` | never |
| a generic *type* instantiation | never |

| Form | Effect |
|---|---|
| `F(x)` | infer everything |
| `F[int](x)` | pin `T`, infer the rest |
| `F[int, string](x)` | pin everything |
| `F[int]` | a function value with `T` pinned |

## Common Mistakes

**Expecting the assignment target to drive inference.** Go reads arguments only;
instantiate explicitly for return-only parameters.

**Calling a numeric generic with an integer literal and getting integer
division.** Write `F[float64](7)` or `F(7.0)`.

**Trying to pin only the second type parameter.** You must supply a prefix;
reorder the parameters if callers keep needing the later one.

**Writing `func(x) T` for a callback.** The parameter needs an explicit type; `x`
alone is read as a type name.

**Passing a method expression where a `func(T) U` is wanted.** Use a method value
bound to a receiver.

**Assuming `T` is the concrete type when the caller holds an interface
variable.** `T` becomes the interface, and `var zero T` is then `nil`.

## Exercises

1. Write `func Collect[T any](seq iter.Seq[T]) []T` and call it on
   `slices.Values([]int{1,2,3})`. Then write a `Collect` whose `T` is
   uninferable and capture the error.
2. Write `Clamp[T cmp.Ordered](v, lo, hi T)` and call it as `Clamp(5, 0, 10.0)`.
   Capture the error, then fix it three ways and say which you prefer.
3. Build `Reduce[T, A any]` and call it so that `A` must be pinned explicitly.
   Reorder the parameters so inference can do it instead.
4. Write a generic function with three type parameters where only the middle one
   is inferable. Demonstrate that you cannot pin just the first and third.
5. Reproduce the `T`-concrete-versus-`T`-interface difference with a function
   that returns `var zero T`, and show a caller that gets a nil-pointer panic
   because of it.
6. Take every error row in the debugging table and reproduce it in a single file,
   one commented-out line at a time. Keep the file as a reference.

---

[← 9.3 Generic Types and Methods](03-generic-types.md) · [Course Index](../README.md) · [9.5 When Generics Are the Wrong Tool →](05-when-not-to.md)
