[← 9.2 Constraints, comparable, and cmp.Ordered](02-constraints.md) · [Course Index](../README.md) · [9.4 Type Inference and Its Limits →](04-inference.md)

# 9.3 — Generic Types and Methods

## What You'll Learn

- `type Stack[T any] struct` and the receiver form `func (s *Stack[T]) Push(v T)`
- An uninstantiated generic type is not a type; `Stack[int]` is
- Self-referential generic types work; generic type aliases work (Go 1.24+)
- Generic methods with their own type parameters work (Go 1.27+) — `Result[T].Map` chains
- A complete, concurrent, type-safe `Cache[K, V]`

## Coming From Another Language

| Other | Go |
|---|---|
| TS `class Stack<T>` — params implicit on methods | the receiver repeats them: `(s *Stack[T])` |
| Rust `impl<T> Stack<T> { fn push(&mut self, v: T) }` | one `func` per method, params in the receiver |
| Rust `impl Stack<i32> { ... }` specialised impl | impossible; no per-instantiation methods |
| TS `Stack` as a bare type | `Stack` alone is a compile error; always `Stack[T]` |
| Rust `Default::default()` on a generic struct | `var s Stack[int]` is already usable if the zero value is |
| TS `type Set<T> = Map<T, true>` | `type Set[T comparable] = map[T]struct{}` — Go 1.24+ |

## A generic struct and its methods

The type parameters are declared once on the type, and every method's receiver
repeats them.

```go
package main

import "fmt"

type Stack[T any] struct {
  items []T
}

func NewStack[T any]() *Stack[T] {
  return &Stack[T]{}
}

func (s *Stack[T]) Push(v T) {
  s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
  if len(s.items) == 0 {
    var zero T
    return zero, false
  }
  v := s.items[len(s.items)-1]
  s.items = s.items[:len(s.items)-1]
  return v, true
}

func (s *Stack[T]) Len() int { return len(s.items) }

func (s Stack[T]) String() string { return fmt.Sprint(s.items) }

type Pair[K comparable, V any] struct {
  Key K
  Val V
}

func (p Pair[K, V]) Swapped() Pair[K, V] { return p }

type Node[T any] struct {
  Val  T
  Next *Node[T]
}

func (n *Node[T]) Len() int {
  if n == nil {
    return 0
  }
  return 1 + n.Next.Len()
}

func main() {
  s := NewStack[string]()
  s.Push("a")
  s.Push("b")
  fmt.Println(s, s.Len())
  v, ok := s.Pop()
  fmt.Println(v, ok, s)
  var empty Stack[int]
  fmt.Println(empty, empty.Len())
  x, ok := empty.Pop()
  fmt.Println(x, ok)

  lit := Stack[int]{items: []int{1, 2, 3}}
  fmt.Println(lit, lit.Len())

  p := Pair[string, int]{"a", 1}
  fmt.Println(p, p.Swapped())

  head := &Node[int]{1, &Node[int]{2, &Node[int]{3, nil}}}
  fmt.Println(head.Len(), head.Val, head.Next.Next.Val)

  var ns fmt.Stringer = &Stack[int]{items: []int{9}}
  fmt.Println(ns)
}
```

Output:

```
[a b] 2
b true [a]
[] 0
0 false
[1 2 3] 3
{a 1} {a 1}
3 1 3
[9]
```

Five things worth pulling out.

`var empty Stack[int]` works with no constructor. The zero value has a nil
`items` slice, and `append` on nil allocates, so the type is usable immediately.
Idiomatic: design generic containers so the zero value is valid, exactly as
`sync.Mutex` and `bytes.Buffer` do.

`T` in the receiver is a *declaration*, not a use — you could write
`func (s *Stack[E]) Len() int` and `E` would bind to the same parameter. Keep the
names matching the type declaration; nothing good comes from renaming.

