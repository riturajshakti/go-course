[← 4.4 Methods: Value vs Pointer Receivers](04-methods.md) · [Course Index](../README.md) · [4.6 Named Types, Type Definitions, and Aliases →](06-named-types.md)

# 4.5 — Struct Embedding and Promotion

## What You'll Learn

- An anonymous field promotes the inner type's fields *and* methods to the outer type
- Embedding is not inheritance: an inner method calling a shadowed method does **not** dispatch to the outer override
- Ambiguity at equal depth is a compile error; a shallower name silently shadows a deeper one
- Embedding an interface in a struct gives you decorators and partial implementations for free

## Coming From Another Language

| Other | Go |
|---|---|
| `class Dog extends Animal` | embedding looks similar and behaves differently — no virtual dispatch |
| Python multiple inheritance + MRO | embedding + depth rules; ties are a compile error, not an MRO walk |
| Ruby `include Module` | the closest analogue: methods are mixed in, `super` does not exist |
| Rust `#[derive(Deref)]` / delegation macros | built into the language as the anonymous field |
| TS `interface A extends B` | works for interfaces too (5.5); for structs it is composition |

Embedding is syntactic delegation: `Outer` gets a field named after the inner type plus
forwarding methods for everything it exposes. It is a mechanism, not a type relationship —
`Dog` is never assignable to `Animal`.

## An anonymous field promotes fields and methods

```go
package main

import "fmt"

type Base struct {
  ID   int
  Name string
}

func (b Base) Describe() string {
  return fmt.Sprintf("Base(%d,%s)", b.ID, b.Name)
}

type Embedded struct {
  Base
  Extra string
}

type Named struct {
  Base  Base
  Extra string
}

func main() {
  e := Embedded{Base: Base{ID: 1, Name: "a"}, Extra: "x"}
  fmt.Println(e.ID, e.Name, e.Extra)
  fmt.Println(e.Describe())
  fmt.Println(e.Base.ID, e.Base.Describe())
  fmt.Printf("%+v\n", e)

  n := Named{Base: Base{ID: 2, Name: "b"}}
  fmt.Println(n.Base.ID)
}
```

Output:

```
1 a x
Base(1,a)
1 Base(1,a)
{Base:{ID:1 Name:a} Extra:x}
2
```

`Embedded` has one field written `Base` — the type name *is* the field name — so both `e.ID`
and `e.Base.ID` reach the same `int`. With the named field (`Named`), only `n.Base.ID`
works. The literal must still use the field name: `Embedded{Base: Base{...}}`. And as
`%+v` shows, the inner struct is not flattened in memory; it is a real nested field.

The implicit field name is the type's *unqualified* name, so embedding `sync.Mutex` creates
a field called `Mutex`, and embedding `*Logger` creates one called `Logger`.

## Embedding is not inheritance

This is the single most important example in the lesson for anyone arriving from an
OO language.

```go
package main

import "fmt"

type Animal struct {
  Name string
}

func (a Animal) Speak() string {
  return "generic noise"
}

func (a Animal) Introduce() string {
  return a.Name + " says " + a.Speak()
}

type Dog struct {
  Animal
}

func (d Dog) Speak() string {
  return "woof"
}

func main() {
  d := Dog{Animal{Name: "rex"}}
  fmt.Println(d.Speak())
  fmt.Println(d.Introduce())
  fmt.Println(d.Animal.Introduce())
}
```

Output:

```
woof
rex says generic noise
rex says generic noise
```

In Java, Python, or TypeScript, `d.Introduce()` would print `rex says woof`. In Go it does
not. `Introduce` is a method on `Animal` whose receiver `a` is an `Animal` — it has no
knowledge that it is embedded in anything, so `a.Speak()` resolves statically to
`Animal.Speak`. There is no vtable entry to override.

