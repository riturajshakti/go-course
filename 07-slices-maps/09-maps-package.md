[← 7.8 The slices Package Reference](08-slices-package.md) · [Course Index](../README.md) · [7.10 sort: Before and Alongside Generics →](10-sort.md)

# 7.9 — The maps Package Reference

## What You'll Learn

- All ten functions in `maps` as of go1.27.1, with verified signatures
- `maps.Keys` and `maps.Values` return `iter.Seq`, **not** slices — the `x/exp/maps` habit does not port
- `slices.Sorted(maps.Keys(m))` is the canonical stable-iteration idiom
- `maps.Clone` is shallow: nested maps and slice values stay aliased

## Coming From Another Language

| Other | Go |
|---|---|
| JS `Object.keys(o)` → array | `maps.Keys(m)` → an iterator; wrap in `slices.Collect` |
| JS `Object.assign(a, b)` | `maps.Copy(a, b)` |
| JS `{...a, ...b}` | `c := maps.Clone(a); maps.Copy(c, b)` |
| Python `d.keys()` — a view | `maps.Keys(m)` — closest analogue, but single-pass |
| Python `d.update(e)` | `maps.Copy(d, e)` |
| Python `dict(d)` / `d.copy()` | `maps.Clone(d)` — also shallow |
| Python `{k: v for ...}` filtering | `maps.DeleteFunc(m, drop)` in place |

The whole package, from `go doc maps` on go1.27.1:

```
func All[Map ~map[K]V, K comparable, V any](m Map) iter.Seq2[K, V]
func Clone[M ~map[K]V, K comparable, V any](m M) M
func Collect[K comparable, V any](seq iter.Seq2[K, V]) map[K]V
func Copy[M1 ~map[K]V, M2 ~map[K]V, K comparable, V any](dst M1, src M2)
func DeleteFunc[M ~map[K]V, K comparable, V any](m M, del func(K, V) bool)
func Equal[M1, M2 ~map[K]V, K, V comparable](m1 M1, m2 M2) bool
func EqualFunc[M1 ~map[K]V1, M2 ~map[K]V2, K comparable, V1, V2 any](m1 M1, m2 M2, eq func(V1, V2) bool) bool
func Insert[Map ~map[K]V, K comparable, V any](m Map, seq iter.Seq2[K, V])
func Keys[Map ~map[K]V, K comparable, V any](m Map) iter.Seq[K]
func Values[Map ~map[K]V, K comparable, V any](m Map) iter.Seq[V]
```

Ten functions. `Clone`, `Copy`, `DeleteFunc`, `Equal` and `EqualFunc` shipped in Go
1.21; `All`, `Keys`, `Values`, `Collect` and `Insert` landed in 1.23 with `iter`.

Note the package doc's own warning: `maps` does nothing special about non-reflexive
keys such as NaN. `maps.Clone` of a map with NaN keys copies them, and `maps.Equal`
will report two such maps unequal; see [7.7](07-map-patterns.md).

## Keys and Values return iterators

This is the single thing that trips people up. `golang.org/x/exp/maps.Keys`
returned `[]K`. The stdlib version returns `iter.Seq[K]`.

```go
package main

import (
  "fmt"
  "maps"
)

func main() {
  m := map[string]int{"a": 1, "b": 2}
  var ks []string = maps.Keys(m)
  fmt.Println(ks)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:21: cannot use maps.Keys(m) (value of func type iter.Seq[string]) as []string value in variable declaration
```

`len` fails too, since there is nothing to count without iterating:

```go
package main

import (
  "fmt"
  "maps"
)

func main() {
  m := map[string]int{"a": 1, "b": 2}
  fmt.Println(len(maps.Keys(m)))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:19: invalid argument: maps.Keys(m) (value of func type iter.Seq[string]) for built-in len
```

`iter.Seq[K]` is literally `func(yield func(K) bool)`. You either range over it or
hand it to something that collects it.

