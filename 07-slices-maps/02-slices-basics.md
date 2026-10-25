[← 7.1 Arrays: Fixed Size, Value Semantics](01-arrays.md) · [Course Index](../README.md) · [7.3 append Semantics and Reallocation →](03-append.md)

# 7.2 — Slices: len, cap, and the Slice Header

## What You'll Learn

- A slice is a three-word header: pointer, len, cap — 24 bytes on 64-bit, whatever the length
- `nil` slice vs empty slice: almost always interchangeable, except for `== nil` and JSON
- `cap` after slicing from the middle runs to the end of the backing array, not to `hi`
- Slices are not comparable; only `== nil` is legal

## Coming From Another Language

| Other | Go |
|---|---|
| JS `Array` — one object, grows itself | `[]T` — a header value; `append` returns a *new* header |
| Python `list` — a reference you mutate | `[]T` is a value; `s = append(s, x)` is required |
| Python `xs[1:3]` — a copy | `s[1:3]` — a view into the same memory |
| Rust `Vec<T>` | closest to `[]T`, but Go has no distinct owned/borrowed split |
| Rust `&[T]` | also `[]T` — Go uses one type for both roles |
| `arr.length` is a property | `len(s)` is a builtin, and there is also `cap(s)` |

## The three-word header

```go
package main

import (
  "fmt"
  "unsafe"
)

func main() {
  s := []int{1, 2, 3}
  fmt.Println(unsafe.Sizeof(s), unsafe.Sizeof(uintptr(0)))
  fmt.Printf("%p %d %d\n", s, len(s), cap(s))

  type header struct {
    ptr      unsafe.Pointer
    len, cap int
  }
  h := *(*header)(unsafe.Pointer(&s))
  fmt.Println(h.ptr != nil, h.len, h.cap)
}
```

Output (the address varies per run):

```
24 8
0x531a69892018 3 3
true 3 3
```

Three 8-byte words. `%p` on a slice prints the backing-array pointer — the first
word. That is the entire slice value: copying a slice copies 24 bytes and shares
the array. (`reflect.SliceHeader` describes the same layout and is deprecated in
favour of `unsafe.Slice` / `unsafe.SliceData`; the hand-rolled struct above is just
to make the three words visible.)

## nil slice vs empty slice

```go
package main

import (
  "encoding/json"
  "fmt"
)

func main() {
  var a []int
  b := []int{}
  c := make([]int, 0)

  fmt.Println(a == nil, b == nil, c == nil)
  fmt.Println(len(a), cap(a), len(b), cap(b), len(c), cap(c))

  a = append(a, 1)
  fmt.Println(a, len(a), cap(a))

  var z []int
  for range z {
    fmt.Println("never")
  }
  fmt.Println(len(z), z == nil)

  type resp struct {
    Items []int `json:"items"`
  }
  j1, _ := json.Marshal(resp{nil})
  j2, _ := json.Marshal(resp{[]int{}})
  fmt.Println(string(j1))
  fmt.Println(string(j2))
  fmt.Println(a[0], z)
}
```

Output:

```
true false false
0 0 0 0 0 0
[1] 1 1
0 true
{"items":null}
{"items":[]}
1 []
```

A nil slice has a nil pointer and zero len and cap. `len`, `cap`, `range`, and
`append` all work on it, and `fmt` prints it as `[]`. So you almost never need
`if s == nil`. Idiomatic: declare `var s []T` and append; do not reach for
`[]T{}`.

The one place it bites is JSON. A nil slice marshals to `null`, an empty slice to
`[]`. API clients that do `data.items.map(...)` break on `null`. If a field must
always be an array, initialise it with `[]T{}` or `make([]T, 0)` before marshalling.

## make

```go
package main

import "fmt"

func main() {
  a := make([]int, 3)
  b := make([]int, 3, 10)
  c := make([]string, 2, 2)

  fmt.Println(a, len(a), cap(a))
  fmt.Println(b, len(b), cap(b))
  fmt.Printf("%q %d %d\n", c, len(c), cap(c))

  d := []int{1, 2, 3}
  e := []int{5: 9}
  f := []int{0: 1, 3: 4}
  fmt.Println(d, len(d), cap(d))
  fmt.Println(e, len(e), cap(e))
  fmt.Println(f, len(f), cap(f))
}
```

Output:

```
[0 0 0] 3 3
[0 0 0] 3 10
["" ""] 2 2
[1 2 3] 3 3
[0 0 0 0 0 9] 6 6
[1 0 0 4] 4 4
```

`make([]T, len)` gives `cap == len` and the elements are zeroed and *readable*.
`make([]T, len, cap)` reserves extra room — the elements past `len` exist but are
unreachable without re-slicing. Indexed literals work on slices too, and as with
arrays the highest index sets the length.

`cap < len` is rejected. With constants, at compile time:

```go
package main

import "fmt"

func main() {
  s := make([]int, 3, 1)
  fmt.Println(s)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:20: invalid argument: length and capacity swapped
```

With a variable, at runtime:

```go
package main

import "fmt"

func main() {
  n := 3
  s := make([]int, n, 1)
  fmt.Println(s)
}
```

This program panics on purpose. Output:

```
panic: runtime error: makeslice: cap out of range

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:7 +0x2c
exit status 2
```

A negative length gives `makeslice: len out of range`.

## Slicing: what happens to len and cap

The key asymmetry: `len` is `hi - lo`, but `cap` is `cap(source) - lo`. Slicing from
the middle loses capacity at the front and keeps all of it at the back.

```go
package main

import "fmt"

func main() {
  a := [6]int{0, 1, 2, 3, 4, 5}
  s := a[:]
  fmt.Println(s, len(s), cap(s))

  m := s[2:4]
  fmt.Println(m, len(m), cap(m))

  t := s[2:]
  fmt.Println(t, len(t), cap(t))

  u := s[:3]
  fmt.Println(u, len(u), cap(u))

  v := s[len(s):]
  fmt.Println(v, len(v), cap(v), v == nil)

  str := "hello world"
  sub := str[6:]
  fmt.Println(sub, len(sub))
  bs := []byte(str)[6:]
  fmt.Println(string(bs), len(bs), cap(bs))
}
```

Output:

```
[0 1 2 3 4 5] 6 6
[2 3] 2 4
[2 3 4 5] 4 4
[0 1 2] 3 6
[] 0 0 false
world 5
world 5 5
```

`s[2:4]` has len 2 but cap 4 — it can grow over `s[4]` and `s[5]`. `s[:3]` keeps the
full cap 6. `s[len(s):]` is a legal empty slice that is *not* nil. Slicing a string
yields a string; slicing an array yields a slice over that array.

The hi bound may go up to `cap`, not just `len`: `m[:4]` is legal and exposes
`[2 3 4 5]`. That is how you reach elements between `len` and `cap`. See
[7.4](04-shared-backing.md) for why this matters.

## Slices are not comparable

```go
package main

import "fmt"

func main() {
  a := []int{1, 2, 3}
  b := []int{1, 2, 3}
  fmt.Println(a == b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: a == b (slice can only be compared to nil)
```

Use `slices.Equal`, which treats nil and empty as equal.

```go
package main

import (
  "fmt"
  "slices"
)

func main() {
  a := []int{1, 2, 3}
  var n []int
  fmt.Println(a == nil, n == nil)
  fmt.Println(slices.Equal(a, []int{1, 2, 3}))
  fmt.Println(slices.Equal(n, []int{}))
}
```

Output:

```
false true
true
true
```

Because slices are not comparable, neither is any struct containing one, and
neither can a slice be a map key.

## Passing a slice to a function

The header is copied; the elements are shared. A callee can change your elements
but not your length.

```go
package main

import "fmt"

type user struct{ name string }

func renameVals(us []user) {
  for _, u := range us {
    u.name = "x"
  }
}

func renameIdx(us []user) {
  for i := range us {
    us[i].name = "x"
  }
}

func renamePtrs(us []*user) {
  for _, u := range us {
    u.name = "x"
  }
}

func appendTo(p *[]user) {
  *p = append(*p, user{"new"})
}

func main() {
  a := []user{{"a"}, {"b"}}
  renameVals(a)
  fmt.Println(a)
  renameIdx(a)
  fmt.Println(a)

  b := []*user{{"a"}, {"b"}}
  renamePtrs(b)
  fmt.Println(*b[0], *b[1])

  appendTo(&a)
  fmt.Println(a, len(a))
}
```

Output:

```
[{a} {b}]
[{x} {x}]
{x} {x}
[{x} {x} {new}] 3
```

`renameVals` mutated loop copies — a no-op. `renameIdx` wrote through the index and
stuck. With `[]*user` the range variable is a pointer, so `u.name = "x"` works. And
because `append` may change len or reallocate, a function that appends must either
return the slice or take `*[]T`. The return form is idiomatic; `*[]T` appears mainly
in decoders. The aliasing consequences are [7.4](04-shared-backing.md).

## 2D slices are not contiguous

A slice of slices is independent allocations, so rows can differ in length and can
grow independently — neither of which an array of arrays allows.

```go
package main

import "fmt"

func main() {
  var arr [3][4]int
  flat := arr[0][:]
  fmt.Println(len(flat), cap(flat))

  rows := make([][]int, 3)
  for i := range rows {
    rows[i] = make([]int, 4)
  }
  fmt.Println(len(rows[0]), cap(rows[0]))

  rows[1] = append(rows[1], 99)
  fmt.Println(rows, len(rows[1]))
}
```

Output:

```
4 4
4 4
[[0 0 0 0] [0 0 0 0 99] [0 0 0 0]] 5
```

Note `cap(arr[0][:])` is 4, not 12: slicing a row of `[3][4]int` gives a slice over
*that row's* array type, so it cannot run into row 1.

For a rectangular grid you have three choices.

