[← 7.6 Maps: Every Operation](06-maps.md) · [Course Index](../README.md) · [7.8 The slices Package Reference →](08-slices-package.md)

# 7.7 — Map Keys, Nested Maps, and Sets

## What You'll Learn

- Key types must be comparable: structs, arrays, pointers, channels and interfaces qualify; slices, maps and funcs do not
- An `any` key accepts a non-comparable dynamic type at compile time and panics at runtime
- `NaN` as a key can be inserted twice and never retrieved
- `map[T]struct{}` is the idiomatic set; maps never shrink, so `delete` does not free buckets

## Coming From Another Language

| Other | Go |
|---|---|
| JS `Set` | `map[T]struct{}` — no built-in set type |
| Python `set` | `map[T]struct{}`; `frozenset` has no equivalent |
| JS object keys are strings | Go keys keep their type; `m[1]` and `m["1"]` are different maps |
| Python tuple keys `d[(a, b)]` | a struct or array key: `m[key{a, b}]` |
| Python `collections.defaultdict(list)` | `m[k] = append(m[k], v)` — the nil slice makes it work |
| Rust `HashMap<K, V>` needs `Hash + Eq` | Go needs `comparable` |

## Valid key types

Anything comparable with `==`. Slices, maps and funcs are not.

```go
package main

import "fmt"

func main() {
  m := map[[]int]string{}
  fmt.Println(m)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:12: invalid map key type []int
```

The same error for `map[map[string]int]string` (`invalid map key type map[string]int`)
and `map[func()]string` (`invalid map key type func()`). Everything else works:

```go
package main

import "fmt"

type key struct {
  user string
  day  int
}

func main() {
  a := map[string]int{"s": 1}
  b := map[[2]int]int{{1, 2}: 3}
  c := map[key]int{{"u", 1}: 5}
  d := map[bool]int{true: 1}
  e := map[float64]int{1.5: 1}
  f := map[any]int{1: 1, "s": 2, [2]int{1, 2}: 3}
  g := map[*int]int{}
  n := 5
  g[&n] = 1
  h := map[chan int]int{}
  h[make(chan int)] = 1
  type iface interface{ M() }
  i := map[iface]int{}

  fmt.Println(a, b, c, d, e)
  fmt.Println(len(f), len(g), len(h), len(i))
  fmt.Printf("%T\n", f)
}
```

Output:

```
map[s:1] map[[1 2]:3] map[{u 1}:5] map[true:1] map[1.5:1]
3 1 1 0
map[interface {}]int
```

A struct key is comparable only if all its fields are, so a struct with a slice
field is rejected the same way `[]int` is.

## Interface keys panic at runtime

`map[any]V` compiles with any key expression. The comparability check moves to
runtime.

```go
package main

import "fmt"

func main() {
  m := map[any]string{}
  m[1] = "int"
  m["s"] = "string"
  m[[2]int{1, 2}] = "array"
  fmt.Println(len(m), m[1], m["s"], m[[2]int{1, 2}])

  fmt.Println(m[int64(1)], m[1] == "int")

  m[[]int{1}] = "slice"
  fmt.Println("unreachable")
}
```

This program panics on purpose. Output:

```
3 int string array
 true
panic: runtime error: hash of unhashable type []int

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:14 +0x428
exit status 2
```

Two things worth noting. `m[int64(1)]` found nothing even though the numeric value
is 1 — interface equality compares the dynamic *type* as well, so `int(1)` and
`int64(1)` are different keys. And a `[]int` key is a `hash of unhashable type`
panic, not a compile error. Avoid `map[any]V` in favour of a concrete key type
whenever you can.

## Struct keys vs nested maps

The two ways to model a composite key. Both work; they differ in what is easy.

```go
package main

import "fmt"

func set(m map[string]map[string]int, outer, inner string, v int) {
  if m[outer] == nil {
    m[outer] = make(map[string]int)
  }
  m[outer][inner] = v
}

type ckey struct {
  outer, inner string
}

func main() {
  nested := map[string]map[string]int{}
  set(nested, "a", "x", 1)
  set(nested, "a", "y", 2)
  set(nested, "b", "x", 3)
  fmt.Println(nested)
  fmt.Println(len(nested), len(nested["a"]), nested["zz"]["x"])

  flat := map[ckey]int{}
  flat[ckey{"a", "x"}] = 1
  flat[ckey{"a", "y"}] = 2
  flat[ckey{"b", "x"}] = 3
  fmt.Println(flat)
  fmt.Println(len(flat), flat[ckey{"zz", "x"}])

  for k := range nested["a"] {
    _ = k
  }
  fmt.Println(len(nested["a"]))
}
```

Output:

```
map[a:map[x:1 y:2] b:map[x:3]]
2 2 0
map[{a x}:1 {a y}:2 {b x}:3]
3 0
2
```