`Node[T]` holds a `*Node[T]`. Self-reference is fine because the instantiation is
finite: `Node[int]` refers to `Node[int]`. What does not work is an infinitely
expanding reference like `Node[Node[T]]`.

`Stack[T]` satisfies `fmt.Stringer` once instantiated. Generic types implement
interfaces normally; the method set of `Stack[int]` is just a method set.

The `String` method takes a value receiver while the rest take pointers. That is
a real mixed-receiver smell in production code, done here so `fmt.Println(s)` and
`fmt.Println(lit)` both print nicely.

## An uninstantiated generic type is not a type

```go
package main

import "fmt"

type Stack[T any] struct {
  items []T
}

func main() {
  var s Stack
  fmt.Println(s)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:9: cannot use generic type Stack[T any] without instantiation
```

The same error appears for a composite literal `Stack{items: []int{1}}`, for a
field of type `Stack`, and for a function parameter of type `Stack`. There is no
context in which the bare name is a type.

## Constraints on the type apply to every method

A constraint declared on the type is in force inside all its methods — you do not
repeat it, and you cannot widen it.

```go
package main

import (
  "cmp"
  "fmt"
)

type SortedSet[T cmp.Ordered] struct {
  items []T
}

func (s *SortedSet[T]) Add(v T) bool {
  i, found := binarySearch(s.items, v)
  if found {
    return false
  }
  s.items = append(s.items, v)
  copy(s.items[i+1:], s.items[i:])
  s.items[i] = v
  return true
}

func (s *SortedSet[T]) Has(v T) bool {
  _, found := binarySearch(s.items, v)
  return found
}

func (s *SortedSet[T]) Items() []T { return s.items }

func binarySearch[T cmp.Ordered](xs []T, v T) (int, bool) {
  lo, hi := 0, len(xs)
  for lo < hi {
    mid := (lo + hi) / 2
    switch cmp.Compare(xs[mid], v) {
    case 0:
      return mid, true
    case -1:
      lo = mid + 1
    default:
      hi = mid
    }
  }
  return lo, false
}

func main() {
  var s SortedSet[string]
  for _, w := range []string{"rust", "go", "ts", "go"} {
    fmt.Print(s.Add(w), " ")
  }
  fmt.Println()
  fmt.Println(s.Items(), s.Has("go"), s.Has("c"))

  var n SortedSet[int]
  for _, v := range []int{5, 1, 9, 1, 3} {
    n.Add(v)
  }
  fmt.Println(n.Items())
}
```

Output:

```
true true true false 
[go rust ts] true false
[1 3 5 9]
```

`Add` returns `false` on the duplicate `"go"`, and both instantiations stay
sorted. `binarySearch` is a free generic function with its own `[T cmp.Ordered]`
rather than a method, because it does not need the receiver — and note it could
not have been a method with its own type parameter before Go 1.27 anyway. The
insert is the usual append-then-shift: grow by one, `copy` the tail right, write
into the gap.

## Generic type aliases

Go 1.24 made type aliases parameterisable. Both forms work on 1.27:

```go
package main

import "fmt"

type Stack[T any] struct {
  items []T
}

type Set[T comparable] = map[T]struct{}

type IntStack = Stack[int]

type Pairs[K comparable, V any] = map[K][]V

func main() {
  s := Set[string]{"a": {}, "b": {}}
  fmt.Println(len(s))
  var is IntStack
  is.items = append(is.items, 1)
  fmt.Println(is.items)
  p := Pairs[string, int]{"x": {1, 2}}
  fmt.Println(p)
  fmt.Printf("%T\n", s)
}
```

Output:

```
2
[1]
map[x:[1 2]]
map[string]struct {}
```

`IntStack` is an alias for an *instantiated* generic type — the oldest form,
legal since 1.18. `Set[T comparable]` is the generic alias, new in 1.24. The
`%T` line proves it is an alias and not a definition: the dynamic type is
`map[string]struct{}`, so `Set[string]` and `map[string]struct{}` are
interchangeable everywhere, including as map keys and in assignments.

