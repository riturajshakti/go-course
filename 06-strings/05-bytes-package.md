[← 6.4 strings.Builder, Replacer, and Reader](04-builder-replacer.md) · [Course Index](../README.md) · [6.6 strconv: Parsing and Formatting →](06-strconv.md)

# 6.5 — The bytes Package Reference

## What You'll Learn

- `bytes` mirrors `strings` function for function, with four deliberate differences
- `bytes.Buffer` as both `io.Reader` and `io.Writer` — the test-capture workhorse
- `Buffer.Bytes()` aliases the internal array, and `NewBuffer` takes ownership of yours
- When `[]byte` is the right type and when `string` is

## Coming From Another Language

| Other | Go |
|---|---|
| Node `Buffer` | `[]byte`, plus `bytes.Buffer` for growable writing |
| Python `bytes` (immutable) | `[]byte` is **mutable** |
| Python `bytearray` | `[]byte` |
| Python `io.BytesIO` | `bytes.Buffer` (write) / `bytes.Reader` (read) |
| Rust `Vec<u8>` / `&[u8]` | `[]byte` / `[]byte` |
| JS `a === b` on arrays | `bytes.Equal(a, b)`; `==` does not compile |

## The mirror

Almost every `strings` function has a `bytes` twin with the same name and semantics,
with `string` parameters and results replaced by `[]byte`. Everything you learned in
6.3 — the cutset-is-a-rune-set trap, `Split`'s one-element result, `Fields`
collapsing whitespace, `Index` returning -1, `Cut`'s three values, `Repeat` panicking
on a negative count — holds identically here.

The mirrored set, all present in go1.27.1:

| Group | Functions (all `[]byte` in, `[]byte`/`int`/`bool` out) |
|---|---|
| Searching | `Contains`, `ContainsAny`, `ContainsRune`, `ContainsFunc`, `Index`, `IndexAny`, `IndexByte`, `IndexRune`, `IndexFunc`, `LastIndex`, `LastIndexAny`, `LastIndexByte`, `LastIndexFunc`, `Count`, `HasPrefix`, `HasSuffix`, `EqualFold` |
| Splitting | `Split`, `SplitN`, `SplitAfter`, `SplitAfterN`, `SplitSeq`, `SplitAfterSeq`, `Fields`, `FieldsFunc`, `FieldsSeq`, `FieldsFuncSeq`, `Lines`, `Join`, `Cut`, `CutLast`, `CutPrefix`, `CutSuffix` |
| Trimming | `Trim`, `TrimLeft`, `TrimRight`, `TrimSpace`, `TrimPrefix`, `TrimSuffix`, `TrimFunc`, `TrimLeftFunc`, `TrimRightFunc` |
| Case / mapping | `ToUpper`, `ToLower`, `ToTitle`, `Title` (deprecated), `ToUpperSpecial`, `ToLowerSpecial`, `ToTitleSpecial`, `ToValidUTF8`, `Map` |
| Building | `Repeat`, `Replace`, `ReplaceAll`, `Clone` |
| Comparing | `Equal`, `Compare`, `EqualFold` |

Three signature quirks to remember: the cutset arguments of `Trim`, `TrimLeft`,
`TrimRight` and the `chars` argument of `ContainsAny`/`IndexAny`/`LastIndexAny` are
still `string`, not `[]byte`, because a cutset is naturally a literal. `Split`
returns `[][]byte`. And the `Seq` variants return `iter.Seq[[]byte]`.

## What bytes has that strings does not

| Name | Signature | Note |
|---|---|---|
| `Equal` | `(a, b []byte) bool` | `==` is illegal on slices |
| `Runes` | `(s []byte) []rune` | the `[]rune(s)` conversion, as a function |
| `Buffer` | type | growable read/write buffer |
| `Reader` | type | read-only cursor over a `[]byte` |
| `NewBuffer` / `NewBufferString` | `([]byte) *Buffer` / `(string) *Buffer` | |
| `MinRead` | `const = 512` | `ReadFrom`'s guaranteed spare capacity |
| `ErrTooLarge` | `var` | panic value when a Buffer cannot grow |

And what `strings` has that `bytes` does not: `Builder` (use `Buffer`) and
`Replacer` (no `[]byte` equivalent — convert, or use a `Buffer` by hand).

