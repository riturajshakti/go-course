[← 7.9 The maps Package Reference](09-maps-package.md) · [Course Index](../README.md) · [8.1 The error Interface and errors.New →](../08-errors/01-error-interface.md)

# 7.10 — sort: Before and Alongside Generics

## What You'll Learn

- Use `slices.Sort` and `slices.SortFunc` for new code; `sort` is what you meet in existing code
- `sort.Slice` is reflect-based and measurably slower than `slices.SortFunc`
- `sort.Sort` is not stable; `sort.Stable` is — with a real demonstration of the difference
- `sort.Reverse` wraps a comparator; `slices.Reverse` flips a slice in place — they are unrelated

## Coming From Another Language

| Other | Go (modern) | Go (`sort`) |
|---|---|---|
| JS `arr.sort(cmp)` | `slices.SortFunc(s, cmp)` | `sort.Slice(s, less)` |
| Python `xs.sort(key=f)` | no key form; write a `cmp` | same |
| Python `sorted(xs, reverse=True)` | `slices.SortFunc` with flipped args | `sort.Sort(sort.Reverse(…))` |
| Rust `v.sort_by` / `sort_unstable_by` | `slices.SortStableFunc` / `SortFunc` | — |
| Java `Comparable` | `cmp.Ordered` constraint | `sort.Interface` |

Start here: **for a slice, use `slices.Sort` or `slices.SortFunc`.** They are
generic, type-safe, allocation-free, and faster. You still need `sort` for
non-slice collections, for code that already implements `sort.Interface`, and for
`sort.Search` on an abstract index space.

## The whole sort package

From `go doc sort` on go1.27.1:

```
func Find(n int, cmp func(int) int) (i int, found bool)
func Float64s(x []float64)
func Float64sAreSorted(x []float64) bool
func Ints(x []int)
func IntsAreSorted(x []int) bool
func IsSorted(data Interface) bool
func Search(n int, f func(int) bool) int
func SearchFloat64s(a []float64, x float64) int
func SearchInts(a []int, x int) int
func SearchStrings(a []string, x string) int
func Slice(x any, less func(i, j int) bool)
func SliceIsSorted(x any, less func(i, j int) bool) bool
func SliceStable(x any, less func(i, j int) bool)
func Sort(data Interface)
func Stable(data Interface)
func Strings(x []string)
func StringsAreSorted(x []string) bool
type Float64Slice []float64
type IntSlice []int
type Interface interface{ ... }
    func Reverse(data Interface) Interface
type StringSlice []string
```

`sort.Ints`, `sort.Strings` and `sort.Float64s` are soft-deprecated: the Go 1.22 doc
comment on each says "this function simply calls `slices.Sort`". They are not marked
`Deprecated:` in the source, so no tool will flag them — but there is no reason to
write them in new code.

## sort.Interface

Three methods on a named type. This is the only API in `sort` that works on
something other than a slice.

```go
package main

import (
  "fmt"
  "sort"
)

type person struct {
  name string
  dept string
  age  int
}

type byAge []person

func (p byAge) Len() int           { return len(p) }
func (p byAge) Less(i, j int) bool { return p[i].age < p[j].age }
func (p byAge) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

func main() {
  ps := []person{{"cat", "ops", 25}, {"ann", "eng", 40}, {"bob", "eng", 30}}
  sort.Sort(byAge(ps))
  fmt.Println(ps)
  fmt.Println(sort.IsSorted(byAge(ps)))

  sort.Sort(sort.Reverse(byAge(ps)))
  fmt.Println(ps)

  sort.Stable(byAge(ps))
  fmt.Println(ps)

  ints := []int{3, 1, 2}
  sort.Ints(ints)
  strs := []string{"c", "a"}
  sort.Strings(strs)
  f := []float64{2.5, 1.5}
  sort.Float64s(f)
  fmt.Println(ints, strs, f)
  fmt.Println(sort.IntsAreSorted(ints), sort.StringsAreSorted(strs), sort.Float64sAreSorted(f))

  fmt.Println(sort.IntSlice{3, 1}.Len())
}
```

Output:

```
[{cat ops 25} {bob eng 30} {ann eng 40}]
true
[{ann eng 40} {bob eng 30} {cat ops 25}]
[{cat ops 25} {bob eng 30} {ann eng 40}]
[1 2 3] [a c] [1.5 2.5]
true true true
2
```

`sort.Interface` is still the right tool when:

- the collection is not a slice (a ring buffer, two parallel slices sorted together, a struct wrapping a slice plus metadata)
- the ordering *is* part of the type's identity and belongs in its API
- you need to pass the ordering around as a value, which `sort.Reverse` does

Note `Less` returns `bool`, and `Swap` means you cannot sort an immutable view.

## sort.Slice: the reflect-based middle generation

`sort.Slice` took `any` plus a `func(i, j int) bool` and used reflection to do the
swapping. It was the idiomatic answer from Go 1.8 until 1.21.

```go
package main

import (
  "fmt"
  "sort"
)

type person struct {
  name string
  dept string
  age  int
}

func main() {
  ps := []person{{"cat", "ops", 25}, {"ann", "eng", 40}, {"bob", "eng", 30}}
  sort.Slice(ps, func(i, j int) bool { return ps[i].age < ps[j].age })
  fmt.Println(ps)

  sort.SliceStable(ps, func(i, j int) bool { return ps[i].dept < ps[j].dept })
  fmt.Println(ps)

  fmt.Println(sort.SliceIsSorted(ps, func(i, j int) bool { return ps[i].dept < ps[j].dept }))

  sort.Slice(ps, func(i, j int) bool {
    if ps[i].dept != ps[j].dept {
      return ps[i].dept < ps[j].dept
    }
    return ps[i].age > ps[j].age
  })
  fmt.Println(ps)
}
```

Output:

```
[{cat ops 25} {bob eng 30} {ann eng 40}]
[{bob eng 30} {ann eng 40} {cat ops 25}]
true
[{ann eng 40} {bob eng 30} {cat ops 25}]
```

Two warts. The `less` closes over `ps` by name — passing the wrong slice is a silent
bug the compiler cannot see. And multi-key comparison becomes a chain of `if`s,
where `slices.SortFunc` + `cmp.Or` is one expression.

## The benchmark

10,000 structs sorted by one int field, with `go test -bench . -benchmem`:

```
goos: darwin
goarch: arm64
pkg: m07
cpu: Apple M4 Pro
BenchmarkSortSlice-12         	    1584	    756984 ns/op	      96 B/op	       3 allocs/op
BenchmarkSlicesSortFunc-12    	    2187	    468843 ns/op	       0 B/op	       0 allocs/op
BenchmarkSortInterface-12     	    2286	    454041 ns/op	      24 B/op	       1 allocs/op
BenchmarkSlicesSortInts-12    	    6171	    191873 ns/op	       0 B/op	       0 allocs/op
BenchmarkSortInts-12          	    6343	    185520 ns/op	      0 B/op	       0 allocs/op
```

The bodies:

```go
package main

import (
  "cmp"
  "math/rand"
  "slices"
  "sort"
  "testing"
)

type item struct {
  key  int
  name string
}

func data(n int) []item {
  r := rand.New(rand.NewSource(1))
  out := make([]item, n)
  for i := range out {
    out[i] = item{key: r.Intn(n), name: "x"}
  }
  return out
}

const N = 10000

func BenchmarkSortSlice(b *testing.B) {
  src := data(N)
  buf := make([]item, N)
  for b.Loop() {
    copy(buf, src)
    sort.Slice(buf, func(i, j int) bool { return buf[i].key < buf[j].key })
  }
}

func BenchmarkSlicesSortFunc(b *testing.B) {
  src := data(N)
  buf := make([]item, N)
  for b.Loop() {
    copy(buf, src)
    slices.SortFunc(buf, func(x, y item) int { return cmp.Compare(x.key, y.key) })
  }
}

type byKeyB []item

func (s byKeyB) Len() int           { return len(s) }
func (s byKeyB) Less(i, j int) bool { return s[i].key < s[j].key }
func (s byKeyB) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

func BenchmarkSortInterface(b *testing.B) {
  src := data(N)
  buf := make([]item, N)
  for b.Loop() {
    copy(buf, src)
    sort.Sort(byKeyB(buf))
  }
}
```

`sort.Slice` is about 1.6x slower than `slices.SortFunc` and allocates, because the
reflect-based swapper cannot be inlined. Hand-written `sort.Interface` is roughly as
fast as `slices.SortFunc` — the cost was never the interface, it was the reflection.
And `sort.Ints` matches `slices.Sort` exactly, because it *is* `slices.Sort`.

## Stability

