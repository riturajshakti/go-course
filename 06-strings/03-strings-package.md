[← 6.2 UTF-8, Indexing, and Iteration](02-utf8-iteration.md) · [Course Index](../README.md) · [6.4 strings.Builder, Replacer, and Reader →](04-builder-replacer.md)

# 6.3 — The strings Package Reference

## What You'll Learn

- Every function in `strings`, grouped by purpose, with its real behaviour on edge cases
- Why `Trim`'s second argument is a **set of runes**, not a prefix — the package's top misuse
- What `Split` does with an empty separator, a missing separator, and an empty input
- The `iter.Seq` variants (`SplitSeq`, `FieldsSeq`, `Lines`, …) and when they beat the slice versions

## Coming From Another Language

| Other | Go |
|---|---|
| JS/Python methods on the string | package functions: `strings.Trim(s, …)`, not `s.Trim(…)` |
| Python `s.strip("ab")` — a char set | `strings.Trim(s, "ab")` — same semantics, same trap |
| JS `s.trim()` | `strings.TrimSpace(s)` |
| Python `s.split(",")` on `""` → `[""]` | `strings.Split("", ",")` → `[""]` — identical |
| Python `s.split()` (whitespace runs) | `strings.Fields(s)` |
| JS `s.replace(a, b)` (first only) | `strings.Replace(s, a, b, 1)`; `ReplaceAll` for all |
| JS `s.indexOf` → -1 | `strings.Index` → -1 |

Everything returns a new string; nothing mutates. Index results are **byte**
offsets, so they feed straight into slicing.

## Searching

| Function | Signature | Returns | Note |
|---|---|---|---|
| `Contains` | `(s, substr string) bool` | bool | `substr == ""` is always true |
| `ContainsAny` | `(s, chars string) bool` | bool | true if **any rune** of `chars` appears |
| `ContainsRune` | `(s string, r rune) bool` | bool | single code point |
| `ContainsFunc` | `(s string, f func(rune) bool) bool` | bool | predicate over runes |
| `Index` | `(s, substr string) int` | byte offset or -1 | `""` → 0 |
| `IndexAny` | `(s, chars string) int` | byte offset or -1 | first position of any rune in `chars` |
| `IndexByte` | `(s string, c byte) int` | byte offset or -1 | fastest; byte, not rune |
| `IndexRune` | `(s string, r rune) int` | byte offset or -1 | handles multibyte |
| `IndexFunc` | `(s string, f func(rune) bool) int` | byte offset or -1 | first rune satisfying `f` |
| `LastIndex` | `(s, substr string) int` | byte offset or -1 | searches from the end |
| `LastIndexAny` | `(s, chars string) int` | byte offset or -1 | |
| `LastIndexByte` | `(s string, c byte) int` | byte offset or -1 | |
| `LastIndexFunc` | `(s string, f func(rune) bool) int` | byte offset or -1 | |
| `Count` | `(s, substr string) int` | int | non-overlapping; `""` → rune count + 1 |
| `EqualFold` | `(s, t string) bool` | bool | Unicode simple case-folding |
| `HasPrefix` | `(s, prefix string) bool` | bool | `""` → true |
| `HasSuffix` | `(s, suffix string) bool` | bool | `""` → true |

```go
package main

import (
  "fmt"
  "strings"
  "unicode"
)

func main() {
  s := "go gopher, go"
  fmt.Println(strings.Contains(s, "go"), strings.Contains(s, "rust"), strings.Contains(s, ""))
  fmt.Println(strings.ContainsAny(s, "xyz"), strings.ContainsAny(s, "xyo"), strings.ContainsAny(s, ""))
  fmt.Println(strings.ContainsRune(s, 'p'), strings.ContainsRune("日本", '本'))
  fmt.Println(strings.ContainsFunc(s, unicode.IsDigit), strings.ContainsFunc("a1", unicode.IsDigit))
  fmt.Println(strings.Index(s, "go"), strings.Index(s, "zz"), strings.Index(s, ""))
  fmt.Println(strings.IndexAny(s, "hp"), strings.IndexByte(s, 'p'), strings.IndexRune("日本", '本'))
  fmt.Println(strings.IndexFunc(s, unicode.IsPunct))
  fmt.Println(strings.LastIndex(s, "go"), strings.LastIndexAny(s, "og"), strings.LastIndexByte(s, 'g'))
  fmt.Println(strings.LastIndexFunc(s, unicode.IsSpace))
  fmt.Println(strings.Count(s, "go"), strings.Count("cheese", "e"), strings.Count("abc", ""))
  fmt.Println(strings.EqualFold("Go", "GO"), strings.EqualFold("ß", "ss"), strings.EqualFold("K", "K"))
  fmt.Println(strings.HasPrefix(s, "go"), strings.HasSuffix(s, "go"), strings.HasPrefix(s, ""))
}
```

