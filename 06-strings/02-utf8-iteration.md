[← 6.1 string, []byte, and []rune](01-string-bytes-runes.md) · [Course Index](../README.md) · [6.3 The strings Package Reference →](03-strings-package.md)

# 6.2 — UTF-8, Indexing, and Iteration

## What You'll Learn

- `for i, r := range s` decodes runes and reports **byte** offsets, so `i` skips
- Slicing on a non-rune boundary silently produces invalid UTF-8
- How `range` and `utf8.Decode*` report invalid bytes (`U+FFFD`, width 1)
- Why an emoji flag or family is several runes, and what that breaks

## Coming From Another Language

| Other | Go |
|---|---|
| JS `for (const c of str)` — code points, no index | `for i, r := range s` — code point **and** byte offset |
| JS `str.charCodeAt(i)` — UTF-16 unit | `s[i]` is a byte; `[]rune(s)[i]` is a code point |
| Python `for c in s` — one-char strings | `r` is an `int32`, not a string |
| Python `s[3]` — O(1) by character | `s[3]` is O(1) by **byte**; by character needs `[]rune` |
| Rust `s.chars()` / `s.char_indices()` | `range` is exactly `char_indices()` |

## UTF-8 in one paragraph

UTF-8 encodes a code point in 1 to 4 bytes. ASCII (U+0000–U+007F) is one byte with
the high bit clear, so ASCII text is byte-identical to its UTF-8 form. Multibyte
sequences start with a byte in 0xC0–0xF4 whose leading bits say how many
continuation bytes follow, and every continuation byte is in 0x80–0xBF. Because
lead bytes and continuation bytes come from disjoint ranges, the encoding is
self-synchronizing: from any byte offset you can find the nearest rune boundary by
scanning backwards at most three bytes. That property is what makes `s[i:j]` cheap
and `utf8.DecodeLastRuneInString` possible.

## Indexing walks bytes, including mid-rune bytes

```go
package main

import "fmt"

func main() {
  s := "héllo"
  for i := 0; i < len(s); i++ {
    fmt.Printf("%d:%d:%q  ", i, s[i], s[i])
  }
  fmt.Println()
  fmt.Printf("%q %q\n", s[1], s[1:2])
  fmt.Printf("% x\n", s)
}
```

Output:

```
0:104:'h'  1:195:'Ã'  2:169:'©'  3:108:'l'  4:108:'l'  5:111:'o'  
'Ã' "\xc3"
68 c3 a9 6c 6c 6f
```

`s[1]` is 195, which `%q` renders as `'Ã'` because a lone `byte` promoted to a rune
*is* U+00C3. That is a lie about the text: byte 195 here is the first half of `é`.
The 1-byte slice `s[1:2]` tells the truth — `"\xc3"`, an invalid fragment.

## range decodes, and the index jumps

This is the key output of the lesson. `i` is a byte offset, so it increases by the
rune's width, not by one.

```go
package main

import "fmt"

func main() {
  s := "h€llo, 世界"
  for i, r := range s {
    fmt.Printf("i=%-2d r=%-4q %-8U width=%d\n", i, r, r, len(string(r)))
  }
}
```

Output:

```
i=0  r='h'  U+0068   width=1
i=1  r='€'  U+20AC   width=3
i=4  r='l'  U+006C   width=1
i=5  r='l'  U+006C   width=1
i=6  r='o'  U+006F   width=1
i=7  r=','  U+002C   width=1
i=8  r=' '  U+0020   width=1
i=9  r='世'  U+4E16   width=3
i=12 r='界'  U+754C   width=3
```

Offsets go 0, 1, 4 — there is no `i == 2` or `i == 3`. So `i` is not a character
counter. If you want a character counter, keep your own, or use the offset for what
it is good at: `s[i:]` and `s[:i]` are always valid slices because `i` is always a
rune boundary.

## Slicing off a boundary produces invalid UTF-8

Nothing checks, nothing panics. You get a string the rest of the library will
decode as the replacement character.

```go
package main

import (
  "fmt"
  "unicode/utf8"
)

func main() {
  s := "héllo"
  bad := s[0:2]
  fmt.Printf("%q %v %d\n", bad, utf8.ValidString(bad), len(bad))
  fmt.Println(bad)
  for i, r := range bad {
    fmt.Printf("  i=%d r=%q %U\n", i, r, r)
  }
  fmt.Printf("%q\n", []rune(bad))
  fmt.Println(utf8.RuneError == '�', string(utf8.RuneError))
}
```