## Embedding a generic type

```go
package main

import "fmt"

type List[T any] struct {
  items []T
}

func (l *List[T]) Add(v T)  { l.items = append(l.items, v) }
func (l *List[T]) Len() int { return len(l.items) }

type Counted[T any] struct {
  List[T]
  adds int
}

func (c *Counted[T]) Add(v T) {
  c.List.Add(v)
  c.adds++
}

type Named struct {
  List[string]
  Label string
}

type Sized interface {
  Len() int
}

func main() {
  var c Counted[int]
  c.Add(1)
  c.Add(2)
  fmt.Println(c.Len(), c.adds)

  n := Named{Label: "tags"}
  n.Add("go")
  fmt.Println(n.Label, n.Len(), n.items)

  var s Sized = &c
  fmt.Println(s.Len())

  sizes := []Sized{&Counted[int]{}, &Named{}, &List[float64]{}}
  for _, x := range sizes {
    fmt.Print(x.Len(), " ")
  }
  fmt.Println()
}
```

Output:

```
2 2
tags 1 [go]
2
0 0 0 
```

Two embedding shapes: `Counted[T]` forwards its own parameter down to
`List[T]`, while `Named` pins the embedding to `List[string]` and is itself a
plain non-generic type. Promotion works as usual — `n.Add` and `n.items` reach
through the embedded `List[string]`. And the last two lines show the escape
hatch: different instantiations are different types, but they can share an
interface, so `[]Sized` holds `Counted[int]`, `Named`, and `List[float64]`
together.

## Generic methods with their own type parameters

Go 1.27 lifted the restriction that a method cannot introduce a type parameter.
Until then the only way to transform a `Result[T]` into a `Result[U]` was a
package-level function, because `func (r Result[T]) Map[U any](...)` was a
compile error. Now it works, and the Rust-style chain is available:

```go
package main

import "fmt"

type Result[T any] struct {
  val T
  err error
}

func Ok[T any](v T) Result[T]          { return Result[T]{val: v} }
func Err[T any](e error) Result[T]     { return Result[T]{err: e} }
func (r Result[T]) IsOk() bool         { return r.err == nil }
func (r Result[T]) Unwrap() (T, error) { return r.val, r.err }

func (r Result[T]) Map[U any](f func(T) U) Result[U] {
  if r.err != nil {
    return Result[U]{err: r.err}
  }
  return Result[U]{val: f(r.val)}
}

func main() {
  r := Ok(21).Map(func(i int) string { return fmt.Sprint(i * 2) })
  fmt.Println(r.Unwrap())

  e := Err[int](fmt.Errorf("boom")).Map(func(i int) string { return "never" })
  fmt.Println(e.Unwrap())
  fmt.Println(r.IsOk(), e.IsOk())

  chained := Ok("7").Map(func(s string) int { return len(s) }).Map(func(n int) bool { return n == 1 })
  fmt.Println(chained.Unwrap())
}
```

Output:

```
42 <nil>
 boom
true false
true <nil>
```

The method's `[U any]` sits alongside the receiver's `[T]`, and `U` is inferred
from the callback at each link in the chain. Two caveats carry over from 9.1: an
interface method still cannot have type parameters, so `Result[T]` cannot satisfy
a `Mappable` interface; and `r.Map` without brackets is not a method value.

Be aware this is very new. On a `go.mod` with `go 1.26` or lower you get
`generic method requires go1.27 or later`, so in shared code the
package-level-function form is still the portable one:

```go
func MapResult[T, U any](r Result[T], f func(T) U) Result[U]
```

## A complete example: Cache[K, V]

Everything together — two type parameters, a constraint on `K`, a mutex, a
helper generic struct for the entry, a method taking a function, and the zero
value of `V` on every miss path.

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

