[← 7.10 sort: Before and Alongside Generics](../07-slices-maps/10-sort.md) · [Course Index](../README.md) · [8.2 Wrapping with fmt.Errorf and %w →](02-wrapping.md)

# 8.1 — The error Interface and errors.New

## What You'll Learn

- `error` is an ordinary interface with exactly one method, `Error() string`
- Errors are values you return and check, not control flow you throw and catch
- `errors.New` returns a pointer, so two identical-text errors are not equal
- The zero value of `error` is `nil` — and a nil pointer stored in an `error` is not

## Coming From Another Language

| Other | Go |
|---|---|
| JS `throw` / `try-catch` | return an `error` as the last result; check it with `if err != nil` |
| Python `raise` / `except ValueError` | sentinel values and typed errors compared with `errors.Is` / `errors.As` |
| Rust `Result<T, E>` | the `(T, error)` pair — same idea, no enum, no exhaustiveness check |
| Rust `?` | nothing. You write the three-line check by hand |
| JS `err instanceof HttpError` | `errors.As(err, &target)` (8.3) |
| Python `finally` | `defer` (3.5) |

Go has `panic`, but it is not the error mechanism — it is for programmer error and
unrecoverable states (3.6). Normal failure is a value: a function that can fail returns
one more result, of type `error`, and the caller inspects it. There is no `?` operator and
no plan to add one. What the explicit check buys you is that every failure path is visible
in the source: you can see, by reading, which calls can fail and what happens when they
do. What it costs you is three lines per call. That trade is the whole design.

## The interface

This is the entire declaration, from the `builtin` package:

```go
type error interface {
  Error() string
}
```

One method. Anything with an `Error() string` method is an `error`, with no declaration of
intent — implicit satisfaction as in 5.1.

## The (T, error) convention

The error is the last result. On failure the other results are the zero value and must not
be used.

```go
package main

import (
  "errors"
  "fmt"
  "strconv"
)

func parsePort(s string) (int, error) {
  n, err := strconv.Atoi(s)
  if err != nil {
    return 0, err
  }
  if n < 1 || n > 65535 {
    return 0, errors.New("port out of range")
  }
  return n, nil
}

func main() {
  for _, s := range []string{"8080", "abc", "99999"} {
    n, err := parsePort(s)
    if err != nil {
      fmt.Printf("parsePort(%q) -> err: %v\n", s, err)
      continue
    }
    fmt.Printf("parsePort(%q) -> %d\n", s, n)
  }
}
```

Output:

```
parsePort("8080") -> 8080
parsePort("abc") -> err: strconv.Atoi: parsing "abc": invalid syntax
parsePort("99999") -> err: port out of range
```

Idiomatic: handle the error first and return early, so the happy path stays at the
left margin. Do not write `else`.

## errors.New and the zero value

`errors.New(text)` gives you the simplest possible error. The zero value of the `error`
interface is `nil`, and `nil` means success.

```go
package main

import (
  "errors"
  "fmt"
)

func main() {
  e1 := errors.New("not found")
  e2 := errors.New("not found")
  fmt.Println(e1 == e2)
  fmt.Println(e1.Error() == e2.Error())
  fmt.Printf("%T %T\n", e1, e2)
  fmt.Println(e1 == e1)

  var zero error
  fmt.Println(zero == nil, zero)
  fmt.Printf("%v %T\n", zero, zero)
}
```

Output:

```
false
true
*errors.errorString *errors.errorString
true
true <nil>
<nil> <nil>
```

`e1 == e2` is **false**. `errors.New` allocates an `errorString` and returns a *pointer* to
it, so comparison is pointer identity, not text. This is deliberate and it is the whole
reason sentinel errors work: a package-level `var ErrNotFound = errors.New("not found")` is
a unique value nobody else can accidentally reproduce by spelling the same string (8.4).

Note also that printing a nil error with `%v` gives `<nil>`, and `%T` gives `<nil>` too —
there is no type in there at all.

## fmt.Errorf for formatting

When the message needs values interpolated, use `fmt.Errorf`. Without a `%w` verb it does
not wrap anything (8.2) — it is just `errors.New(fmt.Sprintf(...))`.

```go
package main

import (
  "errors"
  "fmt"
)

type Config struct {
  Host string
  Port int
}

var ErrEmptyHost = errors.New("empty host")

func New(host string, port int) (*Config, error) {
  if host == "" {
    return nil, ErrEmptyHost
  }
  if port < 1 || port > 65535 {
    return nil, fmt.Errorf("port %d out of range", port)
  }
  return &Config{Host: host, Port: port}, nil
}

func main() {
  c, err := New("localhost", 8080)
  fmt.Println(c, err)

  c, err = New("", 8080)
  fmt.Println(c, err)
  fmt.Println(err == ErrEmptyHost)

  c, err = New("localhost", 0)
  fmt.Println(c, err)
  fmt.Println(err == ErrEmptyHost)
}
```

Output:

```
&{localhost 8080} <nil>
<nil> empty host
true
<nil> port 0 out of range
false
```

Returning `nil` for the error on success is explicit — there is no implicit "no error".

