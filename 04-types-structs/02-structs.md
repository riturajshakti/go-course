[← 4.1 Pointers and the Absence of Pointer Arithmetic](01-pointers.md) · [Course Index](../README.md) · [4.3 Struct Comparison, Copying, and Memory Layout →](03-struct-semantics.md)

# 4.2 — Defining and Instantiating Structs

## What You'll Learn

- Five ways to build a struct value, and why the keyed literal is the only one to use
- `struct{}` occupies zero bytes, which is why it is the idiomatic set element
- Struct values stored in a map are not addressable — `m[k].F = v` does not compile
- Anonymous structs earn their keep in table tests and one-off JSON payloads

## Coming From Another Language

| Other | Go |
|---|---|
| TS `interface User { id: number }` | `type User struct { ID int }` — a concrete layout, not a shape |
| Python `@dataclass` | `type` + struct, no decorator, no `__init__` |
| Rust `struct User { id: u32 }` | nearly identical, including `User { id: 1 }` literals |
| JS object literal `{a: 1}` | an anonymous struct, but the field set is fixed at compile time |
| Python `dict` as a record | a struct; a Go `map` has one value type, not per-key types |

A Go struct is a fixed, flat sequence of typed fields. No dynamic attributes, no
`__slots__`, no prototype chain. Every struct type has a usable zero value, and that zero
value is the starting point of most Go API design.

## Declaring a struct type

```go
package main

import "fmt"

type User struct {
  ID        int
  Name      string
  Email     string
  Active    bool
  tokenHash string
}

func main() {
  var zero User
  fmt.Printf("%+v\n", zero)

  positional := User{1, "ada", "ada@example.com", true, "x"}
  keyed := User{ID: 2, Name: "bob"}
  fmt.Printf("%+v\n", positional)
  fmt.Printf("%+v\n", keyed)

  ptr := &User{ID: 3, Name: "cleo"}
  fmt.Printf("%+v %T\n", *ptr, ptr)

  np := new(User)
  np.ID = 4
  fmt.Printf("%+v\n", *np)
}
```

Output:

```
{ID:0 Name: Email: Active:false tokenHash:}
{ID:1 Name:ada Email:ada@example.com Active:true tokenHash:x}
{ID:2 Name:bob Email: Active:false tokenHash:}
{ID:3 Name:cleo Email: Active:false tokenHash:} *main.User
{ID:4 Name: Email: Active:false tokenHash:}
```

Note `%+v` prints field names; plain `%v` prints only the values.

The positional form compiles today and breaks the moment someone inserts a field, or adds
one at the end (it then fails to compile for too few values — the better of the two
outcomes). The keyed form survives both. Idiomatic: always use the keyed literal
`User{ID: 2, Name: "bob"}`, and leave zero-valued fields out entirely.

## Field grouping and visibility

Fields of the same type can share a line, and a capital initial is the only access modifier.

```go
type Rect struct {
  X, Y          int
  Width, Height int

  label string
  dirty bool
}
```

`X` is visible to other packages; `label` is not. There is no `private` keyword and no
field-level control finer than exported/unexported. Unexported fields are also invisible to
`encoding/json` and friends (see 4.7).

## Nested structs and nested literals

```go
package main

import "fmt"

type Address struct {
  City string
  Zip  string
}

type Company struct {
  Name    string
  HQ      Address
  Billing *Address
}

func main() {
  c := Company{
    Name: "Acme",
    HQ:   Address{City: "Pune", Zip: "411001"},
    Billing: &Address{
      City: "Mumbai",
      Zip:  "400001",
    },
  }
  fmt.Printf("%+v %+v\n", c.Name, c.HQ)
  fmt.Println(c.HQ.City, c.Billing.City)

  nested := struct {
    Outer string
    Inner struct {
      N int
    }
  }{Outer: "o"}
  nested.Inner.N = 7
  fmt.Printf("%+v\n", nested)
}
```

Output:

```
Acme {City:Pune Zip:411001}
Pune Mumbai
{Outer:o Inner:{N:7}}
```

`HQ Address` embeds the bytes of an `Address` inside `Company`; `Billing *Address` stores an
8-byte pointer and may be nil. Printing the struct with the pointer field would show a
hex address that changes every run, so the example prints the fields instead.

An inner `Address` field is named `HQ` and must be written `c.HQ.City`. Drop the field name
and you get *embedding*, where `City` is promoted to the outer struct — 4.5.

## Anonymous structs

A struct type with no name, declared where it is used. Two places where this is genuinely
the right tool: table-driven test cases and ad-hoc response payloads.

```go
package main

import "fmt"

func main() {
  cases := []struct {
    name string
    in   int
    want string
  }{
    {name: "zero", in: 0, want: "zero"},
    {name: "pos", in: 3, want: "many"},
  }
  for _, tc := range cases {
    got := "many"
    if tc.in == 0 {
      got = "zero"
    }
    fmt.Printf("%-5s in=%d got=%q ok=%t\n", tc.name, tc.in, got, got == tc.want)
  }

  payload := struct {
    Status string `json:"status"`
    Count  int    `json:"count"`
  }{Status: "ok", Count: 2}
  fmt.Printf("%+v\n", payload)
}
```

Output:

```
zero  in=0 got="zero" ok=true
pos   in=3 got="many" ok=true
{Status:ok Count:2}
```

Inside a `[]struct{...}` literal the element type is already known, so the per-element
`struct{...}{` prefix is omitted — `{name: "zero", ...}` is enough. This is the shape of
every table test in module 14.

Do not reach for anonymous structs to avoid naming a type that appears in a function
signature. If it crosses an API boundary, name it.

