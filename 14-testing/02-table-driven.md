[← 14.1 go test and the Basics](01-go-test.md) · [Course Index](../README.md) · [14.3 Fixtures, Helpers, TestMain, and Cleanup →](03-fixtures-helpers.md)

# 14.2 — Table-Driven Tests and Subtests

## What You'll Learn

- The table-driven pattern: an anonymous struct slice plus `t.Run`, which is *the* Go test idiom
- Subtest names: spaces become underscores, names nest with `/`, and `-run` targets one exactly
- Parallel subtests, and the pre-1.22 loop-variable bug that made them silently test the same case
- Error tables with `errors.Is` rather than string comparison, and panic tables with a `recover` helper

## Coming From Another Language

| Other | Go |
|---|---|
| `it.each([...])` / `test.each` in Jest | a `[]struct{...}` literal and a `for` loop |
| `@pytest.mark.parametrize` | the same loop — no decorator, no framework magic |
| `describe` nesting | `t.Run` nesting; names join with `/` |
| Rust's `#[test]` per case | one `Test` function with N subtests, each independently reportable |
| `test.only` / `it.skip` | `-run 'TestX/case_name'` from the command line, no source edit |

Nothing here is a framework feature. A table-driven test is a slice literal and a `for` loop, which
is exactly why it won: you can add a case without learning any API, and the cases are *data* you can
read as a specification.

## The pattern

A slug generator, and a table for it.

```go
package slug

import "strings"

func Make(s string) string {
  s = strings.ToLower(strings.TrimSpace(s))
  var b strings.Builder
  lastDash := false
  for _, r := range s {
    switch {
    case r >= 'a' && r <= 'z', r >= '0' && r <= '9':
      b.WriteRune(r)
      lastDash = false
    default:
      if !lastDash && b.Len() > 0 {
        b.WriteByte('-')
        lastDash = true
      }
    }
  }
  return strings.TrimSuffix(b.String(), "-")
}
```

```go
package slug

import "testing"

func TestMake(t *testing.T) {
  tests := []struct {
    name string
    in   string
    want string
  }{
    {"simple", "Hello", "hello"},
    {"two words", "Hello World", "hello-world"},
    {"punctuation", "Go: The Basics!", "go-the-basics"},
    {"leading space", "  trim me", "trim-me"},
    {"digits kept", "Go 1.27 Release", "go-1-27-release"},
    {"empty", "", ""},
    {"only punctuation", "!!!", ""},
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      if got := Make(tt.in); got != tt.want {
        t.Errorf("Make(%q) = %q, want %q", tt.in, got, tt.want)
      }
    })
  }
}
```

`go test -count=1 -v -run TestMake .`. Output:

```
=== RUN   TestMake
=== RUN   TestMake/simple
=== RUN   TestMake/two_words
=== RUN   TestMake/punctuation
=== RUN   TestMake/leading_space
=== RUN   TestMake/digits_kept
=== RUN   TestMake/empty
=== RUN   TestMake/only_punctuation
--- PASS: TestMake (0.00s)
    --- PASS: TestMake/simple (0.00s)
    --- PASS: TestMake/two_words (0.00s)
    --- PASS: TestMake/punctuation (0.00s)
    --- PASS: TestMake/leading_space (0.00s)
    --- PASS: TestMake/digits_kept (0.00s)
    --- PASS: TestMake/empty (0.00s)
    --- PASS: TestMake/only_punctuation (0.00s)
PASS
ok  	example.com/scratch	0.496s
```

The conventions in that code, each of which earns its place:

- **An anonymous struct.** The type is used once, so it needs no name. Fields are `name`, the
  inputs, then `want` — in that order, so a case reads left to right as "this case, this input,
  this result".
- **`tt` for the loop variable.** Near-universal in Go code, alongside `tc`. Keep it short; it
  appears on every line of the body.
- **`t.Run(tt.name, ...)`** so each case is its own reportable test with its own name.
- **The body is identical for every case.** The moment you need an `if tt.name == "..."` branch in
  the body, the table has stopped being the right tool.

