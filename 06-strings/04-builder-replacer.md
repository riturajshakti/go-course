[← 6.3 The strings Package Reference](03-strings-package.md) · [Course Index](../README.md) · [6.5 The bytes Package Reference →](05-bytes-package.md)

# 6.4 — strings.Builder, Replacer, and Reader

## What You'll Learn

- `strings.Builder` beats `+=` by 31x and 110 allocations, with real benchmark numbers
- A `Builder` must not be copied after its first write — the panic text, captured
- `strings.Replacer` does one pass over the input, so chained substitutions do not cascade
- `strings.Reader` turns a string into an `io.Reader`/`Seeker`/`ReaderAt` for free

## Coming From Another Language

| Other | Go |
|---|---|
| JS `arr.push(x)` then `arr.join("")` | `strings.Builder` + `WriteString` |
| Python `"".join(parts)` | `strings.Join(parts, "")` or a `Builder` |
| Python `io.StringIO()` | `strings.Builder` (write) / `strings.Reader` (read) |
| Rust `String::push_str` | `(*Builder).WriteString` |
| Node `Buffer`/stream source | `strings.NewReader(s)` as an `io.Reader` |

## strings.Builder

The zero value is ready to use — no constructor, no `make`. All write methods are
on the pointer receiver, so declare it as a value and let Go take the address.

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  var b strings.Builder
  fmt.Println(b.Len(), b.Cap(), b.String() == "")

  b.WriteString("hello")
  b.WriteByte(' ')
  b.WriteRune('世')
  n, err := b.Write([]byte("!"))
  fmt.Println(n, err)
  fmt.Printf("%q len=%d cap=%d\n", b.String(), b.Len(), b.Cap())

  fmt.Fprintf(&b, " [%d-%s]", 42, "x")
  fmt.Printf("%q\n", b.String())

  b.Reset()
  fmt.Printf("%q len=%d cap=%d\n", b.String(), b.Len(), b.Cap())

  var g strings.Builder
  g.Grow(64)
  fmt.Println(g.Len(), g.Cap())
  g.WriteString("abc")
  fmt.Println(g.Len(), g.Cap())
}
```

Output:

```
0 0 true
1 <nil>
"hello 世!" len=10 cap=16
"hello 世! [42-x]"
"" len=0 cap=0
0 64
3 64
```

Points to note. `Len()` is bytes written, not runes. `Reset()` drops the buffer
entirely, so `Cap()` goes back to 0 — it does not retain capacity for reuse, unlike
`bytes.Buffer`. `Grow(n)` pre-sizes so the whole build is one allocation. And
`String()` is free: it does an unsafe no-copy conversion of the internal `[]byte`,
which is only sound because the Builder never hands that slice out and never lets
you shrink it.

Every write method's error is always nil — the signatures exist only to satisfy
`io.Writer` and friends. Ignoring them is idiomatic here, and `errcheck`-style
linters special-case `Builder`.

## Builder satisfies io.Writer

This is the practical reason to prefer it over manual concatenation: anything that
writes to an `io.Writer` can target it.

```go
package main

import (
  "fmt"
  "strings"
  "text/tabwriter"
)

func main() {
  var b strings.Builder
  w := tabwriter.NewWriter(&b, 0, 0, 2, ' ', 0)
  fmt.Fprintln(w, "name\tqty")
  fmt.Fprintln(w, "widget\t3")
  fmt.Fprintln(w, "bolt\t12")
  w.Flush()
  fmt.Print(b.String())
}
```

Output:

```
name    qty
widget  3
bolt    12
```

## A Builder must not be copied after its first write

The Builder stores a pointer to itself to detect this. Copying it by value — by
assignment, by passing it to a function by value, or by storing it in a struct that
gets copied — trips a check on the next write.

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  var a strings.Builder
  a.WriteString("hi")
  b := a
  b.WriteString(" there")
  fmt.Println(b.String())
}
```

This program panics on purpose. Output:

```
panic: strings: illegal use of non-zero Builder copied by value

goroutine 1 [running]:
strings.(*Builder).copyCheck(...)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/strings/builder.go:41
strings.(*Builder).WriteString(...)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/strings/builder.go:113
main.main()
	/tmp/scratch/main.go:12 +0x150
exit status 2
```

Without that check the copy would share a backing array with the original and the
two would silently corrupt each other. Copying a *zero* Builder is fine, which is
why `var b strings.Builder` as a struct field works until something writes. Always
pass `*strings.Builder`.

## Why Builder beats +=

Real `go test -bench . -benchmem` output, building a 1000-byte string six ways:

```
goos: darwin
goarch: arm64
pkg: m06k4xq9z/p/bench_concat
cpu: Apple M4 Pro
BenchmarkPlusEquals-12     	   18517	     64932 ns/op	  530278 B/op	     999 allocs/op
BenchmarkBuilder-12        	  565959	      2098 ns/op	    3320 B/op	       9 allocs/op
BenchmarkBuilderGrow-12    	  619574	      1814 ns/op	    1024 B/op	       1 allocs/op
BenchmarkBytesBuffer-12    	  397798	      2927 ns/op	    3008 B/op	       6 allocs/op
BenchmarkSprintf-12        	   12553	     95446 ns/op	  546658 B/op	    1998 allocs/op
BenchmarkJoin-12           	  212330	      5828 ns/op	    1024 B/op	       1 allocs/op
PASS
```

