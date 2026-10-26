[← 14.6 Fuzzing](06-fuzzing.md) · [Course Index](../README.md) · [14.8 Test Doubles with Interfaces →](08-test-doubles.md)

# 14.7 — Examples as Tests and Coverage

## What You'll Learn

- `Example`, `ExampleT`, `ExampleT_method`, `ExampleT_suffix`: the naming rules that attach an example to a symbol
- `// Output:` turns an example into a real test; without it the example is compiled but never run
- `-cover`, `-coverprofile`, `go tool cover -func`/`-html`, the three covermodes, and `-coverpkg`
- Why a 100%-covered function can still be wrong — demonstrated, not asserted

## Coming From Another Language

| Other | Go |
|---|---|
| Python doctests | the same idea, but compiled — a stale example breaks the build |
| JSDoc `@example` (not executed) | Go examples are tests; they run in CI |
| Rust doc-tests in `///` comments | Go puts them in `_test.go` files as real functions |
| `coverage.py` / `nyc` / `istanbul` | `go test -cover`, built in, no config file |
| branch coverage by default | Go measures **statement** coverage only |

The thing worth internalizing: a Go example is simultaneously documentation, a compiled unit, and a
test. Documentation that cannot drift is a genuinely different proposition from documentation that
can.

## Example naming rules

Examples live in `_test.go` files, take no arguments, and return nothing. The **name** determines
which symbol the documentation attaches them to.

| Name | Documents |
|---|---|
| `Example()` | the package as a whole |
| `ExampleNew()` | the function `New` |
| `ExampleStack()` | the type `Stack` |
| `ExampleStack_Pop()` | the method `Pop` on `Stack` |
| `ExampleStack_Push_many()` | a second example for `Stack.Push`, suffixed `many` |
| `Example_unordered()` | a second package-level example, suffixed `unordered` |

The grammar is `Example[Type][_Method][_suffix]`. The suffix **must start with a lowercase letter**
— that is how the tool distinguishes `ExampleStack_Pop` (the `Pop` method) from
`ExampleStack_many` (a variant of the `Stack` example). Get the case wrong and `go vet` complains
that the identifier does not exist.

The package under test:

```go
// Package stack implements a last-in-first-out stack of ints.
package stack

import "errors"

// ErrEmpty is returned by Pop and Peek on an empty Stack.
var ErrEmpty = errors.New("stack: empty")

// Stack is a LIFO collection of ints. The zero value is ready to use.
type Stack struct {
  items []int
}

// New returns an empty Stack with room for n items.
func New(n int) *Stack {
  return &Stack{items: make([]int, 0, n)}
}

// Push adds v to the top of the stack.
func (s *Stack) Push(v int) {
  s.items = append(s.items, v)
}

// Pop removes and returns the top item, or ErrEmpty if there is none.
func (s *Stack) Pop() (int, error) {
  if len(s.items) == 0 {
    return 0, ErrEmpty
  }
  v := s.items[len(s.items)-1]
  s.items = s.items[:len(s.items)-1]
  return v, nil
}

// Peek returns the top item without removing it.
func (s *Stack) Peek() (int, error) {
  if len(s.items) == 0 {
    return 0, ErrEmpty
  }
  return s.items[len(s.items)-1], nil
}

// Len returns the number of items on the stack.
func (s *Stack) Len() int { return len(s.items) }

// Sum returns the total of every item on the stack.
func (s *Stack) Sum() int {
  t := 0
  for _, v := range s.items {
    t += v
  }
  return t
}
```

Every naming form, in an **external** test package so the examples read exactly as a user would
write them:

```go
package stack_test

import (
  "fmt"

  "example.com/stack"
)

func Example() {
  s := stack.New(4)
  s.Push(1)
  s.Push(2)
  v, _ := s.Pop()
  fmt.Println(v, s.Len())
  // Output: 2 1
}

func ExampleNew() {
  s := stack.New(8)
  fmt.Println(s.Len())
  // Output: 0
}

func ExampleStack() {
  var s stack.Stack
  s.Push(10)
  s.Push(20)
  fmt.Println(s.Sum())
  // Output: 30
}

func ExampleStack_Pop() {
  var s stack.Stack
  s.Push(42)
  v, err := s.Pop()
  fmt.Println(v, err)
  _, err = s.Pop()
  fmt.Println(err)
  // Output:
  // 42 <nil>
  // stack: empty
}

func ExampleStack_Push_many() {
  var s stack.Stack
  for i := range 5 {
    s.Push(i * i)
  }
  fmt.Println(s.Len(), s.Sum())
  // Output: 5 30
}

func ExampleStack_Peek_empty() {
  var s stack.Stack
  _, err := s.Peek()
  fmt.Println(err)
  // Output: stack: empty
}

func ExampleStack_Len() {
  s := stack.New(2)
  s.Push(7)
  fmt.Println(s.Len())
  // Output: 1
}

func ExampleStack_Sum_noOutputComment() {
  var s stack.Stack
  s.Push(1)
  fmt.Println("this is never printed or compared")
}
```

