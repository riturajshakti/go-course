[← 7.4 The Shared Backing Array Trap](04-shared-backing.md) · [Course Index](../README.md) · [7.6 Maps: Every Operation →](06-maps.md)

# 7.5 — copy, Three-Index Slicing, and Deletion

## What You'll Learn

- `copy` moves `min(len(dst), len(src))` elements and never grows `dst`
- `a[lo:hi:max]` is the full slicing form; `max` is bounded by `cap(a)`, not `len(a)`
- Every deletion pattern: order-preserving, swap-with-last, range, filter-in-place
- `clear(s)`, `s = s[:0]`, and `s = nil` do three different things

## Coming From Another Language

| Other | Go |
|---|---|
| JS `arr.splice(i, 1)` | `s = slices.Delete(s, i, i+1)` |
| JS `arr.filter(f)` | `slices.DeleteFunc(s, notF)` in place, or build a new slice |
| Python `del xs[i]` | no statement form; `slices.Delete` returns a new header |
| Python `xs[:] = []` | `clear(s)` zeroes, `s = s[:0]` truncates — not the same |
| Rust `v.remove(i)` / `v.swap_remove(i)` | `slices.Delete` / the swap-with-last idiom |
| `memcpy` | `copy(dst, src)` — bounds-checked, overlap-safe |

## copy: it does not grow dst

`copy(dst, src)` returns the number of elements moved, which is
`min(len(dst), len(src))`. Capacity is irrelevant.

```go
package main

import "fmt"

func main() {
  dst := make([]int, 3)
  n := copy(dst, []int{1, 2, 3, 4, 5})
  fmt.Println(n, dst)

  dst2 := make([]int, 5)
  n2 := copy(dst2, []int{1, 2})
  fmt.Println(n2, dst2)

  var nilDst []int
  fmt.Println(copy(nilDst, []int{1, 2}), nilDst)

  bad := make([]int, 0, 5)
  fmt.Println(copy(bad, []int{1, 2, 3}), bad, len(bad), cap(bad))

  b := make([]byte, 5)
  fmt.Println(copy(b, "hello world"), string(b))

  s := []int{1, 2, 3, 4, 5}
  fmt.Println(copy(s[1:], s[:4]), s)
  t := []int{1, 2, 3, 4, 5}
  fmt.Println(copy(t[:4], t[1:]), t)
}
```

Output:

```
3 [1 2 3]
2 [1 2 0 0 0]
0 []
0 [] 0 5
5 hello
4 [1 1 2 3 4]
4 [2 3 4 5 5]
```

Line 4 is the bug worth memorising: `make([]int, 0, 5)` has `len 0`, so `copy`
moves **nothing** and returns 0. The capacity is there but `copy` does not use it.
Idiomatic clone: `make([]T, len(src))` then copy, or just `slices.Clone(src)`.

`copy(dst, "string")` is the second special case the spec grants `[]byte` (the
first was `append(b, "s"...)`). And overlapping copies behave as if through a
temporary buffer — both the shift-right and shift-left above are correct, which is
exactly what makes `copy` usable for insert and delete.

## Three-index slicing

`a[lo:hi:max]` gives `len == hi-lo` and `cap == max-lo`.

```go
package main

import "fmt"

func main() {
  a := []int{0, 1, 2, 3, 4, 5, 6, 7}
  for _, e := range [][3]int{{2, 5, 8}, {2, 5, 6}, {2, 5, 5}, {0, 0, 3}} {
    s := a[e[0]:e[1]:e[2]]
    fmt.Printf("a[%d:%d:%d] -> %v len=%d cap=%d\n", e[0], e[1], e[2], s, len(s), cap(s))
  }
}
```

Output:

```
a[2:5:8] -> [2 3 4] len=3 cap=6
a[2:5:6] -> [2 3 4] len=3 cap=4
a[2:5:5] -> [2 3 4] len=3 cap=3
a[0:0:3] -> [] len=0 cap=3
```

The constraint is `lo <= hi <= max <= cap(a)`. Note `cap`, not `len` — you may set
`max` beyond `len(a)` as long as the array is that big. Exceeding `cap` panics,
with the same message whether the bound is a constant or a variable:

