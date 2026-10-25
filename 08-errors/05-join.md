[← 8.4 Sentinel Errors and Custom Error Types](04-sentinels-custom.md) · [Course Index](../README.md) · [8.6 Error Handling Strategy for Libraries vs Services →](06-strategy.md)

# 8.5 — errors.Join and Multiple Errors

## What You'll Learn

- `errors.Join` filters nils, returns `nil` when everything is nil, and joins with newlines
- The joined error implements `Unwrap() []error`, so `Is`/`As` traverse a tree
- Accumulating into a `[]error` and joining once is the validate-all-fields pattern
- `errors.Join` in `defer` collects failures from closing N resources

## Coming From Another Language

| Other | Go |
|---|---|
| Python `ExceptionGroup` / `raise ... from` groups (3.11+) | `errors.Join` — the same tree shape |
| Python `except*` | `errors.Is`/`errors.As`, which search the whole tree |
| JS `AggregateError` (from `Promise.any`) | `errors.Join` |
| JS `Promise.allSettled` then filter rejections | collect into `[]error`, `errors.Join(errs...)` |
| Rust `Vec<Error>` returned by a validator | identical, but `errors.Join` gives it one `error` face |

Before Go 1.20 everyone wrote or imported a multierror type
(`github.com/hashicorp/go-multierror`, `go.uber.org/multierr`). `errors.Join` plus
`Unwrap() []error` made that unnecessary.

## The basics, including the nil behaviour

```go
package main

import (
  "errors"
  "fmt"
)

func main() {
  e1 := errors.New("first")
  e2 := errors.New("second")
  e3 := errors.New("third")

  j := errors.Join(e1, e2, e3)
  fmt.Printf("%q\n", j.Error())
  fmt.Println("---")
  fmt.Println(j)
  fmt.Println("---")
  fmt.Printf("%T\n", j)

  fmt.Println(errors.Join() == nil)
  fmt.Println(errors.Join(nil) == nil)
  fmt.Println(errors.Join(nil, nil, nil) == nil)

  one := errors.Join(e1)
  fmt.Printf("%T %q\n", one, one.Error())

  mixed := errors.Join(nil, e1, nil, e2)
  fmt.Printf("%q\n", mixed.Error())

  u := mixed.(interface{ Unwrap() []error })
  fmt.Println(len(u.Unwrap()))
  fmt.Println(errors.Unwrap(mixed))
}
```

Output:

```
"first\nsecond\nthird"
---
first
second
third
---
*errors.joinError
true
true
true
*errors.joinError "first"
"first\nsecond"
2
<nil>
```

The details that matter:

- **The message is newline-separated**, with no trailing newline. `"first\nsecond\nthird"`.
  Printed, that is three lines — so a joined error in a single-line log format will break
  the line. Formatting it for logs is a real problem, handled below.
- **All-nil returns a plain `nil`**, so `errors.Join(errs...)` on an empty or all-nil slice
  gives you a clean success. This is what makes the accumulate-and-join pattern read well.
- **nils are filtered**, so you can pass results directly without guarding each one.
- **A single non-nil error still gets wrapped** in a `*errors.joinError` — it is not
  returned as-is. Identity is preserved via the tree, not via `==`.
- **`errors.Unwrap` returns `nil`.** `joinError` has `Unwrap() []error`, not
  `Unwrap() error`. Use `errors.Is`/`errors.As`, or assert to
  `interface{ Unwrap() []error }`.

## Is and As traverse a tree

A joined error has several children, each of which may have its own chain. `errors.Is` and
`errors.As` do a depth-first search of the whole thing.

```go
package main

import (
  "errors"
  "fmt"
  "os"
)

var ErrQuota = errors.New("quota exceeded")

type FieldError struct {
  Field string
}

func (e *FieldError) Error() string { return "bad field " + e.Field }

func main() {
  _, pathErr := os.Open("/no/such/file")
  j := errors.Join(
    errors.New("plain"),
    fmt.Errorf("layer: %w", ErrQuota),
    &FieldError{Field: "email"},
    pathErr,
  )
  fmt.Println(j)
  fmt.Println("---")
  fmt.Println("quota:", errors.Is(j, ErrQuota))
  fmt.Println("notexist:", errors.Is(j, os.ErrNotExist))

  var fe *FieldError
  fmt.Println("as FieldError:", errors.As(j, &fe), fe.Field)

  outer := fmt.Errorf("request failed: %w", j)
  fmt.Println(outer)
  fmt.Println("---")
  fmt.Println("quota through wrap:", errors.Is(outer, ErrQuota))

  nested := errors.Join(errors.New("a"), errors.Join(errors.New("b"), errors.New("c")))
  fmt.Printf("%q\n", nested.Error())
  fmt.Println(len(nested.(interface{ Unwrap() []error }).Unwrap()))
}
```

