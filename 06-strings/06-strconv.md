[← 6.5 The bytes Package Reference](05-bytes-package.md) · [Course Index](../README.md) · [6.7 fmt: Every Verb and Flag →](07-fmt-verbs.md)

# 6.6 — strconv: Parsing and Formatting

## What You'll Learn

- Every `Parse*`, `Format*`, `Append*`, and `Quote*` function, with its real error text
- `ParseInt` base 0 auto-detects `0x`/`0o`/`0b`/leading-zero octal and allows `_`
- `*strconv.NumError`, the `ErrSyntax`/`ErrRange` sentinels, and `errors.Is` against them
- `ParseFloat("1e999")` returns `+Inf` **and** an error — the edge case that bites

## Coming From Another Language

| Other | Go |
|---|---|
| JS `parseInt("12abc")` → 12 | `strconv.Atoi("12abc")` → error; Go never partially parses |
| JS `Number("")` → 0 | `strconv.Atoi("")` → error |
| JS `+"1e999"` → Infinity, silently | `ParseFloat` → `+Inf` **with** `ErrRange` |
| Python `int(s)` raises | Go returns `(value, error)`; no exceptions |
| Python `int(s)` ignores surrounding space | Go rejects `" 42"`; trim first |
| Rust `s.parse::<i32>()` | `ParseInt(s, 10, 32)` — same explicitness |

There is no implicit string-to-number conversion anywhere in Go. Every parse is a
call that returns an error you must handle, and every format is a call that names
the base and precision. That verbosity is the feature.

## Integers

| Function | Signature | Returns | Note |
|---|---|---|---|
| `Atoi` | `(s string) (int, error)` | `int` | exactly `ParseInt(s, 10, 0)` with a different error prefix |
| `Itoa` | `(i int) string` | `string` | exactly `FormatInt(int64(i), 10)` |
| `ParseInt` | `(s string, base, bitSize int) (int64, error)` | `int64` | `base == 0` auto-detects; `bitSize == 0` means `int` |
| `ParseUint` | `(s string, base, bitSize int) (uint64, error)` | `uint64` | rejects a leading `-` |
| `FormatInt` | `(i int64, base int) string` | `string` | base 2–36, lowercase digits |
| `FormatUint` | `(i uint64, base int) string` | `string` | |
| `AppendInt` | `(dst []byte, i int64, base int) []byte` | `[]byte` | no intermediate string |
| `AppendUint` | `(dst []byte, i uint64, base int) []byte` | `[]byte` | |
| `IntSize` | `const` | 32 or 64 | bits in an `int` on this platform |

```go
package main

import (
  "fmt"
  "strconv"
)

func main() {
  i, err := strconv.Atoi("-42")
  fmt.Println(i, err)
  fmt.Println(strconv.Itoa(-42), strconv.IntSize)

  for _, in := range []string{"42", "0x2a", "0o52", "0b101010", "1_000", "052"} {
    v, err := strconv.ParseInt(in, 0, 64)
    fmt.Printf("base0 %-10q -> %v %v\n", in, v, err)
  }
  v, err := strconv.ParseInt("ff", 16, 64)
  fmt.Println(v, err)
  v, err = strconv.ParseInt("200", 10, 8)
  fmt.Println(v, err)
  u, err := strconv.ParseUint("255", 10, 8)
  fmt.Println(u, err)
  u, err = strconv.ParseUint("-1", 10, 64)
  fmt.Println(u, err)

  fmt.Println(strconv.FormatInt(-255, 16), strconv.FormatInt(255, 2), strconv.FormatInt(255, 36))
  fmt.Println(strconv.FormatUint(255, 8))
  dst := []byte("n=")
  dst = strconv.AppendInt(dst, -99, 10)
  dst = strconv.AppendUint(dst, 7, 10)
  fmt.Printf("%q\n", dst)
}
```

Output:

```
-42 <nil>
-42 64
base0 "42"       -> 42 <nil>
base0 "0x2a"     -> 42 <nil>
base0 "0o52"     -> 42 <nil>
base0 "0b101010" -> 42 <nil>
base0 "1_000"    -> 1000 <nil>
base0 "052"      -> 42 <nil>
255 <nil>
127 strconv.ParseInt: parsing "200": value out of range
255 <nil>
0 strconv.ParseUint: parsing "-1": invalid syntax
-ff 11111111 73
377
"n=-997"
```

Base 0 accepts exactly what a Go source literal accepts: the three modern prefixes,
C-style leading-zero octal (`"052"` is 42, not 52 — a real hazard for zero-padded
IDs), and `_` digit separators. Underscores are **only** permitted when base is 0.

Note what `ParseInt("200", 10, 8)` returns: the error *and* 127, the clamped
maximum. A range error always comes with the saturated value, so ignoring the error
gives you a plausible-looking wrong number.

Idiomatic: `Atoi` for plain decimal `int`. Reach for `ParseInt` when you need a
specific width, a specific base, or auto-detection.

## Floats

| Function | Signature | Returns | Note |
|---|---|---|---|
| `ParseFloat` | `(s string, bitSize int) (float64, error)` | `float64` | `bitSize` 32 rounds to float32 precision |
| `FormatFloat` | `(f float64, fmt byte, prec, bitSize int) string` | `string` | `prec == -1` means shortest round-trip |
| `AppendFloat` | `(dst []byte, f float64, fmt byte, prec, bitSize int) []byte` | `[]byte` | |
| `ParseComplex` / `FormatComplex` | — | | rarely needed; same shape |

```go
package main

import (
  "fmt"
  "strconv"
)

func main() {
  f, err := strconv.ParseFloat("3.1415", 64)
  fmt.Println(f, err)
  f32, err := strconv.ParseFloat("3.1415926535", 32)
  fmt.Println(f32, float32(f32), err)
  fmt.Println(strconv.ParseFloat("1e999", 64))
  fmt.Println(strconv.ParseFloat("-1e999", 64))
  fmt.Println(strconv.ParseFloat("1e-999", 64))
  fmt.Println(strconv.ParseFloat("NaN", 64))
  fmt.Println(strconv.ParseFloat("Inf", 64))
  fmt.Println(strconv.ParseFloat("0x1p-2", 64))

  x := 1234.5678
  for _, fc := range []byte{'f', 'e', 'E', 'g', 'G', 'b', 'x', 'X'} {
    fmt.Printf("%c prec=-1 %-28s prec=3 %s\n", fc,
      strconv.FormatFloat(x, fc, -1, 64),
      strconv.FormatFloat(x, fc, 3, 64))
  }
  fmt.Println(strconv.FormatFloat(0.1, 'f', -1, 32), strconv.FormatFloat(0.1, 'f', -1, 64))
  fmt.Printf("%q\n", strconv.AppendFloat([]byte("f="), 2.5, 'f', -1, 64))
}
```

Output:

```
3.1415 <nil>
3.1415927410125732 3.1415927 <nil>
+Inf strconv.ParseFloat: parsing "1e999": value out of range
-Inf strconv.ParseFloat: parsing "-1e999": value out of range
0 <nil>
NaN <nil>
+Inf <nil>
0.25 <nil>
f prec=-1 1234.5678                    prec=3 1234.568
e prec=-1 1.2345678e+03                prec=3 1.235e+03
E prec=-1 1.2345678E+03                prec=3 1.235E+03
g prec=-1 1234.5678                    prec=3 1.23e+03
G prec=-1 1234.5678                    prec=3 1.23E+03
b prec=-1 5429686605511341p-42         prec=3 5429686605511341p-42
x prec=-1 0x1.34a456d5cfaadp+10        prec=3 0x1.34ap+10
X prec=-1 0X1.34A456D5CFAADP+10        prec=3 0X1.34AP+10
0.1 0.1
"f=2.5"
```

Four things worth pulling out.