type entry[V any] struct {
  val     V
  expires time.Time
}

type Cache[K comparable, V any] struct {
  mu  sync.RWMutex
  ttl time.Duration
  m   map[K]entry[V]
}

func NewCache[K comparable, V any](ttl time.Duration) *Cache[K, V] {
  return &Cache[K, V]{ttl: ttl, m: make(map[K]entry[V])}
}

func (c *Cache[K, V]) Set(k K, v V) {
  c.mu.Lock()
  defer c.mu.Unlock()
  c.m[k] = entry[V]{val: v, expires: time.Now().Add(c.ttl)}
}

func (c *Cache[K, V]) Get(k K) (V, bool) {
  c.mu.RLock()
  e, ok := c.m[k]
  c.mu.RUnlock()
  if !ok || time.Now().After(e.expires) {
    var zero V
    return zero, false
  }
  return e.val, true
}

func (c *Cache[K, V]) GetOrLoad(k K, load func(K) (V, error)) (V, error) {
  if v, ok := c.Get(k); ok {
    return v, nil
  }
  v, err := load(k)
  if err != nil {
    var zero V
    return zero, err
  }
  c.Set(k, v)
  return v, nil
}

func (c *Cache[K, V]) Len() int {
  c.mu.RLock()
  defer c.mu.RUnlock()
  return len(c.m)
}

type User struct {
  ID   int
  Name string
}

func main() {
  c := NewCache[int, User](50 * time.Millisecond)
  calls := 0
  load := func(id int) (User, error) {
    calls++
    if id == 0 {
      return User{}, fmt.Errorf("user %d: not found", id)
    }
    return User{ID: id, Name: fmt.Sprintf("user%d", id)}, nil
  }

  u, err := c.GetOrLoad(7, load)
  fmt.Println(u, err, calls)
  u, err = c.GetOrLoad(7, load)
  fmt.Println(u, err, calls, "(cached)")

  _, err = c.GetOrLoad(0, load)
  fmt.Println(err, calls, c.Len())

  time.Sleep(60 * time.Millisecond)
  _, ok := c.Get(7)
  fmt.Println("after ttl:", ok)

  strs := NewCache[string, []byte](time.Minute)
  strs.Set("k", []byte("hello"))
  b, ok := strs.Get("k")
  fmt.Println(string(b), ok, strs.Len())

  var wg sync.WaitGroup
  counts := NewCache[int, int](time.Minute)
  for i := range 100 {
    wg.Add(1)
    go func() {
      defer wg.Done()
      counts.Set(i%10, i)
    }()
  }
  wg.Wait()
  fmt.Println("concurrent keys:", counts.Len())
}
```

Output:

```
{7 user7} <nil> 1
{7 user7} <nil> 1 (cached)
user 0: not found 2 1
after ttl: false
hello true 1
concurrent keys: 10
```

Clean under `go vet` and `go run -race`. `V` is a struct in one instantiation and
a `[]byte` in another, and `[]byte` is not comparable — which is fine, because
only `K` is constrained by `comparable`. The error path returns `var zero V`
rather than `User{}`, because `V` is not known to be a struct.

Note `entry[V]` is a separate generic type instantiated inside `Cache[K, V]`'s
own definition. Generic types compose like any other types.

## Generic constructors

`NewCache[int, User](...)` had to name both parameters. Inference works from
arguments, and a constructor often has none that mention `T`.

```go
package main

import "fmt"

type Stack[T any] struct{ items []T }

func NewStack[T any](items ...T) *Stack[T] { return &Stack[T]{items} }

