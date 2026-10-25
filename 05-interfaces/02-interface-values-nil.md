[← 5.1 Implicit Satisfaction](01-implicit-satisfaction.md) · [Course Index](../README.md) · [5.3 Type Assertions and Type Switches →](03-assertions-type-switches.md)

# 5.2 — Interface Values, nil, and the Typed-nil Trap

## What You'll Learn

- An interface value is two words: a dynamic **type** and a **value**; `== nil` is true only when *both* are empty
- A nil `*T` stored in an interface makes that interface **non-nil** — the most expensive bug in Go
- Calling a method on a nil interface panics; calling one on a nil pointer *inside* an interface often works fine
- `==` on interfaces compares type and value, and panics at runtime if the dynamic type is uncomparable

## Coming From Another Language

| Other | Go |
|---|---|
| JS `null` / `undefined` — one sentinel | `nil` is per-type; a nil pointer and a nil interface are different values |
| Python `x is None` — always reliable | `err != nil` can be true when the pointer inside is None-ish |
| Rust `Option<Box<dyn Trait>>` — `None` is unambiguous | a `dyn Trait` slot holding a null pointer is a real value |
| TS `obj?.method()` returns undefined | `i.Method()` on a nil interface panics; on a nil pointer it may succeed |

This is the one lesson in the module that will cost you a production incident if you skip it.
Everything here follows from a single implementation detail: an interface variable is a pair.

## An interface value is a (type, value) pair

```go
package main

import (
  "fmt"
  "reflect"
)

type Stringer interface{ String() string }

type Temp float64

func (t Temp) String() string { return fmt.Sprintf("%.1fC", float64(t)) }

func main() {
  var s Stringer
  fmt.Printf("%v %T %t\n", s, s, s == nil)

  s = Temp(21.5)
  fmt.Printf("%v %T %t\n", s, s, s == nil)
  fmt.Println(reflect.TypeOf(s), reflect.ValueOf(s))

  s = Temp(0)
  fmt.Printf("%v %T %t\n", s, s, s == nil)
}
```

Output:

```
<nil> <nil> true
21.5C main.Temp false
main.Temp 21.5C
0.0C main.Temp false
```

Read the three columns as (value, type, is-nil). The zero interface has no type and no value,
so it is nil. Store `Temp(0)` — a zero *value* — and the interface is not nil, because the type
half is now filled in. **The value being zero has nothing to do with the interface being nil.**

The pair really is two machine words:

```go
package main

import (
  "fmt"
  "unsafe"
)

type Stringer interface{ String() string }

func main() {
  var s Stringer
  var e error
  var a any
  fmt.Println(unsafe.Sizeof(s), unsafe.Sizeof(e), unsafe.Sizeof(a))
  fmt.Println(unsafe.Sizeof(int64(0)), unsafe.Sizeof(&struct{}{}))
}
```

Output:

```
16 16 16
8 8
```

Sixteen bytes on a 64-bit machine: one pointer to a type descriptor (plus method table), one
pointer to the data. `i == nil` compiles to "are both words zero?".

## The typed-nil trap

Here is the bug. A function returns a concrete error pointer type, assigned into an `error`
return. The pointer is nil. The interface is not.

```go
package main

import "fmt"

type MyError struct{ Code int }

func (e *MyError) Error() string { return fmt.Sprintf("failed with code %d", e.Code) }

func doWork(fail bool) *MyError {
  if fail {
    return &MyError{Code: 500}
  }
  return nil
}

func doWorkBroken(fail bool) error {
  var err *MyError
  if fail {
    err = &MyError{Code: 500}
  }
  return err
}

func main() {
  if err := doWork(false); err != nil {
    fmt.Println("direct: got error:", err)
  } else {
    fmt.Println("direct: no error")
  }

  err := doWorkBroken(false)
  fmt.Printf("broken: err=%v err==nil=%t type=%T\n", err, err == nil, err)
  if err != nil {
    fmt.Println("broken: took the error branch")
  }
}
```

Output:

```
direct: no error
broken: err=<nil> err==nil=false type=*main.MyError
broken: took the error branch
```

Look at that middle line carefully. `err` *prints* as `<nil>`. `%T` says it holds a
`*main.MyError`. And `err == nil` is **false**. Every `if err != nil` in your codebase now
reports a failure that did not happen, and the log line reads `<nil>`, giving you nothing to
go on.

