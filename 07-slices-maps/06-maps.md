[← 7.5 copy, Three-Index Slicing, and Deletion](05-copy-three-index.md) · [Course Index](../README.md) · [7.7 Map Keys, Nested Maps, and Sets →](07-map-patterns.md)

# 7.6 — Maps: Every Operation

## What You'll Learn

- Reads from a nil map work; writes panic with `assignment to entry in nil map`
- The comma-ok idiom, and why `m[k] == 0` cannot tell you whether `k` exists
- Iteration order is deliberately randomized — two runs of the same program differ
- Map elements are not addressable: `&m[k]` and `m[k].field = v` are compile errors

## Coming From Another Language

| Other | Go |
|---|---|
| JS `obj[k]` returns `undefined` | `m[k]` returns the **zero value**; use `v, ok := m[k]` |
| JS `k in obj` / `obj.hasOwnProperty(k)` | `_, ok := m[k]` |
| Python `d[k]` raises `KeyError` | `m[k]` never raises; it returns the zero value |
| Python `d.get(k, 0)` | `m[k]` already does this |
| Python `del d[k]` raises on missing | `delete(m, k)` is a silent no-op |
| Python `dict` preserves insertion order | Go randomizes iteration order on purpose |
| Rust `HashMap::get` returns `Option<&V>` | `v, ok := m[k]`, but `v` is a **copy**, not a reference |

## Construction

```go
package main

import "fmt"

func main() {
  a := make(map[string]int)
  b := make(map[string]int, 100)
  c := map[string]int{"x": 1, "y": 2}
  d := map[string]map[string]int{
    "outer": {"inner": 1},
  }
  var e map[string]int

  fmt.Println(a, len(a), a == nil)
  fmt.Println(b, len(b))
  fmt.Println(c, len(c))
  fmt.Println(d, len(d), d["outer"]["inner"])
  fmt.Println(e, len(e), e == nil)
}
```

Output:

```
map[] 0 false
map[] 0
map[x:1 y:2] 2
map[outer:map[inner:1]] 1 1
map[] 0 true
```

The second argument to `make` is a size hint, not a cap — it preallocates buckets
and is purely a performance knob. In a nested literal the inner type can be
omitted: `{"inner": 1}` means `map[string]int{"inner": 1}`. `fmt` prints maps with
keys sorted, which is a courtesy of `fmt` and not a property of the map.

## The nil map: reads fine, writes panic

```go
package main

import "fmt"

func main() {
  var m map[string]int
  fmt.Println(m["missing"], len(m), m == nil)
  v, ok := m["missing"]
  fmt.Println(v, ok)
  for k := range m {
    fmt.Println("never", k)
  }
  delete(m, "x")
  fmt.Println("delete on nil map is fine")
  m["x"] = 1
}
```

This program panics on purpose. Output:

```
0 0 true
0 false
delete on nil map is fine
panic: assignment to entry in nil map

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:15 +0x224
exit status 2
```

Read, `len`, `range`, and `delete` all treat a nil map as empty. Only assignment
panics. This asymmetry with slices — where `append` to nil just works — is the most
common map bug in Go. It surfaces when a struct has a map field that nobody
initialised:

```go
type cache struct{ entries map[string]int }
var c cache
c.entries["k"] = 1
```

A `[]T` field needs no initialisation. A `map[K]V` field does. Idiomatic: give the
struct a constructor, or lazily initialise with
`if m == nil { m = make(map[K]V) }`.

## The comma-ok idiom

`m[k]` returns the zero value for a missing key, so a `0` tells you nothing by
itself.

```go
package main

import "fmt"

func main() {
  counts := map[string]int{"zero": 0, "one": 1}
  v1, ok1 := counts["zero"]
  v2, ok2 := counts["absent"]
  fmt.Println(v1, ok1)
  fmt.Println(v2, ok2)
  fmt.Println(counts["zero"] == counts["absent"])

  flags := map[string]bool{"off": false}
  fmt.Println(flags["off"], flags["unset"])
  _, present := flags["off"]
  fmt.Println(present)

  if _, ok := counts["one"]; ok {
    fmt.Println("has one")
  }
}
```

Output:

```
0 true
0 false
true
false false
true
has one
```