Promotion generates a forwarding wrapper, roughly
`func (d Dog) Introduce() string { return d.Animal.Introduce() }`, and that is where the
chain ends. For polymorphic callbacks, give the inner type an interface field or parameter
instead of relying on embedding.

## Ambiguity at the same depth is an error

```go
package main

import "fmt"

type A struct{ N int }

func (A) Who() string { return "A" }

type B struct{ N int }

func (B) Who() string { return "B" }

type C struct {
  A
  B
}

func main() {
  c := C{}
  fmt.Println(c.N)
  fmt.Println(c.Who())
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:20:17: ambiguous selector c.N
./main.go:21:17: ambiguous selector c.Who
```

Both `A.N` and `B.N` are at depth 1, so `c.N` has no unique meaning. Note the type `C`
itself compiles — only the *use* of the ambiguous selector fails. Qualify it (`c.A.N`) or
shadow it.

## Shallower wins

```go
package main

import "fmt"

type A struct{ N int }

func (A) Who() string { return "A" }

type B struct{ N int }

func (B) Who() string { return "B" }

type C struct {
  A
  B
  N int
}

func (c C) Who() string { return "C over " + c.A.Who() + "/" + c.B.Who() }

func main() {
  c := C{N: 9}
  c.A.N = 1
  c.B.N = 2
  fmt.Println(c.N, c.A.N, c.B.N)
  fmt.Println(c.Who())
}
```

Output:

```
9 1 2
C over A/B
```

A field or method declared on the outer type is at depth 0 and wins outright, with no
warning. This is how you "override": declare the method on the outer type and call the
inner one explicitly, which is exactly what `Outer.Inner.Method()` does above. It is
delegation written by hand, and it is the idiomatic shape.

Name resolution: the shallowest depth with exactly one match wins; a tie at that depth is
the ambiguity error above; and a name shadowed at depth 0 hides everything deeper.

## Embedding a pointer

```go
package main

import "fmt"

type Logger struct {
  Prefix string
}

func (l *Logger) Log(msg string) string {
  return l.Prefix + msg
}

type Service struct {
  *Logger
  Name string
}

func main() {
  s := Service{Logger: &Logger{Prefix: "[svc] "}, Name: "api"}
  fmt.Println(s.Log("up"))
  fmt.Println(s.Prefix)

  s.Prefix = "[api] "
  fmt.Println(s.Logger.Prefix)
}
```

Output:

```
[svc] up
[svc] 
[api] 
```

The field is still named `Logger`, and promotion reaches straight through the pointer:
`s.Prefix` is `s.Logger.Prefix`. The trailing spaces are the prefix values themselves.

Embedding `*T` lets several outer values share one inner value, and keeps the outer struct
small. The cost is a nil field:

```go
package main

import "fmt"

type Logger struct {
  Prefix string
}

func (l *Logger) Log(msg string) string {
  return l.Prefix + msg
}

type Service struct {
  *Logger
  Name string
}

func main() {
  var s Service
  fmt.Printf("%q %v\n", s.Name, s.Logger == nil)
  fmt.Println(s.Log("boom"))
}
```

This program panics on purpose. Output:

```
"" true
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=0x10465cc90]

goroutine 1 [running]:
main.(*Logger).Log(...)
	/tmp/scratch/main.go:10
main.main()
	/tmp/scratch/main.go:21 +0x80
exit status 2
```

Embedding a value keeps the zero value of `Service` usable. Embedding a pointer does not —
so that type needs a constructor (4.8).

## Method sets and embedded pointer receivers

```go
package main

import "fmt"

type Inner struct{ N int }

func (i *Inner) Bump() { i.N++ }

type ByValue struct{ Inner }

type ByPointer struct{ *Inner }

type Bumper interface{ Bump() }

func main() {
  v := ByValue{}
  v.Bump()
  fmt.Println(v.N)

  var b Bumper = &v
  b.Bump()
  fmt.Println(v.N)

  p := ByPointer{Inner: &Inner{}}
  var b2 Bumper = p
  b2.Bump()
  fmt.Println(p.N)
}
```

