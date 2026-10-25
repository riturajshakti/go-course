[← 8.3 errors.Is, errors.As, and Unwrap](03-is-as-unwrap.md) · [Course Index](../README.md) · [8.5 errors.Join and Multiple Errors →](05-join.md)

# 8.4 — Sentinel Errors and Custom Error Types

## What You'll Learn

- Sentinels are package-level `var Err... = errors.New(...)` values, matched with `errors.Is`
- Custom error types carry machine-readable fields, extracted with `errors.As`
- Behaviour interfaces (`NotFound() bool`) let callers match without importing your types
- Every exported error is part of your API, forever

## Coming From Another Language

| Other | Go |
|---|---|
| Python exception class hierarchy (`class NotFound(AppError)`) | no hierarchy; a behaviour interface or a custom `Is` |
| Python `except (A, B)` | `errors.Is(err, A) \|\| errors.Is(err, B)` |
| JS `class HttpError extends Error { constructor(code) }` | `type HTTPError struct { Code int }` with `Error() string` |
| JS `e.code = 'ENOENT'` string codes | a sentinel `var`, compared by identity not text |
| Rust `enum MyError { NotFound, Invalid(String) }` | a sentinel for the unit variants, a struct type for the ones with data |
| Rust's exhaustive `match` on the enum | nothing enforces exhaustiveness; add a `default` branch |

Go has no error subclassing. The three mechanisms below replace it, and choosing between
them is the actual design decision in this lesson.

## Sentinel errors

A package-level `var`, prefixed `Err`, created once with `errors.New`. Identity, not text,
is what callers match.

```go
package main

import (
  "context"
  "database/sql"
  "errors"
  "fmt"
  "io"
  "io/fs"
)

var (
  ErrNotFound = errors.New("not found")
  ErrConflict = errors.New("conflict")
  ErrClosed   = errors.New("closed")
)

func main() {
  fmt.Printf("%v | %v | %v\n", ErrNotFound, ErrConflict, ErrClosed)
  fmt.Printf("%T\n", ErrNotFound)

  wrapped := fmt.Errorf("get user 7: %w", ErrNotFound)
  fmt.Println(wrapped == ErrNotFound)
  fmt.Println(errors.Is(wrapped, ErrNotFound))

  for _, e := range []error{io.EOF, sql.ErrNoRows, fs.ErrNotExist, fs.ErrPermission, context.Canceled, context.DeadlineExceeded} {
    fmt.Printf("%-28T %v\n", e, e)
  }
}
```

Output:

```
not found | conflict | closed
*errors.errorString
false
true
*errors.errorString          EOF
*errors.errorString          sql: no rows in result set
*errors.errorString          file does not exist
*errors.errorString          permission denied
*errors.errorString          context canceled
context.deadlineExceededError context deadline exceeded
```

Use `var`, not `const` — `errors.New` is a function call, so a constant is impossible.
Compare with `errors.Is`, never `==`: the third line of output is the wrapped case failing
`==` while `errors.Is` succeeds.

Almost every stdlib sentinel is a plain `*errors.errorString`.
`context.DeadlineExceeded` is the exception — it is a custom type so it can also satisfy
`net.Error` with `Timeout() bool`.

**The compatibility cost:** an exported sentinel is a permanent API commitment. Callers
will write `errors.Is(err, yourpkg.ErrNotFound)`, and removing it, or changing which
operations return it, breaks them at compile time or (worse) silently. Export the ones
callers genuinely need to branch on; keep the rest unexported.

## io.EOF as an expected signal

The canonical sentinel-as-control-flow pattern: `io.EOF` is not a failure, so you check
for it and convert it to a normal return.

