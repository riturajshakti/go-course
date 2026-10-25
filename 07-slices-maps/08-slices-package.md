[← 7.7 Map Keys, Nested Maps, and Sets](07-map-patterns.md) · [Course Index](../README.md) · [7.9 The maps Package Reference →](09-maps-package.md)

# 7.8 — The slices Package Reference

## What You'll Learn

- Every function in `slices` as of go1.27.1, grouped by purpose, with real output
- `SortFunc` takes a three-way `cmp` returning negative/zero/positive, not a bool `less`
- `Compact` only removes *consecutive* duplicates, so sort first
- `Min`/`Max` panic on an empty slice; `BinarySearch` silently lies on an unsorted one

## Coming From Another Language

| Other | Go |
|---|---|
| JS `arr.indexOf` / `includes` | `slices.Index` / `slices.Contains` |
| JS `arr.sort((a,b) => a-b)` | `slices.SortFunc(s, func(a,b int) int { return a-b })` — same convention |
| Python `sorted(xs, key=f)` | no key function; write a `cmp` |
| Python `xs.sort()` in place vs `sorted()` | `slices.Sort` in place; `slices.Sorted` takes an *iterator* |
| Rust `v.sort_by(|a,b| a.cmp(b))` | `slices.SortFunc`, same `Ordering` shape |
| Rust `v.binary_search(&x)` | `slices.BinarySearch(s, x)` returns `(int, bool)` |

`slices` arrived in Go 1.21. `All`, `Backward`, `Values`, `Collect`, `AppendSeq`,
`Sorted`, `SortedFunc`, `SortedStableFunc`, `Repeat`, `Concat` and `Chunk` are newer
(1.22–1.23) and depend on the `iter` package; everything else was in the original
1.21 release. All signatures below were read from `go doc slices` on go1.27.1.

Every function is generic over `S ~[]E`, so it works on a named slice type like
`type IDs []int` and returns that same type.

## Searching

| Function | Signature | Returns |
|---|---|---|
| `Contains` | `(s S, v E) bool` | `E` must be `comparable` |
| `ContainsFunc` | `(s S, f func(E) bool) bool` | any `E` |
| `Index` | `(s S, v E) int` | first index, or `-1` |
| `IndexFunc` | `(s S, f func(E) bool) int` | first match, or `-1` |
| `BinarySearch` | `(x S, target E) (int, bool)` | index + found; **requires sorted** |
| `BinarySearchFunc` | `(x S, target T, cmp func(E, T) int) (int, bool)` | target may be a different type |

```go
package main

import (
  "fmt"
  "slices"
  "strings"
)

func main() {
  s := []string{"ann", "bob", "cat", "bob"}
  fmt.Println(slices.Contains(s, "bob"), slices.Contains(s, "zoe"))
  fmt.Println(slices.ContainsFunc(s, func(v string) bool { return strings.HasPrefix(v, "c") }))
  fmt.Println(slices.Index(s, "bob"), slices.Index(s, "zoe"))
  fmt.Println(slices.IndexFunc(s, func(v string) bool { return len(v) > 3 }))

  sorted := []int{10, 20, 30, 40}
  fmt.Println(slices.BinarySearch(sorted, 30))
  fmt.Println(slices.BinarySearch(sorted, 25))
  fmt.Println(slices.BinarySearch(sorted, 1))
  fmt.Println(slices.BinarySearch(sorted, 99))

  type p struct {
    name string
    age  int
  }
  ps := []p{{"ann", 25}, {"bob", 30}, {"cat", 40}}
  i, found := slices.BinarySearchFunc(ps, 30, func(e p, t int) int { return e.age - t })
  fmt.Println(i, found)
}
```

Output:

```
true false
true
1 -1
-1
2 true
2 false
0 false
4 false
1 true
```

`BinarySearch` returns the **insertion point** when not found: 25 belongs at index 2,
1 at index 0, 99 at index 4 (== `len`). That makes it a one-call
insert-in-sorted-order primitive paired with `slices.Insert`.

Note `BinarySearchFunc`'s comparator takes `(element, target)` in that order and the
target can be a different type — here `int` against a `p`. That is how you search by
one field without building a throwaway struct.

On an unsorted slice it does not error. It returns garbage.

