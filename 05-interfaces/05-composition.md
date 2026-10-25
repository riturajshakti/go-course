[← 5.4 The Empty Interface and any](04-empty-interface.md) · [Course Index](../README.md) · [5.6 Standard Library Interfaces Worth Knowing →](06-stdlib-interfaces.md)

# 5.5 — Interface Composition and Small Interfaces

## What You'll Learn

- Embedding one interface in another splices in its methods; that is how `io.ReadWriter` is built
- Duplicate methods across embedded interfaces are legal since Go 1.14 — and were a compile error before
- Small interfaces (1-3 methods, named `-er`) are what make a function testable against six different sinks
- "Accept interfaces, return structs", with a before/after where the fat interface blocks the test
- Embedding an interface in a *struct* gives partial override — and a nil-field panic if you are careless

## Coming From Another Language

| Other | Go |
|---|---|
| Java `interface A extends B, C` | `interface A { B; C }` — embedding, no `extends` |
| TS `interface A extends B` | the same effect; Go uses embedding syntax |
| Rust `trait A: B` (supertrait bound) | embedding, but with no separate `impl` to write |
| Java fat service interfaces, 10+ methods | culturally wrong in Go; 1-3 methods is the norm |
| Dependency injection frameworks | a function parameter of interface type |

## Embedding builds bigger interfaces

Here are the real `io` definitions, retyped so this is a runnable program.

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "os"
  "strings"
)

type Reader interface {
  Read(p []byte) (n int, err error)
}

type Writer interface {
  Write(p []byte) (n int, err error)
}

type Closer interface {
  Close() error
}

type ReadWriter interface {
  Reader
  Writer
}

type ReadWriteCloser interface {
  Reader
  Writer
  Closer
}

func main() {
  var rw ReadWriter = &bytes.Buffer{}
  fmt.Fprint(rw, "composed")
  b, _ := io.ReadAll(rw)
  fmt.Println(string(b))

  var r Reader = rw
  var w Writer = rw
  fmt.Printf("%T %T\n", r, w)

  f, _ := os.CreateTemp("", "demo")
  defer os.Remove(f.Name())
  var rwc ReadWriteCloser = f
  fmt.Fprint(rwc, "file")
  rwc.Close()

  var sr Reader = strings.NewReader("x")
  _, isRW := sr.(ReadWriter)
  fmt.Println("strings.Reader is a ReadWriter:", isRW)
}
```

Output:

```
composed
*bytes.Buffer *bytes.Buffer
strings.Reader is a ReadWriter: false
```

Embedding is pure method-set union — `ReadWriter`'s method set is exactly `{Read, Write}`,
indistinguishable from writing both methods out longhand. Consequences:

- A `ReadWriter` value assigns to a `Reader` or `Writer` variable with no conversion, because
  the target's method set is a subset. The reverse needs an assertion, which fails for
  `*strings.Reader` since it has no `Write`.
- `*bytes.Buffer` was never declared to satisfy any of these. It has the methods.
- `*os.File` satisfies `ReadWriteCloser` and about a dozen more.

Idiomatic: compose from existing interfaces rather than restating methods, so the relationship
is visible in the declaration.

## Duplicate methods are allowed since Go 1.14

Embed two interfaces that both declare `Close()`, and the overlap collapses. You can even embed
the same interface twice.

```go
package main

import "fmt"

type Reader interface{ Read() string }
type Writer interface{ Write(string) }

type ReadCloser interface {
  Reader
  Close() error
}

type WriteCloser interface {
  Writer
  Close() error
}

type ReadWriteCloser interface {
  ReadCloser
  WriteCloser
}

type Everything interface {
  Reader
  Reader
  ReadCloser
}

type File struct{ s string }

func (f *File) Read() string   { return f.s }
func (f *File) Write(s string) { f.s += s }
func (f *File) Close() error   { return nil }

func main() {
  var x ReadWriteCloser = &File{}
  x.Write("a")
  fmt.Println(x.Read(), x.Close())

  var e Everything = &File{s: "z"}
  fmt.Println(e.Read())
}
```

Output:

```
a <nil>
z
```

`ReadWriteCloser` gets `Close` from both parents; `Everything` gets `Read` three times. Both
have one of each in their method set.

Set `go 1.13` in `go.mod` and the same file is rejected:

```
# example.com/scratch
./main.go:20:2: duplicate method Close
	./main.go:19:2: other declaration of method Close
