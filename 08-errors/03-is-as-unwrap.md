[← 8.2 Wrapping with fmt.Errorf and %w](02-wrapping.md) · [Course Index](../README.md) · [8.4 Sentinel Errors and Custom Error Types →](04-sentinels-custom.md)

# 8.3 — errors.Is, errors.As, and Unwrap

## What You'll Learn

- `errors.Is` walks the chain comparing with `==`, or calls your `Is(error) bool`
- `errors.As` finds the first error in the chain assignable to a target — which must be a non-nil pointer
- `errors.Unwrap` peels exactly one layer and returns `nil` at the bottom
- `Is` for identity, `As` for extracting fields, `==` almost never

## Coming From Another Language

| Other | Go |
|---|---|
| JS `e instanceof HttpError` | `var he *HTTPError; errors.As(err, &he)` |
| JS `e.code === 'ENOENT'` | `errors.Is(err, fs.ErrNotExist)` |
| Python `except FileNotFoundError as e: e.filename` | `var pe *fs.PathError; errors.As(err, &pe); pe.Path` |
| Python `isinstance(e, ValueError)` | `errors.As` with the concrete type or an interface |
| Rust `matches!(e, MyErr::NotFound)` | `errors.Is(err, ErrNotFound)` |
| Rust `e.downcast_ref::<io::Error>()` | `errors.As` — the closest analogue |

`errors.As` is Go's `catch (e instanceof X)`, written out longhand, with an out-parameter
because Go has no generic downcast syntax.

## errors.Is walks the chain

```go
package main

import (
  "errors"
  "fmt"
)

var ErrNotFound = errors.New("not found")

func main() {
  deep := fmt.Errorf("handler: %w", fmt.Errorf("service: %w", fmt.Errorf("store: %w", ErrNotFound)))
  fmt.Println(deep)
  fmt.Println(errors.Is(deep, ErrNotFound))
  fmt.Println(deep == ErrNotFound)

  other := errors.New("not found")
  fmt.Println(errors.Is(deep, other))

  fmt.Println(errors.Is(nil, ErrNotFound))
  fmt.Println(errors.Is(ErrNotFound, nil))
  fmt.Println(errors.Is(nil, nil))
}
```

Output:

```
handler: service: store: not found
true
false
false
false
false
true
```

Three layers deep, `errors.Is` still finds it. `==` does not. A *different* `errors.New`
with the same text does not match, because comparison is pointer identity (8.1).

Note the nil behaviour: `errors.Is(nil, target)` is false, `errors.Is(err, nil)` is false,
and `errors.Is(nil, nil)` is true. So `errors.Is` is safe to call on a possibly-nil error
without a guard.

## errors.As extracts the concrete error

This is where the payoff is. A real `os.Open` failure produces a `*fs.PathError` with
fields you want.

```go
package main

import (
  "errors"
  "fmt"
  "io/fs"
  "os"
)

func main() {
  _, err := os.Open("/etc/app/missing.yaml")
  wrapped := fmt.Errorf("load config: %w", err)
  fmt.Println(wrapped)

  var pe *fs.PathError
  if errors.As(wrapped, &pe) {
    fmt.Printf("Op=%q Path=%q Err=%v\n", pe.Op, pe.Path, pe.Err)
    fmt.Printf("%T\n", pe.Err)
  }

  fmt.Println(errors.Is(wrapped, fs.ErrNotExist))
  fmt.Println(errors.Is(wrapped, os.ErrNotExist))
  fmt.Println(fs.ErrNotExist == os.ErrNotExist)

  _, err2 := os.Open("/etc/master.passwd")
  fmt.Println(err2)
  var pe2 *fs.PathError
  if errors.As(err2, &pe2) {
    fmt.Printf("Op=%q Path=%q Err=%v\n", pe2.Op, pe2.Path, pe2.Err)
  }
  fmt.Println("permission:", errors.Is(err2, fs.ErrPermission))
}
```

Output:

```
load config: open /etc/app/missing.yaml: no such file or directory
Op="open" Path="/etc/app/missing.yaml" Err=no such file or directory
syscall.Errno
true
true
true
open /etc/master.passwd: permission denied
Op="open" Path="/etc/master.passwd" Err=permission denied
permission: true
```

`errors.As` reached through the `fmt.Errorf` wrap and gave you the struct, so you can log
the path or report which file was missing. `errors.Is` is the right tool for the *kind* of
failure (`fs.ErrNotExist` vs `fs.ErrPermission`), `errors.As` for the *details*.

`os.ErrNotExist` and `fs.ErrNotExist` are the same value — `os` aliases the `io/fs`
sentinels. Prefer the `fs` names in new code.