**Overflow returns a value and an error.** `ParseFloat("1e999", 64)` gives `+Inf`
*and* `ErrRange`. Code that writes `f, _ := strconv.ParseFloat(s, 64)` quietly
propagates an infinity into arithmetic, where it poisons every later sum. Underflow
(`1e-999`) returns 0 with **no** error.

**`"NaN"` and `"Inf"` parse successfully.** So do `"+Inf"`, `"-Inf"`, `"infinity"`,
and any case variation. If your config format should not accept them, check
explicitly with `math.IsNaN` / `math.IsInf`.

**`bitSize: 32` does not change the return type.** You still get a `float64`, but
rounded as if stored in a float32 — hence `3.1415927410125732`. Convert with
`float32(f)` afterwards.

**`prec == -1` is almost always what you want.** It emits the fewest digits that
parse back to the identical bit pattern, which is why `FormatFloat(0.1, 'f', -1, 64)`
is `"0.1"` and not `"0.100000000000000006"`.

| `fmt` byte | Form | `1234.5678`, prec -1 |
|---|---|---|
| `'f'` | no exponent | `1234.5678` |
| `'e'` / `'E'` | decimal exponent | `1.2345678e+03` |
| `'g'` / `'G'` | `%e` for large exponents, else `%f` | `1234.5678` |
| `'b'` | binary exponent, mantissa p exp | `5429686605511341p-42` |
| `'x'` / `'X'` | hex mantissa, binary exponent | `0x1.34a456d5cfaadp+10` |

`'b'` and `'x'` are exact — they round-trip every float64 bit pattern with no
decimal rounding at all, which makes them the right choice for test fixtures and
wire formats that must be bit-identical. Note `'b'` ignores `prec` entirely.

## Bools

| Function | Signature | Accepts / produces |
|---|---|---|
| `ParseBool` | `(str string) (bool, error)` | exactly `1 t T TRUE true True 0 f F FALSE false False` |
| `FormatBool` | `(b bool) string` | `"true"` / `"false"` |
| `AppendBool` | `(dst []byte, b bool) []byte` | |

```go
package main

import (
  "fmt"
  "strconv"
)

func main() {
  for _, s := range []string{"1", "t", "T", "TRUE", "true", "True", "0", "f", "F", "FALSE", "false", "False", "yes", "", "TrUe"} {
    b, err := strconv.ParseBool(s)
    fmt.Printf("%-7q -> %-5v %v\n", s, b, err)
  }
  fmt.Println(strconv.FormatBool(true), strconv.FormatBool(false))
  fmt.Printf("%q\n", strconv.AppendBool([]byte("b="), true))
}
```

Output:

```
"1"     -> true  <nil>
"t"     -> true  <nil>
"T"     -> true  <nil>
"TRUE"  -> true  <nil>
"true"  -> true  <nil>
"True"  -> true  <nil>
"0"     -> false <nil>
"f"     -> false <nil>
"F"     -> false <nil>
"FALSE" -> false <nil>
"false" -> false <nil>
"False" -> false <nil>
"yes"   -> false strconv.ParseBool: parsing "yes": invalid syntax
""      -> false strconv.ParseBool: parsing "": invalid syntax
"TrUe"  -> false strconv.ParseBool: parsing "TrUe": invalid syntax
```

Twelve exact strings, nothing else. `"yes"`, `"on"`, `"y"`, and mixed case like
`"TrUe"` are all errors, and the returned bool on error is `false` — so
`b, _ := strconv.ParseBool(userInput)` silently treats every unrecognized value as
false. That is the wrong default for a feature flag.

## Quoting