| | Nested `map[A]map[B]V` | Struct key `map[key]V` |
|---|---|---|
| Write | needs a nil check / get-or-create | plain `m[k] = v` |
| Read a missing path | safe, returns zero | safe, returns zero |
| "All B for a given A" | `nested[a]` directly | must scan every key |
| `len` | outer count only | total entries |
| Delete all of one A | `delete(nested, a)` | scan and delete |
| Allocations | one map per outer key | one map |

Idiomatic: struct key, unless you genuinely need to enumerate or drop a whole
group. A struct key is one hash lookup, one allocation, and no nil-map trap.

## The nil inner map

The nested form's hazard: reading through a missing outer key is safe, writing is
not.

```go
package main

import "fmt"

func main() {
  m := map[string]map[string]int{}
  fmt.Println(m["a"]["b"], len(m["a"]), m["a"] == nil)
  m["a"]["b"] = 1
  fmt.Println("unreachable")
}
```

This program panics on purpose. Output:

```
0 0 true
panic: assignment to entry in nil map

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:8 +0x1b8
exit status 2
```

`m["a"]` returned the zero value of `map[string]int`, which is nil. The fix is the
`set` helper above, or a get-or-create:

```go
func inner(m map[string]map[string]int, k string) map[string]int {
  if v, ok := m[k]; ok {
    return v
  }
  v := make(map[string]int)
  m[k] = v
  return v
}
```

## Array keys vs slice keys

`[2]int` is comparable, `[]int` is not. When your data arrives as a slice, convert
to a fixed-size array (Go 1.20+) or to a string.

```go
package main

import (
  "fmt"
  "strings"
)

func keyOf(path []string) string { return strings.Join(path, "\x00") }

func main() {
  counts := map[[2]int]int{}
  pairs := [][]int{{1, 2}, {3, 4}, {1, 2}}
  for _, p := range pairs {
    counts[[2]int(p)]++
  }
  fmt.Println(counts)

  paths := map[string]int{}
  paths[keyOf([]string{"a", "b"})] = 1
  fmt.Println(paths[keyOf([]string{"a", "b"})], len(paths))
}
```

Output:

```
map[[1 2]:2 [3 4]:1]
1 1
```

The `\x00` separator matters: joining with `"/"` would make `["a/b"]` and
`["a", "b"]` collide. Idiomatic: use an array key when the arity is fixed, and a
joined string with a byte that cannot appear in the parts when it is not.

## Float keys and NaN

`NaN != NaN`, and a map key is looked up by `==`. The consequences are exactly as
bad as they sound.

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  nan := math.NaN()
  fmt.Println(nan == nan)

  m := map[float64]string{}
  m[nan] = "first"
  m[nan] = "second"
  m[math.NaN()] = "third"
  fmt.Println(len(m))
  v, ok := m[nan]
  fmt.Printf("%q %v\n", v, ok)
  for k, v := range m {
    fmt.Println(math.IsNaN(k), v)
  }
  delete(m, nan)
  fmt.Println(len(m))

  m[0.0] = "zero"
  m[-0.0] = "negzero"
  fmt.Println(m[0.0], len(m))
}
```

Output (the three NaN entries come out in a random order):

```
false
3
"" false
true first
true second
true third
3
negzero 4
```

Three insertions of "the same" NaN produced three distinct entries. None can be
read back, none can be deleted, and `clear` or a fresh map is the only way to get
rid of them. The second gotcha on the same line: `+0.0 == -0.0` is true, so
`m[-0.0] = "negzero"` *overwrote* the zero entry rather than adding one.

Idiomatic: never use a float as a map key unless you have already excluded NaN. If
you must, reject with `math.IsNaN` at the boundary.

## Pointer keys compare by address

```go
package main

import "fmt"

type pt struct{ x int }

func main() {
  a := &pt{1}
  b := &pt{1}
  m := map[*pt]string{a: "a", b: "b"}
  fmt.Println(len(m), m[a], m[b], a == b, *a == *b)

  c := a
  fmt.Println(m[c])

  vm := map[pt]string{{1}: "v"}
  fmt.Println(vm[pt{1}])
}
```

Output:

```
2 a b false true
a
v
```

Two pointers to equal values are different keys. That is sometimes exactly what you
want (identity maps, visited-sets during graph traversal) and sometimes a silent
bug. `map[pt]string` with the value type keys by contents instead.

## Sets

Go has no set type. `map[T]struct{}` is the idiomatic stand-in: `struct{}` occupies
zero bytes, so the map stores only keys.

```go
package main

import (
  "cmp"
  "fmt"
  "maps"
  "slices"
  "unsafe"
)

type set[T comparable] map[T]struct{}

