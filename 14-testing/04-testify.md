[← 14.3 Fixtures, Helpers, TestMain, and Cleanup](03-fixtures-helpers.md) · [Course Index](../README.md) · [14.5 Benchmarks →](05-benchmarks.md)

# 14.4 — testify, Briefly

## What You'll Learn

- `assert` continues on failure, `require` stops — the one distinction that matters
- The assertions worth knowing, and `assert.Equal`'s strict `reflect.DeepEqual` typing (`1 != int64(1)`)
- The multi-line struct diff, which is the real reason people reach for it
- `suite.Suite` and `mock.Mock` in one example each, and when stdlib + go-cmp is the better call

## Coming From Another Language

| Other | testify |
|---|---|
| `expect(x).toBe(y)` | `assert.Equal(t, want, got)` — note `want` comes **first** |
| `assert x == y` (pytest) | `assert.Equal(t, want, got)` |
| `pytest.raises` | `assert.Panics` / `assert.Error` / `assert.ErrorIs` |
| `unittest.TestCase` + `setUp` | `suite.Suite` + `SetupTest` |
| `unittest.mock` / `jest.mock` | `mock.Mock` — but hand-written fakes are usually better in Go |

The argument order is the trap: testify is `(t, expected, actual)` while the stdlib convention
is `got, want`. Getting it backwards produces a diff with the labels swapped, which costs you
real debugging time.

```bash
go get github.com/stretchr/testify@v1.12.1
```

Captured here with testify v1.12.1 (it pulls in `stretchr/objx`, `davecgh/go-spew`,
`pmezard/go-difflib` and `go.yaml.in/yaml/v3`).

## assert vs require

Both packages expose the same function names. `assert` calls `t.Errorf`; `require` calls
`t.FailNow`.

```go
package u

import (
  "testing"

  "github.com/stretchr/testify/assert"
  "github.com/stretchr/testify/require"
)

func TestAssertContinues(t *testing.T) {
  u, err := Find(1)
  assert.NoError(t, err)
  assert.Equal(t, 2, u.ID)
  assert.Equal(t, "Grace", u.Name)
  t.Log("reached the end of the test")
}

func TestRequireStops(t *testing.T) {
  u, err := Find(1)
  require.NoError(t, err)
  require.Equal(t, 2, u.ID)
  require.Equal(t, "Grace", u.Name)
  t.Log("never reached")
}
```

Both tests fail on purpose (the real `Find(1)` returns ID 1, name "Ada"). `go test -count=1 -v .`
— absolute paths in the `Error Trace` are shortened to `/tmp/scratch` here. Output:

```
=== RUN   TestAssertContinues
    ar_test.go:13: 
        	Error Trace:	/tmp/scratch/ar_test.go:13
        	Error:      	Not equal: 
        	            	expected: 2
        	            	actual  : 1
        	Test:       	TestAssertContinues
    ar_test.go:14: 
        	Error Trace:	/tmp/scratch/ar_test.go:14
        	Error:      	Not equal: 
        	            	expected: "Grace"
        	            	actual  : "Ada"
        	            	
        	            	Diff:
        	            	--- Expected
        	            	+++ Actual
        	            	@@ -1 +1 @@
        	            	-Grace
        	            	+Ada
        	Test:       	TestAssertContinues
    ar_test.go:15: reached the end of the test
--- FAIL: TestAssertContinues (0.00s)
=== RUN   TestRequireStops
    ar_test.go:21: 
        	Error Trace:	/tmp/scratch/ar_test.go:21
        	Error:      	Not equal: 
        	            	expected: 2
        	            	actual  : 1
        	Test:       	TestRequireStops
--- FAIL: TestRequireStops (0.00s)
FAIL
FAIL	example.com/scratch	0.902s
FAIL
```

`assert` reported both mismatches and ran to the end. `require` reported the first and stopped —
the `Name` check and the final log never happened.

**Idiomatic:** `require` for preconditions (an error that would make the next line panic),
`assert` for independent checks you want to see all of. `require.NoError(t, err)` after any call
returning an error is the most common testify line in real code, because `assert.NoError` would
let a `nil` pointer reach the next line and panic. `require` calls `FailNow`, so it carries the
same goroutine restriction as `t.Fatal` ([14.1](01-go-test.md)).

## The assertions worth knowing

Every one of these also exists on `require`, and each takes optional trailing
`msgAndArgs ...any`.

