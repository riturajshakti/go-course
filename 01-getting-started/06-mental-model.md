[← 1.5 gofmt, go vet, and the Editor Setup](05-tooling.md) · [Course Index](../README.md) · [2.1 Variables, Zero Values, and Declaration Forms →](../02-syntax-basics/01-variables.md)

# 1.6 — How Go Thinks: A Mental Model

## What You'll Learn

- Errors are return values, not control flow; there is no `try`, `catch`, or `throw`
- Every type has a usable zero value, so there is no `null`, `undefined`, or `Optional`
- Interfaces are satisfied implicitly, and composition replaces inheritance entirely
- Goroutines are not coloured: one function signature works for blocking and concurrent calls
- Explicitness is the consistent tiebreaker — in conversions, conditions, and dependency choices

## Coming From Another Language

| Other | Go |
|---|---|
| `try { } catch (e) { }` | `v, err := f()` then `if err != nil` |
| `null` / `undefined` / `None` / `Option<T>` | the zero value: `0`, `""`, `false`, `nil`, `{}` |
| `class X extends Y` | struct embedding; no inheritance, no virtual dispatch |
| `class X implements I` | nothing declared; the method set decides |
| `async fn` / `await`, coloured functions | `go f()`; every function is callable both ways |
| 400 transitive npm deps | stdlib first, then four or five direct deps |
| Prettier config negotiated per team | gofmt, no options |

## Errors are values

There is no exception mechanism. A function that can fail returns an extra
`error`, and the `if err != nil` block is the single most common five lines in all
Go code.

```go
package main

import (
  "fmt"
  "strconv"
)

func main() {
  n, err := strconv.Atoi("42")
  fmt.Println(n, err)

  n, err = strconv.Atoi("forty-two")
  fmt.Println(n, err)

  if _, err := strconv.Atoi("x"); err != nil {
    fmt.Println("handled:", err)
  }
}
```

Output:

```
42 <nil>
0 strconv.Atoi: parsing "forty-two": invalid syntax
handled: strconv.Atoi: parsing "x": invalid syntax
```

Note the failing call returned `0`, not garbage and not a sentinel — the zero
value, always. And note that the error is a plain value you can store, compare,
pass to a function, or put in a struct field.

The cost is verbosity, and it is real. The payoff is that every failure path is
visible at the call site — you cannot accidentally propagate an error through
five frames, because propagating requires typing `return err`.

Wrapping adds context without losing the original:

```go
package main

import (
  "errors"
  "fmt"
  "os"
)

func load(path string) ([]byte, error) {
  b, err := os.ReadFile(path)
  if err != nil {
    return nil, fmt.Errorf("load %s: %w", path, err)
  }
  return b, nil
}

func main() {
  _, err := load("/nope/missing.txt")
  fmt.Println(err)
  fmt.Println(errors.Is(err, os.ErrNotExist))

  var pe *os.PathError
  fmt.Println(errors.As(err, &pe), pe.Op, pe.Path)
}
```

Output:

```
load /nope/missing.txt: open /nope/missing.txt: no such file or directory
true
true open /nope/missing.txt
```

That is the whole error model, and module 08 is nothing but elaborations of it.

`panic` exists but is not an exception. It is for programmer error — a broken
invariant, an impossible state — and it unwinds and crashes:

```go
package main

import "fmt"

func mustPositive(n int) int {
  if n < 0 {
    panic(fmt.Sprintf("negative: %d", n))
  }
  return n
}

func main() {
  fmt.Println(mustPositive(5))
  fmt.Println(mustPositive(-1))
}
```

This program panics on purpose. Output:

```
5
panic: negative: -1

goroutine 1 [running]:
main.mustPositive(...)
	/tmp/scratch/main.go:7
main.main()
	/tmp/scratch/main.go:14 +0x94
exit status 2
```

`recover` can catch it, but using panic/recover as a control-flow mechanism across
package boundaries is wrong in Go in a way that has no equivalent in Python or
JS. A library that panics on bad input instead of returning an error is a library
with a bug.

## Zero values instead of null

Every type has a zero value, produced by `var x T` with no initializer, and the
design goal is that it be *useful*.