`sort.Sort` and `slices.SortFunc` use pattern-defeating quicksort and reorder equal
elements freely. `sort.Stable` and `slices.SortStableFunc` do not.

```go
package main

import (
  "fmt"
  "sort"
)

type rec struct {
  key int
  seq int
}

type byKey []rec

func (r byKey) Len() int           { return len(r) }
func (r byKey) Less(i, j int) bool { return r[i].key < r[j].key }
func (r byKey) Swap(i, j int)      { r[i], r[j] = r[j], r[i] }

func main() {
  make30 := func() []rec {
    out := make([]rec, 0, 30)
    for i := 0; i < 30; i++ {
      out = append(out, rec{key: i % 2, seq: i})
    }
    return out
  }

  a := make30()
  sort.Sort(byKey(a))
  seqs := []int{}
  for _, r := range a {
    if r.key == 0 {
      seqs = append(seqs, r.seq)
    }
  }
  fmt.Println("Sort  ", seqs)

  b := make30()
  sort.Stable(byKey(b))
  seqs = seqs[:0]
  for _, r := range b {
    if r.key == 0 {
      seqs = append(seqs, r.seq)
    }
  }
  fmt.Println("Stable", seqs)
}
```

Output:

```
Sort   [6 10 2 26 4 24 28 8 16 22 0 20 12 18 14]
Stable [0 2 4 6 8 10 12 14 16 18 20 22 24 26 28]
```

Every element here has an equal key, and `sort.Sort` scrambled their original
sequence. Stability matters when you sort the same data by several keys in
succession (sort by name, then stably by department, and names stay ordered within
each department) and whenever output must be reproducible.

Stable sorts are slower — they do more work or allocate scratch space. Use the
unstable one unless you need the guarantee.

## Search

`sort.Search` is more general than `slices.BinarySearch`: it searches an abstract
index range with a monotone predicate, so the data need not be in a slice at all.

```go
package main

import (
  "fmt"
  "slices"
  "sort"
)

func main() {
  a := []int{10, 20, 30, 40}
  fmt.Println(sort.SearchInts(a, 30), sort.SearchInts(a, 25), sort.SearchInts(a, 99))
  fmt.Println(sort.SearchStrings([]string{"a", "c"}, "b"))
  fmt.Println(sort.SearchFloat64s([]float64{1.5, 2.5}, 2.0))

  fmt.Println(sort.Search(len(a), func(i int) bool { return a[i] >= 25 }))
  fmt.Println(sort.Search(100, func(i int) bool { return i*i >= 50 }))

  i, found := sort.Find(len(a), func(i int) int { return 30 - a[i] })
  fmt.Println(i, found)

  fmt.Println(slices.BinarySearch(a, 25))
}
```

Output:

```
2 2 4
1
1
2
8
2 true
2 false
```

`sort.Search(n, f)` returns the smallest `i` in `[0, n)` for which `f(i)` is true, or
`n` if none is. It does not tell you whether the element was found — you check
`i < n && a[i] == x` yourself. `sort.Find` (Go 1.19) is the three-way version that
does return `found`, and `slices.BinarySearch` is the generic one.

`sort.Search(100, func(i int) bool { return i*i >= 50 })` returning 8 is the point:
there is no slice. Use it for binary search over a numeric range, a paged API, or a
file offset.

## sort.Reverse is not slices.Reverse

`sort.Reverse(data)` returns a *new* `sort.Interface` whose `Less` is flipped — it
sorts nothing by itself. `slices.Reverse(s)` reverses the elements of a slice in
place, with no sorting. Different jobs, confusingly similar names.

```go
package main

import (
  "fmt"
  "slices"
  "sort"
  "strings"
)

type ciStrings []string

func (s ciStrings) Len() int { return len(s) }
func (s ciStrings) Less(i, j int) bool {
  return strings.ToLower(s[i]) < strings.ToLower(s[j])
}
func (s ciStrings) Swap(i, j int) { s[i], s[j] = s[j], s[i] }

func main() {
  words := []string{"banana", "Zebra", "apple", "Cherry"}

  a := slices.Clone(words)
  slices.Sort(a)
  fmt.Println(a)

  b := slices.Clone(words)
  sort.Sort(ciStrings(b))
  fmt.Println(b)

  c := slices.Clone(words)
  slices.SortFunc(c, func(x, y string) int {
    return strings.Compare(strings.ToLower(x), strings.ToLower(y))
  })
  fmt.Println(c)

  d := []string{"a", "c", "b"}
  sort.Sort(sort.Reverse(sort.StringSlice(d)))
  fmt.Println(d)

  e := []string{"a", "c", "b"}
  slices.Sort(e)
  slices.Reverse(e)
  fmt.Println(e)

  f := []string{"a", "c", "b"}
  slices.Reverse(f)
  fmt.Println(f)
}
```