Without `t.Run`, a failure reports the test function's name and you have to read the message to
work out which row broke — and a `t.Fatal` kills the remaining rows. Subtests fix both.

## Subtest names

`t.Run` does not use your string verbatim. **Spaces become underscores.** The name above was
`"two words"`, and the output says `TestMake/two_words`.

The rules the testing package applies:

| In your table | In the output |
|---|---|
| `two words` | `two_words` |
| `a  b` (two spaces) | `a__b` (two underscores) |
| any non-printable rune | escaped, e.g. `\x00` |
| a duplicate name | suffixed `#01`, `#02`, ... |
| `a/b` | stays `a/b` — and now looks like a nesting level |

Subtest names are path-like because `-run` splits on `/`, which is what lets you target one case:

```bash
go test -count=1 -v -run 'TestMake/two_words' .
```

Output:

```
=== RUN   TestMake
=== RUN   TestMake/two_words
--- PASS: TestMake (0.00s)
    --- PASS: TestMake/two_words (0.00s)
PASS
ok  	example.com/scratch	0.226s
```

The parent still runs — it has to, in order to reach the `t.Run` call that creates the subtest —
but only the matching child executes. Write the underscore, not the space: `-run 'TestMake/two
words'` matches nothing. And avoid `/` inside a case name, or you create a phantom nesting level
that makes `-run` behave strangely.

## Reading a table failure

Add one more row to the table above:

```go
{"unicode", "Café Noir", "cafe-noir"},
```

This row fails on purpose — it exposes a real bug, since the `a`-`z` filter drops `é` instead of
folding it. `go test -count=1 -v -run TestMake .`, tail of the output:

```
=== RUN   TestMake/unicode
    slug_test.go:23: Make("Café Noir") = "caf-noir", want "cafe-noir"
--- FAIL: TestMake (0.00s)
    --- PASS: TestMake/simple (0.00s)
    --- PASS: TestMake/two_words (0.00s)
    --- PASS: TestMake/punctuation (0.00s)
    --- PASS: TestMake/leading_space (0.00s)
    --- PASS: TestMake/digits_kept (0.00s)
    --- PASS: TestMake/empty (0.00s)
    --- PASS: TestMake/only_punctuation (0.00s)
    --- FAIL: TestMake/unicode (0.00s)
FAIL
FAIL	example.com/scratch	0.504s
FAIL
```

Every other row still ran and passed. The parent is marked `FAIL` because a child failed.

Without `-v` the report collapses to only what went wrong — the normal way you will see this:

```
--- FAIL: TestMake (0.00s)
    --- FAIL: TestMake/unicode (0.00s)
        slug_test.go:23: Make("Café Noir") = "caf-noir", want "cafe-noir"
FAIL
FAIL	example.com/scratch	0.194s
FAIL
```

Three lines tell you the test, the case, and the exact values. Then
`go test -run 'TestMake/unicode'` iterates on just that case.

## Nesting

`t.Run` nests arbitrarily, and the names join with `/`.

```go
package nest

import "testing"

func TestNested(t *testing.T) {
  for _, codec := range []string{"json", "xml"} {
    t.Run(codec, func(t *testing.T) {
      for _, size := range []string{"small", "large"} {
        t.Run(size, func(t *testing.T) {
          t.Logf("codec=%s size=%s name=%s", codec, size, t.Name())
        })
      }
    })
  }
}
```

`go test -count=1 -v .`. Output:

```
=== RUN   TestNested
=== RUN   TestNested/json
=== RUN   TestNested/json/small
    nest_test.go:10: codec=json size=small name=TestNested/json/small
=== RUN   TestNested/json/large
    nest_test.go:10: codec=json size=large name=TestNested/json/large
=== RUN   TestNested/xml
=== RUN   TestNested/xml/small
    nest_test.go:10: codec=xml size=small name=TestNested/xml/small
=== RUN   TestNested/xml/large
    nest_test.go:10: codec=xml size=large name=TestNested/xml/large
--- PASS: TestNested (0.00s)
    --- PASS: TestNested/json (0.00s)
        --- PASS: TestNested/json/small (0.00s)
        --- PASS: TestNested/json/large (0.00s)
    --- PASS: TestNested/xml (0.00s)
        --- PASS: TestNested/xml/small (0.00s)
        --- PASS: TestNested/xml/large (0.00s)
PASS
ok  	example.com/scratch	0.505s
```