| Function | Signature | Note |
|---|---|---|
| `Quote` | `(s string) string` | Go double-quoted literal; keeps printable non-ASCII |
| `QuoteToASCII` | `(s string) string` | escapes all non-ASCII as `\u`/`\U` |
| `QuoteToGraphic` | `(s string) string` | keeps graphic runes, escapes the rest |
| `QuoteRune` | `(r rune) string` | single-quoted rune literal |
| `QuoteRuneToASCII` | `(r rune) string` | |
| `QuoteRuneToGraphic` | `(r rune) string` | |
| `AppendQuote` + `…ToASCII` / `…ToGraphic` / `…Rune*` | `(dst []byte, …) []byte` | all six have Append forms |
| `Unquote` | `(s string) (string, error)` | accepts `"…"`, `` `…` ``, and `'…'` |
| `UnquoteChar` | `(s string, quote byte) (rune, bool, string, error)` | one escape at a time |
| `QuotedPrefix` | `(s string) (string, error)` | the leading quoted literal, ignoring the rest |
| `CanBackquote` | `(s string) bool` | safe inside backticks |
| `IsPrint` | `(r rune) bool` | Go's "printable", matches `Quote`'s decision |
| `IsGraphic` | `(r rune) bool` | includes non-breaking spaces |

```go
package main

import (
  "fmt"
  "strconv"
)

func main() {
  s := "Hi\t\"世界\"\x00"
  fmt.Println(strconv.Quote(s))
  fmt.Println(strconv.QuoteToASCII(s))
  fmt.Println(strconv.QuoteToGraphic(s))
  fmt.Println(strconv.QuoteRune('世'), strconv.QuoteRuneToASCII('世'), strconv.QuoteRuneToGraphic('世'))
  fmt.Println(strconv.QuoteRune('\n'), strconv.QuoteRune('\''))

  u, err := strconv.Unquote(`"a\tbé"`)
  fmt.Printf("%q %v\n", u, err)
  u, err = strconv.Unquote("`raw\\n`")
  fmt.Printf("%q %v\n", u, err)
  u, err = strconv.Unquote("'x'")
  fmt.Printf("%q %v\n", u, err)
  u, err = strconv.Unquote("no quotes")
  fmt.Printf("%q %v\n", u, err)

  val, multi, tail, err := strconv.UnquoteChar(`érest"`, '"')
  fmt.Printf("%q %v %q %v\n", val, multi, tail, err)

  fmt.Println(strconv.CanBackquote("plain"), strconv.CanBackquote("has`tick"), strconv.CanBackquote("nl\n"))
  fmt.Println(strconv.IsPrint('a'), strconv.IsPrint('\n'), strconv.IsPrint('世'))
  fmt.Println(strconv.IsGraphic(' '), strconv.IsPrint(' '), strconv.IsGraphic(' '))
  fmt.Printf("%s\n", strconv.AppendQuote([]byte("q="), "a\tb"))
  fmt.Println(strconv.QuotedPrefix(`"abc" trailing`))
}
```

Output:

```
"Hi\t\"世界\"\x00"
"Hi\t\"世界\"\x00"
"Hi\t\"世界\"\x00"
'世' '世' '世'
'\n' '\''
"a\tbé" <nil>
"raw\\n" <nil>
"x" <nil>
"" invalid syntax
'é' true "rest\"" <nil>
true false false
true false true
true false true
q="a\tb"
"abc" <nil>
```

`Unquote` on an unquoted string returns the bare `ErrSyntax` (`invalid syntax`), not
a `*NumError` — the quoting functions are the exception to the wrapping convention.
Note `strconv.Quote` is what `%q` uses, and `IsPrint(' ')` is false while
`IsGraphic` is true: a non-breaking space is a graphic character but not one Go will
leave unescaped, which is how `%q` reveals invisible whitespace in debugging.

## Errors

Every `Parse*` failure (except `Unquote`) returns a `*strconv.NumError` wrapping one
of two sentinels.

```go
type NumError struct {
  Func string // "ParseInt", "ParseFloat", "Atoi", …
  Num  string // the input
  Err  error  // ErrSyntax or ErrRange
}
```

```go
package main

import (
  "errors"
  "fmt"
  "strconv"
)