```go
package main

import (
  "bufio"
  "errors"
  "fmt"
  "io"
  "strings"
)

func readLines(r io.Reader) ([]string, error) {
  br := bufio.NewReader(r)
  var out []string
  for {
    line, err := br.ReadString('\n')
    if line != "" {
      out = append(out, strings.TrimSuffix(line, "\n"))
    }
    if errors.Is(err, io.EOF) {
      return out, nil
    }
    if err != nil {
      return out, fmt.Errorf("read line %d: %w", len(out)+1, err)
    }
  }
}

type failReader struct{ n int }

func (f *failReader) Read(p []byte) (int, error) {
  if f.n > 0 {
    f.n--
    p[0] = 'x'
    return 1, nil
  }
  return 0, errors.New("disk on fire")
}

func main() {
  lines, err := readLines(strings.NewReader("a\nb\nc"))
  fmt.Printf("%q %v\n", lines, err)

  lines, err = readLines(strings.NewReader(""))
  fmt.Printf("%q %v\n", lines, err)

  lines, err = readLines(&failReader{n: 3})
  fmt.Printf("%q %v\n", lines, err)
  fmt.Println(errors.Is(err, io.EOF))
}
```

Output:

```
["a" "b" "c"] <nil>
[] <nil>
["xxx"] read line 2: disk on fire
false
```

Two things worth noting. `ReadString` can return data *and* `io.EOF` in the same call —
the final `"c"` has no newline — so you must consume `line` before checking the error.
And `readLines` does not pass `io.EOF` up to its caller: EOF is this function's
termination condition, not its failure. A real error propagates, wrapped.

Idiomatic: convert `io.EOF` to `nil` at the layer where the stream legitimately ends. Never
return a bare `io.EOF` from a function whose contract is not "I am a Reader".

## Custom error types with fields

When the caller needs *data* out of the error — which field failed, which ID was missing,
what status to return — a sentinel cannot carry it. Use a struct.

```go
package main

import (
  "errors"
  "fmt"
)

type ValidationError struct {
  Field  string
  Reason string
}

func (e *ValidationError) Error() string {
  return fmt.Sprintf("field %s: %s", e.Field, e.Reason)
}

type NotFoundError struct {
  Resource string
  ID       string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("%s %s not found", e.Resource, e.ID)
}

var ErrNotFound = errors.New("not found")

func (e *NotFoundError) Is(target error) bool { return target == ErrNotFound }

type HTTPError struct {
  Code int
  Err  error
}

func (e *HTTPError) Error() string { return fmt.Sprintf("http %d: %v", e.Code, e.Err) }
func (e *HTTPError) Unwrap() error { return e.Err }

func main() {
  var e1 error = &ValidationError{Field: "email", Reason: "missing @"}
  fmt.Println(e1)

  var e2 error = &NotFoundError{Resource: "user", ID: "42"}
  fmt.Println(e2)
  fmt.Println(errors.Is(e2, ErrNotFound))

  var e3 error = &HTTPError{Code: 404, Err: e2}
  fmt.Println(e3)
  fmt.Println(errors.Is(e3, ErrNotFound))

  wrapped := fmt.Errorf("handler: %w", e3)
  var ve *ValidationError
  var nf *NotFoundError
  var he *HTTPError
  fmt.Println(errors.As(wrapped, &ve), errors.As(wrapped, &nf), errors.As(wrapped, &he))
  fmt.Printf("%d %s/%s\n", he.Code, nf.Resource, nf.ID)

  w2 := fmt.Errorf("handler: %w", e1)
  if errors.As(w2, &ve) {
    fmt.Printf("field=%q reason=%q\n", ve.Field, ve.Reason)
  }
}
```

Output:

```
field email: missing @
user 42 not found
true
http 404: user 42 not found
true
false true true
404 user/42
field="email" reason="missing @"
```

Three techniques stacked. `*NotFoundError` gets a custom `Is` so callers can match the
coarse sentinel *or* extract the exact resource. `*HTTPError` gets `Unwrap`, so
`errors.Is(e3, ErrNotFound)` reaches through it. And `errors.As` on the fully wrapped error
pulls out both the `*HTTPError` and the `*NotFoundError` from different depths of one chain.

Constructor convention: export `func NewValidationError(field, reason string) error` if
construction needs validation; otherwise a composite literal is fine.

## Pointer vs value receiver, and why pointer

8.1 showed the method-set rule. Here is the behavioural reason to prefer a pointer.