```go
package main

import (
  "fmt"
  "slices"
)

func main() {
  unsorted := []int{5, 1, 9, 3, 7}
  fmt.Println(slices.BinarySearch(unsorted, 1))
  fmt.Println(slices.BinarySearch(unsorted, 9))
  fmt.Println(slices.Contains(unsorted, 1), slices.Index(unsorted, 1))

  sorted := slices.Clone(unsorted)
  slices.Sort(sorted)
  fmt.Println(sorted)
  fmt.Println(slices.BinarySearch(sorted, 1))
  fmt.Println(slices.BinarySearch(sorted, 9))
}
```

Output:

```
0 false
2 true
true 1
[1 3 5 7 9]
0 true
4 true
```

`1` is in the slice at index 1, and `BinarySearch` reported `0 false`. `Contains`
and `Index` are the linear-scan alternatives that are always correct.

## Comparing

| Function | Signature | Returns |
|---|---|---|
| `Equal` | `(s1, s2 S) bool` | element-wise; nil == empty |
| `EqualFunc` | `(s1 S1, s2 S2, eq func(E1, E2) bool) bool` | cross-type |
| `Compare` | `(s1, s2 S) int` | lexicographic: `-1`, `0`, `1` |
| `CompareFunc` | `(s1 S1, s2 S2, cmp func(E1, E2) int) int` | cross-type |

```go
package main

import (
  "fmt"
  "slices"
  "strings"
)

func main() {
  fmt.Println(slices.Equal([]int{1, 2}, []int{1, 2}))
  fmt.Println(slices.Equal([]int(nil), []int{}))
  fmt.Println(slices.Equal([]int{1}, []int{1, 2}))
  fmt.Println(slices.EqualFunc([]string{"A"}, []string{"a"}, strings.EqualFold))

  fmt.Println(slices.Compare([]int{1, 2}, []int{1, 3}))
  fmt.Println(slices.Compare([]int{1, 3}, []int{1, 2}))
  fmt.Println(slices.Compare([]int{1, 2}, []int{1, 2}))
  fmt.Println(slices.Compare([]int{1, 2}, []int{1, 2, 3}))
  fmt.Println(slices.Compare([]int(nil), []int{}))

  fmt.Println(slices.CompareFunc([]int{1, 2}, []string{"1", "22"},
    func(a int, b string) int { return a - len(b) }))
}
```

Output:

```
true
true
false
true
-1
1
0
-1
0
0
```

`Equal` and `Compare` both treat nil and empty as identical — unlike `==` on the
headers. `Compare` compares element by element and then by length, so a prefix sorts
before the longer slice.

## Sorting

| Function | Signature | Note |
|---|---|---|
| `Sort` | `(x S)` | in place, `E cmp.Ordered`, not stable |
| `SortFunc` | `(x S, cmp func(a, b E) int)` | in place, not stable |
| `SortStableFunc` | `(x S, cmp func(a, b E) int)` | in place, stable |
| `IsSorted` | `(x S) bool` | |
| `IsSortedFunc` | `(x S, cmp func(a, b E) int) bool` | |
| `Sorted` | `(seq iter.Seq[E]) []E` | collects an **iterator** and sorts |
| `SortedFunc` | `(seq iter.Seq[E], cmp func(E, E) int) []E` | |
| `SortedStableFunc` | `(seq iter.Seq[E], cmp func(E, E) int) []E` | |
| `Min` / `Max` | `(x S) E` | **panics on empty** |
| `MinFunc` / `MaxFunc` | `(x S, cmp func(a, b E) int) E` | **panics on empty** |

```go
package main

import (
  "cmp"
  "fmt"
  "slices"
)

type p struct {
  name string
  dept string
  age  int
}

func main() {
  a := []int{3, 1, 2}
  slices.Sort(a)
  fmt.Println(a, slices.IsSorted(a))

  s := []string{"b", "A", "a"}
  slices.Sort(s)
  fmt.Println(s)

  ps := []p{{"cat", "ops", 25}, {"ann", "eng", 40}, {"bob", "eng", 30}}
  slices.SortFunc(ps, func(x, y p) int { return cmp.Compare(x.age, y.age) })
  fmt.Println(ps)

  slices.SortFunc(ps, func(x, y p) int {
    return cmp.Or(
      cmp.Compare(x.dept, y.dept),
      cmp.Compare(y.age, x.age),
    )
  })
  fmt.Println(ps)

  fmt.Println(slices.IsSortedFunc(ps, func(x, y p) int { return cmp.Compare(x.dept, y.dept) }))

  fmt.Println(slices.Min([]int{3, 1, 2}), slices.Max([]int{3, 1, 2}))
  fmt.Println(slices.MinFunc(ps, func(x, y p) int { return cmp.Compare(x.age, y.age) }).name)
  fmt.Println(slices.MaxFunc(ps, func(x, y p) int { return cmp.Compare(x.age, y.age) }).name)

  fmt.Println(cmp.Compare(1, 2), cmp.Compare(2, 1), cmp.Compare(1, 1))
  fmt.Println(cmp.Or(0, 0, -1, 5), cmp.Or("", "x", "y"), cmp.Or(0, 0))
  fmt.Println(cmp.Less(1, 2))
}
```

