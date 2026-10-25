[← 7.3 append Semantics and Reallocation](03-append.md) · [Course Index](../README.md) · [7.5 copy, Three-Index Slicing, and Deletion →](05-copy-three-index.md)

# 7.4 — The Shared Backing Array Trap

## What You'll Learn

- Two slices from the same array alias the same memory; writes through one are visible through the other
- `append` to a sub-slice overwrites its parent's elements until capacity runs out, then silently stops
- The three-index form `a[lo:hi:max]` caps capacity and turns the trap into a copy
- A short slice of a huge array keeps the whole array alive

## Coming From Another Language

| Other | Go |
|---|---|
| JS `arr.slice(1,3)` — copies | `a[1:3]` — a view; no copy |
| Python `xs[1:3]` — copies | `a[1:3]` — a view |
| Python `memoryview(b)[1:3]` — a view | the closest analogue |
| Rust `&v[1..3]` — a view, but the borrow checker forbids aliased writes | a view with no such check |

This is the single most common source of real bugs in Go code written by people
coming from JS or Python. There, slicing copies. In Go, slicing is free because
it copies no elements — it produces a new three-word header (pointer, len, cap)
pointing into the *same* array. Two slices sharing that array are two windows
onto one buffer, and nothing in the compiler warns you when you write through
one and read through the other.

## Two slices, one array

```go
package main

import "fmt"

func main() {
  a := []int{1, 2, 3, 4, 5}
  b := a[1:3]
  fmt.Println(a, len(a), cap(a))
  fmt.Println(b, len(b), cap(b))

  b[0] = 99
  fmt.Println(a)
  fmt.Println(b)
}
```

Output:

```
[1 2 3 4 5] 5 5
[2 3] 2 4
[1 99 3 4 5]
[99 3]
```

`b` starts at index 1, so `b[0]` *is* `a[1]`. Note `cap(b)` is 4, not 2: capacity
runs from the slice's start to the end of the backing array, so `b` can grow into
the space that `a[3]` and `a[4]` occupy.

## append to a sub-slice overwrites the parent

```go
package main

import "fmt"

func main() {
  a := []int{1, 2, 3, 4, 5}
  b := a[1:3]
  fmt.Println(a, b, len(b), cap(b))

  b = append(b, 100)
  fmt.Println(a)
  fmt.Println(b)
}
```

Output:

```
[1 2 3 4 5] [2 3] 2 4
[1 2 3 100 5]
[2 3 100]
```

`append` found spare capacity, so it wrote in place — into `a[3]`. The 4 that was
there is gone. No allocation, no copy, no warning.

## The reallocation boundary

This is the part that makes the bug intermittent. Append in place until capacity
is exhausted, then one more append.

```go
package main

import "fmt"

func main() {
  a := []int{1, 2, 3, 4, 5}
  b := a[1:3]

  b = append(b, 100, 200)
  fmt.Println(a, len(b), cap(b))

  b = append(b, 300)
  fmt.Println(a, len(b), cap(b))

  b[0] = -1
  fmt.Println(a)
  fmt.Println(b)
}
```

Output:

```
[1 2 3 100 200] 4 4
[1 2 3 100 200] 5 8
[1 2 3 100 200]
[-1 3 100 200 300]
```

The first `append` filled `b` to `cap` 4 and clobbered two of `a`'s elements. The
second exceeded capacity, so `append` allocated a new array of cap 8 and copied
into it — and from that moment `b` is independent. `a` stops changing, and the
later `b[0] = -1` is invisible to `a`.

So the aliasing is real right up until it isn't, and the switch flips on a
capacity you did not choose. Code that looks correct in a test with 3 elements
starts corrupting data at 5.

## The three-index form is the fix

`a[lo:hi:max]` sets capacity explicitly. Make `max == hi` and the slice has zero
spare capacity, so the first `append` must allocate.

```go
package main

import "fmt"

func main() {
  a := []int{1, 2, 3, 4, 5}
  b := a[1:3:3]
  fmt.Println(b, len(b), cap(b))

  b = append(b, 100)
  fmt.Println(a)
  fmt.Println(b)
}
```

Output:

```
[2 3] 2 2
[1 2 3 4 5]
[2 3 100]
```

`a` is untouched. Idiomatic: when you hand a sub-slice to code that may append to
it, write `a[lo:hi:hi]`.

## Passing a slice to a function

A slice is passed by value, but the value contains a pointer. The callee cannot
change your length, yet it can change your elements.

```go
package main

import "fmt"

func mutate(xs []int) {
  xs[0] = 42
}

func grow(xs []int) {
  xs = append(xs, 999)
  _ = xs
}

func main() {
  a := []int{1, 2, 3}
  mutate(a)
  fmt.Println(a)

  b := make([]int, 3, 10)
  copy(b, []int{1, 2, 3})
  grow(b)
  fmt.Println(b, len(b), cap(b))
  fmt.Println(b[:4][3])
}
```

Output:

```
[42 2 3]
[1 2 3] 3 10
999
```

`grow` appended within the caller's spare capacity. `b`'s length is still 3, so
the 999 is invisible through `b` — but it is really in the array, and re-slicing
to `b[:4]` exposes it. This is why a function that appends must return the slice.

## Filter in place

The `a[:0]` idiom reuses the array to avoid an allocation. It works, with one
artifact worth seeing.

```go
package main

import "fmt"

func main() {
  a := []string{"a", "b", "c", "d"}
  keep := a[:0]
  for _, s := range a {
    if s != "b" {
      keep = append(keep, s)
    }
  }
  fmt.Println(keep)
  fmt.Println(a)
}
```