./main.go:25:2: duplicate method Read
	./main.go:24:2: other declaration of method Read
./main.go:26:2: duplicate method Read
	./main.go:24:2: other declaration of method Read
```

This is worth knowing because it explains the shape of pre-1.14 stdlib code: interfaces were
hand-flattened to dodge the overlap. Duplicates must have **identical** signatures — same
parameter and result types — or it is still an error.

## Composing a small interface out of large ones

Composition is not only for widening. Pick exactly the two capabilities a function needs:

```go
package main

import (
  "fmt"
  "io"
  "os"
  "strings"
)

type SeekReader interface {
  io.Reader
  io.Seeker
}

func rewindAndRead(sr SeekReader) (string, string) {
  a, _ := io.ReadAll(sr)
  sr.Seek(0, io.SeekStart)
  b, _ := io.ReadAll(sr)
  return string(a), string(b)
}

func main() {
  fmt.Println(rewindAndRead(strings.NewReader("abc")))

  f, _ := os.CreateTemp("", "demo")
  defer os.Remove(f.Name())
  f.WriteString("file data")
  f.Seek(0, io.SeekStart)
  fmt.Println(rewindAndRead(f))

  var fits SeekReader
  fits = strings.NewReader("")
  fits = f
  _, ok := any(fits).(io.ReadWriteSeeker)
  fmt.Println("the *os.File also satisfies ReadWriteSeeker:", ok)
}
```

Output:

```
abc abc
file data file data
the *os.File also satisfies ReadWriteSeeker: true
```

`rewindAndRead` accepts an `*os.File` without demanding `Write` or `Close`, so the signature
documents precisely what it touches — and a test can pass a `*strings.Reader`.

## Why small interfaces: one function, six sinks

The whole argument for `io.Writer` having exactly one method. `report` is written once, never
modified, and works with everything.

```go
package main

import (
  "bytes"
  "compress/gzip"
  "crypto/sha256"
  "encoding/hex"
  "fmt"
  "io"
  "net/http/httptest"
  "os"
)

func report(w io.Writer, rows [][2]string) error {
  if _, err := fmt.Fprintln(w, "name,value"); err != nil {
    return err
  }
  for _, r := range rows {
    if _, err := fmt.Fprintf(w, "%s,%s\n", r[0], r[1]); err != nil {
      return err
    }
  }
  return nil
}

var rows = [][2]string{{"cpu", "91"}, {"mem", "42"}}

func main() {
  var buf bytes.Buffer
  report(&buf, rows)
  fmt.Printf("bytes.Buffer: %q\n", buf.String())

  fmt.Print("os.Stdout:    ")
  report(os.Stdout, rows[:1])

  rec := httptest.NewRecorder()
  report(rec, rows)
  fmt.Printf("ResponseWriter: %d %q\n", rec.Code, rec.Body.String())

  h := sha256.New()
  report(h, rows)
  fmt.Println("sha256:       ", hex.EncodeToString(h.Sum(nil))[:16])

  var gz bytes.Buffer
  zw := gzip.NewWriter(&gz)
  report(zw, rows)
  zw.Close()
  fmt.Println("gzip bytes:   ", gz.Len())

  fmt.Println("discard:      ", report(io.Discard, rows))
}
```

Output:

```
bytes.Buffer: "name,value\ncpu,91\nmem,42\n"
os.Stdout:    name,value
cpu,91
ResponseWriter: 200 "name,value\ncpu,91\nmem,42\n"
sha256:        b518dc5d8abce19e
gzip bytes:    50
discard:       <nil>
```

An in-memory buffer, a terminal, an HTTP response, a hash, a compressor, and a black hole. The
`bytes.Buffer` case is the test: assert on a string, no mocking framework, no interface you had
to define. That property is a direct consequence of `Writer` having **one** method — every
candidate sink can plausibly implement it.

Invert it: an interface with eight methods has few possible implementations, so it buys you
nothing at the call site and costs you a 200-line fake in the test.

The naming convention follows from the size. One method `Read` gives `Reader`; `Write` gives
`Writer`. If you cannot name your interface after its single verb, it is probably too big.

## Accept interfaces, return structs

The rule in two halves:

- **Accept interfaces** in function and method parameters — the narrowest one that works. The
  caller then has the widest possible choice of what to pass.
- **Return structs** (concrete types) from constructors. The caller keeps every method and can
  satisfy any interface they like. Returning an interface throws away capability for no gain.

Here is the cost of getting the first half wrong. `greetBefore` needs only `Get`, but asks for
the whole `Store`:

```go
package main