```go
package main

import "fmt"

func main() {
  a := []int{0, 1, 2, 3, 4}
  m := 9
  s := a[1:3:m]
  fmt.Println(s)
}
```

This program panics on purpose. Output:

```
panic: runtime error: slice bounds out of range [::9] with capacity 5

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:8 +0x48
exit status 2
```

The three-index form has exactly one everyday use: `a[lo:hi:hi]`, which zeroes the
spare capacity so the recipient's first `append` must allocate. That is the fix for
the aliasing trap in [7.4](04-shared-backing.md).

## Deletion, every way

```go
package main

import (
  "fmt"
  "slices"
)

func main() {
  a := []string{"a", "b", "c", "d", "e"}
  i := 2
  a = append(a[:i], a[i+1:]...)
  fmt.Println(a, len(a), cap(a))

  b := []string{"a", "b", "c", "d", "e"}
  b = slices.Delete(b, 2, 3)
  fmt.Println(b, len(b), cap(b))
  fmt.Println(b[:cap(b)])

  c := []string{"a", "b", "c", "d", "e"}
  c[2] = c[len(c)-1]
  c = c[:len(c)-1]
  fmt.Println(c)

  d := []string{"a", "b", "c", "d", "e", "f"}
  d = slices.Delete(d, 1, 4)
  fmt.Println(d)

  e := []int{1, 2, 3, 4, 5, 6}
  e = slices.DeleteFunc(e, func(v int) bool { return v%2 == 0 })
  fmt.Println(e)
}
```

Output:

```
[a b d e] 4 5
[a b d e] 4 5
[a b d e ]
[a b e d]
[a e f]
[1 3 5]
```

Four patterns:

- **Order-preserving, O(n):** `append(s[:i], s[i+1:]...)` or `slices.Delete(s, i, i+1)`.
  Both shift the tail down with one `copy`.
- **Unordered, O(1):** move the last element into the hole and truncate. `c` became
  `[a b e d]` — `"e"` jumped to index 2. Use it for sets and worklists where order
  does not matter.
- **Range:** `slices.Delete(s, i, j)` removes a half-open range in one shift.
- **Predicate:** `slices.DeleteFunc(s, pred)` removes everything matching.

Note `cap` stays 5 in all of them. Deletion shortens `len`, never the array. And
look at the third line of output: `b[:cap(b)]` is `[a b d e ]` — the last slot
holds `""`, not the stale `"e"`. `slices.Delete` zeroes the vacated tail.

## The pointer-element memory leak

The hand-rolled idiom does **not** zero the tail, so the removed element stays
reachable.

```go
package main

import (
  "fmt"
  "slices"
)

type big struct{ pad [1024]byte }

func main() {
  s := []*big{{}, {}, {}, {}}
  for i := range s {
    s[i].pad[0] = byte(i + 1)
  }

  naive := append(s[:1], s[2:]...)
  fmt.Println(len(naive), cap(naive))
  tail := naive[:cap(naive)]
  fmt.Println(tail[3] == nil, tail[3].pad[0])

  t := []*big{{}, {}, {}, {}}
  for i := range t {
    t[i].pad[0] = byte(i + 1)
  }
  t = slices.Delete(t, 1, 2)
  ttail := t[:cap(t)]
  fmt.Println(len(t), cap(t), ttail[3] == nil)
}
```

Output:

```
3 4
false 4
3 4 true
```

After the naive delete, slot 3 is beyond `len` but still points at the fourth
`big`, so the GC cannot collect it even though the slice logically holds three
elements. After `slices.Delete` the same slot is `nil`. On go1.27.1,
`slices.Delete` zeroes `s[len(s)-k : len(s)]` for you — this was added in Go 1.22
and is now the reason to prefer it over the append idiom for any pointer-bearing
element type.

If you must hand-roll, zero the tail yourself:

```go
copy(s[i:], s[i+1:])
s[len(s)-1] = nil
s = s[:len(s)-1]
```

## clear vs s[:0] vs s = nil

Three operations that all look like "empty the slice" and all behave differently.