func main() {
  a := NewStack(1, 2, 3)
  fmt.Println(a.items)
  b := NewStack[string]()
  fmt.Println(b.items == nil)
  c := Stack[int]{items: []int{1}}
  fmt.Println(c.items)
}
```

Output:

```
[1 2 3]
true
[1]
```

`NewStack(1, 2, 3)` infers `T` from the variadic arguments; `NewStack[string]()`
has nothing to infer from and must say so. 9.4 covers this failure in detail.
Idiomatic: if the zero value is usable, prefer `var s Stack[int]` over a
constructor — it reads better and sidesteps the question entirely.

## Methods cannot be added from another package

You cannot write methods on `Stack[int]` from a package that does not declare
`Stack`, exactly as with non-generic types; wrap it in your own named type
(`type MyStack struct { pkg.Stack[int] }`) if you need to extend it.

## Quick Reference

| Form | Meaning |
|---|---|
| `type S[T any] struct{...}` | generic type declaration |
| `type S[K comparable, V any] struct{...}` | multiple parameters, separate constraints |
| `S[int]{}` | instantiated composite literal |
| `var s S[int]` | zero value, no constructor needed |
| `func (s *S[T]) M(v T)` | method; receiver repeats the parameters |
| `func (s *S[T]) M[U any](f func(T) U)` | generic method, Go 1.27+ |
| `func NewS[T any]() *S[T]` | generic constructor |
| `type Node[T any] struct{ next *Node[T] }` | self-reference, legal |
| `type IntStack = S[int]` | alias of an instantiated type |
| `type Set[T comparable] = map[T]struct{}` | generic alias, Go 1.24+ |
| `struct{ S[T]; n int }` | embedding, parameter forwarded |
| `struct{ S[string] }` | embedding a fixed instantiation |

| Not allowed | Instead |
|---|---|
| `var s S` | `var s S[int]` |
| a bare `S` as a field or parameter type | always instantiate |
| an interface method with type parameters | a generic function taking the interface |
| a method specialised for `S[int]` | branch inside the method, or a free function |
| adding methods to another package's `S[int]` | embed it in your own type |

## Common Mistakes

**Forgetting the type parameters on the receiver.** Write
`func (s *Stack[T]) Push(v T)`, never `func (s *Stack) Push(v T)`.

**Using the bare generic name as a type.** `Stack` is not a type; `Stack[int]`
is.

**Writing a constructor when the zero value already works.** Prefer
`var s Stack[int]`, and reserve constructors for types with a map or channel to
initialise.

**Returning `V{}` from a method of `Cache[K, V]`.** `V` may not be a struct; use
`var zero V`.

**Relying on generic methods in a library.** They need `go 1.27` in every
consumer's `go.mod`; export a package-level function too.

**Expecting different instantiations to be assignable.** `Stack[int]` and
`Stack[int64]` are unrelated types; share an interface if you need a mixed
collection.

## Exercises

1. Add `Peek() (T, bool)` and `Clear()` to `Stack[T]`, and a `Clone()` that
   returns a `*Stack[T]` sharing nothing with the original. Prove the independence
   by mutating one.
2. Replace `SortedSet`'s hand-rolled `binarySearch` with `slices.BinarySearch`
   and `slices.Insert`. Benchmark both and explain why the stdlib version needs
   the `[S ~[]E, E cmp.Ordered]` signature.
3. Build `OrderedMap[K comparable, V any]` preserving insertion order, with
   `Set`, `Get`, `Delete`, and `Keys`. Make `Delete` O(1) amortised and say what
   you traded away.
4. Give `Cache[K, V]` a `Range(func(K, V) bool)` method that does not hold the
   lock while calling the callback. Explain the race you avoided.
5. Try to declare `type Weird[T any] struct { next Weird[Weird[T]] }` and capture
   the error. Then make the self-reference legal.
6. Write a `Tree[T cmp.Ordered]` with `Insert` and an in-order `All()` iterator
   returning `iter.Seq[T]`. Compare the result with `slices.Sorted`.

---

[← 9.2 Constraints, comparable, and cmp.Ordered](02-constraints.md) · [Course Index](../README.md) · [9.4 Type Inference and Its Limits →](04-inference.md)
