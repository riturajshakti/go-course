[← 4.3 Struct Comparison, Copying, and Memory Layout](03-struct-semantics.md) · [Course Index](../README.md) · [4.5 Struct Embedding and Promotion →](05-embedding.md)

# 4.4 — Methods: Value vs Pointer Receivers

## What You'll Learn

- A value receiver gets a copy, so its mutations are discarded; a pointer receiver mutates the original
- Go auto-takes the address and auto-dereferences for *addressable* values — map elements and function results are not addressable
- The method set rule: `T`'s method set excludes pointer-receiver methods, so `*T` can satisfy an interface when `T` cannot
- A nil `*T` receiver is a legal, callable receiver

## Coming From Another Language

| Other | Go |
|---|---|
| `class C { m() {} }` | `func (c C) m()` declared outside any type block |
| Python `self`, explicit | the receiver, explicit, but named `c`/`s`/`r`, never `self` |
| Rust `&self` / `&mut self` / `self` | value receiver / pointer receiver; no ownership-taking receiver |
| JS `this` is dynamic | the receiver is statically typed; no rebinding, no `bind` |
| Extension methods / monkey patching | impossible across packages — the type must be local |

A method is a function with a receiver. That is the whole difference, and it buys you one
thing: method sets, which is how interfaces are satisfied (module 05).

## Value receiver copies; pointer receiver mutates

```go
package main

import "fmt"

type Counter struct {
  N int
}

func (c Counter) IncValue() {
  c.N++
}

func (c *Counter) IncPointer() {
  c.N++
}

func (c Counter) String() string {
  return fmt.Sprintf("Counter(%d)", c.N)
}

func main() {
  c := Counter{}
  c.IncValue()
  fmt.Println(c.N)

  c.IncPointer()
  fmt.Println(c.N)

  p := &c
  p.IncValue()
  p.IncPointer()
  fmt.Println(c.N)
  fmt.Println(c)
}
```

Output:

```
0
1
2
Counter(2)
```

`IncValue` incremented a copy — a silent no-op, and the compiler says nothing. Note that
`c.IncPointer()` compiles even though `c` is a `Counter`, not a `*Counter`: Go rewrites it
as `(&c).IncPointer()`. The reverse works too — `p.IncValue()` becomes `(*p).IncValue()`.
That symmetry is why receiver choice feels invisible at the call site, and why its one
exception is so surprising.

`String() string` makes the type a `fmt.Stringer`, which is why `fmt.Println(c)` printed
`Counter(2)` instead of `{2}`.

## The auto-address rule needs an addressable value

```go
package main

import "fmt"

type Counter struct {
  N int
}

func (c *Counter) Inc() {
  c.N++
}

func main() {
  m := map[string]Counter{"a": {}}
  m["a"].Inc()
  fmt.Println(m)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:15:10: cannot call pointer method Inc on Counter
```

Go cannot write `&m["a"]`, so it cannot call a pointer method on it. The same applies to a
function's return value:

```go
package main

import "fmt"

type Counter struct {
  N int
}

func (c *Counter) Inc() {
  c.N++
}

func make1() Counter {
  return Counter{}
}

func main() {
  make1().Inc()
  fmt.Println("done")
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:18:11: cannot call pointer method Inc on Counter
```

If `make1` returned `*Counter`, both this and the map version would work. See 4.1 for the
full addressability table.

## The method set rule

This is the rule that bites people, and it is the reason to care about receivers at all.

| Type | Method set contains |
|---|---|
| `T` | methods with receiver `T` |
| `*T` | methods with receiver `T` **and** methods with receiver `*T` |

Interface satisfaction is checked against the method set, with no auto-addressing, because
an interface value holds a copy with no address to take.

```go
package main

import "fmt"

type Stringer interface {
  String() string
}

type Named struct {
  Name string
}

func (n *Named) String() string {
  return "Named:" + n.Name
}

func main() {
  var s Stringer = &Named{Name: "ok"}
  fmt.Println(s.String())

  var bad Stringer = Named{Name: "nope"}
  fmt.Println(bad)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:21:22: cannot use Named{…} (value of struct type Named) as Stringer value in variable declaration: Named does not implement Stringer (method String has pointer receiver)
```

