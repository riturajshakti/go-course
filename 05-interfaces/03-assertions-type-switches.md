[← 5.2 Interface Values, nil, and the Typed-nil Trap](02-interface-values-nil.md) · [Course Index](../README.md) · [5.4 The Empty Interface and any →](04-empty-interface.md)

# 5.3 — Type Assertions and Type Switches

## What You'll Learn

- `i.(T)` panics on failure; `v, ok := i.(T)` never does — and when each form is right
- Asserting to *another interface* is how you detect optional capabilities, which the stdlib does constantly
- Every form of the type switch: multiple types per case, `case nil`, `default`, and the no-binding form
- An assertion unwraps one level only — which is why `errors.As` exists

## Coming From Another Language

| Other | Go |
|---|---|
| TS `x as Foo` — erased, no runtime check | checked at runtime; wrong type panics |
| TS `if ("read" in x)` for capability checks | `if r, ok := x.(Reader); ok` — checked against a method set |
| Python `isinstance(x, Foo)` | `_, ok := x.(Foo)` — but by methods, not by inheritance |
| Rust `match` on an enum, exhaustive | `switch i.(type)` — **not** exhaustive, no compiler help |
| Rust `downcast_ref::<T>()` on `dyn Any` | `i.(T)`, built into the language |

An assertion is not a conversion. `float64(x)` changes a value's representation at compile
time. `i.(float64)` asks the runtime what is actually in the box and refuses to answer wrong.

## The two forms

```go
package main

import "fmt"

func main() {
  var i any = "hello"

  s := i.(string)
  fmt.Println(s, len(s))

  n, ok := i.(int)
  fmt.Println(n, ok)

  s2, ok := i.(string)
  fmt.Println(s2, ok)

  _, ok = i.(fmt.Stringer)
  fmt.Println(ok)
}
```

Output:

```
hello 5
0 false
hello true
false
```

On failure, the comma-ok form gives you the **zero value** of `T` and `false` — not a nil, not
a panic. That `0` in the second line is a real `int`.

Single-return form, wrong type:

```go
package main

import "fmt"

func main() {
  var i any = "hello"
  n := i.(int)
  fmt.Println(n)
}
```

This program panics on purpose. Output:

```
panic: interface conversion: interface {} is string, not int

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:7 +0x34
exit status 2
```

The message names both types, which makes it one of Go's more useful panics.

Idiomatic: use the one-return form only when a failure is a programmer bug you want to crash
on (and that is rarer than it sounds). Everywhere else, comma-ok.

## Asserting on a nil interface

A nil interface has no dynamic type, so every assertion fails.

```go
package main

import "fmt"

func main() {
  var i any
  s, ok := i.(string)
  fmt.Printf("%q %t\n", s, ok)
}
```

Output:

```
"" false
```

```go
package main

import "fmt"

func main() {
  var i any
  fmt.Println(i.(string))
}
```

This program panics on purpose. Output:

```
panic: interface conversion: interface {} is nil, not string

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:7 +0x30
exit status 2
```

Note `is nil` rather than a type name. Combined with 5.2: an interface holding a *nil pointer*
is not a nil interface, and asserting it to that pointer type **succeeds**.

## Impossible assertions are a compile error

If the target type could not possibly satisfy the interface's method set, the compiler stops
you rather than emitting code that always fails.

```go
package main

import "fmt"

type Reader interface{ Read() string }

type Point struct{ X int }

func (p Point) Read() string { return "pt" }

func main() {
  var r Reader = Point{}
  _ = r.(int)
  fmt.Println("ok")
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:13:7: impossible type assertion: r.(int)
	int does not implement Reader (missing method Read)
```

This check is only possible when the source is a non-empty interface. Asserting from `any`
gets no such help, since every type satisfies `any`.

## Asserting to another interface: capability detection

This is the pattern that matters most in practice. You hold a narrow interface; you ask at
runtime whether the concrete value behind it supports something better.

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "strings"
)

func describe(dst io.Writer, src io.Reader) {
  _, rf := dst.(io.ReaderFrom)
  _, wt := src.(io.WriterTo)
  _, sw := dst.(io.StringWriter)
  fmt.Printf("%T: ReaderFrom=%t StringWriter=%t   %T: WriterTo=%t\n", dst, rf, sw, src, wt)
}

type plainWriter struct{}

func (plainWriter) Write(b []byte) (int, error) { return len(b), nil }