Output:

```
"h\xc3" false 2
h�
  i=0 r='h' U+0068
  i=1 r='�' U+FFFD
['h' '�']
true �
```

`range` over the truncated string yields `utf8.RuneError` (U+FFFD) and advances by
**one byte**, so a run of invalid bytes becomes one replacement char per byte.
`[]rune` conversion substitutes U+FFFD the same way. Neither reports an error — the
only way to know is `utf8.ValidString`.

## Invalid input from the outside world

```go
package main

import (
  "fmt"
  "strings"
  "unicode/utf8"
)

func main() {
  bad := "a\xffb\xc3"
  fmt.Println(utf8.ValidString(bad), utf8.RuneCountInString(bad), len(bad))
  for i, r := range bad {
    fmt.Printf("i=%d r=%U %q\n", i, r, r)
  }
  fmt.Printf("%q\n", strings.ToValidUTF8(bad, "?"))
  fmt.Printf("%q\n", strings.ToValidUTF8(bad, ""))
  fmt.Println(utf8.ValidRune(0x110000), utf8.ValidRune(0xD800), utf8.ValidRune('a'))
  fmt.Println(utf8.FullRuneInString("\xe4\xb8"), utf8.FullRuneInString("\xe4\xb8\x96"))
  fmt.Println(utf8.RuneStart(0xe4), utf8.RuneStart(0xb8))
}
```

Output:

```
false 4 4
i=0 r=U+0061 'a'
i=1 r=U+FFFD '�'
i=2 r=U+0062 'b'
i=3 r=U+FFFD '�'
"a?b?"
"ab"
false false true
false true
true false
```

`RuneCountInString` counts 4 for 4 bytes, counting each invalid byte as one rune.
`ValidRune` rejects both out-of-range values and the surrogate range U+D800–U+DFFF,
which UTF-8 forbids. `FullRuneInString` tells you whether a buffer holds a complete
sequence — the thing to check when reading a stream in chunks. `RuneStart` tells you
whether a byte can begin a rune, which is how you walk backwards to a boundary.

Idiomatic: at the boundary where untrusted bytes enter, either reject with
`utf8.ValidString` or sanitize with `strings.ToValidUTF8`. Do not let invalid bytes
reach a database or a template.

## Manual decoding

`utf8.DecodeRuneInString` returns the rune and its width, which is the explicit form
of what `range` does.

```go
package main

import (
  "fmt"
  "unicode/utf8"
)

func main() {
  s := "héllo"
  r, size := utf8.DecodeRuneInString(s[1:])
  fmt.Printf("%q %d\n", r, size)

  lr, lsize := utf8.DecodeLastRuneInString("日本語")
  fmt.Printf("%q %d\n", lr, lsize)

  br, bsize := utf8.DecodeRune([]byte{0xff, 0x41})
  fmt.Printf("%q %d %v\n", br, bsize, br == utf8.RuneError)

  fmt.Println(utf8.RuneLen('a'), utf8.RuneLen('é'), utf8.RuneLen('世'), utf8.RuneLen('😀'))
  fmt.Println(utf8.RuneLen(-1))

  for i := 0; i < len(s); {
    r, w := utf8.DecodeRuneInString(s[i:])
    fmt.Printf("%d:%q ", i, r)
    i += w
  }
  fmt.Println()
}
```

Output:

```
'é' 2
'語' 3
'�' 1 true
1 2 3 4
-1
0:'h' 1:'é' 3:'l' 4:'l' 5:'o' 
```

`DecodeRune` on an invalid byte returns `(RuneError, 1)` — and note that
`(RuneError, 3)` would be the result for a *valid* encoding of U+FFFD, so the size
is how you distinguish real replacement characters from errors. `RuneLen` returns -1
for an unencodable rune.

## Random access by rune

There is no O(1) character index. Convert once if you need many lookups.

