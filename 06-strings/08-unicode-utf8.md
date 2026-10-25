[← 6.7 fmt: Every Verb and Flag](07-fmt-verbs.md) · [Course Index](../README.md) · [7.1 Arrays: Fixed Size, Value Semantics →](../07-slices-maps/01-arrays.md)

# 6.8 — unicode, utf8, and Text Edge Cases

## What You'll Learn

- The full `unicode` and `unicode/utf8` reference: classification, case, range tables, constants
- Two identical-looking `é` strings that compare unequal, and why the stdlib cannot fix it
- `EqualFold` versus `ToLower` for comparison, and the folds `ToLower` misses
- Truncating, sorting, and sanitizing text without corrupting it

## Coming From Another Language

| Other | Go |
|---|---|
| Python `unicodedata.normalize("NFC", s)` | **not in the stdlib**; `golang.org/x/text/unicode/norm` |
| Python `s.casefold()` | `strings.EqualFold(a, b)` (comparison only, no folded string) |
| Python `str.isalpha()` | `unicode.IsLetter(r)` — per rune, not per string |
| JS `s.localeCompare(t)` | **not in the stdlib**; `golang.org/x/text/collate` |
| JS `s.normalize()` | as above |
| Rust `char::is_alphabetic` | `unicode.IsLetter` |
| ICU collation / segmentation | `golang.org/x/text`; nothing in core Go |

Go's core ships Unicode *character data* — categories, case mappings, scripts — and
UTF-8 encoding. It deliberately does not ship normalization, collation, or grapheme
segmentation. Those are in `golang.org/x/text`, which is maintained by the Go team
but versioned separately.

## Classification

| Function | True for |
|---|---|
| `IsLetter` | category L (any script) |
| `IsDigit` | category Nd — decimal digits only |
| `IsNumber` | category N — includes `Ⅷ`, fractions, superscripts |
| `IsSpace` | space, tab, newline, NBSP, and friends |
| `IsUpper` / `IsLower` / `IsTitle` | categories Lu / Ll / Lt |
| `IsPunct` | category P |
| `IsPrint` | letters, marks, numbers, punctuation, symbols, and ASCII space |
| `IsGraphic` | `IsPrint` plus other Unicode spaces (Zs) |
| `IsControl` | category Cc |
| `IsMark` | category M — combining marks |
| `IsSymbol` | category S — `+`, `€`, emoji |
| `Is(table, r)` / `In(r, tables…)` / `IsOneOf(tables, r)` | membership in a `*RangeTable` |

```go
package main

import (
  "fmt"
  "unicode"
)

func main() {
  fmt.Printf("%-6s %-7s %-6s %-6s %-6s %-6s %-6s %-6s %-6s %-6s %-6s %-6s\n",
    "rune", "Letter", "Digit", "Number", "Space", "Upper", "Lower", "Title", "Punct", "Print", "Ctrl", "Symbol")
  for _, r := range []rune{'a', 'A', '7', 'Ⅷ', ' ', '\t', '\n', 'Ǆ', '!', '\x00', '+', '世', '́', '€'} {
    fmt.Printf("%-6q %-7v %-6v %-6v %-6v %-6v %-6v %-6v %-6v %-6v %-6v %-6v\n",
      r, unicode.IsLetter(r), unicode.IsDigit(r), unicode.IsNumber(r), unicode.IsSpace(r),
      unicode.IsUpper(r), unicode.IsLower(r), unicode.IsTitle(r), unicode.IsPunct(r),
      unicode.IsPrint(r), unicode.IsControl(r), unicode.IsSymbol(r))
  }
  fmt.Println()
  fmt.Println(unicode.IsMark('́'), unicode.IsGraphic(' '), unicode.IsGraphic('\n'))
  fmt.Printf("%q %q %q\n", unicode.ToUpper('ǆ'), unicode.ToLower('Ǆ'), unicode.ToTitle('ǆ'))
  fmt.Printf("%q %q\n", unicode.To(unicode.UpperCase, 'a'), unicode.To(unicode.TitleCase, 'ǆ'))
  fmt.Printf("MaxRune=%U MaxASCII=%U MaxLatin1=%U Replacement=%U Version=%s\n",
    unicode.MaxRune, unicode.MaxASCII, unicode.MaxLatin1, unicode.ReplacementChar, unicode.Version)
}
```

