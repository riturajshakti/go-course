[← 13.7 encoding/json: Streaming, Tags, and Custom Marshalers](07-json-advanced.md) · [Course Index](../README.md) · [13.9 math, math/big, and math/rand →](09-math-rand.md)

# 13.8 — regexp

## What You'll Learn

- `regexp` is **RE2**: linear time, no backtracking, therefore no backreferences and no lookahead
- The `Find[All][String][Submatch][Index]` naming scheme — 16 methods that are one system
- Named capture groups extracted into a map, and the `$1` expansion rules with the literal-`$` trap
- Why you compile **once** at package level: a measured 24x difference

## Coming From Another Language

| Other | Go |
|---|---|
| JS `/re/.test(s)` | `re.MatchString(s)` |
| JS `s.match(/re/g)` | `re.FindAllString(s, -1)` |
| JS `(?<name>x)` lookahead `(?=x)` | named groups **yes**, lookahead **never** |
| Python `re.sub(r'\1', ...)` backreference | not supported — RE2 has no backreferences |
| Python's catastrophic backtracking | **impossible**; matching is O(len) |
| PCRE `\d` matching Unicode digits | `\d`/`\w` are **ASCII only**; use `\p{Nd}`/`\p{L}` |
| `re.compile` cached implicitly by Python | no cache; a package-level `var` is on you |

Go's `regexp` implements RE2, which guarantees matching in time linear in the length of
the input times the size of the pattern. That single design decision is the whole story
for someone arriving from JS or Python: it is why `(a+)+b` against a long string of `a`s
cannot hang your server, and it is also why backreferences (`\1`), lookahead (`(?=`) and
lookbehind (`(?<=`) are not merely missing but *unimplementable* — each requires
backtracking. The practical consequence is that a pattern you paste from a PCRE cheat
sheet may simply fail to compile, and the fix is usually to do that part of the work in Go
code instead of in the pattern. The second Go-specific detail: a `*regexp.Regexp` is
safe for concurrent use and expensive to construct, so it belongs in a package-level `var`.

## Compile, MustCompile, and what RE2 rejects

```go
package main

import (
  "fmt"
  "regexp"
)

func main() {
  fmt.Println("--- Compile vs MustCompile ---")
  re, err := regexp.Compile(`\d+`)
  fmt.Println("valid:", re, err)
  re, err = regexp.Compile(`[a-z`)
  fmt.Println("invalid:", re, err)
  fmt.Printf("  error type: %T\n", err)

  fmt.Println("\n--- patterns RE2 rejects ---")
  for _, p := range []string{
    `(\w+) \1`,
    `(?=foo)`,
    `(?!foo)`,
    `(?<=foo)bar`,
    `a**`,
    `x{2,1}`,
    `\C`,
    `[z-a]`,
    `(?#comment)`,
    `\p{Nope}`,
  } {
    _, err := regexp.Compile(p)
    fmt.Printf("  %-16q %v\n", p, err)
  }

  fmt.Println("\n--- patterns RE2 accepts ---")
  for _, p := range []string{
    `(?P<name>\w+)`,
    `(?<name>\w+)`,
    `(?i)hello`,
    `(?s).+`,
    `(?m)^line$`,
    `(?U)a+`,
    `\p{L}+`,
    `[[:alpha:]]+`,
    `a{2,4}`,
    `(?:group)`,
    `\b\w+\b`,
  } {
    _, err := regexp.Compile(p)
    fmt.Printf("  %-16q %v\n", p, err)
  }

  fmt.Println("\n--- CompilePOSIX ---")
  rp, err := regexp.CompilePOSIX(`a+`)
  fmt.Println("  POSIX a+:", rp, err)
  _, err = regexp.CompilePOSIX(`(?i)a`)
  fmt.Println("  POSIX with (?i):", err)
}
```

Output:

```
--- Compile vs MustCompile ---
valid: \d+ <nil>
invalid: <nil> error parsing regexp: missing closing ]: `[a-z`
  error type: *syntax.Error

--- patterns RE2 rejects ---
  "(\\w+) \\1"     error parsing regexp: invalid escape sequence: `\1`
  "(?=foo)"        error parsing regexp: invalid or unsupported Perl syntax: `(?=`
  "(?!foo)"        error parsing regexp: invalid or unsupported Perl syntax: `(?!`
  "(?<=foo)bar"    error parsing regexp: invalid named capture: `(?<=foo)bar`
  "a**"            error parsing regexp: invalid nested repetition operator: `**`
  "x{2,1}"         error parsing regexp: invalid repeat count: `{2,1}`
  "\\C"            error parsing regexp: invalid escape sequence: `\C`
  "[z-a]"          error parsing regexp: invalid character class range: `z-a`
  "(?#comment)"    error parsing regexp: invalid or unsupported Perl syntax: `(?#`
  "\\p{Nope}"      error parsing regexp: invalid character class range: `\p{Nope}`

--- patterns RE2 accepts ---
  "(?P<name>\\w+)" <nil>
  "(?<name>\\w+)"  <nil>
  "(?i)hello"      <nil>
  "(?s).+"         <nil>
  "(?m)^line$"     <nil>
  "(?U)a+"         <nil>
  "\\p{L}+"        <nil>
  "[[:alpha:]]+"   <nil>
  "a{2,4}"         <nil>
  "(?:group)"      <nil>
  "\\b\\w+\\b"     <nil>

--- CompilePOSIX ---
  POSIX a+: a+ <nil>
  POSIX with (?i): error parsing regexp: missing argument to repetition operator: `?`