`go test -count=1 -v .`. Output:

```
=== RUN   Example
--- PASS: Example (0.00s)
=== RUN   ExampleNew
--- PASS: ExampleNew (0.00s)
=== RUN   ExampleStack
--- PASS: ExampleStack (0.00s)
=== RUN   ExampleStack_Pop
--- PASS: ExampleStack_Pop (0.00s)
=== RUN   ExampleStack_Push_many
--- PASS: ExampleStack_Push_many (0.00s)
=== RUN   ExampleStack_Peek_empty
--- PASS: ExampleStack_Peek_empty (0.00s)
=== RUN   ExampleStack_Len
--- PASS: ExampleStack_Len (0.00s)
PASS
ok  	example.com/stack	0.492s
```

Seven examples ran. Count the functions: there were **eight**.
`ExampleStack_Sum_noOutputComment` is absent — the next section is why.

Two syntax notes from `ExampleStack_Pop`. For multi-line expectations, put `// Output:` on its own
line and each expected line after it. And leading/trailing whitespace on each line is trimmed before
comparison, so indentation in the comment does not matter but *internal* spacing does.

## No Output comment: compiled, not run

An example without an `// Output:` comment is still **compiled and type-checked** — it just never
executes and nothing is compared. That is why `ExampleStack_Sum_noOutputComment` never appeared
above, and why its `fmt.Println` produced nothing.

This is a feature, not an oversight. Use it for an example whose output is not reproducible:

```go
func ExampleServe() {
  srv := NewServer(":8080")
  log.Fatal(srv.ListenAndServe())
}
```

You get the compile-time guarantee — rename `ListenAndServe` and the build breaks — without a test
that would block forever. The same applies to examples printing timestamps, random values, or
addresses.

The trap: **forgetting the comment by accident.** The example then silently never runs, and a
reader seeing `-v` output cannot tell the difference between "no output comment" and "not matched
by `-run`". If an example should be executing, check that it appears in `go test -v`.

## A wrong Output comment

The comparison is exact (after per-line trimming), and the failure format is distinctive.

```go
func ExampleStack_Sum() {
  var s stack.Stack
  s.Push(2)
  s.Push(3)
  fmt.Println(s.Sum())
  // Output: 6
}
```

This example fails on purpose — 2+3 is 5. `go test -count=1 -v -run 'ExampleStack_Sum$' .`.
Output:

```
=== RUN   ExampleStack_Sum
--- FAIL: ExampleStack_Sum (0.00s)
got:
5
want:
6
FAIL
FAIL	example.com/stack	0.508s
FAIL
```

Note the format: a bare `got:`/`want:` block with no `file:line` prefix, because the testing
package is comparing captured stdout rather than reporting an assertion. That is how you recognize
an example failure in a long CI log.

This is the mechanism that keeps documentation honest. Change `Sum` to return a float, or change
`ErrEmpty`'s message, and the examples fail — not a stale comment somebody notices a year later.

## Unordered output

Map iteration order is randomized, so `// Output:` cannot work for it. `// Unordered output:`
compares the lines as a multiset.

```go
func Example_unordered() {
  counts := map[string]int{"a": 1, "b": 2, "c": 3}
  for k, v := range counts {
    fmt.Println(k, v)
  }
  // Unordered output:
  // c 3
  // a 1
  // b 2
}
```

Note the expected lines are deliberately written out of order to prove the point. Three consecutive
runs of `go test -count=1 -run 'Example_unordered' .`:

```
ok  	example.com/stack	0.186s
ok  	example.com/stack	0.181s
ok  	example.com/stack	0.184s
```

Stable regardless of iteration order. It still checks the exact *set* of lines, including
duplicates — a missing or extra line fails.

**Idiomatic:** prefer making the output deterministic (collect keys, `slices.Sort`, then print) so
the example doubles as documentation of a stable format. Use `Unordered output:` when the lack of
order is the honest truth about the API.

## Examples as documentation

Examples in an **external test package** (`package stack_test`) are the ones worth writing, for
two reasons: they are forced to use only the exported API, and they show the real import path, so
a reader can copy the body into their own code.

`go doc` surfaces them with the `-ex` flag. `go doc -ex . Stack.Pop`. Output:

```
package stack // import "example.com/stack"

func (s *Stack) Pop() (int, error)
    Pop removes and returns the top item, or ErrEmpty if there is none.

    func ExampleStack_Pop()
```

The example is listed under the method it documents, which is the attachment working. `go doc -ex .`
shows the package-level view:

```
package stack // import "example.com/stack"

Package stack implements a last-in-first-out stack of ints.

var ErrEmpty = errors.New("stack: empty")
type Stack struct{ ... }
    func New(n int) *Stack
    func ExampleNew()
    func ExampleStack()
    func Example()
    func Example_unordered()
```

The terminal `go doc` lists the example *signatures* rather than rendering their bodies. The full
source with its output is rendered by pkgsite — `go doc -http=:6060` serves that locally, and it is
what you see on pkg.go.dev. Not opened here, since it starts a server.

### Whole-file examples

If a `_test.go` file contains exactly **one** example, no test functions, and at most one other
declaration, pkgsite renders the *entire file* — imports and all — as the example. That is how you
show a complete, copy-pasteable program.

```go
package stack_test

import (
  "fmt"

  "example.com/stack"
)

func Example() {
  s := stack.New(3)
  for _, v := range []int{3, 1, 4} {
    s.Push(v)
  }
  for s.Len() > 0 {
    v, _ := s.Pop()
    fmt.Print(v, " ")
  }
  fmt.Println()
  // Output: 4 1 3
}
```

```
=== RUN   Example
--- PASS: Example (0.00s)
PASS
ok  	example.com/stack	0.492s
```

Note `fmt.Print(v, " ")` adds a space between operands only when neither is a string, so the output
is `4 1 3 ` with a trailing space — which the per-line trim forgives. Convention: name the file
`example_test.go`.

## Coverage: the percentage line

`-cover` adds one line per package.

```bash
go test -count=1 -cover .
```

Output:

```
ok  	example.com/stack	0.498s	coverage: 93.3% of statements
```

Read "of statements" literally. Go measures **statement coverage**: did this statement execute at
least once. Not branch coverage, not condition coverage, not path coverage. An `if` with no `else`
contributes its body's statements, so skipping the `else`-less path costs you nothing.

A detail worth noticing: that 93.3% was produced by the **examples alone**, before any `TestXxx`
function existed in the package. Examples with `// Output:` are real tests and they count toward
coverage, which is a reasonable argument for writing them.

## -coverprofile and go tool cover

The percentage alone is nearly useless. The profile tells you *where*.

```bash
go test -count=1 -coverprofile=cover.out .
go tool cover -func=cover.out
```

Output:

```
example.com/stack/stack.go:15:	New		100.0%
example.com/stack/stack.go:20:	Push		100.0%
example.com/stack/stack.go:25:	Pop		100.0%
example.com/stack/stack.go:35:	Peek		66.7%
example.com/stack/stack.go:43:	Len		100.0%
example.com/stack/stack.go:46:	Sum		100.0%
total:				(statements)	93.3%
```

That is the actionable view: `Peek` is at 66.7%, every other function is complete. One
`ExampleStack_Peek_empty` covers the error path but nothing covers the success path. Chasing the
total percentage would tell you to write "more tests"; this tells you to write *one* test.

The profile itself is a simple text format:

```
mode: set
example.com/stack/stack.go:16.2,17.1 1 1
example.com/stack/stack.go:21.2,22.1 1 1
example.com/stack/stack.go:26.2,26.23 1 1
```

Each line is `file:startLine.startCol,endLine.endCol numStatements count`. Tools consume this
directly; `gocov`, Codecov and SonarQube all read it.

The HTML view annotates your source, green for covered and red for not:

```bash
go tool cover -html=cover.out
```

That **opens a browser**, so it was not run here. With `-o` it writes the file instead, which is
what CI does:

```bash
go tool cover -html=cover.out -o coverage.html
```

The generated file is self-contained HTML with the source inlined, a per-file dropdown and a
covered/uncovered legend. It is the fastest way to find the one branch you forgot.

## covermode: set, count, atomic

| Mode | Records | Cost | When |
|---|---|---|---|
| `set` | bool: did this statement run | lowest | the default |
| `count` | int: how many times | low | finding hot statements |
| `atomic` | int, race-safe | significant | required with `-race` or concurrent tests |

`-covermode=count` changes the profile's counts from 0/1 to real tallies:

```
mode: count
example.com/stack/stack.go:16.2,17.1 1 3
```

That statement ran 3 times. Useful for spotting a loop body that executes once in the whole suite.