```go
package main

import "fmt"

type Point struct {
  X, Y int
}

type Server struct {
  Name    string
  Port    int
  Handler func()
  Tags    []string
  Meta    map[string]string
  Next    *Server
}

func main() {
  var p Point
  var s Server
  var i int
  var f float64
  var str string
  var b bool
  var sl []int
  var m map[string]int

  fmt.Printf("%v %v %q %v\n", i, f, str, b)
  fmt.Printf("%+v\n", p)
  fmt.Printf("%+v\n", s)
  fmt.Println(sl == nil, len(sl), m == nil, len(m))
  fmt.Println(m["missing"])
  sl = append(sl, 1)
  fmt.Println(sl)
}
```

Output:

```
0 0 "" false
{X:0 Y:0}
{Name: Port:0 Handler:<nil> Tags:[] Meta:map[] Next:<nil>}
true 0 true 0
0
[1]
```

Three consequences worth internalising. There is no uninitialised memory and no
`undefined` — reading a field you never set gives a defined value. A nil slice has
`len` 0 and `append` works on it, so `var xs []T` is a valid empty list and you
never need `make` just to start appending. And reading a missing map key returns
the value type's zero rather than raising, which is why the comma-ok form
`v, ok := m[k]` exists to distinguish "absent" from "present and zero".

The flip side: `nil` map *writes* panic, and `0` is not distinguishable from
"unset" for an `int` field. When you genuinely need absence, you use a pointer or
the comma-ok form, not an `Option` type.

## No classes, no inheritance

Methods attach to named types by declaration, outside any class body. There is no
`this`, no constructor keyword, and no superclass.

```go
package main

import "fmt"

type Engine struct {
  HP int
}

func (e Engine) Start() string {
  return fmt.Sprintf("engine %dhp started", e.HP)
}

type Car struct {
  Engine
  Brand string
}

func main() {
  c := Car{Engine: Engine{HP: 300}, Brand: "Dacia"}
  fmt.Println(c.Start())
  fmt.Println(c.HP, c.Engine.HP, c.Brand)
  fmt.Printf("%+v\n", c)
}
```

Output:

```
engine 300hp started
300 300 Dacia
{Engine:{HP:300} Brand:Dacia}
```

`Car` embeds `Engine`, so `Engine`'s fields and methods are *promoted* to `Car`.
It reads like inheritance and is not: `c.Start()` is sugar for
`c.Engine.Start()`, resolved statically at compile time. There is no vtable, no
`super`, and `Car` cannot override `Start` polymorphically — a `[]Engine` will
never contain a `Car`. What you get is delegation with less typing, and the
embedded value remains an ordinary accessible field.

## Interfaces are satisfied implicitly

You never write `implements`. A type satisfies an interface by having the methods,
and the interface can be declared *after* the type, in a different package, by
someone who has never seen it.

```go
package main

import "fmt"

type Shape interface {
  Area() float64
}

type Rect struct{ W, H float64 }
type Circle struct{ R float64 }

func (r Rect) Area() float64   { return r.W * r.H }
func (c Circle) Area() float64 { return 3.14159 * c.R * c.R }

func total(shapes []Shape) float64 {
  var sum float64
  for _, s := range shapes {
    sum += s.Area()
  }
  return sum
}

func main() {
  fmt.Printf("%.2f\n", total([]Shape{Rect{3, 4}, Circle{1}}))

  switch v := any(Rect{3, 4}).(type) {
  case Rect:
    fmt.Println("rect", v.W, v.H)
  case Circle:
    fmt.Println("circle", v.R)
  }
}
```

Output:

```
15.14
rect 3 4
```

`Rect` and `Circle` have no idea `Shape` exists. The effect is that interfaces get
defined by *consumers*, sized to what the consumer needs — which is why Go's
most-used interfaces have one method:

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "os"
  "strings"
)

type Loud struct {
  dst io.Writer
}

func (l Loud) Write(p []byte) (int, error) {
  return l.dst.Write([]byte(strings.ToUpper(string(p))))
}

func emit(w io.Writer, msg string) {
  fmt.Fprintf(w, "%s\n", msg)
}