Output:

```
true false true
false true false
true true
false true
0 -1 0
5 5 3
9
11 12 11
10
3 3 4
true false true
true true true
```

Four results worth pinning down. `Contains(s, "")` is true and `Index(s, "")` is 0,
so an empty needle always "matches" — guard against empty user-supplied search
terms. `ContainsAny(s, "")` is false, the opposite convention. `IndexRune("日本",
'本')` is 3, a byte offset, not a character position. `Count("abc", "")` is 4: one
empty match between and around each rune. And `EqualFold("K", "K")` is true
because U+212A KELVIN SIGN folds to `k`, while `EqualFold("ß", "ss")` is false
because simple folding never changes length.

## Splitting and joining

| Function | Signature | Returns | Note |
|---|---|---|---|
| `Split` | `(s, sep string) []string` | `[]string` | `sep == ""` splits into runes |
| `SplitN` | `(s, sep string, n int) []string` | `[]string` | `n > 0` caps count, last holds remainder; `n == 0` → nil; `n < 0` → all |
| `SplitAfter` | `(s, sep string) []string` | `[]string` | separator kept at the end of each piece |
| `SplitAfterN` | `(s, sep string, n int) []string` | `[]string` | as above with a cap |
| `SplitSeq` | `(s, sep string) iter.Seq[string]` | iterator | no slice allocated |
| `SplitAfterSeq` | `(s, sep string) iter.Seq[string]` | iterator | |
| `Fields` | `(s string) []string` | `[]string` | splits on runs of `unicode.IsSpace`; empty input → empty slice |
| `FieldsFunc` | `(s string, f func(rune) bool) []string` | `[]string` | runs of separators collapse |
| `FieldsSeq` | `(s string) iter.Seq[string]` | iterator | |
| `FieldsFuncSeq` | `(s string, f func(rune) bool) iter.Seq[string]` | iterator | |
| `Lines` | `(s string) iter.Seq[string]` | iterator | each line **keeps** its `\n` |
| `Join` | `(elems []string, sep string) string` | `string` | nil → `""` |
| `Cut` | `(s, sep string) (before, after string, found bool)` | 3 values | cuts at the **first** `sep` |
| `CutLast` | `(s, sep string) (before, after string, found bool)` | 3 values | cuts at the **last** `sep` |
| `CutPrefix` | `(s, prefix string) (after string, found bool)` | 2 values | |
| `CutSuffix` | `(s, suffix string) (before string, found bool)` | 2 values | |

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  fmt.Printf("%q\n", strings.Split("a,b,c", ","))
  fmt.Printf("%q\n", strings.Split("abc", ""))
  fmt.Printf("%q\n", strings.Split("日本", ""))
  fmt.Printf("%q\n", strings.Split("abc", ","))
  fmt.Printf("%q %d\n", strings.Split("", ","), len(strings.Split("", ",")))
  fmt.Printf("%q\n", strings.Split("a,,c", ","))
  fmt.Printf("%q\n", strings.SplitN("a,b,c,d", ",", 2))
  fmt.Printf("%q\n", strings.SplitN("a,b,c", ",", 0))
  fmt.Printf("%q\n", strings.SplitN("a,b,c", ",", -1))
  fmt.Printf("%q\n", strings.SplitAfter("a,b,c", ","))
  fmt.Printf("%q\n", strings.SplitAfterN("a,b,c", ",", 2))
  fmt.Printf("%q\n", strings.Fields("  foo   bar  baz  "))
  fmt.Printf("%q %d\n", strings.Fields("   "), len(strings.Fields("   ")))
  fmt.Printf("%q\n", strings.FieldsFunc("a1b2c", func(r rune) bool { return r >= '0' && r <= '9' }))
  fmt.Printf("%q\n", strings.Join([]string{"a", "b", "c"}, "-"))
  fmt.Printf("%q %q\n", strings.Join(nil, "-"), strings.Join([]string{"a"}, "-"))
}
```

Output:

```
["a" "b" "c"]
["a" "b" "c"]
["日" "本"]
["abc"]
[""] 1
["a" "" "c"]
["a" "b,c,d"]
[]
["a" "b" "c"]
["a," "b," "c"]
["a," "b,c"]
["foo" "bar" "baz"]
[] 0
["a" "b" "c"]
"a-b-c"
"" "a"
```

The four results that bite:

- `Split("abc", "")` splits into runes — it is *not* an error, and on `"日本"` it
  correctly yields runes, not bytes.
- `Split("abc", ",")` with no separator present returns a **1-element** slice, not
  an empty one. So `len(parts) >= 1` always; check `len(parts) == 2` when you expect
  exactly one separator, or better, use `Cut`.
- `Split("", ",")` returns `[""]`, length 1 — a slice containing one empty string.
  Splitting empty input and then iterating gives you one bogus empty record. This
  is the classic empty-CSV-line bug.
- `Fields("   ")` returns a length-0 slice, unlike `Split`. `Fields` collapses runs
  of whitespace and never produces empty elements, which is why it is right for
  parsing human-typed input and `Split(s, " ")` is wrong.

## Cut is better than Split for key=value

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  b, a, ok := strings.Cut("key=value=more", "=")
  fmt.Printf("%q %q %v\n", b, a, ok)
  b, a, ok = strings.Cut("novalue", "=")
  fmt.Printf("%q %q %v\n", b, a, ok)
  b, a, ok = strings.CutLast("key=value=more", "=")
  fmt.Printf("%q %q %v\n", b, a, ok)
  after, ok := strings.CutPrefix("prefix-body", "prefix-")
  fmt.Printf("%q %v\n", after, ok)
  after, ok = strings.CutPrefix("body", "prefix-")
  fmt.Printf("%q %v\n", after, ok)
  before, ok := strings.CutSuffix("body.txt", ".txt")
  fmt.Printf("%q %v\n", before, ok)
}
```

