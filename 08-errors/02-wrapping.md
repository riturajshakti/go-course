[← 8.1 The error Interface and errors.New](01-error-interface.md) · [Course Index](../README.md) · [8.3 errors.Is, errors.As, and Unwrap →](03-is-as-unwrap.md)

# 8.2 — Wrapping with fmt.Errorf and %w

## What You'll Learn

- `%w` keeps the original error reachable; `%v` and `%s` flatten it to text
- A fully wrapped message reads `operation: layer: cause` from outside in
- `Unwrap() error` on your own type makes it participate in the chain
- `fmt.Errorf` with multiple `%w` verbs wraps several errors at once (Go 1.20+)

## Coming From Another Language

| Other | Go |
|---|---|
| Python `raise X from e` (`__cause__`) | `fmt.Errorf("...: %w", e)` |
| JS `new Error(msg, { cause: e })` | the same idea, built into `fmt.Errorf` |
| Rust `anyhow::Context::context("...")` | `fmt.Errorf("...: %w", err)` |
| Python traceback showing the call chain | no stack; you build the chain by wrapping at each layer |
| JS `e.stack` | nothing in stdlib — the wrapped message *is* your trace |

Go errors carry no stack trace. What a reader of your logs gets is the message you
composed by wrapping at each layer, so the context you add is the only context there is.
That is the practical reason to wrap: not to preserve a stack, but to turn
`no such file or directory` into something that names the operation that failed.

## %w vs %v: the whole point

Both produce the same string. Only `%w` preserves the identity.

```go
package main

import (
  "errors"
  "fmt"
)

var ErrNotFound = errors.New("not found")

func wrapped() error {
  return fmt.Errorf("load user: %w", ErrNotFound)
}

func formatted() error {
  return fmt.Errorf("load user: %v", ErrNotFound)
}

func main() {
  w := wrapped()
  f := formatted()
  fmt.Println(w)
  fmt.Println(f)
  fmt.Println(w.Error() == f.Error())
  fmt.Println("Is(w, ErrNotFound):", errors.Is(w, ErrNotFound))
  fmt.Println("Is(f, ErrNotFound):", errors.Is(f, ErrNotFound))
  fmt.Println("Unwrap(w):", errors.Unwrap(w))
  fmt.Println("Unwrap(f):", errors.Unwrap(f))
}
```

Output:

```
load user: not found
load user: not found
true
Is(w, ErrNotFound): true
Is(f, ErrNotFound): false
Unwrap(w): not found
Unwrap(f): <nil>
```

Identical text, opposite behaviour. `%v` throws the error away and keeps a copy of its
words; `%w` stores the error itself in a `*fmt.wrapError` that `errors.Is` can walk. This
single difference is the most common cause of "why isn't my `errors.Is` matching" — and
nothing warns you, because the printed message looks right.

Idiomatic: `%w` by default. Use `%v` only when you deliberately want to cut the chain
(see "When not to wrap").

## Three layers and what the message reads like

```go
package main

import (
  "errors"
  "fmt"
  "os"
)

func readConfig(path string) error {
  _, err := os.Open(path)
  if err != nil {
    return fmt.Errorf("read config %q: %w", path, err)
  }
  return nil
}

func loadSettings(path string) error {
  if err := readConfig(path); err != nil {
    return fmt.Errorf("load settings: %w", err)
  }
  return nil
}

func startup() error {
  if err := loadSettings("/etc/app/missing.yaml"); err != nil {
    return fmt.Errorf("startup: %w", err)
  }
  return nil
}

func main() {
  err := startup()
  fmt.Println(err)
  fmt.Println()
  for e := err; e != nil; e = errors.Unwrap(e) {
    fmt.Printf("%T: %v\n", e, e)
  }
  fmt.Println()
  fmt.Println(errors.Is(err, os.ErrNotExist))
}
```

Output:

```
startup: load settings: read config "/etc/app/missing.yaml": open /etc/app/missing.yaml: no such file or directory

*fmt.wrapError: startup: load settings: read config "/etc/app/missing.yaml": open /etc/app/missing.yaml: no such file or directory
*fmt.wrapError: load settings: read config "/etc/app/missing.yaml": open /etc/app/missing.yaml: no such file or directory
*fmt.wrapError: read config "/etc/app/missing.yaml": open /etc/app/missing.yaml: no such file or directory
*fs.PathError: open /etc/app/missing.yaml: no such file or directory
syscall.Errno: no such file or directory

true
```

