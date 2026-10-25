[← 6.8 unicode, utf8, and Text Edge Cases](../06-strings/08-unicode-utf8.md) · [Course Index](../README.md) · [7.2 Slices: len, cap, and the Slice Header →](02-slices-basics.md)

# 7.1 — Arrays: Fixed Size, Value Semantics

## What You'll Learn

- The length is part of the type: `[3]int` and `[4]int` do not assign to each other
- Arrays are values — assignment and function calls copy every element
- Arrays are comparable with `==` when the element type is, which makes them usable as map keys
- A constant out-of-range index is a compile error; a variable one is a runtime panic

## Coming From Another Language

| Other | Go |
|---|---|
| JS `Array` — dynamic, reference | `[N]T` — fixed length, value; `[]T` (next lesson) is the dynamic one |
| Python `list` — dynamic, reference | `[N]T` is closer to `bytes` of fixed size; nothing really matches |
| Python `tuple` — fixed, immutable | `[N]T` is fixed but mutable, and comparable like a tuple |
| Rust `[T; N]` — exactly this | `[N]T`, including `Copy` semantics |
| TS `number[]` has no length in the type | `[3]int` and `[4]int` are *different types* |

Most Go code uses slices, not arrays. Arrays are worth a lesson because they are
the thing slices are built on, and because a handful of jobs — hashes, fixed
buffers, map keys, matrices — want the real array.

## Declaration forms

```go
package main

import "fmt"

func main() {
  var a [3]int
  b := [3]int{1, 2, 3}
  c := [...]int{1, 2, 3, 4}
  d := [5]int{2: 7}
  e := [...]string{3: "d"}
  f := [2][3]int{{1, 2, 3}, {4, 5, 6}}

  fmt.Println(a, len(a))
  fmt.Println(b, len(b))
  fmt.Println(c, len(c))
  fmt.Println(d, len(d))
  fmt.Printf("%q %d\n", e, len(e))
  fmt.Println(f, len(f), len(f[0]))
  fmt.Printf("%T %T %T\n", b, c, e)
}
```

Output:

```
[0 0 0] 3
[1 2 3] 3
[1 2 3 4] 4
[0 0 7 0 0] 5
["" "" "" "d"] 4
[[1 2 3] [4 5 6]] 2 3
[3]int [4]int [4]string
```

The zero value is a usable array of zeroed elements — no allocation step. `[...]T{…}`
lets the compiler count. Indexed (sparse) literals fill the rest with the zero
value, and with `[...]` the *highest index* decides the length: `[...]string{3: "d"}`
has length 4.

## The length is part of the type

```go
package main

import "fmt"

func main() {
  a := [3]int{1, 2, 3}
  var b [4]int
  b = a
  fmt.Println(b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:7: cannot use a (variable of type [3]int) as [4]int value in assignment
```

There is no array type that is generic over length outside of generics. A function
taking `[3]int` cannot be handed a `[4]int`. This is the single biggest reason
everyday Go code reaches for slices.

## Assignment and function calls copy

```go
package main

import "fmt"

func modify(a [3]int) {
  a[0] = 99
}

func modifyPtr(a *[3]int) {
  a[0] = 99
}

func main() {
  a := [3]int{1, 2, 3}
  b := a
  b[0] = 42
  fmt.Println(a, b)

  modify(a)
  fmt.Println(a)

  modifyPtr(&a)
  fmt.Println(a)
}
```

Output:

```
[1 2 3] [42 2 3]
[1 2 3]
[99 2 3]
```

`b := a` copied all three elements. `modify` got its own copy and its write was
discarded. Pass `*[N]T` to mutate, and note `a[0]` works on the pointer directly —
Go auto-dereferences an array pointer for indexing, `len`, `cap`, and `range`.

The copy is not free. `[1000]int` is 8000 bytes, and passing it by value moves all
8000 on every call.

```go
package main

import (
  "fmt"
  "unsafe"
)

func main() {
  var a [3]int
  var b [1000]int
  var c [32]byte
  var d [4]string
  var s []int

  fmt.Println(unsafe.Sizeof(a), unsafe.Sizeof(b), unsafe.Sizeof(c), unsafe.Sizeof(d))
  fmt.Println(unsafe.Sizeof(s))
  fmt.Println(len(a), cap(a))

  p := &a
  p[0] = 7
  fmt.Println(p[0], len(p), cap(p))
  for i, v := range p {
    fmt.Print(i, v, " ")
  }
  fmt.Println()
}
```

Output:

```
24 8000 32 64
24
3 3
7 3 3
0 7 1 0 2 0 
```