```go
package main

import "fmt"

func show(label string, s []int) {
  fmt.Printf("%-10s %v len=%d cap=%d nil=%v\n", label, s, len(s), cap(s), s == nil)
}

func main() {
  base := func() []int { return []int{1, 2, 3, 4, 5} }

  a := base()
  clear(a)
  show("clear", a)

  b := base()
  b = b[:0]
  show("b[:0]", b)
  fmt.Println(b[:cap(b)])

  c := base()
  c = nil
  show("nil", c)
}
```

Output:

```
clear      [0 0 0 0 0] len=5 cap=5 nil=false
b[:0]      [] len=0 cap=5 nil=false
[1 2 3 4 5]
nil        [] len=0 cap=0 nil=true
```

| Operation | len | cap | elements | array |
|---|---|---|---|---|
| `clear(s)` (Go 1.21+) | unchanged | unchanged | all zeroed | reused |
| `s = s[:0]` | 0 | unchanged | **untouched** | reused |
| `s = nil` | 0 | 0 | unreachable | released |
| `s = s[:0]` then `clear(s[:cap(s)])` | 0 | unchanged | zeroed | reused |

`s = s[:0]` is the buffer-reuse idiom: fast, zero allocations, but the old data is
still in the array and visible through `s[:cap(s)]`. `clear` zeroes without
changing length — useful to reset a fixed-size window. `s = nil` is the only one
that lets the array be collected.

## Releasing memory: Clip and Clone

`slices.Clip(s)` returns `s[:len(s):len(s)]` — same elements, capacity trimmed to
length, no copy. It stops future appends from reaching the old tail but does *not*
release the memory, because the pointer still points into the big array. To
actually release, clone.

```go
package main

import (
  "fmt"
  "slices"
)

func main() {
  s := make([]int, 3, 100)
  copy(s, []int{1, 2, 3})
  fmt.Println(len(s), cap(s))
  t := slices.Clip(s)
  fmt.Println(t, len(t), cap(t))
  t = append(t, 9)
  fmt.Println(s[:3], t)

  u := make([]int, 1000)
  keep := slices.Clone(u[:3])
  fmt.Println(len(keep), cap(keep))
  trunc := u[:3]
  fmt.Println(len(trunc), cap(trunc))
}
```

Output:

```
3 100
[1 2 3] 3 3
[1 2 3] [1 2 3 9]
3 3
3 1000
```

`slices.Clone(u[:3])` has cap 3 and lets the 1000-element array go. `u[:3]` has cap
1000 and pins it. Idiomatic: clip when passing a slice onward, clone when storing
it long-term.

## Filter in place, reverse, rotate

```go
package main

import (
  "fmt"
  "slices"
)

func main() {
  xs := []int{1, 2, 3, 4, 5, 6}
  keep := xs[:0]
  for _, v := range xs {
    if v%2 == 1 {
      keep = append(keep, v)
    }
  }
  fmt.Println(keep)
  fmt.Println(xs)

  ys := []int{1, 2, 3, 4, 5}
  slices.Reverse(ys)
  fmt.Println(ys)

  zs := []int{1, 2, 3, 4, 5}
  k := 2
  rotated := append(zs[k:], zs[:k]...)
  fmt.Println(rotated)

  ws := []int{1, 2, 3, 4, 5}
  n := 2
  slices.Reverse(ws[:n])
  slices.Reverse(ws[n:])
  slices.Reverse(ws)
  fmt.Println(ws)
}
```

Output:

```
[1 3 5]
[1 3 5 4 5 6]
[5 4 3 2 1]
[3 4 5 1 2]
[3 4 5 1 2]
```

The `xs[:0]` filter allocates nothing but rewrites `xs` under you — `[1 3 5 4 5 6]`,
with a stale tail. Only use `keep` afterwards; see [7.4](04-shared-backing.md).