**`atomic` is required when tests exercise the instrumented code from multiple goroutines**,
because `set` and `count` increment a plain counter and would themselves be a data race. You rarely
set it by hand: `-race` switches to it automatically. Verified —
`go test -count=1 -race -coverprofile=c3.out .` produced:

```
mode: atomic
```

So `go test -race -cover ./...` is already correct, and the only time to pass `-covermode=atomic`
explicitly is when you have concurrent tests but are not running `-race`.

## -coverpkg and the integration-test surprise

By default each package's coverage is measured **only for itself**. This catches people out
constantly. Two packages, where `api` calls `calc`, and the only test is in `api`:

```go
package calc

func Total(prices []int) int {
  t := 0
  for _, p := range prices {
    t += p
  }
  return t
}

func Average(prices []int) int {
  if len(prices) == 0 {
    return 0
  }
  return Total(prices) / len(prices)
}
```

```go
package api

import "example.com/app/calc"

func Summary(prices []int) (int, int) {
  return calc.Total(prices), calc.Average(prices)
}
```

```go
package api

import "testing"

func TestSummary(t *testing.T) {
  total, avg := Summary([]int{10, 20, 30})
  if total != 60 {
    t.Errorf("total = %d, want 60", total)
  }
  if avg != 20 {
    t.Errorf("avg = %d, want 20", avg)
  }
}
```

`go test -count=1 -cover ./...`. Output:

```
ok  	example.com/app/api	0.491s	coverage: 100.0% of statements
	example.com/app/calc		coverage: 0.0% of statements
```

**`calc` reports 0.0%** even though the passing test just executed `Total` and `Average`. The test
genuinely covered that code; the instrumentation was simply not applied to `calc`, because `calc`
has no test files of its own. This is exactly why end-to-end and integration test suites appear to
cover nothing.

`-coverpkg` fixes it by instrumenting the named packages in every test binary:

```bash
go test -count=1 -coverpkg=./... -coverprofile=all.out ./...
```

Output:

```
ok  	example.com/app/api	0.507s	coverage: 87.5% of statements in ./...
	example.com/app/calc		coverage: 0.0% of statements
```

Note the changed wording — `87.5% of statements in ./...` — and that `calc`'s own line still says
0.0%, because `calc` still ran no tests itself. The real picture is in the profile:

```
example.com/app/api/api.go:5:		Summary		100.0%
example.com/app/calc/calc.go:3:		Total		100.0%
example.com/app/calc/calc.go:11:	Average		66.7%
total:					(statements)	87.5%
```

`calc.Total` is 100% covered and `Average` is 66.7% (the empty-slice branch was never taken) —
credited to the `api` test that actually exercised them. That is the number to report.

The cost: `-coverpkg=./...` instruments every package into every test binary, so builds are slower
and the counts include packages a given test never touches, which *lowers* per-package percentages.
A common compromise is `-coverpkg=./internal/...` to cover your own code but not generated or
vendored packages.

## What coverage does not tell you

Here is a function with **100% statement coverage** and two real bugs.

```go
package disc

func Apply(price, percent int) int {
  return price - price*percent/100
}
```

```go
package disc

import "testing"

func TestApply(t *testing.T) {
  if got := Apply(100, 10); got != 90 {
    t.Errorf("Apply(100, 10) = %d, want 90", got)
  }
}
```

`go test -count=1 -cover .` and the per-function view:

```
ok  	example.com/scratch	0.500s	coverage: 100.0% of statements
```

```
example.com/scratch/disc.go:3:	Apply		100.0%
total:				(statements)	100.0%
```

Perfect coverage. Now ask what happens at the edges:

```go
func TestApplyEdgeCases(t *testing.T) {
  tests := []struct {
    price, percent, want int
  }{
    {100, 10, 90},
    {100, 150, 0},
    {100, -50, 100},
    {0, 10, 0},
  }
  for _, tt := range tests {
    if got := Apply(tt.price, tt.percent); got != tt.want {
      t.Errorf("Apply(%d, %d) = %d, want %d", tt.price, tt.percent, got, tt.want)
    }
  }
}
```

Fails on purpose. Output:

```
=== RUN   TestApplyEdgeCases
    extra_test.go:16: Apply(100, 150) = -50, want 0
    extra_test.go:16: Apply(100, -50) = 150, want 100
--- FAIL: TestApplyEdgeCases (0.00s)
FAIL
FAIL	example.com/scratch	0.487s
FAIL
```

A 150% discount returns a **negative price** and a negative discount **raises** the price. The
function was 100% covered the whole time, because it is a single statement and one test executed
it. Coverage measured that the line ran. It cannot measure that the line is right.

