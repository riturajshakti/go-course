[← 9.1 Type Parameters on Functions](01-type-parameters.md) · [Course Index](../README.md) · [9.3 Generic Types and Methods →](03-generic-types.md)

# 9.2 — Constraints, comparable, and cmp.Ordered

## What You'll Learn

- A constraint is an interface, extended with *type sets*: `interface{ int | ~string }`
- `~T` matches any named type whose underlying type is `T` — the single most common missing piece
- A type-set interface cannot be used as a value type, only as a constraint
- `comparable` permits `==`, excludes slices/maps/funcs, and admits interfaces that can panic at runtime
- `cmp.Ordered`, `cmp.Compare`, `cmp.Less`, `cmp.Or` are the real stdlib tools

## Coming From Another Language

| Other | Go |
|---|---|
| Rust `T: Add<Output=T>` — operators come from traits | no operator traits; you list the *types* that have `+` |
| Rust `impl Add for MyInt` adds `+` to your type | impossible; `~int` lets your type reuse `int`'s `+` |
| TS `T extends string \| number` | `interface{ ~string \| ~int }` — looks similar, works differently |
| TS `keyof`, mapped types | nothing; constraints cannot compute types |
| Rust `where T: Debug` | `[T fmt.Stringer]` — a method-only constraint is just an interface |
| TS structural typing on `T` | only what the constraint declares is visible on `T` |

This is the deepest difference in the module. Rust gives `+` to your type by
implementing a trait. Go has no operator overloading at all, so the only way to
write `a + b` inside a generic function is to constrain `T` to a *set of types
that already support `+`*. That is what a type set is for, and why `cmp.Ordered`
is a 14-line union rather than a one-method interface.

## Type sets

Any interface may list types as well as methods. An interface that lists types
is a constraint.

```go
package main

import "fmt"

type Integer interface {
  int | int64
}

type Celsius int

func Sum[T Integer](xs []T) T {
  var total T
  for _, x := range xs {
    total += x
  }
  return total
}

func main() {
  fmt.Println(Sum([]Celsius{1, 2, 3}))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:20:18: Celsius does not satisfy Integer (possibly missing ~ for int in Integer)
```

The compiler tells you the fix in the error. `Celsius` is not `int` — it is a
distinct named type whose *underlying* type is `int`. `int | int64` means
exactly those two types.

## The ~ approximation token

`~int` means "any type whose underlying type is `int`", which includes `int`
itself.

```go
package main

import "fmt"

type Integer interface {
  ~int | ~int64
}

type Celsius int

func Sum[T Integer](xs []T) T {
  var total T
  for _, x := range xs {
    total += x
  }
  return total
}

func main() {
  fmt.Println(Sum([]Celsius{1, 2, 3}))
  fmt.Printf("%T\n", Sum([]Celsius{1}))
  fmt.Println(Sum([]int{10, 20}))
}
```

Output:

```
6
main.Celsius
30
```

Note the second line: `T` is `Celsius`, not `int`. The type parameter takes the
named type, so the return value keeps it. Idiomatic: write `~` on every element
of a type set unless you have a specific reason to reject named types. Every
constraint in the standard library does.

## A constraint is not a type

A type-set interface can appear only in brackets. Declare a variable of one and
you get a distinct error that tells you a lot about how constraints are
implemented.

```go
package main

import "fmt"

type Number interface {
  ~int | ~float64
}

func main() {
  var n Number = 3
  fmt.Println(n)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:9: cannot use type Number outside a type constraint: interface contains type constraints
```

A method-only interface stays a usable value type; adding a single type-set
element to it takes that away. So `interface{ String() string }` can be both a
constraint and a parameter type, but `interface{ ~int; String() string }` can
only be a constraint.

## Composing constraints

Constraints embed other constraints, and unions distribute over them. This is
the standard numeric hierarchy, which you will end up writing in most projects
that do generic arithmetic.