```go
package main

import (
  "fmt"
  "maps"
  "slices"
)

func main() {
  m := map[string]int{"c": 3, "a": 1, "b": 2}

  fmt.Printf("%T\n", maps.Keys(m))
  fmt.Printf("%T\n", maps.Values(m))
  fmt.Printf("%T\n", maps.All(m))

  fmt.Println(slices.Sorted(maps.Keys(m)))
  fmt.Println(slices.Sorted(maps.Values(m)))

  for _, k := range slices.Sorted(maps.Keys(m)) {
    fmt.Print(k, "=", m[k], " ")
  }
  fmt.Println()

  ks := slices.Collect(maps.Keys(m))
  slices.Sort(ks)
  fmt.Println(ks, len(ks))

  for k := range maps.Keys(m) {
    if k == "a" {
      fmt.Println("found a")
    }
  }

  total := 0
  for _, v := range maps.All(m) {
    total += v
  }
  fmt.Println(total)
}
```

Output:

```
iter.Seq[string]
iter.Seq[int]
iter.Seq2[string,int]
[a b c]
[1 2 3]
a=1 b=2 c=3 
[a b c] 3
found a
6
```

| Want | Call |
|---|---|
| Keys as a sorted slice | `slices.Sorted(maps.Keys(m))` |
| Keys as an unsorted slice | `slices.Collect(maps.Keys(m))` |
| Keys sorted by a custom rule | `slices.SortedFunc(maps.Keys(m), cmp)` |
| Range the keys lazily | `for k := range maps.Keys(m)` |
| Range key and value | `for k, v := range maps.All(m)` — or just `range m` |
| Values as a slice | `slices.Collect(maps.Values(m))` |

`maps.All(m)` and `range m` are equivalent; `All` exists so a map can be the source
of an `iter.Seq2` pipeline. Neither gives a stable order on its own.

## Deterministic iteration

`range m` is randomized ([7.6](06-maps.md)). Sorting the keys is how you get
reproducible output, reproducible tests, and reproducible hashes.

```go
package main

import (
  "fmt"
  "maps"
  "slices"
)

func main() {
  stock := map[string]int{"pear": 4, "apple": 7, "fig": 1}
  for _, k := range slices.Sorted(maps.Keys(stock)) {
    fmt.Printf("%-6s %d\n", k, stock[k])
  }

  byCount := slices.SortedFunc(maps.Keys(stock), func(a, b string) int {
    return stock[b] - stock[a]
  })
  fmt.Println(byCount)
}
```

Output:

```
apple  7
fig    1
pear   4
[apple pear fig]
```

Idiomatic: `slices.Sorted(maps.Keys(m))`, every time you print or serialise a map.

## Clone is shallow

`maps.Clone` copies the top-level key/value pairs. If `V` is itself a map, a slice,
or a pointer, the copy shares it.

```go
package main

import (
  "fmt"
  "maps"
  "slices"
)

func main() {
  orig := map[string]int{"a": 1, "b": 2}
  c := maps.Clone(orig)
  c["a"] = 99
  c["new"] = 3
  fmt.Println(orig, c)

  var nilMap map[string]int
  fmt.Println(maps.Clone(nilMap) == nil)

  nested := map[string]map[string]int{"outer": {"inner": 1}}
  shallow := maps.Clone(nested)
  shallow["outer"]["inner"] = 99
  fmt.Println(nested["outer"]["inner"], shallow["outer"]["inner"])
  shallow["other"] = map[string]int{}
  fmt.Println(len(nested), len(shallow))

  withSlice := map[string][]int{"k": {1, 2}}
  cs := maps.Clone(withSlice)
  cs["k"][0] = 99
  fmt.Println(withSlice["k"], cs["k"])

  deep := map[string]map[string]int{}
  for k, v := range nested {
    deep[k] = maps.Clone(v)
  }
  deep["outer"]["inner"] = 1
  fmt.Println(nested["outer"]["inner"], deep["outer"]["inner"])

  fmt.Println(slices.Sorted(maps.Keys(c)))
}
```

Output:

```
map[a:1 b:2] map[a:99 b:2 new:3]
true
99 99
1 2
[99 2] [99 2]
99 1
[a b new]
```