Both rotations give `[3 4 5 1 2]`. The `append(zs[k:], zs[:k]...)` version is
shorter but corrupts `zs` (it appends into `zs`'s own capacity) and so is only safe
when you discard the original. The triple-reverse is the allocation-free,
non-destructive-in-the-wrong-way answer: it genuinely rotates in place.

## Insertion with copy

For completeness, what `slices.Insert` does under the hood:

```go
package main

import "fmt"

func insert(s []int, i int, v ...int) []int {
  s = append(s, make([]int, len(v))...)
  copy(s[i+len(v):], s[i:])
  copy(s[i:], v)
  return s
}

func main() {
  fmt.Println(insert([]int{1, 2, 5, 6}, 2, 3, 4))
  fmt.Println(insert([]int{1, 2}, 0, 0))
  fmt.Println(insert([]int{1, 2}, 2, 3))
}
```

Output:

```
[1 2 3 4 5 6]
[0 1 2]
[1 2 3]
```

Grow once, shift the tail right, write the new elements. Use `slices.Insert` in
real code; this is here so the mechanics are not magic.

## Quick Reference

| Call | Returns | Notes |
|---|---|---|
| `copy(dst, src)` | `int` moved | `min(len(dst), len(src))`; never grows `dst` |
| `copy(b, "s")` | `int` | only for `dst []byte` |
| `clear(s)` | — | zeroes all `len(s)` elements, Go 1.21+ |
| `a[lo:hi]` | slice | `cap = cap(a)-lo` |
| `a[lo:hi:max]` | slice | `cap = max-lo`, requires `max <= cap(a)` |
| `a[lo:hi:hi]` | slice | zero spare capacity; the aliasing fix |
| `slices.Delete(s, i, j)` | new header | shifts down, zeroes the tail |
| `slices.DeleteFunc(s, f)` | new header | removes all matches |
| `slices.Insert(s, i, v...)` | new header | one allocation |
| `slices.Clip(s)` | `s[:len:len]` | trims cap, no copy, no release |
| `slices.Clone(s)` | new array | `cap == len`; releases the source |
| `slices.Reverse(s)` | — | in place, no return |

| Deletion | Cost | Order | Use when |
|---|---|---|---|
| `slices.Delete(s, i, i+1)` | O(n) | preserved | default |
| `s[i] = s[len(s)-1]; s = s[:len(s)-1]` | O(1) | destroyed | order is irrelevant |
| `slices.DeleteFunc(s, pred)` | O(n) | preserved | removing many |
| `keep := s[:0]` + append loop | O(n), 0 allocs | preserved | hot path, `s` discarded |

| Panic | Cause |
|---|---|
| `slice bounds out of range [::9] with capacity 5` | `a[lo:hi:max]` with `max > cap(a)` |
| `slice bounds out of range [2:1]` | `hi < lo` |

## Common Mistakes

**`c := make([]T, 0, len(src)); copy(c, src)`.** `copy` is bounded by `len(dst)`,
which is 0; use `make([]T, len(src))` or `slices.Clone`.

**Deleting with `append(s[:i], s[i+1:]...)` on a slice of pointers.** The vacated
tail slot still references the removed value; use `slices.Delete`, which zeroes it.

**Using `s = s[:0]` to "clear" a slice whose old contents must not leak.** The
elements are still in the array and reachable via `s[:cap(s)]`; use `clear` or
`s = nil`.

**Expecting `slices.Clip` to free memory.** It only trims `cap`; the pointer still
targets the original array, so clone to release it.

**Rotating with `append(s[k:], s[:k]...)` and then using `s`.** It writes into
`s`'s own backing array; reverse three times instead.

## Exercises

1. Write `func shiftLeft(s []int, n int)` that moves every element down `n`
   positions and zeroes the last `n`, using exactly one `copy` and one `clear`.
2. Implement `func removeAll[T comparable](s []T, v T) []T` twice — with
   `slices.DeleteFunc` and with the `s[:0]` idiom — and benchmark both with
   `-benchmem`.
3. Write a test proving that after `slices.Delete` on a `[]*int`, the removed
   pointer is unreachable: set a `runtime.SetFinalizer` on it, force a GC, and
   assert the finalizer ran. Show it fails with the append idiom.
4. Write `func window(s []int, i, n int) []int` returning `n` elements starting at
   `i`, such that the caller's appends can never touch `s`. Prove it.
5. Build a 10 MB `[]byte`, keep only the first 100 bytes, and show with
   `runtime.ReadMemStats` which of `s[:100]`, `slices.Clip(s[:100])`, and
   `slices.Clone(s[:100])` actually releases the memory after a GC.

---

[← 7.4 The Shared Backing Array Trap](04-shared-backing.md) · [Course Index](../README.md) · [7.6 Maps: Every Operation →](06-maps.md)