Output:

```
rune   Letter  Digit  Number Space  Upper  Lower  Title  Punct  Print  Ctrl   Symbol
'a'    true    false  false  false  false  true   false  false  true   false  false 
'A'    true    false  false  false  true   false  false  false  true   false  false 
'7'    false   true   true   false  false  false  false  false  true   false  false 
'Ⅷ'    false   false  true   false  false  false  false  false  true   false  false 
' '    false   false  false  true   false  false  false  false  true   false  false 
'\t'   false   false  false  true   false  false  false  false  false  true   false 
'\n'   false   false  false  true   false  false  false  false  false  true   false 
'Ǆ'    true    false  false  false  true   false  false  false  true   false  false 
'!'    false   false  false  false  false  false  false  true   true   false  false 
'\x00' false   false  false  false  false  false  false  false  false  true   false 
'+'    false   false  false  false  false  false  false  false  true   false  true  
'世'    true    false  false  false  false  false  false  false  true   false  false 
'́'    false   false  false  false  false  false  false  false  true   false  false 
'€'    false   false  false  false  false  false  false  false  true   false  true  

true true false
'Ǆ' 'ǆ' 'ǅ'
'A' 'ǅ'
MaxRune=U+10FFFF MaxASCII=U+007F MaxLatin1=U+00FF Replacement=U+FFFD Version=17.0.0
```

`IsDigit('Ⅷ')` is false but `IsNumber('Ⅷ')` is true — use `IsDigit` when you mean
"a character I can feed to `strconv`". `IsSpace('\t')` is true while `IsPrint('\t')`
is false. And `ǆ` is the case where title case differs from upper case: `Ǆ` upper,
`ǅ` title, `ǆ` lower — three distinct code points for one digraph.

This toolchain's tables are Unicode 17.0.0; `unicode.Version` tells you at runtime.

## Case folding and range tables

`SimpleFold` walks a cycle of all runes that fold together, which is the primitive
behind `EqualFold`.

```go
package main

import (
  "fmt"
  "unicode"
)

func main() {
  r := 'k'
  for i := 0; i < 4; i++ {
    fmt.Printf("%q(%U) -> ", r, r)
    r = unicode.SimpleFold(r)
  }
  fmt.Println()

  for _, x := range []rune{'A', 'a', 'K', 'ſ'} {
    fmt.Printf("SimpleFold(%q)=%q\n", x, unicode.SimpleFold(x))
  }

  for _, x := range []rune{'a', '世', 'α', 'д', '7'} {
    fmt.Printf("%q latin=%-5v han=%-5v greek=%-5v cyrillic=%-5v digit-table=%v\n",
      x,
      unicode.Is(unicode.Latin, x),
      unicode.Is(unicode.Han, x),
      unicode.Is(unicode.Greek, x),
      unicode.Is(unicode.Cyrillic, x),
      unicode.Is(unicode.Nd, x))
  }
  fmt.Println(unicode.In('世', unicode.Latin, unicode.Han))
  fmt.Println(unicode.IsOneOf([]*unicode.RangeTable{unicode.Latin}, 'a'))
  fmt.Println(len(unicode.Scripts), len(unicode.Categories), len(unicode.Properties))
}
```

Output:

```
'k'(U+006B) -> 'K'(U+212A) -> 'K'(U+004B) -> 'k'(U+006B) -> 
SimpleFold('A')='a'
SimpleFold('a')='A'
SimpleFold('K')='k'
SimpleFold('ſ')='S'
'a' latin=true  han=false greek=false cyrillic=false digit-table=false
'世' latin=false han=false greek=false cyrillic=false digit-table=false
'α' latin=false han=false greek=true  cyrillic=false digit-table=false
'д' latin=false han=false greek=false cyrillic=true  digit-table=false
'7' latin=false han=false greek=false cyrillic=false digit-table=true
true
true
174 38 39
```

Lowercase `k`, KELVIN SIGN `K` (U+212A), and uppercase `K` form a three-element fold
cycle — so `"K"` and `"K"` are case-insensitively equal, which is the detail
that breaks naive `ToLower` comparison below.

`unicode.Scripts`, `Categories`, and `Properties` are maps from name to
`*RangeTable`, so you can look up a script dynamically:
`unicode.Scripts["Devanagari"]`.

Idiomatic validation with a range table:

```go
package main

import (
  "fmt"
  "strings"
  "unicode"
)

func isASCIIIdentifier(s string) bool {
  if s == "" {
    return false
  }
  for i, r := range s {
    switch {
    case r == '_':
    case unicode.IsLetter(r) && r <= unicode.MaxASCII:
    case i > 0 && unicode.IsDigit(r):
    default:
      return false
    }
  }
  return true
}

func main() {
  for _, s := range []string{"foo_bar", "_x9", "9abc", "héllo", "", "a b"} {
    fmt.Printf("%-10q %v\n", s, isASCIIIdentifier(s))
  }
  fmt.Println(strings.IndexFunc("abc 世", func(r rune) bool { return r > unicode.MaxASCII }))
}
```

Output:

```
"foo_bar"  true
"_x9"      true
"9abc"     false
"héllo"    false
""         false
"a b"      false
4
```

## The utf8 package in full

| Name | Signature | Note |
|---|---|---|
| `RuneLen` | `(r rune) int` | 1–4, or -1 if unencodable |
| `RuneCount` | `(p []byte) int` | invalid bytes count as 1 each |
| `RuneCountInString` | `(s string) int` | |
| `DecodeRune` | `(p []byte) (rune, int)` | `(RuneError, 1)` on invalid; `(RuneError, 0)` on empty |
| `DecodeRuneInString` | `(s string) (rune, int)` | |
| `DecodeLastRune` | `(p []byte) (rune, int)` | from the end |
| `DecodeLastRuneInString` | `(s string) (rune, int)` | |
| `EncodeRune` | `(p []byte, r rune) int` | writes into `p`; needs `UTFMax` room |
| `AppendRune` | `(p []byte, r rune) []byte` | the allocating-friendly form |
| `Valid` | `(p []byte) bool` | |
| `ValidString` | `(s string) bool` | |
| `ValidRune` | `(r rune) bool` | rejects surrogates and > MaxRune |
| `FullRune` | `(p []byte) bool` | is the buffer a complete sequence? |
| `FullRuneInString` | `(s string) bool` | |
| `RuneStart` | `(b byte) bool` | can this byte begin a rune? |
| `RuneError` | `= '�'` | the replacement character |
| `RuneSelf` | `= 0x80` | runes below this are 1 byte |
| `MaxRune` | `= '\U0010FFFF'` | |
| `UTFMax` | `= 4` | max bytes per rune |

```go
package main

import (
  "fmt"
  "unicode/utf8"
)

func main() {
  buf := make([]byte, utf8.UTFMax)
  n := utf8.EncodeRune(buf, '世')
  fmt.Println(n, buf[:n], string(buf[:n]))

  out := utf8.AppendRune([]byte("x="), '😀')
  fmt.Printf("%q %x\n", out, out)

  fmt.Println(utf8.UTFMax, utf8.RuneSelf, utf8.MaxRune, utf8.RuneError)
  fmt.Printf("%U %U\n", utf8.MaxRune, utf8.RuneError)

  bad := []byte{0xff, 0xfe}
  fmt.Println(utf8.Valid(bad), utf8.Valid([]byte("ok")))
  fmt.Println(utf8.RuneCount([]byte("héllo")), utf8.RuneCountInString("héllo"))
  fmt.Println(utf8.FullRune([]byte{0xe4, 0xb8}), utf8.FullRune([]byte{0xe4, 0xb8, 0x96}))

  n2 := utf8.EncodeRune(buf, 0x110000)
  fmt.Println(n2, buf[:n2], string(buf[:n2]) == string(utf8.RuneError))
}
```

Output:

```
3 [228 184 150] 世
"x=😀" 783df09f9880
4 128 1114111 65533
U+10FFFF U+FFFD
false true
5 5
false true
3 [239 191 189] true
```

`EncodeRune` on an invalid rune silently writes U+FFFD rather than failing — check
with `ValidRune` first if that matters.

## Normalization: the same é, unequal

This is the edge case that reaches production most often, usually via copy-paste
from a Mac filesystem or an iOS keyboard.