func main() {
  var buf bytes.Buffer
  describe(&buf, strings.NewReader("hi"))
  describe(plainWriter{}, strings.NewReader("hi"))
  describe(&buf, io.LimitReader(strings.NewReader("hi"), 1))
}
```

Output:

```
*bytes.Buffer: ReaderFrom=true StringWriter=true   *strings.Reader: WriterTo=true
main.plainWriter: ReaderFrom=false StringWriter=false   *strings.Reader: WriterTo=true
*bytes.Buffer: ReaderFrom=true StringWriter=true   *io.LimitedReader: WriterTo=false
```

`io.Copy` does exactly this. Its contract is `Copy(dst Writer, src Reader)`, but inside it
tries `src.(io.WriterTo)` and then `dst.(io.ReaderFrom)` — and if either hits, it delegates the
whole transfer and never allocates the 32 KB intermediate buffer. On Linux that path can reach
`sendfile(2)`. Your `plainWriter` still works; it just takes the slow road.

Here is the same shape in your own code:

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "strings"
)

func copyFrom(dst io.Writer, src io.Reader) (int64, string) {
  if rf, ok := dst.(io.ReaderFrom); ok {
    n, _ := rf.ReadFrom(src)
    return n, "used dst.ReadFrom"
  }
  if wt, ok := src.(io.WriterTo); ok {
    n, _ := wt.WriteTo(dst)
    return n, "used src.WriteTo"
  }
  n, _ := io.Copy(struct{ io.Writer }{dst}, struct{ io.Reader }{src})
  return n, "fell back to a plain loop"
}

type plainWriter struct{ n int }

func (p *plainWriter) Write(b []byte) (int, error) { p.n += len(b); return len(b), nil }

type plainReader struct{ done bool }

func (p *plainReader) Read(b []byte) (int, error) {
  if p.done {
    return 0, io.EOF
  }
  p.done = true
  return copy(b, "abc"), nil
}

func main() {
  var buf bytes.Buffer
  fmt.Println(copyFrom(&buf, strings.NewReader("hello")))
  fmt.Println(buf.String())

  fmt.Println(copyFrom(&plainWriter{}, strings.NewReader("hello")))
  fmt.Println(copyFrom(&plainWriter{}, &plainReader{}))
}
```

Output:

```
5 used dst.ReadFrom
hello
5 used src.WriteTo
3 fell back to a plain loop
```

The `struct{ io.Writer }{dst}` wrappers in the fallback hide the extra methods, forcing
`io.Copy` down its own generic path so the demonstration does not recurse into the optimization
we just declined.

Idiomatic: an *optional* interface is the Go answer to an optional method. Keep the required
interface tiny, and probe for extras.

## The type switch

Full form: a binding, several cases, multiple types per case, `nil`, and `default`.

```go
package main

import (
  "fmt"
  "time"
)

func kind(i any) string {
  switch v := i.(type) {
  case nil:
    return "nil interface"
  case int, int64:
    return fmt.Sprintf("integer-ish %v (%T)", v, v)
  case string:
    return fmt.Sprintf("string of len %d", len(v))
  case []byte:
    return fmt.Sprintf("bytes %q", v)
  case error:
    return "error: " + v.Error()
  case fmt.Stringer:
    return "stringer: " + v.String()
  case func() int:
    return fmt.Sprintf("func returning %d", v())
  default:
    return fmt.Sprintf("something else: %T", v)
  }
}

func main() {
  fmt.Println(kind(nil))
  fmt.Println(kind(7))
  fmt.Println(kind(int64(7)))
  fmt.Println(kind("hello"))
  fmt.Println(kind([]byte("hi")))
  fmt.Println(kind(fmt.Errorf("boom")))
  fmt.Println(kind(time.Second))
  fmt.Println(kind(func() int { return 42 }))
  fmt.Println(kind(3.14))
}
```

Output:

```
nil interface
integer-ish 7 (int)
integer-ish 7 (int64)
string of len 5
bytes "hi"
error: boom
stringer: 1s
func returning 42
something else: float64
```

Four things to notice:

- **In a single-type case, `v` has that concrete type.** `len(v)` works in `case string`;
  `v.Error()` works in `case error`.
- **In a multi-type case, `v` keeps the interface type** — here `any`. That is why the
  `int, int64` case uses `%v`/`%T` rather than arithmetic. This surprises everyone once.
- **`case nil` matches the nil interface** and nothing else. Without it, a nil `i` lands in
  `default`.
- **Interface cases are allowed**, and match by method set. `time.Duration` has a `String`
  method, so it hits `case fmt.Stringer`.

Cases are tested **top to bottom**, and interface cases can overlap. Order is a decision:

```go
package main

import "fmt"

type MyErr struct{}

func (MyErr) Error() string  { return "myerr" }
func (MyErr) String() string { return "MYERR" }

func order1(i any) string {
  switch i.(type) {
  case error:
    return "error"
  case fmt.Stringer:
    return "stringer"
  }
  return "neither"
}

func order2(i any) string {
  switch i.(type) {
  case fmt.Stringer:
    return "stringer"
  case error:
    return "error"
  }
  return "neither"
}

func main() {
  fmt.Println(order1(MyErr{}), order2(MyErr{}))
  fmt.Println(order1(42), order2(42))
}
```

Output:

```
error stringer
neither neither
```

`MyErr` satisfies both. Idiomatic: list concrete types before interface types, and more
specific interfaces before broader ones.

## The remaining forms

Drop the binding when you only care which branch ran. An empty case body is legal, and there is
no fallthrough in a type switch.

```go
package main

import "fmt"

func main() {
  var i any = 3.5
  switch i.(type) {
  case int:
    fmt.Println("int")
  case float64:
    fmt.Println("float64")
  }

  switch v := i.(type) {
  case float64:
    fmt.Println(v * 2)
  }

  var j any = "s"
  switch j.(type) {
  case string:
  default:
    fmt.Println("not reached")
  }
  fmt.Println("done")
}
```

Output:

```
float64
7
done
```

With no `default` and no match, the switch does nothing — unlike Rust's `match`, this is not
exhaustive and the compiler will not warn you about a type you forgot. Duplicate concrete cases
*are* caught:

```go
package main

import "fmt"

func main() {
  var i any = 1
  switch i.(type) {
  case int:
    fmt.Println("a")
  case int:
    fmt.Println("b")
  }
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:8: duplicate case int in type switch
	./main.go:8:8: previous case
```

## Assertion vs conversion

Two different operations that look similar.

| | Conversion `T(x)` | Assertion `i.(T)` |
|---|---|---|
| Operand | any value | must be an interface |
| When checked | compile time | run time |
| What it does | reinterprets/converts the value | reveals the type already inside |
| On mismatch | compile error | panic, or `ok == false` |

```go
package main

import "fmt"

type Celsius float64
type Fahrenheit float64

func main() {
  c := Celsius(100)
  f := Fahrenheit(c)
  fmt.Println(c, f, float64(c))

  var i any = c
  v := i.(Celsius)
  fmt.Println(v)

  _, ok := i.(Fahrenheit)
  fmt.Println(ok)
}
```

Output:

```
100 100 100
100
false
```

`Fahrenheit(c)` converts freely — identical underlying type. But an interface holding a
`Celsius` is *not* a `Fahrenheit` and not a `float64`; assertion demands the exact dynamic type:

```go
package main

import "fmt"

func main() {
  var i any = Celsius(10)
  _ = i.(float64)
  fmt.Println("unreachable")
}

type Celsius float64
```

This program panics on purpose. Output:

```
panic: interface conversion: interface {} is main.Celsius, not float64

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:7 +0x34
exit status 2
```

To get the `float64` out, assert to `Celsius` then convert: `float64(i.(Celsius))`.

## Dispatching on error types

The type switch's most common real use. Note the `default` branch and what happens on the last
input.

```go
package main

import (
  "errors"
  "fmt"
)

type NotFoundError struct{ Key string }

func (e *NotFoundError) Error() string { return "not found: " + e.Key }

type RateLimitError struct{ RetryAfter int }

func (e *RateLimitError) Error() string {
  return fmt.Sprintf("rate limited, retry in %ds", e.RetryAfter)
}

var ErrClosed = errors.New("closed")

func status(err error) (int, string) {
  switch e := err.(type) {
  case nil:
    return 200, "ok"
  case *NotFoundError:
    return 404, e.Key
  case *RateLimitError:
    return 429, fmt.Sprint(e.RetryAfter)
  default:
    if errors.Is(err, ErrClosed) {
      return 503, "closed"
    }
    return 500, err.Error()
  }
}

func main() {
  fmt.Println(status(nil))
  fmt.Println(status(&NotFoundError{Key: "u/1"}))
  fmt.Println(status(&RateLimitError{RetryAfter: 30}))
  fmt.Println(status(ErrClosed))
  fmt.Println(status(fmt.Errorf("wrapped: %w", ErrClosed)))
  fmt.Println(status(errors.New("other")))
  fmt.Println(status(fmt.Errorf("wrapped: %w", &NotFoundError{Key: "u/2"})))
}
```

Output:

```
200 ok
404 u/1
429 30
503 closed
503 closed
500 other
500 wrapped: not found: u/2
```