func main() {
  for _, in := range []string{"abc", "", "12.5", " 42", "42 ", "99999999999999999999"} {
    _, err := strconv.Atoi(in)
    fmt.Printf("Atoi(%-24q) err = %v\n", in, err)
  }
  fmt.Println()

  _, err := strconv.ParseInt("300", 10, 8)
  fmt.Println(err)
  var ne *strconv.NumError
  if errors.As(err, &ne) {
    fmt.Printf("Func=%q Num=%q Err=%v\n", ne.Func, ne.Num, ne.Err)
  }
  fmt.Println(errors.Is(err, strconv.ErrRange), errors.Is(err, strconv.ErrSyntax))

  _, err2 := strconv.Atoi("abc")
  fmt.Println(errors.Is(err2, strconv.ErrSyntax), errors.Is(err2, strconv.ErrRange))
  fmt.Println(errors.Unwrap(err2) == strconv.ErrSyntax)

  v, err3 := strconv.ParseFloat("1e999", 64)
  fmt.Println(v, errors.Is(err3, strconv.ErrRange))
  fmt.Println(strconv.ErrSyntax, "|", strconv.ErrRange)
}
```

Output:

```
Atoi("abc"                   ) err = strconv.Atoi: parsing "abc": invalid syntax
Atoi(""                      ) err = strconv.Atoi: parsing "": invalid syntax
Atoi("12.5"                  ) err = strconv.Atoi: parsing "12.5": invalid syntax
Atoi(" 42"                   ) err = strconv.Atoi: parsing " 42": invalid syntax
Atoi("42 "                   ) err = strconv.Atoi: parsing "42 ": invalid syntax
Atoi("99999999999999999999"  ) err = strconv.Atoi: parsing "99999999999999999999": value out of range

strconv.ParseInt: parsing "300": value out of range
Func="ParseInt" Num="300" Err=value out of range
true false
true false
true
+Inf true
invalid syntax | value out of range
```

The error string format is `strconv.<Func>: parsing <quoted input>: <reason>`.
`Atoi` reports itself as `strconv.Atoi` even though it delegates to `ParseInt`,
while `ParseInt("300", 10, 8)` reports `ParseInt` — so you cannot pattern-match the
prefix reliably. Note that `" 42"` and `"42 "` both fail: Go does not trim. And
`"12.5"` is a syntax error for an integer parse, not a truncation.

Idiomatic error handling at an input boundary:

```go
package main

import (
  "errors"
  "fmt"
  "strconv"
  "strings"
)

func parsePort(s string) (int, error) {
  n, err := strconv.Atoi(strings.TrimSpace(s))
  switch {
  case errors.Is(err, strconv.ErrSyntax):
    return 0, fmt.Errorf("port %q is not a number", s)
  case errors.Is(err, strconv.ErrRange):
    return 0, fmt.Errorf("port %q is too large", s)
  case err != nil:
    return 0, err
  case n < 1 || n > 65535:
    return 0, fmt.Errorf("port %d out of range 1-65535", n)
  }
  return n, nil
}

