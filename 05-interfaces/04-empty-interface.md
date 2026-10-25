[← 5.3 Type Assertions and Type Switches](03-assertions-type-switches.md) · [Course Index](../README.md) · [5.5 Interface Composition and Small Interfaces →](05-composition.md)

# 5.4 — The Empty Interface and any

## What You'll Learn

- `any` is a type *alias* for `interface{}` — not a new type, fully interchangeable, idiomatic since 1.18
- You can do nothing with an `any` but store, print, and compare it; everything else needs an assertion
- `[]string` is **not** assignable to `[]any`, and the fix is an explicit loop
- JSON into `map[string]any` makes every number a `float64`
- Generics replaced `any` for containers; `any` still owns `fmt`, encoding, and heterogeneous payloads

## Coming From Another Language

| Other | Go |
|---|---|
| TS `any` — disables checking, allows any operation | `any` allows almost no operation until you assert |
| TS `unknown` | the accurate analogue |
| Python `object` / untyped | similar reach, but every use needs an explicit unbox |
| Rust `Box<dyn Any>` + `downcast_ref` | `any` + `i.(T)`, but built in and far more common |
| Java `Object` with autoboxing | the same boxing cost, same erasure of static type |

If you come from TypeScript, invert your instinct: Go's `any` behaves like TS's `unknown`, not
like TS's `any`. It is a *lock*, not a bypass.

## any and interface{} are the same type

```go
package main

import "fmt"

type MyAny = interface{}

func takesEmpty(v interface{}) string { return fmt.Sprintf("%T", v) }
func takesAny(v any) string           { return fmt.Sprintf("%T", v) }

func main() {
  var a any = 1
  var b interface{} = 1
  var c MyAny = 1
  fmt.Println(a == b, b == c)
  fmt.Println(takesEmpty(a), takesAny(b))

  f := takesEmpty
  var g func(any) string = f
  fmt.Println(g(3.5))

  var xs []any = []any{1, "two", 3.0, nil, true, []int{1}}
  for _, x := range xs {
    fmt.Printf("%T ", x)
  }
  fmt.Println()
}
```

Output:

```
true true
int int
float64
int string float64 <nil> bool []int 
```

Note the third-to-last line: `func(interface{}) string` is assignable to `func(any) string`
with no conversion. An alias is not a distinct type — `any` is literally spelled differently.

Idiomatic: write `any`. `gofmt -r` and `go fix` will not rewrite old code for you, but new code
uses `any` universally.

An interface with zero methods is satisfied by every type, which is why anything goes in. The
`nil` element above is a nil interface *inside* a `[]any` — perfectly legal.

## What you cannot do

Almost everything.

```go
package main

import "fmt"

func main() {
  var v any = 42
  fmt.Println(v + 1)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:15: invalid operation: v + 1 (mismatched types any and untyped int)
```

```go
package main

import "fmt"

func main() {
  var s any = "hello"
  fmt.Println(len(s))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:19: invalid argument: s (variable of interface type any) for built-in len
```

No arithmetic, no `len`, no indexing, no ranging, no field access, no method calls — the method
set is empty. What you *can* do:

```go
package main

import "fmt"

func main() {
  var v any = 42
  fmt.Println(v)
  fmt.Printf("%d %T %v\n", v, v, v)
  fmt.Println(v == 42, v == nil)

  n := v.(int)
  fmt.Println(n + 1)
}
```

Output:

```
42
42 int 42
true false
43
```

Store it, pass it, print it, compare it, assert it. That is the whole list. The assertion is
where the type comes back, and from there it is an ordinary `int`.

## fmt is the canonical any API

Every `fmt` function is variadic over `any`:

```go
func Println(a ...any) (n int, err error)
func Printf(format string, a ...any) (n int, err error)
func Sprintf(format string, a ...any) string
func Fprintf(w io.Writer, format string, a ...any) (n int, err error)
```

`fmt` then type-switches internally on every argument, probes for `Stringer`, `error`, and
`Formatter`, and falls back to reflection. This is `any` used correctly: the set of printable
types is open-ended and genuinely unknowable in advance.

`log/slog` works the same way — alternating key/value pairs of `any`:

```go
package main

import (
  "fmt"
  "log/slog"
  "os"
)

func main() {
  fmt.Println(fmt.Sprintf("%v %s %d", []any{1, "a", 2}...))

  h := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(_ []string, a slog.Attr) slog.Attr {
      if a.Key == "time" {
        return slog.Attr{}
      }
      return a
    },
  })
  l := slog.New(h)
  l.Info("done", "count", 3, "name", "ada", "ratio", 0.5)
}
```