Output:

```
"key" "value=more" true
"novalue" "" false
"key=value" "more" true
"body" true
"body" false
"body" true
```

When not found, `Cut` returns the **whole string as `before`**, an empty `after`,
and false. The fifth line is `CutPrefix` with an absent prefix: it returns the
input unchanged alongside `found == false`, so the returned string is safe to use
either way — the bool is what tells you whether anything happened.

Idiomatic: `Cut` for "split on the first separator", which is almost always what
`SplitN(s, sep, 2)` was being used for.

## The iterator variants

These return `iter.Seq[string]` and allocate no slice. They exist for
`SplitSeq`, `SplitAfterSeq`, `FieldsSeq`, `FieldsFuncSeq`, and `Lines` — there is
no `TrimSeq` or `IndexSeq`.

```go
package main

import (
  "fmt"
  "slices"
  "strings"
)

func main() {
  fmt.Printf("%q\n", slices.Collect(strings.SplitSeq("a,b,c", ",")))
  fmt.Printf("%q\n", slices.Collect(strings.SplitAfterSeq("a,b,c", ",")))
  fmt.Printf("%q\n", slices.Collect(strings.FieldsSeq("  foo  bar ")))
  fmt.Printf("%q\n", slices.Collect(strings.FieldsFuncSeq("a1b2c", func(r rune) bool { return r == '1' || r == '2' })))
  fmt.Printf("%q\n", slices.Collect(strings.Lines("a\nb\nc")))
  fmt.Printf("%q\n", slices.Collect(strings.Lines("a\nb\n")))

  for line := range strings.Lines("x\ny\n") {
    fmt.Printf("line=%q\n", line)
    break
  }
}
```