import (
  "errors"
  "fmt"
)

type Store interface {
  Get(id string) (string, error)
  Put(id, v string) error
  Delete(id string) error
  List() ([]string, error)
  Close() error
}

type MemStore struct{ m map[string]string }

func NewMemStore() *MemStore { return &MemStore{m: map[string]string{}} }

func (s *MemStore) Get(id string) (string, error) {
  v, ok := s.m[id]
  if !ok {
    return "", errors.New("not found")
  }
  return v, nil
}
func (s *MemStore) Put(id, v string) error { s.m[id] = v; return nil }
func (s *MemStore) Delete(id string) error { delete(s.m, id); return nil }
func (s *MemStore) List() ([]string, error) {
  out := make([]string, 0, len(s.m))
  for k := range s.m {
    out = append(out, k)
  }
  return out, nil
}
func (s *MemStore) Close() error { return nil }

type Getter interface {
  Get(id string) (string, error)
}

func greetBefore(s Store, id string) string {
  v, err := s.Get(id)
  if err != nil {
    return "unknown"
  }
  return "hello " + v
}

func greetAfter(g Getter, id string) string {
  v, err := g.Get(id)
  if err != nil {
    return "unknown"
  }
  return "hello " + v
}

type fakeGetter struct{}

func (fakeGetter) Get(id string) (string, error) { return "FAKE-" + id, nil }

func main() {
  s := NewMemStore()
  s.Put("1", "ada")
  fmt.Println(greetBefore(s, "1"), greetAfter(s, "1"))

  fmt.Println(greetAfter(fakeGetter{}, "9"))
}
```

Output:

```
hello ada hello ada
hello FAKE-9
```

Both work on the real store. But the three-line `fakeGetter` can only be passed to
`greetAfter`. Try the other one:

```go
  fmt.Println(greetBefore(fakeGetter{}, "9"))
```

This does not compile. Output:

```
main.go:1:3: expected 'package', found fmt
```

To test `greetBefore` you must implement `Put`, `Delete`, `List`, and `Close` — four methods it
never calls. That is the tax on a fat interface, and it is why `Getter` lives next to
`greetAfter` and not next to `MemStore`.

Note also that `NewMemStore` returns `*MemStore`, not `Store`. Callers needing `Getter` get it
for free; callers needing a method you later add get that for free too.

Over-abstraction warning, one line: do not define an interface that has exactly one
implementation and no test double — return the struct and add the interface when a second
caller actually needs it.

## Embedding an interface in a struct

A struct can embed an interface field. The struct then satisfies that interface by delegation,
and you override selectively. This is the decorator pattern with no boilerplate.

```go
package main

import (
  "fmt"
  "io"
  "strings"
)

type countingReader struct {
  io.Reader
  n int
}

func (c *countingReader) Read(p []byte) (int, error) {
  n, err := c.Reader.Read(p)
  c.n += n
  return n, err
}

func main() {
  cr := &countingReader{Reader: strings.NewReader("hello world")}
  b, _ := io.ReadAll(cr)
  fmt.Println(string(b), cr.n)
}
```

Output:

```
hello world 11
```

`countingReader` declares one method and gets the rest of whatever it wraps. The canonical
use is HTTP middleware that needs to observe the status code:

```go
package main

import (
  "fmt"
  "net/http"
  "net/http/httptest"
)

type statusRecorder struct {
  http.ResponseWriter
  status int
}

func (s *statusRecorder) WriteHeader(code int) {
  s.status = code
  s.ResponseWriter.WriteHeader(code)
}

func middleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    rec := &statusRecorder{ResponseWriter: w, status: 200}
    next.ServeHTTP(rec, r)
    fmt.Println("observed status:", rec.status)
  })
}