func (s set[T]) add(v T)      { s[v] = struct{}{} }
func (s set[T]) has(v T) bool { _, ok := s[v]; return ok }
func (s set[T]) del(v T)      { delete(s, v) }

func sorted[T cmp.Ordered](s set[T]) []T { return slices.Sorted(maps.Keys(s)) }

func union[T comparable](a, b set[T]) set[T] {
  out := set[T]{}
  for k := range a {
    out.add(k)
  }
  for k := range b {
    out.add(k)
  }
  return out
}

func intersect[T comparable](a, b set[T]) set[T] {
  out := set[T]{}
  small, big := a, b
  if len(b) < len(a) {
    small, big = b, a
  }
  for k := range small {
    if big.has(k) {
      out.add(k)
    }
  }
  return out
}

func main() {
  a := set[string]{}
  a.add("x")
  a.add("y")
  a.add("x")
  fmt.Println(len(a), a.has("x"), a.has("z"))
  a.del("x")
  fmt.Println(sorted(a))

  b := set[int]{1: {}, 2: {}, 3: {}}
  c := set[int]{3: {}, 4: {}}
  fmt.Println(sorted(union(b, c)))
  fmt.Println(sorted(intersect(b, c)))

  fmt.Println(unsafe.Sizeof(struct{}{}), unsafe.Sizeof(false))

  bm := map[string]bool{"present": true, "stored-false": false}
  fmt.Println(bm["present"], bm["stored-false"], bm["absent"])
  _, ok := bm["stored-false"]
  fmt.Println(ok)
}
```

Output:

```
2 true false
[y]
[1 2 3 4]
[3]
0 1
true false false
true
```

`struct{}` is 0 bytes against `bool`'s 1, and `set[int]{1: {}}` is the short literal
form. `intersect` iterates the smaller set, which is the difference between O(min)
and O(max).

`map[T]bool` is one character shorter to write and introduces the ambiguity you saw
in [7.6](06-maps.md): `bm["stored-false"]` and `bm["absent"]` both read `false`, so
membership must go through comma-ok anyway. Idiomatic: `map[T]struct{}` for a set,
`map[T]bool` only when you really have three states (true / false / absent).

## Multimaps, grouping and counting

Appending to a missing key works because the zero value of `[]T` is a usable nil
slice.

```go
package main

import (
  "fmt"
  "maps"
  "slices"
  "strings"
)

type person struct {
  name string
  dept string
  age  int
}

func main() {
  tags := map[string][]string{}
  tags["a"] = append(tags["a"], "x")
  tags["a"] = append(tags["a"], "y")
  tags["b"] = append(tags["b"], "z")
  fmt.Println(tags, len(tags["missing"]))

  people := []person{
    {"ann", "eng", 30},
    {"bob", "eng", 40},
    {"cat", "ops", 25},
  }

  byDept := map[string][]person{}
  for _, p := range people {
    byDept[p.dept] = append(byDept[p.dept], p)
  }
  for _, d := range slices.Sorted(maps.Keys(byDept)) {
    names := []string{}
    for _, p := range byDept[d] {
      names = append(names, p.name)
    }
    fmt.Println(d, strings.Join(names, ","))
  }

  counts := map[string]int{}
  for _, p := range people {
    counts[p.dept]++
  }
  fmt.Println(counts)

  index := map[string]person{}
  for _, p := range people {
    index[p.name] = p
  }
  fmt.Println(index["bob"].age)
}
```

Output:

```
map[a:[x y] b:[z]] 0
eng ann,bob
ops cat
map[eng:2 ops:1]
40
```

`m[k] = append(m[k], v)` is the whole `defaultdict(list)` pattern — but you must
write back, because the append returns a new header and map elements are not
addressable. `slices.Sorted(maps.Keys(m))` is the standard way to get a stable
iteration order; see [7.9](09-maps-package.md).

## Struct values vs pointers

The addressability fixes from [7.6](06-maps.md) become a design choice once the map
is long-lived.

| | `map[K]V` (value) | `map[K]*V` (pointer) |
|---|---|---|
| Mutate a field | read-modify-write: `v := m[k]; v.f++; m[k] = v` | `m[k].f++` directly |
| Missing key | usable zero value | `nil`; dereferencing panics |
| Memory | inline in the bucket | pointer + a separate allocation per value |
| Copying out | independent copy | shared, aliased |
| Concurrency | the map needs a lock | the map *and* each value need one |

Idiomatic: value map for small structs you mutate rarely; pointer map when the
struct is large, mutated often, or genuinely shared. The pointer map costs you a nil
check on every access and one allocation per entry.

## Maps never shrink

`delete` removes entries but does not return bucket memory. The only way to shrink
is to build a fresh map.

```go
package main

import (
  "fmt"
  "runtime"
)