Concretely, statement coverage is blind to: input values (one call covers the statement), missing
`else` branches, unvalidated boundaries, integer overflow, error paths you covered but did not
*assert* on, and any requirement you never wrote a test for at all.

### Targets, briefly

Useful as a *floor* to catch untested files and as a *trend* that should not fall. The 100% target
is an anti-pattern: the last 10% is error paths that need elaborate doubles, and chasing it produces
tests written to execute lines rather than to check behaviour — which is worse than the gap, because
it looks like safety. Spend the effort on the error paths that matter ([14.8](08-test-doubles.md))
and on fuzzing the parsers ([14.6](06-fuzzing.md)) instead.

## -json and test2json

`go test -json` emits the same information as `-v` as a stream of JSON events, one per line —
`Action` is `start`, `run`, `output`, `pass`, `fail`, `skip` or `attr`:

```
{"Time":"2026-10-26T13:46:39.523484+05:30","Action":"run","Package":"example.com/stack","Test":"TestPopEmpty"}
{"Time":"2026-10-26T13:46:39.523654+05:30","Action":"pass","Package":"example.com/stack","Test":"TestPopEmpty","Elapsed":0}
```

It is produced by `go tool test2json`, which can also convert the output of an already-built test
binary (`go tool test2json ./pkg.test -test.v`) — the hook every CI test reporter uses.

## Quick Reference

| Example form | Documents |
|---|---|
| `func Example()` | the package |
| `func ExampleF()` | function `F` |
| `func ExampleT()` | type `T` |
| `func ExampleT_M()` | method `M` on `T` |
| `func ExampleT_M_suffix()` | another example for `T.M`; suffix lowercase |
| `// Output:` | makes it a test; exact match, per-line trimmed |
| `// Unordered output:` | lines compared as a multiset |
| no output comment | compiled and type-checked, never run |

| Command | Effect |
|---|---|
| `go test -cover ./...` | the percentage line per package |
| `go test -coverprofile=c.out ./...` | write a profile |
| `go tool cover -func=c.out` | per-function table |
| `go tool cover -html=c.out` | annotated source in a browser |
| `go tool cover -html=c.out -o c.html` | ... to a file instead |
| `go test -covermode=count` | execution counts rather than booleans |
| `go test -race -cover` | implies `-covermode=atomic` |
| `go test -coverpkg=./... ./...` | credit coverage across packages |
| `go doc -ex .` | list examples alongside the docs |
| `go doc -http=:6060` | pkgsite locally, with rendered examples |
| `go test -json ./...` | machine-readable events |

## Common Mistakes

**Forgetting `// Output:`.** The example compiles and is silently never run; check `go test -v`
lists it.

**Capitalizing an example's suffix.** `ExampleStack_Many` is read as a method named `Many`, and
`go vet` reports that it does not exist.

**Using `// Output:` with map iteration.** It passes locally and fails randomly in CI — use
`// Unordered output:` or sort before printing.

**Reading `coverage: 0.0%` as "untested".** A package whose code is exercised only by another
package's tests reports 0.0% without `-coverpkg`.

**Treating statement coverage as branch coverage.** Go measures only whether a statement ran; the
100%-covered `Apply` above had two bugs.

**Chasing 100%.** The last few percent are error paths that cost the most to reach and tests
written to touch lines rather than check behaviour.

**Running concurrent tests with `-covermode=set`.** The counters themselves race; use `atomic`, or
just run `-race`, which selects it.

## Exercises

1. Write examples for every naming form against a small type of your own, including one with no
   output comment. Confirm from `go test -v` exactly which ones ran.
2. Make an example's `// Output:` wrong by one character and capture the failure. Then change the
   code so the example is correct again and confirm nothing else had to change.
3. Write an example that prints a map. Make it fail with `// Output:` across repeated runs, fix it
   with `// Unordered output:`, then fix it a second way by sorting, and argue for one.
4. Get a package to 100% with `go tool cover -func`, then write a table test that finds a bug in
   it anyway. Report the bug and what coverage could never have told you.
5. Build a two-package module where only the outer package has tests. Capture the inner package's
   0.0% line, then produce the honest number with `-coverpkg` and explain the difference.
6. Generate `coverage.html` for a package with a partly covered function and identify the exact red
   statement. Write the one test that turns it green.
7. Run the same suite with `-covermode=set` and `-covermode=count` and diff the two profiles.
   Explain what the count column let you conclude that the boolean did not.

---

[← 14.6 Fuzzing](06-fuzzing.md) · [Course Index](../README.md) · [14.8 Test Doubles with Interfaces →](08-test-doubles.md)