The shape is `outermost: ...: innermost`, read left to right from the highest layer down
to the syscall. Each wrap contributes one `name: ` prefix, so the convention from 8.1 —
lowercase, no trailing punctuation, no "failed to" — is what makes the concatenation
readable. Five levels deep, and `errors.Is(err, os.ErrNotExist)` still matches the
`syscall.Errno` at the bottom.

Note the format verb placement: `%w` goes last, after the context, with `: ` before it.

## Multiple %w in one Errorf

Go 1.20 allows more than one `%w`. The result is a `*fmt.wrapErrors` with
`Unwrap() []error` rather than `Unwrap() error`.

```go
package main

import (
  "errors"
  "fmt"
)

var (
  ErrAuth  = errors.New("unauthorized")
  ErrQuota = errors.New("quota exceeded")
)

func main() {
  err := fmt.Errorf("reject request: %w and %w", ErrAuth, ErrQuota)
  fmt.Println(err)
  fmt.Println(errors.Is(err, ErrAuth), errors.Is(err, ErrQuota))
  fmt.Printf("%T\n", err)
  fmt.Println(errors.Unwrap(err))

  u, ok := err.(interface{ Unwrap() []error })
  fmt.Println(ok)
  if ok {
    for _, e := range u.Unwrap() {
      fmt.Println("  -", e)
    }
  }
}
```

Output:

```
reject request: unauthorized and quota exceeded
true true
*fmt.wrapErrors
<nil>
true
  - unauthorized
  - quota exceeded
```

`errors.Unwrap` returns `nil` for a multi-wrap error: it only understands
`Unwrap() error`. `errors.Is` and `errors.As` understand both and walk a tree. 8.5 covers
`errors.Join`, which is the right tool when you have a *slice* of errors rather than a
sentence that mentions two of them.

## %w with a nil error

Nothing special happens. You get a formatting error embedded in the text and a wrapper
around nothing.

```go
package main

import (
  "errors"
  "fmt"
)

func main() {
  e1 := fmt.Errorf("layer: %w", error(nil))
  fmt.Printf("%q\n", e1.Error())
  fmt.Println(errors.Unwrap(e1))
  fmt.Printf("%T\n", e1)

  var nilErr error
  e2 := fmt.Errorf("layer: %w", nilErr)
  fmt.Printf("%q %v\n", e2.Error(), errors.Unwrap(e2))

  e3 := fmt.Errorf("layer: %w", errors.New("real"))
  fmt.Printf("%q\n", e3.Error())
}
```

Output:

```
"layer: %!w(<nil>)"
<nil>
*fmt.wrapError
"layer: %!w(<nil>)" <nil>
"layer: real"
```

`%!w(<nil>)` in a log line means someone wrapped unconditionally without checking
`err != nil` first. The returned value is non-nil, so a caller's `err != nil` succeeds
and you have invented a failure. Always guard the wrap.

## Unwrap() error on your own type

Implement `Unwrap() error` and your type joins the chain. Omit it and the chain stops
there — which is sometimes what you want.

```go
package main

import (
  "errors"
  "fmt"
)

var ErrDenied = errors.New("permission denied")

type QueryError struct {
  Query string
  Err   error
}

func (e *QueryError) Error() string {
  return fmt.Sprintf("query %q: %v", e.Query, e.Err)
}

func (e *QueryError) Unwrap() error { return e.Err }

type NoUnwrap struct {
  Err error
}

func (e *NoUnwrap) Error() string { return "opaque: " + e.Err.Error() }

func main() {
  var err error = &QueryError{Query: "SELECT 1", Err: ErrDenied}
  fmt.Println(err)
  fmt.Println(errors.Is(err, ErrDenied))
  fmt.Println(errors.Unwrap(err))

  wrapped := fmt.Errorf("handle request: %w", err)
  fmt.Println(wrapped)
  fmt.Println(errors.Is(wrapped, ErrDenied))

  var opaque error = &NoUnwrap{Err: ErrDenied}
  fmt.Println(opaque)
  fmt.Println(errors.Is(opaque, ErrDenied))
  fmt.Println(errors.Unwrap(opaque))
}
```

Output:

```
query "SELECT 1": permission denied
true
permission denied
handle request: query "SELECT 1": permission denied
true
opaque: permission denied
false
<nil>
```

`*NoUnwrap` holds the cause and prints it, but `errors.Is` returns false — the text is
there, the identity is not. Deciding whether your type has `Unwrap` is deciding whether
callers may match against what you wrapped.

For a type that holds several causes, implement `Unwrap() []error` instead (8.5). Do not
implement both.

## Wrapping in defer with a named result

3.5 showed `defer` with a named result. This is the one idiom where it earns its keep:
one wrap covering every return path in the function.