```

The rejection messages are precise and quote the offending fragment, which makes them safe
to surface to a user supplying a pattern. `\1` is reported as an *invalid escape*, not as
"backreferences unsupported" — worth recognizing.

Note the modern spelling: **both** `(?P<name>...)` (the original RE2/Python form) and
`(?<name>...)` (the PCRE/JS form) compile on this toolchain. `(?<=` still fails, because
that is lookbehind, not a named group — and the error says `invalid named capture`, which
is confusing until you know the parser tried to read it as one.

`regexp.Compile` returns `(*Regexp, error)`; `regexp.MustCompile` panics instead, and is
what you use for a pattern that is a constant in your source. `CompilePOSIX` restricts to
POSIX ERE (no `(?i)`, no `\d`, leftmost-longest semantics) and is almost never what you
want.

```go
package main

import (
  "fmt"
  "regexp"
)

var bad = regexp.MustCompile(`user_(?P<id>\d+`)

func main() {
  fmt.Println(bad)
}
```

This panics on purpose. Output:

```
panic: regexp: Compile(`user_(?P<id>\d+`): error parsing regexp: missing closing ): `user_(?P<id>\d+`

goroutine 1 [running]:
regexp.MustCompile({0x1048c1abf, 0xf})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/regexp/regexp.go:312 +0xb0
main.init()
	/tmp/scratch/main.go:8 +0x28
exit status 2
```

The panic happens in `init()`, before `main` runs, which is the point: a bad pattern in a
package-level `MustCompile` crashes at startup rather than on the first unlucky request.
Idiomatic: `MustCompile` at package level for literal patterns, `Compile` with error
handling for anything from config or user input.

## Syntax reference

| Construct | Matches |
|---|---|
| `.` | any character except `\n` (any, with `(?s)`) |
| `[abc]` / `[^abc]` | one of / none of |
| `[a-z0-9]` | ranges |
| `\d` `\D` | ASCII digit / non-digit |
| `\w` `\W` | ASCII `[0-9A-Za-z_]` / its complement |
| `\s` `\S` | `[\t\n\f\r ]` / its complement |
| `\p{L}` `\pL` | any Unicode letter (`\P{L}` for the complement) |
| `\p{Nd}` `\p{Lu}` `\p{Han}` `\p{Cyrillic}` | Unicode category or script |
| `[[:alpha:]]` `[[:digit:]]` `[[:space:]]` `[[:punct:]]` | POSIX classes (ASCII) |
| `^` `$` | start / end of **text** (of line, with `(?m)`) |
| `\b` `\B` | ASCII word boundary / non-boundary |
| `x*` `x+` `x?` | 0+, 1+, 0 or 1 — greedy |
| `x*?` `x+?` `x??` | the same, non-greedy |
| `x{n}` `x{n,}` `x{n,m}` | exact / at least / range |
| `x\|y` | alternation (leftmost-first, not longest) |
| `(x)` | capturing group |
| `(?:x)` | non-capturing group |
| `(?P<n>x)` / `(?<n>x)` | named capturing group |
| `(?i)` `(?s)` `(?m)` `(?U)` | flags from here on |
| `(?i:x)` | flags scoped to `x` |
| `(?-i)` `(?-i:x)` | turn a flag off |
| `\$` `\.` etc. | escape a metacharacter |

Not available, at all: backreferences, lookahead, lookbehind, atomic groups, possessive
quantifiers, recursion, conditionals, `\G`, inline comments.

## The Find family is one naming scheme

There are 16 `Find*` methods, and memorizing them individually is wasted effort. They are
a single grid:

```
Find [All] [String] [Submatch] [Index]
      |      |         |         |
      |      |         |         +-- return []int offsets instead of the text
      |      |         +------------ include the capture groups
      |      +---------------------- operate on string instead of []byte
      +----------------------------- every match, not just the leftmost
```

| Component | Effect on the return type |
|---|---|
| base | one leftmost match |
| `All` | a slice of matches; takes an extra `n int` (`-1` = all) |
| `String` | `string` in and out, instead of `[]byte` |
| `Submatch` | one more level of slice: `[whole, g1, g2, …]` |
| `Index` | `[]int` byte offsets, in `start, end` pairs |

| Method | Returns on a match |
|---|---|
| `Find(b)` / `FindString(s)` | `[]byte` / `string` |
| `FindIndex(b)` / `FindStringIndex(s)` | `[]int{start, end}` |
| `FindSubmatch(b)` / `FindStringSubmatch(s)` | `[][]byte` / `[]string` |
| `FindSubmatchIndex` / `FindStringSubmatchIndex` | `[]int`, 2 entries per group |
| `FindAll(b, n)` / `FindAllString(s, n)` | `[][]byte` / `[]string` |
| `FindAllIndex` / `FindAllStringIndex` | `[][]int` |
| `FindAllSubmatch` / `FindAllStringSubmatch` | `[][][]byte` / `[][]string` |
| `FindAllSubmatchIndex` / `FindAllStringSubmatchIndex` | `[][]int` |
| `FindReaderIndex(r)` / `FindReaderSubmatchIndex(r)` | `[]int` — the only reader finders |

```go
package main

import (
  "fmt"
  "regexp"
  "strings"
)

var re = regexp.MustCompile(`(\w+)@(\w+)\.com`)

const text = "mail a@x.com and b@y.com now"