The error message names the cause outright: *method String has pointer receiver*. Remember
that sentence; you will meet it most often when a `json.Unmarshaler`, `error`, or
`http.Handler` implementation is defined on `*T` and something passes a `T`. The fix is
either `&Named{...}` at the call site or a value receiver on the method. Module 05 covers
the interface side in depth.

## A nil pointer receiver is legal

Calling a method on a nil `*T` does not panic. Only *dereferencing* the nil receiver does,
so a method that checks for nil first is perfectly usable.

```go
package main

import "fmt"

type List struct {
  Head *node
}

type node struct {
  Val  int
  Next *node
}

func (l *List) Len() int {
  if l == nil {
    return 0
  }
  n := 0
  for cur := l.Head; cur != nil; cur = cur.Next {
    n++
  }
  return n
}

func (l *List) String() string {
  if l == nil {
    return "<empty>"
  }
  return fmt.Sprintf("list of %d", l.Len())
}

func main() {
  var l *List
  fmt.Println(l.Len())
  fmt.Println(l.String())
  fmt.Println(l)

  real := &List{Head: &node{Val: 1, Next: &node{Val: 2}}}
  fmt.Println(real.Len(), real)
}
```

Output:

```
0
<empty>
<empty>
2 list of 2
```

The method is dispatched statically from the pointer's *type*, not from the pointee, so
there is nothing to load before the call. This is how tree and list types make the nil
pointer a valid empty value. It is also why a nil `*T` stored in an interface is non-nil as
an interface value — the typed-nil trap in 5.2.

## Methods on named non-struct types

Any type you *define* in your package can have methods, not just structs.

```go
package main

import "fmt"

type Celsius float64

func (c Celsius) Fahrenheit() float64 {
  return float64(c)*9/5 + 32
}

func (c Celsius) String() string {
  return fmt.Sprintf("%.1f°C", float64(c))
}

type IDs []int

func (ids IDs) Sum() int {
  t := 0
  for _, v := range ids {
    t += v
  }
  return t
}

func (ids *IDs) Add(v int) {
  *ids = append(*ids, v)
}

func main() {
  c := Celsius(100)
  fmt.Println(c, c.Fahrenheit())

  var ids IDs
  ids.Add(1)
  ids.Add(2)
  fmt.Println(ids, ids.Sum())
}
```

Output:

```
100.0°C 212
[1 2] 3
```

`Add` needs a pointer receiver because `append` may return a new slice header that has to
be written back. `Sum` only reads, so a value receiver is enough — the copied header still
points at the same array. 4.6 covers defined types properly.

## The type must be local to the package

```go
package main

import (
  "fmt"
  "time"
)

func (t time.Time) Tomorrow() time.Time {
  return t.Add(24 * time.Hour)
}

func main() {
  fmt.Println(time.Now().Tomorrow())
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:9: cannot define new methods on non-local type time.Time
./main.go:13:26: time.Now().Tomorrow undefined (type time.Time has no field or method Tomorrow)
```

No extension methods, no monkey patching, no `impl Trait for ForeignType`. To attach
behaviour to someone else's type you define your own: `type Deadline time.Time` (which
loses the original's methods — 4.6) or wrap it in a struct and embed it (4.5).

## Method values and method expressions

```go
package main

import "fmt"

type Counter struct {
  N int
}

func (c Counter) Get() int {
  return c.N
}

func (c *Counter) Inc() {
  c.N++
}

func main() {
  c := Counter{N: 5}

  bound := c.Get
  c.N = 50
  fmt.Println(bound())

  boundPtr := (&c).Inc
  boundPtr()
  fmt.Println(c.N)

  expr := Counter.Get
  fmt.Println(expr(Counter{N: 7}))

  exprPtr := (*Counter).Inc
  exprPtr(&c)
  fmt.Println(c.N)

  fmt.Printf("%T\n%T\n", bound, expr)
}
```