| Assertion | Passes when |
|---|---|
| `Equal(t, want, got)` | `ObjectsAreEqual` — bytes compared directly, else `reflect.DeepEqual` |
| `NotEqual(t, x, got)` | the above is false |
| `EqualValues(t, want, got)` | equal after type conversion — `1 == int64(1)` |
| `Nil(t, v)` / `NotNil(t, v)` | `v` is nil (handles typed nils, unlike `== nil`) |
| `True(t, b)` / `False(t, b)` | the bool |
| `NoError(t, err)` | `err == nil`, and prints the error if not |
| `Error(t, err)` | `err != nil` |
| `ErrorIs(t, err, target)` | `errors.Is` |
| `ErrorAs(t, err, &target)` | `errors.As` |
| `ErrorContains(t, err, s)` | `err.Error()` contains `s` |
| `Len(t, v, n)` | `len(v) == n`, for any type with a length |
| `Empty(t, v)` / `NotEmpty(t, v)` | zero value, empty collection, or nil |
| `Contains(t, haystack, needle)` | substring, slice element, or map key |
| `ElementsMatch(t, a, b)` | same elements, **any order** |
| `InDelta(t, want, got, delta)` | floats within `delta` |
| `Panics(t, f)` / `NotPanics(t, f)` | `f` panics or does not |
| `Eventually(t, cond, wait, tick)` | `cond` returns true before `wait` elapses |

All of them in one passing test — error assertions, collection assertions, `InDelta`,
`Panics` and `Eventually` together:

```go
func TestCatalogue(t *testing.T) {
  user, err := Find(1)
  assert.NoError(t, err)
  assert.Equal(t, 1, user.ID)
  assert.Len(t, user.Tags, 2)
  assert.Contains(t, user.Tags, "admin")
  assert.ElementsMatch(t, []string{"early", "admin"}, user.Tags)
  assert.NotEmpty(t, user.Name)

  _, err = Find(99)
  assert.ErrorIs(t, err, ErrNotFound)
  assert.ErrorContains(t, err, "find 99")

  var missing *MissingError
  _, err2 := FindTyped(99)
  assert.ErrorAs(t, err2, &missing)
  assert.Equal(t, 99, missing.ID)

  assert.Nil(t, nilUser())
  assert.InDelta(t, 0.3, 0.1+0.2, 1e-9)
  assert.Panics(t, func() { panic("boom") })

  done := make(chan struct{})
  go func() { time.Sleep(20 * time.Millisecond); close(done) }()
  assert.Eventually(t, func() bool {
    select {
    case <-done:
      return true
    default:
      return false
    }
  }, time.Second, 5*time.Millisecond)
}
```

`go test -count=1 -v -run TestCatalogue .`. Output:

```
=== RUN   TestCatalogue
--- PASS: TestCatalogue (0.02s)
PASS
ok  	example.com/scratch	0.662s
```

`ElementsMatch` matched `["early","admin"]` against `["admin","early"]` — order-insensitive, where
`Equal` would reject it. `assert.Nil` passing on a typed `(*User)(nil)` matters too: a plain
`v == nil` on an interface holding a typed nil is false
([5.2](../05-interfaces/02-interface-values-nil.md)).

## The int-vs-int64 surprise

`assert.Equal` compares with `reflect.DeepEqual` after a `[]byte` fast path, and `DeepEqual`
requires **identical types**.

```go
func TestIntVsInt64(t *testing.T) {
  assert.Equal(t, 1, int64(1))
}
```

This fails on purpose. Output:

```
=== RUN   TestIntVsInt64
    diff_test.go:10: 
        	Error Trace:	/tmp/scratch/diff_test.go:10
        	Error:      	Not equal: 
        	            	expected: int(1)
        	            	actual  : int64(1)
        	Test:       	TestIntVsInt64
--- FAIL: TestIntVsInt64 (0.00s)
```

`expected: int(1)` / `actual: int64(1)` — equal values, failing test. You hit this constantly
against anything returning `int64` (`time.Duration`, `sql` counts, `strconv.ParseInt`) or
`float64` (decoded JSON numbers). `assert.EqualValues` converts first and passes:

```go
func TestEqualValues(t *testing.T) {
  assert.EqualValues(t, 1, int64(1))
}
```

```
=== RUN   TestEqualValues
--- PASS: TestEqualValues (0.00s)
```

Use `EqualValues` sparingly: it also makes genuine type bugs invisible. Converting explicitly in
the test (`assert.Equal(t, int64(1), got)`) documents the type and is usually better.

## The struct diff

This is the feature that sells testify. A struct with a wrong field and a wrong slice element:

```go
func TestStructDiff(t *testing.T) {
  got, err := Find(1)
  assert.NoError(t, err)
  want := &User{ID: 1, Name: "Ada", Email: "ada@example.org", Tags: []string{"admin", "founder"}}
  assert.Equal(t, want, got)
}
```

Fails on purpose. Output:

```
=== RUN   TestStructDiff
    diff_test.go:17: 
        	Error Trace:	/tmp/scratch/diff_test.go:17
        	Error:      	Not equal: 
        	            	expected: &u.User{ID:1, Name:"Ada", Email:"ada@example.org", Tags:[]string{"admin", "founder"}}
        	            	actual  : &u.User{ID:1, Name:"Ada", Email:"ada@example.com", Tags:[]string{"admin", "early"}}
        	            	
        	            	Diff:
        	            	--- Expected
        	            	+++ Actual
        	            	@@ -3,6 +3,6 @@
        	            	  Name: (string) (len=3) "Ada",
        	            	- Email: (string) (len=15) "ada@example.org",
        	            	+ Email: (string) (len=15) "ada@example.com",
        	            	  Tags: ([]string) (len=2) {
        	            	   (string) (len=5) "admin",
        	            	-  (string) (len=7) "founder"
        	            	+  (string) (len=5) "early"
        	            	  }
        	Test:       	TestStructDiff
--- FAIL: TestStructDiff (0.00s)
```

Two differences pinpointed, with lengths shown so a trailing space would be visible. A
hand-written `reflect.DeepEqual` check plus `t.Errorf("got %+v, want %+v", ...)` prints two long
lines and leaves you to spot the difference. That gap is the honest case for testify.

## suite.Suite

For xUnit-style setup/teardown around a group of tests sharing state.

```go
package u

import (
  "testing"

  "github.com/stretchr/testify/suite"
)

type CounterSuite struct {
  suite.Suite
  n int
}

func (s *CounterSuite) SetupTest()    { s.n = 10 }
func (s *CounterSuite) TearDownTest() { s.T().Logf("teardown, n=%d", s.n) }

func (s *CounterSuite) TestIncrement() {
  s.n++
  s.Equal(11, s.n)
}

func (s *CounterSuite) TestReset() {
  s.n = 0
  s.Zero(s.n)
}

func TestCounterSuite(t *testing.T) {
  suite.Run(t, new(CounterSuite))
}
```

`go test -count=1 -v -run TestCounterSuite .`. Output:

```
=== RUN   TestCounterSuite
=== RUN   TestCounterSuite/TestIncrement
    suite_test.go:15: teardown, n=11
=== RUN   TestCounterSuite/TestReset
    suite_test.go:15: teardown, n=0
--- PASS: TestCounterSuite (0.00s)
    --- PASS: TestCounterSuite/TestIncrement (0.00s)
    --- PASS: TestCounterSuite/TestReset (0.00s)
PASS
ok  	example.com/scratch	0.541s
```

Methods become subtests, assertions are methods on the embedded suite (`s.Equal`), and
`SetupTest` runs before each — which is why `TestIncrement` sees 10 and not whatever `TestReset`
left. `SetupSuite`/`TearDownSuite` are the once-per-suite versions, and the hook names must match
exactly or they silently never run.

The trade-off: suites interact awkwardly with `t.Parallel`, and shared mutable state between test
methods is exactly what [14.3](03-fixtures-helpers.md) warns about. A table test plus a setup
helper does the same job with less machinery.

## mock.Mock

```go
type Mailer interface {
  Send(to, body string) error
}

type MockMailer struct{ mock.Mock }

func (m *MockMailer) Send(to, body string) error {
  args := m.Called(to, body)
  return args.Error(0)
}

func Notify(m Mailer, u *User) error {
  return m.Send(u.Email, "welcome, "+u.Name)
}

func TestNotifyWithMock(t *testing.T) {
  m := new(MockMailer)
  m.On("Send", "ada@example.com", "welcome, Ada").Return(nil).Once()

  u, err := Find(1)
  require.NoError(t, err)
  require.NoError(t, Notify(m, u))

  m.AssertExpectations(t)
}
```

Output:

```
=== RUN   TestNotifyWithMock
--- PASS: TestNotifyWithMock (0.00s)
```

`On(...)` declares an expected call with exact arguments, `Return` supplies results, `Once`
constrains the count, and `AssertExpectations` fails if a declared call never happened. Argument
matching is by value, with `mock.Anything` and `mock.MatchedBy(fn)` as escapes.

The costs: the method bodies are hand-written boilerplate anyway, argument mismatches are runtime
failures rather than compile errors, and the string `"Send"` is never checked against the
interface. **A hand-written fake is usually better in Go** — a short struct with the behaviour you
need, type-checked at compile time. [14.8](08-test-doubles.md) builds those.

## Honest trade-offs

**Stdlib `if got != want`** is idiomatic, dependency-free, and the only thing you can rely on
being present in any Go codebase. The standard library and most of the Go core team's code uses
it exclusively. Its weakness is composite comparison: you need `reflect.DeepEqual`,
`slices.Equal`, or hand-rolled diff output.