```go
package main

import (
  "cmp"
  "fmt"
)

type Signed interface {
  ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Unsigned interface {
  ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

type Integer interface {
  Signed | Unsigned
}

type Float interface {
  ~float32 | ~float64
}

type Number interface {
  Integer | Float
}

type Stringish interface {
  ~string | ~[]byte
}

type NamedNumber interface {
  Number
  String() string
}

type Money int64

func (m Money) String() string { return fmt.Sprintf("$%d.%02d", m/100, m%100) }

func Sum[T Number](xs []T) T {
  var total T
  for _, x := range xs {
    total += x
  }
  return total
}

func Join[T Stringish](xs []T) string {
  var out string
  for _, x := range xs {
    out += string(x)
  }
  return out
}

func Report[T NamedNumber](xs []T) string {
  return Sum(xs).String()
}

func MinOf[T cmp.Ordered](xs []T) (T, bool) {
  if len(xs) == 0 {
    var zero T
    return zero, false
  }
  m := xs[0]
  for _, x := range xs[1:] {
    m = min(m, x)
  }
  return m, true
}

func main() {
  fmt.Println(Sum([]int{1, 2, 3}), Sum([]float64{1.5, 2.5}), Sum([]uint8{200, 100}))
  fmt.Println(Join([]string{"a", "b"}), Join([][]byte{[]byte("x"), []byte("y")}))
  fmt.Println(Report([]Money{1250, 399}))
  fmt.Println(MinOf([]string{"go", "c", "rust"}))
  fmt.Println(MinOf([]int{}))
}
```

Output:

```
6 4 44
ab xy
$16.49
c true
0 false
```

Two things in that first line. `Sum([]float64{1.5, 2.5})` printed `4`, not `4.0`
— `%v` formatting, not a bug. `Sum([]uint8{200, 100})` printed `44`: generic
arithmetic wraps exactly as concrete arithmetic does, because `T` really is
`uint8` inside the loop. Generics give you no protection from overflow.

`Report` shows a constraint with both a type set and a method. It compiles and
works, but note how narrow it is: `T` must be a numeric type *and* have a
`String` method, which in practice means you wrote the type yourself. A
constraint combining both is almost always over-engineered.

## comparable

`comparable` is predeclared. It permits `==` and `!=` and nothing else — no
ordering.

```go
package main

import "fmt"

func Index[T comparable](xs []T, want T) int {
  for i, x := range xs {
    if x == want {
      return i
    }
  }
  return -1
}

func main() {
  fmt.Println(Index([][]int{{1}, {2}}, []int{2}))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:15:20: []int does not satisfy comparable
```

Slices, maps, and functions are not comparable, so they cannot instantiate
`comparable`. Everything else can: all numeric types, strings, booleans,
pointers, channels, arrays of comparable types, and structs whose every field is
comparable.

## comparable admits interfaces, which can panic

Since Go 1.20 an interface type satisfies `comparable` — but comparing two
interface values holding uncomparable dynamic types panics at runtime.

```go
package main

import "fmt"

func Index[T comparable](xs []T, want T) int {
  for i, x := range xs {
    if x == want {
      return i
    }
  }
  return -1
}

func main() {
  ok := []any{1, "a", 2.5}
  fmt.Println(Index(ok, "a"))

  bad := []any{[]int{1, 2}}
  fmt.Println(Index(bad, any([]int{1, 2})))
}
```

This program panics on purpose. Output:

```
1
panic: runtime error: comparing uncomparable type []int

goroutine 1 [running]:
main.Index[...](...)
	/tmp/scratch/main.go:7
main.main()
	/tmp/scratch/main.go:19 +0x164
exit status 2
```

So `comparable` is a compile-time guarantee only for non-interface types. The
moment `T` is `any` or any other interface, the guarantee degrades to a runtime
check. This is the one hole in Go's comparability story, and it exists because
`map[any]V` has always been legal with the same panic.

