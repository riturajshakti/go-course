[← 4.8 Constructors and the Functional Options Pattern](../04-types-structs/08-constructors-options.md) · [Course Index](../README.md) · [5.2 Interface Values, nil, and the Typed-nil Trap →](02-interface-values-nil.md)

# 5.1 — Implicit Satisfaction

## What You'll Learn

- There is no `implements` keyword: a type satisfies an interface by having the methods, and nothing else
- Satisfaction is checked at the *assignment*, not at the type declaration — so the error appears far from the type
- `method has pointer receiver` is the error you will hit most; it comes from method sets, not from interfaces
- `var _ Iface = (*T)(nil)` is how you move that check back to the type's own file
- Interfaces belong in the package that *consumes* them, not the one that provides the type

## Coming From Another Language

| Other | Go |
|---|---|
| Java `class Foo implements Bar` | nothing — having the methods *is* implementing |
| TS `class Foo implements Bar` (optional, checked) | no annotation exists at all |
| TS structural typing on object shapes | the same idea, but on method sets only |
| Rust `impl Trait for Type`, needed in the trait's or type's crate | no impl block, no coherence rule, no orphan rule |
| Python duck typing, discovered at call time | duck typing, checked at compile time |

The closest analogue is TypeScript's structural typing: if the shape matches, it fits. The
difference from Rust and Java is that the *provider* of a type writes nothing to opt in. You
can define an interface today and have it satisfied by a type written five years ago in a
package you do not control.

## An interface and two types that satisfy it

```go
package main

import "fmt"

type Shape interface {
  Area() float64
  Perimeter() float64
}

type Rect struct {
  W, H float64
}

func (r Rect) Area() float64      { return r.W * r.H }
func (r Rect) Perimeter() float64 { return 2 * (r.W + r.H) }

type Square struct {
  S float64
}

func (s Square) Area() float64      { return s.S * s.S }
func (s Square) Perimeter() float64 { return 4 * s.S }

func describe(s Shape) {
  fmt.Printf("%T area=%.1f perim=%.1f\n", s, s.Area(), s.Perimeter())
}

func main() {
  describe(Rect{3, 4})
  describe(Square{5})
}
```

Output:

```
main.Rect area=12.0 perim=14.0
main.Square area=25.0 perim=20.0
```

Neither `Rect` nor `Square` mentions `Shape`. The method sets match, so they fit. `%T` on an
interface value prints the *dynamic* type — the concrete type currently inside it. That is the
subject of 5.2.

## Satisfaction is checked at the assignment

Nothing about `Rect` is validated when `Rect` is declared. The compiler checks only at the
point where a `Rect` is used as a `Shape`. Remove `describe(Rect{3,4})` and a `Rect` missing
`Area` compiles fine.

```go
package main

type Stringer interface {
  String() string
}

type Point struct {
  X, Y int
}

func (p Point) Describe() string { return "point" }

func main() {
  var s Stringer = Point{1, 2}
  _ = s
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:14:20: cannot use Point{…} (value of struct type Point) as Stringer value in variable declaration: Point does not implement Stringer (missing method String)
```

The error names the missing method. That part is easy. The next one is not.

## The pointer-receiver error

This is the single most common compile error Go newcomers hit.

```go
package main

import "fmt"

type Counter interface {
  Inc()
  Value() int
}

type Hits struct {
  n int
}

func (h *Hits) Inc()      { h.n++ }
func (h Hits) Value() int { return h.n }

func main() {
  var c Counter = Hits{}
  fmt.Println(c)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:18:19: cannot use Hits{} (value of struct type Hits) as Counter value in variable declaration: Hits does not implement Counter (method Inc has pointer receiver)
```

`Hits` *has* an `Inc` method. You can call `h.Inc()` on a `Hits` variable. But `Hits` does not
satisfy `Counter`, because the rule is about method *sets*, not about what you can type.

The fix is one character:

```go
package main

import "fmt"

type Counter interface {
  Inc()
  Value() int
}

type Hits struct {
  n int
}

func (h *Hits) Inc()      { h.n++ }
func (h Hits) Value() int { return h.n }

func main() {
  var c Counter = &Hits{}
  c.Inc()
  c.Inc()
  fmt.Println(c.Value(), fmt.Sprintf("%T", c))
}
```

Output:

```
2 *main.Hits
```

## Method sets

4.4 introduced receivers. Here is the rule that makes interfaces behave the way they do:

| Type | Method set |
|---|---|
| `T` | methods declared with receiver `T` |
| `*T` | methods declared with receiver `T` **or** `*T` |

So `*T` can always stand in for `T`, never the reverse. Every interface a `T` satisfies, a `*T`
also satisfies.