```go
package main

import "fmt"

func main() {
  rows, cols := 3, 4

  jagged := [][]int{{1}, {1, 2}, {1, 2, 3}}
  fmt.Println(jagged, len(jagged))

  g := make([][]int, rows)
  for i := range g {
    g[i] = make([]int, cols)
  }
  g[1][2] = 7
  fmt.Println(g)

  flat := make([]int, rows*cols)
  g2 := make([][]int, rows)
  for i := range g2 {
    g2[i] = flat[i*cols : (i+1)*cols : (i+1)*cols]
  }
  g2[1][2] = 7
  fmt.Println(g2)
  fmt.Println(flat)
}
```

Output:

```
[[1] [1 2] [1 2 3]] 3
[[0 0 0 0] [0 0 7 0] [0 0 0 0]]
[[0 0 0 0] [0 0 7 0] [0 0 0 0]]
[0 0 0 0 0 0 7 0 0 0 0 0]
```

The third form allocates once and carves rows out of it, with `[lo:hi:hi]` so no
row can append into its neighbour. Idiomatic for the common case: the loop of
`make`. Reach for the one-allocation trick only when profiling says to.

## The zero value is useful

No constructor, no `make`, no nil check — just declare and append.

```go
package main

import (
  "fmt"
  "unsafe"
)

func main() {
  var s []int
  fmt.Println(s == nil, len(s), cap(s))
  s = append(s, 1, 2, 3)
  fmt.Println(s)

  var m map[string]int
  fmt.Println(m == nil, len(m), m["missing"])

  type counter struct {
    hits []string
  }
  var c counter
  c.hits = append(c.hits, "a")
  fmt.Println(c, unsafe.Sizeof(c))
}
```

Output:

```
true 0 0
[1 2 3]
true 0 0
{[a]} 24
```

A struct with a nil slice field needs no initialisation. A nil *map* also reads
fine, but writing to it panics — that trap is [7.6](06-maps.md).

## Quick Reference

| Expression | len | cap |
|---|---|---|
| `var s []T` | 0 | 0, and `s == nil` |
| `[]T{}` | 0 | 0, not nil |
| `make([]T, n)` | n | n |
| `make([]T, n, m)` | n | m |
| `[]int{5: 9}` | 6 | 6 |
| `s[lo:hi]` | `hi-lo` | `cap(s)-lo` |
| `s[lo:]` | `len(s)-lo` | `cap(s)-lo` |
| `s[:hi]` | `hi` | `cap(s)` |
| `s[lo:hi:max]` | `hi-lo` | `max-lo` |
| `a[:]` where `a` is `[N]T` | N | N |

| Operation | nil slice | empty slice |
|---|---|---|
| `len` / `cap` | 0 | 0 |
| `range` | zero iterations | zero iterations |
| `append` | works | works |
| `fmt.Println` | `[]` | `[]` |
| `== nil` | true | false |
| `json.Marshal` | `null` | `[]` |
| `slices.Equal` to each other | true | true |

| Error / panic | Cause |
|---|---|
| `invalid argument: length and capacity swapped` | `make([]T, 3, 1)` with constants |
| `makeslice: cap out of range` | `make([]T, n, 1)` with `n > 1` at runtime |
| `makeslice: len out of range` | negative or absurd length |
| `slice can only be compared to nil` | `s1 == s2` |
| `index out of range [i] with length n` | index `>= len`, even if `< cap` |

## Common Mistakes

**Checking `if s == nil` before using a slice.** `len`, `range`, and `append` all
handle nil; test `len(s) == 0` if you need emptiness.

**Returning a nil slice from a JSON API handler.** It serialises to `null` rather
than `[]`; initialise with `make([]T, 0)` when the contract says array.

**Expecting `cap(s[2:4])` to be 2.** Capacity runs to the end of the backing
array, so it is `cap(s)-2`.

**Writing `make([]T, n)` and then appending.** You now have `n` zero values
followed by your data; use `make([]T, 0, n)` to preallocate.

**Comparing slices with `==`.** It does not compile; use `slices.Equal` or
`bytes.Equal`.

## Exercises

1. Write `func describe(s []int)` printing `len`, `cap`, and whether `s == nil`.
   Call it with all six constructions from the Quick Reference and confirm the
   table.
2. Given `s := make([]int, 0, 8)`, produce sub-slices with len 3 and caps of 8, 5,
   and 3 respectively, using only slicing expressions.
3. Build a `[][]byte` of 1000 rows of 64 bytes two ways — a loop of `make`, and one
   flat allocation carved up — and compare allocation counts with
   `go test -bench . -benchmem`.
4. Write `func dedupKey(rows [][]int) int` that counts distinct rows. Slices are not
   comparable and not map keys, so convert each row to a comparable key; explain
   your choice.
5. Implement `func grow(p *[]int, n int)` that ensures `cap(*p) >= n` without
   changing `len`, then prove with `cap` that no further append allocates.

---

[← 7.1 Arrays: Fixed Size, Value Semantics](01-arrays.md) · [Course Index](../README.md) · [7.3 append Semantics and Reallocation →](03-append.md)