## cmp.Ordered and the cmp package

`cmp.Ordered` is the stdlib constraint for `<`, `<=`, `>`, `>=`:

```go
type Ordered interface {
  ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
    ~float32 | ~float64 |
    ~string
}
```

That is the whole thing — a hand-written union, because Go has no `PartialOrd`
trait to appeal to. The package also ships three functions.

```go
package main

import (
  "cmp"
  "fmt"
  "math"
  "slices"
)

func Max2[T cmp.Ordered](a, b T) T {
  if a > b {
    return a
  }
  return b
}

func main() {
  fmt.Println(cmp.Compare(3, 7), cmp.Compare(7, 3), cmp.Compare(3, 3))
  fmt.Println(cmp.Compare("abc", "abd"))
  fmt.Println(cmp.Less(1.5, 2.0), cmp.Less("b", "a"))
  fmt.Println(cmp.Or("", "", "fallback"))
  fmt.Println(cmp.Or(0, 42, 7))
  fmt.Println(Max2(3, 9), Max2("go", "rust"))

  nan := math.NaN()
  fmt.Println(cmp.Compare(1.0, nan), cmp.Less(nan, 1.0), nan < 1.0)

  xs := []int{3, 1, 2}
  slices.SortFunc(xs, cmp.Compare)
  fmt.Println(xs)
}
```

Output:

```
-1 1 0
-1
true false
fallback
42
9 rust
1 true false
[1 2 3]
```

`cmp.Or` returns the first non-zero argument — the config-defaulting helper, and
note it is constrained by `comparable`, not `Ordered`, since it only needs `==`.

The NaN line is the reason `cmp.Compare` exists. Raw `nan < 1.0` is `false` and
so is `nan > 1.0`, which breaks sort invariants. `cmp.Compare` defines NaN as
less than everything, so `cmp.Less(nan, 1.0)` is `true` while the plain operator
says `false`. Idiomatic: pass `cmp.Compare` to `slices.SortFunc` rather than
writing your own comparator — it is already the right signature.

`golang.org/x/exp/constraints` predates `cmp` and still exists for `Signed`,
`Unsigned`, `Integer`, `Float`, and `Complex`; use `cmp.Ordered` from the
standard library for ordering and copy the numeric unions into your own package
if you need them.

## Operators allowed per constraint

The constraint decides which operators the compiler will accept on a `T`. There
is no partial credit.

| Constraint | Allowed on a `T` |
|---|---|
| `any` | `=`, `==` against nothing; pass, store, return only |
| `comparable` | `==`, `!=` |
| `cmp.Ordered` | `==`, `!=`, `<`, `<=`, `>`, `>=`, `+` (all its members support it) |
| `~int \| ~float64` | all arithmetic, all comparison |
| `~int \| ~string` | `+`, all comparison — but not `-`, `*`, `/` |
| `~[]byte \| ~string` | `len`, indexing, slicing, `range` |
| `interface{ String() string }` | `.String()` only; no `==` |

The `~int | ~string` row is the interesting one: `+` is legal because both
members have a `+`, while `-` is not, because strings do not. The compiler
checks each operator against every member of the type set.

## Constraint type inference

When a constraint has a *core type* like `~[]E`, the compiler can infer `E` from
`S`. This is how every function in `slices` is declared.

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

`Rev` returns `S`, so a named slice type survives the round trip — `IDs` in,
`IDs` out. Declaring it as `func Rev[E any](s []E) []E` would have returned a
plain `[]int`. That is precisely why `slices.Clone` is `[S ~[]E, E any]`.

## Writing your own constraints

