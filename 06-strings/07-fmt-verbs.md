[← 6.6 strconv: Parsing and Formatting](06-strconv.md) · [Course Index](../README.md) · [6.8 unicode, utf8, and Text Edge Cases →](08-unicode-utf8.md)

# 6.7 — fmt: Every Verb and Flag

## What You'll Learn

- Every verb for every kind, every flag, width/precision from args, and argument indexes
- `%v` vs `%+v` vs `%#v`, and why `%#v` ignores your `String()` method
- Every malformed-verb marker — `%!d(string=…)`, `%!s(MISSING)`, `%!(EXTRA …)`, `%!v(PANIC=…)` — captured from real runs
- `Print` vs `Println` spacing, and `go vet`'s real Printf diagnostics

## Coming From Another Language

| Other | Go |
|---|---|
| Python `f"{x}"` / `"{}".format(x)` | `fmt.Sprintf("%v", x)` — no interpolation syntax in Go |
| Python `%r` | `%#v` (Go syntax) or `%q` (quoted string) |
| JS `console.log(obj)` — deep inspect | `%+v` for field names, `%#v` for Go syntax |
| JS template literals | `fmt.Sprintf`, or `+` for simple cases |
| Rust `{:?}` / `{:#?}` | `%+v` / no stdlib pretty-printer |
| Rust `Display` / `Debug` | `String() string` / no separate Debug |

## The printing functions

| Function | Signature | Destination | Spacing |
|---|---|---|---|
| `Print` | `(a ...any) (int, error)` | stdout | space **only between two non-strings** |
| `Printf` | `(format string, a ...any) (int, error)` | stdout | you control it |
| `Println` | `(a ...any) (int, error)` | stdout | space between all, trailing `\n` |
| `Sprint` / `Sprintf` / `Sprintln` | `… string` | a string | same three rules |
| `Fprint` / `Fprintf` / `Fprintln` | `(w io.Writer, …)` | any writer | same three rules |
| `Append` / `Appendf` / `Appendln` | `(b []byte, …) []byte` | appends to `b` | same three rules |
| `Errorf` | `(format string, a ...any) error` | an `error` | `%w` wraps — see 8.2 |
| `Sscan` / `Sscanf` / `Sscanln` | `(str string, a ...any) (int, error)` | parses from a string | |
| `Fscan` / `Fscanf` / `Fscanln` | `(r io.Reader, …)` | parses from a reader | |
| `Scan` / `Scanf` / `Scanln` | `(a ...any) (int, error)` | parses from stdin | |

## Integers

```go
package main

import "fmt"

func main() {
  n := 200
  r := rune(n)
  fmt.Printf("%%v  %v\n", n)
  fmt.Printf("%%d  %d\n", n)
  fmt.Printf("%%b  %b\n", n)
  fmt.Printf("%%o  %o\n", n)
  fmt.Printf("%%O  %O\n", n)
  fmt.Printf("%%x  %x\n", n)
  fmt.Printf("%%X  %X\n", n)
  fmt.Printf("%%c  %c\n", n)
  fmt.Printf("%%q  %q\n", r)
  fmt.Printf("%%U  %U\n", n)
  fmt.Printf("%%#U %#U\n", n)
  fmt.Printf("%%T  %T\n", n)
  fmt.Printf("%%#x %#x | %%#o %#o | %%#b %#b\n", n, n, n)
  fmt.Printf("%%+d %+d | %%+d %+d\n", n, -n)
}
```

Output:

```
%v  200
%d  200
%b  11001000
%o  310
%O  0o310
%x  c8
%X  C8
%c  È
%q  'È'
%U  U+00C8
%#U U+00C8 'È'
%T  int
%#x 0xc8 | %#o 0310 | %#b 0b11001000
%+d +200 | %+d -200
```

`%q` on an integer produces a **rune literal**, not a quoted number — and `go vet`
rejects `%q` on an `int` while accepting it on a `rune`, which is why this program
converts first.

## Floats