Output:

```
["a" "b" "c"]
["a," "b," "c"]
["foo" "bar"]
["a" "b" "c"]
["a\n" "b\n" "c"]
["a\n" "b\n"]
line="x\n"
```

`Lines` keeps the trailing newline on each line — unlike `Split(s, "\n")`, which
strips it and adds a phantom empty final element when the input ends in `\n`. Note
`Lines("a\nb\n")` yields exactly two lines, which is the behaviour you actually
want. Idiomatic: use the `Seq` form when you may `break` early or when you are
feeding a loop and never need the slice.

## Trimming — the cutset is a SET OF RUNES

This is the most-misused part of the package. In `Trim`, `TrimLeft`, `TrimRight`,
and `TrimSpace`, the second argument is a set of individual runes, each of which is
removed repeatedly from the end. It is **not** a prefix or suffix string.

| Function | Signature | Second arg is | Note |
|---|---|---|---|
| `Trim` | `(s, cutset string) string` | rune **set** | strips from both ends |
| `TrimLeft` | `(s, cutset string) string` | rune **set** | |
| `TrimRight` | `(s, cutset string) string` | rune **set** | |
| `TrimSpace` | `(s string) string` | — | `unicode.IsSpace` on both ends |
| `TrimPrefix` | `(s, prefix string) string` | literal **string** | no-op if absent |
| `TrimSuffix` | `(s, suffix string) string` | literal **string** | no-op if absent |
| `TrimFunc` | `(s string, f func(rune) bool) string` | predicate | both ends |
| `TrimLeftFunc` | `(s string, f func(rune) bool) string` | predicate | |
| `TrimRightFunc` | `(s string, f func(rune) bool) string` | predicate | |

```go
package main

import (
  "fmt"
  "strings"
  "unicode"
)

func main() {
  fmt.Printf("%q\n", strings.Trim("xxhelloxx", "x"))
  fmt.Printf("%q\n", strings.Trim("  hi\t\n ", " \t\n"))
  fmt.Printf("%q\n", strings.TrimLeft("xoxo hello oxox", "ox"))
  fmt.Printf("%q\n", strings.TrimRight("xoxo hello oxox", "ox"))
  fmt.Printf("%q\n", strings.Trim("xoxo hello oxox", "ox"))
  fmt.Printf("%q\n", strings.TrimSpace("\t\n  hi \r\n "))
  fmt.Printf("%q\n", strings.TrimPrefix("xoxo hello", "xoxo "))
  fmt.Printf("%q\n", strings.TrimPrefix("hello", "xoxo "))
  fmt.Printf("%q\n", strings.TrimSuffix("file.txt", ".txt"))
  fmt.Printf("%q\n", strings.TrimSuffix("file.txt", ".md"))
  fmt.Printf("%q\n", strings.TrimFunc("123abc456", unicode.IsDigit))
  fmt.Printf("%q\n", strings.TrimLeftFunc("123abc456", unicode.IsDigit))
  fmt.Printf("%q\n", strings.TrimRightFunc("123abc456", unicode.IsDigit))
  fmt.Printf("%q\n", strings.Trim("héllo", "oéh"))
}
```

Output:

```
"hello"
"hi"
" hello oxox"
"xoxo hello "
" hello "
"hi"
"hello"
"hello"
"file"
"file.txt"
"abc"
"abc456"
"123abc"
"ll"
```

`TrimLeft("xoxo hello oxox", "ox")` eats the entire `xoxo` prefix — all four runes,
because each of them is in the set `{o, x}`. It did not look for the substring
`"ox"`. If you want to remove a literal prefix once, that is `TrimPrefix`, and it
leaves the string alone when the prefix is absent (line 8). Note `Trim("héllo",
"oéh")` gives `"ll"` — the cutset is runes, so multibyte `é` works correctly.

## Case and mapping