```go
package main

import (
  "bytes"
  "fmt"
)

func main() {
  a := []byte("abc")
  c := []byte("abc")
  fmt.Println(bytes.Equal(a, c), bytes.Equal(nil, []byte{}), bytes.Equal(a, []byte("abd")))
  fmt.Println(bytes.Compare(a, []byte("abd")), bytes.Compare(a, c), bytes.Compare([]byte("b"), a))
  fmt.Println(bytes.EqualFold([]byte("Go"), []byte("gO")))
  fmt.Printf("%q\n", bytes.Clone(a))
  fmt.Println(bytes.Clone(nil) == nil)
  fmt.Println(bytes.Runes([]byte("héllo")))
  fmt.Printf("%q\n", bytes.ToUpper([]byte("héllo")))
  fmt.Printf("%q\n", bytes.Trim([]byte("xxhixx"), "x"))
  fmt.Printf("%q\n", bytes.Split([]byte("a,b,c"), []byte(",")))
  fmt.Printf("%q\n", bytes.Join([][]byte{[]byte("a"), []byte("b")}, []byte("-")))
  fmt.Println(bytes.Contains([]byte("seafood"), []byte("foo")), bytes.Index([]byte("chicken"), []byte("ken")))
  fmt.Printf("%q\n", bytes.Map(func(r rune) rune { return r + 1 }, []byte("abc")))
  fmt.Printf("%q\n", bytes.Repeat([]byte("ab"), 3))
  fmt.Println(bytes.ContainsFunc([]byte("a1"), func(r rune) bool { return r >= '0' && r <= '9' }))
  fmt.Println(bytes.MinRead)
}
```

Output:

```
true true false
-1 0 1
true
"abc"
true
[104 233 108 108 111]
"HÉLLO"
"hi"
["a" "b" "c"]
"a-b"
true 4
"bcd"
"ababab"
true
512
```

`bytes.Equal(nil, []byte{})` is **true** — it compares contents, so nil and empty
are equal, unlike `a == nil`. `bytes.Clone(nil)` returns nil rather than an empty
slice. And `bytes.Map` decodes runes, so it operates on code points even though the
input is bytes.

`==` on two slices does not compile, which is why `Equal` exists:

```go
package main

import "fmt"

func main() {
  a := []byte("abc")
  b := []byte("abc")
  fmt.Println(a == b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: a == b (slice can only be compared to nil)
```

## bytes.Buffer

The workhorse. Zero value ready, grows as needed, and satisfies `io.Writer`,
`io.Reader`, `io.ByteWriter`, `io.ByteScanner`, `io.RuneScanner`, `io.ReaderFrom`,
and `io.WriterTo` — so it sits on either end of an `io.Copy`.

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "strings"
)

func main() {
  var b bytes.Buffer
  fmt.Println(b.Len(), b.Cap())

  b.WriteString("hello ")
  b.Write([]byte("world"))
  b.WriteByte('!')
  b.WriteRune('世')
  fmt.Printf("%q len=%d cap=%d\n", b.String(), b.Len(), b.Cap())

  fmt.Fprintf(&b, " n=%d", 7)
  fmt.Printf("%q\n", b.String())

  line, err := b.ReadString(' ')
  fmt.Printf("%q %v remaining=%q\n", line, err, b.String())

  c, _ := b.ReadByte()
  fmt.Printf("%q\n", c)
  fmt.Printf("next3=%q rest=%q\n", b.Next(3), b.String())

  b.Truncate(2)
  fmt.Printf("%q len=%d\n", b.String(), b.Len())
  b.Reset()
  fmt.Printf("%q len=%d cap=%d\n", b.String(), b.Len(), b.Cap())

  var sink bytes.Buffer
  n, _ := sink.ReadFrom(strings.NewReader("from a reader"))
  fmt.Println(n, sink.String())
  var out strings.Builder
  m, _ := sink.WriteTo(&out)
  fmt.Println(m, out.String(), sink.Len())

  var rw bytes.Buffer
  var _ io.Reader = &rw
  var _ io.Writer = &rw
  rw.WriteString("abc")
  data, _ := io.ReadAll(&rw)
  fmt.Printf("%q\n", data)
}
```

Output:

```
0 0
"hello world!世" len=15 cap=64
"hello world!世 n=7"
"hello " <nil> remaining="world!世 n=7"
'w'
next3="orl" rest="d!世 n=7"
"d!" len=2
"" len=0 cap=64
13 from a reader
13 from a reader 0
"abc"
```

A Buffer is a FIFO, not an array: reading consumes from the front, and `String()`
shows only the **unread** portion. So after `ReadString(' ')`, `"hello "` is gone.
`Truncate(n)` keeps the first `n` unread bytes, `Reset()` empties it but — unlike
`strings.Builder` — **keeps the capacity** (64 here), which is why a Buffer is the
right choice for a `sync.Pool`-style reusable scratch space. `WriteTo` drains it to
zero.

| Method | Signature | Note |
|---|---|---|
| `Write` | `(p []byte) (int, error)` | `io.Writer` |
| `WriteString` | `(s string) (int, error)` | no `[]byte` conversion needed |
| `WriteByte` | `(c byte) error` | |
| `WriteRune` | `(r rune) (int, error)` | UTF-8 encodes |
| `Read` | `(p []byte) (int, error)` | `io.Reader`; consumes |
| `ReadByte` / `UnreadByte` | `() (byte, error)` / `() error` | |
| `ReadRune` / `UnreadRune` | `() (rune, int, error)` / `() error` | |
| `ReadBytes` | `(delim byte) ([]byte, error)` | includes the delimiter |
| `ReadString` | `(delim byte) (string, error)` | includes the delimiter |
| `Next` | `(n int) []byte` | consumes n bytes; **only valid until the next write** |
| `Peek` | `(n int) ([]byte, error)` | does not consume |
| `Bytes` | `() []byte` | **aliases** the internal array |
| `AvailableBuffer` | `() []byte` | empty slice with spare cap, for append-then-Write |
| `Available` | `() int` | spare capacity |
| `String` | `() string` | copies the unread portion |
| `Len` | `() int` | unread bytes |
| `Cap` | `() int` | total capacity |
| `Truncate` | `(n int)` | keep the first n unread bytes |
| `Reset` | `()` | empty, **keeps capacity** |
| `Grow` | `(n int)` | reserve |
| `ReadFrom` | `(io.Reader) (int64, error)` | `io.ReaderFrom` |
| `WriteTo` | `(io.Writer) (int64, error)` | `io.WriterTo`; drains |

`ErrTooLarge` is a panic value, not a returned error: a Buffer that cannot grow
panics with `bytes.ErrTooLarge`.

## The test-capture pattern

This is why most Go programmers meet `bytes.Buffer` first. Write production code
against `io.Writer`, and a Buffer captures it in the test.

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "os"
)

func report(w io.Writer, items map[string]int, order []string) {
  for _, k := range order {
    fmt.Fprintf(w, "%s=%d\n", k, items[k])
  }
}

func main() {
  items := map[string]int{"a": 1, "b": 2}
  order := []string{"a", "b"}

  report(os.Stdout, items, order)

  var buf bytes.Buffer
  report(&buf, items, order)
  fmt.Printf("captured %d bytes: %q\n", buf.Len(), buf.String())
}
```