```go
package main

import (
  "fmt"
  "strings"
  "unicode/utf8"
)

func main() {
  nfc := "é"
  nfd := "é"
  fmt.Printf("%q vs %q -> equal=%v\n", nfc, nfd, nfc == nfd)
  fmt.Printf("look identical: %s %s\n", nfc, nfd)
  fmt.Printf("bytes  %x vs %x\n", nfc, nfd)
  fmt.Printf("len    %d vs %d\n", len(nfc), len(nfd))
  fmt.Printf("runes  %d vs %d\n", utf8.RuneCountInString(nfc), utf8.RuneCountInString(nfd))
  fmt.Printf("fold   %v\n", strings.EqualFold(nfc, nfd))
  fmt.Printf("upper  %q vs %q\n", strings.ToUpper(nfc), strings.ToUpper(nfd))
}
```

Output:

```
"é" vs "é" -> equal=false
look identical: é é
bytes  c3a9 vs 65cc81
len    2 vs 3
runes  1 vs 2
fold   false
upper  "É" vs "É"
```

NFC composes `é` into one code point (U+00E9, 2 bytes). NFD decomposes it into `e`
plus U+0301 COMBINING ACUTE ACCENT (3 bytes). They render identically, and `==`,
`EqualFold`, `len`, and rune count all disagree about them. No function in the
standard library will reconcile these; `golang.org/x/text/unicode/norm` provides
`norm.NFC.String(s)`.

Idiomatic: normalize to NFC once, at the boundary where text enters (form handler,
file import), and store only normalized text. Comparing user-typed names, emails,
or filenames without this is a real-world bug class, not a curiosity.

## EqualFold versus ToLower

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  pairs := [][2]string{{"Go", "GO"}, {"K", "K"}, {"ß", "SS"}, {"İ", "i"}, {"ﬁ", "fi"}, {"ſ", "s"}}
  for _, p := range pairs {
    fmt.Printf("%-4q %-10q EqualFold=%-6v ToLower==%-6v\n",
      p[0], p[1], strings.EqualFold(p[0], p[1]),
      strings.ToLower(p[0]) == strings.ToLower(p[1]))
  }
  fmt.Println()
  fmt.Printf("ToUpper(ß)=%q ToUpper(ẛ)=%q ToUpper(ﬁ)=%q\n",
    strings.ToUpper("ß"), strings.ToUpper("ẛ"), strings.ToUpper("ﬁ"))
  fmt.Printf("ToLower(İ)=%q len=%d\n", strings.ToLower("İ"), len(strings.ToLower("İ")))
}
```

Output:

```
"Go" "GO"       EqualFold=true   ToLower==true  
"K"  "K"        EqualFold=true   ToLower==true  
"ß"  "SS"       EqualFold=false  ToLower==false 
"İ"  "i"        EqualFold=false  ToLower==true  
"ﬁ"  "fi"       EqualFold=false  ToLower==false 
"ſ"  "s"        EqualFold=true   ToLower==false 
```

The last two rows are the interesting ones, and they disagree in **opposite**
directions:

- `ſ` (long s) versus `s`: `EqualFold` says equal (they share a fold cycle),
  `ToLower` comparison says not equal. `EqualFold` is right.
- `İ` (dotted capital I) versus `i`: `ToLower` says equal because `ToLower('İ')`
  is `'i'`, while `EqualFold` says no. Here Unicode full case folding agrees with
  `EqualFold`; the `ToLower` match is a lossy accident.

Neither handles `ß`/`SS` or `ﬁ`/`fi`, because those are *full* case foldings that
change length and Go implements only *simple* folding. For those you need
`golang.org/x/text/cases`.

Idiomatic: `strings.EqualFold` for case-insensitive comparison. Never
`ToLower(a) == ToLower(b)`.

`ToUpper("ß")` is `"ß"` unchanged and `ToUpper("ﬁ")` is `"ﬁ"` unchanged — again, a
per-rune mapping cannot grow the string. But `ToUpper("ẛ")` *is* `"Ṡ"`, because that
one happens to have a single-rune uppercase.

## The Turkish dotless i

```go
package main

import (
  "fmt"
  "strings"
  "unicode"
)