## Implementing error on your own type

Any type works, not just structs. The receiver choice matters.

```go
package main

import (
  "errors"
  "fmt"
)

type ValueErr struct {
  Field string
}

func (e ValueErr) Error() string {
  return "bad value in " + e.Field
}

type PtrErr struct {
  Field string
}

func (e *PtrErr) Error() string {
  return "bad pointer value in " + e.Field
}

func main() {
  var err error = ValueErr{Field: "name"}
  fmt.Println(err)

  err = &ValueErr{Field: "name"}
  fmt.Println(err)

  err = &PtrErr{Field: "age"}
  fmt.Println(err)

  fmt.Println(errors.Is(err, err))

  a := ValueErr{Field: "x"}
  b := ValueErr{Field: "x"}
  var ea, eb error = a, b
  fmt.Println(ea == eb)

  pa := &PtrErr{Field: "x"}
  pb := &PtrErr{Field: "x"}
  var pea, peb error = pa, pb
  fmt.Println(pea == peb)
}
```

Output:

```
bad value in name
bad value in name
bad pointer value in age
true
true
false
```

Two consequences. A *value* receiver means both `ValueErr{...}` and `&ValueErr{...}`
satisfy `error`, and two value-receiver errors with equal fields compare **equal** — which
can be a feature or a landmine. A *pointer* receiver means only `*PtrErr` satisfies
`error`, and comparison is identity.

## The method-set consequence

With a pointer receiver, the value type does not implement the interface. This is the
method-set rule from 4.4 and 5.1, and it bites most often on errors.

```go
package main

import "fmt"

type ValErr struct{ Field string }

func (e ValErr) Error() string { return "bad " + e.Field }

type PtrOnly struct{ Field string }

func (e *PtrOnly) Error() string { return "bad " + e.Field }

func main() {
  var a error = ValErr{"x"}
  var b error = &ValErr{"y"}
  fmt.Println(a, b)

  var c error = &PtrOnly{"z"}
  fmt.Println(c)

  var d error = PtrOnly{"w"}
  fmt.Println(d)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:21:17: cannot use PtrOnly{…} (value of struct type PtrOnly) as error value in variable declaration: PtrOnly does not implement error (method Error has pointer receiver)
```

Idiomatic: use a pointer receiver for error types and always return `&MyError{...}`. The
stdlib does this (`*fs.PathError`, `*strconv.NumError`, `*url.Error`).

## The typed-nil trap

5.2 covered this for interfaces generally. Here is where it actually costs people a
production outage: a nil `*MyError` stored in an `error` is not a nil `error`.

```go
package main

import "fmt"

type MyError struct {
  Code int
}

func (e *MyError) Error() string {
  return fmt.Sprintf("my error code %d", e.Code)
}

func broken() error {
  var e *MyError
  return e
}

func fixed() error {
  var e *MyError
  if e != nil {
    return e
  }
  return nil
}

func main() {
  err := broken()
  fmt.Println(err == nil)
  fmt.Printf("%v %T\n", err, err)

  err2 := fixed()
  fmt.Println(err2 == nil)
  fmt.Printf("%v %T\n", err2, err2)

  if err := broken(); err != nil {
    fmt.Println("broken() looked like a failure")
  }
}
```

Output:

```
false
<nil> *main.MyError
true
<nil> <nil>
broken() looked like a failure
```

The interface value holds a type (`*main.MyError`) and a nil pointer, so it is not equal to
`nil`. It even *prints* as `<nil>` while failing the check — the worst possible
combination. Idiomatic: never declare a concrete error type as your function's result
type. Declare the result as `error` and return a literal `nil`.

## Error strings: the convention

Error strings get concatenated by wrapping, so they are fragments, not sentences.

| Rule | Bad | Good |
|---|---|---|
| Lowercase first letter (unless a proper noun or acronym) | `"Failed to open file"` | `"open file"` |
| No trailing punctuation | `"port out of range."` | `"port out of range"` |
| No "failed to" / "error:" at the leaf | `"error: failed to parse port"` | `"parse port"` |
| State what was attempted, let the caller add context | `"could not read user from database"` | `"read user"` |

`staticcheck`'s **ST1005** flags capitalized error strings and trailing punctuation.
(The staticcheck build available here predates this toolchain's export-data format and
refused to analyze the module, so its exact message is not reproduced; the check ID is
the stable reference.) `go vet` does not check this, and `golangci-lint` enables
`staticcheck` including ST1005 by default under its `stylecheck`/`staticcheck` linters.

## Ignoring an error

`_` discards it. That is defensible when the operation cannot meaningfully fail or when a
failure changes nothing you can act on.

```go
package main

import (
  "fmt"
  "os"
  "strconv"
)

func mustAtoi(s string) int {
  n, _ := strconv.Atoi(s)
  return n
}

func main() {
  fmt.Println(mustAtoi("42"), mustAtoi("oops"))

  var buf []byte
  buf = strconv.AppendInt(buf, 7, 10)
  _, _ = os.Stdout.Write(buf)
  fmt.Println()

  defer func() {
    if r := recover(); r != nil {
      fmt.Println("recovered:", r)
    }
  }()
  var m map[string]int
  m["x"] = 1
}
```