Output:

```
a=1
b=2
captured 8 bytes: "a=1\nb=2\n"
```

## Buffer.Bytes() aliases the internal array

`Bytes()` is not a copy. It is a window on the Buffer's own storage, and it goes
stale the moment the Buffer is written to or reset.

```go
package main

import (
  "bytes"
  "fmt"
)

func main() {
  var b bytes.Buffer
  b.WriteString("hello")
  bs := b.Bytes()
  fmt.Printf("%q\n", bs)
  bs[0] = 'H'
  fmt.Printf("%q\n", b.String())

  b.Reset()
  b.WriteString("WORLD")
  fmt.Printf("stale view: %q\n", bs)

  backing := []byte("seed-")
  nb := bytes.NewBuffer(backing)
  nb.WriteString("more")
  fmt.Printf("%q %q\n", nb.String(), backing)

  sb := bytes.NewBufferString("seed-")
  sb.WriteString("more")
  fmt.Printf("%q\n", sb.String())

  empty := bytes.NewBuffer(make([]byte, 0, 64))
  empty.WriteString("no prefix")
  fmt.Printf("%q cap=%d\n", empty.String(), empty.Cap())
}
```

Output:

```
"hello"
"Hello"
stale view: "WORLD"
"seed-more" "seed-"
"seed-more"
"no prefix" cap=64
```

Writing through `bs` changed the Buffer's contents. Worse, after `Reset` and a new
write, the same `bs` now shows `"WORLD"` — a view onto reused memory. Idiomatic: use
`String()`, or `bytes.Clone(b.Bytes())`, for anything that outlives the next write.

The `NewBuffer` ownership gotcha is on the fourth line: `bytes.NewBuffer(backing)`
does **not** copy — it takes your slice as the initial contents and may write into
and beyond it. `NewBufferString` copies, since a string cannot be adopted. Passing
`make([]byte, 0, 64)` is the useful form: an empty Buffer with capacity pre-reserved.

| Constructor | Initial contents | Note |
|---|---|---|
| `var b bytes.Buffer` | empty | the default |
| `bytes.NewBuffer(bs)` | `bs` | **adopts** `bs`; do not use `bs` afterwards |
| `bytes.NewBuffer(make([]byte, 0, n))` | empty, cap n | pre-sized empty buffer |
| `bytes.NewBufferString(s)` | `s` | copies |

## bytes.Reader

The `[]byte` counterpart to `strings.Reader`, with the same interface set plus
`Reset`.

```go
package main

import (
  "bytes"
  "fmt"
  "io"
)

func main() {
  r := bytes.NewReader([]byte("hello 世界"))
  fmt.Println(r.Len(), r.Size())
  buf := make([]byte, 5)
  n, _ := r.Read(buf)
  fmt.Printf("%d %q\n", n, buf[:n])
  r.Seek(0, io.SeekStart)
  all, _ := io.ReadAll(r)
  fmt.Printf("%q\n", all)
}
```

