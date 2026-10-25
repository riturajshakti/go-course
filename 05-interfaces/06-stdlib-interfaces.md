[← 5.5 Interface Composition and Small Interfaces](05-composition.md) · [Course Index](../README.md) · [6.1 string, []byte, and []rune →](../06-strings/01-string-bytes-runes.md)

# 5.6 — Standard Library Interfaces Worth Knowing

## What You'll Learn

- The directory: every interface in `io`, `fmt`, `sort`, `encoding`, `net/http`, and friends worth recognising
- Implementing one makes your type work with code you have never seen — the whole payoff of implicit satisfaction
- `String()` built with `%v` on its own receiver overflows the stack; `go vet` catches it
- Which interface to reach for when you want your type to print, sort, marshal, hash, or parse a flag

## Coming From Another Language

| Other | Go |
|---|---|
| Python dunder protocol (`__str__`, `__iter__`, `__lt__`) | the closest analogue: named interfaces instead of magic names |
| JS `toString`, `Symbol.iterator`, `toJSON` | `String()`, `iter.Seq`, `MarshalJSON()` |
| Java `Comparable`, `Iterable`, `Serializable` | `sort.Interface`, `iter.Seq`, `encoding/json` marshalers |
| Rust `Display`, `Ord`, `Serialize` | `fmt.Stringer`, `sort.Interface`, `json.Marshaler` |

The difference: these are ordinary interfaces in ordinary packages. Nothing registers your type,
and you can satisfy one without importing the package that declares it.

## The io group

The most important interfaces in Go. Everything streaming is built on the first two.

| Interface | Method(s) | Satisfied by | Why you care |
|---|---|---|---|
| `io.Reader` | `Read(p []byte) (int, error)` | `*os.File`, `*strings.Reader`, `*bytes.Buffer`, `*bytes.Reader`, `net.Conn`, `http.Request.Body`, `*gzip.Reader` | the universal input; `io.Copy`, `bufio.Scanner`, `json.Decoder` all take one |
| `io.Writer` | `Write(p []byte) (int, error)` | `*os.File`, `*bytes.Buffer`, `*strings.Builder`, `net.Conn`, `http.ResponseWriter`, `hash.Hash`, `io.Discard` | the universal output; `fmt.Fprintf` target |
| `io.Closer` | `Close() error` | files, conns, response bodies, `*gzip.Writer` | pair with `defer`; a `Close` error on a writer is a real error |
| `io.ReadWriter` | `Reader`+`Writer` | `*bytes.Buffer`, `*os.File`, `net.Conn` | bidirectional streams |
| `io.ReadCloser` | `Reader`+`Closer` | `http.Request.Body`, `http.Response.Body` | wrap with `io.NopCloser` to make a `Reader` fit |
| `io.WriteCloser` | `Writer`+`Closer` | `*gzip.Writer`, pipe write ends | flush-on-close semantics |
| `io.ReadWriteCloser` | all three | `*os.File`, `net.Conn` | the full stream |
| `io.Seeker` | `Seek(off int64, whence int) (int64, error)` | `*os.File`, `*bytes.Reader`, `*strings.Reader` | rewind and random access; `io.SeekStart/Current/End` |
| `io.ReadSeeker` | `Reader`+`Seeker` | the above | `http.ServeContent` needs it for range requests |
| `io.ReadWriteSeeker` | `Reader`+`Writer`+`Seeker` | `*os.File` | editing a file in place |
| `io.ReaderAt` | `ReadAt(p []byte, off int64) (int, error)` | `*os.File`, `*bytes.Reader` | concurrent positional reads, no shared cursor; `zip.NewReader` needs it |
| `io.WriterAt` | `WriteAt(p []byte, off int64) (int, error)` | `*os.File` | parallel chunked downloads |
| `io.ReaderFrom` | `ReadFrom(r Reader) (int64, error)` | `*bytes.Buffer`, `*os.File`, `*net.TCPConn` | the fast path `io.Copy` probes for |
| `io.WriterTo` | `WriteTo(w Writer) (int64, error)` | `*bytes.Buffer`, `*strings.Reader` | the other fast path |
| `io.ByteReader` | `ReadByte() (byte, error)` | `*bufio.Reader`, `*strings.Reader`, `*bytes.Buffer` | required by `binary.ReadVarint` |
| `io.ByteWriter` | `WriteByte(byte) error` | `*bufio.Writer`, `*bytes.Buffer`, `*strings.Builder` | avoids a one-byte slice |
| `io.ByteScanner` | `ByteReader`+`UnreadByte() error` | `*bufio.Reader`, `*bytes.Reader` | one-byte lookahead for parsers |
| `io.RuneReader` | `ReadRune() (rune, int, error)` | `*bufio.Reader`, `*strings.Reader` | `regexp` uses it to run over a stream |
| `io.RuneScanner` | `RuneReader`+`UnreadRune() error` | `*bufio.Reader`, `*strings.Reader` | rune-level lookahead |
| `io.StringWriter` | `WriteString(string) (int, error)` | `*bytes.Buffer`, `*strings.Builder`, `*bufio.Writer` | `io.WriteString` probes for it to skip a `[]byte` conversion |