`counts["zero"] == counts["absent"]` is `true` and utterly uninformative. The same
trap is sharper with `map[string]bool`: a stored `false` and an absent key are
indistinguishable by value. Idiomatic: `if _, ok := m[k]; ok` for existence, and
only drop the `ok` when the zero value is a legitimate answer.

## Update, delete, len, clear

```go
package main

import "fmt"

func main() {
  m := map[string]int{"a": 1, "b": 2, "c": 3}
  fmt.Println(len(m))

  delete(m, "b")
  delete(m, "nosuch")
  fmt.Println(len(m), m["b"])

  m["a"] = 10
  m["new"] = 1
  fmt.Println(len(m), m["a"])

  clear(m)
  fmt.Println(len(m), m == nil)

  m["x"] = 1
  m = nil
  fmt.Println(len(m), m == nil)
}
```

Output:

```
3
2 0
3 10
0 false
0 true
```

`delete` is a statement, not an expression — it returns nothing, and deleting a key
that is not there (or from a nil map) is a no-op. `clear(m)` (Go 1.21+) removes
every entry, keeping the same map object so other holders of the same map see it
emptied. `m = nil` only rebinds your variable; other holders keep the full map.

## Increment a missing key

Because `m[k]` yields the zero value, `m[k]++` and `m[k] += x` work on keys that do
not exist yet.

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  counts := map[string]int{}
  for _, w := range strings.Fields("a b a c b a") {
    counts[w]++
  }
  fmt.Println(counts["a"], counts["b"], counts["c"], counts["z"])

  totals := map[string]float64{}
  totals["sum"] += 1.5
  totals["sum"] += 2.5
  fmt.Println(totals)

  ops := map[string]func(int, int) int{
    "add": func(a, b int) int { return a + b },
    "mul": func(a, b int) int { return a * b },
  }
  fmt.Println(ops["add"](2, 3), ops["mul"](2, 3))
  if f, ok := ops["div"]; ok {
    fmt.Println(f(6, 3))
  } else {
    fmt.Println("no div")
  }

  anyMap := map[string]any{"n": 1, "s": "x", "sl": []int{1}}
  fmt.Println(anyMap["n"], anyMap["s"], anyMap["sl"])

  nested := map[string]map[string]int{"a": {"x": 1}}
  fmt.Println(nested["a"]["x"], nested["missing"]["x"], len(nested["missing"]))
}
```

Output:

```
3 2 1 0
map[sum:4]
5 6
no div
1 x [1]
1 0 0
```

No "if key exists, else initialise" dance — the zero value does it. Note the last
line: reading through a missing nested map is safe, because `nested["missing"]`
yields a nil map and reading a nil map is fine. *Writing* `nested["missing"]["x"] = 1`
would panic; that is [7.7](07-map-patterns.md).

A map value can be any type, including a func (the dispatch-table pattern) or a
slice. The comma-ok form is how you avoid calling a nil func.

## Iteration order is randomized

The runtime picks a random starting bucket for every `range`. This is deliberate:
it stops code from accidentally depending on an order that is not guaranteed.

```go
package main

import "fmt"

func main() {
  m := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4, "e": 5}
  for k := range m {
    fmt.Print(k, " ")
  }
  fmt.Println()
}
```

Three separate runs of that same binary:

```
a b c d e 
```

```
e a b c d 
```

```
a b c d e 
```

The order varies run to run and even between two `range` loops in one program. Do
not write tests that assert it, do not build output from it. To get a stable order,
sort the keys — `slices.Sorted(maps.Keys(m))`, covered in [7.9](09-maps-package.md).

## Mutating during iteration

Deleting during `range` is defined: an entry removed before it is reached will not
be produced. Adding is *undefined* — the new entry may or may not be visited.

```go
package main

import "fmt"