Absolute times are machine-specific (darwin/arm64, Apple M4 Pro); the ratios and
the allocation counts are not. The benchmarked loop body:

```go
func BenchmarkPlusEquals(b *testing.B) {
  for b.Loop() {
    s := ""
    for i := 0; i < n; i++ {
      s += "x"
    }
    _ = s
  }
}

func BenchmarkBuilderGrow(b *testing.B) {
  for b.Loop() {
    var sb strings.Builder
    sb.Grow(n)
    for i := 0; i < n; i++ {
      sb.WriteString("x")
    }
    _ = sb.String()
  }
}
```

`+=` allocates 999 times and copies 530 KB to produce 1 KB of output, because round
*k* copies *k* bytes — the O(n²). `Builder` grows geometrically: 9 allocations.
With `Grow` it is exactly 1. `fmt.Sprintf` in the loop is worse than `+=` because
it adds formatting overhead on top of the same quadratic copying.

| Approach | 1000 appends | Allocs | Use when |
|---|---|---|---|
| `a + b + c` single expression | fine | 1 | you know all parts at once |
| `s += x` in a loop | 64932 ns | 999 | never |
| `fmt.Sprintf("%s%s", s, x)` in a loop | 95446 ns | 1998 | never |
| `strings.Builder` | 2098 ns | 9 | the default for loops |
| `strings.Builder` + `Grow` | 1814 ns | 1 | you can estimate the size |
| `bytes.Buffer` | 2927 ns | 6 | you also need to read it back, or want `[]byte` |
| `strings.Join(parts, sep)` | 5828 ns | 1 | you already have the slice |

Idiomatic: `strings.Builder` when you are appending in a loop; `strings.Join` when
the pieces are already in a slice and you want a separator. `Join` looks slower here
only because the benchmark excludes building the 1000-element slice it needs.

## strings.Replacer

`NewReplacer` takes old/new pairs and does a **single left-to-right pass**, which is
the behaviour chained `ReplaceAll` calls cannot give you. It is safe for concurrent
use and worth building once as a package-level value.

```go
package main

import (
  "fmt"
  "os"
  "strings"
)

func main() {
  r := strings.NewReplacer("<", "&lt;", ">", "&gt;", "&", "&amp;")
  fmt.Println(r.Replace("<a href='x'>b & c</a>"))

  swap := strings.NewReplacer("a", "b", "b", "a")
  fmt.Println(swap.Replace("abab"))
  fmt.Println(strings.ReplaceAll(strings.ReplaceAll("abab", "a", "b"), "b", "a"))

  n, err := r.WriteString(os.Stdout, "\n<tag>\n")
  fmt.Println(n, err)
}
```

Output:

```
&lt;a href='x'&gt;b &amp; c&lt;/a&gt;
baba
aaaa

&lt;tag&gt;
13 <nil>
```

The swap is the point: `Replacer` turns `"abab"` into `"baba"`, while the two
chained `ReplaceAll` calls produce `"aaaa"` because the second pass sees the output
of the first. Note also that the HTML escaper handles `&` correctly even though
`&lt;` contains an `&` — a single pass never revisits what it emitted.

When two patterns could match at the same position, the **earliest pair in the
argument list** wins. `WriteString` streams the result to an `io.Writer` with no
intermediate string.

An odd number of arguments is a panic, not an error:

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  r := strings.NewReplacer("a", "b", "c")
  fmt.Println(r.Replace("abc"))
}
```

This program panics on purpose. Output:

```
panic: strings.NewReplacer: odd argument count

goroutine 1 [running]:
strings.NewReplacer(...)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/strings/replace.go:34
main.main()
	/tmp/scratch/main.go:9 +0x30
exit status 2
```

It fires at construction time, so a package-level `var` with a typo crashes at
program start rather than in a request handler — which is the good case.

## strings.Reader

`NewReader` gives you an `io.Reader`, `io.ReaderAt`, `io.ByteReader`,
`io.ByteScanner`, `io.RuneReader`, `io.RuneScanner`, `io.Seeker`, and `io.WriterTo`
over a string, allocating only the small struct. This is how you feed a literal to
code that wants a stream.

```go
package main

import (
  "bufio"
  "encoding/json"
  "fmt"
  "io"
  "os"
  "strings"
)