`t.Name()` returns the full path, and the summary indents four spaces per level. Each `/`-separated
part of `-run` is matched against the corresponding level:

```bash
go test -count=1 -v -run 'TestNested/json/large' .
```

Output:

```
=== RUN   TestNested
=== RUN   TestNested/json
=== RUN   TestNested/json/large
    nest_test.go:10: codec=json size=large name=TestNested/json/large
--- PASS: TestNested (0.00s)
    --- PASS: TestNested/json (0.00s)
        --- PASS: TestNested/json/large (0.00s)
PASS
ok  	example.com/scratch	0.202s
```

Each part is still an unanchored regexp, so `-run 'TestNested//large'` (empty middle part) matches
every codec's `large`. Two levels is usually the practical limit for readability.

## Parallel subtests

Call `t.Parallel()` as the first line of the subtest body. Each subtest pauses; once the parent's
loop finishes, they all run together.

```go
package p

import "testing"

func TestParallelTable(t *testing.T) {
  tests := []struct {
    name string
    in   int
    want int
  }{
    {"one", 1, 2},
    {"two", 2, 4},
    {"three", 3, 6},
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      t.Parallel()
      if got := tt.in * 2; got != tt.want {
        t.Errorf("%s: double(%d) = %d, want %d", tt.name, tt.in, got, tt.want)
      }
      t.Logf("case %q checked in=%d", tt.name, tt.in)
    })
  }
}
```

`go test -count=1 -v .` on Go 1.27 — the completion order varies between runs. Output:

```
=== RUN   TestParallelTable
=== RUN   TestParallelTable/one
=== PAUSE TestParallelTable/one
=== RUN   TestParallelTable/two
=== PAUSE TestParallelTable/two
=== RUN   TestParallelTable/three
=== PAUSE TestParallelTable/three
=== CONT  TestParallelTable/one
=== CONT  TestParallelTable/three
=== NAME  TestParallelTable/one
    p_test.go:21: case "one" checked in=1
=== CONT  TestParallelTable/two
=== NAME  TestParallelTable/three
    p_test.go:21: case "three" checked in=3
=== NAME  TestParallelTable/two
    p_test.go:21: case "two" checked in=2
--- PASS: TestParallelTable (0.00s)
    --- PASS: TestParallelTable/one (0.00s)
    --- PASS: TestParallelTable/three (0.00s)
    --- PASS: TestParallelTable/two (0.00s)
PASS
ok  	example.com/scratch	0.491s
```

Each case logged its *own* input: `one`/1, `two`/2, `three`/3. Hold on to that.

The ordering rule that matters: **the parent's serial part runs to completion before any parallel
child body runs.** So anything the parent does after the loop — including a `defer` that tears down
a fixture — happens *before* the children use it. The fix is `t.Cleanup`, which is deferred until
the subtests finish ([14.3](03-fixtures-helpers.md)).

## The pre-1.22 loop variable

Before Go 1.22, `for _, tt := range tests` created **one** `tt` reused by every iteration. A
parallel subtest closure captured that single variable and did not read it until after the loop had
finished — by which time it held the last row.

The behaviour is selected by the `go` directive in `go.mod`, so the same file demonstrates both.
Change nothing but `go 1.27.1` to `go 1.21` and re-run:

```
module example.com/scratch

go 1.21
```

`go test -count=1 -v .`. Output:

```
=== RUN   TestParallelTable
=== RUN   TestParallelTable/one
=== PAUSE TestParallelTable/one
=== RUN   TestParallelTable/two
=== PAUSE TestParallelTable/two
=== RUN   TestParallelTable/three
=== PAUSE TestParallelTable/three
=== CONT  TestParallelTable/one
=== CONT  TestParallelTable/three
    p_test.go:21: case "three" checked in=3
=== NAME  TestParallelTable/one
    p_test.go:21: case "three" checked in=3
=== CONT  TestParallelTable/two
    p_test.go:21: case "three" checked in=3
--- PASS: TestParallelTable (0.00s)
    --- PASS: TestParallelTable/three (0.00s)
    --- PASS: TestParallelTable/one (0.00s)
    --- PASS: TestParallelTable/two (0.00s)
PASS
ok  	example.com/scratch	0.491s
```

Read it carefully. The subtests are still *named* `one`, `two` and `three` — the name was evaluated
during the loop, when `tt` was still correct. But all three bodies logged `case "three" checked
in=3`. The suite tested one row three times and **passed**, which is the worst possible failure
mode: a green test that checks nothing. Note also that `in=3` with `want 6` is consistent, so no
assertion fired.

Which is why you will see this line throughout pre-1.22 Go code:

```go
for _, tt := range tests {
  tt := tt
  t.Run(tt.name, func(t *testing.T) {
    t.Parallel()
    ...
  })
}
```

`tt := tt` shadows the loop variable with a fresh per-iteration copy. On Go 1.22+ it is a no-op —
harmless, and `go vet` does not flag it, but it is noise. Delete it when you raise the `go`
directive; keep it if the module is pinned below 1.22, because there the bug is real.

**One rule:** if the module's `go` line is 1.22 or later, you need no copy. If it is lower, every
parallel subtest closure needs one. The `go` line in `go.mod` is the switch, not your installed
toolchain version.

## Error tables, done right

Two ways to put errors in a table. Start with the coarse one, `wantErr bool`:

```go
package age

import (
  "errors"
  "fmt"
  "strconv"
)

var (
  ErrEmpty    = errors.New("age: empty input")
  ErrNotInt   = errors.New("age: not an integer")
  ErrOutRange = errors.New("age: out of range")
)

func Parse(s string) (int, error) {
  if s == "" {
    return 0, ErrEmpty
  }
  n, err := strconv.Atoi(s)
  if err != nil {
    return 0, fmt.Errorf("parsing %q: %w", s, ErrNotInt)
  }
  if n < 0 || n > 150 {
    return 0, fmt.Errorf("age %d: %w", n, ErrOutRange)
  }
  return n, nil
}
```

```go
func TestParseWantErr(t *testing.T) {
  tests := []struct {
    name    string
    in      string
    want    int
    wantErr bool
  }{
    {"valid", "42", 42, false},
    {"zero", "0", 0, false},
    {"empty", "", 0, true},
    {"not a number", "abc", 0, true},
    {"negative", "-1", 0, true},
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      got, err := Parse(tt.in)
      if (err != nil) != tt.wantErr {
        t.Fatalf("Parse(%q) error = %v, wantErr %v", tt.in, err, tt.wantErr)
      }
      if got != tt.want {
        t.Errorf("Parse(%q) = %d, want %d", tt.in, got, tt.want)
      }
    })
  }
}
```

`(err != nil) != tt.wantErr` is the standard idiom and reads worse than it is: "did we get an
error, and is that what we expected". It passes, but it only proves *an* error happened. Those three
failing rows would still pass if `Parse` returned the wrong error for every one of them.

The better table stores the sentinel itself and uses `errors.Is`:

```go
func TestParseErrorsIs(t *testing.T) {
  tests := []struct {
    name    string
    in      string
    want    int
    wantErr error
  }{
    {"valid", "42", 42, nil},
    {"empty", "", 0, ErrEmpty},
    {"not a number", "abc", 0, ErrNotInt},
    {"negative", "-1", 0, ErrOutRange},
    {"too large", "200", 0, ErrOutRange},
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      got, err := Parse(tt.in)
      if !errors.Is(err, tt.wantErr) {
        t.Fatalf("Parse(%q) error = %v, want %v", tt.in, err, tt.wantErr)
      }
      if got != tt.want {
        t.Errorf("Parse(%q) = %d, want %d", tt.in, got, tt.want)
      }
    })
  }
}
```