func main() {
  m := map[int]int{1: 1, 2: 2, 3: 3, 4: 4}
  visited := 0
  for k := range m {
    visited++
    for d := range m {
      if d != k {
        delete(m, d)
      }
    }
  }
  fmt.Println(visited, len(m))

  n := map[int]int{1: 1}
  seen := 0
  for k := range n {
    seen++
    if seen > 20 {
      fmt.Println("runaway")
      break
    }
    if k < 5 {
      n[k+100] = 0
    }
  }
  fmt.Println(seen, len(n))
}
```

Output:

```
1 1
2 2
```

The first loop deletes the other three keys on its first iteration and then stops —
one visit, one entry left, deterministic regardless of which key came first. The
second loop added key 101 and happened to visit it, so `seen` is 2; on another
implementation or map size it might be 1. That value is not something to rely on.
Idiomatic: collect keys first, then mutate.

## Maps are reference-like

A map value is a pointer to a runtime hash table. Passing it to a function shares
the table — no `*map[K]V` needed.

```go
package main

import "fmt"

func addKey(m map[string]int) {
  m["added"] = 1
}

func reassign(m map[string]int) {
  m = map[string]int{"replaced": 1}
  _ = m
}

func addElem(s []int) {
  s = append(s, 1)
  _ = s
}

func main() {
  m := map[string]int{"orig": 0}
  addKey(m)
  fmt.Println(m, len(m))
  reassign(m)
  fmt.Println(m, len(m))

  a := m
  a["via-alias"] = 2
  fmt.Println(m, len(m))

  s := []int{1}
  addElem(s)
  fmt.Println(s, len(s))
}
```

Output:

```
map[added:1 orig:0] 2
map[added:1 orig:0] 2
map[added:1 orig:0 via-alias:2] 3
[1] 1
```

`addKey` worked and `addElem` did not — that is the whole difference. A map's
identity lives behind the pointer, so adding a key is visible everywhere. A slice's
length lives *in* the copied header, so growing it is not. `reassign` only rebinds
the local parameter, as always.

You need `*map[K]V` only to replace the whole map in the caller, which is rare.

## Maps are not comparable

```go
package main

import "fmt"