The conversion happened at `return err`. Assigning a `*MyError` into an `error` slot fills in
the type word unconditionally — the runtime does not inspect the pointer to decide whether to
leave the interface empty. Two non-zero... one non-zero word is enough.

## Anatomy of the three nils

```go
package main

import "fmt"

type MyError struct{ Code int }

func (e *MyError) Error() string { return fmt.Sprintf("code %d", e.Code) }

func main() {
  var p *MyError
  var i error = p
  var j error

  fmt.Printf("p:  %T %v   p == nil: %t\n", p, p, p == nil)
  fmt.Printf("i:  %T %v   i == nil: %t\n", i, i, i == nil)
  fmt.Printf("j:  %T %v   j == nil: %t\n", j, j, j == nil)
  fmt.Println("i == j:", i == j)

  if v, ok := i.(*MyError); ok {
    fmt.Println("unwrapped pointer is nil:", v == nil)
  }
}
```

Output:

```
p:  *main.MyError <nil>   p == nil: true
i:  *main.MyError <nil>   i == nil: false
j:  <nil> <nil>   j == nil: true
i == j: false
unwrapped pointer is nil: true
```

Three distinct things:

| | type word | value word | `== nil` |
|---|---|---|---|
| `p` (a `*MyError`) | n/a — it is not an interface | nil | `true` |
| `i` (an `error` holding nil `*MyError`) | `*main.MyError` | nil | `false` |
| `j` (a zero `error`) | nil | nil | `true` |

`i == j` is false because their type words differ. And the assertion in the last two lines
succeeds — the interface genuinely holds a `*MyError`, which happens to be nil.

## The fixes

Two of them, and you want both habits.

```go
package main

import "fmt"

type MyError struct{ Code int }

func (e *MyError) Error() string { return fmt.Sprintf("failed with code %d", e.Code) }

func fixedExplicit(fail bool) error {
  if fail {
    return &MyError{Code: 500}
  }
  return nil
}

func fixedConcrete(fail bool) *MyError {
  if fail {
    return &MyError{Code: 500}
  }
  return nil
}

func main() {
  fmt.Println(fixedExplicit(false) == nil, fixedExplicit(true) == nil)

  e := fixedConcrete(false)
  fmt.Println(e == nil)

  var asIface error = e
  fmt.Println(asIface == nil)
}
```

Output:

```
true false
true
false
```

1. **`return nil` literally.** In `fixedExplicit`, the success path writes the untyped `nil`
   into an `error` return, so both words stay zero. Never `return someNilTypedVar` from a
   function whose result is an interface.
2. **Never declare a concrete error type as a return type.** `fixedConcrete` is honest about
   itself — `e == nil` works — but the last two lines show the landmine is still armed: the
   moment anybody assigns the result to an `error`, it becomes non-nil. Declare the return as
   `error`.

Idiomatic: interface-typed returns, `return nil` on the happy path. Rule of thumb: if a
function's signature says `error`, no concrete error type should appear in any `return`
statement except where a real error is constructed.

## How it bites in real code

The dangerous version is not one function — it is a helper returning a concrete type and a
caller that forwards it.

```go
package main

import (
  "errors"
  "fmt"
)

type ValidationError struct{ Field string }

func (v *ValidationError) Error() string { return "invalid field " + v.Field }

func validate(name string) *ValidationError {
  if name == "" {
    return &ValidationError{Field: "name"}
  }
  return nil
}

func save(name string) error {
  if err := validate(name); err != nil {
    return fmt.Errorf("save: %w", err)
  }
  return nil
}

func saveBroken(name string) error {
  return validate(name)
}

func main() {
  fmt.Println("good, correct :", save("ada"))
  fmt.Println("bad,  correct :", save(""))
  fmt.Println("good, broken  :", saveBroken("ada"), saveBroken("ada") == nil)

  var ve *ValidationError
  fmt.Println("errors.As on the broken nil:", errors.As(saveBroken("ada"), &ve), ve == nil)
}
```

Output:

```
good, correct : <nil>
bad,  correct : save: invalid field name
good, broken  : <nil> false
errors.As on the broken nil: true true
```

`saveBroken` is a one-line forward and it is wrong. `save` is correct because the `if err !=
nil` guard operates on the *concrete* pointer, where nil means nil, and the happy path returns
a literal `nil`.

The last line is the cruellest part: `errors.As` reports success and writes a nil pointer into
`ve`. Your error-handling code then dereferences `ve.Field` and panics, several layers away
from the bug. `errors.As` is covered in 8.3.