`go test -count=1 -v .`. Output:

```
=== RUN   TestParseErrorsIs
=== RUN   TestParseErrorsIs/valid
=== RUN   TestParseErrorsIs/empty
=== RUN   TestParseErrorsIs/not_a_number
=== RUN   TestParseErrorsIs/negative
=== RUN   TestParseErrorsIs/too_large
--- PASS: TestParseErrorsIs (0.00s)
    --- PASS: TestParseErrorsIs/valid (0.00s)
    --- PASS: TestParseErrorsIs/empty (0.00s)
    --- PASS: TestParseErrorsIs/not_a_number (0.00s)
    --- PASS: TestParseErrorsIs/negative (0.00s)
    --- PASS: TestParseErrorsIs/too_large (0.00s)
PASS
ok  	example.com/scratch	0.492s
```

The `nil` row works for free: `errors.Is(nil, nil)` is `true`, so a single check covers both the
success and the failure cases. That is why one `wantErr error` field beats a `wantErr bool` plus a
separate sentinel field.

### Why not compare the error string

Because wrapping breaks it. Here is the same table matching on `err.Error()`:

```go
func TestParseStringCompare(t *testing.T) {
  tests := []struct {
    name   string
    in     string
    errStr string
  }{
    {"empty", "", "age: empty input"},
    {"not a number", "abc", "age: not an integer"},
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      _, err := Parse(tt.in)
      if err.Error() != tt.errStr {
        t.Errorf("Parse(%q) error = %q, want %q", tt.in, err.Error(), tt.errStr)
      }
    })
  }
}
```

The second row fails on purpose. `go test -count=1 -v -run TestParseStringCompare .`. Output:

```
=== RUN   TestParseStringCompare
=== RUN   TestParseStringCompare/empty
=== RUN   TestParseStringCompare/not_a_number
    strcmp_test.go:18: Parse("abc") error = "parsing \"abc\": age: not an integer", want "age: not an integer"
--- FAIL: TestParseStringCompare (0.00s)
    --- PASS: TestParseStringCompare/empty (0.00s)
    --- FAIL: TestParseStringCompare/not_a_number (0.00s)
FAIL
FAIL	example.com/scratch	0.477s
FAIL
```

The `empty` row passes only because that error is returned unwrapped. The `not a number` row fails
because `fmt.Errorf("parsing %q: %w", ...)` added context — a *good* change to the code that a
string-matching test punishes. `errors.Is` sees through every layer of `%w`.

**Idiomatic:** `wantErr error` plus `errors.Is`. For a custom error type whose fields you need to
inspect, use `errors.As` into a typed variable. Never compare error strings except when the string
*is* the contract, such as a message shown to a user. Cross-reference
[8.3](../08-errors/03-is-as-unwrap.md) and [8.6](../08-errors/06-strategy.md).

## Panic tables

A panic needs a `recover` helper. The helper returns both the message and whether it fired, so one
table covers panicking and non-panicking cases.

```go
package q

type Queue struct {
  items []int
  cap   int
}

func New(capacity int) *Queue {
  if capacity <= 0 {
    panic("q: capacity must be positive")
  }
  return &Queue{cap: capacity}
}

func (q *Queue) Push(v int) {
  if len(q.items) == q.cap {
    panic("q: queue full")
  }
  q.items = append(q.items, v)
}
```

```go
package q

import (
  "fmt"
  "testing"
)

func didPanic(t *testing.T, fn func()) (msg string, panicked bool) {
  t.Helper()
  defer func() {
    if r := recover(); r != nil {
      msg, panicked = fmt.Sprint(r), true
    }
  }()
  fn()
  return "", false
}

func TestNewPanics(t *testing.T) {
  tests := []struct {
    name      string
    capacity  int
    wantPanic string
  }{
    {"positive", 4, ""},
    {"zero", 0, "q: capacity must be positive"},
    {"negative", -3, "q: capacity must be positive"},
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      msg, panicked := didPanic(t, func() { New(tt.capacity) })
      if panicked != (tt.wantPanic != "") {
        t.Fatalf("New(%d) panicked = %v, want %v", tt.capacity, panicked, tt.wantPanic != "")
      }
      if msg != tt.wantPanic {
        t.Errorf("New(%d) panic = %q, want %q", tt.capacity, msg, tt.wantPanic)
      }
    })
  }
}
```