Output:

```
1
2
1
```

The rule composes with 4.4: embedding `Inner` promotes `Bump` into `*ByValue`'s method set
but not `ByValue`'s, so `ByValue` does not satisfy `Bumper` and `&v` is required. Embedding
`*Inner` promotes `Bump` into *both* method sets, because the receiver is already a
pointer — `ByPointer` satisfies `Bumper` as a value. This is why types that must satisfy an
interface as values embed a pointer.

## Embedding sync.Mutex

The canonical real-world embedding, and the reason the type names are what they are.

```go
package main

import (
  "fmt"
  "sync"
)

type Registry struct {
  sync.Mutex
  items map[string]int
}

type SafeCounter struct {
  mu sync.Mutex
  n  int
}

func NewRegistry() *Registry {
  return &Registry{items: map[string]int{}}
}

func (r *Registry) Bump(k string) {
  r.Lock()
  defer r.Unlock()
  r.items[k]++
}

func (c *SafeCounter) Inc() {
  c.mu.Lock()
  defer c.mu.Unlock()
  c.n++
}

func main() {
  r := NewRegistry()
  var wg sync.WaitGroup
  for i := 0; i < 100; i++ {
    wg.Add(1)
    go func() { defer wg.Done(); r.Bump("hits") }()
  }
  wg.Wait()
  fmt.Println(r.items["hits"])
}
```

Output:

```
100
```

`sync.Mutex`'s zero value is ready to use, so embedding it keeps `Registry`'s zero value
valid and `r.Lock()` reads well — but it also promotes `Lock`/`Unlock` into the public API.
Idiomatic: a named `mu sync.Mutex` field unless locking is deliberately part of the
contract. Either way the type must be used through a pointer; copying a struct containing a
mutex copies the lock state, which `go vet`'s copylocks check catches.

## Embedding an interface in a struct

An interface-typed anonymous field promotes the interface's methods, forwarding each to
whatever value is stored. Declare one of them on the outer type and you have a decorator
that overrides a single method without implementing the rest.

```go
package main

import (
  "fmt"
  "strings"
)

type Store interface {
  Get(k string) (string, bool)
  Put(k, v string)
  Delete(k string)
}

type mapStore struct {
  m map[string]string
}

func (s *mapStore) Get(k string) (string, bool) { v, ok := s.m[k]; return v, ok }
func (s *mapStore) Put(k, v string)             { s.m[k] = v }
func (s *mapStore) Delete(k string)             { delete(s.m, k) }

type UpperStore struct {
  Store
}

func (u UpperStore) Put(k, v string) {
  u.Store.Put(k, strings.ToUpper(v))
}

type ReadOnly struct {
  Store
}

func (ReadOnly) Put(k, v string) {
  panic("read-only store")
}

func main() {
  base := &mapStore{m: map[string]string{}}

  var s Store = UpperStore{Store: base}
  s.Put("a", "hello")
  fmt.Println(s.Get("a"))
  s.Delete("a")
  fmt.Println(s.Get("a"))

  var ro Store = ReadOnly{Store: base}
  base.Put("b", "x")
  fmt.Println(ro.Get("b"))

  defer func() { fmt.Println("recovered:", recover()) }()
  ro.Put("b", "y")
}
```

Output:

```
HELLO true
 false
x true
recovered: read-only store
```

`UpperStore` implements `Store` by declaring one method; `Get` and `Delete` are forwarded.
This is how `http.ResponseWriter` middleware wrappers and one-method test doubles are built.

Left nil, the embedded interface is also how you build a partial fake: implement the two
methods your test touches, and any unexpected call panics instead of silently returning
zero values. Interfaces embed in *interfaces* too — composition rather than delegation,
covered in 5.5.

## Embedded structs in JSON

