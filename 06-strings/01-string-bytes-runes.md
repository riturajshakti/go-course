[← 5.6 Standard Library Interfaces Worth Knowing](../05-interfaces/06-stdlib-interfaces.md) · [Course Index](../README.md) · [6.2 UTF-8, Indexing, and Iteration →](02-utf8-iteration.md)

# 6.1 — string, []byte, and []rune

## What You'll Learn

- A `string` is an immutable byte slice header (pointer + length), not an array of characters
- `s[i]` yields a `byte` (`uint8`), and `len(s)` counts bytes, not characters
- Which of `string`/`[]byte`/`[]rune` conversions allocate, and the two cases the compiler elides
- Raw vs interpreted literals, every escape form, and why `+=` in a loop is O(n²)

## Coming From Another Language

| Other | Go |
|---|---|
| JS `str[0]` — a 1-char string | `s[0]` — a `uint8` number |
| JS `str.length` — UTF-16 code units | `len(s)` — UTF-8 **bytes** |
| Python `str` (code points) vs `bytes` | `string` is bytes; `[]rune` is code points |
| Python `s is None` possible | a `string` is never nil; zero value is `""` |
| Rust `String`/`&str` (guaranteed UTF-8) | `string` is *conventionally* UTF-8, not validated |

Rust's `&str` is the closest analogue, with one critical difference: Rust guarantees
the bytes are valid UTF-8 and panics if you build one that isn't. Go does not
validate. A Go `string` is an arbitrary immutable byte sequence that the standard
library *interprets* as UTF-8 when you ask it to. Everything in this module follows
from that.

## A string is a byte slice with no write access

Indexing gives you a number. `%c` and `%q` are what turn it back into something
readable.

```go
package main

import "fmt"

func main() {
  s := "héllo"
  fmt.Println(s[0], s[1], s[2])
  fmt.Printf("%T %T\n", s[0], []rune(s)[0])
  fmt.Printf("%c %q %d %U\n", s[0], s[0], s[0], s[0])
  r := []rune(s)[1]
  fmt.Printf("%c %q %d %U\n", r, r, r, r)
}
```

Output:

```
104 195 169
uint8 int32
h 'h' 104 U+0068
é 'é' 233 U+00E9
```

`s[1]` is 195 and `s[2]` is 169 — together they are the two UTF-8 bytes of `é`.
Neither one alone is a character. `[]rune(s)[1]` is the code point 233.

## len is bytes

```go
package main

import (
  "fmt"
  "unicode/utf8"
)

func main() {
  s := "héllo"
  fmt.Println(len(s), utf8.RuneCountInString(s))
  fmt.Println(len([]byte(s)), len([]rune(s)))
  fmt.Println(len("日本語"), utf8.RuneCountInString("日本語"))
  fmt.Println(len(""), len("a"))
}
```

Output:

```
6 5
6 5
9 3
0 1
```

`len("héllo") == 6` with five characters. This is the single most common
off-by-some bug when validating "max 20 characters" against user input.
Idiomatic: `len(s)` for buffer sizes and byte offsets, `utf8.RuneCountInString(s)`
when a human would say "characters".

## Immutability

There is no write path into a string's bytes. This is a compile error, not a
runtime panic.

```go
package main

import "fmt"

func main() {
  s := "hello"
  s[0] = 'H'
  fmt.Println(s)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:3: cannot assign to s[0] (neither addressable nor a map index expression)
```

To mutate, convert to `[]byte`, change it, convert back — which copies twice.

## Conversions

`[]byte(s)` and `[]rune(s)` both produce fresh, independent, writable slices.
`string(bs)` copies back.

```go
package main

import "fmt"

func main() {
  s := "héllo"
  bs := []byte(s)
  rs := []rune(s)
  fmt.Println(bs)
  fmt.Println(rs)
  bs[0] = 'H'
  fmt.Println(s, string(bs))
  fmt.Println(string(rs), string(rs[1:]))
  fmt.Println(string(bs[0:2]), string([]byte{104, 105}))
  fmt.Println(string(rune(233)), string(byte(233)) == "\xe9")
}
```

Output:

```
[104 195 169 108 108 111]
[104 233 108 108 111]
héllo Héllo
héllo éllo
H� hi
é false
```

Two things to note. `string(bs[0:2])` is `"H\xc3"` — half of a rune, printed as the
replacement character. And `string(byte(233))` is **not** the single byte 233: the
byte is widened to `rune(233)` and then UTF-8 encoded into two bytes, so it does not
equal `"\xe9"`. `string(someInt)` converting a number to its code point rather than
its decimal text is the classic trap; `go vet` flags the `int` case but not `byte`.
Use `strconv.Itoa` for the decimal text.