```go
package main

import (
  "fmt"
  "unicode/utf8"
)

func runeAt(s string, n int) rune {
  for i, r := range s {
    _ = i
    if n == 0 {
      return r
    }
    n--
  }
  return utf8.RuneError
}

func main() {
  s := "h€llo, 世界"
  rs := []rune(s)
  fmt.Printf("%q %q %q\n", rs[1], rs[7], rs[len(rs)-1])
  fmt.Println(len(rs), len(s))
  fmt.Printf("%q %q\n", runeAt(s, 1), runeAt(s, 100))
  fmt.Println(string(rs[2:5]))
}
```

Output:

```
'€' '世' '界'
9 15
'€' '�'
llo
```

`[]rune(s)` is O(len(s)) in time and allocates 4 bytes per rune — for this 15-byte
string, 36 bytes. Idiomatic: `range` for a single pass, `[]rune` only when you need
repeated indexing or in-place rune reordering.

## Reversing a string

By rune is correct for most text and wrong for emoji sequences.

```go
package main

import (
  "fmt"
  "unicode/utf8"
)

func reverseRunes(s string) string {
  rs := []rune(s)
  for i, j := 0, len(rs)-1; i < j; i, j = i+1, j-1 {
    rs[i], rs[j] = rs[j], rs[i]
  }
  return string(rs)
}

func main() {
  for _, s := range []string{"héllo", "🇮🇳", "👨‍👩‍👧", "👍🏽", "é"} {
    fmt.Printf("%-12q len=%-3d runes=%-3d codepoints=%v\n",
      s, len(s), utf8.RuneCountInString(s), []rune(s))
  }
  fmt.Println()
  fmt.Println(reverseRunes("héllo, 世界"))
  fmt.Printf("%q\n", reverseRunes("👨‍👩‍👧"))
  fmt.Printf("%q\n", reverseRunes("é"))
}
```

Output:

```
"héllo"      len=6   runes=5   codepoints=[104 233 108 108 111]
"🇮🇳"         len=8   runes=2   codepoints=[127470 127475]
"👨\u200d👩\u200d👧" len=18  runes=5   codepoints=[128104 8205 128105 8205 128103]
"👍🏽"         len=8   runes=2   codepoints=[128077 127997]
"é"         len=3   runes=2   codepoints=[101 769]

界世 ,olléh
"👧\u200d👩\u200d👨"
"́e"
```

The India flag is two regional-indicator runes. The family is five runes joined by
U+200D ZERO WIDTH JOINER, which `%q` escapes as `‍`. The thumbs-up carries a
separate skin-tone modifier. Reversing by rune reorders the family members, so the
joiners end up between the wrong people.

Note the `"é"` here is the *precomposed* U+00E9 — one rune, two bytes — so reversing
it is a no-op. The same character can also be written *decomposed* as `e` plus a
combining accent, and then reversing really does detach the mark:

```go
package main

import "fmt"

func reverseRunes(s string) string {
  rs := []rune(s)
  for i, j := 0, len(rs)-1; i < j; i, j = i+1, j-1 {
    rs[i], rs[j] = rs[j], rs[i]
  }
  return string(rs)
}

func main() {
  precomposed := "é"
  decomposed := "é"
  fmt.Printf("%q len=%d runes=%d\n", precomposed, len(precomposed), len([]rune(precomposed)))
  fmt.Printf("%q len=%d runes=%d\n", decomposed, len(decomposed), len([]rune(decomposed)))
  fmt.Println(precomposed == decomposed)
  fmt.Printf("%q\n", reverseRunes(decomposed))
}
```

Output:

```
"é" len=2 runes=1
"é" len=3 runes=2
false
"́e"
```

Two strings that render identically, compare unequal, and reverse differently. One
user-perceived character is a *grapheme cluster*, and the standard library has no
grapheme segmentation or normalization; those live in `golang.org/x/text` (and
`rivo/uniseg`). See 6.8.

## Case conversion is not byte-local

```go
package main

import (
  "fmt"
  "strings"
  "unicode"
)

func main() {
  fmt.Println(strings.ToUpper("héllo straße"))
  fmt.Println(strings.ToLower("HÉLLO STRASSE"))
  fmt.Printf("%q %d\n", strings.ToUpper("ß"), len(strings.ToUpper("ß")))
  fmt.Printf("%q\n", string(unicode.ToUpper('ß')))
  fmt.Println(strings.ToUpper("istanbul"))
  fmt.Println(strings.ToUpperSpecial(unicode.TurkishCase, "istanbul"))
  fmt.Println(strings.ToLowerSpecial(unicode.TurkishCase, "ISTANBUL"))
  fmt.Printf("%q %q\n", strings.ToTitle("hello"), strings.ToUpper("hello"))
}
```