func main() {
  r := strings.NewReader("hello 世界")
  fmt.Println(r.Len(), r.Size())

  buf := make([]byte, 5)
  n, err := r.Read(buf)
  fmt.Printf("%d %v %q len=%d\n", n, err, buf[:n], r.Len())

  b, _ := r.ReadByte()
  fmt.Printf("%q\n", b)
  ch, size, _ := r.ReadRune()
  fmt.Printf("%q %d\n", ch, size)
  r.UnreadRune()
  ch, _, _ = r.ReadRune()
  fmt.Printf("%q\n", ch)

  pos, _ := r.Seek(0, io.SeekStart)
  fmt.Println(pos, r.Len())

  at := make([]byte, 3)
  r.ReadAt(at, 6)
  fmt.Printf("%q\n", at)

  r.Seek(0, io.SeekStart)
  written, _ := r.WriteTo(os.Stdout)
  fmt.Printf("\nwrote %d\n", written)

  sc := bufio.NewScanner(strings.NewReader("l1\nl2\nl3"))
  for sc.Scan() {
    fmt.Printf("%q ", sc.Text())
  }
  fmt.Println()

  var v struct {
    A int    `json:"a"`
    B string `json:"b"`
  }
  json.NewDecoder(strings.NewReader(`{"a":1,"b":"x"}`)).Decode(&v)
  fmt.Printf("%+v\n", v)
}
```

Output:

```
12 12
5 <nil> "hello" len=7
' '
'世' 3
'世'
0 12
"世"
hello 世界
wrote 12
"l1" "l2" "l3" 
{A:1 B:x}
```

`Len()` is bytes **remaining**; `Size()` is the total and never changes. `ReadAt`
does not move the read cursor, which is what makes a `strings.Reader` usable as the
backing store for `io.SectionReader` or `archive/zip`. `Reset(s)` reuses the struct
for a new string without allocating.

Idiomatic: `strings.NewReader(s)` in tests wherever production code reads from a
file or a network body. For `[]byte` input use `bytes.NewReader` (6.5).

## Quick Reference

### strings.Builder

| Method | Signature | Note |
|---|---|---|
| `WriteString` | `(s string) (int, error)` | the common case; error always nil |
| `Write` | `(p []byte) (int, error)` | satisfies `io.Writer` |
| `WriteByte` | `(c byte) error` | |
| `WriteRune` | `(r rune) (int, error)` | UTF-8 encodes |
| `String` | `() string` | no copy |
| `Len` | `() int` | bytes written |
| `Cap` | `() int` | buffer capacity |
| `Grow` | `(n int)` | reserve n more bytes |
| `Reset` | `()` | drops the buffer; `Cap` returns to 0 |

### strings.Replacer

| Method | Signature | Note |
|---|---|---|
| `NewReplacer` | `(oldnew ...string) *Replacer` | panics on odd count |
| `Replace` | `(s string) string` | single pass; earliest pair wins |
| `WriteString` | `(w io.Writer, s string) (int, error)` | streams, no intermediate string |

### strings.Reader

| Method | Signature | Note |
|---|---|---|
| `NewReader` | `(s string) *Reader` | |
| `Read` | `([]byte) (int, error)` | `io.Reader` |
| `ReadAt` | `([]byte, int64) (int, error)` | does not move the cursor |
| `ReadByte` / `UnreadByte` | `() (byte, error)` / `() error` | `io.ByteScanner` |
| `ReadRune` / `UnreadRune` | `() (rune, int, error)` / `() error` | `io.RuneScanner` |
| `Seek` | `(int64, int) (int64, error)` | `io.Seeker` |
| `Len` | `() int` | bytes **remaining** |
| `Size` | `() int64` | total, constant |
| `WriteTo` | `(io.Writer) (int64, error)` | `io.WriterTo`; drains the reader |
| `Reset` | `(s string)` | reuse without allocating |

## Common Mistakes

**Copying a `strings.Builder` after writing to it.** It panics with "illegal use of
non-zero Builder copied by value"; pass `*strings.Builder`.

**Chaining `ReplaceAll` calls where later patterns can match earlier output.** Use
one `strings.Replacer`, which makes a single pass.

**Rebuilding a `Replacer` on every call.** Construction does the pattern-matching
setup work; make it a package-level `var` and reuse it — it is concurrency-safe.

**Expecting `Builder.Reset()` to keep capacity for reuse.** It drops the buffer; use
`bytes.Buffer` if you want to recycle one, or call `Grow` again.

**Reaching for `bytes.Buffer` just to build a string.** `strings.Builder` avoids the
final copy in `String()`; use `Buffer` only when you need to read back or want
`[]byte` out.

## Exercises

1. Write `func CSVLine(fields []string) string` that quotes any field containing a
   comma or quote, using one `strings.Builder` with a single `Grow` based on an
   estimated size. Benchmark against a `+=` version at 20 fields.
2. Prove the Builder copy panic three different ways: assignment, passing by value
   to a function, and copying a struct that embeds a Builder by value. Then make
   each one compile cleanly by switching to a pointer.
3. Build a `*strings.Replacer` for the five XML entities and compare it against
   five chained `ReplaceAll` calls on input containing `&amp;` already — explain
   the difference in output, not just the timing.
4. Write a function taking an `io.Reader` that returns the longest line. Test it
   with `strings.NewReader` for three fixtures, then run the same function against
   a real file via `os.Open` with no code changes.
5. Use `strings.NewReader` with `io.SectionReader` to expose bytes 3–8 of a string
   as its own `io.Reader`, and explain why this needs `ReadAt` rather than `Read`.

---

[← 6.3 The strings Package Reference](03-strings-package.md) · [Course Index](../README.md) · [6.5 The bytes Package Reference →](05-bytes-package.md)
