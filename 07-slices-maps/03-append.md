[← 7.2 Slices: len, cap, and the Slice Header](02-slices-basics.md) · [Course Index](../README.md) · [7.4 The Shared Backing Array Trap →](04-shared-backing.md)

# 7.3 — append Semantics and Reallocation

## What You'll Learn

- `append` returns a new header; ignoring the return value is a bug (sometimes a compile error)
- The real capacity progression on go1.27: 2x doubling up to 256, then roughly 1.25x
- Preallocating with `make([]T, 0, n)` removes 18 of 19 allocations in a 10k-element loop
- Insertion idioms and why `slices.Insert` replaced the nested-append trick

## Coming From Another Language

| Other | Go |
|---|---|
| JS `arr.push(x)` — mutates in place | `s = append(s, x)` — you must reassign |
| Python `xs.append(x)` / `xs.extend(ys)` | `append(s, x)` / `append(s, ys...)` |
| Python `xs + ys` | `append(append([]T{}, xs...), ys...)` or `slices.Concat` |
| Rust `v.push(x)` on `&mut Vec` | Go has no `&mut`; `append` returns the new value |
| JS `arr.splice(i, 0, x)` | `slices.Insert(s, i, x)` |

`append` is a builtin with the signature `func append(s []T, vs ...T) []T`. It is
not a method and it does not mutate `s`'s header — it returns a new one. That one
fact explains every surprise in this lesson.

## Every call form

```go
package main

import "fmt"

func main() {
  var s []int
  s = append(s, 1)
  s = append(s, 2, 3, 4)
  t := []int{5, 6}
  s = append(s, t...)
  fmt.Println(s, len(s), cap(s))

  b := []byte("abc")
  b = append(b, "def"...)
  b = append(b, 'g')
  fmt.Println(string(b), len(b), cap(b))

  var n []string
  n = append(n, "x")
  fmt.Println(n, n == nil)

  empty := append([]int{}, []int{}...)
  fmt.Println(empty, len(empty), empty == nil)
}
```

Output:

```
[1 2 3 4 5 6] 6 8
abcdefg 7 8
[x] false
[] 0 false
```

Appending to a nil slice works and allocates on demand. `append(b, "def"...)` is a
special case the spec allows only for `[]byte` plus `string` — there is no generic
"spread a different type" rule.

## You must use the result

Discarding it outright does not even compile.

```go
package main

import "fmt"

func main() {
  s := []int{1, 2, 3}
  append(s, 4)
  fmt.Println(s)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:3: append(s, 4) (value of type []int) is not used
```

The compiler catches the bare call. What it cannot catch is assigning to something
that goes out of scope — including a parameter.

```go
package main

import "fmt"

func addSilently(s []int) {
  s = append(s, 99)
  _ = s
}

func main() {
  a := make([]int, 3, 10)
  addSilently(a)
  fmt.Println(a, len(a), cap(a))
  fmt.Println(a[:4])

  b := []int{1, 2, 3}
  _ = append(b, 4)
  fmt.Println(b, len(b), cap(b))
}
```

Output:

```
[0 0 0] 3 10
[0 0 0 99]
[1 2 3] 3 3
```

Silent data loss. The 99 was really written into the caller's backing array —
re-slicing to `a[:4]` proves it — but `a`'s length never moved, so it is invisible.
`go vet` does not flag either case. The rule has no exceptions: a function that
appends returns the slice, or takes `*[]T`.

## Growth: the real capacity progression

Appending one element at a time and printing `cap` whenever it changes, on
go1.27.1 darwin/arm64:

```go
package main

import "fmt"

func main() {
  var s []int
  last := -1
  for i := 0; i < 2100; i++ {
    s = append(s, i)
    if cap(s) != last {
      fmt.Print(cap(s), " ")
      last = cap(s)
    }
  }
  fmt.Println()

  var b []byte
  last = -1
  for i := 0; i < 4100; i++ {
    b = append(b, byte(i))
    if cap(b) != last {
      fmt.Print(cap(b), " ")
      last = cap(b)
    }
  }
  fmt.Println()

  type big struct{ a, b, c [8]int }
  var g []big
  last = -1
  for i := 0; i < 600; i++ {
    g = append(g, big{})
    if cap(g) != last {
      fmt.Print(cap(g), " ")
      last = cap(g)
    }
  }
  fmt.Println()
}
```

Output:

```
4 8 16 32 64 128 256 512 848 1280 1792 2560 
32 64 128 256 512 896 1408 2048 3072 4096 5376 
1 2 4 8 16 32 64 128 256 512 853 
```

Three things to read off this:

- Doubling holds up to 256 elements. Past that the factor drops toward 1.25 —
  512 to 848 is 1.66x, 1792 to 2560 is 1.43x, converging downward. The runtime
  computes a target and then rounds up to a malloc size class, which is why the
  numbers are 848 and 1280 rather than round powers.
- The first capacity depends on the element size, not just the count. `[]int`
  (8 bytes) starts at 4, `[]byte` at 32, and the 192-byte struct at 1 — the runtime
  is sizing the first allocation to a size class, not a count.
- None of this is in the spec. It has changed before (the 2x→1.25x threshold moved
  in Go 1.18) and will change again. Never write code whose correctness depends on
  a particular `cap`.

## Preallocating

`append` amortises to O(1), but the copies are real. Here is a 10,000-element build,
three ways. Run with `go test -bench . -benchmem`:

```
goos: darwin
goarch: arm64
pkg: m07
cpu: Apple M4 Pro
BenchmarkAppendNil-12         	   31423	     37361 ns/op	  357626 B/op	      19 allocs/op
BenchmarkAppendPrealloc-12    	   60259	     19841 ns/op	   81920 B/op	       1 allocs/op
BenchmarkIndexPrealloc-12     	  198949	      5999 ns/op	   81920 B/op	       1 allocs/op
```

The benchmark bodies:

```go
package main

import "testing"

const N = 10000

func BenchmarkAppendNil(b *testing.B) {
  for b.Loop() {
    var s []int
    for i := 0; i < N; i++ {
      s = append(s, i)
    }
  }
}

func BenchmarkAppendPrealloc(b *testing.B) {
  for b.Loop() {
    s := make([]int, 0, N)
    for i := 0; i < N; i++ {
      s = append(s, i)
    }
  }
}

func BenchmarkIndexPrealloc(b *testing.B) {
  for b.Loop() {
    s := make([]int, N)
    for i := 0; i < N; i++ {
      s[i] = i
    }
  }
}
```

19 allocations and 357 KB of churn for 81 KB of data — the growth path allocated
and abandoned every intermediate array. `make([]T, 0, n)` gets it to one allocation
and roughly halves the time. Writing by index with `make([]T, n)` is faster still
(no bounds-growth check per element) but only works when you know every index up
front.

Idiomatic: whenever you are building a slice from a source whose length you know,
write `make([]T, 0, len(src))`.

## Concatenation

```go
package main

import "fmt"

func main() {
  src := []int{1, 2, 3}
  a := append([]int(nil), src...)
  b := append([]int{}, src...)
  c := make([]int, 0, len(src))
  c = append(c, src...)

  x := []int{1, 2}
  y := []int{3, 4}
  cat := append(append([]int{}, x...), y...)

  fmt.Println(a, b, c, cat)
  fmt.Println(cap(a), cap(b), cap(c), cap(cat))

  bad := append(x, y...)
  fmt.Println(bad, x, cap(x))
}
```

Output:

```
[1 2 3] [1 2 3] [1 2 3] [1 2 3 4]
3 3 3 4
[1 2 3 4] [1 2] 2
```

`append(x, y...)` is the trap: it is only a copy when `x` has no spare capacity.
Here `cap(x) == 2` so it reallocated and `x` survived, but with `cap(x) >= 4` it
would have overwritten `x`'s tail. Use `slices.Concat(x, y)` (Go 1.22+) or
`append(append([]T{}, x...), y...)`. See [7.4](04-shared-backing.md).

## Insertion

The classic idiom nests two appends. It works, and it allocates twice.

```go
package main

import (
  "fmt"
  "slices"
)

func main() {
  s := []int{1, 2, 4, 5}
  i := 2

  a := append(s[:i], append([]int{3}, s[i:]...)...)
  fmt.Println(a)
  fmt.Println(s)

  t := []int{1, 2, 4, 5}
  t = slices.Insert(t, 2, 3)
  fmt.Println(t, len(t), cap(t))

  u := []int{1, 2, 5}
  u = slices.Insert(u, 2, 3, 4)
  fmt.Println(u)

  v := []int{1, 2}
  v = slices.Insert(v, 0, 0)
  v = slices.Insert(v, len(v), 9)
  fmt.Println(v)
}
```

Output:

```
[1 2 3 4 5]
[1 2 4 5]
[1 2 3 4 5] 5 8
[1 2 3 4 5]
[0 1 2 9]
```

`s` looks untouched only because `cap(s) == 4` forced a reallocation. Give it room
and the idiom scribbles on the original:

```go
package main

import "fmt"

func main() {
  s := make([]int, 4, 8)
  copy(s, []int{1, 2, 4, 5})
  i := 2
  a := append(s[:i], append([]int{3}, s[i:]...)...)
  fmt.Println(a)
  fmt.Println(s, len(s), cap(s))
  fmt.Println(s[:5])
}
```