```go
package main

import (
  "encoding/json"
  "fmt"
)

type Meta struct {
  CreatedAt string `json:"created_at"`
  Version   int    `json:"version"`
}

type Flat struct {
  Meta
  Name string `json:"name"`
}

type Nested struct {
  Meta Meta   `json:"meta"`
  Name string `json:"name"`
}

func main() {
  m := Meta{CreatedAt: "2026-01-01", Version: 2}
  fb, _ := json.Marshal(Flat{Meta: m, Name: "flat"})
  nb, _ := json.Marshal(Nested{Meta: m, Name: "nested"})
  fmt.Println(string(fb))
  fmt.Println(string(nb))
}
```

Output:

```
{"created_at":"2026-01-01","version":2,"name":"flat"}
{"meta":{"created_at":"2026-01-01","version":2},"name":"nested"}
```

`encoding/json` flattens an embedded struct's fields into the parent object, which is the
clean way to share a common envelope across payload types. A `json:"meta"` tag on the
embedded field restores nesting; 4.7 and 13.6 go further.

## Quick Reference

| Declaration | Field name | Promotes |
|---|---|---|
| `Base` | `Base` | `Base`'s fields and methods |
| `*Base` | `Base` | same, through the pointer; nil panics |
| `pkg.Base` | `Base` | the unqualified type name is the field name |
| `Base Base` | `Base` | nothing — an ordinary named field |
| `SomeInterface` | `SomeInterface` | the interface's method set, forwarded |

| Situation | Result |
|---|---|
| name at depth 0 and depth 1 | depth 0 wins, silently |
| same name in two fields at depth 1 | `ambiguous selector` compile error on use |
| inner method calls a shadowed method | the inner one runs; **no** virtual dispatch |
| embed `T`, methods on `*T` | only `*Outer` satisfies the interface |
| embed `*T`, methods on `*T` | `Outer` and `*Outer` both satisfy it |
| embedded struct, `encoding/json` | fields are flattened into the parent object |

## Common Mistakes

**Expecting an inner method to call the outer override.** Go has no virtual dispatch; pass
behaviour through an interface field instead of embedding.

**Embedding two types that share a field or method name.** The selector becomes a compile
error; qualify it or declare the name on the outer type.

**Embedding `*T` and relying on the zero value.** The field is nil and the first promoted
call panics; embed a value or provide a constructor.

**Embedding `sync.Mutex` on a type whose API should not expose locking.** It promotes
`Lock`/`Unlock` to callers; use a named `mu` field.

**Embedding a big interface to "inherit" an implementation.** Prefer a named field plus
explicit delegation unless you are deliberately writing a decorator.

Composition over inheritance is not advice in Go — inheritance is simply absent, and
embedding is sugar over the explicit delegation you would otherwise write by hand.

## Exercises

1. Build `type ReadWriteLogger struct { io.Reader; io.Writer }` over a `strings.Reader` and a
   `bytes.Buffer`, then add a `Read` that counts bytes while delegating to the embedded one.
2. Reproduce the `Animal`/`Dog` non-dispatch result, then make `d.Introduce()` print
   `rex says woof` by introducing a `Speaker` interface. Note what had to change.
3. Embed two structs that both have a `Name` field. Capture the ambiguous-selector error,
   then resolve it three ways: qualify, shadow at depth 0, and rename one field.
4. Write a `CountingStore` that embeds a `Store` interface and counts `Get` calls, and a
   `PartialStore` that embeds the interface as nil and implements only `Get`. Show what
   happens when a test calls `Put` on the second.
5. Define a struct embedding `sync.Mutex` and pass it by value to a function. Run
   `go vet` and quote the warning, then fix it with a pointer receiver.

---

[← 4.4 Methods: Value vs Pointer Receivers](04-methods.md) · [Course Index](../README.md) · [4.6 Named Types, Type Definitions, and Aliases →](06-named-types.md)