func main() {
  for _, in := range []string{" 8080 ", "abc", "99999999999999999999", "70000", "80"} {
    n, err := parsePort(in)
    fmt.Printf("%-24q -> %-6d %v\n", in, n, err)
  }
}
```

Output:

```
" 8080 "                 -> 8080   <nil>
"abc"                    -> 0      port "abc" is not a number
"99999999999999999999"   -> 0      port "99999999999999999999" is too large
"70000"                  -> 0      port 70000 out of range 1-65535
"80"                     -> 80     <nil>
```

This is where errors actually matter — parsing config, query parameters, and request
bodies. See 16.4 for request parsing and 16.8 for validation.

## strconv versus fmt

```
BenchmarkItoa-12        	100000000	        10.08 ns/op	       8 B/op	       1 allocs/op
BenchmarkSprintfD-12    	42598698	        28.31 ns/op	       8 B/op	       1 allocs/op
BenchmarkAtoi-12        	281680387	         4.111 ns/op	       0 B/op	       0 allocs/op
BenchmarkSscanfD-12     	 6641258	       178.9 ns/op	      72 B/op	       3 allocs/op
```

`Itoa` is ~3x faster than `Sprintf("%d", …)` and `Atoi` ~44x faster than
`Sscanf`, because `fmt` goes through reflection on an `any` argument. Machine-specific
numbers (darwin/arm64, Apple M4 Pro). Idiomatic: `strconv` whenever you are
converting a single value; `fmt` when you are assembling a message.

## Quick Reference

| Need | Use |
|---|---|
| decimal string → `int` | `Atoi` |
| `int` → decimal string | `Itoa` |
| hex/octal/binary, or a width limit | `ParseInt(s, base, bitSize)` |
| auto-detect `0x`/`0o`/`0b` | `ParseInt(s, 0, 64)` |
| `int` → other base | `FormatInt(i, base)` |
| float from text | `ParseFloat(s, 64)` |
| float → shortest exact text | `FormatFloat(f, 'f', -1, 64)` |
| float → exact bits | `FormatFloat(f, 'x', -1, 64)` |
| bool from text | `ParseBool` (12 accepted strings only) |
| append without an intermediate string | `AppendInt` / `AppendFloat` / `AppendBool` / `AppendQuote` |
| escape for display or a log | `Quote` (or `%q`) |
| parse a Go literal | `Unquote` |
| tell syntax from range errors | `errors.Is(err, strconv.ErrSyntax / ErrRange)` |

| Gotcha | Reality |
|---|---|
| `ParseFloat("1e999")` | returns `+Inf` **and** `ErrRange` |
| `ParseFloat("1e-999")` | returns 0, **no** error |
| `ParseFloat("NaN"/"Inf")` | succeeds |
| `ParseInt("200", 10, 8)` | returns 127 **and** `ErrRange` |
| `ParseInt("052", 0, 64)` | 42 — leading zero is octal |
| `_` separators | only when `base == 0` |
| `Atoi(" 42")` | error; Go does not trim |
| `ParseBool("yes")` | error, and returns `false` |
| `Atoi` error prefix | `strconv.Atoi`, not `strconv.ParseInt` |

## Common Mistakes

**Discarding the error from `ParseFloat`.** Overflow hands back `±Inf` along with
`ErrRange` and poisons downstream arithmetic.

**Assuming `ParseInt` with base 0 treats `"052"` as fifty-two.** The leading zero
means octal; pass base 10 explicitly for user-entered numbers.

**Passing untrimmed input to `Atoi`.** Surrounding whitespace is a syntax error;
`strings.TrimSpace` first.

**Using `ParseBool` on `"yes"`/`"on"` style config.** Only twelve exact strings
parse; map your own vocabulary before calling it.

**Reaching for `fmt.Sprintf("%d", n)` in a hot loop.** `strconv.Itoa` is ~3x
faster and skips the reflection path.

## Exercises

1. Write `func ParseAny(s string) (any, error)` that tries `ParseInt`, then
   `ParseFloat`, then `ParseBool`, then falls back to the string, and tabulate its
   result for `"42"`, `"42.0"`, `"0x2a"`, `"true"`, `"1"`, `""`, and `"1e999"`.
   Decide what `"1"` should be and defend it.
2. Show every `FormatFloat` verb for `math.Pi`, `0.1`, `1e308`, `math.SmallestNonzeroFloat64`,
   and `math.NaN()` in one table, then prove `prec: -1` round-trips all of them
   through `ParseFloat` exactly.
3. Write a table-driven test asserting the **exact** error strings for `Atoi("")`,
   `Atoi("x")`, `ParseInt("300",10,8)`, and `ParseUint("-1",10,64)`, and explain
   why asserting on error strings is usually the wrong choice.
4. Build `func EnvInt(key string, def int) (int, error)` that distinguishes unset
   from malformed from out-of-range, using `errors.Is` against both sentinels, and
   test it with `t.Setenv`.
5. Benchmark `AppendInt` into a reused `[]byte` against `Itoa` plus a
   `strings.Builder` when serializing 10000 integers as a CSV line, with
   `-benchmem`.

---

[← 6.5 The bytes Package Reference](05-bytes-package.md) · [Course Index](../README.md) · [6.7 fmt: Every Verb and Flag →](07-fmt-verbs.md)