```go
package main

import (
  "errors"
  "fmt"
)

type valErr struct{ Field string }

func (e valErr) Error() string { return "bad " + e.Field }

type ptrErr struct{ Field string }

func (e *ptrErr) Error() string { return "bad " + e.Field }

func main() {
  var a error = valErr{"x"}
  var b error = valErr{"x"}
  fmt.Println(a == b, errors.Is(a, b))

  var c error = &ptrErr{"x"}
  var d error = &ptrErr{"x"}
  fmt.Println(c == d, errors.Is(c, d))

  var ve valErr
  fmt.Println(errors.As(a, &ve), ve.Field)

  e := valErr{"y"}
  var f error = e
  e.Field = "z"
  fmt.Println(f)

  p := &ptrErr{"y"}
  var g error = p
  p.Field = "z"
  fmt.Println(g)
}
```

Output:

```
true true
false false
true x
bad y
bad z
```

A value receiver gives you equality by fields, which sounds nice until two unrelated
failures at different call sites compare equal and a dedupe or a `map[error]int` silently
merges them. It also *copies* the struct into the interface — the `bad y` line shows the
error frozen at the moment it was stored, which is sometimes what you want and sometimes a
confusing stale snapshot. A pointer receiver gives identity semantics and matches every
stdlib error type.

Idiomatic: pointer receiver, construct with `&MyError{...}`, declare `As` targets as
`*MyError`.

## Behaviour interfaces

Dave Cheney's "assert for behaviour, not type": define an interface describing what the
error *means* and let `errors.As` find anything that satisfies it. Callers never import
your error types.

```go
package main

import (
  "errors"
  "fmt"
)

type NotFound interface {
  NotFound() bool
}

type Temporary interface {
  Temporary() bool
}

func IsNotFound(err error) bool {
  var nf NotFound
  return errors.As(err, &nf) && nf.NotFound()
}

func IsTemporary(err error) bool {
  var t Temporary
  return errors.As(err, &t) && t.Temporary()
}

type userMissing struct{ ID string }

func (e *userMissing) Error() string  { return "user " + e.ID + " does not exist" }
func (e *userMissing) NotFound() bool { return true }

type dbDown struct{}

func (dbDown) Error() string   { return "database unreachable" }
func (dbDown) Temporary() bool { return true }
func (dbDown) NotFound() bool  { return false }

func main() {
  errs := []error{
    fmt.Errorf("service: %w", &userMissing{ID: "42"}),
    fmt.Errorf("service: %w", dbDown{}),
    errors.New("something else"),
  }
  for _, err := range errs {
    fmt.Printf("%-45v notfound=%-5v temporary=%v\n", err, IsNotFound(err), IsTemporary(err))
  }
}
```

Output:

```
service: user 42 does not exist               notfound=true  temporary=false
service: database unreachable                 notfound=false temporary=true
something else                                notfound=false temporary=false
```

`userMissing` and `dbDown` are both unexported. The only exported things are the
interfaces and the two `Is...` helpers, so the concrete types can change freely. Note
`dbDown` returns `NotFound() bool` as `false` — satisfying the interface and answering no
is different from not satisfying it, and the `&& nf.NotFound()` in the helper handles
both.

The cost: three declarations per concept instead of one `var`, and nothing stops another
package from accidentally satisfying your interface with an unrelated `NotFound()` method.

## Choosing between the three

| Strategy | Caller writes | Carries data | API surface | Use when |
|---|---|---|---|---|
| Sentinel `var ErrX` | `errors.Is(err, pkg.ErrX)` | no | the `var` is permanent | a small fixed set of outcomes the caller branches on |
| Custom type `*XError` | `var e *pkg.XError; errors.As(err, &e)` | yes, typed fields | the type and every exported field | the caller needs values (field name, status, ID, retry-after) |
| Opaque + behaviour interface | `pkg.IsNotFound(err)` or `errors.As(err, &iface)` | via methods | only the interface/helper | you want freedom to change the concrete types, or many packages produce the condition |