Implement `Write` and the whole `fmt` family starts working on your type:

```go
package main

import (
  "bufio"
  "fmt"
  "io"
  "strings"
)

type prefixWriter struct {
  prefix string
  out    io.Writer
}

func (p prefixWriter) Write(b []byte) (int, error) {
  for _, line := range strings.SplitAfter(string(b), "\n") {
    if line == "" {
      continue
    }
    if _, err := io.WriteString(p.out, p.prefix+line); err != nil {
      return 0, err
    }
  }
  return len(b), nil
}

type repeatReader struct {
  s    string
  left int
}

func (r *repeatReader) Read(b []byte) (int, error) {
  if r.left == 0 {
    return 0, io.EOF
  }
  r.left--
  return copy(b, r.s), nil
}

func main() {
  var sb strings.Builder
  w := prefixWriter{prefix: "[log] ", out: &sb}
  fmt.Fprintf(w, "hello %s\n", "world")
  fmt.Fprintln(w, "second")
  fmt.Print(sb.String())

  n, err := io.Copy(io.Discard, &repeatReader{s: "ab\n", left: 3})
  fmt.Println(n, err)

  sc := bufio.NewScanner(&repeatReader{s: "line\n", left: 3})
  count := 0
  for sc.Scan() {
    count++
    fmt.Println(count, sc.Text())
  }
  fmt.Println(sc.Err())

  all, _ := io.ReadAll(&repeatReader{s: "x", left: 4})
  fmt.Printf("%q\n", all)
}
```

Output:

```
[log] hello world
[log] second
9 <nil>
1 line
2 line
3 line
<nil>
"xxxx"
```

Two methods, and `fmt.Fprintf`, `io.Copy`, `io.ReadAll`, and `bufio.Scanner` all work. None of
those were written with your types in mind.

Contract notes that cause real bugs: `Read` may return `0 < n` **and** `io.EOF` together, so
process `n` bytes before checking the error; returning `(0, nil)` repeatedly is treated as
broken; and `Write` must either consume all of `p` or return an error.

## The fmt group

| Interface | Method(s) | Who calls it | Notes |
|---|---|---|---|
| `fmt.Stringer` | `String() string` | every `fmt` verb that prints a value (`%v`, `%s`, `Println`), `log`, `slog` | the most implemented interface in Go |
| `fmt.GoStringer` | `GoString() string` | `%#v` only | Go-syntax representation |
| `fmt.Formatter` | `Format(f State, verb rune)` | `fmt` for *every* verb, overriding everything | full control; how `github.com/pkg/errors` printed stack traces |
| `fmt.Scanner` | `Scan(s ScanState, verb rune) error` | `fmt.Scan`/`Fscan`/`Sscan` | custom input parsing; rarely used |
| `error` | `Error() string` | `fmt` prefers it over `Stringer` | see the error group below |

```go
package main

import "fmt"

type Point struct{ X, Y int }

func (p Point) String() string   { return fmt.Sprintf("(%d,%d)", p.X, p.Y) }
func (p Point) GoString() string { return fmt.Sprintf("Point{X:%d, Y:%d}", p.X, p.Y) }

type Plain struct{ X, Y int }

func main() {
  p := Point{1, 2}
  fmt.Println(p)
  fmt.Printf("%v|%s|%d|%#v|%q\n", p, p, p, p, p)
  fmt.Println([]Point{{1, 2}, {3, 4}})
  fmt.Println(map[string]Point{"a": {1, 2}})
  fmt.Println(&p)

  q := Plain{1, 2}
  fmt.Printf("%v|%+v|%#v\n", q, q, q)
}
```

Output:

```
(1,2)
(1,2)|(1,2)|{1 2}|Point{X:1, Y:2}|"(1,2)"
[(1,2) (3,4)]
map[a:(1,2)]
(1,2)
{1 2}|{X:1 Y:2}|main.Plain{X:1, Y:2}
```

Three things worth noting. `%d` ignores `String()` — numeric verbs go to the fields, printing
`{1 2}`. The method is used **inside** slices and maps automatically. And `&p` prints `(1,2)`
too, because `*Point`'s method set includes the value-receiver `String`.

A value receiver on `String()` is near-always right: with a pointer receiver, `fmt.Println(p)`
on a `Point` *value* silently prints `{1 2}` instead.

### The Stringer infinite-recursion trap

The single most common `Stringer` mistake: formatting the receiver with `%v` inside `String()`.

```go
package main

import "fmt"

type Bad struct{ X int }

func (b Bad) String() string { return fmt.Sprintf("Bad %v", b) }

func main() {
  fmt.Println(Bad{1})
}
```

`go vet` catches it:

```
main.go:7:56: fmt.Sprintf format %v with arg b causes recursive (main.Bad).String method call
```

Run it anyway and the program dies. `%v` on `b` calls `String()`, which calls `%v` on `b`:

This program crashes on purpose. Output:

```
runtime: goroutine stack exceeds 1000000000-byte limit
runtime: sp=0x725e21de0350 stack=[0x725e21de0000, 0x725e41de0000]
fatal error: stack overflow

runtime stack:
runtime.throw({0x102c3e7a7?, 0x102bd1304?})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/panic.go:1243 +0x38 fp=0x16d25e730 sp=0x16d25e700 pc=0x102c10a68
runtime.newstack()
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/stack.go:1207 +0x480 fp=0x16d25e860 sp=0x16d25e730 pc=0x102bf7480
runtime.morestack()
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/asm_arm64.s:487 +0x70 fp=0x16d25e860 sp=0x16d25e860 pc=0x102c145a0

goroutine 1 gp=0x725e01c721e0 m=0 mp=0x102d16900 [running]:
fmt.newPrinter()
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/fmt/print.go:145 +0x98 fp=0x725e21de0350 sp=0x725e21de0350 pc=0x102c368b8
fmt.Sprintf({0x102c3d308, 0x6}, {0x725e21de03e8, 0x1, 0x1})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/fmt/print.go:230 +0x30 fp=0x725e21de03b0 sp=0x725e21de0350 pc=0x102c36ab0
main.Bad.String(...)
	/tmp/scratch/main.go:7
main.(*Bad).String(0x0?)
	<autogenerated>:1 +0x50 fp=0x725e21de0400 sp=0x725e21de03b0 pc=0x102c3cd10
fmt.(*pp).handleMethods(0x725e07611080, {0x102cffb88, 0x102c46728}, {0x0?, 0x0?, 0x102bb5928?}, 0x0?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/fmt/print.go:666 +0x288 fp=0x725e21de05f0 sp=0x725e21de0400 pc=0x102c38f08
fmt.(*pp).printArg(0x725e07611080, {0x102cffb88, 0x102c46728}, 0x76)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/fmt/print.go:746 +0x488 fp=0x725e21de06a0 sp=0x725e21de05f0 pc=0x102c39658
```

*(The `fmt.newPrinter` / `Sprintf` / `String` / `handleMethods` / `printArg` cycle then repeats
for tens of thousands of frames. I have truncated the repeated frames and the trailing GC
goroutines; the full dump is about 100 MB.)*

Two fixes: print the **fields**, or convert to a local type that has no `String` method.

```go
package main

import "fmt"

type Bad struct{ X int }

func (b Bad) String() string { return fmt.Sprintf("Bad %d", b.X) }

type Alias struct{ X int }

func (a Alias) String() string {
  type plain Alias
  return fmt.Sprintf("Alias%+v", plain(a))
}

func main() {
  fmt.Println(Bad{1}, Alias{2})
}
```

Output:

```
Bad 1 Alias{X:2}
```

`type plain Alias` is a *definition*, not an alias, so `plain` has an empty method set — the
trick for printing all fields of a `Stringer` without recursing. Note `%+v` on a named struct
type prints the type name too.

## The error group

| Interface / convention | Signature | Used by | Notes |
|---|---|---|---|
| `error` | `Error() string` | everything | the only one built into the language |
| `Unwrap() error` | — | `errors.Is`, `errors.As`, `errors.Unwrap` | makes your type a link in the chain |
| `Unwrap() []error` | — | `errors.Is`/`As` | multiple causes; what `errors.Join` returns |
| `Is(target error) bool` | — | `errors.Is` | custom matching when `==` is wrong |
| `As(target any) bool` | — | `errors.As` | custom extraction; rarely needed |

These are not declared interfaces you can name — `errors.Is` asserts to an anonymous
`interface{ Is(error) bool }`. Implementing the method is the whole opt-in.

```go
package main

import (
  "errors"
  "fmt"
)

type QueryError struct {
  Query string
  Err   error
}

func (e *QueryError) Error() string { return e.Query + ": " + e.Err.Error() }
func (e *QueryError) Unwrap() error { return e.Err }

type HTTPError struct{ Code int }

func (e HTTPError) Error() string { return fmt.Sprintf("http %d", e.Code) }
func (e HTTPError) Is(target error) bool {
  t, ok := target.(HTTPError)
  return ok && (t.Code == e.Code || (t.Code == 0 && e.Code >= 400))
}

var ErrNoRows = errors.New("no rows")

func main() {
  err := &QueryError{Query: "SELECT 1", Err: ErrNoRows}
  fmt.Println(err)
  fmt.Println(errors.Is(err, ErrNoRows), errors.Unwrap(err) == ErrNoRows)

  var qe *QueryError
  fmt.Println(errors.As(fmt.Errorf("outer: %w", err), &qe), qe.Query)

  fmt.Println(errors.Is(HTTPError{404}, HTTPError{404}))
  fmt.Println(errors.Is(HTTPError{503}, HTTPError{0}))
  fmt.Println(errors.Is(HTTPError{200}, HTTPError{0}))

  joined := errors.Join(ErrNoRows, HTTPError{500})
  fmt.Println(errors.Is(joined, ErrNoRows), errors.Is(joined, HTTPError{500}))
}
```

Output:

```
SELECT 1: no rows
true true
true SELECT 1
true
true
false
true true
```