```go
package main

import "fmt"

type Number interface {
  ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64
}

type Keyed[K comparable] interface {
  Key() K
}

type User struct {
  ID   int
  Name string
}

func (u User) Key() int { return u.ID }

func Average[T Number](xs []T) float64 {
  if len(xs) == 0 {
    return 0
  }
  var total T
  for _, x := range xs {
    total += x
  }
  return float64(total) / float64(len(xs))
}

func ByKey[K comparable, T Keyed[K]](xs []T) map[K]T {
  m := make(map[K]T, len(xs))
  for _, x := range xs {
    m[x.Key()] = x
  }
  return m
}

func main() {
  fmt.Println(Average([]int{1, 2, 3, 4}))
  fmt.Println(Average([]float32{1.5, 2.5}))
  m := ByKey[int]([]User{{1, "ada"}, {2, "bob"}})
  fmt.Println(m[2].Name, len(m))
}
```

Output:

```
2.5
2
bob 2
```

`Keyed[K]` is a generic constraint — a constraint parameterised by a type
itself. `ByKey[int]` supplies `K` and lets `T` be inferred from the argument,
which is the partial-instantiation form 9.4 covers.

A constraint with one method and no type set adds nothing a plain interface
parameter does not already give you, and costs you a type parameter at every
call site. 9.5 makes that case with benchmarks.

## Quick Reference

| Constraint | Definition | Use |
|---|---|---|
| `any` | `interface{}` | no operations needed |
| `comparable` | predeclared | map keys, `==`, dedup, sets |
| `cmp.Ordered` | the union above | sorting, `min`, `max`, ranges |
| `interface{ ~int \| ~float64 }` | type set | arithmetic |
| `interface{ ~string \| ~[]byte }` | type set | text that may be either |
| `interface{ ~[]E }` | core type | slice utilities; enables inference of `E` |
| `interface{ fmt.Stringer }` | method only | usable as a value type too |
| `interface{ C1 \| C2 }` | embedded union | composing numeric hierarchies |
| `interface{ C; M() }` | embedded + method | rarely what you want |

| Token | Meaning |
|---|---|
| `\|` | union: `T` may be any listed type |
| `~T` | any type with underlying type `T` |
| no `~` | exactly that named type |
| embedding | the embedded constraint's type set intersects in |

## Common Mistakes

**Omitting `~` and then hitting "does not satisfy".** Write `~int`, not `int`,
unless you deliberately want to reject named types.

**Declaring a variable of a type-set interface.** It is a constraint only; use a
method-only interface when you need a value type.

**Expecting `comparable` to allow `<`.** Use `cmp.Ordered` for ordering;
`comparable` is `==` and `!=` only.

**Instantiating `comparable` with `any` and assuming it is safe.** The comparison
compiles but panics on an uncomparable dynamic type.

**Sorting floats with a hand-written `a < b` comparator.** NaN breaks the
invariant; pass `cmp.Compare` instead.

**Reaching for `golang.org/x/exp/constraints`.** `cmp.Ordered` is in the standard
library; copy the numeric unions locally instead of adding the dependency.

## Exercises

1. Define `Clamp[T cmp.Ordered](v, lo, hi T) T`. Verify it on `int`, `string`,
   and a named `type Score int`. Then call it with `lo > hi` and decide what it
   should do.
2. Write a constraint that admits `~string` and `~[]byte` and a `Length` function
   over it. Add `~[]rune` and explain why `len` still compiles.
3. Try to write `Sum[T comparable]`. Capture the compile error for `+`, then
   widen the constraint minimally until it compiles.
4. Build `Dedupe[T comparable](xs []T) []T` with a `map[T]struct{}`. Instantiate
   it with `[]any` containing a map value and capture the runtime panic.
5. Write `func Keys[M ~map[K]V, K comparable, V any](m M) []K` so a named map
   type is accepted. Then remove the `~` and capture the error.
6. Build a `Stat[T Number]` returning min, max, and mean. Instantiate with
   `uint8` and values summing past 255; show the wraparound and fix it by
   widening the accumulator.

---

[← 9.1 Type Parameters on Functions](01-type-parameters.md) · [Course Index](../README.md) · [9.3 Generic Types and Methods →](03-generic-types.md)