```go
package main

import "fmt"

func main() {
  f := 1234.5678
  for _, v := range []string{"%v", "%f", "%F", "%e", "%E", "%g", "%G", "%b", "%x", "%X", "%T"} {
    fmt.Printf("%-4s %s\n", v, fmt.Sprintf(v, f))
  }
  fmt.Printf("%.2f %8.2f %-8.2f| %08.2f %+.2f\n", f, f, f, f, f)
  fmt.Printf("%.0f %.10f\n", f, f)
  fmt.Printf("%v %v %v\n", 1e21, 1e20, 0.000001)
}
```

Output:

```
%v   1234.5678
%f   1234.567800
%F   1234.567800
%e   1.234568e+03
%E   1.234568E+03
%g   1234.5678
%G   1234.5678
%b   5429686605511341p-42
%x   0x1.34a456d5cfaadp+10
%X   0X1.34A456D5CFAADP+10
%T   float64
1234.57  1234.57 1234.57 | 01234.57 +1234.57
1235 1234.5678000000
1e+21 1e+20 1e-06
```

`%v` for a float is `%g`: shortest round-trip, switching to an exponent for large
and small magnitudes. So `%v` on 1e21 prints `1e+21` and on 1e20 prints `1e+20` —
if you need a fixed-point column, say `%.2f` explicitly. Default `%f` precision is
6 digits, unlike `%v`.

## Strings, bytes, booleans, runes

```go
package main

import "fmt"

func main() {
  s := "héllo"
  bs := []byte(s)
  for _, v := range []string{"%v", "%s", "%q", "%x", "%X", "%T"} {
    fmt.Printf("%-4s string %-18s bytes %s\n", v, fmt.Sprintf(v, s), fmt.Sprintf(v, bs))
  }
  fmt.Printf("%% x %x | %%% x\n", s, s)
  fmt.Printf("%#q %+q\n", s, s)
  fmt.Printf("[%10s][%-10s][%.3s][%10.3s]\n", s, s, s, s)
  fmt.Printf("%t %t\n", true, false)
  fmt.Printf("%v %q %c %U\n", 'A', 'A', 'A', 'A')
}
```

Output:

```
%v   string héllo              bytes [104 195 169 108 108 111]
%s   string héllo              bytes héllo
%q   string "héllo"            bytes "héllo"
%x   string 68c3a96c6c6f       bytes 68c3a96c6c6f
%X   string 68C3A96C6C6F       bytes 68C3A96C6C6F
%T   string string             bytes []uint8
% x 68c3a96c6c6f | %68 c3 a9 6c 6c 6f
`héllo` "héllo"
[     héllo][héllo     ][hél][       hél]
true false
65 'A' A U+0041
```

`%v` on a `[]byte` prints the numbers; `%s` prints it as text. The space flag with
`%x` inserts a separator between bytes — handy for hex dumps. `%#q` prefers
backquotes, `%+q` forces ASCII-only escaping. Precision on a string counts
**runes**, not bytes: `%.3s` of `"héllo"` is `"hél"`, which is 4 bytes, and `%.3s`
of `"日本語語"` is `"日本語"`. Width, by contrast, pads to a count of runes too — so
neither aligns correctly for CJK or emoji, whose display width is 2 columns.

## Structs, maps, slices, pointers

```go
package main

import "fmt"

type Point struct {
  X, Y int
  name string
}

func main() {
  p := Point{1, 2, "origin"}
  fmt.Printf("%v\n%+v\n%#v\n%T\n", p, p, p, p)

  pp := &p
  fmt.Printf("%v\n%+v\n", pp, pp)

  sl := []int{1, 2, 3}
  fmt.Printf("%v %+v %#v %d\n", sl, sl, sl, sl)

  m := map[string]int{"z": 26, "a": 1, "m": 13}
  fmt.Printf("%v\n%#v\n", m, m)

  nested := map[string][]Point{"k": {p}}
  fmt.Printf("%v\n%+v\n", nested, nested)

  arr := [2]string{"a", "b"}
  fmt.Printf("%v %q %#v\n", arr, arr, arr)
}
```

Output:

```
{1 2 origin}
{X:1 Y:2 name:origin}
main.Point{X:1, Y:2, name:"origin"}
main.Point
&{1 2 origin}
&{X:1 Y:2 name:origin}
[1 2 3] [1 2 3] []int{1, 2, 3} [1 2 3]
map[a:1 m:13 z:26]
map[string]int{"a":1, "m":13, "z":26}
map[k:[{1 2 origin}]]
map[k:[{X:1 Y:2 name:origin}]]
[a b] ["a" "b"] [2]string{"a", "b"}
```

Three things. **`fmt` prints unexported fields** — `name:"origin"` appears in all
three forms, because `fmt` uses reflection to read them even though your code
cannot. Do not `%+v` a struct holding a password or token.

**Map keys are printed in sorted order** since Go 1.12. That is deterministic and
safe to assert on in tests, unlike `range` over a map, whose order is deliberately
randomized. For mixed-type keys `fmt` uses a defined ordering (nils, then
booleans, numbers, strings, …).

**A pointer to a struct prints as `&{…}`** at the top level, but a pointer nested
deeper prints as a hex address — `%v` only follows one level of indirection.

Verbs apply recursively: `%d` on a slice formats each element, and `%+v` propagates
through maps and slices to nested structs.

## Flags, width, precision, argument indexes

| Flag | Meaning |
|---|---|
| `+` | always show sign (`%+d`); ASCII-only (`%+q`); field names (`%+v`) |
| `-` | left-align within the width |
| `#` | alternate form: `0x`/`0`/`0b` prefixes, Go syntax (`%#v`), backquotes (`%#q`) |
| ` ` (space) | space for the sign of positive numbers; separator between hex bytes |
| `0` | pad with leading zeros instead of spaces (after any sign) |

| Form | Meaning |
|---|---|
| `%6d` | minimum width 6 |
| `%.2f` | precision 2 |
| `%6.2f` | both |
| `%*d` | width taken from the **next argument** |
| `%.*f` | precision taken from the next argument |
| `%[2]s` | use argument 2; later verbs continue from 3 |
| `%%` | a literal percent sign |

```go
package main

import "fmt"

func main() {
  fmt.Printf("[%6d][%-6d][%06d][%+6d][% d]\n", 42, 42, 42, 42, 42)
  fmt.Printf("[%*d][%-*d][%.*f]\n", 6, 42, 6, 42, 2, 3.14159)
  fmt.Printf("[%[2]s %[1]s]\n", "world", "hello")
  fmt.Printf("[%[1]d %[1]x %[1]b]\n", 255)
  fmt.Printf("[%6.2f][%-6.2f][%06.2f]\n", 3.14159, 3.14159, 3.14159)
  fmt.Printf("100%% done\n")
  fmt.Printf("[% x][%#v]\n", []byte("ab"), "s")
}
```

Output:

```
[    42][42    ][000042][   +42][ 42]
[    42][42    ][3.14]
[hello world]
[255 ff 11111111]
[  3.14][3.14  ][003.14]
100% done
[61 62]["s"]
```

`%[1]d %[1]x %[1]b` printing one value three ways without repeating the argument is
the most useful thing in this table.

## Stringer and Formatter dispatch

```go
package main

import "fmt"

type Temp float64

func (t Temp) String() string { return fmt.Sprintf("%.1f°C", float64(t)) }

type Bad struct{ N int }

func (b Bad) String() string { panic("boom") }

type Err struct{ Code int }

func (e Err) String() string { return "stringer" }
func (e Err) Error() string  { return fmt.Sprintf("error code %d", e.Code) }

type Hex int

func (h Hex) Format(f fmt.State, verb rune) {
  switch verb {
  case 'v', 'd':
    fmt.Fprintf(f, "Hex(%#x)", int(h))
  default:
    fmt.Fprintf(f, "%%!%c(Hex)", verb)
  }
}

func main() {
  t := Temp(21.456)
  fmt.Printf("%v | %s | %#v | %T | %f\n", t, t, t, t, float64(t))
  fmt.Println(t)

  e := Err{404}
  fmt.Printf("%v | %s\n", e, e)

  h := Hex(255)
  fmt.Printf("%v | %d | %s\n", h, h, h)

  fmt.Printf("%v\n", Bad{1})
}
```

Output:

```
21.5°C | 21.5°C | 21.456 | main.Temp | 21.456000
21.5°C
error code 404 | error code 404
Hex(0xff) | Hex(0xff) | %!s(Hex)
%!v(PANIC=String method: boom)
```

The dispatch order is: `Formatter` (full control over every verb), then — for
`%v`/`%s`/`%q`/`%x`/`%X` — `error`'s `Error()`, then `Stringer`'s `String()`, then
reflection.

**`error` beats `Stringer`.** `Err` has both; `%v` and `%s` print `"error code
404"`, and `"stringer"` is unreachable. Never implement both on one type.

**`%#v` ignores `String()`** and prints the underlying value (`21.456`), which is
exactly what you want when debugging a type whose `String()` is lying to you. `%T`
and the numeric verbs also bypass it.

**A panicking `String()` does not crash your program.** `fmt` recovers and emits
`%!v(PANIC=String method: boom)`. So this marker in a log means a `String()` method
panicked — very often a nil-pointer dereference in a method with a pointer receiver.

## Every malformed-verb marker

These strings are worth memorizing; they are what you will actually see in a log.

```go
package main

import "fmt"

func main() {
  fmt.Println(fmt.Sprintf("%d", "hello"))
  fmt.Println(fmt.Sprintf("%s", 42))
  fmt.Println(fmt.Sprintf("%s"))
  fmt.Println(fmt.Sprintf("%d", 1, 2, 3))
  fmt.Println(fmt.Sprintf("%z", 1))
  fmt.Println(fmt.Sprintf("%!", 1))
  fmt.Println(fmt.Sprintf("%[3]d", 1))
  fmt.Println(fmt.Sprintf("%.*d", "x", 1))
}
```

Every line here is a deliberate mistake, and each one is also a `go vet` failure —
see the next section. Output:

```
%!d(string=hello)
%!s(int=42)
%!s(MISSING)
1%!(EXTRA int=2, int=3)
%!z(int=1)
%!!(int=1)
%!d(BADINDEX)
%!(BADPREC)1
```

| Marker | Cause |
|---|---|
| `%!d(string=hello)` | verb/type mismatch; shows the verb, actual type, and value |
| `%!s(MISSING)` | more verbs than arguments |
| `%!(EXTRA int=2, int=3)` | more arguments than verbs, appended at the end |
| `%!z(int=1)` | unknown verb |
| `%!d(BADINDEX)` | `%[n]` out of range |
| `%!(BADPREC)` | `%.*` given a non-int |
| `%!(BADWIDTH)` | `%*` given a non-int |
| `%!v(PANIC=…)` | the value's `String()`/`Error()`/`Format()` panicked |
| `%!s(*T=nil)` style | a nil value where the verb needed a method |

Note `fmt` never returns an error for these and never panics — it embeds the
complaint in the output. A log line containing `%!` is always a bug in the format
string, not in the data.

## go vet catches all of them

```go
package main

import "fmt"

func main() {
  fmt.Printf("%d\n", "hello")
  fmt.Printf("%s %s\n", "one")
  fmt.Printf("%s\n", "a", "b")
  fmt.Printf("no verbs\n", 1)
  fmt.Println("hi %s", "x")
}
```

`go vet` on this file, real output:

```
# m06k4xq9z/p/aw_vet
./main.go:6:14: fmt.Printf format %d has arg "hello" of wrong type string
./main.go:7:17: fmt.Printf format %s reads arg #2, but call has 1 arg
./main.go:8:2: fmt.Printf call needs 1 arg but has 2 args
./main.go:9:27: fmt.Printf call has arguments but no formatting directives
./main.go:10:2: fmt.Println call has possible Printf formatting directive %s
```

The last one is the most valuable: `Println` does not interpret `%s`, so
`fmt.Println("hi %s", "x")` prints the format string literally. vet catches it.
`go test` runs a subset of vet automatically, which is why these rarely reach
production. vet also checks your own functions if they wrap `Printf` and are named
in the `-printf.funcs` flag or follow the `…f` convention.

## Print vs Println spacing

`Println` always separates and always ends with a newline. `Print` inserts a space
**only between two operands when neither is a string**.