Output:

```
[1 2 3 4 5]
[1 2 3 4] 4 8
[1 2 3 4 5]
```

`s` now reads `[1 2 3 4]` — index 2 was overwritten in place. Idiomatic:
`slices.Insert`, which is one allocation instead of two and does the shifting with
`copy`:

```
BenchmarkInsertIdiom-12     	56907032	        20.76 ns/op	      88 B/op	       2 allocs/op
BenchmarkSlicesInsert-12    	81858649	        14.95 ns/op	      64 B/op	       1 allocs/op
```

## append never shrinks capacity

Truncating with `s[:0]` keeps the array. The next append reuses it.

```go
package main

import "fmt"

func main() {
  s := make([]int, 0, 4)
  s = append(s, 1, 2, 3, 4, 5)
  fmt.Println(len(s), cap(s))
  s = s[:0]
  fmt.Println(len(s), cap(s))
  s = append(s, 9)
  fmt.Println(s, len(s), cap(s))
  fmt.Println(s[:5])
}
```

Output:

```
5 8
0 8
[9] 1 8
[9 2 3 4 5]
```

Capacity stayed at 8 across the truncation, and the old elements 2..5 are still in
the array — `s[:5]` reveals them. That is the buffer-reuse pattern (good for
avoiding allocations) and the stale-data pattern (bad if you hand out `s[:cap(s)]`
or hold pointers). `slices.Clip` and `clear` are the tools for that; see
[7.5](05-copy-three-index.md).

## Aliasing is the next lesson

Because `append` writes in place whenever `cap` allows, appending to a slice that
shares a backing array with another slice silently overwrites the other one's
elements — until a reallocation makes them independent. That is the most common
real bug in Go slice code, and [7.4](04-shared-backing.md) is devoted to it.

## Quick Reference

| Call | Effect |
|---|---|
| `append(s, v)` | one element |
| `append(s, a, b, c)` | several |
| `append(s, t...)` | spread another `[]T` |
| `append(b, "str"...)` | string into `[]byte` (special case) |
| `append([]T(nil), s...)` | clone (`slices.Clone` is clearer) |
| `append(s[:i], s[i+1:]...)` | delete index `i` (`slices.Delete` is clearer) |
| `slices.Insert(s, i, v...)` | insert, one allocation |
| `slices.Concat(a, b, c)` | concatenate, never aliases |

| Fact | Detail |
|---|---|
| Returns | a new header; the argument's header is unchanged |
| Writes in place | when `cap(s)-len(s) >= len(vs)` |
| Reallocates | otherwise, copying all existing elements |
| Growth factor | 2x up to 256 elements, then toward 1.25x, rounded to a size class |
| First capacity | depends on element size (4 for `int`, 32 for `byte`) |
| Spec guarantee | only `cap >= len`; never rely on an exact value |
| Nil receiver | fine |

## Common Mistakes

**Calling `append` without assigning the result.** The bare statement is a compile
error and `_ = append(...)` loses the data silently; always write `s = append(s, …)`.

**Appending inside a function and expecting the caller to see it.** The caller's
length never changes; return the slice or take `*[]T`.

**Writing `make([]T, n)` and then appending `n` items.** You end up with `2n`
elements, the first `n` of them zero; use `make([]T, 0, n)`.

**Using `append(a, b...)` to concatenate.** It mutates `a`'s backing array when `a`
has spare capacity; use `slices.Concat`.

**Depending on a specific capacity after append.** The growth factor is an
implementation detail that has already changed between releases.

## Exercises

1. Reproduce the capacity progression for `[]string`, `[]struct{}`, and
   `[]*int`. Explain why the first capacity differs between them.
2. Write `func appendN(n int) []int` with and without preallocation, benchmark both
   at n = 10, 1000, and 100000, and find the n at which preallocation stops
   mattering.
3. Write `func push(s *[]int, v int)` and `func push2(s []int, v int) []int`.
   Benchmark both and explain the difference in allocation counts.
4. Implement `func insertSorted(s []int, v int) []int` using `slices.BinarySearch`
   plus `slices.Insert`, and verify it with a fuzz test.
5. Build a slice of 1,000,000 ints with `var s []int` + append, and record
   `runtime.ReadMemStats().TotalAlloc` before and after. Compare with the
   preallocated version and explain the gap using the growth factors above.

---

[← 7.2 Slices: len, cap, and the Slice Header](02-slices-basics.md) · [Course Index](../README.md) · [7.4 The Shared Backing Array Trap →](04-shared-backing.md)