For `map[string]int` the clone is fully independent. For `map[string]map[string]int`
it is not: writing `shallow["outer"]["inner"] = 99` changed `nested` too. Adding a
new *outer* key only touched `shallow`, because that is the level that was copied.
Same story for `map[string][]int`.

`maps.Clone(nil)` returns nil, not an empty map — one of the few places the
distinction leaks out of the `maps` package.

A deep clone needs one level of manual recursion per nesting level, as the `deep`
loop above shows.

## Copy, DeleteFunc, Equal

```go
package main

import (
  "fmt"
  "maps"
  "slices"
)

func main() {
  dst := map[string]int{"a": 1, "b": 2}
  src := map[string]int{"b": 20, "c": 30}
  maps.Copy(dst, src)
  fmt.Println(slices.Sorted(maps.Keys(dst)), dst["b"], dst["c"], len(dst))

  m := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4}
  maps.DeleteFunc(m, func(k string, v int) bool { return v%2 == 0 })
  fmt.Println(slices.Sorted(maps.Keys(m)))

  fmt.Println(maps.Equal(map[string]int{"a": 1}, map[string]int{"a": 1}))
  fmt.Println(maps.Equal(map[string]int{"a": 1}, map[string]int{"a": 2}))
  fmt.Println(maps.Equal(map[string]int(nil), map[string]int{}))
  fmt.Println(maps.EqualFunc(
    map[string]int{"a": 1},
    map[string]string{"a": "x"},
    func(a int, b string) bool { return a == len(b) },
  ))
}
```

Output:

```
[a b c] 20 30 3
[a c]
true
false
true
true
```

`maps.Copy(dst, src)` merges, with `src` winning on conflicts — `dst["b"]` became 20.
It writes to `dst`, so `dst` must not be nil.

`maps.DeleteFunc` mutates in place and returns nothing. Deleting during the
underlying iteration is the defined case from [7.6](06-maps.md), so this is safe.

`maps.Equal` requires `V comparable`, so it will not compile for `map[string][]int` —
use `maps.EqualFunc(a, b, slices.Equal)` there. Like `slices.Equal`, it treats nil
and empty as equal. It is shallow: two maps whose values are equal *maps* compare
unequal under `Equal` (maps are not comparable) and must go through `EqualFunc`.

## Collect, Insert, and building maps

`Collect` turns an `iter.Seq2` into a fresh map; `Insert` merges one into an existing
map.

```go
package main

import (
  "fmt"
  "maps"
  "slices"
)

func zip[K comparable, V any](ks []K, vs []V) map[K]V {
  m := make(map[K]V, min(len(ks), len(vs)))
  for i := 0; i < len(ks) && i < len(vs); i++ {
    m[ks[i]] = vs[i]
  }
  return m
}

func main() {
  ks := []string{"a", "b", "c"}
  vs := []int{1, 2, 3}
  m := zip(ks, vs)
  fmt.Println(slices.Sorted(maps.Keys(m)), m["b"])

  collected := maps.Collect(maps.All(m))
  fmt.Println(maps.Equal(m, collected))

  target := map[string]int{"z": 0}
  maps.Insert(target, maps.All(m))
  fmt.Println(slices.Sorted(maps.Keys(target)), len(target))

  inv := map[int]string{}
  for k, v := range m {
    inv[v] = k
  }
  fmt.Println(slices.Sorted(maps.Keys(inv)), inv[2])

  dupes := map[string]int{"a": 1, "b": 1, "c": 2}
  bad := map[int]string{}
  for k, v := range dupes {
    bad[v] = k
  }
  fmt.Println(len(dupes), len(bad))

  good := map[int][]string{}
  for _, k := range slices.Sorted(maps.Keys(dupes)) {
    good[dupes[k]] = append(good[dupes[k]], k)
  }
  fmt.Println(good)
}
```

Output:

```
[a b c] 2
true
[a b c z] 4
[1 2 3] b
3 2
map[1:[a b] 2:[c]]
```

`maps.Collect(maps.All(m))` is a verbose `maps.Clone(m)`; it earns its keep when the
sequence comes from somewhere else — a database cursor, a `slices.All` pipeline, a
generator.