func main() {
  fmt.Println(strings.ToUpper("istanbul"))
  fmt.Println(strings.ToUpperSpecial(unicode.TurkishCase, "istanbul"))
  fmt.Println(strings.ToLower("ISTANBUL"))
  fmt.Println(strings.ToLowerSpecial(unicode.TurkishCase, "ISTANBUL"))
  fmt.Println(strings.ToUpperSpecial(unicode.AzeriCase, "i") == strings.ToUpperSpecial(unicode.TurkishCase, "i"))
}
```

Output:

```
ISTANBUL
İSTANBUL
istanbul
ıstanbul
true
```

In Turkish and Azeri, dotted and dotless i are different letters, so uppercasing `i`
gives `İ` and lowercasing `I` gives `ı`. This is why case-insensitive comparison
should never route through a locale-dependent case mapping, and why
`unicode.TurkishCase` exists as an explicit opt-in. `unicode.AzeriCase` is the same
table.

## Emoji: ZWJ sequences and modifiers

```go
package main

import (
  "fmt"
  "strings"
  "unicode"
  "unicode/utf8"
)

func main() {
  samples := []string{"👩‍💻", "👨‍👩‍👧‍👦", "👍🏿", "🇯🇵", "é", "é"}
  for _, s := range samples {
    fmt.Printf("%-8s bytes=%-3d runes=%-2d marks=%d\n", s, len(s), utf8.RuneCountInString(s), countMarks(s))
  }
  fmt.Println()
  fam := "👨‍👩‍👧‍👦"
  for i, r := range fam {
    fmt.Printf("i=%-3d %U %q\n", i, r, r)
  }
  fmt.Println()
  fmt.Printf("ToUpper keeps emoji: %q\n", strings.ToUpper("go 👍🏿"))
  fmt.Printf("first rune only: %q\n", string([]rune(fam)[0]))
}

func countMarks(s string) int {
  n := 0
  for _, r := range s {
    if unicode.IsMark(r) {
      n++
    }
  }
  return n
}
```

Output:

```
👩‍💻      bytes=11  runes=3  marks=0
👨‍👩‍👧‍👦  bytes=25  runes=7  marks=0
👍🏿       bytes=8   runes=2  marks=0
🇯🇵       bytes=8   runes=2  marks=0
é        bytes=2   runes=1  marks=0
é       bytes=3   runes=2  marks=1

i=0   U+1F468 '👨'
i=4   U+200D '\u200d'
i=7   U+1F469 '👩'
i=11  U+200D '\u200d'
i=14  U+1F467 '👧'
i=18  U+200D '\u200d'
i=21  U+1F466 '👦'

ToUpper keeps emoji: "GO 👍🏿"
first rune only: "👨"
```

One family emoji is 25 bytes and 7 runes: four people joined by three U+200D ZERO
WIDTH JOINERs. A skin-tone thumbs-up is a base emoji plus a Fitzpatrick modifier.
A flag is two regional indicators. So "one character" to a user is up to 7 runes
here, and taking `[]rune(s)[0]` yields a lone `👨`.

This is the grapheme-cluster problem. Nothing in the standard library segments
them; use `golang.org/x/text` or `github.com/rivo/uniseg` when you need display
width, cursor movement, or a user-facing character count.

## BOM handling

A UTF-8 byte-order mark is three bytes (EF BB BF) that decode to U+FEFF — a real,
zero-width rune that silently joins your first field.

```go
package main

import (
  "fmt"
  "strings"
  "unicode/utf8"
)

const bom = "\xef\xbb\xbf"

func main() {
  in := bom + "id,name\n1,a\n"
  fmt.Printf("%q\n", in[:10])
  fmt.Println(len(in), strings.HasPrefix(in, bom), utf8.ValidString(in))

  first := strings.Split(strings.Split(in, "\n")[0], ",")[0]
  fmt.Printf("naive first field: %q == \"id\" is %v\n", first, first == "id")

  clean := strings.TrimPrefix(in, bom)
  cfirst := strings.Split(strings.Split(clean, "\n")[0], ",")[0]
  fmt.Printf("cleaned first field: %q == \"id\" is %v\n", cfirst, cfirst == "id")

  r, size := utf8.DecodeRuneInString(in)
  fmt.Printf("leading rune %U size=%d\n", r, size)
  fmt.Println(utf8.RuneCountInString(bom))
}
```

Output:

```
"﻿id,name"
15 true true
naive first field: "﻿id" == "id" is false
cleaned first field: "id" == "id" is true
leading rune U+FEFF size=3
1
```

The BOM is valid UTF-8, so `ValidString` will not catch it, and `%q` is the only
thing that makes it visible. Excel writes one on every CSV export. Idiomatic:
`strings.TrimPrefix(s, "\xef\xbb\xbf")` on the first line of any imported text.

Note the BOM must be written as the three raw bytes in Go source: a literal
`"﻿"` is rejected by the compiler as an illegal byte order mark.

## Truncating without splitting a rune

The naive `s[:n]` corrupts the last character. Two correct helpers: one by rune
count, one by byte budget.

```go
package main