func main() {
  a := map[string]int{"x": 1}
  b := map[string]int{"x": 1}
  fmt.Println(a == b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: a == b (map can only be compared to nil)
```

Use `maps.Equal` ([7.9](09-maps-package.md)). As with slices, a struct containing a
map is not comparable either, and a map cannot be a map key.

## Map elements are not addressable

The runtime may move entries when the table grows, so there is no stable address to
hand out.

```go
package main

import "fmt"

func main() {
  m := map[string]int{"x": 1}
  p := &m["x"]
  fmt.Println(p)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:9: invalid operation: cannot take address of m["x"] (map index expression of type int)
```

The consequence that bites daily: you cannot assign to a field of a struct *value*
stored in a map.

```go
package main

import "fmt"

type pt struct{ x, y int }

func main() {
  m := map[string]pt{"a": {1, 2}}
  m["a"].x = 9
  fmt.Println(m)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:3: cannot assign to struct field m["a"].x in map
```

Three fixes:

```go
package main

import "fmt"

type pt struct{ x, y int }

func main() {
  m := map[string]pt{"a": {1, 2}}
  v := m["a"]
  v.x = 9
  m["a"] = v
  fmt.Println(m)

  p := map[string]*pt{"a": {1, 2}}
  p["a"].x = 9
  fmt.Println(*p["a"])

  s := map[string][]int{"a": {1, 2}}
  s["a"][0] = 9
  s["a"] = append(s["a"], 3)
  fmt.Println(s)
}
```

Output:

```
map[a:{9 2}]
{9 2}
map[a:[9 2 3]]
```

- **Read-modify-write** (`v := m[k]; v.x = 9; m[k] = v`): keeps value semantics,
  costs a copy. Idiomatic for small structs.
- **Map of pointers** (`map[K]*V`): mutate in place, but now nil is possible and
  the values are individually heap-allocated.
- **Map of slices** (`map[K][]T`): `s[k][0] = 9` works because indexing a *slice*
  is addressable — the slice header is copied, the elements are not. But
  `append` still needs the write-back.

## Concurrent access is a fatal error

Concurrent writes (or a write racing a read) are detected by the runtime and abort
the process. This is a `fatal error`, not a `panic`: `recover` cannot catch it.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  m := map[int]int{}
  var wg sync.WaitGroup
  for g := 0; g < 8; g++ {
    wg.Add(1)
    go func(g int) {
      defer wg.Done()
      for i := 0; i < 10000; i++ {
        m[g*10000+i] = i
      }
    }(g)
  }
  wg.Wait()
  fmt.Println(len(m))
}
```

This program crashes on purpose. Output (goroutine dumps truncated; the goroutine
numbers and addresses vary per run):

```
fatal error: concurrent map writes

goroutine 35 [running]:
internal/runtime/maps.fatal({0x1025e30f7?, 0x0?})
	/usr/local/go/src/runtime/panic.go:1195 +0x20
main.main.func1(0x1)
	/tmp/scratch/main.go:16 +0x68
created by main.main in goroutine 1
	/tmp/scratch/main.go:13 +0x50
exit status 2
```

The detection is best-effort — a race can also silently corrupt the table or
produce `concurrent map read and map write`. Run with `-race` to catch it reliably.
Fix with a `sync.RWMutex` around the map, or `sync.Map` for append-mostly caches;
see 11.3.

## Quick Reference

| Operation | Syntax | On missing key | On nil map |
|---|---|---|---|
| Create | `make(map[K]V)` / `make(map[K]V, n)` / `map[K]V{…}` | — | — |
| Read | `v := m[k]` | zero value | zero value |
| Read + test | `v, ok := m[k]` | `zero, false` | `zero, false` |
| Write | `m[k] = v` | inserts | **panics** |
| Compound | `m[k]++`, `m[k] += x` | starts from zero | **panics** |
| Delete | `delete(m, k)` | no-op | no-op |
| Count | `len(m)` | — | 0 |
| Empty | `clear(m)` (1.21+) | — | no-op |
| Iterate | `for k, v := range m` | — | zero iterations |
| Compare | `m == nil` only | — | — |

| Error / fatal | Cause |
|---|---|
| `assignment to entry in nil map` | writing to a `var m map[K]V` |
| `invalid operation: a == b (map can only be compared to nil)` | `m1 == m2` |
| `cannot take address of m["x"]` | `&m[k]` |
| `cannot assign to struct field m["a"].x in map` | `m[k].field = v` on a value type |
| `fatal error: concurrent map writes` | unsynchronised concurrent write |
| `fatal error: concurrent map read and map write` | a read racing a write |

| Map vs slice | Map | Slice |
|---|---|---|
| Zero value usable for writing | no — panics | yes — `append` works |
| Passing to a function | mutations fully shared | elements shared, length not |
| `clear` | removes all entries | zeroes elements, keeps `len` |
| Order | randomized | index order |
| Comparable | no (nil only) | no (nil only) |

## Common Mistakes

**Writing to a map field of a struct you never initialised.** `var m map[K]V` is
nil and assignment panics; build it with `make` in a constructor.

**Using `if m[k] != 0` to test existence.** A stored zero is indistinguishable from
a missing key; use `if _, ok := m[k]; ok`.

**Writing `m[k].field = v` for a struct value.** Map elements are not addressable;
read-modify-write, or store `*V`.

**Relying on iteration order, including in tests.** It is randomized per run; sort
`maps.Keys(m)` when you need determinism.

**Sharing a map across goroutines without a lock.** It is a fatal error that
`recover` cannot catch; guard with `sync.RWMutex` or use `sync.Map`.

## Exercises

1. Write `func mustGet[K comparable, V any](m map[K]V, k K) V` that panics with a
   useful message on a missing key, and a `getOr(m, k, def)` that does not.
2. Implement a `Counter` struct with a `map[string]int` field and a lazily
   initialising `Inc(string)` method. Write a test that calls `Inc` on the zero
   value of `Counter` and passes.
3. Build a `map[string]struct{ hits int }` and a `map[string]*struct{ hits int }`,
   write `Inc` for both, and benchmark with `-benchmem`. Explain the allocation
   difference.
4. Write a program that prints a map's keys in a stable order without using the
   `maps` package, then show it producing the same output across 10 runs.
5. Demonstrate `concurrent map read and map write` (not the write/write variant),
   then fix the same program with `sync.RWMutex` and confirm with `go run -race`.

---

[← 7.5 copy, Three-Index Slicing, and Deletion](05-copy-three-index.md) · [Course Index](../README.md) · [7.7 Map Keys, Nested Maps, and Sets →](07-map-patterns.md)