func main() {
  var buf bytes.Buffer
  emit(&buf, "to a bytes.Buffer")
  fmt.Print(buf.String())

  emit(os.Stdout, "to stdout")
  emit(Loud{dst: os.Stdout}, "to my own writer")
}
```

Output:

```
to a bytes.Buffer
to stdout
TO MY OWN WRITER
```

One `io.Writer` method made `emit` work against an in-memory buffer, a file
descriptor, and a type invented in this file. Files, network connections, HTTP
response bodies, gzip encoders, and hash functions are all `io.Writer`, so they
compose without adapters. This is Go's main abstraction mechanism, and it is
almost the only one.

## Goroutines are not coloured

In JS and Python, `async` infects every caller: an async function can only be
awaited from an async function, so adding concurrency forces a signature change
up the whole call stack. Go has no such split. `fetch` below is an ordinary
blocking function, and `go` makes the *call site* concurrent.

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

func fetch(path string) int {
  time.Sleep(100 * time.Millisecond)
  return len(path)
}

func main() {
  start := time.Now()
  paths := []string{"/a", "/bb", "/ccc"}
  results := make([]int, len(paths))

  var wg sync.WaitGroup
  for i, p := range paths {
    wg.Add(1)
    go func() {
      defer wg.Done()
      results[i] = fetch(p)
    }()
  }
  wg.Wait()

  fmt.Println(results)
  fmt.Println("concurrent:", time.Since(start) < 200*time.Millisecond)
}
```

Output:

```
[2 3 4]
concurrent: true
```

Three 100 ms sleeps finished in well under 200 ms. `fetch` needed no `async`
marker, no `Promise`, no executor, and it remains directly callable as
`fetch("/a")`. The runtime multiplexes goroutines onto OS threads and parks them
on blocking syscalls, so blocking code *is* the async code. A goroutine starts at
~2 KB of stack that grows on demand, which is why spawning ten thousand is
routine.

What you trade away: the scheduler means output interleaving is nondeterministic,
and sharing `results` across goroutines is only safe here because each writes a
distinct index. Go gives you `go` cheaply and `-race` to check your work; modules
10 through 12 are about using it without corrupting data.

## Explicit beats clever

No implicit numeric conversion, at all, in either direction:

```go
package main

import "fmt"

func main() {
  var i int = 3
  var f float64 = 2.5
  fmt.Println(i + f)

  var n int32 = 7
  var m int64 = 8
  fmt.Println(n + m)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: i + f (mismatched types int and float64)
./main.go:12:15: invalid operation: n + m (mismatched types int32 and int64)
```

Note `int32 + int64` fails too — same signedness, different width, still an
error. The fix is always a visible conversion:

```go
package main

import "fmt"

func main() {
  var i int = 3
  var f float64 = 2.5
  fmt.Println(float64(i) + f)

  var n int32 = 7
  var m int64 = 8
  fmt.Println(int64(n) + m)

  b := []byte("hi")
  fmt.Println(string(b), b)
}
```

Output:

```
5.5
15
hi [104 105]
```

And no truthiness — a condition must be a `bool`:

```go
package main

import "fmt"

func main() {
  s := ""
  n := 0
  if s {
    fmt.Println("x")
  }
  if n {
    fmt.Println("y")
  }
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:6: non-boolean condition in if statement
./main.go:11:6: non-boolean condition in if statement
```

So `if len(s) > 0`, `if n != 0`, `if err != nil`. Every condition states what it
tests. The same instinct produces the unused-import and unused-variable errors
from lesson 1.2, and `defer` instead of a destructor or `with` block:

```go
package main

import (
  "fmt"
  "os"
  "strings"
)

func main() {
  f, err := os.CreateTemp("", "demo")
  if err != nil {
    fmt.Println(err)
    return
  }
  defer os.Remove(f.Name())
  defer f.Close()

  fmt.Fprintln(f, "line one")
  f.Sync()

  b, _ := os.ReadFile(f.Name())
  fmt.Println(strings.TrimSpace(string(b)))
  fmt.Println("temp file will be removed by defer")
}
```

Output:

```
line one
temp file will be removed by defer
```

The cleanup sits next to the acquisition, in LIFO order, and runs on every exit
path including a panic. Nothing is hidden in a type's destructor.

## Generics exist and the culture resists them

Type parameters arrived in 1.18 and are used sparingly. The stdlib uses them in
exactly three packages — `slices`, `maps`, `cmp` — and almost nowhere else.

```go
package main

import "fmt"

type Stack[T any] struct {
  items []T
}

func (s *Stack[T]) Push(v T) {
  s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
  var zero T
  if len(s.items) == 0 {
    return zero, false
  }
  v := s.items[len(s.items)-1]
  s.items = s.items[:len(s.items)-1]
  return v, true
}

func main() {
  var s Stack[int]
  s.Push(1)
  s.Push(2)
  fmt.Println(s.Pop())
  fmt.Println(s.Pop())
  fmt.Println(s.Pop())
}
```