`go test -count=1 -v -run TestNewPanics .`. Output:

```
=== RUN   TestNewPanics
=== RUN   TestNewPanics/positive
=== RUN   TestNewPanics/zero
=== RUN   TestNewPanics/negative
--- PASS: TestNewPanics (0.00s)
    --- PASS: TestNewPanics/positive (0.00s)
    --- PASS: TestNewPanics/zero (0.00s)
    --- PASS: TestNewPanics/negative (0.00s)
PASS
ok  	example.com/scratch	0.491s
```

Two details. The `defer` must be inside the helper, not the subtest, because `recover` only works in
a function deferred from the frame that is panicking ([3.6](../03-control-flow/06-panic-recover.md)).
And `recover()` returns `any`, so `fmt.Sprint` is the simple way to get a comparable string — if the
panic value is a typed error you care about, return the `any` and use `errors.As` on it instead.

## Map-based vs slice-based tables

A `map[string]struct{...}` puts the name in the key, which removes the `name` field:

```go
func TestMapTable(t *testing.T) {
  tests := map[string]struct {
    pushes int
    cap    int
    want   bool
  }{
    "under capacity": {2, 4, false},
    "at capacity":    {4, 4, false},
    "over capacity":  {5, 4, true},
  }
  for name, tt := range tests {
    t.Run(name, func(t *testing.T) {
      queue := New(tt.cap)
      _, panicked := didPanic(t, func() {
        for i := range tt.pushes {
          queue.Push(i)
        }
      })
      if panicked != tt.want {
        t.Errorf("%d pushes into cap %d: panicked = %v, want %v", tt.pushes, tt.cap, panicked, tt.want)
      }
    })
  }
}
```

`go test -count=1 -v -run TestMapTable .`. Output:

```
=== RUN   TestMapTable
=== RUN   TestMapTable/under_capacity
=== RUN   TestMapTable/at_capacity
=== RUN   TestMapTable/over_capacity
--- PASS: TestMapTable (0.00s)
    --- PASS: TestMapTable/under_capacity (0.00s)
    --- PASS: TestMapTable/at_capacity (0.00s)
    --- PASS: TestMapTable/over_capacity (0.00s)
PASS
ok  	example.com/scratch	0.491s
```

Map iteration order is randomized ([7.6](../07-slices-maps/06-maps.md)), so **the execution order
changes between runs**. Six consecutive runs of the above produced, in order of the `=== RUN` lines:

```
under_capacity at_capacity over_capacity
under_capacity at_capacity over_capacity
under_capacity at_capacity over_capacity
under_capacity at_capacity over_capacity
over_capacity under_capacity at_capacity
at_capacity over_capacity under_capacity
```

With only three keys the randomization often lands on the same order, which makes this worse than
fully random: an order dependence can hide for many runs and then surface once.

| | slice table | map table |
|---|---|---|
| Order | fixed, as written | randomized per run |
| Duplicate names | allowed (suffixed `#01`) | impossible — compile error |
| The `name` field | needed | the key is the name |
| Surfaces order dependence | no | yes, eventually |
| Reads as a spec | yes — cases in a deliberate sequence | less so |
| Diffs | clean, one line per case | reordering churn when gofmt aligns keys |

**Idiomatic:** the slice. It is what the standard library uses, it keeps a deliberate progression
(simple cases first, edge cases last), and it allows duplicate names. Reach for the map when you
specifically want order randomization, or use `-shuffle` on the slice version instead
([14.3](03-fixtures-helpers.md)) which randomizes without giving up the ordering you wrote.

## When a table hurts

When the cases need materially different setup or assertions, a table forces you to add
`setupFn`/`checkFn` function fields and optional pointers for "not applicable", at which point you
have built a worse version of three plainly-written test functions — write those instead.

