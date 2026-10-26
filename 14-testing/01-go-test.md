[← 13.15 reflect: The Basics](../13-stdlib/15-reflect.md) · [Course Index](../README.md) · [14.2 Table-Driven Tests and Subtests →](02-table-driven.md)

# 14.1 — go test and the Basics

## What You'll Learn

- The `_test.go` convention, `func TestXxx(t *testing.T)`, and why test files never reach your binary
- `package foo` vs `package foo_test`: internal tests see everything, external tests see only the public API
- `t.Error` continues, `t.Fatal` stops — shown in real output, not described
- The `go test` flag surface, the `(cached)` line, exit codes, and the if-got-want convention that replaces an assertion library

## Coming From Another Language

| Other | Go |
|---|---|
| Jest/Vitest/pytest are third-party installs | `testing` is in the standard library; `go test` is in the toolchain |
| `expect(x).toBe(y)`, `assert x == y` | no assertions at all — you write `if got != want { t.Errorf(...) }` |
| Tests live in `__tests__/` or `tests/` | tests live *beside* the code, as `foo_test.go` in the same directory |
| `describe`/`it` blocks, nested | `func TestXxx(*testing.T)` plus `t.Run` subtests ([14.2](02-table-driven.md)) |
| `#[cfg(test)] mod tests` in Rust | same idea — the compiler drops test code from normal builds, no attribute needed |
| pytest fixtures, `beforeEach` | `t.Cleanup`, `t.TempDir`, `TestMain` ([14.3](03-fixtures-helpers.md)) |

The one that trips everyone up: Go ships no assertion library and the team has no intention of
adding one. You compare values with `==` or a comparison helper and you write the failure message
yourself. That feels primitive for a day and then stops bothering you, because a hand-written
message says exactly what went wrong while `expected true to be false` does not. [14.4](04-testify.md)
covers testify for when you want the terser form anyway.

## The file and function convention

Three rules, and the toolchain enforces all of them.

1. The file name must end in `_test.go`.
2. The function must be `func TestXxx(t *testing.T)` — exported-style name, `Xxx` must not start
   with a lowercase letter, and the signature must match exactly.
3. The file lives in the same directory as the code it tests.

A package to test. `stringsx.go`:

```go
package stringsx

import "strings"

func Reverse(s string) string {
  r := []rune(s)
  for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
    r[i], r[j] = r[j], r[i]
  }
  return string(r)
}

func Title(s string) string {
  if s == "" {
    return ""
  }
  return strings.ToUpper(s[:1]) + s[1:]
}

func wordCount(s string) int {
  return len(strings.Fields(s))
}

func WordCount(s string) int { return wordCount(s) }
```

`stringsx_test.go` — same package, so it can reach the unexported `wordCount`:

```go
package stringsx

import "testing"

func TestReverse(t *testing.T) {
  got := Reverse("Hello")
  want := "olleH"
  if got != want {
    t.Errorf("Reverse(%q) = %q, want %q", "Hello", got, want)
  }
}

func TestTitle(t *testing.T) {
  if got := Title("go"); got != "Go" {
    t.Errorf("Title(%q) = %q, want %q", "go", got, "Go")
  }
}

func TestWordCountUnexported(t *testing.T) {
  if got := wordCount("a b c"); got != 3 {
    t.Errorf("wordCount() = %d, want 3", got)
  }
}
```

`go test -count=1 ./...`. Output:

```
ok  	example.com/scratch	0.581s
```

That is the whole passing report: `ok`, the package path, elapsed time. Note the elapsed time —
**every `go test` timing in this module varies run to run**, so read `0.581s` as "some small
number" and never diff it. The rest of this module uses `-count=1` consistently, for a reason
covered below.

Test files are excluded from the regular build. There is no build tag, no directory convention, no
flag: `go build` and `go install` simply skip every `_test.go` file in the package. So a test can
import `testing`, declare helpers, and pull in heavy test-only dependencies without any of it
reaching your production binary.

## Internal vs external test packages