func main() {
  h := middleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("X-Demo", "1")
    w.WriteHeader(404)
    fmt.Fprint(w, "nope")
  }))

  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))
  fmt.Println(rec.Code, rec.Body.String(), rec.Header().Get("X-Demo"))
}
```

Output:

```
observed status: 404
404 nope 1
```

`Header` and `Write` are promoted from the embedded `http.ResponseWriter`; only `WriteHeader`
is intercepted. 16.6 builds on this.

The sharp edge: delegation is a *runtime* lookup through the field, so the compiler is satisfied
even when the field is nil.

```go
package main

import (
  "fmt"
  "io"
)

type halfBroken struct {
  io.ReadWriter
}

func main() {
  var rw io.ReadWriter = &halfBroken{}
  fmt.Println("compiles fine:", rw != nil)
  _, err := rw.Write([]byte("x"))
  fmt.Println(err)
}
```

This program panics on purpose. Output:

```
compiles fine: true
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x20 pc=0x1040fb5b4]

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:15 +0x84
exit status 2
```

An embedded interface puts every one of its methods into the struct's method set *immediately*,
whether or not anything is behind them. So `halfBroken` satisfies `io.ReadWriter` with zero
methods written and crashes on the first call. Always set the field in a constructor, and
embed an interface only when you mean to delegate most of it — otherwise embed a concrete type
(4.5) or declare a named field.

## Quick Reference

| Form | Effect |
|---|---|
| `interface { A; B }` | method-set union of `A` and `B` |
| `interface { A; M() }` | `A`'s methods plus `M` |
| `interface { A; A }` | legal since Go 1.14; one copy in the set |
| duplicate method, differing signature | compile error, all versions |
| `struct { A; ... }` where `A` is an interface | struct satisfies `A` by delegating to the field |
| overriding method on the struct | shadows the promoted one; call `s.A.M()` to reach the original |

| `io` composite | Equals |
|---|---|
| `io.ReadWriter` | `Reader` + `Writer` |
| `io.ReadCloser` | `Reader` + `Closer` |
| `io.WriteCloser` | `Writer` + `Closer` |
| `io.ReadWriteCloser` | `Reader` + `Writer` + `Closer` |
| `io.ReadSeeker` | `Reader` + `Seeker` |
| `io.ReadWriteSeeker` | `Reader` + `Writer` + `Seeker` |

| Design rule | Why |
|---|---|
| 1-3 methods per interface | more candidate implementations, cheaper fakes |
| Name it `-er` after its verb | if you cannot, it is too big |
| Declare it in the consuming package | the consumer knows what it needs |
| Accept interfaces | widest choice for the caller |
| Return structs | caller keeps every method |
| No interface until the second implementation | abstraction you cannot yet justify is guesswork |

## Common Mistakes

**Declaring a fat interface mirroring every method of one struct.** Split it per call site;
nobody can write a fake for it otherwise.

**Returning an interface from a constructor.** Return the concrete type so callers keep methods
the interface omits.

**Embedding an interface in a struct and leaving the field nil.** The struct satisfies the
interface at compile time and panics on the first delegated call; set it in a constructor.

**Embedding two interfaces with the same method name but different signatures.** That is still
an error; only identical signatures merge.

**Defining an interface in the package that implements it.** Put it in the consumer, and skip
it entirely until a second implementation exists.

## Exercises

1. Define `Reader`, `Writer`, `Closer`, and every two- and three-way composite. Write a
   function per composite and pass a `*os.File` to all of them unchanged.
2. Write `type teeWriter struct { io.Writer; log io.Writer }` whose `Write` copies to both.
   Wrap `os.Stdout` and a `bytes.Buffer`, verify the buffer, and then show what happens when
   `log` is nil.
3. Take a 6-method `UserRepository` interface and refactor it into the smallest interfaces your
   call sites actually need. Count the methods in the test fakes before and after.
4. Write a `statusRecorder` that also counts bytes written, and confirm `Flush` still reaches
   the underlying writer by asserting your type to `http.Flusher`. Explain why that assertion
   fails and how to fix it.
5. Embed `io.ReadWriter` in a struct, assign it to an `io.ReadWriter` variable without setting
   the field, and capture the panic. Then make the zero value safe by overriding both methods.

---

[← 5.4 The Empty Interface and any](04-empty-interface.md) · [Course Index](../README.md) · [5.6 Standard Library Interfaces Worth Knowing →](06-stdlib-interfaces.md)