Output:

```
42 0
7
recovered: assignment to entry in nil map
```

`mustAtoi("oops")` silently returns 0 — fine for a trusted constant, a bug for user input.
`fmt.Println` to stdout failing is not something a CLI can respond to. What is *not*
defensible is ignoring the error from `f.Close()` on a file you wrote to, or from
`tx.Commit()`. `errcheck` (bundled in `golangci-lint`) reports unchecked error returns and
is worth enabling with an allowlist for `fmt.Print*`.

Note the nil-map panic at the end: that is programmer error, not an `error` value, and it
is why `panic` exists alongside `error`.

## panic vs error, and log.Fatal

| Situation | Use |
|---|---|
| The caller can plausibly recover or report | return an `error` |
| Invariant violated by a bug in the calling code (nil required arg, index out of range) | `panic` |
| Package cannot initialize at all (bad embedded regexp, bad template) | `panic` in `init` / a `MustXxx` helper |
| A CLI's top level cannot continue | `log.Fatal` — in `main` only |
| Anywhere inside a library | never `log.Fatal`, never `os.Exit` |

`log.Fatal` calls `os.Exit(1)`, which skips every `defer` in the program. Confine it to
`main`. 3.6 covers `panic`/`recover` mechanics.

## Errors in a goroutine

A goroutine's error has nowhere to return to. Writing it to a shared variable is a data
race; dropping it loses the failure.

```go
package main

import (
  "errors"
  "fmt"
)

func main() {
  errc := make(chan error, 1)
  go func() {
    errc <- errors.New("worker failed")
  }()
  fmt.Println(<-errc)
}
```

Output:

```
worker failed
```

Channels, `sync.WaitGroup`, and `errgroup` are modules 10 and 11 — 11.8 is the one you
will actually reach for. The rule to carry forward: a goroutine that can fail must have a
declared channel for its error before you start it.

## Quick Reference

| Thing | Signature / form | Note |
|---|---|---|
| `error` | `interface { Error() string }` | one method, implicitly satisfied |
| `errors.New` | `func(text string) error` | returns `*errorString`; identity comparison |
| `fmt.Errorf` | `func(format string, a ...any) error` | no `%w` means no wrapping |
| success | `return value, nil` | explicit nil |
| failure | `return zeroValue, err` | other results must be ignored |
| check | `if err != nil { return ..., err }` | early return, no `else` |
| ignore | `_, _ = f()` | only when failure is not actionable |
| `errors.Is(err, target)` | `func(err, target error) bool` | 8.3; use instead of `==` |

| Receiver | Satisfies `error` | Comparison |
|---|---|---|
| `func (e T) Error() string` | `T` and `*T` | `==` compares fields |
| `func (e *T) Error() string` | `*T` only | `==` compares pointers |

## Common Mistakes

**Declaring a function's result as `*MyError` instead of `error`.** Returning a nil
`*MyError` makes `err != nil` true; always declare the result as `error`.

**Comparing errors with `==` against a sentinel.** Any wrapping breaks it; use
`errors.Is` (8.3).

**Expecting two `errors.New` calls with the same text to be equal.** They are distinct
pointers; store one in a package-level `var` and reuse it.

**Capitalizing error strings or ending them with a period.** They get embedded in longer
messages by wrapping; keep them lowercase fragments (ST1005).

**Using the other return values after a non-nil error.** They are the zero value by
convention and carry no information; return immediately.

**Calling `log.Fatal` inside a library function.** It exits the process and skips every
`defer`; return the error and let `main` decide.

## Exercises

1. Write `func Divide(a, b float64) (float64, error)` returning an error for `b == 0`.
   Then write a second version that returns `(float64, *DivError)` and demonstrate, with a
   caller that stores the result in an `error` variable, that the success case now fails
   the `err != nil` check.
2. Define `type Celsius float64` with an `Error() string` method on a value receiver.
   Show that a `Celsius` value satisfies `error` and that `errors.New("x") == errors.New("x")`
   is false but two equal `Celsius` errors compare equal. Explain the risk.
3. Write a `Stack` type with `Push`/`Pop` where `Pop` on an empty stack returns
   `(zero, ErrEmpty)` with a package-level sentinel. Prove with a test that the sentinel
   survives a round trip through a function that returns it unchanged, and that a
   `fmt.Errorf("pop: %v", ErrEmpty)` version does not.
4. Take the `broken()`/`fixed()` program and add a third function that returns a non-nil
   `*MyError` through an `error` result. Print `%T`, `%v`, and `err == nil` for all three
   and write down the rule you would give a reviewer in one sentence.
5. Instrument a program that opens three files, two of which do not exist, and collect the
   errors without stopping at the first. Compare your approach with `errors.Join` after
   reading 8.5.

---

[← 7.10 sort: Before and Alongside Generics](../07-slices-maps/10-sort.md) · [Course Index](../README.md) · [8.2 Wrapping with fmt.Errorf and %w →](02-wrapping.md)