A directory may contain **two** packages: `foo` and `foo_test`. Both can hold tests, and `go test`
compiles and runs both.

| | `package stringsx` | `package stringsx_test` |
|---|---|---|
| Name | internal test package | external test package |
| Sees unexported identifiers | yes | no |
| Must import the package under test | no | yes |
| Can import packages that import it | no (import cycle) | yes |
| Tests | implementation details | the public API, as a user sees it |

Add a second file in the same directory. `stringsx_ext_test.go`:

```go
package stringsx_test

import (
  "testing"

  "example.com/scratch"
)

func TestReversePublic(t *testing.T) {
  if got := stringsx.Reverse("abc"); got != "cba" {
    t.Errorf("Reverse(%q) = %q, want %q", "abc", got, "cba")
  }
}
```

The import path is the module/package path; the identifier you then use is the package *name*
(`stringsx`), which is why the import looks mismatched here — the scratch module is
`example.com/scratch` but the package in it is `stringsx`.

`go test -count=1 -v ./...` runs tests from both packages in one binary. Output:

```
=== RUN   TestReverse
--- PASS: TestReverse (0.00s)
=== RUN   TestTitle
--- PASS: TestTitle (0.00s)
=== RUN   TestWordCountUnexported
--- PASS: TestWordCountUnexported (0.00s)
=== RUN   TestReversePublic
--- PASS: TestReversePublic (0.00s)
PASS
ok  	example.com/scratch	0.508s
```

`package foo_test` enforces that you only touch the public API. Reach for an unexported name and it
does not compile. With `wordCount` unexported:

```go
package stringsx_test

import (
  "testing"

  "example.com/scratch"
)

func TestUnexported(t *testing.T) {
  if stringsx.wordCount("a b") != 2 {
    t.Error("bad")
  }
}
```

This does not compile. Output:

```
# example.com/scratch_test [example.com/scratch.test]
./vis_test.go:10:15: name wordCount not exported by package stringsx
```

Note the header: `example.com/scratch_test` is the external test package, built as part of the
`example.com/scratch.test` binary. (If nothing inside the package referenced `wordCount` either,
the compiler reports `undefined: stringsx.wordCount` instead — same cause, different wording
depending on whether the symbol survived to the export data.)

**Idiomatic:** default to `package foo` (internal) because it is less typing and lets you unit-test
helpers; switch a file to `package foo_test` when you specifically want to pin the public API, write
documentation-grade examples ([14.7](07-examples-coverage.md)), or break an import cycle — a test
that needs to import something that imports your package has no other option.

## t.Error vs t.Fatal

`Error`/`Errorf` mark the test failed and **keep going**. `Fatal`/`Fatalf` mark it failed and
**stop that test function immediately**.

```go
package calc

import "testing"

func TestWithError(t *testing.T) {
  t.Error("first problem")
  t.Error("second problem")
  t.Log("this line still runs")
}

func TestWithFatal(t *testing.T) {
  t.Fatal("first problem")
  t.Error("never reached")
}
```

Both tests fail on purpose. `go test -count=1 -v -run 'TestWith' .`. Output:

```
=== RUN   TestWithError
    errfatal_test.go:6: first problem
    errfatal_test.go:7: second problem
    errfatal_test.go:8: this line still runs
--- FAIL: TestWithError (0.00s)
=== RUN   TestWithFatal
    errfatal_test.go:12: first problem
--- FAIL: TestWithFatal (0.00s)
FAIL
FAIL	example.com/scratch	0.513s
FAIL
```

Three messages from the first test, one from the second. `TestWithFatal` never reached its second
line.

Under the hood `Fatal` calls `runtime.Goexit()`, which unwinds the test's goroutine running deferred
functions. Two consequences worth knowing:

- Deferred cleanup still runs after a `Fatal`. It is not `os.Exit`.
- `Fatal` must be called from the goroutine running the test. Calling it from a goroutine you
  spawned kills *that* goroutine and the test carries on, usually hanging or passing wrongly. From a
  helper goroutine, use `t.Error` and return, or send the failure back over a channel.

