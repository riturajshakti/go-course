[← 4.2 Defining and Instantiating Structs](02-structs.md) · [Course Index](../README.md) · [4.4 Methods: Value vs Pointer Receivers →](04-methods.md)

# 4.3 — Struct Comparison, Copying, and Memory Layout

## What You'll Learn

- `==` works on a struct exactly when every field is comparable, and compares field-wise
- Assignment copies the whole struct, but a slice/map field copies only its header — the two copies still alias
- Field *order* changes `unsafe.Sizeof` because of alignment padding: 40 bytes versus 24 for the same fields
- `reflect.DeepEqual` is the fallback for non-comparable structs, with caveats

All `Sizeof`/`Alignof`/`Offsetof` numbers below are from darwin/arm64 (64-bit, 8-byte
pointer, 8-byte max alignment). They differ on 32-bit targets.

## Coming From Another Language

| Other | Go |
|---|---|
| Python `==` on a dataclass compares fields | same, and it is built in, not generated |
| Python `is` vs `==` | `p1 == p2` on pointers is `is`; on values it is `==` |
| JS `{a:1} === {a:1}` is `false` | `Point{1} == Point{1}` is `true` |
| Rust `#[derive(PartialEq, Clone)]` | comparison and copying are automatic, no derive |
| Rust move semantics | assignment always copies; the original stays valid |

## == is field-wise and built in

```go
package main

import "fmt"

type Point struct {
  X, Y int
}

type Line struct {
  A, B Point
  Name string
}

func main() {
  p1 := Point{1, 2}
  p2 := Point{1, 2}
  p3 := Point{1, 3}
  fmt.Println(p1 == p2, p1 == p3)

  l1 := Line{Point{0, 0}, Point{1, 1}, "diag"}
  l2 := Line{Point{0, 0}, Point{1, 1}, "diag"}
  fmt.Println(l1 == l2)

  seen := map[Point]string{}
  seen[Point{1, 2}] = "first"
  seen[Point{1, 2}] = "second"
  fmt.Println(len(seen), seen[Point{1, 2}])

  var zero Point
  fmt.Println(zero == Point{})
}
```

Output:

```
true false
true
1 second
true
```

Comparison recurses into nested struct fields and ignores padding bytes. Because a
comparable struct is also hashable, it can be a map key — the two `Point{1, 2}` writes
addressed the same entry. `v == T{}` is the standard "is this the zero value" test.

## Comparability is a property of the field types

```go
package main

import "fmt"

type Bag struct {
  Items []string
}

func main() {
  a := Bag{Items: []string{"x"}}
  b := Bag{Items: []string{"x"}}
  fmt.Println(a == b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:12:15: invalid operation: a == b (struct containing []string cannot be compared)
```

| Field type | Comparable? |
|---|---|
| numeric, `string`, `bool`, pointer, channel | yes |
| array of a comparable type | yes |
| struct of comparable fields | yes |
| interface | yes at compile time; panics at run time if the dynamic type is not comparable |
| slice, map, func | **no** |

One slice field anywhere in the tree makes the whole struct non-comparable, and therefore
unusable as a map key. That constraint is a real design input: if a type needs to be a key,
keep an array or a string where you were tempted to put a slice.

## reflect.DeepEqual for the non-comparable case

```go
package main

import (
  "fmt"
  "reflect"
)

type Bag struct {
  Items []string
  Meta  map[string]int
}

func main() {
  a := Bag{Items: []string{"x"}, Meta: map[string]int{"n": 1}}
  b := Bag{Items: []string{"x"}, Meta: map[string]int{"n": 1}}
  fmt.Println(reflect.DeepEqual(a, b))

  var n1, n2 []string
  fmt.Println(reflect.DeepEqual(n1, n2), reflect.DeepEqual(n1, []string{}))
}
```

Output:

```
true
true false
```

Caveats worth knowing before you use it: it is reflection-slow, it distinguishes a nil
slice from an empty one (last line), it compares unexported fields too, and it reports
false for two functionally identical `func` fields. In tests, `github.com/google/go-cmp`
is the better tool; in production code, write an `Equal` method.

## Assignment copies the struct — but only one level