Both failures above are genuine: the first file does not exist, the second exists but is
mode 0600 and owned by root.

## errors.As panics on a non-pointer target

The target must be a non-nil pointer to a type implementing `error`, or to an interface.
Get it wrong and it is a runtime panic, not a compile error.

```go
package main

import (
  "errors"
  "fmt"
  "io/fs"
  "os"
)

func main() {
  _, err := os.Open("/no/such/file")
  var pe fs.PathError
  fmt.Println(errors.As(err, pe))
}
```

`go vet` catches this one:

```
main.go:13:14: second argument to errors.As must be a non-nil pointer to either a type that implements error, or to any interface type
```

Run it anyway and it panics on purpose. Output:

```
panic: errors: target must be a non-nil pointer

goroutine 1 [running]:
errors.As({0x104b27048?, 0x228fed6940f0?}, {0x104b1e258?, 0x228fed694120?})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/errors/wrap.go:112 +0x1cc
main.main()
	/tmp/scratch/main.go:13 +0x74
exit status 2
```

A *nil* pointer of the right type panics identically — `errors.As(err, pe)` where
`var pe *fs.PathError` gives the same `target must be a non-nil pointer`. You must pass
the address of the variable: `errors.As(err, &pe)`. Keep `go vet` in your pipeline and
this class of bug never reaches runtime.

## A custom Is for semantic matching

`errors.Is` checks, at each level of the chain, whether the error has a method
`Is(error) bool` and calls it. That lets one error match a *class* of targets.

```go
package main

import (
  "errors"
  "fmt"
)

type HTTPError struct {
  Code int
}

func (e *HTTPError) Error() string { return fmt.Sprintf("http %d", e.Code) }

var ErrClientSide = errors.New("client error")
var ErrServerSide = errors.New("server error")

func (e *HTTPError) Is(target error) bool {
  switch target {
  case ErrClientSide:
    return e.Code >= 400 && e.Code < 500
  case ErrServerSide:
    return e.Code >= 500 && e.Code < 600
  }
  return false
}

func main() {
  for _, code := range []int{404, 429, 500, 503, 200} {
    err := fmt.Errorf("request: %w", &HTTPError{Code: code})
    fmt.Printf("%d client=%v server=%v\n", code, errors.Is(err, ErrClientSide), errors.Is(err, ErrServerSide))
  }

  a := &HTTPError{Code: 404}
  b := &HTTPError{Code: 404}
  fmt.Println(a == b, errors.Is(a, b))
}
```

Output:

```
404 client=true server=false
429 client=true server=false
500 client=false server=true
503 client=false server=true
200 client=false server=false
```

```
false false
```

One type, two semantic sentinels, no table of every status code. Note the last line: two
separate `&HTTPError{404}` values are neither `==` nor `errors.Is`-equal, because the
custom `Is` only answers the two class sentinels and falls through to `false`. If you want
value equality too, add a `case` for `*HTTPError` in `Is`, or use a value receiver.

Implementing `As(any) bool` is also possible but almost never needed — the default
assignability check covers normal types. Add it only when your error must present itself
as some *other* concrete type.

## errors.As with an interface target

This is the behaviour-based check: match on what the error can *do*, not what it is.

```go
package main

import (
  "errors"
  "fmt"
  "net"
  "os"
)

type Temporary interface {
  Temporary() bool
}

type flaky struct{}

func (flaky) Error() string   { return "flaky backend" }
func (flaky) Temporary() bool { return true }

func main() {
  err := fmt.Errorf("call api: %w", flaky{})
  var t Temporary
  if errors.As(err, &t) {
    fmt.Println("temporary:", t.Temporary())
  }

  _, nerr := net.Dial("tcp", "127.0.0.1:1")
  fmt.Println(nerr)
  var oe *net.OpError
  if errors.As(nerr, &oe) {
    fmt.Printf("Op=%q Net=%q\n", oe.Op, oe.Net)
  }
  var se *os.SyscallError
  fmt.Println("syscall:", errors.As(nerr, &se))
  if se != nil {
    fmt.Printf("Syscall=%q Err=%v\n", se.Syscall, se.Err)
  }

  var t2 Temporary
  fmt.Println(errors.As(nil, &t2))
  fmt.Println(errors.Unwrap(nil))
}
```

Output:

```
temporary: true
dial tcp 127.0.0.1:1: connect: connection refused
Op="dial" Net="tcp"
syscall: true
Syscall="connect" Err=connection refused
false
<nil>
```

A real refused connection, unwrapped two levels into the `os.SyscallError`. An interface
target means any package's error can satisfy your check without importing your types — the
pattern 8.4 builds on.

