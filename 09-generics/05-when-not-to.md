[← 9.4 Type Inference and Its Limits](04-inference.md) · [Course Index](../README.md) · [10.1 Goroutines and the Scheduler →](../10-concurrency-basics/01-goroutines.md)

# 9.5 — When Generics Are the Wrong Tool

## What You'll Learn

- The eight shapes where a type parameter adds noise and nothing else
- The six where it genuinely earns its place, with `any`+assertion as the strongest case
- Real benchmark numbers: generic matches concrete for value types, but a pointer `T` costs 3x
- A decision table, and the rule "write the concrete version first, generalize when you have three"

## Coming From Another Language

| Other | Go |
|---|---|
| TS: generics are free, erased, everywhere | compiled per shape; a pointer `T` pays a dictionary cost |
| Rust: `impl Trait` and `dyn Trait` are both idiomatic | the interface is the default; a type parameter is the exception |
| TS `.map().filter().reduce()` chains | a `for` loop is idiomatic; no method chaining on slices |
| Rust: zero-cost generics are the baseline | Go's interfaces are cheap enough that generics often buy nothing |
| TS: `<T>` for a one-method callback | a plain interface parameter, every time |

Go's generics were added in 1.18 to solve two specific problems: container types
and slice/map utilities. Interfaces already solved runtime polymorphism in 2009
and still do. The question is never "could this be generic" — it almost always
could — but "does the type parameter remove a conversion, an assertion, or a
duplicated function body?" If it does not, it is noise.

## Wrong: an interface already does it

```go
package main

import (
  "fmt"
  "io"
  "os"
  "strings"
)

func GreetGeneric[W io.Writer](w W, name string) {
  fmt.Fprintf(w, "hello %s\n", name)
}

func Greet(w io.Writer, name string) {
  fmt.Fprintf(w, "hello %s\n", name)
}

type Validator interface {
  Validate() error
}

func ValidateAllGeneric[T Validator](xs []T) error {
  for _, x := range xs {
    if err := x.Validate(); err != nil {
      return err
    }
  }
  return nil
}

func ValidateAll(xs []Validator) error {
  for _, x := range xs {
    if err := x.Validate(); err != nil {
      return err
    }
  }
  return nil
}

type Form struct{ Name string }

func (f Form) Validate() error {
  if f.Name == "" {
    return fmt.Errorf("name is required")
  }
  return nil
}

func main() {
  Greet(os.Stdout, "go")
  GreetGeneric(os.Stdout, "go")
  var b strings.Builder
  Greet(&b, "buffer")
  fmt.Print(b.String())

  fmt.Println(ValidateAllGeneric([]Form{{"a"}, {""}}))
  fmt.Println(ValidateAll([]Validator{Form{"a"}, Form{""}}))
}
```

Output:

```
hello go
hello go
hello buffer
name is required
name is required
```

`GreetGeneric` and `Greet` do the same thing. The generic version is worse on
three counts: callers see `[W io.Writer]` in the docs, the compiler generates an
instantiation per argument type, and `W` is useless — the body only calls
`io.Writer` methods, so nothing is gained by knowing the concrete type.

This is the "constraint with one method" antipattern. The rule: if the body uses
*only* methods from the constraint, and `T` appears in exactly one parameter, use
an interface parameter.

`ValidateAll` is the one real difference worth knowing: the generic version takes
`[]Form` directly, the interface version needs `[]Validator`, and Go will not
convert `[]Form` to `[]Validator` implicitly. So if your callers hold
`[]ConcreteType` and copying it into a `[]Interface` is a real cost, the generic
form earns its keep. If they hold a mixed slice already, it does not.

## Wrong: you only ever use one type

```go
package main

import "fmt"

type Celsius float64

func FormatGeneric[T ~float64](t T) string { return fmt.Sprintf("%.1f", float64(t)) }

func Format(c Celsius) string { return fmt.Sprintf("%.1f", float64(c)) }

func main() {
  fmt.Println(FormatGeneric(Celsius(21.5)), Format(21.5))
}
```

Output:

```
21.5 21.5
```

Identical output; `Format(21.5)` is shorter at the call site because the untyped
constant converts implicitly, which the generic version blocks. Write the
concrete version. Generalize when you have three callers with three types — not
two, and never zero.