Output:

```
2 true
1 true
0 false
```

That works, and `var zero T` is the idiom for "the zero value of an unknown type".
But the default reflex in Go code is still an interface or a concrete type, not a
type parameter. The reasoning: interfaces already give you polymorphism, generics
complicate inference and error messages, and a `Map[K,V]`-heavy utility layer is
code nobody can grep. Module 09.5 is a whole lesson on when generics are the wrong
tool — a lesson that would be strange in a Rust or TypeScript course.

## Fast compiles and one static binary

The import DAG plus no header files plus no template instantiation makes the
compiler fast enough to use as a REPL substitute. A program importing `net/http`
reaches 188 packages:

```bash
go list -deps . | wc -l
go clean -cache
time go build -o app .
time go build -o app .
```

Output:

```
     188
go build -o app .  12.58s user 3.50s system 478% cpu 3.361 total
go build -o app .  0.08s user 0.44s system 596% cpu 0.088 total
```

3.4 seconds from a cold cache for 188 packages, 88 ms warm. Those numbers are
machine-specific but the ratio holds: the build cache is per-package and content
addressed, so `go run .` after a one-line edit is effectively instant. This is
why Go has no REPL and nobody misses it.

The output is one file with no runtime to install:

```bash
ls -lh app
otool -L app
```

Output:

```
5.2M app
app:
	/usr/lib/libSystem.B.dylib (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libresolv.9.dylib (compatibility version 0.0.0, current version 0.0.0)
	/System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 0.0.0, current version 0.0.0)
	/System/Library/Frameworks/Security.framework/Versions/A/Security (compatibility version 0.0.0, current version 0.0.0)
```

5.2 MB, and the only dynamic links are macOS system libraries for DNS and the
keychain. Cross-compiling needs no toolchain install and no container:

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o app-linux .
file app-linux
go build -ldflags="-s -w" -o app-small .
ls -lh app-small
```

Output:

```
app-linux: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=..., with debug_info, not stripped
3.5M app-small
```

`statically linked` is the operative word: that binary runs in a `FROM scratch`
container. `-ldflags="-s -w"` drops the symbol table and DWARF for 3.5 MB, at the
cost of readable panic tracebacks. The 5 MB floor is the garbage collector,
scheduler, and reflection metadata — unavoidable, and the reason Go binaries are
large next to C and small next to a JVM or Node image.

## Stdlib first

The 188-package HTTP server above has zero third-party dependencies. A routed
JSON API with a test client is still zero:

```go
package main

import (
  "encoding/json"
  "fmt"
  "net/http"
  "net/http/httptest"
)

func handler(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "application/json")
  json.NewEncoder(w).Encode(map[string]any{"id": r.PathValue("id"), "ok": true})
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("GET /users/{id}", handler)

  srv := httptest.NewServer(mux)
  defer srv.Close()

  resp, err := http.Get(srv.URL + "/users/7")
  if err != nil {
    fmt.Println("error:", err)
    return
  }
  defer resp.Body.Close()

  var body map[string]any
  json.NewDecoder(resp.Body).Decode(&body)
  fmt.Println(resp.Status, resp.Header.Get("Content-Type"))
  fmt.Println(body)
}
```

Output:

```
200 OK application/json
map[id:7 ok:true]
```

Method-and-wildcard routing, JSON codec, production HTTP server, HTTP client, and
an in-process test server — all stdlib. The Express equivalent of this file
installs a dependency tree; here `go.mod` still has no `require` block.

The stdlib is also covered by Go's compatibility promise: code written against
Go 1.0 compiles today. Third-party code carries no such guarantee, which changes
the cost calculus of a dependency.

## A little copying beats a little dependency

The Go proverb, and it is load-bearing. A six-line helper you own has no version,
no supply chain, no transitive tree, and no upstream that can be deleted.

```go
package main

import (
  "fmt"
  "slices"
  "strings"
)

func contains(xs []string, want string) bool {
  for _, x := range xs {
    if x == want {
      return true
    }
  }
  return false
}