func main() {
  fmt.Println("--- matching ---")
  fmt.Println("MatchString:      ", re.MatchString(text))
  fmt.Println("MatchReader:      ", re.MatchReader(strings.NewReader(text)))
  ok, err := regexp.MatchString(`\d+`, "abc 123")
  fmt.Println("regexp.MatchString:", ok, err)
  ok, err = regexp.MatchString(`[`, "x")
  fmt.Println("package-level bad pattern:", ok, err)
  fmt.Println("no match:         ", re.MatchString("nothing here"))

  fmt.Println("\n--- Find (leftmost one) ---")
  fmt.Printf("FindString:            %q\n", re.FindString(text))
  fmt.Println("FindStringIndex:      ", re.FindStringIndex(text))
  fmt.Printf("FindStringSubmatch:    %q\n", re.FindStringSubmatch(text))
  fmt.Println("FindStringSubmatchIndex:", re.FindStringSubmatchIndex(text))

  fmt.Println("\n--- FindAll (n = -1 means all) ---")
  fmt.Printf("FindAllString(-1):     %q\n", re.FindAllString(text, -1))
  fmt.Printf("FindAllString(1):      %q\n", re.FindAllString(text, 1))
  fmt.Printf("FindAllString(0):      %q\n", re.FindAllString(text, 0))
  fmt.Println("FindAllStringIndex:   ", re.FindAllStringIndex(text, -1))
  fmt.Printf("FindAllStringSubmatch: %q\n", re.FindAllStringSubmatch(text, -1))
  fmt.Println("FindAllStringSubmatchIndex:", re.FindAllStringSubmatchIndex(text, -1))

  fmt.Println("\n--- the NO-MATCH returns ---")
  none := "nothing"
  fmt.Printf("FindString:            %q (empty string, NOT nil)\n", re.FindString(none))
  fmt.Println("Find:                 ", re.Find([]byte(none)), "(nil)")
  fmt.Println("FindStringIndex:      ", re.FindStringIndex(none), "(nil)")
  fmt.Println("FindStringSubmatch:   ", re.FindStringSubmatch(none), "(nil)")
  fmt.Println("FindAllString:        ", re.FindAllString(none, -1), "(nil)")
  fmt.Println("len of nil submatch:  ", len(re.FindStringSubmatch(none)))

  fmt.Println("\n--- an unmatched optional group ---")
  opt := regexp.MustCompile(`(a)(b)?`)
  m := opt.FindStringSubmatch("a")
  fmt.Printf("match=%q len=%d group2 empty=%v\n", m, len(m), m[2] == "")
  fmt.Println("index form:", opt.FindStringSubmatchIndex("a"))
}
```

Output:

```
--- matching ---
MatchString:       true
MatchReader:       true
regexp.MatchString: true <nil>
package-level bad pattern: false error parsing regexp: missing closing ]: `[`
no match:          false

--- Find (leftmost one) ---
FindString:            "a@x.com"
FindStringIndex:       [5 12]
FindStringSubmatch:    ["a@x.com" "a" "x"]
FindStringSubmatchIndex: [5 12 5 6 7 8]

--- FindAll (n = -1 means all) ---
FindAllString(-1):     ["a@x.com" "b@y.com"]
FindAllString(1):      ["a@x.com"]
FindAllString(0):      []
FindAllStringIndex:    [[5 12] [17 24]]
FindAllStringSubmatch: [["a@x.com" "a" "x"] ["b@y.com" "b" "y"]]
FindAllStringSubmatchIndex: [[5 12 5 6 7 8] [17 24 17 18 19 20]]

--- the NO-MATCH returns ---
FindString:            "" (empty string, NOT nil)
Find:                  [] (nil)
FindStringIndex:       [] (nil)
FindStringSubmatch:    [] (nil)
FindAllString:         [] (nil)
len of nil submatch:   0

--- an unmatched optional group ---
match=["a" "a" ""] len=3 group2 empty=true
index form: [0 1 0 1 -1 -1]
```

The three methods you actually use are `MatchString`, `FindStringSubmatch` and
`FindAllString`. The rest exist for `[]byte` inputs (no allocation when your data is
already bytes) and for the `Index` forms (to slice the original rather than copy out of it).

**The no-match returns.** `FindString` returns `""`, which is indistinguishable from
matching an empty string — so never test for a match with `FindString(s) != ""`. Every
other `Find*` returns `nil`, so `if m := re.FindStringSubmatch(s); m != nil` is the
idiomatic guard. `MatchString` is the right call when you only need the boolean.

**`n` in the `All` variants.** `-1` means every match; a positive `n` caps it; and `0`
returns an empty non-nil slice, which is almost certainly a bug in your code rather than
an intent.

**An unmatched optional group** is `""` in the string form and `-1 -1` in the index form —
the only way to distinguish "matched the empty string" from "did not participate".

`regexp.MatchString(pattern, s)` is the package-level convenience that compiles on every
call. It returns an error for a bad pattern, and you should essentially never use it: see
the benchmark below.

## Replacing

| Method | Expansion | Takes |
|---|---|---|
| `ReplaceAllString(src, repl)` | `$1`, `${name}` | `string` |
| `ReplaceAll(src, repl)` | `$1`, `${name}` | `[]byte` |
| `ReplaceAllLiteralString(src, repl)` | **none** | `string` |
| `ReplaceAllLiteral(src, repl)` | **none** | `[]byte` |
| `ReplaceAllStringFunc(src, f)` | n/a | `func(string) string` |
| `ReplaceAllFunc(src, f)` | n/a | `func([]byte) []byte` |

Expansion rules: `$1` is group 1, `${1}` the same, `$name`/`${name}` a named group, `$$` a
literal `$`, and an unknown group expands to the **empty string** with no error.

```go
package main

import (
  "fmt"
  "regexp"
  "strings"
)

var kv = regexp.MustCompile(`(?P<key>\w+)=(?P<val>\w+)`)