Output:

```
5
51
7
52
func() int
func(main.Counter) int
```

A *method value* (`c.Get`) captures the receiver at the moment it is created. Because `Get`
has a value receiver, `bound` froze a copy with `N == 5`, which is why it printed 5 after
`c.N = 50`. `(&c).Inc` captured the pointer, so it still mutates the live `c`.

A *method expression* (`Counter.Get`) is an ordinary function whose first parameter is the
receiver — the mechanism behind `sort.Slice` callbacks and most "pass a method as a
function" code.

## Choosing a receiver

| Choose | When |
|---|---|
| `*T` | the method mutates the receiver |
| `*T` | `T` is large enough that copying it per call matters |
| `*T` | `T` contains a `sync.Mutex`, `sync.WaitGroup`, or any no-copy type |
| `*T` | any other method on `T` already needs a pointer |
| `T` | `T` is small and immutable in practice (`time.Time`, a `Point`) |
| `T` | you want the type usable as a map key / interface value without `&` |

The consistency rule matters more than the individual calls: pick one receiver kind per
type and use it for every method. Mixed receivers mean `T` and `*T` have different method
sets, so which one satisfies an interface becomes a thing readers have to work out.

Receiver names are short and derived from the type — `c Counter`, `s *Server`, `r *Reader`.
Never `this` or `self`; `go vet` will not complain but every reviewer will.

## Quick Reference

| Syntax | Meaning |
|---|---|
| `func (c T) M()` | value receiver; `c` is a copy |
| `func (c *T) M()` | pointer receiver; `c` points at the original |
| `func (T) M()` | receiver unnamed (unused) |
| `v.M()` | calls `M`; auto-`&v` or auto-`*v` if `v` is addressable |
| `v.M` | method value; receiver captured now |
| `T.M` | method expression; `func(T, ...)` |
| `(*T).M` | pointer method expression; `func(*T, ...)` |

| Rule | Consequence |
|---|---|
| method set of `T` excludes `*T` methods | `T` may fail an interface that `*T` satisfies |
| auto-address requires addressability | `m[k].PtrMethod()` and `f().PtrMethod()` are compile errors |
| nil `*T` receiver is callable | nil can be a valid empty value |
| the type must be declared in this package | no methods on `time.Time`, `int`, or `[]string` |

## Common Mistakes

**Mutating through a value receiver.** The change lands on a copy and is lost; use `*T`.

**Mixing value and pointer receivers on one type.** The two method sets diverge and
interface satisfaction becomes surprising; pick one and stick to it.

**Storing a `T` in an interface when the methods are on `*T`.** You get
`method String has pointer receiver`; pass `&v` instead.

**Calling a pointer method on a map element or a function result.** Neither is addressable;
use `map[K]*T`, or assign to a local variable first.

**Taking a method value from a value receiver and expecting later mutations to show.** The
receiver was copied at capture time; take it from a pointer.

## Exercises

1. Write `type Stack []int` with `Push`, `Pop`, and `Len`. Decide the receiver for each,
   then deliberately give `Push` a value receiver and demonstrate the resulting bug.
2. Define `type Temp float64` with `String()` on the value and a `Set(string) error` on the
   pointer. Satisfy `fmt.Stringer` and `flag.Value` and show which needs `&`.
3. Store a `Counter` in a `map[string]Counter` and try to call a pointer method. Capture the
   error, then make it work with `map[string]*Counter` and with read-modify-write.
4. Build a binary tree whose `Insert` and `InOrder` methods handle a nil `*Node` receiver so
   that `var root *Node; root = root.Insert(5)` works.
5. Write `func apply(f func(Counter) int, c Counter) int` and pass it `Counter.Get` as a
   method expression. Then write the `*Counter` variant and explain why the signature changes.

---

[← 4.3 Struct Comparison, Copying, and Memory Layout](03-struct-semantics.md) · [Course Index](../README.md) · [4.5 Struct Embedding and Promotion →](05-embedding.md)
