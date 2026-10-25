[← 4.5 Struct Embedding and Promotion](05-embedding.md) · [Course Index](../README.md) · [4.7 Struct Tags →](07-struct-tags.md)

# 4.6 — Named Types, Type Definitions, and Aliases

## What You'll Learn

- `type Celsius float64` creates a brand-new type that inherits the operators but **no** methods
- `type Temp = float64` creates an alias: the same type under a second spelling
- Two defined types with the same underlying type never assign without an explicit conversion
- Any type can be defined — slice, map, func, chan — and that is how `http.HandlerFunc` and `sort.Interface` work

## Coming From Another Language

| Other | Go |
|---|---|
| TS `type ID = number` | `type ID = int` — an alias, zero safety gained |
| TS branded types (`number & {__brand}`) | `type ID int` — a real distinct type, the point of the feature |
| Python `NewType('ID', int)` | `type ID int`, but enforced by the compiler, not a checker |
| Rust `struct Id(u32)` (newtype) | `type Id uint32` — same intent, no `.0` to unwrap |
| Rust `type Id = u32` | `type Id = uint32` — also an alias in both languages |
| `typedef` in C | the alias form; the definition form has no C equivalent |

Go has exactly two `type` forms and they do different things. Mixing them up is the
most common source of "why won't this assign" confusion.

## Definition vs alias

```go
package main

import "fmt"

type Celsius float64
type Fahrenheit float64
type Temp = float64

func main() {
  c := Celsius(100)
  var t Temp = 100

  fmt.Printf("%T %T %T\n", c, t, Fahrenheit(212))
  fmt.Println(c*2, c+1, c/4)

  var f float64 = 1.5
  t = f
  fmt.Println(t)

  fmt.Println(float64(c) == 100)
  fmt.Println(Fahrenheit(c))

  const untyped = 37
  var body Celsius = untyped
  fmt.Println(body)
}
```

Output:

```
main.Celsius float64 main.Fahrenheit
200 101 25
1.5
true
100
37
```

Three things to read off this:

`Temp`'s dynamic type prints as `float64`, not `main.Temp` — an alias is not a type, it is a
name for one, so `t = f` needs no conversion.

`Celsius` keeps `float64`'s *operators*: `c*2`, `c+1`, `c/4` all work, and the result is a
`Celsius`. A defined type's operator set comes from its underlying type.

An untyped constant is assignable to any defined type with a compatible underlying type, so
`var body Celsius = 37` compiles with no conversion. This is why `time.Second * 3` works
while `someDuration * someInt` does not.

## Conversion is required between defined types

```go
package main

import "fmt"

type Celsius float64
type Fahrenheit float64

func main() {
  c := Celsius(100)
  var f Fahrenheit = c
  var x float64 = c
  fmt.Println(f, x)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:22: cannot use c (variable of float64 type Celsius) as Fahrenheit value in variable declaration
./main.go:11:19: cannot use c (variable of float64 type Celsius) as float64 value in variable declaration
```

Even assigning back to the underlying `float64` needs `float64(c)`. That friction is the
whole feature: a `UserID` can never be silently passed where an `OrderID` is expected. The
conversion itself is free at run time — it changes the static type, not a bit.

## A definition inherits operators, not methods

```go
package main

import (
  "fmt"
  "time"
)

type Deadline time.Time
type DeadlineAlias = time.Time

func main() {
  d := Deadline(time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC))
  var a DeadlineAlias = time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC)

  fmt.Println(a.Year())
  fmt.Println(time.Time(d).Year())
}
```

Output:

```
2026
2026
```

`a.Year()` works because `DeadlineAlias` *is* `time.Time`. `d.Year()` does not:

```go
package main

import (
  "fmt"
  "time"
)

type Deadline time.Time

func main() {
  d := Deadline(time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC))
  fmt.Println(d.Year())
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:12:17: d.Year undefined (type Deadline has no field or method Year)
```

A defined type starts with an empty method set. You convert to call the original's methods,
or you declare your own — that blank slate is often the reason to define the type. To keep
the methods instead, embed the type in a struct (4.5).

## Defining types on any underlying type