```go
package main

import "fmt"

type Bag struct {
  Label string
  Items []string
}

func main() {
  a := Bag{Label: "a", Items: []string{"x", "y"}}
  b := a

  b.Label = "b"
  fmt.Println(a.Label, b.Label)

  b.Items[0] = "CHANGED"
  fmt.Println(a.Items, b.Items)

  b.Items = append(b.Items, "z")
  fmt.Println(len(a.Items), len(b.Items))
}
```

Output:

```
a b
[CHANGED y] [CHANGED y]
2 3
```

This is the most important line in the lesson. `b := a` copied all of `Bag`: the string
header and the 24-byte slice header. The `Label` change is isolated. But the copied slice
header points at the *same* backing array, so `b.Items[0] = "CHANGED"` is visible through
`a`. There is no deep copy in Go — struct assignment is a shallow `memmove`. The `append`
then reallocated `b.Items` (cap was 2), leaving `a` at length 2.

The same applies to `map` and pointer fields: the copy shares the pointee. If you need an
independent value, write a `Clone` method that copies the slices and maps explicitly (see
7.4 for the full aliasing story).

## Copies in function calls and range loops

```go
package main

import "fmt"

type Big struct {
  Label string
  N     int
}

func mutate(b Big) {
  b.N = 999
}

func main() {
  items := []Big{{"a", 1}, {"b", 2}}

  for _, it := range items {
    it.N = 100
  }
  fmt.Println(items)

  for i := range items {
    items[i].N = 100
  }
  fmt.Println(items)

  mutate(items[0])
  fmt.Println(items[0])
}
```

Output:

```
[{a 1} {b 2}]
[{a 100} {b 100}]
{a 100}
```

`range` assigns a fresh copy to `it` each iteration, so the first loop changed nothing.
`for i := range` and indexing is the fix. Same story for the function call: `mutate` got a
copy.

Go 1.22 changed loop variables to be per-iteration, which fixes the closure-capture bug —
but `it` was always a copy of the element, and that has not changed.

## Memory layout: Sizeof, Alignof, Offsetof

Field order matters, because each field must start at an offset that is a multiple of its
alignment, and the struct's total size is rounded up to its own alignment.

```go
package main

import (
  "fmt"
  "unsafe"
)

type Bad struct {
  A bool
  B int64
  C bool
  D int64
  E bool
}

type Good struct {
  B int64
  D int64
  A bool
  C bool
  E bool
}

func main() {
  fmt.Println(unsafe.Sizeof(Bad{}), unsafe.Alignof(Bad{}))
  fmt.Println(unsafe.Sizeof(Good{}), unsafe.Alignof(Good{}))

  fmt.Println(unsafe.Offsetof(Bad{}.A), unsafe.Offsetof(Bad{}.B),
    unsafe.Offsetof(Bad{}.C), unsafe.Offsetof(Bad{}.D), unsafe.Offsetof(Bad{}.E))
  fmt.Println(unsafe.Offsetof(Good{}.B), unsafe.Offsetof(Good{}.D),
    unsafe.Offsetof(Good{}.A), unsafe.Offsetof(Good{}.C), unsafe.Offsetof(Good{}.E))
}
```

Output:

```
40 8
24 8
0 8 16 24 32
0 8 16 17 18
```

Same five fields, 40 bytes versus 24. In `Bad`, each `bool` sits at an 8-aligned offset and
is followed by 7 wasted bytes. In `Good`, the two `int64`s come first and the three `bool`s
pack into offsets 16, 17, 18, with 5 bytes of tail padding to reach a multiple of 8.

Idiomatic: order fields largest-alignment first when a type is allocated in bulk (millions
of rows, a hot cache), and ignore layout entirely otherwise. Readability usually wins.

The `fieldalignment` analyzer from `golang.org/x/tools/go/analysis/passes/fieldalignment`
(also available through `golangci-lint`) reports structs it can shrink — see 15.3.

## Header sizes and the zero-size struct