## Calling a method on a nil interface panics

```go
package main

import "fmt"

type Greeter interface{ Greet() string }

func main() {
  var g Greeter
  fmt.Println(g.Greet())
}
```

This program panics on purpose. Output:

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=0x100d4354c]

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:9 +0x1c
exit status 2
```

No type word means no method table to look the method up in. The `pc=` address varies between
runs and machines; the message does not.

## A nil pointer inside an interface can work fine

Here is the mirror image, and it is not a bug — it is a technique. The method receives nil as
its receiver, and a pointer receiver can test for it.

```go
package main

import "fmt"

type Tree struct {
  Left, Right *Tree
  Val         int
}

func (t *Tree) Sum() int {
  if t == nil {
    return 0
  }
  return t.Val + t.Left.Sum() + t.Right.Sum()
}

type Summer interface{ Sum() int }

func main() {
  var t *Tree
  fmt.Println(t.Sum())

  var s Summer = t
  fmt.Println(s.Sum(), s == nil)

  full := &Tree{Val: 1, Left: &Tree{Val: 2}, Right: &Tree{Val: 3}}
  fmt.Println(full.Sum())
}
```

Output:

```
0
0 false
6
```

The recursion needs no nil checks on the children because `Sum` handles a nil receiver itself.
Dispatch works because the interface's type word is `*Tree`, which is all the runtime needs to
find `Sum`; the nil receiver is just an argument.

This only works for **pointer** receivers. A value receiver must dereference to make the copy:

```go
package main

import "fmt"

type Box struct{ N int }

func (b Box) Get() int { return b.N }

type Getter interface{ Get() int }

func main() {
  var p *Box
  var g Getter = p
  fmt.Println(g.Get())
}
```

This program panics on purpose. Output:

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=0x1044a754c]

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:14 +0x1c
exit status 2
```

The method set of `*Box` includes `Get`, so the assignment compiles; the auto-generated wrapper
dereferences `p` to build the `Box` copy, and that is where it dies.

## Comparing interface values

`==` compares both halves: equal only if the dynamic types are identical *and* the values are
equal under that type's `==`.

```go
package main

import "fmt"

func main() {
  var a, b any = 1, 1
  fmt.Println(a == b)

  a, b = 1, int64(1)
  fmt.Println(a == b)

  a, b = "x", "x"
  fmt.Println(a == b)

  type P struct{ X, Y int }
  a, b = P{1, 2}, P{1, 2}
  fmt.Println(a == b)

  a, b = []int{1}, []int{1}
  fmt.Println(a == b)
}
```

This program panics on purpose. Output:

```
true
false
true
true
panic: runtime error: comparing uncomparable type []int

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:20 +0x190
exit status 2
```

Line by line: `1 == 1` is true. `int(1) == int64(1)` is **false** — the types differ, and the
compiler will not complain because both sides are `any`. Strings and comparable structs behave
as expected. And then comparing two interfaces holding slices panics *at runtime*, because
slices, maps, and functions have no `==`.

This is a genuine hole in the type system: `a == b` type-checks for any two values of interface
type, and whether it panics depends on data.

The same panic reaches you through map keys:

```go
package main

import "fmt"

type Tags struct {
  Name string
  List []string
}

func main() {
  seen := map[any]bool{}
  seen[Tags{Name: "a"}] = true
  fmt.Println(len(seen))
}
```

This program panics on purpose. Output:

```
panic: runtime error: hash of unhashable type main.Tags

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:12 +0x94
exit status 2
```

`Tags` is uncomparable because one field is a slice — and `map[any]bool` accepts it at compile
time. A `map[Tags]bool` would have been rejected outright.

Where comparison *is* well-defined, it is the basis of sentinel errors:

```go
package main

import "fmt"

type E1 struct{}
type E2 struct{}

func (E1) Error() string { return "e1" }
func (E2) Error() string { return "e2" }

var ErrGone = E1{}

func main() {
  var a error = E1{}
  var b error = E1{}
  var c error = E2{}
  fmt.Println(a == b, a == c, a == ErrGone)

  type key struct{ a, b int }
  var k1 any = key{1, 2}
  var k2 any = key{1, 2}
  fmt.Println(k1 == k2)

  var p1 any = &E1{}
  var p2 any = &E1{}
  fmt.Println(p1 == p2, p1 == p1)
}
```

Output:

```
true false true
true
false true
```