Inversion is where the collision shows up. `dupes` has three keys but only two
distinct values, so the naive inversion lost an entry — and *which* one it lost
depends on map iteration order, so the result is not even deterministic. Invert to
`map[V][]K` when values may repeat, and sort the keys if you want a stable grouping.

## Quick Reference

| Function | Mutates | Returns | Note |
|---|---|---|---|
| `maps.Keys(m)` | no | `iter.Seq[K]` | wrap in `slices.Sorted`/`Collect` |
| `maps.Values(m)` | no | `iter.Seq[V]` | duplicates kept |
| `maps.All(m)` | no | `iter.Seq2[K, V]` | same as `range m` |
| `maps.Collect(seq)` | no | `map[K]V` | fresh map |
| `maps.Insert(m, seq)` | `m` | — | merge, `seq` wins |
| `maps.Clone(m)` | no | `M` | **shallow**; nil in, nil out |
| `maps.Copy(dst, src)` | `dst` | — | merge, `src` wins |
| `maps.DeleteFunc(m, f)` | `m` | — | filter in place |
| `maps.Equal(a, b)` | no | `bool` | needs `V comparable`; nil == empty |
| `maps.EqualFunc(a, b, eq)` | no | `bool` | cross-type values |

| Recipe | Code |
|---|---|
| Sorted keys | `slices.Sorted(maps.Keys(m))` |
| Keys slice | `slices.Collect(maps.Keys(m))` |
| Values slice | `slices.Collect(maps.Values(m))` |
| Merge b into a | `maps.Copy(a, b)` |
| Shallow copy | `maps.Clone(m)` |
| Deep copy, one level | `for k, v := range m { out[k] = maps.Clone(v) }` |
| Filter | `maps.DeleteFunc(m, func(k K, v V) bool { … })` |
| Compare `map[K][]T` | `maps.EqualFunc(a, b, slices.Equal)` |
| Invert | `out[v] = append(out[v], k)` |
| Spread two maps | `c := maps.Clone(a); maps.Copy(c, b)` |

| Error | Cause |
|---|---|
| `cannot use maps.Keys(m) (value of func type iter.Seq[string]) as []string value` | treating the iterator as a slice |
| `invalid argument: maps.Keys(m) … for built-in len` | `len` on an iterator |
| `V does not satisfy comparable` | `maps.Equal` on a map of slices; use `EqualFunc` |

## Common Mistakes

**Assigning `maps.Keys(m)` to a `[]K`.** It is an `iter.Seq[K]`; wrap it in
`slices.Collect` or `slices.Sorted`.

**Treating `maps.Clone` as a deep copy.** Nested maps, slices and pointers stay
shared; clone each level you care about.

**Calling `maps.Copy` with a nil destination.** It writes to `dst` and will panic
with `assignment to entry in nil map`.

**Using `maps.Equal` on `map[K][]V`.** `[]V` is not comparable, so it does not
compile; use `maps.EqualFunc(a, b, slices.Equal)`.

**Inverting a map with `inv[v] = k` when values repeat.** Entries are lost
non-deterministically; invert to `map[V][]K`.

## Exercises

1. Write `func deepClone[K comparable, V any](m map[K]map[K]V) map[K]map[K]V` and a
   test proving the result shares nothing with the input.
2. Implement `func diff[K, V comparable](a, b map[K]V) (added, removed, changed map[K]V)`
   using only `maps` and `slices`, with deterministic output.
3. Write `func mergeN[K comparable, V any](ms ...map[K]V) map[K]V` with last-wins
   semantics, then a variant that reports conflicting keys.
4. Build `func groupBy[T any, K comparable](xs []T, f func(T) K) map[K][]T` and use
   `slices.Sorted(maps.Keys(...))` to print the groups reproducibly across 10 runs.
5. Write `func topKeys[K comparable](m map[K]int, n int) []K` returning the n keys
   with the largest values, breaking ties deterministically, using
   `slices.SortedFunc`.

---

[← 7.8 The slices Package Reference](08-slices-package.md) · [Course Index](../README.md) · [7.10 sort: Before and Alongside Generics →](10-sort.md)