Output:

```
plain
layer: quota exceeded
bad field email
open /no/such/file: no such file or directory
---
quota: true
notexist: true
as FieldError: true email
request failed: plain
layer: quota exceeded
bad field email
open /no/such/file: no such file or directory
---
quota through wrap: true
"a\nb\nc"
2
```

`errors.Is` found `ErrQuota` inside the second child's chain and `os.ErrNotExist` four
levels down inside the fourth child's. `errors.As` pulled the `*FieldError` out of the
third. Wrapping the whole join with `%w` adds a chain node above the tree and nothing
breaks.

Note the last two lines: a nested join's *message* flattens to `"a\nb\nc"` — you cannot
see the structure in the text — but the *tree* still has 2 direct children. If you iterate
`Unwrap()` expecting leaves, you will get a join back.

Also note `errors.As` with a joined error finds the **first** match in depth-first order.
If three children are `*FieldError`, you get one. Iterating is below.

## Accumulating in a loop: validate every field

The pattern: a `[]error`, appended to on each failure, joined once at the end. The all-nil
behaviour means the success path needs no special case.

```go
package main

import (
  "errors"
  "fmt"
  "strings"
)

type FieldError struct {
  Field  string
  Reason string
}

func (e *FieldError) Error() string {
  return fmt.Sprintf("%s: %s", e.Field, e.Reason)
}

type Signup struct {
  Name     string
  Email    string
  Password string
  Age      int
}

func (s Signup) Validate() error {
  var errs []error
  if s.Name == "" {
    errs = append(errs, &FieldError{"name", "required"})
  }
  if !strings.Contains(s.Email, "@") {
    errs = append(errs, &FieldError{"email", "must contain @"})
  }
  if len(s.Password) < 8 {
    errs = append(errs, &FieldError{"password", "must be at least 8 characters"})
  }
  if s.Age < 13 {
    errs = append(errs, &FieldError{"age", "must be at least 13"})
  }
  return errors.Join(errs...)
}

func fields(err error) []string {
  var out []string
  var fe *FieldError
  if u, ok := err.(interface{ Unwrap() []error }); ok {
    for _, e := range u.Unwrap() {
      if errors.As(e, &fe) {
        out = append(out, fe.Field)
      }
    }
  }
  return out
}

func main() {
  good := Signup{Name: "Ada", Email: "ada@example.com", Password: "correcthorse", Age: 36}
  fmt.Println("good:", good.Validate())

  bad := Signup{Email: "nope", Password: "short", Age: 9}
  err := bad.Validate()
  fmt.Println("bad:")
  fmt.Println(err)
  fmt.Println("---")
  fmt.Println("count:", len(err.(interface{ Unwrap() []error }).Unwrap()))
  fmt.Println("fields:", fields(err))

  var fe *FieldError
  fmt.Println("first via As:", errors.As(err, &fe), fe.Field)

  partial := Signup{Name: "Bob", Email: "bob@example.com", Password: "x", Age: 40}
  fmt.Println("partial:", partial.Validate())
}
```

Output:

```
good: <nil>
bad:
name: required
email: must contain @
password: must be at least 8 characters
age: must be at least 13
---
count: 4
fields: [name email password age]
first via As: true name
partial: password: must be at least 8 characters
```

`Validate` has no `if len(errs) == 0 { return nil }` — `errors.Join` handles it. `fields`
is the iterating counterpart to `errors.As`: assert to `interface{ Unwrap() []error }`,
loop the children, `errors.As` each one. That is how you build a
`map[string]string` of field errors for a JSON 422 response (16.8).

A form validator is the textbook case for joining: the user wants all four problems at
once, not one per round trip.

## errors.Join in defer: closing N resources

Closing several resources and collecting every failure, while not losing the body's error.