Output:

```
HÉLLO STRAßE
héllo strasse
"ß" 2
"ß"
ISTANBUL
İSTANBUL
ıstanbul
"HELLO" "HELLO"
```

`strings.ToUpper("ß")` leaves it unchanged, because the correct uppercase is the
two-character `SS` and Go's `ToUpper` is a per-rune mapping that cannot change
length. In Turkish, dotless and dotted i are distinct letters, which is why
`ToUpperSpecial(unicode.TurkishCase, ...)` exists. More in 6.8.

## byte vs rune vs grapheme

| Unit | Go type | Count of `"👍🏽"` | Count of `"é"` (NFD) | Use for |
|---|---|---|---|---|
| byte | `byte` / `uint8` | 8 | 3 | buffers, offsets, I/O, `len` |
| code point (rune) | `rune` / `int32` | 2 | 2 | per-character logic, `range`, `unicode.*` |
| grapheme cluster | none in stdlib | 1 | 1 | cursor movement, truncation for display, column width |

## Quick Reference

| Form | Yields | Note |
|---|---|---|
| `for i := 0; i < len(s); i++` | `s[i]` is a `byte` | no decoding; fastest for ASCII |
| `for i, r := range s` | rune + **byte offset** | decodes; invalid bytes become U+FFFD, width 1 |
| `for _, b := range []byte(s)` | each byte | conversion elided by the compiler |
| `for i, r := range []rune(s)` | rune + rune index | allocates; use when you need the rune index |
| `utf8.DecodeRuneInString(s[i:])` | `(rune, size)` | explicit single-step decode |
| `utf8.DecodeLastRuneInString(s)` | `(rune, size)` | decode from the end |
| `utf8.RuneCountInString(s)` | `int` | code points, counting invalid bytes as 1 each |
| `utf8.ValidString(s)` | `bool` | the only way to detect bad bytes |
| `utf8.RuneLen(r)` | `int` | 1–4, or -1 if unencodable |
| `strings.ToValidUTF8(s, repl)` | `string` | replace each invalid run |

## Common Mistakes

**Using the `range` index as a character number.** It is a byte offset that skips;
maintain a separate counter if you need an ordinal.

**Slicing a string at an arbitrary byte offset to truncate it.** It can split a
rune; snap to a boundary with `utf8.RuneStart` or count runes (see 6.8).

**Assuming `[]rune(s)` makes reversal or indexing safe for any text.** Emoji
sequences and combining marks span multiple runes and break apart.

**Checking a length limit with `len(s)` on user input.** Use
`utf8.RuneCountInString` for a human-facing "characters" limit.

**Trusting external bytes to be UTF-8.** Validate with `utf8.ValidString` or
sanitize with `strings.ToValidUTF8` at the boundary.

## Exercises

1. Write `func runeOffsets(s string) []int` returning every byte offset at which a
   rune starts, using only `range`. Verify against
   `utf8.RuneCountInString` and show the result for `"a€b😀c"`.
2. Write `func validPrefixLen(b []byte) int` returning the length of the longest
   prefix that is complete valid UTF-8, using `utf8.FullRune` and `utf8.DecodeRune`.
   Test it with the three truncations of `[]byte("世")`.
3. Take `"héllo"` and print every one of the 21 slices `s[i:j]` with `%q` plus
   `utf8.ValidString`. Count how many are invalid and explain the pattern.
4. Implement `func reverseGraphemesNaive(s string) string` that keeps a combining
   mark attached to its base rune by treating `unicode.IsMark` runes as part of the
   preceding cluster. Show it fixing `"é"` and still failing on `"👨‍👩‍👧"`.
5. Benchmark three ways of counting runes — `range` with a counter,
   `utf8.RuneCountInString`, and `len([]rune(s))` — over a 1 MB mixed ASCII/CJK
   string, with `-benchmem`.

---

[← 6.1 string, []byte, and []rune](01-string-bytes-runes.md) · [Course Index](../README.md) · [6.3 The strings Package Reference →](03-strings-package.md)