**Idiomatic:** `Fatal` when continuing is pointless or unsafe — a failed setup, a `nil` you are about
to dereference. `Error` when you want to see every independent problem in one run.

```go
f, err := os.Open(path)
if err != nil {
  t.Fatalf("open %s: %v", path, err)
}
if got := f.Name(); got != path {
  t.Errorf("Name() = %q, want %q", got, path)
}
```

## Logging, failing, and skipping

| Method | Effect |
|---|---|
| `t.Log(args...)` / `t.Logf(f, args...)` | record a message; shown only with `-v` or if the test fails |
| `t.Error` / `t.Errorf` | log + `Fail()` — test fails, execution continues |
| `t.Fatal` / `t.Fatalf` | log + `FailNow()` — test fails, function stops |
| `t.Fail()` | mark failed with no message |
| `t.FailNow()` | mark failed and stop, no message |
| `t.Failed() bool` | has this test failed so far? |
| `t.Skip` / `t.Skipf` | log + `SkipNow()` — stop and report SKIP |
| `t.SkipNow()` | stop and report SKIP, no message |
| `t.Skipped() bool` | was this test skipped? |
| `t.Name() string` | the test's full name, e.g. `TestFoo/sub_case` |
| `t.Attr(key, value)` | emit a key/value attribute for CI tooling |
| `t.Output() io.Writer` | a writer feeding the test log, for multi-line output |

A skip is not a failure: the package still reports `ok`. Use it for environment-dependent tests.

```go
package calc

import (
  "runtime"
  "testing"
)

func TestLogsOnPass(t *testing.T) {
  t.Log("setup done")
  if Add(1, 1) != 2 {
    t.Errorf("Add(1, 1) = %d, want 2", Add(1, 1))
  }
}

func TestSkippedAlways(t *testing.T) {
  t.Skip("not implemented yet")
}

func TestSkipOnPlatform(t *testing.T) {
  if runtime.GOOS != "linux" {
    t.Skipf("requires linux, running on %s", runtime.GOOS)
  }
  t.Log("linux-only body")
}

func TestSlow(t *testing.T) {
  if testing.Short() {
    t.Skip("skipping slow test in -short mode")
  }
  t.Log("doing the slow thing")
}

func TestReportsFailed(t *testing.T) {
  t.Errorf("deliberate")
  t.Logf("t.Failed() = %v", t.Failed())
}
```

`TestReportsFailed` fails on purpose. First `go test -count=1 .` with no `-v`. Output:

```
--- FAIL: TestReportsFailed (0.00s)
    skip_test.go:34: deliberate
    skip_test.go:35: t.Failed() = true
FAIL
FAIL	example.com/scratch	0.608s
FAIL
```

This is the rule people miss: **without `-v` you see logs only from tests that failed.** The passing
`TestLogsOnPass` logged "setup done" and you cannot see it; the skips are invisible too. The
failing test dumps its whole log, including the `t.Logf` that proves `t.Failed()` returned `true`
mid-test.

Now `go test -count=1 -v .`. Output:

```
=== RUN   TestLogsOnPass
    skip_test.go:9: setup done
--- PASS: TestLogsOnPass (0.00s)
=== RUN   TestSkippedAlways
    skip_test.go:16: not implemented yet
--- SKIP: TestSkippedAlways (0.00s)
=== RUN   TestSkipOnPlatform
    skip_test.go:21: requires linux, running on darwin
--- SKIP: TestSkipOnPlatform (0.00s)
=== RUN   TestSlow
    skip_test.go:30: doing the slow thing
--- PASS: TestSlow (0.00s)
=== RUN   TestReportsFailed
    skip_test.go:34: deliberate
    skip_test.go:35: t.Failed() = true
--- FAIL: TestReportsFailed (0.00s)
FAIL
FAIL	example.com/scratch	0.495s
FAIL
```