The custom `Is` lets `HTTPError{0}` act as a wildcard for any 4xx/5xx. Module 08 is the full
treatment.

## sort.Interface

| Interface | Methods | Notes |
|---|---|---|
| `sort.Interface` | `Len() int`, `Less(i, j int) bool`, `Swap(i, j int)` | the pre-generics sorting protocol |
| `sort.IntSlice`, `sort.StringSlice`, `sort.Float64Slice` | — | named slice types that already implement it |

```go
package main

import (
  "fmt"
  "sort"
)

type Person struct {
  Name string
  Age  int
}

type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

func main() {
  people := []Person{{"ada", 36}, {"bob", 22}, {"cy", 51}}

  sort.Sort(ByAge(people))
  fmt.Println(people)

  sort.Sort(sort.Reverse(ByAge(people)))
  fmt.Println(people)

  fmt.Println(sort.IsSorted(ByAge(people)))

  sort.Slice(people, func(i, j int) bool { return people[i].Name < people[j].Name })
  fmt.Println(people, sort.SliceIsSorted(people, func(i, j int) bool { return people[i].Name < people[j].Name }))

  ss := sort.StringSlice{"c", "a", "b"}
  ss.Sort()
  fmt.Println(ss, ss.Len())
}
```

Output:

```
[{bob 22} {ada 36} {cy 51}]
[{cy 51} {ada 36} {bob 22}]
false
[{ada 36} {bob 22} {cy 51}] true
[a b c] 3
```

`sort.Reverse` is a lovely piece of design: it takes an `Interface` and returns an `Interface`
whose `Less` has the arguments flipped — a decorator built on an interface embedded in a struct
(5.5).

Idiomatic in new code: `slices.SortFunc`, with `sort.Slice` as the older alternative. Implement
`sort.Interface` when the ordering *is* a property of the type, or when an API demands it.
7.10 covers the modern options.

## The encoding group

| Interface | Method | Called by |
|---|---|---|
| `json.Marshaler` | `MarshalJSON() ([]byte, error)` | `encoding/json` on encode |
| `json.Unmarshaler` | `UnmarshalJSON([]byte) error` | `encoding/json` on decode |
| `encoding.TextMarshaler` | `MarshalText() ([]byte, error)` | `json`, `xml`, `yaml` libs, `slog`, map keys |
| `encoding.TextUnmarshaler` | `UnmarshalText([]byte) error` | the same, plus `flag` via `TextVar` |
| `encoding.BinaryMarshaler` | `MarshalBinary() ([]byte, error)` | `encoding/gob`, caches |
| `encoding.BinaryUnmarshaler` | `UnmarshalBinary([]byte) error` | `encoding/gob` |
| `xml.Marshaler` / `Unmarshaler` | `MarshalXML` / `UnmarshalXML` | `encoding/xml` |

`MarshalText` is the better default: one implementation and `json`, `xml`, `slog`, and map-key
encoding all pick it up. `MarshalJSON` wins when both exist.

```go
package main

import (
  "encoding/json"
  "fmt"
  "strings"
  "time"
)

type Status int

const (
  Active Status = iota
  Suspended
)

var names = map[Status]string{Active: "active", Suspended: "suspended"}

func (s Status) MarshalJSON() ([]byte, error) { return json.Marshal(names[s]) }

func (s *Status) UnmarshalJSON(b []byte) error {
  var str string
  if err := json.Unmarshal(b, &str); err != nil {
    return err
  }
  for k, v := range names {
    if v == str {
      *s = k
      return nil
    }
  }
  return fmt.Errorf("unknown status %q", str)
}

type Date struct{ t time.Time }

func (d Date) MarshalText() ([]byte, error) { return []byte(d.t.Format("2006-01-02")), nil }

func (d *Date) UnmarshalText(b []byte) error {
  t, err := time.Parse("2006-01-02", string(b))
  d.t = t
  return err
}

type Account struct {
  ID     int    `json:"id"`
  Status Status `json:"status"`
  Opened Date   `json:"opened"`
}

func main() {
  a := Account{ID: 7, Status: Suspended, Opened: Date{t: time.Date(2024, 3, 1, 0, 0, 0, 0, time.UTC)}}
  b, err := json.Marshal(a)
  fmt.Println(string(b), err)

  var back Account
  err = json.Unmarshal(b, &back)
  fmt.Println(back.ID, names[back.Status], back.Opened.t.Format("Jan 2 2006"), err)

  err = json.Unmarshal([]byte(`{"status":"nope"}`), &back)
  fmt.Println("bad status:", err)

  d := json.NewDecoder(strings.NewReader(`{"id":1,"status":"active","opened":"2020-01-01"}`))
  var c Account
  fmt.Println(d.Decode(&c), c.ID, names[c.Status])
}
```

Output:

```
{"id":7,"status":"suspended","opened":"2024-03-01"} <nil>
7 suspended Mar 1 2024 <nil>
bad status: unknown status "nope"
<nil> 1 active
```

An `int`-backed enum now round-trips as a readable string, and the error from `UnmarshalJSON`
propagates out of `json.Unmarshal` unchanged.