An array's size *is* its contents. A slice is always 24 bytes on 64-bit regardless
of length. For an array, `len` and `cap` are equal, and both are compile-time
constants usable in a constant expression.

## Arrays are comparable

```go
package main

import "fmt"

type point struct{ x, y int }

func main() {
  a := [3]int{1, 2, 3}
  b := [3]int{1, 2, 3}
  c := [3]int{1, 2, 4}
  fmt.Println(a == b, a == c)

  p := [2]point{{1, 2}, {3, 4}}
  q := [2]point{{1, 2}, {3, 4}}
  fmt.Println(p == q)

  grid := map[[2]int]string{
    {0, 0}: "origin",
    {1, 2}: "target",
  }
  fmt.Println(grid[[2]int{1, 2}], len(grid))
}
```

Output:

```
true false
true
target 2
```

`==` compares element by element. Because arrays are comparable, they work as map
keys — which slices never can. Note the literal shorthand inside the map: the key
type can be omitted, so `{1, 2}` means `[2]int{1, 2}`.

Comparability is inherited from the element type. An array of slices is not
comparable.

```go
package main

import "fmt"

func main() {
  a := [3][]int{}
  b := [3][]int{}
  fmt.Println(a == b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: a == b ([3][]int cannot be compared)
```

## range copies the array

`range a` evaluates `a` once, and for an array that evaluation is a copy. Writes to
the original during the loop are invisible to `v`. The same loop over a slice sees
them, because the slice header's pointer is what was copied.

```go
package main

import "fmt"

func main() {
  a := [3]int{1, 2, 3}
  for i, v := range a {
    if i == 0 {
      a[1] = 99
      a[2] = 99
    }
    fmt.Println(i, v)
  }
  fmt.Println(a)

  s := []int{1, 2, 3}
  for i, v := range s {
    if i == 0 {
      s[1] = 99
      s[2] = 99
    }
    fmt.Println(i, v)
  }
  fmt.Println(s)
}
```

Output:

```
0 1
1 2
2 3
[1 99 99]
0 1
1 99
2 99
[1 99 99]
```

Both arrays end up `[1 99 99]`, but the array loop printed the pre-write values.
Idiomatic: `for i := range a` and index explicitly when the body mutates, or range
over `&a` to avoid the copy entirely.

## Out of range: compile time vs runtime

A constant index is checked by the compiler.

```go
package main

import "fmt"

func main() {
  a := [3]int{1, 2, 3}
  fmt.Println(a[5])
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:17: invalid argument: index 5 out of bounds [0:3]
```

Make the index a variable and the check moves to runtime.

```go
package main

import "fmt"

func main() {
  a := [3]int{1, 2, 3}
  i := 5
  fmt.Println(a[i])
}
```

This program panics on purpose. Output:

```
panic: runtime error: index out of range [5] with length 3

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:8 +0x1c
exit status 2
```

## Arrays of arrays are contiguous

A `[3][4]int` is 12 ints laid out end to end. A `[3][]int` is three slice headers
pointing at three separate allocations.

```go
package main

import (
  "fmt"
  "unsafe"
)

func main() {
  var m [3][4]int
  m[1][2] = 7
  fmt.Println(m)
  fmt.Println(unsafe.Sizeof(m), 3*4*8)

  flat := (*[12]int)(unsafe.Pointer(&m))
  fmt.Println(flat[6])

  var s [3][]int
  for i := range s {
    s[i] = make([]int, 4)
  }
  s[1][2] = 7
  fmt.Println(s, unsafe.Sizeof(s))
}
```

Output:

```
[[0 0 0 0] [0 0 7 0] [0 0 0 0]]
96 96
7
[[0 0 0 0] [0 0 7 0] [0 0 0 0]] 72
```

`m[1][2]` is `flat[1*4+2]` — one allocation, cache-friendly, and reinterpretable as
a flat array. The slice-of-slices version is 72 bytes of headers plus three heap
blocks, and rows can be different lengths (jagged). For a real matrix the array
form, or a single flat slice with manual indexing, wins.

## When an array is the right answer

Hashes, fixed protocol buffers, and map keys.

```go
package main

import (
  "crypto/sha256"
  "fmt"
)

func checksum(data []byte) [32]byte {
  return sha256.Sum256(data)
}

func main() {
  h := checksum([]byte("hello"))
  fmt.Printf("%x\n", h)
  fmt.Printf("%T %d\n", h, len(h))

  seen := map[[32]byte]bool{h: true}
  fmt.Println(seen[checksum([]byte("hello"))])

  var buf [64]byte
  n := copy(buf[:], "payload")
  fmt.Println(n, string(buf[:n]))
}
```

Output:

```
2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
[32]uint8 32
true
7 payload
```