## Wrong: you need runtime type behaviour

If the function must branch on the dynamic type, the type parameter is dead
weight — you will convert to `any` and switch anyway (9.1), which means the
parameter should have been an interface or `any` from the start.

## Wrong: reflection and encoding boundaries

There is one legitimate generic shape at an encoding boundary, and it is narrow:

```go
package main

import (
  "encoding/json"
  "fmt"
)

func DecodeInto[T any](data []byte) (T, error) {
  var v T
  err := json.Unmarshal(data, &v)
  return v, err
}

type Config struct {
  Host string `json:"host"`
  Port int    `json:"port"`
}

func main() {
  c, err := DecodeInto[Config]([]byte(`{"host":"h","port":1}`))
  fmt.Printf("%+v %v\n", c, err)

  xs, err := DecodeInto[[]int]([]byte(`[1,2,3]`))
  fmt.Println(xs, err)

  bad, err := DecodeInto[Config]([]byte(`{"port":"nope"}`))
  fmt.Printf("%+v %v\n", bad, err)
}
```

Output:

```
{Host:h Port:1} <nil>
[1 2 3] <nil>
{Host: Port:0} json: cannot unmarshal string into Go struct field Config.port of type int
```

This is worth having — it replaces `var c Config; json.Unmarshal(data, &c)` with
one expression. But notice `json.Unmarshal` still works entirely through
`reflect`, so the type parameter bought *syntax*, not type safety or speed. Any
generic wrapper over a reflection-based API has this ceiling: go past "name the
target type once" and you are fighting the library.

## Wrong: heterogeneous collections

Generics give you a container of *one* type, chosen per instantiation. They
cannot give you a list of mixed types.

```go
package main

import "fmt"

type Bag[T any] struct{ items []T }

func (b *Bag[T]) Add(v T) { b.items = append(b.items, v) }

func main() {
  var b Bag[any]
  b.Add(1)
  b.Add("two")
  b.Add(3.0)
  fmt.Println(b.items)

  var typed Bag[int]
  typed.Add(1)
  typed.Add("two")
  fmt.Println(typed.items)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:18:13: cannot use "two" (untyped string constant) as int value in argument to typed.Add
```

`Bag[any]` compiles and holds mixed values — but `Bag[any]` is a `[]any` with
extra steps. If you want heterogeneity, write `[]any` and say so.

## Wrong: Map/Filter/Reduce chains

```go
package main

import (
  "fmt"
  "strings"
)

func Map[T, U any](xs []T, f func(T) U) []U {
  out := make([]U, len(xs))
  for i, x := range xs {
    out[i] = f(x)
  }
  return out
}

func Filter[T any](xs []T, keep func(T) bool) []T {
  var out []T
  for _, x := range xs {
    if keep(x) {
      out = append(out, x)
    }
  }
  return out
}

func Reduce[T, A any](xs []T, init A, f func(A, T) A) A {
  acc := init
  for _, x := range xs {
    acc = f(acc, x)
  }
  return acc
}

type User struct {
  Name   string
  Age    int
  Active bool
}

var users = []User{
  {"ada", 36, true},
  {"bob", 17, true},
  {"cy", 44, false},
  {"dee", 22, true},
}

func withGenerics() string {
  return strings.Join(
    Map(
      Filter(users, func(u User) bool { return u.Active && u.Age >= 18 }),
      func(u User) string { return strings.ToUpper(u.Name) },
    ),
    ", ",
  )
}

func withLoop() string {
  var names []string
  for _, u := range users {
    if u.Active && u.Age >= 18 {
      names = append(names, strings.ToUpper(u.Name))
    }
  }
  return strings.Join(names, ", ")
}

func main() {
  fmt.Println(withGenerics())
  fmt.Println(withLoop())
  fmt.Println(Reduce(users, 0, func(n int, u User) int { return n + u.Age }))
  total := 0
  for _, u := range users {
    total += u.Age
  }
  fmt.Println(total)
}
```

Output:

```
ADA, DEE
ADA, DEE
119
119
```