```go
package main

import (
  "errors"
  "fmt"
  "io"
)

type res struct {
  name   string
  failOn bool
}

func (r *res) Close() error {
  if r.failOn {
    return fmt.Errorf("close %s: %w", r.name, io.ErrClosedPipe)
  }
  return nil
}

func work(fail ...bool) (err error) {
  var resources []*res
  names := []string{"db", "cache", "queue"}
  for i, n := range names {
    resources = append(resources, &res{name: n, failOn: i < len(fail) && fail[i]})
  }
  defer func() {
    var cerrs []error
    for _, r := range resources {
      if cerr := r.Close(); cerr != nil {
        cerrs = append(cerrs, cerr)
      }
    }
    err = errors.Join(err, errors.Join(cerrs...))
  }()
  return nil
}

func workWithBody(bodyErr error, fail ...bool) (err error) {
  var resources []*res
  names := []string{"db", "cache"}
  for i, n := range names {
    resources = append(resources, &res{name: n, failOn: i < len(fail) && fail[i]})
  }
  defer func() {
    var cerrs []error
    for _, r := range resources {
      if cerr := r.Close(); cerr != nil {
        cerrs = append(cerrs, cerr)
      }
    }
    err = errors.Join(err, errors.Join(cerrs...))
  }()
  return bodyErr
}

func main() {
  fmt.Println("all ok:", work())
  fmt.Println("---")
  fmt.Println(work(true, false, true))
  fmt.Println("---")
  err := workWithBody(errors.New("body failed"), true, true)
  fmt.Println(err)
  fmt.Println("---")
  fmt.Println("closedpipe:", errors.Is(err, io.ErrClosedPipe))
  fmt.Println("count:", len(err.(interface{ Unwrap() []error }).Unwrap()))
}
```

Output:

```
all ok: <nil>
---
close db: io: read/write on closed pipe
close queue: io: read/write on closed pipe
---
body failed
close db: io: read/write on closed pipe
close cache: io: read/write on closed pipe
---
closedpipe: true
count: 2
```

`err = errors.Join(err, errors.Join(cerrs...))` is the whole idiom. When the body
succeeded and nothing failed to close, both arguments are nil and the result is nil. When
the body failed, its error comes first. When only closes failed, you get just those.

Note `count: 2`, not 3: the outer join has two children, the body error and the inner
join. The message flattens; the tree does not. If you need flat children, build one slice
and join once.

Note also that the close ordering in the output follows `resources`, not reverse order — a
loop over a slice in a single deferred closure is not LIFO like stacked `defer`s (3.5).

## Flattening and printing nicely

Newlines are hostile to single-line log formats. Flatten the tree and format it yourself.

```go
package main

import (
  "errors"
  "fmt"
  "strings"
)

func flatten(err error) []error {
  if err == nil {
    return nil
  }
  if u, ok := err.(interface{ Unwrap() []error }); ok {
    var out []error
    for _, e := range u.Unwrap() {
      out = append(out, flatten(e)...)
    }
    return out
  }
  return []error{err}
}

func bullets(err error) string {
  var b strings.Builder
  for _, e := range flatten(err) {
    fmt.Fprintf(&b, "  - %v\n", e)
  }
  return b.String()
}

func main() {
  nested := errors.Join(
    errors.New("a"),
    errors.Join(errors.New("b"), errors.Join(errors.New("c"), errors.New("d"))),
  )
  fmt.Printf("%q\n", nested.Error())
  fmt.Println("direct children:", len(nested.(interface{ Unwrap() []error }).Unwrap()))
  fmt.Println("flattened:", len(flatten(nested)))
  fmt.Print(bullets(nested))

  fmt.Println("---")
  e1 := errors.New("unauthorized")
  e2 := errors.New("quota exceeded")
  j := errors.Join(e1, e2)
  w := fmt.Errorf("reject: %w and %w", e1, e2)
  fmt.Printf("join:  %q\n", j.Error())
  fmt.Printf("multi: %q\n", w.Error())
  fmt.Println(errors.Is(j, e1), errors.Is(w, e1))
}
```

Output:

```
"a\nb\nc\nd"
direct children: 2
flattened: 4
  - a
  - b
  - c
  - d
---
join:  "unauthorized\nquota exceeded"
multi: "reject: unauthorized and quota exceeded"
true true
```

`flatten` recurses through `Unwrap() []error` nodes and returns only leaves — four from a
tree with two direct children. For a `slog` call, pass `flatten(err)` as a slice attribute
rather than the raw error, or replace newlines with `; `.

Note `flatten` deliberately does not follow `Unwrap() error`, so a `fmt.Errorf("ctx: %w",
leaf)` child stays one leaf with its full context intact.