Because logs are buffered until a test ends, writing `t.Log` as a progress indicator does not work —
nothing appears while the test runs. For long tests, log at the end or write to `os.Stderr`.

## testing.Short() and -short

`testing.Short()` reports whether `-short` was passed. It is just a bool; what it means is entirely
up to you, by convention "skip the expensive ones".

`go test -count=1 -v -short -run 'TestSlow' .`. Output:

```
=== RUN   TestSlow
    skip_test.go:28: skipping slow test in -short mode
--- SKIP: TestSlow (0.00s)
PASS
ok  	example.com/scratch	0.189s
```

The package reports `ok` and exit status 0 — skipped tests do not fail a build. The usual pairing is
a fast `go test -short ./...` on every save and the full suite in CI. Build tags are the heavier
alternative when a test should not even compile by default ([14.3](03-fixtures-helpers.md)).

## t.Attr and t.Output

Both are newer additions and both are mainly for tooling.

```go
package p

import (
  "fmt"
  "testing"
)

func TestAttr(t *testing.T) {
  t.Attr("owner", "payments-team")
  t.Attr("issue", "GO-1421")
  if Add(2, 2) != 4 {
    t.Error("bad")
  }
}

func TestOutput(t *testing.T) {
  fmt.Fprintf(t.Output(), "table:\n")
  fmt.Fprintf(t.Output(), "  a=%d b=%d\n", 1, 2)
}
```

`go test -count=1 -v .`. Output:

```
=== RUN   TestAttr
=== ATTR  TestAttr owner payments-team
=== ATTR  TestAttr issue GO-1421
--- PASS: TestAttr (0.00s)
=== RUN   TestOutput
    table:
      a=1 b=2
--- PASS: TestOutput (0.00s)
PASS
ok  	example.com/scratch	0.503s
```

`t.Attr` writes `=== ATTR` lines, and crucially a structured record under `-json`, which is how a CI
system picks it up:

```json
{"Action":"attr","Package":"example.com/scratch","Test":"TestAttr","Key":"owner","Value":"payments-team"}
```

The key may not contain whitespace and the value may not contain newlines. What an attribute
*means* is left to your CI system — ownership, a ticket ID, a flakiness marker.

`t.Output()` returns an `io.Writer` into the test log. Unlike `t.Log` it adds no `file:line` prefix
and no trailing newline, so it is the right target for a multi-line table or a hex dump you want
indented into the log — handy when you are printing a diff yourself.

## t.Parallel, briefly

`t.Parallel()` signals that this test may run concurrently with other parallel tests. The test
pauses, the parent finishes its serial part, then the parallel set resumes together.

```go
package p

import (
  "testing"
  "time"
)

func TestSerialA(t *testing.T) {
  time.Sleep(50 * time.Millisecond)
  t.Log("serial A done")
}

func TestParallelB(t *testing.T) {
  t.Parallel()
  time.Sleep(50 * time.Millisecond)
  t.Log("parallel B done")
}

func TestParallelC(t *testing.T) {
  t.Parallel()
  time.Sleep(50 * time.Millisecond)
  t.Log("parallel C done")
}
```

`go test -count=1 -v .` — the two parallel tests finish in whichever order the scheduler picks, so
B and C can swap in the output. Output:

```
=== RUN   TestSerialA
    par_test.go:10: serial A done
--- PASS: TestSerialA (0.05s)
=== RUN   TestParallelB
=== PAUSE TestParallelB
=== RUN   TestParallelC
=== PAUSE TestParallelC
=== CONT  TestParallelB
=== CONT  TestParallelC
    par_test.go:22: parallel C done
--- PASS: TestParallelC (0.05s)
=== NAME  TestParallelB
    par_test.go:16: parallel B done
--- PASS: TestParallelB (0.05s)
PASS
ok  	example.com/scratch	0.593s
```

Read the trace: each parallel test gets `=== RUN` then immediately `=== PAUSE`. Only once every
top-level test has been started does the runner issue `=== CONT` and let them run together — that
is why the total is about 0.1s, not 0.15s. `=== NAME` is a marker the runner prints when it switches
back to reporting a different test's buffered log.