```go
package main

import "fmt"

func main() {
  fmt.Print("a", "b", 1, 2, "c", 3, "\n")
  fmt.Println("a", "b", 1, 2, "c", 3)
  fmt.Printf("%q\n", fmt.Sprint("a", "b", 1, 2, "c", 3))
  fmt.Printf("%q\n", fmt.Sprintln("a", "b", 1, 2, "c", 3))
  fmt.Printf("%q\n", fmt.Sprint(1, 2, 3))
  fmt.Printf("%q\n", fmt.Sprint("x", "y"))
  fmt.Printf("%q\n", fmt.Sprintf("%s%s", "x", "y"))
  fmt.Printf("%q\n", fmt.Append([]byte("p:"), 1, 2))
  fmt.Printf("%q\n", fmt.Appendf([]byte("p:"), "%d-%d", 1, 2))
  fmt.Printf("%q\n", fmt.Appendln([]byte("p:"), 1, 2))
}
```

Output:

```
ab1 2c3
a b 1 2 c 3
"ab1 2c3"
"a b 1 2 c 3\n"
"1 2 3"
"xy"
"xy"
"p:1 2"
"p:1-2"
"p:1 2\n"
```

`Sprint("a","b",1,2,"c",3)` is `"ab1 2c3"`: no space around the strings, one space
between `1` and `2` because both are non-strings. This rule surprises everyone once.
Idiomatic: use `Println` for ad-hoc output and `Printf` when the layout matters;
reach for `Print` only when you are deliberately concatenating.

## Printing nil

```go
package main

import (
  "errors"
  "fmt"
)

type T struct{ X int }

func main() {
  var ep *T
  var sl []int
  var m map[string]int
  var i any
  var e error
  var fn func()
  var ch chan int

  fmt.Printf("ptr    %v %+v %#v\n", ep, ep, ep)
  fmt.Printf("slice  %v %#v len=%d nil=%v\n", sl, sl, len(sl), sl == nil)
  fmt.Printf("map    %v %#v nil=%v\n", m, m, m == nil)
  fmt.Printf("any    %v %#v\n", i, i)
  fmt.Printf("error  %v | %v\n", e, errors.New("real"))
  fmt.Printf("func   %v\n", fn == nil)
  fmt.Printf("chan   %v\n", ch)
  fmt.Printf("string %q\n", "")

  var pp *T
  fmt.Printf("deref-safe %v, non-nil %v\n", pp, &T{5})
  fmt.Println(e == nil, ep == nil)
}
```

Output:

```
ptr    <nil> <nil> (*main.T)(nil)
slice  [] []int(nil) len=0 nil=true
map    map[] map[string]int(nil) nil=true
any    <nil> <nil>
error  <nil> | real
func   true
chan   <nil>
string ""
deref-safe <nil>, non-nil &{5}
true true
```

A nil error and a nil pointer both print `<nil>` under `%v`, which is why `%v` alone
cannot tell you whether an `error` variable holds a typed nil pointer — the trap from
5.2. `%#v` distinguishes them: `(*main.T)(nil)` versus `<nil>`. Note also that a nil
slice prints as `[]`, identical to an empty one, while `%#v` shows `[]int(nil)`.

Idiomatic: `%v` for logs, `%#v` when you are debugging nilness or type identity.

## The Scan family, briefly