func heapMB() float64 {
  runtime.GC()
  var ms runtime.MemStats
  runtime.ReadMemStats(&ms)
  return float64(ms.HeapAlloc) / (1 << 20)
}

func main() {
  base := heapMB()
  m := make(map[int][128]byte)
  for i := 0; i < 200_000; i++ {
    m[i] = [128]byte{}
  }
  full := heapMB()
  fmt.Printf("len=%d heap grew %.0f MB\n", len(m), full-base)

  for i := 0; i < 199_999; i++ {
    delete(m, i)
  }
  after := heapMB()
  fmt.Printf("len=%d heap now %.0f MB above base\n", len(m), after-base)

  fresh := make(map[int][128]byte, len(m))
  for k, v := range m {
    fresh[k] = v
  }
  m = fresh
  final := heapMB()
  fmt.Printf("len=%d heap now %.0f MB above base\n", len(m), final-base)
}
```

Output (heap figures are approximate and vary a little between runs):

```
len=200000 heap grew 36 MB
len=1 heap now 36 MB above base
len=1 heap now 0 MB above base
```

36 MB held for one entry. `clear(m)` has the same property — it empties the map
without releasing buckets, which is exactly what you want for reuse and exactly
wrong for a long-lived map that spiked once. For a cache that grows and shrinks,
periodically rebuild it, or shard it so whole shards can be dropped.

## Quick Reference

| Key type | Valid? | Note |
|---|---|---|
| `string`, numeric, `bool`, `rune` | yes | |
| `[N]T` | yes, if `T` comparable | `[2]int` works, `[2][]int` does not |
| `struct{…}` | yes, if all fields comparable | the composite-key idiom |
| `*T` | yes | compares by address, not contents |
| `chan T` | yes | compares by channel identity |
| `interface{…}` / `any` | compiles | panics at runtime on a non-comparable dynamic type |
| `[]T` | no | `invalid map key type []T` |
| `map[K]V` | no | `invalid map key type map[K]V` |
| `func(…)` | no | `invalid map key type func(…)` |
| `float64` | yes, but | `NaN` is unretrievable; `+0 == -0` |

| Pattern | Code |
|---|---|
| Set | `map[T]struct{}`; add `s[v] = struct{}{}`; has `_, ok := s[v]` |
| Multimap | `m[k] = append(m[k], v)` |
| Counter | `m[k]++` |
| Group by | `g[key(x)] = append(g[key(x)], x)` |
| Composite key | `type k struct{ a, b string }` |
| Nested get-or-create | `if m[k] == nil { m[k] = make(...) }` |
| Stable iteration | `for _, k := range slices.Sorted(maps.Keys(m))` |
| Shrink | rebuild into `make(map[K]V, len(m))` |

| Error / panic | Cause |
|---|---|
| `invalid map key type []int` | non-comparable key type, compile time |
| `hash of unhashable type []int` | non-comparable dynamic type in an `any` key |
| `assignment to entry in nil map` | writing to a missing inner map |

## Common Mistakes

**Writing `m[outer][inner] = v` on a nested map.** The inner map is nil until you
create it; use a get-or-create helper or a struct key.

**Using `map[string]bool` as a set.** A stored `false` is indistinguishable from an
absent key; use `map[T]struct{}`.

**Joining slice elements with `/` to build a key.** Separators that can appear in
the data cause collisions; use `"\x00"` or an array key.

**Using a `float64` key without excluding NaN.** NaN keys can be inserted
repeatedly and never read back or deleted.

**Expecting `delete` or `clear` to release memory.** Buckets are retained; rebuild
the map into a fresh one.

## Exercises

1. Write `func group[T any, K comparable](xs []T, key func(T) K) map[K][]T` and use
   it to group a `[]person` by department and by age decade.
2. Implement `Set[T]` with `Add`, `Has`, `Delete`, `Len`, `Union`, `Intersect`,
   `Difference`, and `All() iter.Seq[T]`. Benchmark `map[T]struct{}` against
   `map[T]bool` at 1,000,000 elements with `-benchmem`.
3. Build a two-level cache as `map[string]map[string][]byte` and then as
   `map[struct{ a, b string }][]byte`. Measure memory with `runtime.ReadMemStats`
   at 10,000 outer x 10 inner entries and explain the difference.
4. Write `func invert[K, V comparable](m map[K]V) map[V][]K` that survives duplicate
   values, and a variant returning `(map[V]K, error)` that reports a collision.
5. Demonstrate the NaN key problem, then write a `FloatMap` wrapper whose `Set`
   returns an error for NaN and whose `Get` works for every other float, `-0.0`
   included.

---

[← 7.6 Maps: Every Operation](06-maps.md) · [Course Index](../README.md) · [7.8 The slices Package Reference →](08-slices-package.md)