In practice a real package uses sentinels for the two or three coarse outcomes and one
custom type for the rich one. That combination is the capstone below.

Whichever you pick, it is part of your contract. Document it in the function's doc
comment:

```go
// Get returns the user with the given id.
//
// Get returns an error wrapping [ErrNotFound] if no such user exists,
// and an error wrapping [context.DeadlineExceeded] if the query times out.
func (s *Store) Get(ctx context.Context, id string) (User, error)
```

Say "wrapping", not "returns ErrNotFound" — that tells the caller to use `errors.Is`.

## Capstone: a store package and its caller

Two packages. `store` exports two sentinels and one typed error; `main` dispatches with
`errors.Is` and `errors.As`.

`store/store.go`:

```go
package store

import (
  "errors"
  "fmt"
  "strings"
)

var (
  ErrNotFound = errors.New("not found")
  ErrConflict = errors.New("already exists")
)

type ValidationError struct {
  Field  string
  Reason string
}

func (e *ValidationError) Error() string {
  return fmt.Sprintf("invalid %s: %s", e.Field, e.Reason)
}

type User struct {
  ID    string
  Email string
}

type Store struct {
  users map[string]User
}

func New() *Store {
  return &Store{users: map[string]User{"1": {ID: "1", Email: "a@example.com"}}}
}

func (s *Store) Get(id string) (User, error) {
  u, ok := s.users[id]
  if !ok {
    return User{}, fmt.Errorf("get user %s: %w", id, ErrNotFound)
  }
  return u, nil
}

func (s *Store) Create(u User) error {
  if u.ID == "" {
    return fmt.Errorf("create user: %w", &ValidationError{Field: "id", Reason: "must not be empty"})
  }
  if !strings.Contains(u.Email, "@") {
    return fmt.Errorf("create user: %w", &ValidationError{Field: "email", Reason: "must contain @"})
  }
  if _, ok := s.users[u.ID]; ok {
    return fmt.Errorf("create user %s: %w", u.ID, ErrConflict)
  }
  s.users[u.ID] = u
  return nil
}
```

`main.go`:

```go
package main

import (
  "errors"
  "fmt"

  "example.com/app/store"
)

func describe(err error) string {
  switch {
  case err == nil:
    return "ok"
  case errors.Is(err, store.ErrNotFound):
    return "404 not found"
  case errors.Is(err, store.ErrConflict):
    return "409 conflict"
  }
  var ve *store.ValidationError
  if errors.As(err, &ve) {
    return fmt.Sprintf("400 bad request: field=%s reason=%s", ve.Field, ve.Reason)
  }
  return "500 internal"
}

func main() {
  s := store.New()

  _, err := s.Get("1")
  fmt.Printf("%-38v -> %s\n", err, describe(err))

  _, err = s.Get("99")
  fmt.Printf("%-38v -> %s\n", err, describe(err))

  err = s.Create(store.User{ID: "2", Email: "b@example.com"})
  fmt.Printf("%-38v -> %s\n", err, describe(err))

  err = s.Create(store.User{ID: "2", Email: "b@example.com"})
  fmt.Printf("%-38v -> %s\n", err, describe(err))

  err = s.Create(store.User{ID: "", Email: "c@example.com"})
  fmt.Printf("%-38v -> %s\n", err, describe(err))

  err = s.Create(store.User{ID: "3", Email: "nope"})
  fmt.Printf("%-38v -> %s\n", err, describe(err))

  err = errors.New("unexpected")
  fmt.Printf("%-38v -> %s\n", err, describe(err))
}
```

Output:

```
<nil>                                  -> ok
get user 99: not found                 -> 404 not found
<nil>                                  -> ok
create user 2: already exists          -> 409 conflict
create user: invalid id: must not be empty -> 400 bad request: field=id reason=must not be empty
create user: invalid email: must contain @ -> 400 bad request: field=email reason=must contain @
unexpected                             -> 500 internal
```

The shape to take away: a `switch { case errors.Is(...) }` for the coarse outcomes, then
`errors.As` for the one that carries fields, then a default that does *not* guess. 8.6
turns this `describe` function into a real HTTP status mapper.