Note the receivers: **marshal on the value, unmarshal on the pointer**. Unmarshal must mutate,
so it needs `*T`; and `json.Unmarshal` is always given a pointer, whose method set includes
both.

The sharp edge: embedding a type that already has these methods means *its* implementation
wins.

```go
package main

import (
  "encoding/json"
  "fmt"
  "time"
)

type DateEmbed struct{ time.Time }

func (d DateEmbed) MarshalText() ([]byte, error) { return []byte(d.Format("2006-01-02")), nil }

type Date struct{ t time.Time }

func (d Date) MarshalText() ([]byte, error) { return []byte(d.t.Format("2006-01-02")), nil }

func main() {
  when := time.Date(2024, 3, 1, 0, 0, 0, 0, time.UTC)

  b1, _ := json.Marshal(DateEmbed{when})
  fmt.Println("embedded time.Time:", string(b1))

  b2, _ := json.Marshal(Date{when})
  fmt.Println("named field:       ", string(b2))
}
```

Output:

```
embedded time.Time: "2024-03-01T00:00:00Z"
named field:        "2024-03-01"
```

`time.Time` has its own `MarshalJSON`, which outranks the promoted `MarshalText` — so the
embedded version ignores your format entirely. Use a named field.

## The database/sql group

| Interface | Method | Direction |
|---|---|---|
| `driver.Valuer` | `Value() (driver.Value, error)` | Go value -> database |
| `sql.Scanner` | `Scan(src any) error` | database -> Go value |

`driver.Value` is restricted to `int64`, `float64`, `bool`, `[]byte`, `string`, `time.Time`, or
`nil`. Implement both and your type works as a query argument and a `rows.Scan` destination.

```go
package main

import (
  "database/sql/driver"
  "fmt"
  "strings"
)

type Tags []string

func (t Tags) Value() (driver.Value, error) { return strings.Join(t, ","), nil }

func (t *Tags) Scan(src any) error {
  switch v := src.(type) {
  case nil:
    *t = nil
    return nil
  case string:
    *t = strings.Split(v, ",")
    return nil
  case []byte:
    *t = strings.Split(string(v), ",")
    return nil
  default:
    return fmt.Errorf("cannot scan %T into Tags", src)
  }
}

func main() {
  var _ driver.Valuer = Tags(nil)

  v, err := Tags{"a", "b"}.Value()
  fmt.Printf("%q %v\n", v, err)

  var t Tags
  fmt.Println(t.Scan("x,y,z"), t, len(t))
  fmt.Println(t.Scan([]byte("p,q")), t)
  fmt.Println(t.Scan(nil), t == nil)
  fmt.Println(t.Scan(42))
}
```

Output:

```
"a,b" <nil>
<nil> [x y z] 3
<nil> [p q]
<nil> true
cannot scan int into Tags
```

The `nil` and `[]byte` cases are mandatory in practice: NULL columns arrive as `nil`, and
drivers differ on whether text comes back as `string` or `[]byte`. Module 17 uses this.

## The net/http group

| Interface | Method(s) | Notes |
|---|---|---|
| `http.Handler` | `ServeHTTP(ResponseWriter, *Request)` | the server's single abstraction |
| `http.HandlerFunc` | — | a func type with `ServeHTTP`, adapting plain functions |
| `http.ResponseWriter` | `Header() Header`, `Write([]byte) (int, error)`, `WriteHeader(int)` | given to you; wrap it to observe (5.5) |
| `http.RoundTripper` | `RoundTrip(*Request) (*Response, error)` | client-side middleware; stub it to test without a network |
| `http.Flusher` | `Flush()` | streaming/SSE; assert on the `ResponseWriter` |
| `http.Hijacker` | `Hijack() (net.Conn, *bufio.ReadWriter, error)` | take over the raw TCP connection |
| `http.CloseNotifier` | `CloseNotify() <-chan bool` | deprecated; use `Request.Context()` |
| `http.File`, `http.FileSystem` | — | custom file serving; prefer `http.FS(fs.FS)` |
| `http.CookieJar` | `SetCookies`, `Cookies` | client cookie storage |

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

type greetHandler struct{ who string }

func (g greetHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "hello %s from %s", g.who, r.URL.Path)
}

type fakeTransport struct{ body string }

func (f fakeTransport) RoundTrip(r *http.Request) (*http.Response, error) {
  return &http.Response{
    StatusCode: 200,
    Body:       io.NopCloser(strings.NewReader(f.body + " " + r.URL.Host)),
    Header:     http.Header{},
    Request:    r,
  }, nil
}