```go
package main

import (
  "fmt"
  "sort"
)

type IntSlice []int

func (s IntSlice) Len() int           { return len(s) }
func (s IntSlice) Less(i, j int) bool { return s[i] < s[j] }
func (s IntSlice) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

type Counts map[string]int

func (c Counts) Total() int {
  t := 0
  for _, v := range c {
    t += v
  }
  return t
}

type Handler func(string) string

func (h Handler) Twice(s string) string {
  return h(h(s))
}

type Pipe chan int

func (p Pipe) SendAll(vs ...int) {
  for _, v := range vs {
    p <- v
  }
  close(p)
}

func main() {
  s := IntSlice{3, 1, 2}
  sort.Sort(s)
  fmt.Println(s)

  c := Counts{"a": 1, "b": 2}
  fmt.Println(c.Total())

  var h Handler = func(s string) string { return s + "!" }
  fmt.Println(h("hi"), h.Twice("hi"))

  p := make(Pipe, 3)
  p.SendAll(1, 2, 3)
  for v := range p {
    fmt.Print(v, " ")
  }
  fmt.Println()
}
```

Output:

```
[1 2 3]
3
hi! hi!!
1 2 3 
```

`IntSlice` with `Len`/`Less`/`Swap` is `sort.Interface`, the pre-generics way to sort —
7.10 compares it with `slices.Sort` and `sort.Slice`. Literals and `make` work on a defined
type exactly as on its underlying type, including `IntSlice{3, 1, 2}` and `make(Pipe, 3)`.

## A function type with methods: http.HandlerFunc

`Handler` above is a toy version of the most important instance of this pattern in the
standard library:

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
  f(w, r)
}
```

A plain function becomes an `http.Handler` by conversion, with no wrapper struct.

```go
package main

import (
  "fmt"
  "net/http"
  "net/http/httptest"
)

func hello(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintln(w, "hello", r.URL.Path)
}

func main() {
  var h http.Handler = http.HandlerFunc(hello)

  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, httptest.NewRequest("GET", "/x", nil))
  fmt.Print(rec.Body.String())

  rec2 := httptest.NewRecorder()
  http.HandlerFunc(hello).ServeHTTP(rec2, httptest.NewRequest("GET", "/y", nil))
  fmt.Print(rec2.Body.String())
}
```

Output:

```
hello /x
hello /y
```

`http.HandlerFunc(hello)` is a type conversion, not a constructor call. Module 16 builds on
this constantly; `sort.Reverse` and `template.FuncMap` are the same trick.

## Underlying types and assignability

Every type has an *underlying type*: for a predeclared type it is itself, and for
`type A B` it is `B`'s underlying type. So `type A Celsius` has underlying type `float64`,
not `Celsius`. The rules that matter:

| `var x T = v` compiles when | Example |
|---|---|
| `T` and `v`'s type are identical | `var a float64 = someFloat64` |
| identical underlying types and at least one of the two is **unnamed** | `var p Pipe = make(chan int)` |
| `T` is an interface and `v`'s type implements it | `var w io.Writer = os.Stdout` |
| channel case: identical element types, one side unnamed, at least one bidirectional | `var s chan<- int = myPipe` |
| `v` is an untyped constant representable in `T` | `var c Celsius = 37` |
| `v` is `nil` and `T` is a pointer, func, slice, map, chan, or interface | `var m Counts = nil` |

"Unnamed" means a type literal like `[]int`, `chan int`, `map[string]int`, or
`func(int) int` — not a `type` name. That is why `[]int` assigns to `IntSlice` and
`IntSlice` assigns to `[]int`, while `IntSlice` and another `type OtherSlice []int` do not
assign to each other.

```go
package main

import "fmt"

type Pipe chan int

func main() {
  p := make(Pipe, 1)
  var send chan<- int = p
  send <- 1
  fmt.Println(<-p)

  plain := make(chan int, 1)
  var pp Pipe = plain
  pp <- 2
  fmt.Println(<-plain)
}
```

Output:

```
1
2
```

## When an alias is actually the right tool

Aliases exist for large-scale refactoring, not for domain modelling. A type can be moved to
a new package while the old name keeps compiling:

```go
package oldpkg

type Config = newpkg.Config
```

Both names now denote one type, so values pass between code that was not migrated and code
that was. Delete the alias when the last caller moves. The other legitimate uses are in the
language itself: `byte = uint8`, `rune = int32`, `any = interface{}`.

```go
package main

import "fmt"

type MyBytes []byte