Output:

```
[Cherry Zebra apple banana]
[apple banana Cherry Zebra]
[apple banana Cherry Zebra]
[c b a]
[c b a]
[b c a]
```

The first three lines are the collation point: `slices.Sort` is byte-wise, so every
capitalised word sorts before every lowercase one. A case-insensitive ordering needs
a custom comparator, whether via `sort.Interface` or `slices.SortFunc` — the latter
in three lines instead of nine. (For real human-language collation you want
`golang.org/x/text/collate`; byte order is not alphabetical order outside ASCII.)

The last three lines: `sort.Sort(sort.Reverse(…))` and `slices.Sort` +
`slices.Reverse` both give descending order. Bare `slices.Reverse` on unsorted input
just flips it. Idiomatic descending sort: `slices.SortFunc(s, func(a, b T) int { return cmp.Compare(b, a) })`
— flip the arguments, one pass.

## NaN in float sorts

`<` is not a strict weak ordering when NaN is present, which breaks every
comparison-based sort differently.

```go
package main

import (
  "fmt"
  "math"
  "slices"
  "sort"
)

func main() {
  f := []float64{3, math.NaN(), 1, math.Inf(1), math.Inf(-1), 2}
  slices.Sort(f)
  fmt.Println(f)

  g := []float64{3, math.NaN(), 1, 2}
  sort.Float64s(g)
  fmt.Println(g)

  h := []float64{3, math.NaN(), 1, 2}
  sort.Slice(h, func(i, j int) bool { return h[i] < h[j] })
  fmt.Println(h)

  fmt.Println(slices.Min([]float64{1, math.NaN(), 2}))
  fmt.Println(slices.Max([]float64{1, math.NaN(), 2}))
}
```

Output:

```
[NaN -Inf 1 2 3 +Inf]
[NaN 1 2 3]
[3 NaN 1 2]
NaN
NaN
```

`slices.Sort` uses `cmp.Less`, which defines NaN as less than everything, so NaN
sorts first — before `-Inf`. `sort.Float64s` delegates to `slices.Sort` and agrees.
But a hand-written `h[i] < h[j]` in `sort.Slice` produces `[3 NaN 1 2]`: not sorted
at all, because the predicate is inconsistent and the algorithm's invariants break.

`slices.Min` and `slices.Max` both return NaN whenever one is present — NaN
propagates, matching IEEE 754 but surprising most people. Filter NaN out before
sorting or reducing floats.

## Decision table

| Situation | Use |
|---|---|
| `[]int`, `[]string`, any `cmp.Ordered` slice | `slices.Sort(s)` |
| Slice of structs, one or many keys | `slices.SortFunc(s, cmp)` with `cmp.Or` |
| Equal elements must keep their order | `slices.SortStableFunc(s, cmp)` |
| Descending | `slices.SortFunc(s, func(a,b T) int { return cmp.Compare(b,a) })` |
| Collection that is not a slice | `sort.Sort` + `sort.Interface` |
| Ordering is part of the type's API | `sort.Interface` on a named type |
| Need to pass the ordering as a value | `sort.Interface`, with `sort.Reverse` to flip it |
| Existing code already has `sort.Interface` | leave it; it is as fast as `slices.SortFunc` |
| New code, tempted by `sort.Slice` | use `slices.SortFunc` instead |
| Binary search in a sorted slice | `slices.BinarySearch` |
| Binary search over an index range or external data | `sort.Search` / `sort.Find` |
| Sorting a map | `slices.Sorted(maps.Keys(m))` or collect entries and `slices.SortFunc` |

## Sorting a map's entries