`errors.As(nil, &target)` is false, and `errors.Unwrap(nil)` is `nil`, so neither needs a
guard.

## errors.Unwrap and manual chain walking

`errors.Unwrap` peels exactly one layer. Loop until `nil` to see the whole chain.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "os"
  "time"
)

func main() {
  _, err := os.Open("/no/such/file")
  top := fmt.Errorf("outer: %w", fmt.Errorf("middle: %w", err))

  depth := 0
  for e := top; e != nil; e = errors.Unwrap(e) {
    fmt.Printf("%d %T %v\n", depth, e, e)
    depth++
  }
  fmt.Println("depth:", depth)

  ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond)
  defer cancel()
  <-ctx.Done()
  cerr := fmt.Errorf("fetch: %w", ctx.Err())
  fmt.Println(cerr)
  fmt.Println("deadline:", errors.Is(cerr, context.DeadlineExceeded))
  fmt.Println("canceled:", errors.Is(cerr, context.Canceled))

  ctx2, cancel2 := context.WithCancel(context.Background())
  cancel2()
  fmt.Println(ctx2.Err(), errors.Is(ctx2.Err(), context.Canceled))
}
```

Output:

```
0 *fmt.wrapError outer: middle: open /no/such/file: no such file or directory
1 *fmt.wrapError middle: open /no/such/file: no such file or directory
2 *fs.PathError open /no/such/file: no such file or directory
3 syscall.Errno no such file or directory
depth: 4
fetch: context deadline exceeded
deadline: true
canceled: false
context canceled true
```

Two wraps you wrote plus two the stdlib built: the chain is four deep. Walking it by hand
is a debugging and logging tool, not a control-flow tool — use `Is`/`As` for decisions.

`context.DeadlineExceeded` and `context.Canceled` are distinct sentinels, and the
distinction matters: a deadline means the work was too slow and may be worth retrying, a
cancellation means the caller gave up and retrying is pointless (8.6).

Note `errors.Unwrap` returns `nil` for a multi-error (`Unwrap() []error`), so this loop
stops at a join. `Is` and `As` traverse the tree; manual walking does not (8.5).

## io.EOF vs io.ErrUnexpectedEOF

`io.EOF` is not a failure — it is the normal end-of-stream signal, and it is a sentinel
you compare, never wrap for the caller to guess at.

```go
package main

import (
  "encoding/binary"
  "errors"
  "fmt"
  "io"
  "strings"
)