import (
  "fmt"
  "unicode/utf8"
)

func truncateBytesNaive(s string, n int) string {
  if len(s) <= n {
    return s
  }
  return s[:n]
}

func TruncateRunes(s string, maxRunes int) string {
  if maxRunes <= 0 {
    return ""
  }
  count := 0
  for i := range s {
    if count == maxRunes {
      return s[:i]
    }
    count++
  }
  return s
}

func TruncateBytes(s string, maxBytes int) string {
  if len(s) <= maxBytes {
    return s
  }
  for maxBytes > 0 && !utf8.RuneStart(s[maxBytes]) {
    maxBytes--
  }
  return s[:maxBytes]
}

func main() {
  s := "héllo 世界"
  fmt.Printf("%q len=%d runes=%d\n", s, len(s), utf8.RuneCountInString(s))

  n := truncateBytesNaive(s, 2)
  fmt.Printf("naive(2)  %q valid=%v\n", n, utf8.ValidString(n))

  for _, k := range []int{0, 1, 3, 7, 8, 100} {
    t := TruncateRunes(s, k)
    fmt.Printf("TruncateRunes(%-3d) %-14q valid=%v\n", k, t, utf8.ValidString(t))
  }
  for _, k := range []int{2, 8, 9, 12} {
    t := TruncateBytes(s, k)
    fmt.Printf("TruncateBytes(%-3d) %-14q valid=%v len=%d\n", k, t, utf8.ValidString(t), len(t))
  }
}
```

Output:

```
"héllo 世界" len=13 runes=8
naive(2)  "h\xc3" valid=false
TruncateRunes(0  ) ""             valid=true
TruncateRunes(1  ) "h"            valid=true
TruncateRunes(3  ) "hél"          valid=true
TruncateRunes(7  ) "héllo 世"      valid=true
TruncateRunes(8  ) "héllo 世界"     valid=true
TruncateRunes(100) "héllo 世界"     valid=true
TruncateBytes(2  ) "h"            valid=true
TruncateBytes(8  ) "héllo "       valid=true len=7
TruncateBytes(9  ) "héllo "       valid=true len=7
TruncateBytes(12 ) "héllo 世"      valid=true len=10
```

`TruncateRunes` allocates nothing — it uses `range` to find a rune boundary and
returns a sub-slice. `TruncateBytes` is what you want for a database column limit
(`VARCHAR(n)` counts bytes in Postgres's `bytea` and characters in `text`, so know
which you have): it backs off with `RuneStart` until the cut is legal. Neither
preserves grapheme clusters, so both can still split a family emoji from its ZWJ.

## Sorting is byte order, not alphabetical order

```go
package main

import (
  "fmt"
  "slices"
  "strings"
)