The `users` map is unsynchronized; concurrent `Create` calls would race (module 11).

## Quick Reference

| Declaration | Form |
|---|---|
| sentinel | `var ErrNotFound = errors.New("not found")` |
| custom type | `type XError struct{ ... }` + `func (e *XError) Error() string` |
| carries a cause | add `func (e *XError) Unwrap() error { return e.Err }` |
| matches a sentinel | add `func (e *XError) Is(target error) bool` |
| behaviour interface | `type NotFound interface{ NotFound() bool }` |
| behaviour helper | `func IsNotFound(err error) bool { var nf NotFound; return errors.As(err, &nf) && nf.NotFound() }` |

| Convention | Rule |
|---|---|
| sentinel name | `Err` prefix: `ErrNotFound`, `ErrClosed` |
| type name | `Error` suffix: `ValidationError`, `PathError` |
| receiver | pointer |
| return | `fmt.Errorf("op %s: %w", id, ErrX)` — wrap, do not return bare |
| doc comment | "returns an error wrapping [ErrX] if ..." |
| exported sentinel | permanent API; export only what callers branch on |

| stdlib sentinel | Package | Meaning |
|---|---|---|
| `io.EOF` | `io` | stream ended legally |
| `io.ErrUnexpectedEOF` | `io` | stream ended mid-value |
| `sql.ErrNoRows` | `database/sql` | `QueryRow.Scan` found nothing |
| `fs.ErrNotExist` | `io/fs` (aliased as `os.ErrNotExist`) | no such file |
| `fs.ErrPermission` | `io/fs` | not permitted |
| `fs.ErrExist` | `io/fs` | already exists |
| `context.Canceled` | `context` | caller gave up |
| `context.DeadlineExceeded` | `context` | too slow |

## Common Mistakes

**Comparing a sentinel with `==` instead of `errors.Is`.** Any wrap upstream breaks it;
`errors.Is` costs nothing extra.

**Declaring an error type with a value receiver.** Two unrelated failures compare equal
and the struct is snapshotted into the interface; use a pointer receiver.

**Returning a bare sentinel from every layer.** The message then has no context about
what was being attempted; wrap it with `fmt.Errorf("op %s: %w", id, ErrX)`.

**Exporting a sentinel for every internal failure.** Each one is a permanent API
commitment; export the ones callers must branch on and keep the rest unexported.

**Returning `io.EOF` from a function that is not a Reader.** Callers cannot tell your
"done" from a truncated stream; convert it to `nil` or to your own sentinel.

**Declaring an `errors.As` target as `var e MyError` when the receiver is `*MyError`.**
It never matches; declare `var e *MyError`.

## Exercises

1. Add `Update` and `Delete` to the capstone `store` package, returning `ErrNotFound` for
   a missing ID and a `*ValidationError` for a bad email. Extend `describe` and verify
   every branch, including the default.
2. Replace the capstone's `ErrNotFound` sentinel with a `NotFoundError{Resource, ID}` type
   plus a custom `Is` that still matches `ErrNotFound`. Show that both the old
   `errors.Is(err, store.ErrNotFound)` callers and new `errors.As` callers work unchanged.
3. Build a `retry` package exporting only `type Retryable interface { Retryable() bool }`
   and `func Do(n int, f func() error) error`. Have `Do` stop early on a non-retryable
   error. Test it with a function returning a `net.Dial` failure and one returning a
   `*ValidationError`.
4. Write a table test for the capstone's `describe` using `errors.Is`/`errors.As` only —
   no string comparison. Then change a message in `store` and confirm your tests still
   pass.
5. Design the error API for a JSON config loader that can fail from: file missing, bad
   permissions, malformed JSON (with line number), and an unknown key (with the key name).
   Decide sentinel vs type vs behaviour for each, write the doc comments, and justify each
   choice in one sentence.

---

[← 8.3 errors.Is, errors.As, and Unwrap](03-is-as-unwrap.md) · [Course Index](../README.md) · [8.5 errors.Join and Multiple Errors →](05-join.md)