func main() {
  const in = "a=1 b=2 c=3"

  fmt.Println("--- ReplaceAllString and $ expansion ---")
  fmt.Printf("$1/$2:        %q\n", kv.ReplaceAllString(in, "$2/$1"))
  fmt.Printf("${key}:       %q\n", kv.ReplaceAllString(in, "${key}:${val}"))
  fmt.Printf("$key bare:    %q\n", kv.ReplaceAllString(in, "$key=$val"))
  fmt.Printf("$1x ambiguous: %q\n", kv.ReplaceAllString(in, "$1x"))
  fmt.Printf("${1}x fixed:  %q\n", kv.ReplaceAllString(in, "${1}x"))
  fmt.Printf("literal $$:   %q\n", kv.ReplaceAllString(in, "$$${val}"))
  fmt.Printf("unknown $9:   %q\n", kv.ReplaceAllString(in, "[$9]"))

  fmt.Println("\n--- ReplaceAllLiteralString: no expansion ---")
  fmt.Printf("%q\n", kv.ReplaceAllLiteralString(in, "$2/$1"))

  fmt.Println("\n--- ReplaceAllStringFunc ---")
  fmt.Printf("%q\n", kv.ReplaceAllStringFunc(in, strings.ToUpper))

  fmt.Println("\n--- the Func gets the WHOLE match, not groups ---")
  fmt.Printf("%q\n", kv.ReplaceAllStringFunc(in, func(m string) string {
    g := kv.FindStringSubmatch(m)
    return g[2] + "=" + g[1]
  }))

  fmt.Println("\n--- Split ---")
  sp := regexp.MustCompile(`\s*,\s*`)
  fmt.Printf("n=-1: %q\n", sp.Split("a, b ,c,  d", -1))
  fmt.Printf("n=2:  %q\n", sp.Split("a, b ,c,  d", 2))
  fmt.Printf("n=0:  %q (nil)\n", sp.Split("a,b", 0))
  fmt.Printf("n=1:  %q\n", sp.Split("a,b,c", 1))
  fmt.Printf("no separator present: %q\n", sp.Split("abc", -1))
  empty := regexp.MustCompile(``)
  fmt.Printf("empty pattern: %q\n", empty.Split("abc", -1))
  fmt.Printf("leading/trailing sep: %q\n", sp.Split(",a,", -1))

  fmt.Println("\n--- Expand / ExpandString ---")
  var out []byte
  for _, m := range kv.FindAllStringSubmatchIndex(in, -1) {
    out = kv.ExpandString(out, "$key -> $val\n", in, m)
  }
  fmt.Printf("%s", out)
}
```

Output:

```
--- ReplaceAllString and $ expansion ---
$1/$2:        "1/a 2/b 3/c"
${key}:       "a:1 b:2 c:3"
$key bare:    "a=1 b=2 c=3"
$1x ambiguous: "  "
${1}x fixed:  "ax bx cx"
literal $$:   "$1 $2 $3"
unknown $9:   "[] [] []"

--- ReplaceAllLiteralString: no expansion ---
"$2/$1 $2/$1 $2/$1"

--- ReplaceAllStringFunc ---
"A=1 B=2 C=3"

--- the Func gets the WHOLE match, not groups ---
"1=a 2=b 3=c"

--- Split ---
n=-1: ["a" "b" "c" "d"]
n=2:  ["a" "b ,c,  d"]
n=0:  [] (nil)
n=1:  ["a,b,c"]
no separator present: ["abc"]
empty pattern: ["a" "b" "c"]
leading/trailing sep: ["" "a" ""]

--- Expand / ExpandString ---
a -> 1
b -> 2
c -> 3
```

The `$1x` line is the trap worth memorizing. `"$1x"` produced `"  "` — two spaces and
nothing else — because `$1x` parses as the group *named* `1x`, which does not exist, and
an unknown group expands to empty. The `${1}x` form gives the intended `"ax bx cx"`.
**Always brace the group name** in a replacement that is followed by any word character.

`$9` on a pattern with two groups also expands silently to `""`. There is no error for
referring to a group that does not exist, in either form.

`ReplaceAllLiteralString` is what you want when the replacement is user-supplied text that
may contain a `$` — otherwise `"costs $5"` becomes `"costs "`.

`ReplaceAllStringFunc` passes the **whole match**, not the submatches — the most common
complaint about it. Re-run `FindStringSubmatch` on the match inside the callback, as above;
it is cheap because the match is short. If you need groups for every replacement, build the
result with `FindAllStringSubmatchIndex` and a `strings.Builder`.

`Split` with `n=0` returns nil, and `n=1` returns the whole input unsplit. An empty pattern
splits between every rune. Leading and trailing separators produce empty strings at the
ends, exactly like `strings.Split`.

## Named groups into a map

```go
package main

import (
  "fmt"
  "regexp"
)

var logRe = regexp.MustCompile(
  `^(?P<ip>\S+) \[(?P<ts>[^\]]+)\] "(?P<method>[A-Z]+) (?P<path>\S+)" (?P<status>\d{3}) (?P<bytes>\d+|-)$`)

func namedGroups(re *regexp.Regexp, s string) map[string]string {
  m := re.FindStringSubmatch(s)
  if m == nil {
    return nil
  }
  out := make(map[string]string, len(m))
  for i, name := range re.SubexpNames() {
    if i > 0 && name != "" {
      out[name] = m[i]
    }
  }
  return out
}