Output:

```
1 a 2
level=INFO msg=done count=3 name=ada ratio=0.5
```

The `ReplaceAttr` hook drops the timestamp so this output is reproducible; without it the
`time=` field changes every run.

## []string is not []any

This is the single most common `any` confusion, and the error is worth memorising.

```go
package main

import "fmt"

func join(parts []any) int { return len(parts) }

func main() {
  names := []string{"a", "b", "c"}
  fmt.Println(join(names))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:20: cannot use names (variable of type []string) as []any value in argument to join
```

It bites hardest with `fmt`, because `...any` looks like it should accept anything:

```go
package main

import "fmt"

func main() {
  names := []string{"a", "b"}
  fmt.Println(names...)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:15: cannot use names (variable of type []string) as []any value in argument to fmt.Println
```

The reason is memory layout, not pedantry. A `[]string` is a contiguous run of 16-byte string
headers; a `[]any` is a run of 16-byte interface pairs. Converting means rewriting every
element, which is an O(n) allocation — so Go makes you ask for it:

```go
package main

import "fmt"

func main() {
  names := []string{"a", "b", "c"}

  var vs []any = make([]any, len(names))
  for i, n := range names {
    vs[i] = n
  }
  fmt.Println(vs, len(vs))

  m := map[string]string{"k": "v"}
  var mv map[string]any = make(map[string]any, len(m))
  for k, v := range m {
    mv[k] = v
  }
  fmt.Println(mv)

  fmt.Println(names)
  fmt.Println(vs...)
}
```

Output:

```
[a b c] 3
map[k:v]
[a b c]
a b c
```

The same rule covers maps, channels, and `[][]any`. Note the last two lines: passing the
`[]string` as a *single* argument works fine and prints `[a b c]`; only the `...` spread needs
the real `[]any`.

## JSON into map[string]any

The place you will meet `any` whether you want to or not.

```go
package main

import (
  "encoding/json"
  "fmt"
)

func main() {
  data := []byte(`{"id":7,"name":"ada","score":99.5,"active":true,"tags":["x","y"],"meta":{"n":1},"boss":null}`)

  var m map[string]any
  if err := json.Unmarshal(data, &m); err != nil {
    panic(err)
  }

  for _, k := range []string{"id", "name", "score", "active", "tags", "meta", "boss"} {
    fmt.Printf("%-7s %-20T %v\n", k, m[k], m[k])
  }

  id := m["id"].(float64)
  fmt.Println(int(id) + 1)

  tags := m["tags"].([]any)
  fmt.Println(len(tags), tags[0].(string))

  meta := m["meta"].(map[string]any)
  fmt.Println(meta["n"])
}
```

Output:

```
id      float64              7
name    string               ada
score   float64              99.5
active  bool                 true
tags    []interface {}       [x y]
meta    map[string]interface {} map[n:1]
boss    <nil>                <nil>
8
2 x
1
```

The mapping `encoding/json` uses when the target is `any`:

| JSON | Go type in an `any` |
|---|---|
| number | `float64` — **always**, even `7` |
| string | `string` |
| `true`/`false` | `bool` |
| array | `[]any` |
| object | `map[string]any` |
| `null` | `nil` (a nil interface) |

`id` is `float64` holding 7. Asserting it to `int` does not work:

```go
package main

import (
  "encoding/json"
  "fmt"
)

func main() {
  var m map[string]any
  json.Unmarshal([]byte(`{"id":7}`), &m)
  id := m["id"].(int)
  fmt.Println(id)
}
```

This program panics on purpose. Output:

```
panic: interface conversion: interface {} is float64, not int

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:11 +0x114
exit status 2
```

And `float64` silently loses precision past 2^53, which matters for the 64-bit IDs that real
APIs hand out. `json.Number` is the escape hatch:

```go
package main

import (
  "bytes"
  "encoding/json"
  "fmt"
)

func main() {
  data := []byte(`{"id":10000000000000001}`)

  var m map[string]any
  json.Unmarshal(data, &m)
  fmt.Printf("%T %v\n", m["id"], m["id"])

  dec := json.NewDecoder(bytes.NewReader(data))
  dec.UseNumber()
  var n map[string]any
  if err := dec.Decode(&n); err != nil {
    fmt.Println("decode error:", err)
  }
  fmt.Printf("%T %v\n", n["id"], n["id"])
  i, err := n["id"].(json.Number).Int64()
  fmt.Println(i, err)
}
```