The loop is seven lines; the pipeline is eight and reads inside-out because Go
has no method chaining — `Filter` is the innermost call but the first operation.
Add a third stage and the nesting gets worse. The loop also allocates once
instead of twice, and a debugger can step through it.

This is not a matter of taste the Go team left open. The standard library
shipped `slices.SortFunc`, `slices.IndexFunc`, `slices.DeleteFunc`,
`slices.ContainsFunc`, and `slices.CompactFunc` — and deliberately no
`slices.Map`, no `slices.Filter`, no `slices.Reduce`. The ones that shipped
either cannot be written as a clean loop (sorting) or return a scalar. The ones
that did not are the ones where a loop wins.

Idiomatic: `for` loop. Keep `Map` and `Filter` in your own package if your team
wants them, and expect reviewers to prefer the loop.

## Wrong: when callers must annotate everything

If every call site needs `F[string, int, bool](...)` because inference cannot
reach the parameters (9.4), the generality is costing the caller more than it
saves. Either reorder the parameters so inference works, or write the concrete
functions.

## Right: containers

The original motivation, and still the clearest win. `Stack[T]`, `Cache[K, V]`,
`Set[T]`, `Tree[T]`, `OrderedMap[K, V]` — anything that stores values of a type
the container does not care about. 9.3's `Cache[K, V]` is the shape: without
generics it is `map[any]any` plus assertions at every boundary.

## Right: slices and maps utilities

The `slices` and `maps` packages exist only because of generics, and they
replaced a thousand hand-written `indexOfString` functions. Note their
signatures: `func Index[S ~[]E, E comparable](s S, v E) int` — two type
parameters so named slice types survive (9.2).

## Right: constraints-based numeric code

```go
package main

import (
  "cmp"
  "fmt"
)

type Number interface {
  ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64
}

func Sum[T Number](xs []T) T {
  var t T
  for _, x := range xs {
    t += x
  }
  return t
}

func MinMax[T cmp.Ordered](xs []T) (T, T) {
  lo, hi := xs[0], xs[0]
  for _, x := range xs[1:] {
    lo, hi = min(lo, x), max(hi, x)
  }
  return lo, hi
}

type Bytes int64

func main() {
  fmt.Println(Sum([]int{1, 2, 3}), Sum([]float64{1.5, 2.5}), Sum([]Bytes{100, 200}))
  fmt.Println(MinMax([]int{5, 1, 9}))
  fmt.Println(MinMax([]string{"go", "c", "rust"}))
}
```

Output:

```
6 4 300
1 9
c rust
```

There is no interface-based alternative here. Writing this with interfaces means
an `Add(x) Adder` method per type and boxing every value — the benchmark below
shows what that costs.

## Right: deleting any + assertion boilerplate

The strongest argument, because the generic version removes a *panic*.

```go
package main

import "fmt"

type AnyStore struct{ m map[string]any }

func (s *AnyStore) Put(k string, v any) { s.m[k] = v }
func (s *AnyStore) GetInt(k string) int { return s.m[k].(int) }

type Store[V any] struct{ m map[string]V }

func NewStore[V any]() *Store[V] { return &Store[V]{m: map[string]V{}} }

func (s *Store[V]) Put(k string, v V) { s.m[k] = v }
func (s *Store[V]) Get(k string) (V, bool) {
  v, ok := s.m[k]
  return v, ok
}

func main() {
  old := &AnyStore{m: map[string]any{}}
  old.Put("n", 42)
  fmt.Println(old.GetInt("n"))

  new := NewStore[int]()
  new.Put("n", 42)
  fmt.Println(new.Get("n"))
  fmt.Println(new.Get("missing"))

  old.Put("s", "oops")
  fmt.Println(old.GetInt("s"))
}
```

This program panics on purpose. Output:

```
42
42 true
0 false
panic: interface conversion: interface {} is string, not int

goroutine 1 [running]:
main.(*AnyStore).GetInt(...)
	/tmp/scratch/main.go:8
main.main()
	/tmp/scratch/main.go:31 +0x38c
exit status 2
```

`Store[int].Put("n", "oops")` would not have compiled. That is the entire value
proposition of generics in one line: a runtime panic became a compile error. When
you can show that, use the type parameter.

## Right: type-safe sync wrappers and typed helpers