```go
package main

import (
  "errors"
  "fmt"
  "os"
)

func readAll(path string) (err error) {
  defer func() {
    if err != nil {
      err = fmt.Errorf("readAll %s: %w", path, err)
    }
  }()

  f, err := os.Open(path)
  if err != nil {
    return err
  }
  defer f.Close()
  return nil
}

func copyFile(src string) (err error) {
  f, err := os.Open(src)
  if err != nil {
    return fmt.Errorf("open src: %w", err)
  }
  defer func() {
    if cerr := f.Close(); cerr != nil && err == nil {
      err = fmt.Errorf("close src: %w", cerr)
    }
  }()
  return nil
}

func main() {
  fmt.Println(readAll("/etc/hosts"))
  err := readAll("/no/such/path")
  fmt.Println(err)
  fmt.Println(errors.Is(err, os.ErrNotExist))
  fmt.Println(copyFile("/etc/hosts"))
  fmt.Println(copyFile("/no/such/path"))
}
```

Output:

```
<nil>
readAll /no/such/path: open /no/such/path: no such file or directory
true
<nil>
open src: open /no/such/path: no such file or directory
```

Two patterns in one program. `readAll` wraps whatever error it returns, from any of its
returns, with no repetition. `copyFile` promotes a `Close` failure into the result — the
standard way to not silently drop a close error on a file you wrote to. The `err == nil`
guard means a real error from the body wins over a close error.

The `if err != nil` inside the deferred closure is required: without it you wrap `nil`
and get `%!w(<nil>)`.

## Message vs identity

Two errors can print identically and behave completely differently. This is why you never
test or branch on error text.

```go
package main

import (
  "errors"
  "fmt"
  "strings"
)

var ErrNotFound = errors.New("not found")

func main() {
  a := fmt.Errorf("get user 42: %w", ErrNotFound)
  b := errors.New("get user 42: not found")

  fmt.Println(a.Error() == b.Error())
  fmt.Println(errors.Is(a, ErrNotFound), errors.Is(b, ErrNotFound))
  fmt.Println(strings.Contains(b.Error(), "not found"))
}
```

Output:

```
true
true false
true
```

`strings.Contains` "works" on both — which is exactly the trap. It is correct here by
accident, and it breaks the day someone rewords a message, translates it, or adds a path
that happens to contain the substring.

## When not to wrap

Three cases.

**Don't leak implementation details across an API boundary.** If your `store` package
wraps a `pgx` error with `%w`, every caller can now match against pgx sentinels, and
swapping to another driver becomes a breaking change. Translate at the boundary instead.

```go
package main

import (
  "errors"
  "fmt"
)

var ErrNotFound = errors.New("not found")

var errPgxNoRows = errors.New("no rows in result set")

func leaky(id int) error {
  return fmt.Errorf("store.GetUser(%d): pgx: scan row: %w", id, errPgxNoRows)
}

func translated(id int) error {
  err := errPgxNoRows
  if errors.Is(err, errPgxNoRows) {
    return fmt.Errorf("get user %d: %w", id, ErrNotFound)
  }
  return err
}

func main() {
  fmt.Println(leaky(42))
  fmt.Println(errors.Is(leaky(42), ErrNotFound))

  err := translated(42)
  fmt.Println(err)
  fmt.Println(errors.Is(err, ErrNotFound))
  fmt.Println(errors.Is(err, errPgxNoRows))
}
```

Output:

```
store.GetUser(42): pgx: scan row: no rows in result set
false
get user 42: not found
true
false
```

`translated` converts the driver's sentinel into its own and does not expose the driver's.
Callers match `store.ErrNotFound`; the driver is free to change.

**Do wrap a sentinel you want callers to match.** This is the point people get backwards.
Wrapping does *not* hide a sentinel — `errors.Is` walks through the wrap. So wrap freely:
you add context and keep matchability. The only reason not to wrap a sentinel is if you do
not want it to be part of your API at all.

**Don't wrap the same error four times with synonyms.** Each layer adds words; redundant
layers add noise.

```go
package main

import (
  "errors"
  "fmt"
  "os"
)

func noisy(path string) error {
  _, err := os.Open(path)
  if err != nil {
    err = fmt.Errorf("failed to open file: %w", err)
    err = fmt.Errorf("error reading config: %w", err)
    err = fmt.Errorf("could not load settings: %w", err)
    return fmt.Errorf("startup failed: %w", err)
  }
  return nil
}

func clean(path string) error {
  _, err := os.Open(path)
  if err != nil {
    return fmt.Errorf("load settings: %w", err)
  }
  return nil
}

func main() {
  fmt.Println(noisy("/no/such/x.yaml"))
  fmt.Println(clean("/no/such/x.yaml"))
  fmt.Println(errors.Is(noisy("/no/such/x.yaml"), os.ErrNotExist))
}
```