Output:

```
[a c d]
[a c d d]
```

`keep` is correct. But `a` has been rewritten under you, and its tail still holds
a stale `"d"`. Only use this when you are done with the original, and use `keep`,
never `a`, afterwards. For pointer elements, zero the tail so the dropped values
can be collected.

## A small slice pins a large array

Capacity keeps the whole backing array reachable, so returning `xs[:3]` from a
million-element slice retains all million.

```go
package main

import "fmt"

func firstThree(xs []int) []int {
  return xs[:3]
}

func firstThreeCloned(xs []int) []int {
  return append([]int(nil), xs[:3]...)
}

func main() {
  big := make([]int, 1_000_000)
  for i := range big {
    big[i] = i
  }
  a := firstThree(big)
  b := firstThreeCloned(big)
  fmt.Println(a, len(a), cap(a))
  fmt.Println(b, len(b), cap(b))
}
```

Output:

```
[0 1 2] 3 1000000
[0 1 2] 3 3
```

Both print the same three elements; the `cap` tells you which one is holding 8 MB
hostage. When a long-lived value is a small window on a big buffer, copy it.

## Three ways to copy, all equivalent here

```go
package main

import (
  "fmt"
  "slices"
)

func main() {
  a := []int{1, 2, 3, 4, 5}

  c1 := slices.Clone(a[1:3])
  c2 := make([]int, len(a[1:3]))
  copy(c2, a[1:3])
  c3 := append([]int(nil), a[1:3]...)

  c1[0], c2[0], c3[0] = 7, 8, 9
  fmt.Println(a)
  fmt.Println(c1, cap(c1))
  fmt.Println(c2, cap(c2))
  fmt.Println(c3, cap(c3))
}
```

Output:

```
[1 2 3 4 5]
[7 3] 2
[8 3] 2
[9 3] 2
```

Idiomatic: `slices.Clone`. All three detach fully from `a`.

## Strings do not have this problem

Strings are immutable, so slicing one is always safe to share. Converting to
`[]byte` copies.

```go
package main

import "fmt"

func main() {
  s := "hello"
  t := s[1:3]
  b := []byte(s)
  b[0] = 72
  fmt.Println(s, t, string(b))
}
```

Output:

```
hello el Hello
```

## Capacity is not length

Indexing past `len` panics even when the capacity is there.

```go
package main

import "fmt"

func main() {
  a := []int{1, 2, 3, 4, 5}
  b := a[1:3]
  fmt.Println(b[2])
}
```

This program panics on purpose. Output:

```
panic: runtime error: index out of range [2] with length 2

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:8 +0x1c
exit status 2
```

Note the message says `length 2`, not capacity 4. To reach that element you must
re-slice: `b[:3][2]`.

## Quick Reference

| Expression | len | cap | Aliases source? |
|---|---|---|---|
| `a[lo:hi]` | `hi-lo` | `cap(a)-lo` | yes |
| `a[lo:hi:max]` | `hi-lo` | `max-lo` | yes, but append reallocates at `max` |
| `a[lo:hi:hi]` | `hi-lo` | `hi-lo` | yes; first append copies |
| `a[:0]` | 0 | `cap(a)` | yes; the filter-in-place idiom |
| `slices.Clone(a[lo:hi])` | `hi-lo` | `hi-lo` | no |
| `append([]T(nil), a[lo:hi]...)` | `hi-lo` | `hi-lo` | no |
| `copy(dst, a[lo:hi])` | `len(dst)` | — | no |

| Rule | Why |
|---|---|
| A function that appends must return the slice | the caller's `len` never changes |
| Hand out `a[lo:hi:hi]` | the callee's append cannot reach your elements |
| Clone before storing a sub-slice long-term | capacity pins the whole array |
| Never assume append reallocated | it only does so past `cap` |

## Common Mistakes

**Assuming `a[1:3]` copies, as it does in JS and Python.** It is a view; clone it
with `slices.Clone` when you need independence.

**Appending to a sub-slice you did not create.** It may overwrite the parent's
elements; cap it with `a[lo:hi:hi]` before passing it on.

**Writing `append(xs, v)` without using the result.** The new element may land
outside the caller's length and vanish; always assign the return value.

**Returning `xs[:n]` from a function given a huge slice.** It retains the entire
backing array; copy the window out instead.

**Reading `a` after filtering in place with `a[:0]`.** Its tail holds stale
elements; use the returned slice only.

## Exercises

1. Build `a := []int{1,2,3,4,5,6,7,8}` and `b := a[2:4]`. Append to `b` one
   element at a time, printing `a`, `len(b)`, and `cap(b)` each round. Identify
   the exact append at which `a` stops changing, and explain why.
2. Write `func split(xs []int) (left, right []int)` returning the two halves such
   that appending to `left` cannot corrupt `right`. Prove it with a test that
   fails against the naive two-index version.
3. Write `func dedupe(xs []int) []int` that removes consecutive duplicates in
   place with the `xs[:0]` idiom and allocates nothing. Then write the allocating
   version and compare with `go test -bench . -benchmem`.
4. Demonstrate the retention leak with `runtime.ReadMemStats`: hold `xs[:1]` of a
   50 MB slice, force a GC, and print `HeapAlloc`. Repeat with a clone and show
   the difference.
5. Given `[]*string`, filter in place and then show, with a finalizer or by
   inspecting the tail, that the dropped pointers are still reachable. Fix it by
   zeroing the tail.

---

[← 7.3 append Semantics and Reallocation](03-append.md) · [Course Index](../README.md) · [7.5 copy, Three-Index Slicing, and Deletion →](05-copy-three-index.md)