func main() {
  line := `10.0.0.1 [2026-03-07T15:04:05Z] "GET /api/users?p=2" 200 4096`

  fmt.Println("--- SubexpNames / NumSubexp / SubexpIndex ---")
  fmt.Printf("SubexpNames: %q\n", logRe.SubexpNames())
  fmt.Println("NumSubexp:  ", logRe.NumSubexp())
  for _, n := range []string{"ip", "status", "nope"} {
    fmt.Printf("SubexpIndex(%q) = %d\n", n, logRe.SubexpIndex(n))
  }

  fmt.Println("\n--- named groups into a map ---")
  g := namedGroups(logRe, line)
  for _, k := range []string{"ip", "ts", "method", "path", "status", "bytes"} {
    fmt.Printf("  %-7s %q\n", k, g[k])
  }
  fmt.Println("  no match returns nil:", namedGroups(logRe, "garbage") == nil)

  fmt.Println("\n--- config methods ---")
  fmt.Println("String():   ", regexp.MustCompile(`a+b`).String())
  fmt.Println("NumSubexp():", regexp.MustCompile(`(a)(b)(?:c)`).NumSubexp())

  alt := regexp.MustCompile(`a|ab`)
  fmt.Printf("a|ab default:  %q\n", alt.FindString("ab"))
  longest := regexp.MustCompile(`a|ab`)
  longest.Longest()
  fmt.Printf("a|ab Longest(): %q\n", longest.FindString("ab"))

  for _, p := range []string{`x`, `a+b`, `abc`, `(?i)abc`} {
    pre, complete := regexp.MustCompile(p).LiteralPrefix()
    fmt.Printf("LiteralPrefix(%-8q) = %q complete=%v\n", p, pre, complete)
  }

  fmt.Println("\n--- QuoteMeta ---")
  for _, s := range []string{`a.b*c`, `1+1=2`, `price: $5 (50%)`, `plain`} {
    fmt.Printf("  %-18q -> %q\n", s, regexp.QuoteMeta(s))
  }
  user := "a.b"
  fmt.Println("  unquoted matches too much:", regexp.MustCompile(user).MatchString("axb"))
  fmt.Println("  quoted is exact:         ",
    regexp.MustCompile(regexp.QuoteMeta(user)).MatchString("axb"),
    regexp.MustCompile(regexp.QuoteMeta(user)).MatchString("a.b"))
}
```

Output:

```
--- SubexpNames / NumSubexp / SubexpIndex ---
SubexpNames: ["" "ip" "ts" "method" "path" "status" "bytes"]
NumSubexp:   6
SubexpIndex("ip") = 1
SubexpIndex("status") = 5
SubexpIndex("nope") = -1

--- named groups into a map ---
  ip      "10.0.0.1"
  ts      "2026-03-07T15:04:05Z"
  method  "GET"
  path    "/api/users?p=2"
  status  "200"
  bytes   "4096"
  no match returns nil: true

--- config methods ---
String():    a+b
NumSubexp(): 2
a|ab default:  "a"
a|ab Longest(): "ab"
LiteralPrefix("x"     ) = "x" complete=true
LiteralPrefix("a+b"   ) = "a" complete=false
LiteralPrefix("abc"   ) = "abc" complete=true
LiteralPrefix("(?i)abc") = "" complete=false

--- QuoteMeta ---
  "a.b*c"            -> "a\\.b\\*c"
  "1+1=2"            -> "1\\+1=2"
  "price: $5 (50%)"  -> "price: \\$5 \\(50%\\)"
  "plain"            -> "plain"
  unquoted matches too much: true
  quoted is exact:          false true
```

`SubexpNames()` is parallel to the submatch slice, index 0 being the whole match and
therefore always `""`. Unnamed groups also give `""`, which is why the `name != ""` check
matters. `SubexpIndex(name)` returns `-1` for a name that does not exist, so
`m[re.SubexpIndex("typo")]` panics — a helper like `namedGroups` is worth the few lines.

`alt.FindString("ab")` returns `"a"` by default: Go's default is **leftmost-first** like
Perl, so `a|ab` prefers the first alternative that matches at the leftmost position.
`Longest()` switches the whole `*Regexp` to leftmost-longest (POSIX) semantics and makes it
return `"ab"`. Call it immediately after compiling and never concurrently with a match.

`regexp.QuoteMeta` is mandatory for any pattern fragment that comes from outside your code.
Without it `"a.b"` matched `"axb"`; with it, only the literal `"a.b"`.

## Inline flags

```go
package main

import (
  "fmt"
  "regexp"
)