Two things to carry forward: `-parallel n` caps how many run at once (default `GOMAXPROCS`), and
**parallel subtests run after their parent's serial part returns**, which has real consequences for
cleanup and shared fixtures. [14.3](03-fixtures-helpers.md) covers that, and
[14.2](02-table-driven.md) covers `t.Parallel()` inside `t.Run`.

## The flag table

| Flag | Effect |
|---|---|
| `./...` | all packages in the module, recursively (a package pattern, not a flag) |
| `-v` | verbose: every test's `RUN`/`PASS`/`SKIP`, plus all `t.Log` output |
| `-run regexp` | run only matching tests; split on `/` for subtests (`-run 'TestX/case'`) |
| `-skip regexp` | the inverse of `-run` |
| `-count n` | run each test n times; `-count=1` defeats the test cache |
| `-race` | enable the race detector ([11.5](../11-sync-context/05-race-detector.md)) |
| `-cover` | report statement coverage ([14.7](07-examples-coverage.md)) |
| `-coverprofile f` | write a coverage profile to `f`; implies `-cover` |
| `-coverpkg patterns` | measure coverage of other packages too |
| `-timeout d` | panic with a full goroutine dump after d (default `10m`); `0` disables |
| `-failfast` | do not start new tests after the first failure |
| `-shuffle on\|off\|N` | randomize test order; prints the seed for reproduction |
| `-json` | machine-readable event stream (same info as `-v`) |
| `-list regexp` | list matching tests/benchmarks/fuzz targets without running them |
| `-short` | sets `testing.Short()`; you decide what it skips |
| `-parallel n` | max concurrently running `t.Parallel()` tests (default `GOMAXPROCS`) |
| `-cpu 1,2,4` | run the whole suite once per `GOMAXPROCS` value |
| `-bench regexp` | run matching benchmarks ([14.5](05-benchmarks.md)) |
| `-benchmem` | add allocation columns to benchmark output |
| `-benchtime t\|Nx` | how long or how many iterations per benchmark |
| `-fuzz regexp` | fuzz the one matching target ([14.6](06-fuzzing.md)) |
| `-fuzztime t\|Nx` | how long to fuzz |
| `-c` | compile the test binary and do not run it |
| `-vet list` | control the vet checks `go test` runs first; `-vet=off` disables |
| `-fullpath` | absolute file names in failure messages |

`-run` takes a **regular expression**, not a glob, and it is unanchored:

```bash
go test -run TestReverse ./...      # matches TestReverse AND TestReversePublic
go test -run '^TestReverse$' ./...  # exactly one
go test -run 'Reverse|Title' ./...  # either
go test -list '.*' ./...            # what is in here?
```

`go test -list '.*' .`. Output:

```
TestLogsOnPass
TestSkippedAlways
TestSkipOnPlatform
TestSlow
TestReportsFailed
ok  	example.com/scratch	0.190s
```

`-list` shows only top-level names — subtests are created at runtime, so they cannot be listed.

## The (cached) line

`go test` caches results. If the package's source, its dependencies, the flags and the environment
are all unchanged, the second run does not execute anything.

Run `go test ./...` twice in a row:

```
ok  	example.com/scratch	0.192s
ok  	example.com/scratch	(cached)
```

That `(cached)` confuses people into thinking the test was skipped, or that their edit was ignored.
It was neither: the toolchain is telling you the inputs are identical, so the recorded `ok` still
applies. Caching only happens for runs it can characterize — a test that reads an unexpected file or
an undeclared environment variable is not cached.

`-count=1` is the standard way to force a real run:

```
ok  	example.com/scratch	0.204s
```

`-count=1` is not special-cased; it just makes the flag set differ from any cached entry, and every
count other than the default still runs. Use it whenever an `ok` needs to mean "it ran just now" —
in CI, after changing something the cache cannot see, and in every example in this module.

## Compiling the test binary, and exit codes