func main() {
  words := []string{"zebra", "Apple", "apple", "Ärger", "banana", "Zürich", "école", "edge"}
  byBytes := slices.Clone(words)
  slices.Sort(byBytes)
  fmt.Println(byBytes)

  ci := slices.Clone(words)
  slices.SortFunc(ci, func(a, b string) int { return strings.Compare(strings.ToLower(a), strings.ToLower(b)) })
  fmt.Println(ci)

  fmt.Println("Z" < "a", "Ä" < "B", "é" < "f")
}
```

Output:

```
[Apple Zürich apple banana edge zebra Ärger école]
[Apple apple banana edge zebra Zürich Ärger école]
true false false
```

Byte order puts all uppercase ASCII before all lowercase, and every non-ASCII word
(`Ärger`, `école`) after every ASCII one, because their lead bytes are ≥ 0xC3. A
case-insensitive sort fixes the ASCII part and still strands the accented words at
the end. No ordering of bytes will ever produce `école` next to `edge`; that needs
locale-aware collation from `golang.org/x/text/collate`.

## If you need X, use Y

| You need | Use | In stdlib? |
|---|---|---|
| byte length | `len(s)` | yes |
| character count | `utf8.RuneCountInString(s)` | yes |
| user-perceived character count | grapheme segmentation | **no** — `rivo/uniseg`, `x/text` |
| iterate characters | `for i, r := range s` | yes |
| validate external bytes | `utf8.ValidString` / `utf8.Valid` | yes |
| sanitize external bytes | `strings.ToValidUTF8(s, "�")` | yes |
| case-insensitive compare | `strings.EqualFold` | yes |
| full case folding (`ß`→`ss`) | `x/text/cases` | **no** |
| locale-aware case | `strings.To*Special(unicode.TurkishCase, …)` | yes, Turkish/Azeri only |
| NFC/NFD normalization | `x/text/unicode/norm` | **no** |
| alphabetical sort for humans | `x/text/collate` | **no** |
| classify a rune | `unicode.Is*` | yes |
| script or category membership | `unicode.Is(unicode.Han, r)` | yes |
| truncate safely | `utf8.RuneStart` loop, or count runes | yes |
| strip a BOM | `strings.TrimPrefix(s, "\xef\xbb\xbf")` | yes |
| display column width | wcwidth | **no** — `mattn/go-runewidth` |

## Quick Reference

| Constant | Value |
|---|---|
| `utf8.UTFMax` | 4 |
| `utf8.RuneSelf` | 0x80 |
| `utf8.RuneError` | U+FFFD |
| `utf8.MaxRune` / `unicode.MaxRune` | U+10FFFF |
| `unicode.MaxASCII` | U+007F |
| `unicode.MaxLatin1` | U+00FF |
| `unicode.ReplacementChar` | U+FFFD |
| `unicode.Version` | `"17.0.0"` on this toolchain |

| Comparison | Equal for `"é"` NFC vs NFD? | For `"K"` vs `"K"`? |
|---|---|---|
| `==` | no | no |
| `strings.EqualFold` | no | **yes** |
| `ToLower(a) == ToLower(b)` | no | yes |
| `norm.NFC` then `==` (x/text) | yes | no |
| `norm.NFC` then `EqualFold` | yes | yes |

## Common Mistakes

**Comparing user-entered text without normalizing.** Two identical-looking `é`
strings differ in bytes; normalize to NFC at the input boundary with `x/text`.

**Using `ToLower(a) == ToLower(b)` for case-insensitive matching.** It misses folds
like `ſ`/`s`; use `strings.EqualFold`.

**Truncating with `s[:n]`.** It can split a rune and produce invalid UTF-8; back off
with `utf8.RuneStart` or count runes.

**Treating a rune count as a user-visible character count.** A family emoji is 7
runes and one character; you need grapheme segmentation.

**Forgetting the BOM on imported CSV or text files.** It is valid UTF-8 so nothing
errors, and your first column name silently gains an invisible rune.

## Exercises

1. Write `func FoldKey(s string) string` producing a canonical comparison key, and
   document exactly which of NFC/NFD, `ſ`/`s`, `K`/`K`, `İ`/`i`, and `ß`/`ss`
   it unifies using only the stdlib. Then list what you would need `x/text` for.
2. Write `func SafeTruncate(s string, maxBytes int, ellipsis string) string` that
   never exceeds `maxBytes` **including** the ellipsis and never splits a rune.
   Test it at every byte budget from 0 to `len(s)` for `"héllo 世界"` and assert
   `utf8.ValidString` holds for all of them.
3. Build a table of all runes `r` where `unicode.SimpleFold(r) != r` within
   U+0000–U+0400 by walking each fold cycle, and find every case where a cycle has
   more than two members.
4. Write a fuzz test (`go test -fuzz`) for a function that takes arbitrary `[]byte`,
   runs `strings.ToValidUTF8`, and asserts the output is always valid UTF-8 and
   never longer in runes than the input.
5. Implement a naive grapheme-cluster counter handling only combining marks
   (`unicode.IsMark`), ZWJ sequences (U+200D), regional indicator pairs, and
   Fitzpatrick modifiers (U+1F3FB–U+1F3FF). Compare its answers against
   `github.com/rivo/uniseg` for 20 emoji and explain each disagreement.

---

[← 6.7 fmt: Every Verb and Flag](07-fmt-verbs.md) · [Course Index](../README.md) · [7.1 Arrays: Fixed Size, Value Semantics →](../07-slices-maps/01-arrays.md)