## struct{} is zero bytes

```go
package main

import (
  "fmt"
  "unsafe"
)

func main() {
  type nothing struct{}
  var n nothing
  fmt.Println(unsafe.Sizeof(n))

  set := map[string]struct{}{}
  set["go"] = struct{}{}
  set["rust"] = struct{}{}
  _, ok := set["go"]
  _, no := set["zig"]
  fmt.Println(len(set), ok, no)
  fmt.Println(unsafe.Sizeof(set["go"]))
}
```

Output:

```
0
2 true false
0
```

`struct{}{}` is the awkward-looking value of the empty struct type: the first `struct{}` is
the type, the `{}` is the literal. `map[K]struct{}` is Go's set — the values cost nothing,
versus one byte per entry for `map[K]bool`. The tradeoff is readability, so `map[K]bool`
remains common in non-hot code where `set[k]` reads better than `_, ok := set[k]`.

`chan struct{}` is the same idea for signal-only channels (module 10).

## Struct values in a map are not addressable

```go
package main

import "fmt"

type Counter struct {
  N int
}

func main() {
  m := map[string]Counter{"a": {N: 1}}
  m["a"].N = 2
  fmt.Println(m)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:11:3: cannot assign to struct field m["a"].N in map
```

A map may rehash and move its storage, so a map element has no stable address and you
cannot write through one. Three fixes:

```go
package main

import "fmt"

type Counter struct {
  N int
}

func main() {
  m := map[string]Counter{"a": {N: 1}}

  c := m["a"]
  c.N = 2
  m["a"] = c
  fmt.Println(m)

  pm := map[string]*Counter{"a": {N: 1}}
  pm["a"].N = 2
  fmt.Println(*pm["a"])

  vals := []Counter{{N: 1}, {N: 2}}
  vals[0].N = 10
  fmt.Println(vals)

  ptrs := []*Counter{{N: 1}, {N: 2}}
  ptrs[0].N = 10
  fmt.Println(*ptrs[0], *ptrs[1])
}
```

Output:

```
map[a:{2}]
{2}
[{10} {2}]
{10} {2}
```

Read-modify-write (`c := m["a"]`) works and keeps value semantics. `map[K]*T` makes the
fields directly assignable. *Slice* elements, unlike map elements, are addressable, so
`vals[0].N = 10` compiles fine — this asymmetry trips everyone once.

## Slices of structs vs slices of pointers

| | `[]T` | `[]*T` |
|---|---|---|
| Memory | one contiguous block, cache-friendly | a block of pointers plus N allocations |
| `for _, v := range` | `v` is a copy; writes to `v` are lost | `v` is a pointer; writes land |
| `s[i].F = x` | works (slice elements are addressable) | works |
| nil elements | impossible | possible, and a panic waiting to happen |
| GC pressure | low | one object per element to trace |

Idiomatic: `[]T` unless the elements are large, must be shared, or must be nil-able.

## Quick Reference

| Form | Result | Note |
|---|---|---|
| `var u User` | zero value `User` | no allocation ceremony |
| `User{}` | zero value `User` | useful as an argument |
| `User{ID: 1}` | keyed literal | idiomatic; omitted fields are zero |
| `User{1, "a", "", false, ""}` | positional literal | must list every field, in order |
| `&User{ID: 1}` | `*User` | the usual way to get a pointer |
| `new(User)` | `*User`, all zero | no inline field values |
| `struct{ A int }{A: 1}` | anonymous struct value | tests, ad-hoc payloads |
| `struct{}{}` | the zero-size value | set elements, signal channels |

| Expression | Addressable? |
|---|---|
| `v.F` where `v` is a variable | yes |
| `s[i].F` on a slice | yes |
| `a[i].F` on an array variable | yes |
| `m[k].F` on a map | no — compile error |
| `f().F` on a function result | no |
| `p.F` where `p` is `*T` | yes |

## Common Mistakes

**Using a positional literal for a struct you do not own.** Adding a field upstream
silently reassigns your values or breaks the build; always use keyed literals.

**Writing `m[k].Field = v` for a `map[K]struct`.** Map elements are not addressable; use
`map[K]*T`, or read the element, modify it, and assign it back.

**Mutating the loop variable in `for _, v := range structs`.** `v` is a copy; index with
`for i := range` and write `structs[i].F` instead.

**Declaring an anonymous struct for a type that appears in a function signature.** Callers
cannot name it; give it a `type` declaration.

**Assuming a nil `*Address` field is harmless.** It is, until something reads
`c.Billing.City`; check it or store a value field with a useful zero.

## Exercises

1. Declare `type Order struct { ID int; Total float64; Lines []Line }` and build one with a
   keyed literal, a positional literal, and `new`. Then add a field at position 2 and record
   which of the three still compiles.
2. Build `map[string]Counter`, try `m["a"].N++`, capture the exact compiler error, then make
   it work twice: once with read-modify-write and once with `map[string]*Counter`.
3. Implement a set of `rune` with `map[rune]struct{}` and a `Has`, `Add`, `Remove` API.
   Compare `unsafe.Sizeof` of the value types against a `map[rune]bool` version.
4. Write a table test (anonymous struct slice) for a `func classify(n int) string` with
   cases for negative, zero, one, and large inputs, and run it with `go test -v`.
5. Benchmark summing a `Total` field over `[]Order` versus `[]*Order` with 1e6 elements using
   `go test -bench . -benchmem`, and explain the allocation count difference.

---

[← 4.1 Pointers and the Absence of Pointer Arithmetic](01-pointers.md) · [Course Index](../README.md) · [4.3 Struct Comparison, Copying, and Memory Layout →](03-struct-semantics.md)