```go
package main

import (
  "cmp"
  "fmt"
  "maps"
  "slices"
  "sort"
)

type entry struct {
  word  string
  count int
}

func main() {
  counts := map[string]int{"go": 5, "rust": 3, "zig": 5, "c": 1}

  entries := make([]entry, 0, len(counts))
  for w, c := range counts {
    entries = append(entries, entry{w, c})
  }
  slices.SortFunc(entries, func(a, b entry) int {
    return cmp.Or(cmp.Compare(b.count, a.count), cmp.Compare(a.word, b.word))
  })
  fmt.Println(entries)

  sort.Slice(entries, func(i, j int) bool {
    if entries[i].count != entries[j].count {
      return entries[i].count > entries[j].count
    }
    return entries[i].word < entries[j].word
  })
  fmt.Println(entries)

  keys := slices.SortedFunc(maps.Keys(counts), func(a, b string) int {
    return cmp.Or(cmp.Compare(counts[b], counts[a]), cmp.Compare(a, b))
  })
  fmt.Println(keys)
}
```

Output:

```
[{go 5} {zig 5} {rust 3} {c 1}]
[{go 5} {zig 5} {rust 3} {c 1}]
[go zig rust c]
```

All three agree. Note the tie-break on `word` is what makes the output
deterministic: `go` and `zig` both have count 5, and without the second key their
order would follow map iteration order and vary between runs. Idiomatic: always add
a total-order tie-break when sorting data that came out of a map.

## Quick Reference

| `sort` | `slices` / `cmp` equivalent |
|---|---|
| `sort.Ints(s)` | `slices.Sort(s)` (it already delegates) |
| `sort.Strings(s)` | `slices.Sort(s)` |
| `sort.Float64s(s)` | `slices.Sort(s)` |
| `sort.IntsAreSorted(s)` | `slices.IsSorted(s)` |
| `sort.Slice(s, less)` | `slices.SortFunc(s, cmp)` |
| `sort.SliceStable(s, less)` | `slices.SortStableFunc(s, cmp)` |
| `sort.SliceIsSorted(s, less)` | `slices.IsSortedFunc(s, cmp)` |
| `sort.SearchInts(s, x)` | `slices.BinarySearch(s, x)` |
| `sort.Find(n, cmp)` | `slices.BinarySearchFunc(s, t, cmp)` |
| `sort.Sort(data)` | no equivalent — keep it for non-slices |
| `sort.Stable(data)` | no equivalent |
| `sort.Reverse(data)` | flip the `cmp` arguments |
| `sort.IsSorted(data)` | no equivalent |

| Trap | Reality |
|---|---|
| `sort.Reverse` vs `slices.Reverse` | one wraps a comparator, one flips elements |
| `sort.Slice` closing over the wrong variable | the compiler cannot catch it |
| `sort.Sort` preserving equal-element order | it does not; use `sort.Stable` |
| `h[i] < h[j]` with NaN | inconsistent predicate, arbitrary output |
| `slices.Min` on floats with NaN | returns NaN |
| Sorting map data without a tie-break | order varies between runs |

## Common Mistakes

**Reaching for `sort.Slice` in new code.** It is reflect-based and 1.6x slower;
`slices.SortFunc` is type-safe and allocation-free.

**Confusing `sort.Reverse` with `slices.Reverse`.** The first wraps an ordering, the
second mutates a slice; for a descending sort flip the `cmp` arguments.

**Using `sort.Sort` and expecting stable output.** Equal elements are reordered
arbitrarily; use `sort.Stable` or `slices.SortStableFunc`.

**Writing a `less` with `<` over floats that may be NaN.** The predicate is
inconsistent and the result is not sorted; filter NaN first.

**Sorting entries pulled from a map by one key only.** Ties resolve by map iteration
order, so the output varies; add a total-order tie-break.

## Exercises

1. Implement `sort.Interface` for a type holding two parallel slices (`names` and
   `scores`) so that sorting by score keeps the names aligned. Show why
   `slices.SortFunc` cannot do this directly.
2. Convert a `sort.Slice` call with three sort keys to `slices.SortFunc` with
   `cmp.Or`, and benchmark both at 100,000 elements.
3. Write `func searchPage(total int, fetch func(i int) int, target int) (int, bool)`
   using `sort.Search` over a paged data source, and test it against a local slice.
4. Demonstrate that `slices.SortFunc` is unstable and `slices.SortStableFunc` is,
   using a comparator over a key with many duplicates. Make the test deterministic.
5. Write `func sortFloats(s []float64) []float64` that moves NaN to the end rather
   than the front (the opposite of `slices.Sort`), and prove it with `math.IsNaN`.

---

[← 7.9 The maps Package Reference](09-maps-package.md) · [Course Index](../README.md) · [8.1 The error Interface and errors.New →](../08-errors/01-error-interface.md)