Output:

```
[1 2 3] true
[A a b]
[{cat ops 25} {bob eng 30} {ann eng 40}]
[{ann eng 40} {bob eng 30} {cat ops 25}]
true
1 3
cat
ann
-1 1 0
-1 x 0
true
```

`slices.Sort` on strings is byte-wise, so `"A"` (0x41) precedes `"a"` (0x61) — not a
locale collation.

The `cmp` package is the other half of this API. `cmp.Compare(a, b)` is the
three-way comparison you almost always want, and `cmp.Or(vs...)` returns the first
non-zero argument — which is exactly multi-key sorting: compare by dept ascending,
then by age *descending* by flipping the arguments. `cmp.Less` exists but is rarely
what a sort wants.

### SortFunc takes a cmp, not a less

This is the one real trap for anyone who knows `sort.Slice`.

```go
package main

import (
  "cmp"
  "fmt"
  "slices"
)

type p struct {
  name string
  age  int
}

func main() {
  stable := []p{{"a", 1}, {"b", 1}, {"c", 0}}
  slices.SortStableFunc(stable, func(x, y p) int { return cmp.Compare(x.age, y.age) })
  fmt.Println(stable)

  byLess := []int{3, 1, 2}
  slices.SortFunc(byLess, func(x, y int) int {
    if x < y {
      return -1
    }
    return 1
  })
  fmt.Println(byLess)
}
```

Output:

```
[{c 0} {a 1} {b 1}]
[1 2 3]
```

The second comparator never returns 0 — it is a bool `less` translated naively into
`-1`/`1`. It happens to sort correctly here, but it violates the contract (equal
elements must compare 0), which makes `SortStableFunc` non-stable and
`BinarySearchFunc` wrong. `sort.Slice` wanted `func(i, j int) bool`; `slices.SortFunc`
wants a three-way int. Always write `cmp.Compare(x.f, y.f)`.

`SortStableFunc` preserved `a` before `b` among the age-1 entries. `SortFunc` gives
no such guarantee, and uses pattern-defeating quicksort, which is faster.

### Min and Max panic on empty

```go
package main

import (
  "fmt"
  "slices"
)

func main() {
  fmt.Println(slices.Min([]int{}))
}
```

This program panics on purpose. Output:

```
panic: slices.Min: empty list

goroutine 1 [running]:
slices.Min[...](...)
	/usr/local/go/src/slices/sort.go:67
main.main()
	/tmp/scratch/main.go:9 +0x30
exit status 2
```

`slices.Max([]int(nil))` gives `panic: slices.Max: empty list` the same way. There
is no `(E, bool)` variant — guard with `len(s) > 0` yourself.

## Modifying

| Function | Signature | Note |
|---|---|---|
| `Insert` | `(s S, i int, v ...E) S` | shifts right |
| `Delete` | `(s S, i, j int) S` | half-open; zeroes the vacated tail |
| `DeleteFunc` | `(s S, del func(E) bool) S` | zeroes the tail |
| `Replace` | `(s S, i, j int, v ...E) S` | replaces `s[i:j]` with `v` |
| `Reverse` | `(s S)` | in place, returns nothing |
| `Compact` | `(s S) S` | removes **consecutive** duplicates |
| `CompactFunc` | `(s S, eq func(E, E) bool) S` | |
| `Clip` | `(s S) S` | `s[:len:len]`; trims cap |
| `Grow` | `(s S, n int) S` | ensures room for `n` more |
| `Clone` | `(s S) S` | shallow copy; nil in, nil out |
| `Concat` | `(slices ...S) S` | always a fresh array |
| `Repeat` | `(x S, count int) S` | `count` must be `>= 0` |