## errors.Join vs multiple %w

| | `errors.Join(e1, e2)` | `fmt.Errorf("reject: %w and %w", e1, e2)` |
|---|---|---|
| Message | `"unauthorized\nquota exceeded"` | `"reject: unauthorized and quota exceeded"` |
| Type | `*errors.joinError` | `*fmt.wrapErrors` |
| `Unwrap() []error` | yes | yes |
| `errors.Is` / `As` | traverse both children | traverse both children |
| nil filtering | yes; all-nil returns `nil` | no; `%!w(<nil>)` in the message |
| Variable count | `errs...` | fixed at the format string |

Use `errors.Join` when you have a *collection* of independent failures. Use multiple `%w`
when you are writing one sentence that happens to mention two specific errors.

## When a joined error is the wrong answer

If the first failure makes the rest meaningless — a connection that failed, so every
subsequent query will too — fail fast and return one error; a list of fifty identical
timeouts is noise, not information.

## Quick Reference

| Expression | Result |
|---|---|
| `errors.Join(e1, e2, e3)` | `*errors.joinError`, message `"e1\ne2\ne3"` |
| `errors.Join()` | `nil` |
| `errors.Join(nil, nil)` | `nil` |
| `errors.Join(nil, e1)` | join with 1 child, message `"e1"` |
| `errors.Join(errs...)` on an empty slice | `nil` |
| `errors.Unwrap(join)` | `nil` — wrong tool |
| `join.(interface{ Unwrap() []error }).Unwrap()` | the direct children |
| `errors.Is(join, target)` | depth-first over the whole tree |
| `errors.As(join, &t)` | the first depth-first match |
| `fmt.Errorf("ctx: %w", join)` | a chain node above the tree; `Is`/`As` still work |

| Pattern | Code |
|---|---|
| validate all | `var errs []error; ...; return errors.Join(errs...)` |
| merge body + cleanup | `err = errors.Join(err, cerr)` in a deferred closure |
| count children | `if u, ok := err.(interface{ Unwrap() []error }); ok { len(u.Unwrap()) }` |
| extract all of a type | loop `u.Unwrap()`, `errors.As` each child |
| your own multi-error type | implement `Unwrap() []error` (not both forms) |

## Common Mistakes

**Calling `errors.Unwrap` on a joined error.** It returns `nil` because `joinError` has
`Unwrap() []error`; use `errors.Is`/`errors.As` or assert to the multi-unwrap interface.

**Adding `if len(errs) == 0 { return nil }` before `errors.Join(errs...)`.** It is already
the behaviour; the guard is dead code.

**Logging a joined error into a single-line format.** The embedded newlines break the
line; flatten it or replace the newlines first.

**Assuming `Unwrap()` returns leaves.** A nested join appears as one child whose message
has already flattened; recurse if you need leaves.

**Joining a hundred identical downstream failures.** One root cause produced them all;
fail fast on the first instead.

**Implementing both `Unwrap() error` and `Unwrap() []error` on one type.** Only the
single-error form is a valid `Unwrap`, and having both is ambiguous; pick one.

## Exercises

1. Extend `Signup.Validate` with a `Username` field checked by three separate rules.
   Confirm the joined error reports all three, and write `func fieldMap(err error)
   map[string][]string` grouping reasons by field.
2. Write `func CloseAll(cs ...io.Closer) error` that closes every closer even if earlier
   ones fail, and returns a joined error. Test it with a mix of successes and failures and
   assert with `errors.Is` that each specific failure is findable.
3. Implement your own `type MultiError struct { Errs []error }` with `Error() string` and
   `Unwrap() []error`. Show `errors.Is` and `errors.As` working through it, then show that
   adding an `Unwrap() error` method instead silently breaks the traversal.
4. Build a worker that processes ten items, three of which fail, and returns a joined error
   that names each failed item's index. Then rewrite it to fail fast and argue in two
   sentences which version you would ship.
5. Write `func Count(err error) int` returning the number of leaf errors in an arbitrary
   tree mixing `errors.Join`, `fmt.Errorf("%w")`, and multi-`%w`. Verify it against a
   handmade tree where you know the answer.

---

[← 8.4 Sentinel Errors and Custom Error Types](04-sentinels-custom.md) · [Course Index](../README.md) · [8.6 Error Handling Strategy for Libraries vs Services →](06-strategy.md)