```go
package main

import (
  "fmt"
  "sync"
)

type TypedMap[K comparable, V any] struct {
  m sync.Map
}

func (t *TypedMap[K, V]) Store(k K, v V) { t.m.Store(k, v) }

func (t *TypedMap[K, V]) Load(k K) (V, bool) {
  v, ok := t.m.Load(k)
  if !ok {
    var zero V
    return zero, false
  }
  return v.(V), true
}

func (t *TypedMap[K, V]) Range(f func(K, V) bool) {
  t.m.Range(func(k, v any) bool { return f(k.(K), v.(V)) })
}

func Collect[T any](ch <-chan T, n int) []T {
  out := make([]T, 0, n)
  for v := range ch {
    out = append(out, v)
    if len(out) == n {
      return out
    }
  }
  return out
}

type Server struct {
  addr string
  port int
}

type Option[T any] func(*T)

func WithAddr(a string) Option[Server] { return func(s *Server) { s.addr = a } }
func WithPort(p int) Option[Server]    { return func(s *Server) { s.port = p } }

func New[T any](opts ...Option[T]) *T {
  var t T
  for _, o := range opts {
    o(&t)
  }
  return &t
}

func main() {
  var tm TypedMap[string, int]
  tm.Store("a", 1)
  tm.Store("b", 2)
  fmt.Println(tm.Load("a"))
  fmt.Println(tm.Load("z"))
  sum := 0
  tm.Range(func(_ string, v int) bool { sum += v; return true })
  fmt.Println(sum)

  ch := make(chan int, 5)
  var wg sync.WaitGroup
  wg.Add(1)
  go func() {
    defer wg.Done()
    for i := 1; i <= 5; i++ {
      ch <- i
    }
    close(ch)
  }()
  fmt.Println(Collect(ch, 3))
  wg.Wait()

  s := New(WithAddr("localhost"), WithPort(8080))
  fmt.Printf("%+v\n", *s)
}
```

Output:

```
1 true
0 false
3
[1 2 3]
{addr:localhost port:8080}
```

`TypedMap` is the honest version of this pattern: the assertions still happen
inside `Load` and `Range`, but they are now provably safe because `Store` is the
only way in. The boilerplate moved from every call site to one place. `Option[T]`
is the functional-options pattern (4.8) with the target type parameterised, which
is how `errgroup`-style helpers and most config builders are written now.

## Performance

Four implementations of "sum 1000 ints", benchmarked with `go test -bench . -benchmem`
on an Apple M4 Pro, Go 1.27.1, darwin/arm64. Numbers are machine-specific; the
*ratios* are the point.

```
goos: darwin
goarch: arm64
pkg: m09bench
cpu: Apple M4 Pro
BenchmarkConcrete-12      4972933    240.4 ns/op       0 B/op      0 allocs/op
BenchmarkGeneric-12       5151447    235.8 ns/op       0 B/op      0 allocs/op
BenchmarkInterface-12      201596   5635   ns/op    7816 B/op    977 allocs/op
BenchmarkAnyAssert-12     4018468    298.1 ns/op       0 B/op      0 allocs/op
```

`Generic` matches `Concrete` exactly — for a value type like `int`, the compiler
generates real specialised code and `+=` is a machine add. `Interface` (an
`Adder` interface with an `Add` method) is 24x slower and allocates 977 times,
because every intermediate result boxes into an interface. `AnyAssert` (a
`[]any` with a type assertion per element) is 25% slower with no allocations,
since the values were already boxed when the slice was built.

Now the same comparison where `T` is a *pointer* type:

```
BenchmarkPtrConcrete-12   4609314    244.3 ns/op       0 B/op      0 allocs/op
BenchmarkPtrGeneric-12    1636756    731.5 ns/op       0 B/op      0 allocs/op
BenchmarkPtrInterface-12  1385054    867.2 ns/op       0 B/op      0 allocs/op
```

The generic version is now 3x slower than concrete and only 16% faster than the
interface. This is GCShape stenciling. The compiler does not generate one copy
per type argument; it generates one per *shape*, and every pointer type has the
same shape, so `NamesGeneric[*A]` and `NamesGeneric[*B]` share a single body.
That body cannot know which concrete method to call, so it receives a hidden
dictionary argument and loads the method address from it — an indirect call,
much like an interface's itable lookup. You get the type safety; you do not get
the devirtualization.