func main() {
  r := strings.NewReader("abcde")
  buf := make([]byte, 2)
  for {
    n, err := r.Read(buf)
    fmt.Printf("n=%d data=%q err=%v\n", n, buf[:n], err)
    if err != nil {
      fmt.Println("is EOF:", errors.Is(err, io.EOF))
      break
    }
  }

  short := strings.NewReader("ab")
  var v uint32
  err := binary.Read(short, binary.BigEndian, &v)
  fmt.Println(err)
  fmt.Println("is EOF:", errors.Is(err, io.EOF))
  fmt.Println("is ErrUnexpectedEOF:", errors.Is(err, io.ErrUnexpectedEOF))

  empty := strings.NewReader("")
  err = binary.Read(empty, binary.BigEndian, &v)
  fmt.Println(err, errors.Is(err, io.EOF))

  _, err = io.ReadFull(strings.NewReader("ab"), make([]byte, 4))
  fmt.Println(err, errors.Is(err, io.ErrUnexpectedEOF))
}
```

Output:

```
n=2 data="ab" err=<nil>
n=2 data="cd" err=<nil>
n=1 data="e" err=<nil>
n=0 data="" err=EOF
is EOF: true
unexpected EOF
is EOF: false
is ErrUnexpectedEOF: true
EOF true
unexpected EOF true
```

The distinction: **`io.EOF` means the stream ended where ending was legal.
`io.ErrUnexpectedEOF` means it ended mid-value.** Reading a `uint32` from a 2-byte reader
is truncation (`ErrUnexpectedEOF`); from a 0-byte reader it is a clean end (`EOF`). A
protocol decoder that treats them the same silently accepts truncated input.

`errors.Is(err, io.EOF)` rather than `err == io.EOF` is the safe habit — identical here,
but correct if anything in the stack starts wrapping.

## Is vs As vs ==

| Question | Tool | Example |
|---|---|---|
| Is this *this specific* error? | `errors.Is` | `errors.Is(err, fs.ErrNotExist)` |
| Is this a *kind* of error, and what are its fields? | `errors.As` | `var pe *fs.PathError; errors.As(err, &pe)` |
| Can this error *do* something? | `errors.As` with an interface target | `var t Temporary; errors.As(err, &t)` |
| Exactly equal, guaranteed unwrapped | `==` | only inside the package that created it |
| What is the full chain, for a log? | `errors.Unwrap` in a loop | debugging |

Rule: reach for `errors.Is` first. Use `errors.As` the moment you need a value out of the
error. Never write `==` against a sentinel in code that receives errors from elsewhere,
because one `fmt.Errorf("...: %w", err)` added upstream breaks it with no compile error.

Each `Is`/`As` call walks the chain, calling `Unwrap` and (for `Is`) an interface
assertion per level — a handful of nanoseconds for realistic depths, irrelevant next to
whatever I/O produced the error.

## Quick Reference

| Function | Signature | Returns |
|---|---|---|
| `errors.Is` | `func(err, target error) bool` | true if any error in the tree `==` target or its `Is(target)` says so |
| `errors.As` | `func(err error, target any) bool` | true and assigns if any error in the tree is assignable to `*target` |
| `errors.Unwrap` | `func(err error) error` | one layer down, or `nil` |
| `errors.Join` | `func(errs ...error) error` | a tree node (8.5) |

| Method you implement | Effect |
|---|---|
| `Unwrap() error` | one child in the chain |
| `Unwrap() []error` | several children; a tree node |
| `Is(target error) bool` | custom identity for `errors.Is` |
| `As(target any) bool` | custom extraction for `errors.As`; rarely needed |

| Nil input | `Is` | `As` | `Unwrap` |
|---|---|---|---|
| `err == nil` | false (unless target nil) | false | `nil` |
| `target == nil` | true only if `err == nil` | panics if target not a pointer | n/a |

| Common stdlib target | Match with |
|---|---|
| `fs.ErrNotExist` / `os.ErrNotExist` | `errors.Is` |
| `fs.ErrPermission` | `errors.Is` |
| `sql.ErrNoRows` | `errors.Is` |
| `io.EOF`, `io.ErrUnexpectedEOF` | `errors.Is` |
| `context.Canceled`, `context.DeadlineExceeded` | `errors.Is` |
| `*fs.PathError` (Op, Path, Err) | `errors.As` |
| `*net.OpError` (Op, Net, Addr) | `errors.As` |
| `*os.SyscallError` (Syscall, Err) | `errors.As` |
| `*strconv.NumError` (Func, Num, Err) | `errors.As` |
| `*json.UnmarshalTypeError` (Field, Type) | `errors.As` |

## Common Mistakes

**Passing a value instead of a pointer to `errors.As`.** It panics at runtime with
`target must be a non-nil pointer`; pass `&target` and keep `go vet` enabled.

**Using `==` against a sentinel on an error that crossed a package boundary.** One
upstream `%w` wrap breaks it silently; use `errors.Is`.

**Declaring the `As` target as a value type when the error has a pointer receiver.**
`var pe fs.PathError` never matches a `*fs.PathError`; declare `var pe *fs.PathError`.

**Treating `io.EOF` as a failure to report.** It is the normal end of a stream; only
`io.ErrUnexpectedEOF` means truncation.

**Walking the chain with `errors.Unwrap` to make a decision.** It stops at a joined
error and reimplements `Is` badly; use `errors.Is`/`errors.As`.

**Not distinguishing `context.Canceled` from `context.DeadlineExceeded`.** One means the
caller left, the other means you were slow; only the second is a retry candidate.

## Exercises

1. Write a function that takes an `error` and returns a human sentence, dispatching on
   `fs.ErrNotExist`, `fs.ErrPermission`, and `*fs.PathError` (naming the path). Drive it
   with real `os.Open` failures for a missing file, a file you have no permission to read,
   and a directory opened for writing.
2. Give `*HTTPError` a value-equality `Is` so that two `&HTTPError{404}` values match,
   while still matching `ErrClientSide`. Write a table test covering all four
   combinations.
3. Implement `type Retryable interface { Retryable() bool }` and a `net`-backed function
   that returns a retryable error on connection refused and a non-retryable one on an
   unknown host. Dispatch with `errors.As` on the interface and verify with real
   `net.Dial` calls.
4. Write `func Chain(err error) []error` returning every error in the chain, outermost
   first, using only `errors.Unwrap`. Then build an error with `errors.Join` and show why
   your function is incomplete.
5. Benchmark `errors.Is` against a chain 1, 5, and 50 wraps deep with
   `go test -bench . -benchmem`. Report ns/op and state, in one sentence, whether the
   result should change how you write error handling.

---

[← 8.2 Wrapping with fmt.Errorf and %w](02-wrapping.md) · [Course Index](../README.md) · [8.4 Sentinel Errors and Custom Error Types →](04-sentinels-custom.md)