`sha256.Sum256` returns `[32]byte`, not `[]byte`, precisely so the result is a
comparable value you can use as a key and pass without an allocation. `var buf
[64]byte` followed by `buf[:]` is the standard way to get a scratch slice whose
backing array the compiler can keep on the stack.

## Slice to array, and slice to array pointer

Go 1.20 added the conversion `[N]T(s)`, which copies, and `(*[N]T)(s)`, which
aliases.

```go
package main

import "fmt"

func main() {
  s := []int{1, 2, 3, 4, 5}
  a := [4]int(s[:4])
  fmt.Println(a, len(a))

  p := (*[3]int)(s)
  p[0] = 99
  fmt.Println(s, *p)

  b := [5]int(s)
  fmt.Println(b)
}
```

Output:

```
[1 2 3 4] 4
[99 2 3 4 5] [99 2 3]
[99 2 3 4 5]
```

The pointer form shares memory — `p[0] = 99` changed `s`. The value form copies.
Both require `len(s) >= N`, checked at runtime.

```go
package main

import "fmt"

func main() {
  s := []int{1, 2}
  a := [4]int(s)
  fmt.Println(a)
}
```

This program panics on purpose. Output:

```
panic: runtime error: cannot convert slice with length 2 to array or pointer to array with length 4

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:7 +0x1c
exit status 2
```

Note it is `len`, not `cap`, that must be big enough.

## Quick Reference

| Form | Meaning |
|---|---|
| `var a [3]int` | zero-valued array of 3 |
| `[3]int{1, 2, 3}` | full literal |
| `[...]int{1, 2, 3, 4}` | length inferred as 4 |
| `[5]int{2: 7}` | indexed literal, length 5 |
| `[...]string{3: "d"}` | indexed literal, length 4 (highest index + 1) |
| `[2][3]int{{1,2,3},{4,5,6}}` | 2D, contiguous |
| `a[:]` | slice over the whole array |
| `[N]T(s)` | copy a slice into an array (Go 1.20+) |
| `(*[N]T)(s)` | array pointer aliasing the slice (Go 1.20+) |

| Property | Arrays | Slices |
|---|---|---|
| Length in the type | yes | no |
| Assignment | copies all elements | copies a 24-byte header |
| `==` | yes, if element type is comparable | no (nil only) |
| Valid map key | yes | no |
| `len` / `cap` | equal, compile-time constant | independent, runtime |
| `range` of the value | iterates a copy | iterates shared elements |
| `unsafe.Sizeof` | `N * sizeof(T)` | 24 on 64-bit |

| Error | When |
|---|---|
| `cannot use a ([3]int) as [4]int` | different lengths are different types |
| `invalid argument: index 5 out of bounds [0:3]` | constant index, compile time |
| `index out of range [5] with length 3` | variable index, runtime panic |
| `cannot convert slice with length 2 to array ... length 4` | `[N]T(s)` with `len(s) < N` |

## Common Mistakes

**Writing `[3]int` where the size should flex.** Use `[]int`; the length in the
type makes arrays almost unusable as a general parameter.

**Passing a large array by value in a hot path.** Every call copies
`N * sizeof(T)` bytes; pass `*[N]T` or a slice instead.

**Expecting `for _, v := range a` to see writes made inside the loop.** `range`
iterates a copy of the array; index with `a[i]` or range over `&a`.

**Using `a[:]` and then letting the slice outlive the array's scope.** It is
correct, but the slice keeps the whole array alive and escapes it to the heap.

**Assuming `[N]T(s)` checks `cap`.** It checks `len`, and panics at runtime when
the slice is too short.

## Exercises

1. Write `func sum(a [4]int) int` and `func sumPtr(a *[4]int) int`, benchmark both
   with `[4096]int` using `go test -bench . -benchmem`, and report the per-call
   difference.
2. Build `map[[3]string]int` keyed by `{country, region, city}` and count rows from
   a slice of structs. Then rewrite it with a struct key and compare readability.
3. Implement a 4x4 matrix multiply twice: once with `[4][4]float64` and once with
   `[][]float64`. Benchmark both and explain the gap from the memory layout.
4. Given `s := []byte("hello world")`, produce a `[5]byte` of the first word and a
   `*[5]byte` aliasing the same bytes. Mutate through the pointer and show which of
   the two changed.
5. Write a function that takes `[]int` and returns `([8]int, bool)` — the first
   eight elements and whether there were at least eight — without ever panicking.

---

[← 6.8 unicode, utf8, and Text Edge Cases](../06-strings/08-unicode-utf8.md) · [Course Index](../README.md) · [7.2 Slices: len, cap, and the Slice Header →](02-slices-basics.md)