| Conversion | Allocates? | Note |
|---|---|---|
| `[]byte(s)` | yes | fresh writable copy |
| `[]rune(s)` | yes | decodes UTF-8; O(len(s)) |
| `string(bs)` | yes | copies the bytes |
| `string(rs)` | yes | re-encodes to UTF-8 |
| `string(r)` where `r` is `rune` | small | UTF-8 encodes one code point |
| `s[i:j]` | **no** | shares the original bytes |

## The two conversions the compiler elides

A `[]byte(s)` whose result never escapes is optimized away. Ranging over one, and
using one as a map key, are the two cases worth knowing.

```go
package main

import "fmt"

func main() {
  s := "héllo"
  total := 0
  for _, b := range []byte(s) {
    total += int(b)
  }
  fmt.Println(total)

  counts := map[string]int{"héllo": 1}
  bs := []byte("héllo")
  fmt.Println(counts[string(bs)])

  fmt.Println(len(string(bs)))
}
```

Output:

```
795
1
6
```

Benchmarked, the difference is 1 allocation versus 0:

```
BenchmarkBytesConvEscaping-12     	117129032	        10.23 ns/op	      48 B/op	       1 allocs/op
BenchmarkRangeOverBytesConv-12    	25192749	        47.73 ns/op	       0 B/op	       0 allocs/op
BenchmarkMapLookupBytesKey-12     	203641864	         5.864 ns/op	       0 B/op	       0 allocs/op
```

Numbers are from `go test -bench . -benchmem` on this machine (darwin/arm64, Apple
M4 Pro) and will differ on yours; the allocation counts will not.

## Substring slicing shares memory

`s[i:j]` allocates nothing — it is a new header pointing into the same bytes. Safe,
because nobody can write through either one.

```go
package main

import "fmt"

func main() {
  big := "hello world, this is a long string"
  sub := big[0:5]
  fmt.Println(sub, len(sub))

  bs := []byte("hello")
  fmt.Println(string(bs[1:3]))
}
```

Output:

```
hello 5
el
```

The retention note still applies: a 5-byte substring of a 50 MB string keeps all
50 MB alive, because the header points into it. Break the link with
`strings.Clone(sub)` when the small piece outlives the big one.

## Concatenation and why += in a loop is O(n²)

Strings are immutable, so every `+` builds a brand-new string and copies both
operands. In a loop, round *k* copies *k* bytes.

```go
package main

import "fmt"

func main() {
  s := "a" + "b" + "c"
  fmt.Println(s)

  out := ""
  for i := 0; i < 5; i++ {
    out += "x"
  }
  fmt.Println(out, len(out))
}
```

Output:

```
abc
xxxxx 5
```

Correct, but at 1000 iterations it is 999 allocations and half a megabyte of copying:

```
BenchmarkPlusEquals-12     	   18517	     64932 ns/op	  530278 B/op	     999 allocs/op
BenchmarkBuilder-12        	  565959	      2098 ns/op	    3320 B/op	       9 allocs/op
```

A single-expression `a + b + c` is fine — the compiler sizes it once. It is the loop
that hurts. See 6.4 for `strings.Builder`.

## Literals: raw and interpreted

Backticks are raw: no escape processing, newlines are literal, and there is no way
to include a backtick.

```go
package main

import "fmt"

func main() {
  raw := `line one
  tab?	no: \n stays literal`
  fmt.Println(raw)
  fmt.Printf("%q\n", raw)

  interp := "tab:\there\nnewline above\\backslash \"quoted\" \x41 é \U0001F600 \101"
  fmt.Println(interp)
  fmt.Printf("%q\n", interp)
}
```

Output:

```
line one
	tab?	no: \n stays literal
"line one\n\ttab?\tno: \\n stays literal"
tab:	here
newline above\backslash "quoted" A é 😀 A
"tab:\there\nnewline above\\backslash \"quoted\" A é 😀 A"
```

Raw literals are what you want for regexes, JSON fixtures, and Windows paths.

| Escape | Meaning |
|---|---|
| `\n` `\r` `\t` `\v` `\f` `\b` `\a` | the usual control characters |
| `\\` `\"` | backslash, double quote (interpreted literals) |
| `\'` | single quote (rune literals only) |
| `\xFF` | one raw byte, two hex digits |
| `é` | code point, 4 hex digits |
| `\U0001F600` | code point, 8 hex digits |
| `\101` | one raw byte, three **octal** digits |

`\xFF` and `\377` insert a byte that may not be valid UTF-8; `\u`/`\U` always insert
a correctly encoded code point.

## byte and rune are aliases

`byte` is `uint8`, `rune` is `int32`. Rune literals use single quotes and are
untyped integer constants.