func main() {
  mux := http.NewServeMux()
  mux.Handle("/s/", greetHandler{who: "struct"})
  mux.Handle("/f/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "hello func")
  }))

  for _, p := range []string{"/s/x", "/f/y"} {
    rec := httptest.NewRecorder()
    mux.ServeHTTP(rec, httptest.NewRequest("GET", p, nil))
    fmt.Println(rec.Code, rec.Body.String())
  }

  rec := httptest.NewRecorder()
  _, isFlusher := any(rec).(http.Flusher)
  _, isHijacker := any(rec).(http.Hijacker)
  fmt.Println("recorder: Flusher =", isFlusher, " Hijacker =", isHijacker)

  c := &http.Client{Transport: fakeTransport{body: "stubbed"}}
  resp, err := c.Get("http://example.invalid/anything")
  if err != nil {
    panic(err)
  }
  b, _ := io.ReadAll(resp.Body)
  fmt.Println(resp.StatusCode, string(b), err)
}
```

Output:

```
200 hello struct from /s/x
200 hello func
recorder: Flusher = true  Hijacker = false
200 stubbed example.invalid <nil>
```

A struct and a function serve the same `Handler` role. And `RoundTripper` is how you test HTTP
clients with no server at all — note the request to `example.invalid` never left the process.
The `Flusher`/`Hijacker` probe is the capability pattern from 5.3: always comma-ok, since not
every `ResponseWriter` implements them.

## context.Context

One interface, four methods, threaded through every blocking API in modern Go.

| Method | Returns |
|---|---|
| `Deadline() (time.Time, bool)` | when it will be cancelled, if ever |
| `Done() <-chan struct{}` | closed on cancellation; `nil` for `Background` |
| `Err() error` | `nil`, `context.Canceled`, or `context.DeadlineExceeded` |
| `Value(key any) any` | request-scoped data |

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "time"
)

type traceKey struct{}

func main() {
  bg := context.Background()
  fmt.Printf("%T %v %v\n", bg, bg.Done(), bg.Err())

  ctx := context.WithValue(bg, traceKey{}, "abc123")
  fmt.Println(ctx.Value(traceKey{}), ctx.Value("missing"))

  cctx, cancel := context.WithCancel(ctx)
  cancel()
  <-cctx.Done()
  fmt.Println(cctx.Err(), errors.Is(cctx.Err(), context.Canceled))
  fmt.Println(cctx.Value(traceKey{}))

  tctx, tcancel := context.WithTimeout(bg, 10*time.Millisecond)
  defer tcancel()
  <-tctx.Done()
  fmt.Println(tctx.Err(), errors.Is(tctx.Err(), context.DeadlineExceeded))
  dl, ok := tctx.Deadline()
  fmt.Println(ok, !dl.IsZero())
}
```

Output:

```
context.backgroundCtx <nil> <nil>
abc123 <nil>
context canceled true
abc123
context deadline exceeded true
true true
```

Each `With*` returns a new implementation wrapping the parent, so `Value` lookups walk up the
chain — `cctx` still sees the trace ID. `Background().Done()` is a nil channel, which blocks
forever in a `select` (10.5). Idiomatic: unexported struct types as value keys, never strings.
Module 11 covers all of it.

## sync.Locker

```go
type Locker interface {
  Lock()
  Unlock()
}
```

Satisfied by `*sync.Mutex`, `*sync.RWMutex`, and `rwmutex.RLocker()`. Useful for writing code
that works with either lock flavour, and for a no-op lock in single-threaded paths.

## hash.Hash

| Interface | Methods | Satisfied by |
|---|---|---|
| `hash.Hash` | `io.Writer` + `Sum([]byte) []byte`, `Reset()`, `Size() int`, `BlockSize() int` | `sha256.New()`, `md5.New()`, `crc32.NewIEEE()`, `fnv.New64a()`, `hmac.New(...)` |
| `hash.Hash32` / `Hash64` | `Hash` + `Sum32()` / `Sum64()` | `crc32`, `fnv` |

Because `hash.Hash` embeds `io.Writer`, you feed it with `io.Copy` or `fmt.Fprintf`:

```go
package main

import (
  "crypto/md5"
  "crypto/sha256"
  "encoding/hex"
  "fmt"
  "hash"
  "hash/crc32"
  "hash/fnv"
  "io"
  "strings"
  "sync"
)

func fingerprint(h hash.Hash, r io.Reader) string {
  io.Copy(h, r)
  return hex.EncodeToString(h.Sum(nil))
}

type noopLock struct{}

func (noopLock) Lock()   {}
func (noopLock) Unlock() {}

func bump(l sync.Locker, n *int) {
  l.Lock()
  *n++
  l.Unlock()
}

func main() {
  for _, h := range []hash.Hash{sha256.New(), md5.New(), crc32.NewIEEE(), fnv.New64a()} {
    fmt.Printf("%-18T size=%-2d block=%-3d %s\n", h, h.Size(), h.BlockSize(),
      fingerprint(h, strings.NewReader("hello")))
  }

  var n int
  var mu sync.Mutex
  var rw sync.RWMutex
  for _, l := range []sync.Locker{&mu, &rw, rw.RLocker(), noopLock{}} {
    bump(l, &n)
  }
  fmt.Println(n)
}
```

Output:

```
*sha256.Digest     size=32 block=64  2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
*md5.digest        size=16 block=64  5d41402abc4b2a76b9719d911017c592
*crc32.digest      size=4  block=1   3610a686
*fnv.sum64a        size=8  block=1   a430d84680aabd0b
4
```