func main() {
  const multi = "first line\nSECOND line\nthird"

  fmt.Println("--- (?i) case-insensitive ---")
  fmt.Println("  without:", regexp.MustCompile(`second`).MatchString(multi))
  fmt.Println("  with:   ", regexp.MustCompile(`(?i)second`).MatchString(multi))
  fmt.Printf("  scoped: %q\n", regexp.MustCompile(`(?i:ab)c`).FindAllString("ABc ABC abc", -1))

  fmt.Println("\n--- (?s) dot matches newline ---")
  fmt.Printf("  without: %q\n", regexp.MustCompile(`first.*third`).FindString(multi))
  fmt.Printf("  with:    %q\n", regexp.MustCompile(`(?s)first.*third`).FindString(multi))

  fmt.Println("\n--- (?m) ^ and $ match at line boundaries ---")
  fmt.Printf("  without: %q\n", regexp.MustCompile(`^\w+`).FindAllString(multi, -1))
  fmt.Printf("  with:    %q\n", regexp.MustCompile(`(?m)^\w+`).FindAllString(multi, -1))
  fmt.Printf("  (?m)$:   %q\n", regexp.MustCompile(`(?m)\w+$`).FindAllString(multi, -1))

  fmt.Println("\n--- (?U) swap greedy and non-greedy ---")
  html := "<a><b>"
  fmt.Printf("  <.+>:      %q\n", regexp.MustCompile(`<.+>`).FindString(html))
  fmt.Printf("  <.+?>:     %q\n", regexp.MustCompile(`<.+?>`).FindString(html))
  fmt.Printf("  (?U)<.+>:  %q\n", regexp.MustCompile(`(?U)<.+>`).FindString(html))
  fmt.Printf("  (?U)<.+?>: %q\n", regexp.MustCompile(`(?U)<.+?>`).FindString(html))

  fmt.Println("\n--- combined, and turning a flag off ---")
  fmt.Printf("  (?is): %q\n", regexp.MustCompile(`(?is)FIRST.*THIRD`).FindString(multi))
  fmt.Printf("  (?im)^s: %q\n", regexp.MustCompile(`(?im)^s\w+`).FindAllString(multi, -1))
  fmt.Printf("  (?i)a(?-i:B)c on AbC: %v\n",
    regexp.MustCompile(`(?i)a(?-i:B)c`).MatchString("AbC"))
  fmt.Printf("  (?i)a(?-i:B)c on aBc: %v\n",
    regexp.MustCompile(`(?i)a(?-i:B)c`).MatchString("aBc"))

  fmt.Println("\n--- non-ASCII ---")
  const uni = "café Ünïcode 日本語 Привет 123"
  fmt.Printf(`\w+:       %q`+"\n", regexp.MustCompile(`\w+`).FindAllString(uni, -1))
  fmt.Printf(`\p{L}+:    %q`+"\n", regexp.MustCompile(`\p{L}+`).FindAllString(uni, -1))
  fmt.Printf(`\p{Han}+:  %q`+"\n", regexp.MustCompile(`\p{Han}+`).FindAllString(uni, -1))
  fmt.Printf(`\p{Cyrillic}+: %q`+"\n", regexp.MustCompile(`\p{Cyrillic}+`).FindAllString(uni, -1))
  fmt.Printf(`[[:alpha:]]+: %q`+"\n", regexp.MustCompile(`[[:alpha:]]+`).FindAllString(uni, -1))
  fmt.Printf("(?i) on é:  %v\n", regexp.MustCompile(`(?i)CAFÉ`).MatchString("café"))
  fmt.Printf("indexes are BYTE offsets: %v for %q\n",
    regexp.MustCompile(`code`).FindStringIndex(uni), "code")
}
```

Output:

```
--- (?i) case-insensitive ---
  without: false
  with:    true
  scoped: ["ABc" "abc"]

--- (?s) dot matches newline ---
  without: ""
  with:    "first line\nSECOND line\nthird"

--- (?m) ^ and $ match at line boundaries ---
  without: ["first"]
  with:    ["first" "SECOND" "third"]
  (?m)$:   ["line" "line" "third"]

--- (?U) swap greedy and non-greedy ---
  <.+>:      "<a><b>"
  <.+?>:     "<a>"
  (?U)<.+>:  "<a>"
  (?U)<.+?>: "<a><b>"

--- combined, and turning a flag off ---
  (?is): "first line\nSECOND line\nthird"
  (?im)^s: ["SECOND"]
  (?i)a(?-i:B)c on AbC: false
  (?i)a(?-i:B)c on aBc: true

--- non-ASCII ---
\w+:       ["caf" "n" "code" "123"]
\p{L}+:    ["café" "Ünïcode" "日本語" "Привет"]
\p{Han}+:  ["日本語"]
\p{Cyrillic}+: ["Привет"]
[[:alpha:]]+: ["caf" "n" "code"]
(?i) on é:  true
indexes are BYTE offsets: [11 15] for "code"
```

Flags apply from where they appear to the end of the enclosing group; `(?i:x)` scopes them
to `x` and `(?-i:x)` turns them off, which is how `(?i)a(?-i:B)c` matches `aBc` but not
`AbC`.

The non-ASCII section is the practically important one. **`\w` and `\d` are ASCII-only.**
`\w+` on `"café Ünïcode"` returned `["caf" "n" "code"]` — it stopped at `é`, skipped it,
matched `n`, stopped at `ï`, and so on. So did `[[:alpha:]]`, which is a POSIX class and
also ASCII. For real text you need `\p{L}` (any letter), `\p{Nd}` (any decimal digit) or a
script like `\p{Han}`. This silently mangles any pattern applied to user-supplied names.

`(?i)` *does* work across Unicode case pairs — `(?i)CAFÉ` matches `café` — so the
case-folding is Unicode-aware even though the shorthand classes are not.

All indexes are **byte** offsets, not rune offsets: `code` starts at byte 11 of
`"café Ünïcode …"` even though it is rune 9. Slicing the original string with them is
correct; counting characters with them is not (6.2).

## Performance

Two measurements that change how you write regexp code.

```go
package main

import (
  "regexp"
  "strings"
  "testing"
)

var pkgRe = regexp.MustCompile(`^user_(\d+)$`)
var prefixRe = regexp.MustCompile(`^user_`)

const input = "user_12345"

func BenchmarkMustCompileInLoop(b *testing.B) {
  for b.Loop() {
    re := regexp.MustCompile(`^user_(\d+)$`)
    if !re.MatchString(input) {
      b.Fatal("no match")
    }
  }
}

func BenchmarkPackageLevelVar(b *testing.B) {
  for b.Loop() {
    if !pkgRe.MatchString(input) {
      b.Fatal("no match")
    }
  }
}

func BenchmarkRegexpHasPrefix(b *testing.B) {
  for b.Loop() {
    if !prefixRe.MatchString(input) {
      b.Fatal()
    }
  }
}

func BenchmarkStringsHasPrefix(b *testing.B) {
  for b.Loop() {
    if !strings.HasPrefix(input, "user_") {
      b.Fatal()
    }
  }
}