func main() {
  xs := []string{"a", "b", "c"}
  fmt.Println(contains(xs, "b"))
  fmt.Println(slices.Contains(xs, "b"))
  fmt.Println(strings.Contains("abc", "b"))
}
```

Output:

```
true
true
true
```

In 2026 `slices.Contains` exists, so this particular copy is unnecessary — but for
a decade it did not, and the community answer was the five-line loop rather than
a `lodash.includes` package. That instinct is why a production Go service
typically has five to fifteen direct dependencies, and why this course's full
stack is four: chi, pgx, golang-migrate, and a JWT library.

The proverb is not an argument against all dependencies. Do not reimplement TLS,
a Postgres wire protocol, or JWT signing. It is an argument against the
*trivial* dependency, and the line is roughly: if you could write it correctly in
an afternoon and test it fully, write it.

## Quick Reference

| Go's answer | Instead of |
|---|---|
| `v, err := f()` | `try` / `catch` / `throw` |
| `panic` for programmer error only | exceptions as control flow |
| the zero value | `null`, `undefined`, `None`, `Option<T>` |
| `v, ok := m[k]` | a `KeyError` or `undefined` |
| struct embedding | `extends`, `super`, abstract classes |
| implicit interface satisfaction | `implements`, declared conformance |
| consumer-defined one-method interfaces | large pre-declared interfaces |
| `go f()` on any function | `async` / `await`, coloured functions |
| channels and `sync` | callbacks, promises, event loops |
| `defer` | destructors, `with`, `finally`, RAII |
| explicit `float64(i)` | implicit numeric promotion |
| `if n != 0` | truthiness |
| gofmt | a formatter config file |
| stdlib + a handful of deps | a large dependency tree |
| one static binary | a runtime plus a dependency directory |

| Trade-off | What you pay |
|---|---|
| errors as values | three lines per fallible call |
| no inheritance | some duplication where a base class would serve |
| zero values | `0` and `""` are ambiguous with "unset" |
| goroutines | nondeterministic interleaving; you own the data races |
| sparse generics | occasional near-duplicate functions per type |
| static binary | ~5 MB floor and a GC you cannot remove |

## Common Mistakes

**Wrapping everything in `recover` to emulate a try/catch.** `recover` is for
containing a panic at a process boundary; recovering a panic from a library you
call is a sign that library is misbehaving.

**Writing `*string` or `*int` everywhere to get nullability.** The zero value is
usually the right answer; reach for a pointer only when "unset" must be
distinguishable from zero.

**Reaching for embedding to build a type hierarchy.** Promotion is static
delegation, not polymorphism — `[]Base` will never hold your derived type; use an
interface.

**Declaring a large interface up front and then implementing it.** Define the
interface in the consumer, with only the methods that consumer calls, after the
concrete type exists.

**Ignoring the race detector because the program "works".** Unsynchronised access
is undefined behaviour, not a slow path; `go test -race` is the only way to know.

**Importing a package for something you could write in five lines.** Each
dependency is a version to track, a supply chain to trust, and a transitive tree
you did not choose.

## Exercises

1. Rewrite a function that currently returns `(T, error)` so it panics instead,
   then write the `recover` wrapper a caller would need. Compare the two call
   sites and say which you would ship.
2. Define a `Config` struct where "unset" must be distinguishable from the zero
   value for three fields. Solve it twice — pointers, and a parallel `isSet` map
   — and argue for one.
3. Take a function with an `*os.File` parameter and widen it to `io.Writer`
   without changing its body. Then call it with `os.Stdout`, a `bytes.Buffer`, an
   `httptest.ResponseRecorder`, and a type you write yourself.
4. Model "an Admin is a User with extra permissions" twice: with embedding, and
   with an interface. Then write a function taking a slice of both kinds and
   explain which model made it possible.
5. Write a sequential function that makes three 100 ms calls, measure it, then
   make it concurrent without changing the called function's signature. Run it
   under `-race`.
6. Count your current project's direct and transitive dependencies with
   `go list -m all | wc -l`, then pick the smallest one and estimate the lines
   you would write to replace it.
7. Build a program importing `net/http` for `linux/amd64`, strip it with
   `-ldflags="-s -w"`, and put it in a `FROM scratch` Dockerfile. Report the
   image size.

---

[← 1.5 gofmt, go vet, and the Editor Setup](05-tooling.md) · [Course Index](../README.md) · [2.1 Variables, Zero Values, and Declaration Forms →](../02-syntax-basics/01-variables.md)