Scanning treats newlines as space (`Scan`), requires them to match (`Scanf`), or
stops at them (`Scanln`). All take pointers.

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  var a int
  var b string
  n, err := fmt.Sscan("42 hello", &a, &b)
  fmt.Println(n, err, a, b)

  var x, y int
  n, err = fmt.Sscanf("10-20", "%d-%d", &x, &y)
  fmt.Println(n, err, x, y)

  n, err = fmt.Sscanf("nope", "%d", &x)
  fmt.Println(n, err)

  var p, q string
  n, err = fmt.Fscan(strings.NewReader("one two"), &p, &q)
  fmt.Println(n, err, p, q)

  var line string
  n, err = fmt.Sscanln("single", &line)
  fmt.Println(n, err, line)
}
```

Output:

```
2 <nil> 42 hello
2 <nil> 10 20
0 expected integer
2 <nil> one two
1 <nil> single
```

The first return is the number of items successfully assigned, so check it as well
as the error. Idiomatic: for anything beyond a throwaway script, use `bufio.Scanner`
plus `strconv` (6.6) or `encoding/json` — the Scan family is slow (6.6's benchmark:
179 ns/op versus 4 ns for `Atoi`) and its whitespace rules are fiddly.

## Quick Reference

| Verb | Applies to | Produces |
|---|---|---|
| `%v` | anything | default format |
| `%+v` | structs | with field names |
| `%#v` | anything | Go syntax; ignores `String()` |
| `%T` | anything | the type |
| `%%` | — | a literal `%` |
| `%t` | bool | `true`/`false` |
| `%d` | integers | base 10 |
| `%b` `%o` `%O` `%x` `%X` | integers | base 2, 8, `0o`8, 16 lower/upper |
| `%c` | integer | the rune |
| `%q` | integer / string | rune literal / quoted string |
| `%U` `%#U` | integer | `U+00C8` / with the rune |
| `%f` `%F` | floats | no exponent, 6 digits default |
| `%e` `%E` | floats | decimal exponent |
| `%g` `%G` | floats | shortest; what `%v` uses |
| `%b` `%x` `%X` | floats | binary / hex exponent (exact) |
| `%s` | string, `[]byte`, Stringer, error | the text |
| `%x` `%X` | string, `[]byte` | hex of the bytes |
| `%p` | pointer, slice, map, chan, func | hex address |

| Need | Use |
|---|---|
| debug a struct | `%+v`, or `%#v` for Go syntax |
| see invisible characters | `%q` |
| check a type at runtime | `%T` |
| build an error | `fmt.Errorf` with `%w` (8.2) |
| write into a Builder/Buffer | `fmt.Fprintf(&b, …)` |
| one value, many verbs | `%[1]d %[1]x %[1]b` |
| fast single-value conversion | `strconv`, not `fmt` (6.6) |

## Common Mistakes

**Passing arguments to `Println` with a format string.** It prints `%s` literally;
`go vet` reports "possible Printf formatting directive".

**Implementing both `String()` and `Error()` on one type.** `Error()` wins for
`%v`/`%s` and the `String()` is dead code.

**Logging a struct with `%+v` when it contains secrets.** `fmt` reads unexported
fields by reflection; implement `String()` that redacts them.

**Using `%v` to check whether an `error` is really nil.** A typed nil pointer also
prints `<nil>`; use `%#v` or compare against nil directly.

**Reaching for `fmt.Sprintf` to convert a single number.** `strconv` is ~3x faster
and skips reflection.

## Exercises

1. Write one program printing the value `255` under `%v %d %b %o %O %x %X %c %q %U
   %#U %T` and the value `255.0` under every float verb, and produce the whole
   thing as a two-column markdown table generated by the program itself.
2. Implement `Stringer` on a `Password` type that always prints `[REDACTED]`, then
   show with `%v`, `%s`, `%+v`, and `%#v` which of the four still leaks the value.
   Fix the leak.
3. Deliberately trigger all of `%!d(string=…)`, `%!s(MISSING)`, `%!(EXTRA …)`,
   `%!d(BADINDEX)`, `%!(BADWIDTH)`, and `%!v(PANIC=…)`, then run `go vet` and note
   which it catches at compile time and which only appear at runtime.
4. Write a `Format(f fmt.State, verb rune)` method for a `Money` type honoring the
   `-` and `+` flags and a width, by querying `f.Flag('-')` and `f.Width()`.
   Verify `%10v`, `%-10v`, and `%+v` all do the right thing.
5. Assert in a test that `fmt.Sprintf("%v", map[string]int{…})` is byte-identical
   across 1000 runs, then show the same assertion failing for a `range` loop over
   that map, and explain the difference.

---

[← 6.6 strconv: Parsing and Formatting](06-strconv.md) · [Course Index](../README.md) · [6.8 unicode, utf8, and Text Edge Cases →](08-unicode-utf8.md)