One `fingerprint` function, four algorithms. `Sum(nil)` appends the digest to the argument and
does **not** reset the hash — call `Reset()` to reuse it.

## io/fs

| Interface | Methods | Satisfied by |
|---|---|---|
| `fs.FS` | `Open(name string) (File, error)` | `os.DirFS`, `embed.FS`, `fstest.MapFS`, `zip.Reader` |
| `fs.File` | `Stat() (FileInfo, error)`, `Read([]byte) (int, error)`, `Close() error` | what `Open` returns |
| `fs.DirEntry` | `Name()`, `IsDir()`, `Type()`, `Info()` | `ReadDir`, `WalkDir` callbacks |
| `fs.FileInfo` | `Name()`, `Size()`, `Mode()`, `ModTime()`, `IsDir()`, `Sys()` | `Stat` results |
| `fs.ReadDirFS`, `ReadFileFS`, `StatFS`, `GlobFS`, `SubFS` | extension interfaces | optional fast paths `fs.ReadFile` etc. probe for |

Take an `fs.FS` instead of a directory path and your code is testable with no disk at all:

```go
package main

import (
  "fmt"
  "io/fs"
  "testing/fstest"
)

func countLines(fsys fs.FS, name string) (int, error) {
  b, err := fs.ReadFile(fsys, name)
  if err != nil {
    return 0, err
  }
  n := 0
  for _, c := range b {
    if c == '\n' {
      n++
    }
  }
  return n, nil
}

func main() {
  fsys := fstest.MapFS{
    "a.txt":      {Data: []byte("one\ntwo\n")},
    "sub/b.txt":  {Data: []byte("x\n")},
    "sub/c.json": {Data: []byte("{}")},
  }

  n, err := countLines(fsys, "a.txt")
  fmt.Println(n, err)
  _, err = countLines(fsys, "missing.txt")
  fmt.Println(err)

  fs.WalkDir(fsys, ".", func(p string, d fs.DirEntry, err error) error {
    info, _ := d.Info()
    fmt.Printf("%-12s dir=%-5t size=%d\n", p, d.IsDir(), info.Size())
    return nil
  })

  matches, _ := fs.Glob(fsys, "sub/*.txt")
  fmt.Println(matches)

  f, _ := fsys.Open("a.txt")
  st, _ := f.Stat()
  fmt.Println(st.Name(), st.Mode().IsRegular())
  f.Close()
}
```

Output:

```
2 <nil>
open missing.txt: file does not exist
.            dir=true  size=0
a.txt        dir=false size=8
sub          dir=true  size=0
sub/b.txt    dir=false size=2
sub/c.json   dir=false size=2
[sub/b.txt]
a.txt true
```

The same function runs against `os.DirFS("/real/path")` or an `embed.FS` unchanged. 13.10
covers the package.

## flag.Value

```go
type Value interface {
  String() string
  Set(string) error
}
```

Implement it and `flag.Var` accepts your type, with your parsing and your error message.

```go
package main

import (
  "flag"
  "fmt"
  "os"
  "strings"
  "time"
)

type CSV []string

func (c *CSV) String() string { return strings.Join(*c, ",") }
func (c *CSV) Set(v string) error {
  if v == "" {
    return fmt.Errorf("empty value")
  }
  *c = append(*c, v)
  return nil
}

type Level int

func (l *Level) String() string { return [...]string{"debug", "info", "warn"}[*l] }
func (l *Level) Set(v string) error {
  for i, n := range []string{"debug", "info", "warn"} {
    if n == v {
      *l = Level(i)
      return nil
    }
  }
  return fmt.Errorf("invalid level %q", v)
}

func main() {
  fs := flag.NewFlagSet("demo", flag.ContinueOnError)
  fs.SetOutput(os.Stdout)

  var tags CSV
  var lvl Level
  fs.Var(&tags, "tag", "repeatable tag")
  fs.Var(&lvl, "level", "debug|info|warn")
  timeout := fs.Duration("timeout", time.Second, "how long")

  err := fs.Parse([]string{"-tag", "a", "-tag", "b", "-level", "warn", "-timeout", "250ms"})
  fmt.Println(err, tags.String(), lvl.String(), *timeout)

  fs2 := flag.NewFlagSet("demo", flag.ContinueOnError)
  fs2.SetOutput(os.Stdout)
  var l2 Level
  fs2.Var(&l2, "level", "debug|info|warn")
  fmt.Println("---")
  fmt.Println("parse error:", fs2.Parse([]string{"-level", "nope"}))
}
```

Output:

```
<nil> a,b warn 250ms
---
invalid value "nope" for flag -level: invalid level "nope"
Usage of demo:
  -level value
    	debug|info|warn
parse error: invalid value "nope" for flag -level: invalid level "nope"
```

Pointer receivers throughout, since `Set` must mutate. A repeatable flag falls out of
`Set` appending, and `flag` wraps your error with the flag name and prints usage. `flag.TextVar`
reuses `encoding.TextUnmarshaler` instead, if your type already has it. 13.11 covers `flag`.

## Everything else worth recognising