## cmp.Diff in one line

For comparing structs and slices, `github.com/google/go-cmp`'s `cmp.Diff(want, got)` returns a
readable unified diff and `""` when equal, so a table body becomes
`if d := cmp.Diff(tt.want, got); d != "" { t.Errorf("mismatch (-want +got):\n%s", d) }` —
see [14.4](04-testify.md).

## Quick Reference

| Form | Meaning |
|---|---|
| `tests := []struct{ name string; ... }{...}` | the table |
| `for _, tt := range tests` | iterate; `tt` or `tc` by convention |
| `t.Run(tt.name, func(t *testing.T) {...})` | one named subtest per case |
| `t.Parallel()` first in the body | run the cases concurrently |
| `tt := tt` | per-iteration copy; needed only below `go 1.22` |
| `t.Name()` | the full `Parent/child/grandchild` path |
| `(err != nil) != tt.wantErr` | coarse error check |
| `errors.Is(err, tt.wantErr)` | the right error check; `nil` row works too |

| Command | Effect |
|---|---|
| `go test -run TestMake` | the whole table |
| `go test -run 'TestMake/two_words'` | one case (underscores, not spaces) |
| `go test -run 'TestMake/.*space.*'` | cases matching a pattern |
| `go test -run 'TestNested/json/large'` | one nested case |
| `go test -v -run TestMake` | every subtest name and result |

| Name transform | Result |
|---|---|
| space | `_` |
| duplicate | `name#01`, `name#02` |
| non-printable | escaped (`\x00`) |
| `/` | a nesting separator — avoid in case names |

## Common Mistakes

**Adding an `if tt.name == "special"` branch to the subtest body.** The table's premise is one
body for all cases; write that case as its own test function.

**Omitting `tt := tt` in a module pinned below `go 1.22`.** Every parallel subtest then tests the
last row while reporting all the right names, and the suite passes green.

**Leaving `tt := tt` in and thinking it is still required.** From `go 1.22` the loop variable is
per-iteration; the copy is dead weight.

**Comparing `err.Error()` to a string.** Any `%w` wrapping added later breaks the test even though
the behaviour is unchanged — store the sentinel and use `errors.Is`.

**Using `-run 'TestMake/two words'` with a literal space.** `t.Run` replaced it with an underscore,
so the pattern matches nothing and you get a silent `ok` with no tests run.

**Calling `t.Fatal` in a parallel subtest's shared setup.** It stops only that subtest; the others
proceed against half-built state.

**Forgetting that the parent returns before parallel children run.** A `defer cleanup()` in the
parent fires while the children still need the fixture — use `t.Cleanup`
([14.3](03-fixtures-helpers.md)).

## Exercises

1. Convert three separate `TestFoo`, `TestFooEmpty`, `TestFooUnicode` functions into one table with
   `t.Run`, then target the unicode case alone from the command line without editing the file.
2. Write a table where two rows have the same `name`. Run with `-v` and record exactly how the
   testing package disambiguates them, then target the second one with `-run`.
3. Build a table for a `Parse` function with three distinct sentinel errors. Write it twice, once
   with `wantErr bool` and once with `wantErr error` + `errors.Is`, then introduce a bug that
   returns the *wrong* sentinel and record which version catches it.
4. Pin a module to `go 1.21`, write a parallel table test without `tt := tt`, and capture the
   output proving every subtest ran the last row. Then fix it three ways: `tt := tt`, passing `tt`
   as a parameter, and raising the `go` directive.
5. Take a slice table that passes and convert it to a map table. Run it 20 times in a loop and
   report how many distinct execution orders you observed.
6. Write a panic table for a function that panics with a custom error type rather than a string,
   using `errors.As` on the recovered value instead of `fmt.Sprint`.

---

[← 14.1 go test and the Basics](01-go-test.md) · [Course Index](../README.md) · [14.3 Fixtures, Helpers, TestMain, and Cleanup →](03-fixtures-helpers.md)