**testify** is terser for long assertion runs and much better at telling you *what* differs in a
struct. Its costs: a dependency tree of five modules, `(t, expected, actual)` argument order that
inverts the Go convention, the `int`/`int64` strictness, and a pull toward suites and mocks that
Go code generally reads better without.

**go-cmp** (`github.com/google/go-cmp`) is the middle ground and what the Go team actually
recommends for comparing composites. It is one import, it has no assertion DSL, and `cmp.Diff`
returns a string that is empty when equal:

```go
package u

import (
  "testing"

  "github.com/google/go-cmp/cmp"
)

func TestCmpDiff(t *testing.T) {
  got, err := Find(1)
  if err != nil {
    t.Fatalf("Find(1): %v", err)
  }
  want := &User{ID: 1, Name: "Ada", Email: "ada@example.org", Tags: []string{"admin", "founder"}}
  if d := cmp.Diff(want, got); d != "" {
    t.Errorf("Find(1) mismatch (-want +got):\n%s", d)
  }
}
```

Fails on purpose, with go-cmp v0.7.0. Output:

```
=== RUN   TestCmpDiff
    cmp_test.go:16: Find(1) mismatch (-want +got):
          &u.User{
          	ID:    1,
          	Name:  "Ada",
        - 	Email: "ada@example.org",
        + 	Email: "ada@example.com",
          	Tags: []string{
          		"admin",
        - 		"founder",
        + 		"early",
          	},
          }
--- FAIL: TestCmpDiff (0.00s)
FAIL
FAIL	example.com/scratch	0.253s
FAIL
```

Cleaner than testify's for the same mismatch, in Go syntax, with the `(-want +got)` legend saying
which sign is which. go-cmp also offers `cmpopts.IgnoreFields`, `cmpopts.EquateApprox` and
`cmp.Comparer`, and it panics on unexported fields unless you opt in — deliberate, so tests do not
quietly depend on internals.

**The recommendation for this course:** stdlib `if got != want` by default, `cmp.Diff` when
comparing composites. Use testify when you join a codebase that already has it — mixing both
styles in one package is worse than either.

## Quick Reference

| Package | On failure |
|---|---|
| `github.com/stretchr/testify/assert` | `t.Errorf` — continue |
| `github.com/stretchr/testify/require` | `t.FailNow` — stop |
| `github.com/stretchr/testify/suite` | xUnit-style `SetupTest`/`TearDownTest` |
| `github.com/stretchr/testify/mock` | expectation-based mocks |
| `github.com/google/go-cmp/cmp` | no assertions; `cmp.Diff` returns a diff string |

| Gotcha | Detail |
|---|---|
| Argument order | `(t, expected, actual)` — the reverse of `got, want` |
| `Equal` typing | `reflect.DeepEqual`; `1 != int64(1)` |
| `EqualValues` | converts types first; hides real type bugs |
| `assert.NoError` then dereference | continues on failure and panics; use `require` |
| Suite hook names | must match exactly or they silently never run |
| `mock.Mock` method names | strings, not checked against the interface |

## Common Mistakes

**Writing `assert.Equal(t, got, want)`.** The labels in the diff are then reversed, so the output
blames the wrong side.

**Using `assert.NoError` before dereferencing the result.** Execution continues past the failure
and the test panics on a nil pointer — `require.NoError` is what you want.

**Expecting `assert.Equal(t, 1, int64(1))` to pass.** `reflect.DeepEqual` requires identical
types; convert explicitly rather than reaching for `EqualValues`.

**Reaching for `mock.Mock` for a two-method interface.** A hand-written fake is shorter,
compile-checked, and readable — see [14.8](08-test-doubles.md).

**Mixing testify and stdlib assertions in one package.** Readers cannot tell which failure format
to expect; pick one per package.

## Exercises

1. Write one test with four `assert` calls where the second fails, then convert it to `require`.
   Capture both outputs and state exactly how many assertions ran in each.
2. Write a function returning `time.Duration` and assert its value against an untyped `100`.
   Capture the failure, then fix it two ways and argue for one.
3. Take a 6-field struct with a nested slice and a map. Write the mismatch assertion three ways —
   `reflect.DeepEqual` + `t.Errorf`, `assert.Equal`, `cmp.Diff` — and rank the three failure
   messages on how fast they let you find the wrong field.
4. Convert the `CounterSuite` above into a table-driven test with a setup helper and no suite.
   Compare the line counts and decide which you would keep.
5. Replace the `MockMailer` with a hand-written fake that records the calls it received, and
   assert on the recorded slice. Which version fails to compile if you add a parameter to `Send`?

---

[← 14.3 Fixtures, Helpers, TestMain, and Cleanup](03-fixtures-helpers.md) · [Course Index](../README.md) · [14.5 Benchmarks →](05-benchmarks.md)