func BenchmarkRegexpSubmatch(b *testing.B) {
  for b.Loop() {
    m := pkgRe.FindStringSubmatch(input)
    if m == nil {
      b.Fatal()
    }
    _ = m[1]
  }
}

func BenchmarkStringsCut(b *testing.B) {
  for b.Loop() {
    rest, ok := strings.CutPrefix(input, "user_")
    if !ok {
      b.Fatal()
    }
    _ = rest
  }
}
```

`go test -bench . -benchmem`, machine-specific (darwin/arm64, Apple M4 Pro). Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkMustCompileInLoop-12    	  938062	      1248 ns/op	    4306 B/op	      52 allocs/op
BenchmarkPackageLevelVar-12      	22503691	        53.27 ns/op	       0 B/op	       0 allocs/op
BenchmarkRegexpHasPrefix-12      	57460140	        20.66 ns/op	       0 B/op	       0 allocs/op
BenchmarkStringsHasPrefix-12     	769931193	         1.560 ns/op	       0 B/op	       0 allocs/op
BenchmarkRegexpSubmatch-12       	18649660	        63.99 ns/op	      32 B/op	       1 allocs/op
BenchmarkStringsCut-12           	743454312	         1.618 ns/op	       0 B/op	       0 allocs/op
```

**Compile once.** `MustCompile` inside the loop costs **1248 ns and 52 allocations**; the
package-level variable costs **53 ns and zero**. That is a 23x difference, and it is
entirely the compilation. There is no internal pattern cache, so the package-level
`regexp.MustCompile` in a `var` block is not a style preference — it is the only correct
way to use this package in a hot path. The same applies to `regexp.MatchString(pattern, s)`,
which compiles on every call.

**Don't use regexp for what `strings` does.** `^user_` as a regexp is 20.7 ns; the
equivalent `strings.HasPrefix` is **1.56 ns** — 13x faster. Extracting the suffix is 64 ns
with a submatch (plus an allocation) versus 1.6 ns with `strings.CutPrefix` — 39x, and
arguably more readable. A separate run of the same shape measured a literal substring
search at 37.1 ns by regexp versus 3.3 ns by `strings.Contains`, an 11x gap.

| Instead of | Use |
|---|---|
| `regexp.MustCompile("^foo").MatchString(s)` | `strings.HasPrefix(s, "foo")` |
| `regexp.MustCompile("foo$").MatchString(s)` | `strings.HasSuffix(s, "foo")` |
| `regexp.MustCompile("foo").MatchString(s)` | `strings.Contains(s, "foo")` |
| `regexp.MustCompile("^k=(.*)$")` submatch | `strings.CutPrefix(s, "k=")` |
| `regexp.MustCompile("(.*?)=(.*)")` submatch | `strings.Cut(s, "=")` |
| `(?i)foo` against a whole string | `strings.EqualFold(s, "foo")` |

Reach for `regexp` when the pattern genuinely varies — alternation with quantifiers,
character classes, named extraction from a structured line. For a fixed string, a prefix,
a suffix or a single split, `strings` wins on both speed and clarity.

And do not parse HTML, XML or JSON with it. Those grammars are recursive and a regular
expression provably cannot match balanced delimiters; use `encoding/json` (13.6),
`encoding/xml`, or `golang.org/x/net/html`.

## A realistic example: parsing log lines into a struct

```go
package main

import (
  "fmt"
  "regexp"
  "strconv"
  "time"
)

var accessRe = regexp.MustCompile(
  `^(?P<ip>\S+) \[(?P<ts>[^\]]+)\] "(?P<method>[A-Z]+) (?P<path>\S+) HTTP/(?P<ver>[\d.]+)" ` +
    `(?P<status>\d{3}) (?P<bytes>\d+|-)(?: "(?P<ua>[^"]*)")?$`)

type Entry struct {
  IP     string
  At     time.Time
  Method string
  Path   string
  Status int
  Bytes  int64
  Agent  string
}

func parseLine(line string) (Entry, error) {
  m := accessRe.FindStringSubmatch(line)
  if m == nil {
    return Entry{}, fmt.Errorf("line does not match access-log format: %q", line)
  }
  g := func(name string) string {
    i := accessRe.SubexpIndex(name)
    if i < 0 {
      return ""
    }
    return m[i]
  }

  at, err := time.Parse(time.RFC3339, g("ts"))
  if err != nil {
    return Entry{}, fmt.Errorf("timestamp: %w", err)
  }
  status, err := strconv.Atoi(g("status"))
  if err != nil {
    return Entry{}, fmt.Errorf("status: %w", err)
  }
  var size int64
  if b := g("bytes"); b != "-" {
    size, err = strconv.ParseInt(b, 10, 64)
    if err != nil {
      return Entry{}, fmt.Errorf("bytes: %w", err)
    }
  }
  return Entry{
    IP: g("ip"), At: at, Method: g("method"), Path: g("path"),
    Status: status, Bytes: size, Agent: g("ua"),
  }, nil
}