| Interface | Signature | Where |
|---|---|---|
| `sort.Interface` | see above | `sort` |
| `driver.Valuer`, `sql.Scanner` | see above | `database/sql` |
| `net.Conn` | `io.ReadWriteCloser` + addresses + deadlines | `net` |
| `net.Listener` | `Accept() (Conn, error)`, `Close()`, `Addr()` | `net` |
| `net.Addr` | `Network() string`, `String() string` | `net` |
| `net.Error` | `error` + `Timeout() bool` | distinguish timeouts from real failures |
| `iter.Seq[V]` / `iter.Seq2[K,V]` | func types, not interfaces | range-over-func, 13.14 |
| `slog.Handler` | `Enabled`, `Handle`, `WithAttrs`, `WithGroup` | custom log backends, 13.12 |
| `slog.LogValuer` | `LogValue() slog.Value` | lazy/redacted log fields |
| `testing.TB` | the common `*testing.T` / `*testing.B` surface | helpers usable from tests and benchmarks, 14.3 |
| `reflect.Type` | a large interface | 13.15 |
| `time.Time` methods | not an interface — a concrete type | people expect a `Clock` interface; define your own |

## Quick Reference

| I want my type to... | Implement |
|---|---|
| print nicely with `fmt`, `log`, `slog` | `String() string` |
| print in Go syntax under `%#v` | `GoString() string` |
| control every format verb | `fmt.Formatter` |
| be an error | `Error() string` |
| participate in `errors.Is`/`As` | `Unwrap() error`, optionally `Is`/`As` |
| be sortable by `sort.Sort` | `Len`, `Less`, `Swap` |
| encode to JSON specially | `MarshalJSON` / `UnmarshalJSON` |
| encode to JSON, XML, slog, and map keys with one method | `MarshalText` / `UnmarshalText` |
| be a gob/cache payload | `MarshalBinary` / `UnmarshalBinary` |
| go in and out of a SQL column | `Value()` and `Scan(any)` |
| be an output stream | `Write([]byte) (int, error)` |
| be an input stream | `Read([]byte) (int, error)` |
| serve HTTP | `ServeHTTP(ResponseWriter, *Request)` |
| intercept HTTP client calls | `RoundTrip(*Request) (*Response, error)` |
| be a command-line flag | `String()` and `Set(string) error` |
| be a lock | `Lock()`, `Unlock()` |
| be a hash | `io.Writer` + `Sum`, `Reset`, `Size`, `BlockSize` |
| be a filesystem | `Open(string) (fs.File, error)` |

| Receiver rule | Why |
|---|---|
| `String()`, `MarshalJSON()`, `MarshalText()`, `Value()` on the **value** | so both `T` and `*T` print/encode |
| `UnmarshalJSON()`, `UnmarshalText()`, `Scan()`, `Set()`, `Read()`, `Write()` on the **pointer** | they mutate |
| `sort.Interface` on a slice type by **value** | the slice header is copied but the array is shared |

## Common Mistakes

**Calling `fmt.Sprintf("%v", x)` inside `x`'s own `String()`.** It recurses until the stack
overflows; print the fields, or convert to a local `type plain T`.

**Putting `String()` on a pointer receiver.** Printing a value of the type then skips it
silently; use a value receiver.

**Putting `UnmarshalJSON` on a value receiver.** It compiles and silently does nothing, because
it mutates a copy.

**Embedding `time.Time` and adding `MarshalText`.** `time.Time`'s own `MarshalJSON` outranks it;
use a named field instead.

**Forgetting the `nil` and `[]byte` cases in `sql.Scanner.Scan`.** NULL columns and
driver-specific text representations will both reach you.

**Asserting `http.Flusher` without comma-ok.** Not every `ResponseWriter` implements it; the
single-return form panics.

## Exercises

1. Implement `fmt.Stringer`, `json.Marshaler`, `encoding.TextMarshaler`, and `driver.Valuer` on
   one `Money` type backed by `int64` cents. Determine experimentally which method `json.Marshal`
   picks, and which `slog` picks.
2. Write a `type teeReader struct { io.Reader; log io.Writer }` that logs every byte read, then
   plug it into `json.NewDecoder` to see exactly what the decoder consumes and when.
3. Implement `sort.Interface` on a type whose `Less` reads a `[]string` of field names, so the
   sort key is chosen at runtime. Then write the `slices.SortFunc` equivalent and compare.
4. Implement `flag.Value` for a `time.Time` in `2006-01-02` form, and again using
   `flag.TextVar` with `encoding.TextUnmarshaler`. Capture the error text for an invalid date in
   both.
5. Write a `countingTransport` implementing `http.RoundTripper` that wraps
   `http.DefaultTransport` and counts requests per host. Test it entirely with
   `httptest.NewServer`.
6. Pick a type from your own code and make it satisfy three stdlib interfaces it does not
   currently satisfy. For each, name a stdlib function that now accepts it and prove it with
   five lines of code.

---

[← 5.5 Interface Composition and Small Interfaces](05-composition.md) · [Course Index](../README.md) · [6.1 string, []byte, and []rune →](../06-strings/01-string-bytes-runes.md)