`go test -c` compiles the package's tests into an executable instead of running them. Useful for
copying a test to another machine, running it under a debugger, or passing flags repeatedly.

```bash
go test -c -o pkg.test .
./pkg.test -test.v -test.run TestSerialA
```

Output:

```
=== RUN   TestSerialA
    par_test.go:10: serial A done
--- PASS: TestSerialA (0.05s)
PASS
```

Note the `-test.` prefix: when you run the binary directly it is **mandatory**. Via `go test` it is
optional (`-v` and `-test.v` both work).

Exit codes, verified on this toolchain:

| Situation | Exit status |
|---|---|
| all tests pass (or skip) | 0 |
| any test fails | 1 |
| the package fails to compile | 1 (with `[build failed]`) |
| a data race under `-race` | 1 (the test is failed) |
| `-timeout` exceeded | 1, after a full goroutine dump |

There is no distinct code for "failed to build" versus "test failed", so CI scripts key off the
output, not the status. A `-failfast` run still exits 1:

```
--- FAIL: TestOne (0.00s)
    ff_test.go:5: one failed
FAIL
FAIL	example.com/scratch	0.203s
FAIL
```

Without it, both failures are reported:

```
--- FAIL: TestOne (0.00s)
    ff_test.go:5: one failed
--- FAIL: TestTwo (0.00s)
    ff_test.go:6: two failed
FAIL
FAIL	example.com/scratch	0.509s
FAIL
```

`-failfast` stops starting *new* tests; it does not kill tests already running, so with
`t.Parallel()` you can still see several failures.

## No assertions: the if-got-want convention

The standard library gives you `if`, `==`, and a formatted message. The convention that has
settled across the Go ecosystem is to name your two values `got` and `want` and report both.

The whole value of the convention is in the failure message. Two tests of the same bug:

```go
package calc

import "testing"

func TestAddBadMessage(t *testing.T) {
  if Add(2, 3) != 6 {
    t.Error("Add failed")
  }
}

func TestAddGoodMessage(t *testing.T) {
  a, b := 2, 3
  got, want := Add(a, b), 6
  if got != want {
    t.Errorf("Add(%d, %d) = %d, want %d", a, b, got, want)
  }
}
```

Both fail on purpose. `go test -count=1 -v -run 'TestAdd' .`. Output:

```
=== RUN   TestAddBadMessage
    msg_test.go:7: Add failed
--- FAIL: TestAddBadMessage (0.00s)
=== RUN   TestAddGoodMessage
    msg_test.go:15: Add(2, 3) = 5, want 6
--- FAIL: TestAddGoodMessage (0.00s)
FAIL
FAIL	example.com/scratch	0.488s
FAIL
```

`Add failed` sends you to the source to find out what was even called. `Add(2, 3) = 5, want 6`
diagnoses the bug from the log line alone — and here the expectation is the thing that is wrong,
which the good message makes obvious and the bad one hides.

The format to internalize: **what you called, what you got, what you wanted.**

```go
t.Errorf("Reverse(%q) = %q, want %q", in, got, want)
t.Errorf("Parse(%q): unexpected error: %v", in, err)
t.Errorf("len(items) = %d, want %d", len(items), 3)
```

Conventions that come with it:

- `%q` for strings, always. It shows quotes, escapes and invisible characters; `%v` would print
  `foo ` and ` foo` identically to the eye.
- `%v` for arbitrary values, `%#v` when you need to see types and structure.
- No trailing punctuation and no "Error:" prefix — `go test` already prefixes `file:line`.
- Lowercase start, like error strings ([8.1](../08-errors/01-error-interface.md)).
- `got, want` in that order, every time. Readers of Go tests parse it without thinking; reversing
  it in one test will cost someone an hour.

For values that `==` cannot compare — slices, maps, structs with slices in them — use
`slices.Equal`, `maps.Equal` ([7.8](../07-slices-maps/08-slices-package.md),
[7.9](../07-slices-maps/09-maps-package.md)), `reflect.DeepEqual`, or `go-cmp`'s `cmp.Diff` for a
readable diff. [14.4](04-testify.md) covers both testify and go-cmp.