The last line is the bug. A wrapped `*NotFoundError` returns **500**, because an assertion
inspects exactly one level: the dynamic type is `*fmt.wrapError`, not `*NotFoundError`.

## Why errors.As exists

```go
package main

import (
  "errors"
  "fmt"
)

type NotFoundError struct{ Key string }

func (e *NotFoundError) Error() string { return "not found: " + e.Key }

func main() {
  err := fmt.Errorf("layer: %w", &NotFoundError{Key: "u/2"})

  _, ok := err.(*NotFoundError)
  fmt.Println("assertion:", ok)

  var nf *NotFoundError
  fmt.Println("errors.As:", errors.As(err, &nf))
  fmt.Println(nf.Key)
}
```

Output:

```
assertion: false
errors.As: true
u/2
```

`errors.As` is a type assertion in a loop over `Unwrap()`. Idiomatic: **never type-switch on
errors in modern Go** — use `errors.As` for typed errors and `errors.Is` for sentinels. Module
08 covers the whole mechanism; this lesson just shows you the sharp edge it files down.

## Cost, and a design smell

An assertion is cheap: a pointer comparison against the type descriptor for a concrete target,
or a small cached method-table lookup for an interface target. It is not free, but it is
nowhere near reflection, and it is never the reason your program is slow.

The real cost is design. A type switch over *your own* concrete types usually means a method
belonged on those types instead — add it to the interface and let dispatch do the work. A type
switch is justified when you do not own the types (`fmt`, `encoding/json`), when you are
probing for optional capabilities, or when the branches genuinely do unrelated things.

## Quick Reference

| Form | Behaviour |
|---|---|
| `v := i.(T)` | `v` is `T`; panics if the dynamic type is not exactly `T` |
| `v, ok := i.(T)` | `ok` false and `v` the zero value on failure; never panics |
| `_, ok := i.(I)` | capability probe against interface `I`'s method set |
| `switch i.(type) { ... }` | branch on type, no binding |
| `switch v := i.(type) { ... }` | `v` typed per case |
| `case T:` | `v` has concrete type `T` |
| `case T1, T2:` | `v` keeps `i`'s **interface** type |
| `case I:` (an interface) | matches any dynamic type satisfying `I` |
| `case nil:` | matches only a nil interface |
| `default:` | `v` keeps `i`'s interface type |

| Error / panic | Cause |
|---|---|
| `interface conversion: interface {} is string, not int` | one-return assertion, wrong type |
| `interface conversion: interface {} is nil, not string` | one-return assertion on a nil interface |
| `impossible type assertion` (compile) | target cannot satisfy the source interface |
| `duplicate case int in type switch` (compile) | the same concrete type listed twice |

## Common Mistakes

**Using `i.(T)` where a failure is possible.** Use `v, ok := i.(T)` and handle the `false`
branch; the one-return form is for invariants only.

**Doing arithmetic on the binding in a multi-type case.** `case int, int64` leaves `v` as the
interface type; split into separate cases if you need the value.

**Type-switching on errors.** A wrapped error has a different dynamic type; use `errors.As`
and `errors.Is`.

**Forgetting `case nil` and letting a nil interface reach `default`.** `default` will then call
a method on it and panic.

**Expecting the compiler to flag a missing case.** A type switch is not exhaustive; add a
`default` that logs or returns an error for the unexpected.

## Exercises

1. Write `func sum(vals []any) (float64, []error)` that adds up anything numeric using a type
   switch covering all eleven numeric types plus `string` (parsed with `strconv`), and returns
   one error per unusable element.
2. Define `type Flusher interface{ Flush() error }` and a `write(w io.Writer, b []byte) error`
   that flushes only if the writer supports it. Test it with `os.Stdout`, a
   `*bufio.Writer`, and a `*bytes.Buffer`, printing which path each took.
3. Reproduce the three panics from this lesson — wrong type, nil interface, and `main.Celsius`
   vs `float64` — then make each one non-panicking with the smallest possible edit.
4. Take the `status` function and rewrite it with `errors.As` so that wrapped errors map to the
   right status code. Then rewrite it a third way, as a method on each error type, and argue
   which version you would want in a service.
5. Write a type switch with both `case error` and `case fmt.Stringer`, feed it a type satisfying
   both, and then reverse the case order. Add a third case for a concrete type that also
   satisfies both and show that case order stops mattering.

---

[← 5.2 Interface Values, nil, and the Typed-nil Trap](02-interface-values-nil.md) · [Course Index](../README.md) · [5.4 The Empty Interface and any →](04-empty-interface.md)