```go
package main

import (
  "fmt"
  "slices"
  "strings"
)

func main() {
  fmt.Println(slices.Insert([]int{1, 2, 5}, 2, 3, 4))
  fmt.Println(slices.Delete([]int{1, 2, 3, 4}, 1, 3))
  fmt.Println(slices.DeleteFunc([]int{1, 2, 3, 4}, func(v int) bool { return v%2 == 0 }))
  fmt.Println(slices.Replace([]int{1, 2, 3, 4}, 1, 3, 9))
  fmt.Println(slices.Replace([]int{1, 2}, 1, 1, 8, 9))

  r := []int{1, 2, 3}
  slices.Reverse(r)
  fmt.Println(r)

  fmt.Println(slices.Compact([]int{1, 1, 2, 2, 2, 3, 1}))
  fmt.Println(slices.Compact([]int{3, 1, 3, 1}))
  dup := []int{3, 1, 3, 1}
  slices.Sort(dup)
  fmt.Println(slices.Compact(dup))

  fmt.Println(slices.CompactFunc([]string{"A", "a", "B"}, strings.EqualFold))

  g := slices.Grow([]int{1}, 100)
  fmt.Println(len(g), cap(g) >= 101)
  fmt.Println(slices.Clip(g[:1]), cap(slices.Clip(g[:1])))

  c := slices.Clone([]int{1, 2})
  fmt.Println(c, cap(c))
  fmt.Println(slices.Clone([]int(nil)) == nil)

  fmt.Println(slices.Concat([]int{1}, []int{2, 3}, nil, []int{4}))
  fmt.Println(slices.Repeat([]int{1, 2}, 3))
  fmt.Println(slices.Repeat([]string{"x"}, 0))
}
```

Output:

```
[1 2 3 4 5]
[1 4]
[1 3]
[1 9 4]
[1 8 9 2]
[3 2 1]
[1 2 3 1]
[3 1 3 1]
[1 3]
[A B]
1 true
[1] 1
[1 2] 2
true
[1 2 3 4]
[1 2 1 2 1 2]
[]
```

Two things to read carefully.

`Compact([]int{1,1,2,2,2,3,1})` gives `[1 2 3 1]` — the trailing 1 survives because
it is not adjacent to the leading 1s. `Compact([]int{3,1,3,1})` changes nothing at
all. Sort first and you get a real dedupe: `[1 3]`. Idiomatic dedupe:
`slices.Sort(s); s = slices.Compact(s)`.

`Replace(s, 1, 1, 8, 9)` with `i == j` is an insertion at index 1 — the empty range
is replaced by two elements, giving `[1 8 9 2]`.

All of `Insert`, `Delete`, `DeleteFunc`, `Replace`, `Compact`, `Grow` and `Clip`
mutate `s`'s backing array *and* return a new header. Use the return value and stop
using the argument. `Concat` and `Clone` always allocate fresh, so they never alias.

## Iterators

| Function | Signature | Yields |
|---|---|---|
| `All` | `(s Slice) iter.Seq2[int, E]` | index, value |
| `Backward` | `(s Slice) iter.Seq2[int, E]` | index, value, reversed |
| `Values` | `(s Slice) iter.Seq[E]` | values only |
| `Collect` | `(seq iter.Seq[E]) []E` | a slice from any sequence |
| `AppendSeq` | `(s Slice, seq iter.Seq[E]) Slice` | appends a sequence |
| `Chunk` | `(s Slice, n int) iter.Seq[Slice]` | sub-slices of up to `n` |

```go
package main

import (
  "fmt"
  "maps"
  "slices"
)

func main() {
  s := []string{"a", "b", "c"}

  for i, v := range slices.All(s) {
    fmt.Print(i, v, " ")
  }
  fmt.Println()

  for i, v := range slices.Backward(s) {
    fmt.Print(i, v, " ")
  }
  fmt.Println()

  for v := range slices.Values(s) {
    fmt.Print(v, " ")
  }
  fmt.Println()

  m := map[string]int{"b": 2, "a": 1, "c": 3}
  fmt.Println(slices.Sorted(maps.Keys(m)))
  fmt.Println(slices.Collect(maps.Keys(m)) != nil)
  fmt.Println(slices.SortedFunc(maps.Keys(m), func(x, y string) int { return int(y[0]) - int(x[0]) }))

  fmt.Println(slices.AppendSeq([]string{"z"}, slices.Values(s)))

  for chunk := range slices.Chunk([]int{1, 2, 3, 4, 5}, 2) {
    fmt.Print(chunk, " ")
  }
  fmt.Println()
}
```