The reason value types do not get pointer methods is addressability. `h.Inc()` on a variable
works because the compiler rewrites it as `(&h).Inc()` — it needs the address. An interface
value holds a *copy*, with no address you could take, so the rewrite is impossible and the
method cannot be in `T`'s set.

You can see the same rule outside interfaces entirely. Map elements are not addressable:

```go
package main

type Hits struct{ n int }

func (h *Hits) Inc() { h.n++ }

func main() {
  m := map[string]Hits{"a": {}}
  m["a"].Inc()
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:10: cannot call pointer method Inc on Hits
```

And when it *is* addressable, mutation through a value receiver silently does nothing — a
different bug with the same root:

```go
package main

import "fmt"

type Hits struct{ n int }

func (h *Hits) Inc()      { h.n++ }
func (h Hits) Value() int { return h.n }

func main() {
  h := Hits{}
  h.Inc()
  fmt.Println(h.Value())

  m := map[string]Hits{"a": {}}
  fmt.Println(m["a"].Value())
}
```

Output:

```
1
0
```

Idiomatic: if any method on a type needs a pointer receiver, give *every* method on that type a
pointer receiver, and pass `*T` everywhere. Mixed receiver sets are the source of most of this
confusion.

## The compile-time assertion idiom

Because satisfaction is checked at assignment, a type can drift out of conformance and you will
only learn at the far-away call site — or in a different package. Pin it next to the type:

```go
var _ Closer = (*File)(nil)
```

`(*File)(nil)` is a typed nil pointer: no allocation, no value, just a thing with the right
type for the compiler to check. Assigning to `_` means it exists purely for that check.

Get the pointer-ness wrong and you get the familiar error, now in the right file:

```go
package main

type Closer interface{ Close() error }

type File struct{}

func (f *File) Close() error { return nil }

var _ Closer = File{}

func main() {}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:16: cannot use File{} (value of struct type File) as Closer value in variable declaration: File does not implement Closer (method Close has pointer receiver)
```

Idiomatic: one `var _ Iface = (*T)(nil)` line per interface your exported type promises to
satisfy, placed directly under the type or its constructor.

## One type, many interfaces

A type does not pick an interface, so it accumulates them.

```go
package main

import (
  "fmt"
  "strings"
)

type Named interface{ Name() string }
type Greeter interface{ Greet() string }
type Resetter interface{ Reset() }

type User struct {
  name string
}

func (u User) Name() string  { return u.name }
func (u User) Greet() string { return "hi, " + u.name }
func (u *User) Reset()       { u.name = "" }

func main() {
  u := &User{"ada"}
  var n Named = u
  var g Greeter = u
  var r Resetter = u
  fmt.Println(n.Name(), g.Greet())
  r.Reset()
  fmt.Printf("%q\n", n.Name())

  var sb strings.Builder
  var w interface{ WriteString(string) (int, error) } = &sb
  w.WriteString("also satisfied")
  fmt.Println(sb.String())
}
```

Output:

```
ada hi, ada
""
also satisfied
```

Three interface variables, one underlying `*User` — `r.Reset()` is visible through `n`. The
last two lines show an *anonymous* interface type declared inline, satisfied by
`*strings.Builder`, which was written long before you declared it.

## Non-struct types satisfy interfaces too

Any named type can have methods, so any named type can satisfy an interface. Structs are not
special.

```go
package main

import "fmt"

type Stringer interface{ String() string }

type Celsius float64

func (c Celsius) String() string { return fmt.Sprintf("%.1f°C", float64(c)) }

type IDs []int

func (ids IDs) String() string { return fmt.Sprintf("%d ids", len(ids)) }

type Handler func(string) string

func (h Handler) String() string { return "handler" }

var _ Stringer = Celsius(0)
var _ Stringer = IDs(nil)
var _ Stringer = Handler(nil)

func main() {
  var ss []Stringer = []Stringer{Celsius(21.5), IDs{1, 2, 3}, Handler(nil)}
  for _, s := range ss {
    fmt.Printf("%T -> %s\n", s, s)
  }
}
```

Output:

```
main.Celsius -> 21.5°C
main.IDs -> 3 ids
main.Handler -> handler
```

A method on a *function* type is how `http.HandlerFunc` turns a plain function into an
`http.Handler`. 5.6 covers that.

## Define the interface where it is consumed

This is where Go's design culture diverges most sharply from Java and C#. There, the provider
of `UserStore` ships a `UserStore` interface alongside it. In Go, the interface is declared by
the *caller*, listing only the methods that caller needs.