| Function | Signature | Returns | Note |
|---|---|---|---|
| `ToUpper` | `(s string) string` | `string` | per-rune; cannot change length (`ß` stays `ß`) |
| `ToLower` | `(s string) string` | `string` | |
| `ToTitle` | `(s string) string` | `string` | maps every rune to title case — not "Title Case Words" |
| `Title` | `(s string) string` | `string` | **deprecated**: naive word boundaries |
| `ToUpperSpecial` | `(c unicode.SpecialCase, s string) string` | `string` | e.g. `unicode.TurkishCase` |
| `ToLowerSpecial` | `(c unicode.SpecialCase, s string) string` | `string` | |
| `ToTitleSpecial` | `(c unicode.SpecialCase, s string) string` | `string` | |
| `ToValidUTF8` | `(s, replacement string) string` | `string` | each invalid run → `replacement` |
| `Map` | `(mapping func(rune) rune, s string) string` | `string` | return -1 from `mapping` to **drop** the rune |

```go
package main

import (
  "fmt"
  "strings"
  "unicode"
)

func main() {
  fmt.Printf("%q %q %q\n", strings.ToUpper("hello wörld"), strings.ToLower("HELLO WÖRLD"), strings.ToTitle("hello"))
  fmt.Printf("%q\n", strings.ToUpperSpecial(unicode.TurkishCase, "iii"))
  fmt.Printf("%q\n", strings.ToLowerSpecial(unicode.TurkishCase, "III"))
  fmt.Printf("%q\n", strings.ToTitleSpecial(unicode.TurkishCase, "iii"))
  fmt.Printf("%q\n", strings.ToValidUTF8("a\xffb", "�"))
  rot13 := func(r rune) rune {
    switch {
    case r >= 'a' && r <= 'z':
      return 'a' + (r-'a'+13)%26
    case r >= 'A' && r <= 'Z':
      return 'A' + (r-'A'+13)%26
    }
    return r
  }
  fmt.Printf("%q\n", strings.Map(rot13, "Hello, Gophers!"))
  drop := func(r rune) rune {
    if unicode.IsDigit(r) {
      return -1
    }
    return r
  }
  fmt.Printf("%q\n", strings.Map(drop, "a1b2c3"))
}
```

Output:

```
"HELLO WÖRLD" "hello wörld" "HELLO"
"İİİ"
"ııı"
"İİİ"
"a�b"
"Uryyb, Tbcuref!"
"abc"
```

`ToTitle("hello")` is `"HELLO"`, not `"Hello"` — title case of a *rune* differs from
upper case only for a handful of digraphs like `ǅ`. There is no stdlib replacement
for word-capitalizing `Title`; write it yourself with `Fields` or use
`golang.org/x/text/cases`.

## Building, replacing, repeating, comparing

| Function | Signature | Returns | Note |
|---|---|---|---|
| `Repeat` | `(s string, count int) string` | `string` | `count == 0` → `""`; **panics** if negative |
| `Replace` | `(s, old, new string, n int) string` | `string` | `n < 0` means all; `old == ""` inserts between every rune |
| `ReplaceAll` | `(s, old, new string) string` | `string` | `Replace(s, old, new, -1)` |
| `Clone` | `(s string) string` | `string` | fresh copy; breaks sharing with a big parent |
| `Compare` | `(a, b string) int` | -1, 0, 1 | only for sort comparators; prefer `<`/`==` |

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  fmt.Printf("%q %q %q\n", strings.Repeat("ab", 3), strings.Repeat("ab", 0), strings.Repeat("", 5))
  fmt.Printf("%q\n", strings.Replace("aaaa", "a", "b", 2))
  fmt.Printf("%q\n", strings.Replace("aaaa", "a", "b", -1))
  fmt.Printf("%q\n", strings.ReplaceAll("aaaa", "a", "b"))
  fmt.Printf("%q\n", strings.Replace("abc", "", "-", -1))
  fmt.Printf("%q\n", strings.Replace("abc", "x", "y", -1))
  c := strings.Clone("hello"[0:3])
  fmt.Printf("%q\n", c)
  fmt.Println(strings.Compare("a", "b"), strings.Compare("b", "a"), strings.Compare("", ""))
}
```

Output:

```
"ababab" "" ""
"bbaa"
"bbbb"
"bbbb"
"-a-b-c-"
"abc"
"hel"
-1 1 0
```

A negative count panics rather than returning empty:

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  fmt.Println(strings.Repeat("ab", -1))
}
```