## Quick Reference

| Item | Form |
|---|---|
| Test file | `foo_test.go`, in the package's own directory |
| Test function | `func TestXxx(t *testing.T)` |
| Internal test package | `package foo` — sees unexported identifiers |
| External test package | `package foo_test` — public API only |
| Run everything | `go test ./...` |
| Force a real run | `go test -count=1 ./...` |
| One test | `go test -run '^TestName$' ./...` |
| See logs and names | `go test -v ./...` |
| Standard CI gate | `go test -race -count=1 ./...` |

| `*testing.T` method | Fails? | Stops? | Note |
|---|---|---|---|
| `Log`, `Logf` | no | no | visible with `-v` or on failure |
| `Error`, `Errorf` | yes | no | the default choice |
| `Fatal`, `Fatalf` | yes | yes | `runtime.Goexit`; defers still run |
| `Fail` | yes | no | no message |
| `FailNow` | yes | yes | no message |
| `Failed` | — | — | returns the current state |
| `Skip`, `Skipf` | no | yes | reports `SKIP`, package stays `ok` |
| `SkipNow` | no | yes | no message |
| `Skipped` | — | — | returns the current state |
| `Parallel` | — | — | pause, then run with other parallel tests |
| `Name` | — | — | `TestFoo` or `TestFoo/sub_name` |
| `Attr` | no | no | `=== ATTR` line plus a `-json` record |
| `Output` | no | no | `io.Writer` into the test log |

## Common Mistakes

**Calling `t.Fatal` from a goroutine you spawned.** It runs `runtime.Goexit` on *that* goroutine, so
the test keeps running and may pass; use `t.Error` and return, or report the failure over a channel.

**Being confused by `(cached)` and thinking the test did not run.** The inputs were identical so the
recorded result still stands; add `-count=1` whenever you need proof it executed now.

**Writing `t.Error("failed")`.** The message is the entire value of a hand-written test — use
`t.Errorf("F(%v) = %v, want %v", in, got, want)` so the log line alone diagnoses the bug.

**Expecting `t.Log` to appear while the test runs.** Logs are buffered per test and flushed at the
end, and without `-v` a passing test's logs are discarded entirely.

**Reversing `got` and `want`.** Every Go reader assumes got-then-want; a swapped message makes a
correct value look like the bug.

**Using `-run TestFoo` and wondering why `TestFooBar` ran.** `-run` is an unanchored regexp; write
`-run '^TestFoo$'` for exactly one.

**Putting tests in a `tests/` directory.** They must sit beside the code they test, or they cannot
be an internal test package and `go test ./...` will not associate them with the package.

## Exercises

1. Write a `Clamp(v, lo, hi int) int` with a test for each of: below range, in range, above range,
   and `lo > hi`. Make one fail on purpose and confirm the message tells you what was called,
   what you got and what you wanted — without opening the test file.
2. Take a package with an unexported helper. Write the same test twice, once in `package foo` and
   once in `package foo_test`, and capture the exact compile error from the external version
   touching the unexported name.
3. Write a test that fails three independent assertions. Run it with `t.Error` and with `t.Fatal`,
   and record how many messages each produced and why.
4. Run a package's tests twice with no changes, then once with `-count=1`, then edit a comment in a
   non-test file and run again. Explain each `(cached)` or timing line you saw.
5. Add `t.Attr("owner", ...)` to three tests and write a one-line shell pipeline over
   `go test -json` that prints each test name with its owner.
6. Build a test binary with `go test -c`, then run it directly with `-test.run`, `-test.v` and
   `-test.count=2`. Confirm what happens if you drop the `-test.` prefix.

---

[← 13.15 reflect: The Basics](../13-stdlib/15-reflect.md) · [Course Index](../README.md) · [14.2 Table-Driven Tests and Subtests →](02-table-driven.md)