Output:

```
float64 1e+16
json.Number 10000000000000001
10000000000000001 <nil>
```

The ID came back as `1e+16` — the trailing `1` is gone, with no error anywhere. `UseNumber`
keeps the digits as a string until you choose how to parse them.

Idiomatic: decode into a **struct**, not a `map[string]any`. Then `ID int64` just works, you
get field names checked, and you never see a `float64`. Reach for `map[string]any` only for
genuinely unknown shapes — passthrough proxies, schema-less config, JSON you are inspecting
rather than using. 13.6 covers the struct path.

## Boxing costs an allocation

Putting a value in an interface usually means copying it to the heap so the value word can
point at it.

```go
package main

import (
  "fmt"
  "runtime"
)

var sink any

func boxed(n int) {
  for i := 0; i < n; i++ {
    sink = i
  }
}

func small(n int) {
  for i := 0; i < n; i++ {
    sink = i % 256
  }
}

func allocs(f func(int), n int) uint64 {
  var a, b runtime.MemStats
  runtime.GC()
  runtime.ReadMemStats(&a)
  f(n)
  runtime.ReadMemStats(&b)
  return b.Mallocs - a.Mallocs
}

func main() {
  fmt.Println("sink = i        :", allocs(boxed, 100000), "allocations")
  fmt.Println("sink = i % 256  :", allocs(small, 100000), "allocations")
}
```

Output:

```
sink = i        : 99744 allocations
sink = i % 256  : 0 allocations
```

Roughly one allocation per boxed `int`. The second loop is free because the runtime keeps a
static table of boxed small integers (0-255) to point at. Pointers and strings also avoid the
copy in many cases. The count is slightly under 100000 because a few iterations were in flight
across the measurement boundary.

The lesson is not "avoid `any`" — it is "do not funnel a hot numeric loop through it".

## any as a type vs any as a constraint

Since 1.18, `any` appears in two very different positions.

```go
var v any              // a TYPE: one value, dynamic type erased
func f[T any](x T) T   // a CONSTRAINT: "T may be any type", T stays static
```

The difference is everything:

```go
package main

import "fmt"

type StackAny struct{ items []any }

func (s *StackAny) Push(v any) { s.items = append(s.items, v) }
func (s *StackAny) Pop() (any, bool) {
  if len(s.items) == 0 {
    return nil, false
  }
  v := s.items[len(s.items)-1]
  s.items = s.items[:len(s.items)-1]
  return v, true
}

type Stack[T any] struct{ items []T }

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }
func (s *Stack[T]) Pop() (T, bool) {
  var zero T
  if len(s.items) == 0 {
    return zero, false
  }
  v := s.items[len(s.items)-1]
  s.items = s.items[:len(s.items)-1]
  return v, true
}

func firstAny(xs []any) any { return xs[0] }
func first[T any](xs []T) T { return xs[0] }

func main() {
  var a StackAny
  a.Push(1)
  a.Push("oops")
  v, _ := a.Pop()
  fmt.Printf("%T %v\n", v, v)

  var s Stack[int]
  s.Push(1)
  s.Push(2)
  n, _ := s.Pop()
  fmt.Println(n+1, fmt.Sprintf("%T", n))

  fmt.Printf("%T %T\n", firstAny([]any{1}), first([]int{1}))
}
```

Output:

```
string oops
3 int
int int
```

`StackAny` happily accepted a `string` into what was meant to be a stack of ints, and `Pop`
returns something you must assert before using. `Stack[int]` rejects it at compile time, and
`Pop` returns a usable `int` — note `n+1` compiles, which is impossible with `any`.

```go
package main

type Stack[T any] struct{ items []T }

func (s *Stack[T]) Push(v T) { s.items = append(s.items, v) }

func main() {
  var s Stack[int]
  s.Push(1)
  s.Push("two")
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:10: cannot use "two" (untyped string constant) as int value in argument to s.Push
```

Module 09 covers type parameters properly.

## Comparing anys can panic

Carried over from 5.2, because it is `any`-shaped code that triggers it.