func main() {
  lines := []string{
    `10.0.0.1 [2026-03-07T15:04:05Z] "GET /api/users HTTP/1.1" 200 4096 "curl/8.0"`,
    `10.0.0.2 [2026-03-07T15:04:06Z] "POST /api/orders HTTP/2.0" 201 -`,
    `bad line`,
    `10.0.0.3 [not-a-time] "GET / HTTP/1.1" 200 1`,
    `10.0.0.4 [2026-03-07T15:04:07Z] "DELETE /x HTTP/1.1" 999 0`,
  }
  for _, l := range lines {
    e, err := parseLine(l)
    if err != nil {
      fmt.Println("  ERROR:", err)
      continue
    }
    fmt.Printf("  %-9s %s %-6s %-14s %d %6d agent=%q\n",
      e.IP, e.At.Format(time.TimeOnly), e.Method, e.Path, e.Status, e.Bytes, e.Agent)
  }
}
```

Output:

```
  10.0.0.1  15:04:05 GET    /api/users     200   4096 agent="curl/8.0"
  10.0.0.2  15:04:06 POST   /api/orders    201      0 agent=""
  ERROR: line does not match access-log format: "bad line"
  ERROR: timestamp: parsing time "not-a-time" as "2006-01-02T15:04:05Z07:00": cannot parse "not-a-time" as "2006"
  10.0.0.4  15:04:07 DELETE /x             999      0 agent=""
```

The shape worth copying. The regexp handles only *structure* — splitting the line into
fields — and every field is then validated by the right parser: `time.Parse` (13.4),
`strconv.Atoi` (6.6). Trying to validate a timestamp inside the pattern produces an
unreadable regexp that still cannot reject February 30th.

`(?: "(?P<ua>[^"]*)")?` is an optional non-capturing group wrapping a named one, which is
how you make a trailing field optional. The missing user-agent on line 2 yields `""` via
the group, not an error.

Note the last line: status `999` parses happily. The pattern constrains the *shape*
(`\d{3}`), not the *domain*. Range checks belong in Go, not in `[1-5]\d\d`.

The `g(name)` closure is the small ergonomic win: it turns `m[accessRe.SubexpIndex("ip")]`
into `g("ip")` and survives someone reordering the groups.

## Quick Reference

| Need | Call |
|---|---|
| Compile a literal pattern | `var re = regexp.MustCompile(...)` at package level |
| Compile a dynamic pattern | `regexp.Compile(p)`, handle the error |
| Does it match? | `re.MatchString(s)` |
| The first match | `re.FindString(s)` |
| The first match's groups | `re.FindStringSubmatch(s)` — check for nil |
| All matches (with groups) | `re.FindAllString(s, -1)` / `re.FindAllStringSubmatch(s, -1)` |
| Named groups | `re.SubexpNames()` + `re.SubexpIndex(name)` |
| Replace with groups | `re.ReplaceAllString(s, "${name}")` |
| Replace literally | `re.ReplaceAllLiteralString(s, repl)` |
| Replace with a function | `re.ReplaceAllStringFunc(s, f)` |
| Split | `re.Split(s, -1)` |
| Escape user input | `regexp.QuoteMeta(s)` |
| POSIX leftmost-longest | `re.Longest()` right after compiling |
| Unicode letters/digits | `\p{L}`, `\p{Nd}` — **not** `\w`, `\d` |

| Trap | Reality |
|---|---|
| `\1`, `(?=`, `(?<=` | not supported and never will be — RE2 |
| `\w`, `\d`, `[[:alpha:]]` on non-ASCII | ASCII only; `café` matches as `caf` |
| `FindString(s) != ""` as a match test | an empty match is also `""`; use `MatchString` |
| `"$1x"` in a replacement | that is group `1x`; write `"${1}x"` |
| `$` in a user-supplied replacement | expands; use `ReplaceAllLiteralString` |
| `MustCompile` inside a function | 24x slower; hoist to a package `var` |
| `a\|ab` matching the longer option | leftmost-**first**; call `Longest()` |
| `m[re.SubexpIndex("typo")]` | index `-1`, panic |
| Regexp indexes as character positions | they are byte offsets |
| Parsing HTML/JSON with regexp | the grammars are not regular |

## Common Mistakes

**Calling `regexp.MustCompile` inside a function that runs often.** Compilation is 24x the
cost of the match; declare it as a package-level `var`.

**Using `\w` or `\d` on text that might not be ASCII.** They are ASCII-only, so accented
letters silently split your matches; use `\p{L}` and `\p{Nd}`.

**Writing `"$1suffix"` in a replacement.** It parses as the group named `1suffix` and
expands to nothing; brace it as `"${1}suffix"`.

**Interpolating user input into a pattern.** Either their metacharacters change the
meaning or the pattern fails to compile; wrap it in `regexp.QuoteMeta`.

**Using a regexp where `strings.HasPrefix` or `strings.Cut` would do.** It is 13-37x
slower and harder to read.

**Pasting a PCRE pattern with a lookahead.** RE2 cannot express it; split the check into
two matches or do it in Go.

## Exercises

1. Write `func validate(pattern string) error` that compiles a user-supplied pattern and
   returns a message naming the unsupported construct when it detects a backreference or
   lookahead, rather than passing through `invalid escape sequence`.
2. Implement `func camelToSnake(s string) string` with one regexp and
   `ReplaceAllStringFunc`, correct for `HTTPServer` → `http_server` and `userID` →
   `user_id`. Then write a `strings`-only version and benchmark both.
3. Build a template expander: given `"Hello ${name}, you are ${age}"` and a
   `map[string]string`, substitute with `ReplaceAllStringFunc` and leave unknown keys
   untouched. Handle a literal `$$`.
4. Extend `parseLine` to a streaming parser over `bufio.Scanner` (13.3) that returns
   `([]Entry, []error)`, and benchmark it against a hand-written `strings.Cut` parser on
   100,000 lines. Report the ratio.
5. Demonstrate that RE2 cannot catastrophically backtrack: time `(a+)+$` against 50,000
   `a`s in Go, and explain in one paragraph what the same pattern does in a backtracking
   engine.

---

[← 13.7 encoding/json: Streaming, Tags, and Custom Marshalers](07-json-advanced.md) · [Course Index](../README.md) · [13.9 math, math/big, and math/rand →](09-math-rand.md)