Output:

```
0a 1b 2c 
2c 1b 0a 
a b c 
[a b c]
true
[c b a]
[z a b c]
[1 2] [3 4] [5] 
```

`slices.All(s)` is pointless over a plain slice — `range s` already does that. These
exist so a slice can feed a pipeline that speaks `iter.Seq`. `Backward` is the
genuinely useful one: Go has no reverse-range syntax.

`slices.Sorted(maps.Keys(m))` is the single most common use of this group and the
canonical answer to "iterate a map in a stable order". The chunks `Chunk` yields are
sub-slices of the original — they alias it, so do not append to them. See 13.14 for
`iter` itself.

## Quick Reference

| Need | Call |
|---|---|
| Does it contain x? | `slices.Contains(s, x)` |
| Where is x? | `slices.Index(s, x)` |
| Sort | `slices.Sort(s)` / `slices.SortFunc(s, cmp)` |
| Stable sort | `slices.SortStableFunc(s, cmp)` |
| Multi-key sort | `cmp.Or(cmp.Compare(...), cmp.Compare(...))` |
| Deep-ish equality | `slices.Equal(a, b)` |
| Dedupe | `slices.Sort(s); s = slices.Compact(s)` |
| Copy | `slices.Clone(s)` |
| Concatenate | `slices.Concat(a, b)` |
| Insert / delete | `slices.Insert` / `slices.Delete` |
| Filter | `slices.DeleteFunc(s, drop)` |
| Reverse | `slices.Reverse(s)` |
| Min / max | `slices.Min(s)` / `slices.Max(s)` after a `len` check |
| Lookup in sorted data | `slices.BinarySearch(s, x)` |
| Trim capacity | `slices.Clip(s)` |
| Preallocate | `slices.Grow(s, n)` |
| Iterate backwards | `for i, v := range slices.Backward(s)` |
| Map keys, sorted | `slices.Sorted(maps.Keys(m))` |
| Batch | `for c := range slices.Chunk(s, n)` |

| Trap | Reality |
|---|---|
| `SortFunc` with a bool `less` | wants three-way int; `cmp.Compare` |
| `Compact` on unsorted input | only consecutive duplicates removed |
| `BinarySearch` on unsorted input | wrong answer, no error |
| `Min`/`Max` on empty | `panic: slices.Min: empty list` |
| `Clip` to release memory | only trims `cap`; use `Clone` |
| Ignoring the return of `Delete`/`Insert` | the argument's backing array was mutated anyway |
| Appending to a `Chunk` sub-slice | it aliases the source |

## Common Mistakes

**Passing a bool `less` to `SortFunc`.** It needs negative/zero/positive; wrap your
field comparison in `cmp.Compare`.

**Calling `slices.Compact` to deduplicate.** It only collapses adjacent runs; sort
first.

**Using `slices.BinarySearch` on data you did not sort.** It returns a plausible
wrong answer; use `slices.Index` or sort first.

**Calling `slices.Min` on a possibly-empty slice.** It panics with no comparable
alternative; check `len` first.

**Ignoring the result of `slices.Delete`.** The source array is mutated regardless,
so the old header now shows stale data.

## Exercises

1. Write `func topN[T any](s []T, n int, cmp func(a, b T) int) []T` returning the n
   largest, using only `slices` functions, and without mutating the input.
2. Sort a `[]struct{ last, first string; age int }` by last name, then first name,
   then age descending, in one `cmp.Or`. Add a test asserting the full order.
3. Implement `func dedupeStable[T comparable](s []T) []T` that removes duplicates
   while preserving first-appearance order. Benchmark it against
   `slices.Sort` + `slices.Compact` and explain the trade-off.
4. Write `func insertSorted[T cmp.Ordered](s []T, v T) []T` with `BinarySearch` plus
   `Insert`, and a fuzz test asserting `slices.IsSorted` after every insertion.
5. Implement `func windows[T any](s []T, n int) iter.Seq[[]T]` yielding every
   overlapping window of length `n`, and contrast it with `slices.Chunk`.

---

[← 7.7 Map Keys, Nested Maps, and Sets](07-map-patterns.md) · [Course Index](../README.md) · [7.9 The maps Package Reference →](09-maps-package.md)