```go
package main

import "fmt"

func main() {
  var b byte = 'A'
  var r rune = '世'
  fmt.Printf("%T %d %c | %T %d %c\n", b, b, b, r, r, r)
  fmt.Println('A', '世', '\n', '\'')
  var s string
  fmt.Printf("%q %d %v\n", s, len(s), s == "")
}
```

Output:

```
uint8 65 A | int32 19990 世
65 19990 10 39
"" 0 true
```

`fmt.Println('A')` prints 65 because a rune *is* an integer. The zero value of
`string` is `""`, and `s == nil` would not even compile — strings are never nil, so
there is no nil-vs-empty distinction to worry about, unlike slices and maps.

## Comparison is byte order, not alphabetical order

`==`, `<`, `>` work directly on strings and compare bytes lexicographically.
Strings are comparable, so they work as map keys and in comparable structs.

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  fmt.Println("apple" < "banana", "Zebra" < "apple", "a" < "A")
  fmt.Println(strings.Compare("a", "b"), strings.Compare("b", "a"), strings.Compare("a", "a"))
  fmt.Println("é" > "z", "ä" < "b")

  m := map[string]int{"a": 1}
  type key struct{ s string }
  m2 := map[key]int{{"x"}: 2}
  fmt.Println(m["a"], m2[key{"x"}])
}
```

Output:

```
true true false
-1 1 0
true false
1 2
```

`"Zebra" < "apple"` is true because uppercase bytes sort before lowercase, and
`"é" > "z"` because its first byte is 0xC3. This is not locale-aware collation and
never will be; see 6.8.

Idiomatic: use `==` and `<` directly. `strings.Compare` exists only for when you
need the three-way `-1/0/1` for a sort comparator.

## The unsafe conversions

`unsafe.String` and `unsafe.Slice` can alias a `[]byte` as a `string` with no copy.
Reserve them for profiled hot paths where you can prove the bytes are never written
again, and expect reviewers to object.

## Quick Reference

| Expression | Type | Note |
|---|---|---|
| `s[i]` | `byte` | one byte, may be mid-rune |
| `s[i:j]` | `string` | no copy; shares bytes |
| `len(s)` | `int` | bytes |
| `utf8.RuneCountInString(s)` | `int` | code points |
| `[]byte(s)` / `[]rune(s)` | slice | copies |
| `string(bs)` / `string(rs)` | `string` | copies |
| `string(r)` (rune) | `string` | UTF-8 encodes one code point |
| `string(n)` (int) | `string` | code point, **not** decimal — use `strconv.Itoa` |
| `a + b` | `string` | new allocation each time |
| `` `raw` `` | `string` | no escapes, literal newlines, no backtick |

| Need | Use |
|---|---|
| mutate text | `[]byte`, or `strings.Builder` |
| random access by character | `[]rune(s)` once, then index |
| iterate characters | `for i, r := range s` (6.2) |
| build in a loop | `strings.Builder` (6.4) |
| detach a substring from a big string | `strings.Clone` |
| number to text | `strconv.Itoa` / `strconv.FormatInt` (6.6) |

## Common Mistakes

**Treating `len(s)` as a character count.** It is bytes; use
`utf8.RuneCountInString` when the limit is meant for humans.

**Using `string(n)` to render an integer.** It produces the code point with that
value; use `strconv.Itoa(n)`.

**Building a string with `+=` inside a loop.** Each round reallocates and copies
everything; use `strings.Builder`.

**Expecting `s[0]` to be a one-character string.** It is a `uint8`; print it with
`%c` or slice with `s[0:1]`.

**Holding a short substring of a huge string indefinitely.** The slice header pins
the whole original; copy it with `strings.Clone`.

## Exercises

1. Write `func Stats(s string) (bytes, runes int)` and run it over `"a"`, `"é"`,
   `"日本語"`, `"👍🏿"`, and `""`. Explain each pair of numbers in terms of UTF-8
   widths.
2. Write `func ToUpperASCII(s string) string` that uppercases only `a`–`z` by
   converting to `[]byte` and mutating in place. Confirm with `é` and `世` in the
   input that the multibyte runes survive untouched.
3. Benchmark `+=` against `strings.Builder` for 10, 100, 1000, and 10000
   iterations with `go test -bench . -benchmem`. Plot allocations against n and
   confirm the quadratic byte count.
4. Demonstrate the retention problem: build a 10 MB string, keep only `s[:10]`,
   force a GC, and compare `runtime.ReadMemStats` HeapAlloc against the same code
   using `strings.Clone`.
5. Write every escape form that produces the byte 0x41 (`A`) as a one-line
   program, and then write three literals that produce a byte sequence for which
   `utf8.ValidString` returns false.

---

[← 5.6 Standard Library Interfaces Worth Knowing](../05-interfaces/06-stdlib-interfaces.md) · [Course Index](../README.md) · [6.2 UTF-8, Indexing, and Iteration →](02-utf8-iteration.md)