This program panics on purpose. Output:

```
panic: strings: negative Repeat count

goroutine 1 [running]:
strings.Repeat({0x1027ac00f?, 0x0?}, 0x0?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/strings/strings.go:609 +0x4a0
main.main()
	/tmp/scratch/main.go:9 +0x2c
exit status 2
```

So `strings.Repeat(sep, n-1)` with `n == 0` from a computed length is a live crash;
clamp it.

## Quick Reference

| Need | Use |
|---|---|
| split on the first separator | `Cut` (not `SplitN(s, sep, 2)`) |
| split on the last separator | `CutLast` |
| split human-typed whitespace | `Fields` (not `Split(s, " ")`) |
| iterate lines of a blob | `Lines` (keeps `\n`) |
| remove a literal prefix/suffix | `TrimPrefix` / `TrimSuffix` |
| remove any of these characters | `Trim` / `TrimLeft` / `TrimRight` |
| strip surrounding whitespace | `TrimSpace` |
| case-insensitive equality | `EqualFold` (not `ToLower(a) == ToLower(b)`) |
| join with a separator | `Join` |
| drop or transform runes | `Map` (return -1 to drop) |
| detach a substring from a big parent | `Clone` |
| a sort comparator | `Compare` |

| Edge case | Result |
|---|---|
| `Index(s, "")` | 0 |
| `Contains(s, "")` | true |
| `ContainsAny(s, "")` | false |
| `Count(s, "")` | rune count + 1 |
| `Split("", sep)` | `[""]` — length 1 |
| `Split(s, "")` | one element per rune |
| `Split(s, sep)` with no `sep` | `[s]` — length 1 |
| `Fields("   ")` | length 0 |
| `Cut(s, sep)` not found | `(s, "", false)` |
| `TrimPrefix(s, p)` not present | `s` unchanged |
| `Repeat(s, 0)` | `""` |
| `Repeat(s, -1)` | panic |
| `Join(nil, sep)` | `""` |
| `ToUpper("ß")` | `"ß"` unchanged |

## Common Mistakes

**Using `TrimLeft(s, "prefix")` to remove a prefix.** It removes every leading rune
that appears anywhere in `"prefix"`; use `TrimPrefix`.

**Assuming `Split` returns an empty slice when the separator is missing.** It
returns a one-element slice holding the whole string; test `found` from `Cut`
instead.

**Iterating `Split("", ",")` as if it were empty.** It has one empty element; check
for empty input first.

**Comparing with `ToLower(a) == ToLower(b)` for case insensitivity.** Use
`EqualFold`, which handles folds like `K`/`K` that `ToLower` misses (6.8).

**Calling `Repeat` with a computed count that can go negative.** It panics; clamp
to zero.

## Exercises

1. Write `func ParseQuery(s string) map[string]string` for input like
   `"a=1&b=2&c"`, using only `SplitSeq` and `Cut`, and decide what `"c"` with no
   `=` should map to. Handle `""` without producing a bogus entry.
2. Show, with a table-driven test, four inputs on which `Trim(s, cutset)` and
   `TrimPrefix(s, cutset)` give different answers, and one on which they agree.
3. Write `func TitleWords(s string) string` capitalizing the first rune of each
   whitespace-separated word without using the deprecated `strings.Title`.
   Verify it on `"hello wörld"` and `"  spaced   out "`.
4. Write `func CountLines(s string) int` three ways — `Count(s, "\n")`,
   `len(Split(s, "\n"))`, and iterating `Lines` — and construct inputs (`""`,
   `"a"`, `"a\n"`, `"a\n\n"`) where all three disagree.
5. Benchmark `Index` against `IndexByte` for a single-byte needle over a 1 MB
   string, and `Contains` against a compiled `regexp` for a literal substring, with
   `-benchmem`.

---

[← 6.2 UTF-8, Indexing, and Iteration](02-utf8-iteration.md) · [Course Index](../README.md) · [6.4 strings.Builder, Replacer, and Reader →](04-builder-replacer.md)