```go
package main

import "fmt"

func dedupe(xs []any) []any {
  var out []any
  for _, x := range xs {
    found := false
    for _, o := range out {
      if o == x {
        found = true
        break
      }
    }
    if !found {
      out = append(out, x)
    }
  }
  return out
}

func main() {
  fmt.Println(dedupe([]any{1, 2, 1, "a", "a"}))
  fmt.Println(dedupe([]any{[]int{1}, []int{1}}))
}
```

This program panics on purpose. Output:

```
[1 2 a]
panic: runtime error: comparing uncomparable type []int

goroutine 1 [running]:
main.dedupe(...)
	/tmp/scratch/main.go:10
main.main()
	/tmp/scratch/main.go:24 +0x2e8
exit status 2
```

The function is correct for every comparable input and crashes on a slice. `func dedupe[T
comparable](xs []T) []T` makes that impossible to call wrongly — the constraint is checked at
the call site.

## When any is the right answer

| Use `any` | Use a type parameter | Use a concrete type or interface |
|---|---|---|
| `fmt`-style formatting of arbitrary values | containers, `Map`/`Filter`, `Min`/`Max` | anything with a known shape |
| encoding/decoding unknown shapes | anything where the caller knows `T` | your own domain types |
| a passthrough cache or registry keyed by type | type-safe wrappers over one `T` | the 95% case |
| `context.WithValue` payloads | | |
| heterogeneous log/metric attributes | | |

Before generics, `any` was the only way to write a reusable container, so older code is full of
`[]any`. That use is obsolete. What remains is genuine type erasure: places where the set of
types is open and the code treats values opaquely or reflectively.

Rule of thumb: if you find yourself asserting an `any` back to one specific type at every call
site, you wanted a type parameter. If you find yourself type-switching over an open set, `any`
is right.

## Quick Reference

| Form | Meaning |
|---|---|
| `any` | alias for `interface{}`; use this spelling |
| `var v any = x` | box `x`; type word set, usually one allocation |
| `v.(T)`, `v, ok := v.(T)` | the only way to use the value |
| `[]any`, `map[string]any` | heterogeneous containers; element conversion is explicit |
| `...any` | the `fmt`/`slog` variadic signature |
| `[T any]` | type *parameter* list — a constraint, not a type |
| `dec.UseNumber()` | decode JSON numbers as `json.Number`, not `float64` |

| Operation on an `any` | Allowed? |
|---|---|
| assign, pass, return, store in a slice/map | yes |
| `fmt.Print*`, `%v`, `%T` | yes |
| `==`, `!=` | yes, but panics on uncomparable dynamic types |
| type assertion, type switch | yes |
| arithmetic, `len`, index, range, field, method | **no** |

## Common Mistakes

**Passing `[]string` where `[]any` is wanted.** Build a `[]any` with an explicit loop; the
layouts differ, so no implicit conversion exists.

**Asserting a JSON-decoded number to `int`.** It is a `float64`; assert to `float64` and
convert, or use `dec.UseNumber()`.

**Decoding into `map[string]any` out of habit.** Decode into a struct unless the shape is
genuinely unknown.

**Writing `[]any`-based containers in new code.** Use a type parameter; you get compile-time
checking and no boxing.

**Comparing two `any` values with `==` on untrusted data.** It panics on slices, maps, and
funcs; constrain with `comparable` or compare after asserting.

## Exercises

1. Write `func flatten(v any) []any` that walks a value decoded from JSON — nested
   `map[string]any` and `[]any` — and returns every scalar leaf. Test it on a three-level
   document.
2. Write `toAny[T any](xs []T) []any` and use it to make `fmt.Println(toAny(names)...)` work
   for a `[]string`, a `[]int`, and a `[]struct{}`.
3. Decode `{"id": 9007199254740993}` into a `map[string]any` and print the value; then decode
   the same bytes into a struct with `ID int64` and with `UseNumber`. Explain each result.
4. Implement a cache `type Cache struct { m map[string]any }` with `Get(key) (any, bool)`, then
   rewrite it as `Cache[T any]`. Compare the call sites and count the assertions you deleted.
5. Measure the boxing cost yourself: benchmark a loop summing `[]int` against one summing
   `[]any` with `go test -bench . -benchmem`, then a generic `[T Number]` version. Report
   ns/op and allocs/op for all three.

---

[← 5.3 Type Assertions and Type Switches](03-assertions-type-switches.md) · [Course Index](../README.md) · [5.5 Interface Composition and Small Interfaces →](05-composition.md)