Output:

```
12 12
5 "hello"
"hello 世界"
```

Use `bytes.NewReader` over `bytes.NewBuffer` for read-only input: it is smaller, it
is seekable, and it never modifies the slice. A `Buffer` can only be read forwards
once.

## Which type, and what conversion costs

Converting between them copies, with two compiler-elided exceptions (6.1):

```
BenchmarkBytesConvEscaping-12     	117129032	        10.23 ns/op	      48 B/op	       1 allocs/op
BenchmarkStringConvEscaping-12    	143967189	         8.198 ns/op	      48 B/op	       1 allocs/op
BenchmarkRangeOverBytesConv-12    	25192749	        47.73 ns/op	       0 B/op	       0 allocs/op
BenchmarkMapLookupBytesKey-12     	203641864	         5.864 ns/op	       0 B/op	       0 allocs/op
```

A 43-byte payload: one 48-byte allocation each way when the result escapes, zero
when it does not. Machine-specific numbers (darwin/arm64, Apple M4 Pro); the
allocation counts are the stable part.

| Use `[]byte` | Use `string` |
|---|---|
| anything from `io.Reader` / `os.File` / `net.Conn` | map keys |
| buffers you will mutate or reuse | struct fields in long-lived values |
| `encoding/json`, `crypto`, `hash` inputs | values compared with `==` or `<` |
| avoiding a copy on a hot I/O path | anything that must not change under you |

Idiomatic: keep data as `[]byte` for the whole I/O path and convert to `string` once,
at the point where it becomes an identifier or a map key. Converting back and forth
in a loop is the anti-pattern.

## Quick Reference

| strings | bytes | Difference |
|---|---|---|
| `strings.Builder` | `bytes.Buffer` | Buffer is also readable; `Reset` keeps capacity |
| `strings.NewReader` | `bytes.NewReader` | identical interface set |
| `strings.Replacer` | — | no `[]byte` version |
| `a == b` | `bytes.Equal(a, b)` | `==` illegal on slices |
| `strings.Clone` | `bytes.Clone` | `Clone(nil)` returns nil |
| — | `bytes.Runes` | same as `[]rune(s)` |
| `strings.Compare` | `bytes.Compare` | both -1/0/1 |

| Trap | Fix |
|---|---|
| `Bytes()` aliases internal storage | `bytes.Clone(b.Bytes())` or `b.String()` |
| `NewBuffer(bs)` adopts `bs` | use `NewBufferString`, or stop using `bs` |
| `Next(n)` invalid after the next write | copy it if you keep it |
| Buffer `String()` shows only unread data | do not read from it if you need the whole thing |
| converting in a loop | convert once at the boundary |

## Common Mistakes

**Keeping the slice from `Buffer.Bytes()` after another write.** It aliases storage
that may be reused; clone it or use `String()`.

**Passing a live slice to `bytes.NewBuffer`.** The Buffer adopts it and writes
through it; use `NewBufferString` or a fresh `make`.

**Comparing `[]byte` with `==`.** It does not compile; use `bytes.Equal`, or
`bytes.Compare` when you need an ordering.

**Reading a Buffer and then calling `String()` expecting everything.** Reads
consume; `String()` returns only the unread remainder.

**Round-tripping `string(b)` and `[]byte(s)` inside a loop.** Each conversion copies;
pick one representation per code path.

## Exercises

1. Write `func countLines(r io.Reader) (int, error)` using a `bytes.Buffer` and
   `ReadBytes('\n')`, and test it with `bytes.NewReader`, `strings.NewReader`, and a
   real file. Decide what a final line with no newline should count as.
2. Demonstrate the `Bytes()` aliasing hazard as a failing test: capture
   `b.Bytes()`, write 10 KB more to force a reallocation, and show the captured
   slice is now detached and stale. Explain why the result differs from the
   small-write case.
3. Write the same word-frequency counter twice, once keyed by `string(word)` and
   once keeping `[]byte` with a manual hash, and benchmark both over a 5 MB text
   with `-benchmem`.
4. Build a `bytes.Buffer` pool with `sync.Pool`, resetting on Put, and benchmark it
   against allocating a fresh Buffer per call. Verify with `-race` that nothing
   leaks between users.
5. Implement `func ReplaceAllBytes(s []byte, pairs ...[]byte) []byte` giving
   `strings.Replacer`'s single-pass semantics for `[]byte`, and prove with the
   `a`/`b` swap that it differs from chained `bytes.ReplaceAll`.

---

[← 6.4 strings.Builder, Replacer, and Reader](04-builder-replacer.md) · [Course Index](../README.md) · [6.6 strconv: Parsing and Formatting →](06-strconv.md)