Empty structs of the same type compare equal. Two *pointers* to empty structs do not — pointer
equality is address equality, which is why `errors.New("x") != errors.New("x")` and why
sentinels must be package-level variables compared by identity (8.4).

## Detecting a typed nil

There is no language-level test. If you must (writing a logging or middleware layer that
receives errors from code you do not control), reflect:

```go
package main

import (
  "fmt"
  "reflect"
)

type MyError struct{}

func (e *MyError) Error() string { return "boom" }

func isNil(i any) bool {
  if i == nil {
    return true
  }
  v := reflect.ValueOf(i)
  switch v.Kind() {
  case reflect.Ptr, reflect.Map, reflect.Slice, reflect.Chan, reflect.Func, reflect.Interface:
    return v.IsNil()
  }
  return false
}

func main() {
  var p *MyError
  var i error = p
  fmt.Println(i == nil, isNil(i))
  fmt.Println(isNil(nil), isNil(&MyError{}), isNil(0), isNil([]int(nil)))

  v := reflect.ValueOf(i)
  fmt.Println(v.Kind(), v.Type(), v.IsNil())
}
```

Output:

```
false true
true false false true
ptr *main.MyError true
```

Note `isNil(0)` is false: `reflect.Value.IsNil` panics on kinds that cannot be nil, hence the
`switch`. This is a workaround for other people's bugs, not a pattern to design around. Fix
the signature instead.

## Quick Reference

| Expression | Meaning |
|---|---|
| `var i I` | both words nil; `i == nil` is true |
| `i = v` (concrete `v`) | type word = `v`'s type, value word = `v`; never nil |
| `i = nil` | resets both words to nil |
| `i == nil` | true only if the type word is empty |
| `%T` on `i` | the dynamic type, or `<nil>` |
| `%v` on `i` | the dynamic value, which may print `<nil>` while `i != nil` |
| `unsafe.Sizeof(i)` | 16 on a 64-bit platform, any interface type |
| `i == j` | dynamic types equal **and** values equal; may panic |
| `reflect.ValueOf(i).IsNil()` | whether the *inner* value is nil |

| Situation | Result |
|---|---|
| method call on nil interface | panic: nil pointer dereference |
| method call on nil `*T` inside interface, pointer receiver that checks nil | works |
| method call on nil `*T` inside interface, value receiver | panic on the implicit deref |
| `==` on interfaces holding slices, maps, or funcs | runtime panic |
| uncomparable value as a `map[any]T` key | runtime panic: hash of unhashable type |
| nil `*T` returned from a function declared `error` | non-nil error |

## Common Mistakes

**Declaring a function's return type as `*MyError` instead of `error`.** Any caller assigning
it to an `error` gets a non-nil error from your success path; always return `error`.

**Writing `var err *MyError; ...; return err` in a function returning `error`.** Return the
untyped `nil` literal on the success path instead.

**Assuming `i == nil` is false means the value inside is usable.** It may be a nil pointer;
the type word alone makes the interface non-nil.

**Comparing `any` values with `==` on data you do not control.** It panics on slices, maps,
and funcs; use `reflect.DeepEqual` or compare concrete types after asserting.

**Treating a nil interface and a nil pointer as the same thing when logging.** `%v` prints
`<nil>` for both; print `%T` too, or you will spend an afternoon on it.

## Exercises

1. Write `func f() error` that returns a nil `*MyError` and a `func g() error` that returns a
   literal nil. Call both in a loop, print `err`, `err == nil`, and `%T` for each, and explain
   the difference from the two-word representation.
2. Build a middleware-shaped chain: three functions each taking and returning `error`, with the
   innermost one returning a typed nil. Find how far the bogus error propagates and where you
   would put the single fix.
3. Implement `Sum()` on a linked list with a pointer receiver that tolerates nil, store a nil
   list in an interface, and call it. Then change the receiver to a value receiver and capture
   the panic.
4. Write a function `equalAny(a, b any) (eq bool, err error)` that returns an error instead of
   panicking when the dynamic type is uncomparable. Use `reflect.Type.Comparable`.
5. Take a struct with a `[]string` field and try to use it as a key in `map[any]bool` and in
   `map[YourStruct]bool`. Note which failure is at compile time and which at runtime, and argue
   which API you would rather ship.

---

[← 5.1 Implicit Satisfaction](01-implicit-satisfaction.md) · [Course Index](../README.md) · [5.3 Type Assertions and Type Switches →](03-assertions-type-switches.md)