Output:

```
startup failed: could not load settings: error reading config: failed to open file: open /no/such/x.yaml: no such file or directory
load settings: open /no/such/x.yaml: no such file or directory
true
```

The first says "something went wrong" four times before saying anything. Rule of thumb:
wrap once per *package* boundary with information the inner layer did not have (an ID, a
path, the operation name) — not once per function.

`fmt.Errorf` allocates: the formatted string plus the wrapper struct. On an error path
that is irrelevant; inside a hot loop that returns errors as normal flow, it is not.

Historical note: before Go 1.13 everyone used `github.com/pkg/errors` for
`Wrap`/`Cause`/stack traces; `%w` plus `errors.Is`/`As` replaced it and the stdlib won.

## Quick Reference

| Form | Chain preserved | `errors.Unwrap` returns |
|---|---|---|
| `fmt.Errorf("ctx: %w", err)` | yes | `err` |
| `fmt.Errorf("ctx: %v", err)` | no | `nil` |
| `fmt.Errorf("ctx: %s", err)` | no | `nil` |
| `fmt.Errorf("a %w b %w", e1, e2)` | yes (tree) | `nil` — use `Unwrap() []error` |
| `fmt.Errorf("ctx: %w", nil)` | no | `nil`, message gets `%!w(<nil>)` |
| `errors.New("ctx: " + err.Error())` | no | `nil` |

| Method on your type | Meaning |
|---|---|
| `Error() string` | required; the message |
| `Unwrap() error` | one cause; joins the chain |
| `Unwrap() []error` | several causes; joins the tree (8.5) |
| neither | opaque; `errors.Is` stops at your type |

| Idiom | Code |
|---|---|
| wrap at a layer | `return fmt.Errorf("op %s: %w", id, err)` |
| wrap every return | `func f() (err error) { defer func(){ if err != nil { err = fmt.Errorf("f: %w", err) } }() ... }` |
| promote a Close error | `defer func(){ if cerr := f.Close(); cerr != nil && err == nil { err = cerr } }()` |
| cut the chain on purpose | `return fmt.Errorf("op: %v", internalErr)` |

## Common Mistakes

**Writing `%v` where you meant `%w`.** The message is identical and `errors.Is` silently
returns false; grep your codebase for `Errorf` with an error argument and no `%w`.

**Wrapping without checking `err != nil` first.** You get a non-nil error reading
`%!w(<nil>)`; guard every wrap.

**Forgetting `Unwrap() error` on a custom error type that holds a cause.** `errors.Is`
cannot see past it; add the method or document the type as opaque.

**Wrapping a driver or third-party error with `%w` in an exported function.** You have
made that dependency part of your API; translate to your own sentinel at the boundary.

**Wrapping at every function instead of every boundary.** The message becomes four
synonyms for "failed"; add context the inner layer did not have, or add nothing.

**Branching on `strings.Contains(err.Error(), ...)`.** Messages are not an API; use
`errors.Is` or `errors.As` (8.3).

## Exercises

1. Take the three-layer `startup` program and change exactly one `%w` to `%v`. Predict
   which `errors.Is` calls still succeed, then verify. Repeat with the innermost and the
   outermost wrap and explain the difference.
2. Write `func process(path string) (err error)` that opens a file, writes to it, and
   closes it, using the deferred-close idiom so that a write error wins over a close error
   but a close error is never lost. Prove both orderings with a test.
3. Implement `type RetryError struct { Attempts int; Err error }` with `Error`, `Unwrap`,
   and a constructor. Show that `errors.Is(retryErr, os.ErrNotExist)` works through it,
   then delete `Unwrap` and show the test failing.
4. Build a two-package module where `inner` returns a private sentinel and `outer`
   translates it to an exported one. Write a test asserting that `errors.Is(outer.Do(),
   inner.ErrX)` is false (the detail does not leak) while `errors.Is(outer.Do(),
   outer.ErrY)` is true.
5. Measure it: benchmark `fmt.Errorf("x: %w", err)` against returning `err` unchanged with
   `go test -bench . -benchmem`. Report ns/op and allocs/op, then decide where in your own
   code that cost would actually matter.

---

[← 8.1 The error Interface and errors.New](01-error-interface.md) · [Course Index](../README.md) · [8.3 errors.Is, errors.As, and Unwrap →](03-is-as-unwrap.md)