func main() {
  var b byte = 65
  var u uint8 = b
  fmt.Println(b, u, b == u)

  var a any = "boxed"
  var i interface{} = a
  fmt.Println(i)

  type Score int
  var s Score = 10
  var plain int = int(s)
  fmt.Println(s, plain)

  mb := MyBytes("hi")
  fmt.Println(string(mb), len(mb))

  ch := make(chan int)
  type Ch = chan int
  var c2 Ch = ch
  fmt.Println(c2 == ch)
}
```

Output:

```
65 65 true
boxed
10 10
hi 2
true
```

`byte` and `uint8` interchange with no conversion because they are the same type; `any` and
`interface{}` likewise. Types can be declared inside a function (`type Score int`), which is
handy for local helpers.

If you write an alias expecting type safety, you got nothing. If you write a definition
expecting the underlying type's methods, you got nothing either. Pick deliberately.

## Conversion is not assertion

`T(v)` is a *conversion*: a compile-time-checked change of static type, always valid or
always a compile error.

`v.(T)` is a *type assertion*: a run-time check on the dynamic type inside an interface
value, which can fail and panic (or return `ok == false`). It only applies to interfaces.
That is 5.3.

`Celsius(f)` is a conversion, verified when you compile. `someAny.(string)` is an
assertion, checked when you run.

## Quick Reference

| Form | Creates | Methods of the old type? | Needs conversion? |
|---|---|---|---|
| `type A B` | a new, distinct type | no | yes, both directions |
| `type A = B` | another name for `B` | yes | no |

| Defined on | Example | Typical use |
|---|---|---|
| numeric | `type Celsius float64` | units, IDs, enums with `iota` |
| string | `type Env string` | closed sets of string constants |
| slice | `type IntSlice []int` | `sort.Interface`, domain collections |
| map | `type Counts map[string]int` | methods over a map |
| func | `type Handler func(...)` | `http.HandlerFunc`, options (4.8) |
| chan | `type Pipe chan int` | methods over a channel |
| struct | `type User struct{...}` | the common case (4.2) |

| Conversion | Legal? |
|---|---|
| `Celsius(someFloat64)` | yes — identical underlying type |
| `Fahrenheit(someCelsius)` | yes — explicit, identical underlying type |
| `someCelsius + someFahrenheit` | no — mismatched types |
| `int(someFloat64)` | yes — truncates |
| `string(someInt)` | yes, but means "the rune with that code point"; `go vet` flags it |
| `[]byte(someString)` | yes — copies |

## Common Mistakes

**Writing `type ID = int` and expecting type safety.** That is an alias; use `type ID int`
to get a distinct type the compiler will police.

**Writing `type Deadline time.Time` and expecting `.Year()`.** A definition starts with no
methods; convert back, or embed `time.Time` in a struct.

**Converting between two defined types over and over at a boundary.** If the conversions
pile up, the type was drawn in the wrong place; put it at the edge of the system.

**Using `string(n)` on an integer to format it.** That yields the rune for code point `n`;
use `strconv.Itoa` or `fmt.Sprint`.

**Confusing `T(v)` with `v.(T)`.** The first is a compile-time conversion; the second is a
run-time assertion on an interface value and can panic.

## Exercises

1. Define `type UserID int` and `type OrderID int`, write `func fetchUser(UserID)`, and show
   the exact compiler error when an `OrderID` is passed. Then show the conversion that
   silences it and argue whether it should be allowed at that call site.
2. Define `type Env string` with constants `Dev`, `Staging`, `Prod` and a `Valid() bool`
   method. Demonstrate that an arbitrary string literal still converts, and discuss what
   that means for enum safety in Go.
3. Implement `type ByLength []string` satisfying `sort.Interface` and sort with `sort.Sort`.
   Then do the same job with `slices.SortFunc` and compare line counts.
4. Write `type Middleware func(http.Handler) http.Handler` with a `Then(Middleware)` method
   that composes two, and chain three middlewares through it.
5. Create `type Celsius float64` and `type Fahrenheit float64` with `To` methods both ways.
   Then add `type Kelvin = Celsius` and show which conversions the alias removes.

---

[← 4.5 Struct Embedding and Promotion](05-embedding.md) · [Course Index](../README.md) · [4.7 Struct Tags →](07-struct-tags.md)