```go
package main

import (
  "fmt"
  "unsafe"
)

type Tagged struct {
  A int64 `json:"a" db:"col_a"`
  B bool  `json:"b"`
}

type Untagged struct {
  A int64
  B bool
}

type Empty struct{}

func main() {
  fmt.Println(unsafe.Sizeof(Tagged{}), unsafe.Sizeof(Untagged{}))
  fmt.Println(unsafe.Sizeof(Empty{}), unsafe.Alignof(Empty{}))
  fmt.Println(unsafe.Sizeof([1000]Empty{}))

  type Hdr struct {
    S  string
    Sl []int
    M  map[string]int
    P  *int
    I  any
  }
  fmt.Println(unsafe.Sizeof(Hdr{}))
  fmt.Println(unsafe.Offsetof(Hdr{}.S), unsafe.Offsetof(Hdr{}.Sl),
    unsafe.Offsetof(Hdr{}.M), unsafe.Offsetof(Hdr{}.P), unsafe.Offsetof(Hdr{}.I))
}
```

Output:

```
16 16
0 1
0
72
0 16 40 48 56
```

Struct tags are compile-time metadata and cost zero bytes — `Tagged` and `Untagged` are the
same size. `struct{}` is 0 bytes with alignment 1, and an array of a thousand of them is
still 0. From the offsets you can read the header sizes directly: `string` 16, `[]int` 24,
`map` 8, `*int` 8, `any` 16.

`unsafe.Sizeof` measures the struct itself, never what its pointers reach. A `Hdr` holding a
million-element slice still reports 72.

## Pointer equality is not value equality

```go
package main

import "fmt"

type Point struct {
  X, Y int
}

func main() {
  a := &Point{1, 2}
  b := &Point{1, 2}
  fmt.Println(a == b, *a == *b)

  c := a
  fmt.Println(a == c)
}
```

Output:

```
false true
true
```

This bites when a struct has a `*T` field: `==` on the outer struct compares the pointer,
so two structs with distinct-but-identical pointees compare unequal.

## Quick Reference

| Operation | Works when | Note |
|---|---|---|
| `a == b` | all fields comparable | field-wise, recurses, ignores padding |
| `a != b` | same | |
| `v == T{}` | T comparable | the zero-value check |
| `map[T]V` | T comparable | compile error otherwise |
| `b := a` | always | shallow: shares slice/map/pointer targets |
| `reflect.DeepEqual(a, b)` | always | slow; nil slice != empty slice |

| Function | Returns | Note |
|---|---|---|
| `unsafe.Sizeof(v)` | `uintptr` bytes of the value | compile-time constant; excludes pointees |
| `unsafe.Alignof(v)` | required alignment | max over fields |
| `unsafe.Offsetof(s.F)` | byte offset of `F` in `s` | needs a selector, not a type |

| Type | Size on darwin/arm64 |
|---|---|
| `bool`, `int8` | 1 |
| `int`, `int64`, `*T`, `map`, `chan`, `func` | 8 |
| `string`, `any` (interface) | 16 |
| `[]T` | 24 |
| `struct{}` | 0 |

## Common Mistakes

**Treating `b := a` as a deep copy.** Slice, map, and pointer fields still alias; write an
explicit `Clone` when independence matters.

**Adding a `[]T` field to a type used as a map key.** That makes the struct
non-comparable and the map declaration a compile error; use an array or a string instead.

**Using `reflect.DeepEqual` in production hot paths.** It is reflection-slow and treats nil
and empty slices as different; write an `Equal` method.

**Mutating `it` in `for _, it := range structs`.** It is a per-iteration copy; use
`for i := range` and `structs[i]`.

**Reordering struct fields for size before measuring.** The win only matters in bulk
allocation; run `fieldalignment` and check `Sizeof` rather than guessing.

## Exercises

1. Write `type Key struct { Tenant string; Shard int }` and use it as a map key over 1000
   inserts. Then change `Shard int` to `Shards []int` and capture the exact compiler error.
2. Take a struct with a `[]byte` field, copy it, mutate the copy's bytes, and print both.
   Then add a `Clone()` that fixes it and prove the fix.
3. Design a struct with `bool, string, int32, bool, float64, int8` in that order. Print its
   `Sizeof`, then reorder for minimum size and print again. Explain each offset.
4. Compare `a == b` and `reflect.DeepEqual(a, b)` on a comparable struct with
   `go test -bench . -benchmem` and report the ratio.
5. Build a struct with an `any` field holding a `[]int`, compare two such values with `==`,
   and capture the runtime panic. Explain why this one is not a compile error.

---

[← 4.2 Defining and Instantiating Structs](02-structs.md) · [Course Index](../README.md) · [4.4 Methods: Value vs Pointer Receivers →](04-methods.md)