So: generics are free for value types doing arithmetic, and roughly
interface-priced for pointer types calling methods. If you reached for a type
parameter to avoid interface dispatch on a `*T`, measure first.

Two more costs in one line each. Compile time grows with the number of distinct
shapes instantiated, noticeably in packages with deeply generic APIs. Binary size
grows the same way, since each shape is a separate function body.

## The decision table

| Situation | Use |
|---|---|
| the body calls only constraint methods, `T` in one parameter | interface parameter |
| callers hold `[]Concrete` and copying to `[]Interface` is real cost | type parameter |
| storing values of a caller-chosen type | generic type |
| the container must hold mixed types | `[]any` / `map[string]any` |
| arithmetic over several numeric types | type parameter + type set |
| you must branch on the dynamic type | interface, or `any` + type switch |
| one type today, maybe more later | concrete; generalize at three |
| removing a type assertion that can panic | type parameter |
| a slice or map utility | type parameter with `[S ~[]E, E any]` |
| a method needs its own type parameter | Go 1.27+: fine; older: a free function |
| hot path with pointer `T` and method calls | measure; the dictionary may erase the win |
| every call site needs explicit brackets | reorder the parameters, or go concrete |

## Quick Reference

| Smell | Replacement |
|---|---|
| `[T io.Writer](w T, ...)` | `(w io.Writer, ...)` |
| `[T any](v T) string` with `any(v).(type)` inside | `(v any) string` |
| `Bag[any]` | `[]any` |
| `Map(Filter(xs, p), f)` | a `for` loop |
| one instantiation in the whole repo | the concrete type |
| `[T Constraint]` where the body ignores `T`'s identity | an interface |

| Keeper | Why |
|---|---|
| `Cache[K, V]`, `Set[T]`, `Stack[T]` | no `any`, no assertions |
| `Sum[T Number]`, `MinMax[T cmp.Ordered]` | no interface alternative exists |
| `Store[V]` replacing `map[string]any` | a panic became a compile error |
| `TypedMap[K, V]` over `sync.Map` | assertions confined to one file |
| `Option[T]` functional options | one options machine for every config type |
| `slices` / `maps` | the standard library's own answer |

## Common Mistakes

**Making a function generic because the constraint is an interface you already
have.** If the body only calls those methods, take the interface.

**Writing `Map`/`Filter` chains because they are idiomatic elsewhere.** They are
not idiomatic in Go; the standard library omitted them on purpose.

**Using `Bag[any]` for a mixed collection.** That is `[]any`; say so.

**Assuming a type parameter is always faster than an interface.** For a pointer
`T` calling methods it is within 16% of the interface, because of the shape
dictionary.

**Generalizing on the second caller.** Write it concretely twice; generalize at
three.

**Keeping a generic wrapper around a `reflect`-based API and expecting speed.**
The reflection cost dominates; you bought syntax only.

## Exercises

1. Take the `Map`/`Filter` pipeline above, add a sort and a group-by stage, and
   write both versions. Count lines and allocations with `-benchmem`, then decide.
2. Find a generic function in your own code whose constraint has exactly one
   method. Rewrite it as an interface parameter and diff the call sites.
3. Reproduce the pointer-`T` benchmark on your machine with 1, 2, and 8 distinct
   pointer types instantiated. Explain why the timing does not change with the
   count.
4. Write the interface-based `Sum` (an `Adder` interface) and confirm the
   allocation count matches the element count. Explain where each allocation
   comes from.
5. Convert a `map[string]any` config holder in a real project to a set of typed
   accessors. Count the type assertions you deleted and the panics they could
   have thrown.
6. Build the same `LRU` cache twice — once `map[any]any`, once `LRU[K, V]`. Run
   both under `go vet` and a fuzz test that passes wrong-typed keys, and report
   which failures became compile errors.

---

[← 9.4 Type Inference and Its Limits](04-inference.md) · [Course Index](../README.md) · [10.1 Goroutines and the Scheduler →](../10-concurrency-basics/01-goroutines.md)