```go
package main

import (
  "fmt"
  "os"
  "strings"
)

type Loader interface {
  Load(key string) (string, bool)
}

func render(l Loader, keys []string) string {
  var sb strings.Builder
  for _, k := range keys {
    v, ok := l.Load(k)
    if !ok {
      v = "<missing>"
    }
    fmt.Fprintf(&sb, "%s=%s\n", k, v)
  }
  return sb.String()
}

type EnvStore struct{}

func (EnvStore) Load(k string) (string, bool) { return os.LookupEnv(k) }

type MapStore map[string]string

func (m MapStore) Load(k string) (string, bool) { v, ok := m[k]; return v, ok }

func main() {
  os.Setenv("REGION", "eu-west-1")
  fmt.Print(render(EnvStore{}, []string{"REGION", "NOPE"}))
  fmt.Print(render(MapStore{"REGION": "local"}, []string{"REGION", "NOPE"}))
}
```

Output:

```
REGION=eu-west-1
NOPE=<missing>
REGION=local
NOPE=<missing>
```

Neither store imports `Loader`, and `render` does not import either store. The dependency
points one way: toward the consumer. The slogan is **accept interfaces, return structs** —
functions take the narrow interface they need, constructors return the concrete type so callers
keep every method. 5.5 makes the case with a before/after.

## Interfaces embed interfaces

An interface can list another interface instead of a method name, which splices in its methods.

```go
package main

import "fmt"

type Reader interface{ Read() string }
type Writer interface{ Write(string) }

type ReadWriter interface {
  Reader
  Writer
}

type Buf struct{ s string }

func (b *Buf) Read() string   { return b.s }
func (b *Buf) Write(s string) { b.s += s }

func main() {
  var rw ReadWriter = &Buf{}
  rw.Write("ab")
  rw.Write("cd")
  var r Reader = rw
  fmt.Println(r.Read())
}
```

Output:

```
abcd
```

Note the last assignment: a `ReadWriter` value is assignable to a `Reader` variable, because
every type satisfying the former satisfies the latter. 5.5 covers composition properly.

## Quick Reference

| Form | Meaning |
|---|---|
| `type I interface { M() }` | interface with one method |
| `type I interface { J; K }` | interface embedding interfaces `J` and `K` |
| `interface{ M() }` | anonymous interface type, usable inline |
| `var i I = v` | compile-time check that `v`'s type satisfies `I` |
| `var _ I = (*T)(nil)` | assert `*T` satisfies `I`, no value created |
| `var _ I = T{}` | assert the *value* type `T` satisfies `I` |

| Rule | Consequence |
|---|---|
| Method set of `T` = value-receiver methods | `T` cannot satisfy an interface needing a pointer method |
| Method set of `*T` = all methods | `*T` satisfies everything `T` does, and more |
| Checked at assignment, not declaration | the error surfaces at the call site |
| No `implements`, no registration | third-party types satisfy your interfaces for free |
| Any named type can have methods | slices, maps, funcs, and `float64` aliases all qualify |

| Error text | Cause |
|---|---|
| `missing method String` | the method does not exist on either receiver form |
| `method Inc has pointer receiver` | you passed `T`; pass `&T` |
| `cannot call pointer method Inc on Hits` | the value is not addressable (map element, function result) |

## Common Mistakes

**Passing `T{}` where the interface needs a pointer method.** Pass `&T{}`; the method set of
`T` excludes pointer-receiver methods.

**Mixing value and pointer receivers on one type.** Pick one — pointer receivers if any method
mutates — so the type's method set is not half-empty.

**Declaring an interface in the package that provides the implementation.** Declare it in the
package that calls it, with only the methods that caller uses.

**Expecting a compile error when you declare a type that is "supposed to" implement an
interface.** Nothing is checked until assignment; add `var _ I = (*T)(nil)` to get the error
where you want it.

**Defining an interface before you have two implementations.** Return the concrete struct until
a second caller or a test actually needs the abstraction.

## Exercises

1. Define `type Temperature interface { Celsius() float64 }` and satisfy it with three types:
   a struct, a `float64`-based named type, and a function type. Put each in its own file and add
   the `var _` assertion to each.
2. Write a type whose `Scale(f float64)` method mutates, and whose `String()` does not. Try to
   satisfy an interface requiring both from a value, read the error, then fix it two ways: by
   passing a pointer, and by making `Scale` return a new value instead of mutating.
3. Take `render`/`Loader` above and add a third implementation that wraps another `Loader` and
   lowercases every key before delegating. Note that you needed no change to `render`.
4. Find a type in the standard library that satisfies an interface you declare yourself (try a
   one-method interface matching something on `*os.File` or `*strings.Reader`), and prove it
   with a `var _` assertion.
5. Declare `type Any interface{}` — an interface with zero methods — and assign five unrelated
   values to it. Predict what you can do with those values before reading 5.4.

---

[← 4.8 Constructors and the Functional Options Pattern](../04-types-structs/08-constructors-options.md) · [Course Index](../README.md) · [5.2 Interface Values, nil, and the Typed-nil Trap →](02-interface-values-nil.md)
