[← 14.2 Table-Driven Tests and Subtests](02-table-driven.md) · [Course Index](../README.md) · [14.4 testify, Briefly →](04-testify.md)

# 14.3 — Fixtures, Helpers, TestMain, and Cleanup

## What You'll Learn

- `t.Cleanup` runs LIFO, survives `t.Fatal`, and lets a *helper* register its own teardown — which `defer` cannot
- `t.Helper()` moves the reported line from inside your assertion helper to the call site
- `t.TempDir`, `t.Setenv`, `t.Chdir`: what is cleaned up for you, and which of them forbid `t.Parallel`
- `TestMain`, build-tag-gated integration tests, golden files with `-update`, `testdata/`, and `-shuffle`

## Coming From Another Language

| Other | Go |
|---|---|
| pytest fixtures with `yield` and scopes | `t.Cleanup` for per-test, `TestMain` for per-package. No scope system |
| `beforeEach`/`afterEach` | just code at the top of the test, plus `t.Cleanup` |
| `conftest.py`, `jest.setup.js` | `TestMain(m *testing.M)` in one `_test.go` file |
| `tmp_path` fixture | `t.TempDir()`, auto-removed, no fixture plumbing |
| `monkeypatch.setenv` | `t.Setenv`, auto-restored |
| Jest snapshots (`toMatchSnapshot`) | golden files in `testdata/`, with your own `-update` flag |
| `@pytest.mark.integration` + `-m` | `//go:build integration` + `-tags integration` |

Go has no fixture framework and no dependency injection for tests. Setup is a function call at the
top of your test; teardown is `t.Cleanup`. That is the whole system, and it scales further than it
looks because `t.Cleanup` composes.

## t.Cleanup, and why it beats defer

`t.Cleanup(f)` registers `f` to run when the test (and all its subtests) finish. Registered
functions run in **LIFO** order, like `defer`.

```go
package p

import "testing"

func TestCleanupOrder(t *testing.T) {
  t.Cleanup(func() { t.Log("cleanup 1 (registered first)") })
  t.Cleanup(func() { t.Log("cleanup 2") })
  t.Cleanup(func() { t.Log("cleanup 3 (registered last)") })
  t.Log("test body")
}

func TestCleanupVsDefer(t *testing.T) {
  defer t.Log("defer in the test body")
  t.Cleanup(func() { t.Log("cleanup in the test body") })
  t.Log("body")
}
```

`go test -count=1 -v -run 'TestCleanupOrder|TestCleanupVsDefer' .`. Output:

```
=== RUN   TestCleanupOrder
    clean_test.go:9: test body
    clean_test.go:8: cleanup 3 (registered last)
    clean_test.go:7: cleanup 2
    clean_test.go:6: cleanup 1 (registered first)
--- PASS: TestCleanupOrder (0.00s)
=== RUN   TestCleanupVsDefer
    clean_test.go:15: body
    clean_test.go:16: defer in the test body
    clean_test.go:14: cleanup in the test body
--- PASS: TestCleanupVsDefer (0.00s)
PASS
ok  	example.com/scratch	0.508s
```

Note the second test: within the test function itself, `defer` runs **before** any `t.Cleanup`,
because the deferred call fires as the function returns while cleanups run after. Mixing the two in
one function is therefore a readability trap.

So why bother, when `defer` is shorter? Because **`defer` is bound to the function it is written
in**, and setup usually belongs in a helper. A helper that opens a resource cannot `defer` its
close — the `defer` would fire when the *helper* returns, destroying the thing it just handed you.

This does not work:

```go
func newStoreBroken(t *testing.T) *Store {
  dir, _ := os.MkdirTemp("", "s")
  defer os.RemoveAll(dir)
  s, _ := Open(dir)
  return s
}
```

The directory is gone before the test uses the store. `t.Cleanup` is bound to the *test*, not the
helper, so this does (assuming a `Store` with a `Close() error` method):

```go
func newStore(t *testing.T) *Store {
  t.Helper()
  dir := t.TempDir()
  s, err := Open(dir)
  if err != nil {
    t.Fatalf("Open(%s): %v", dir, err)
  }
  t.Cleanup(func() {
    if err := s.Close(); err != nil {
      t.Errorf("Close: %v", err)
    }
  })
  return s
}
```

Now a test is one line — `s := newStore(t)` — and teardown is guaranteed. The properties that make
this work:

- **It is bound to the test, not the enclosing function**, so helpers can register their own
  teardown. This is the whole reason the API exists.
- **It runs even after `t.Fatal`**, because `Fatal` is `runtime.Goexit`, not `os.Exit`.
- **It composes.** Three helpers registering three cleanups unwind in reverse call order.
- **It runs after subtests finish**, including parallel ones — which `defer` in the parent does not.
- A cleanup may call `t.Error` to fail the test. It may not call `t.Parallel`.

**Idiomatic:** `t.Cleanup` in helpers and anywhere subtests are involved; plain `defer` is fine for
something opened and closed inside one test function.

## t.Helper and the line number

This is the single most useful two-word addition to a test file, and the reason is entirely in the
failure output.

```go
package p

import "testing"

func assertEqNoHelper(t *testing.T, got, want string) {
  if got != want {
    t.Errorf("got %q, want %q", got, want)
  }
}

func assertEqHelper(t *testing.T, got, want string) {
  t.Helper()
  if got != want {
    t.Errorf("got %q, want %q", got, want)
  }
}

func TestWithoutHelper(t *testing.T) {
  assertEqNoHelper(t, Normalize("  GO  "), "golang")
}

func TestWithHelper(t *testing.T) {
  assertEqHelper(t, Normalize("  GO  "), "golang")
}
```

The two functions are identical except for `t.Helper()`. Both tests fail on purpose.
`go test -count=1 -v .`. Output:

```
=== RUN   TestWithoutHelper
    help_test.go:7: got "go", want "golang"
--- FAIL: TestWithoutHelper (0.00s)
=== RUN   TestWithHelper
    help_test.go:23: got "go", want "golang"
--- FAIL: TestWithHelper (0.00s)
FAIL
FAIL	example.com/scratch	0.494s
FAIL
```

That is the entire point. **Line 7** is the `t.Errorf` inside the helper — the same line for every
failure in the file, telling you nothing about which assertion broke. **Line 23** is the call site
in `TestWithHelper`, which is where the bug is.

With a helper called from twenty places, the no-`Helper` version reports line 7 twenty times and you
cannot tell them apart. With `t.Helper()` each failure points at its own caller, and your editor
jumps straight there.

Rules:

- Call it as the **first statement**. It marks the function as a helper for the rest of the call.
- It affects only line reporting. It does not change failure semantics.
- It applies to the whole chain: if helper A (marked) calls helper B (marked), the reported line is
  A's caller.
- Mark **every** test helper that can report a failure — assertion helpers, setup helpers that
  `t.Fatal`, `recover` wrappers.
- Take `t testing.TB` rather than `*testing.T` if the helper should also serve benchmarks and fuzz
  targets; `Helper()` is on `TB`.

```go
func mustJSON(t testing.TB, v any) []byte {
  t.Helper()
  b, err := json.Marshal(v)
  if err != nil {
    t.Fatalf("Marshal(%#v): %v", v, err)
  }
  return b
}
```

## t.TempDir

Returns a fresh directory, removed automatically when the test and its subtests finish. No
`MkdirTemp`, no `RemoveAll`, no error to check.

```go
func TestTempDir(t *testing.T) {
  dir := t.TempDir()
  t.Logf("TempDir = %s", dir)
  f := filepath.Join(dir, "data.txt")
  if err := os.WriteFile(f, []byte("hello"), 0o600); err != nil {
    t.Fatalf("WriteFile: %v", err)
  }
  t.Run("sub", func(t *testing.T) {
    t.Logf("subtest TempDir = %s", t.TempDir())
  })
}

func TestTempDirGone(t *testing.T) {
  var saved string
  t.Run("inner", func(t *testing.T) {
    saved = t.TempDir()
    t.Logf("inner dir = %s", saved)
  })
  if _, err := os.Stat(saved); os.IsNotExist(err) {
    t.Logf("after the subtest, %s is removed", saved)
  } else {
    t.Errorf("expected removal, got err=%v", err)
  }
}
```

`go test -count=1 -v .` on darwin — the path shape is OS-specific and the random digits differ every
run. Output:

```
=== RUN   TestTempDir
    clean_test.go:11: TempDir = /var/folders/3l/gf781gh95p75bmw55xppgfhh0000gn/T/TestTempDir834185835/001
=== RUN   TestTempDir/sub
    clean_test.go:17: subtest TempDir = /var/folders/3l/gf781gh95p75bmw55xppgfhh0000gn/T/TestTempDirsub982806333/001
--- PASS: TestTempDir (0.00s)
    --- PASS: TestTempDir/sub (0.00s)
=== RUN   TestTempDirGone
=== RUN   TestTempDirGone/inner
    clean_test.go:25: inner dir = /var/folders/3l/gf781gh95p75bmw55xppgfhh0000gn/T/TestTempDirGoneinner3596895887/001
=== NAME  TestTempDirGone
    clean_test.go:28: after the subtest, /var/folders/3l/gf781gh95p75bmw55xppgfhh0000gn/T/TestTempDirGoneinner3596895887/001 is removed
--- PASS: TestTempDirGone (0.00s)
    --- PASS: TestTempDirGone/inner (0.00s)
PASS
ok  	example.com/scratch	0.509s
```

Read the path: `<os temp>/<TestName><random>/001`. The test name is embedded (with `/` stripped, so
the subtest became `TestTempDirsub`), which is a real debugging aid when you keep a directory around
to inspect.

- **Per `*testing.T`, not per test function.** A subtest calling `t.TempDir()` gets its *own*
  directory, and it is removed when that subtest ends — as the second test proves.
- Calling it twice in the same test gives two directories: `001`, `002`.
- Removal failure fails the test, which catches a file you left open on Windows.
- It calls `t.Fatal` itself if creation fails, so there is no error return.

## t.Setenv, and why it forbids t.Parallel

`t.Setenv(k, v)` sets an environment variable and restores the previous state via `t.Cleanup`.

```go
package conf

import (
  "os"
  "testing"
)

func TestRegionDefault(t *testing.T) {
  if got := Region(); got != "us-east-1" {
    t.Errorf("Region() = %q, want %q", got, "us-east-1")
  }
}

func TestRegionFromEnv(t *testing.T) {
  t.Setenv("APP_REGION", "eu-west-2")
  if got := Region(); got != "eu-west-2" {
    t.Errorf("Region() = %q, want %q", got, "eu-west-2")
  }
}

func TestRegionRestored(t *testing.T) {
  if v, ok := os.LookupEnv("APP_REGION"); ok {
    t.Errorf("APP_REGION leaked from the previous test: %q", v)
  } else {
    t.Log("APP_REGION is unset again")
  }
}

func TestSetenvParallel(t *testing.T) {
  t.Parallel()
  t.Setenv("APP_REGION", "ap-south-1")
}
```

The environment is **process-global**, so a parallel test changing it would corrupt every other
test running at that moment. The testing package refuses rather than letting you write a flaky
suite. `TestSetenvParallel` panics on purpose. `go test -count=1 -v .`. Output:

```
=== RUN   TestRegionDefault
--- PASS: TestRegionDefault (0.00s)
=== RUN   TestRegionFromEnv
--- PASS: TestRegionFromEnv (0.00s)
=== RUN   TestRegionRestored
    env_test.go:25: APP_REGION is unset again
--- PASS: TestRegionRestored (0.00s)
=== RUN   TestSetenvParallel
=== PAUSE TestSetenvParallel
=== CONT  TestSetenvParallel
--- FAIL: TestSetenvParallel (0.00s)
panic: testing: test using t.Setenv, t.Chdir, or cryptotest.SetGlobalRandom can not use t.Parallel [recovered, repanicked]

goroutine 38 [running]:
testing.tRunner.func1.2({0x102dff9c0, 0x102dd4268})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:2123 +0x1a0
testing.tRunner.func1()
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:2126 +0x2c8
panic({0x102dff9c0?, 0x102dd4268?})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/panic.go:859 +0x120
testing.(*T).checkParallel(...)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:1992
testing.(*T).Setenv(0x572e195d0908?, {0x102cb393a?, 0x572e19589758?}, {0x102cb3944?, 0x572e19589758?})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:2006 +0x74
example.com/scratch.TestSetenvParallel(0x572e195d0908)
	/tmp/scratch/env_test.go:31 +0x40
testing.tRunner(0x572e195d0908, 0x102e1f6f0)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:2193 +0xc4
created by testing.(*T).Run in goroutine 1
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:2258 +0x3b8
FAIL	example.com/scratch	0.494s
FAIL
```

`test using t.Setenv, t.Chdir, or cryptotest.SetGlobalRandom can not use t.Parallel` — and the ban
covers **parallel ancestors** too, so a subtest cannot call `t.Setenv` if any parent called
`t.Parallel`. `TestRegionRestored` passing proves the restore works: the variable set by the
previous test is unset again.

If you need both parallelism and configuration, do not use the environment. Pass config as a
parameter or a struct field — which is the design the constraint is nudging you toward
([18.2](../18-production/02-config.md)).

## t.Chdir

Changes the working directory for the duration of the test, restoring it afterwards, and on Unix
also sets `PWD`.

```go
func TestChdir(t *testing.T) {
  before, err := os.Getwd()
  if err != nil {
    t.Fatalf("Getwd: %v", err)
  }
  t.Cleanup(func() {
    after, _ := os.Getwd()
    t.Logf("after the test, back to the original dir: %v", after == before)
  })

  dir := t.TempDir()
  t.Chdir(dir)

  if err := os.WriteFile("local.txt", []byte("hi"), 0o600); err != nil {
    t.Fatalf("WriteFile: %v", err)
  }
  now, err := os.Getwd()
  if err != nil {
    t.Fatalf("Getwd: %v", err)
  }
  t.Logf("inside the test, cwd basename = %s", filepath.Base(now))
  t.Logf("PWD env basename = %s", filepath.Base(os.Getenv("PWD")))
}
```

`go test -count=1 -v .`. Output:

```
=== RUN   TestChdir
    chdir_test.go:29: inside the test, cwd basename = 001
    chdir_test.go:30: PWD env basename = 001
    chdir_test.go:16: after the test, back to the original dir: true
--- PASS: TestChdir (0.00s)
PASS
ok  	example.com/scratch	0.780s
```

The relative `os.WriteFile("local.txt", ...)` landed in the temp dir, and the cleanup confirms the
restore. Like `t.Setenv`, `t.Chdir` is process-global so it **cannot be used with `t.Parallel`** —
the same panic.

One subtlety the output above demonstrates. The verification cleanup is registered **before**
`t.Chdir`, and that is deliberate: cleanups run LIFO, so registering it *after* would make it run
*before* `t.Chdir`'s own restore, and it would report `false`. When you need to observe state after
another mechanism's cleanup, register yours first.

## TestMain

`func TestMain(m *testing.M)` replaces the default test entry point for the whole package. It runs
once, sets up, calls `m.Run()`, tears down, and exits with the code `m.Run()` returned.

```go
package store

import (
  "errors"
  "fmt"
  "io/fs"
  "os"
  "testing"
)

var shared *Store

func TestMain(m *testing.M) {
  dir, err := os.MkdirTemp("", "storetest")
  if err != nil {
    fmt.Fprintf(os.Stderr, "setup: %v\n", err)
    os.Exit(1)
  }
  fmt.Println("TestMain: created", "<tempdir>")

  shared, err = Open(dir)
  if err != nil {
    os.RemoveAll(dir)
    fmt.Fprintf(os.Stderr, "setup: %v\n", err)
    os.Exit(1)
  }

  code := m.Run()

  fmt.Println("TestMain: tearing down")
  if err := os.RemoveAll(dir); err != nil {
    fmt.Fprintf(os.Stderr, "teardown: %v\n", err)
  }
  os.Exit(code)
}

func TestPutGet(t *testing.T) {
  type user struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
  }
  if err := shared.Put("u1", user{"Ada", 36}); err != nil {
    t.Fatalf("Put: %v", err)
  }
  var got user
  if err := shared.Get("u1", &got); err != nil {
    t.Fatalf("Get: %v", err)
  }
  if got.Name != "Ada" || got.Age != 36 {
    t.Errorf("Get() = %+v, want {Name:Ada Age:36}", got)
  }
}

func TestGetMissing(t *testing.T) {
  var got map[string]any
  err := shared.Get("nope", &got)
  if !errors.Is(err, fs.ErrNotExist) {
    t.Errorf("Get(nope) error = %v, want fs.ErrNotExist", err)
  }
}
```

`go test -count=1 -v .`. Output:

```
TestMain: created <tempdir>
=== RUN   TestPutGet
--- PASS: TestPutGet (0.00s)
=== RUN   TestGetMissing
--- PASS: TestGetMissing (0.00s)
PASS
TestMain: tearing down
ok  	example.com/scratch	0.500s
```

The setup line prints before any test and the teardown line after `PASS`, which is the proof it
bracketed the run. `fmt.Println` rather than `t.Log`, because there is no `*testing.T` here.

The rules, and the one that bites:

- **`os.Exit` does not run deferred functions.** So `defer os.RemoveAll(dir)` in `TestMain` is a
  silent leak. Teardown must happen *between* `m.Run()` and `os.Exit`, exactly as above.
- You must call `os.Exit(code)` (or let `TestMain` return, which the testing package now treats as
  `os.Exit(m.Run()'s code)` only if you never called `m.Run()` — do not rely on that; be explicit).
- Returning without exiting on a non-zero code makes a failing suite report success.
- One `TestMain` per package, maximum. It is a build error to have two.
- `flag.Parse()` is called by `m.Run()`; if you need flags *before* that, call `flag.Parse()`
  yourself first.
- A `TestMain` that fails setup should `os.Exit(1)` after cleaning up what it created.

**Use it for one thing: a resource genuinely shared by the whole package** — a database connection
pool, a spun-up test server, a loaded dictionary. Everything per-test belongs in a helper with
`t.Cleanup`, which is cheaper to reason about and keeps tests independent.

A safer shape that gets deferred cleanup back by wrapping the body in a function:

```go
func TestMain(m *testing.M) {
  os.Exit(run(m))
}

func run(m *testing.M) int {
  dir, err := os.MkdirTemp("", "storetest")
  if err != nil {
    fmt.Fprintf(os.Stderr, "setup: %v\n", err)
    return 1
  }
  defer os.RemoveAll(dir)

  shared, err = Open(dir)
  if err != nil {
    fmt.Fprintf(os.Stderr, "setup: %v\n", err)
    return 1
  }
  return m.Run()
}
```

Now `defer` works, because `os.Exit` happens in the caller after `run` has returned.

## Build-tag-gated integration tests

A test that needs a real database or network should not run — or even compile — by default. A build
constraint at the top of the file does both.

```go
//go:build integration

package report

import "testing"

func TestAgainstRealService(t *testing.T) {
  t.Log("talking to the real service")
  rows := []Row{{"live", 1}}
  if got := Render("Live", rows); got == "" {
    t.Error("Render returned empty")
  }
}
```

The `//go:build integration` line must be at the very top, followed by a **blank line** before
`package`. Default run, `go test -count=1 -v .`. Output:

```
=== RUN   TestRender
--- PASS: TestRender (0.00s)
PASS
ok  	example.com/scratch	0.201s
```

The integration test is absent — not skipped, *not compiled*. `go test -list '.*' .` confirms it
does not exist as far as the toolchain is concerned:

```
TestRender
ok  	example.com/scratch	0.195s
```

Now `go test -count=1 -v -tags integration .`. Output:

```
=== RUN   TestRender
--- PASS: TestRender (0.00s)
=== RUN   TestAgainstRealService
    integration_test.go:8: talking to the real service
--- PASS: TestAgainstRealService (0.00s)
PASS
ok  	example.com/scratch	0.501s
```

| Gate | Mechanism | Compiled by default? | Visible to `-list`? |
|---|---|---|---|
| `//go:build integration` | build constraint | no | no |
| `if testing.Short() { t.Skip() }` | runtime check | yes | yes |
| `if os.Getenv("DB_URL") == "" { t.Skip() }` | runtime check | yes | yes |

Build tags are right when the file imports heavy test-only dependencies or must not compile without
them; a runtime skip is right when you want the test visible and self-documenting. One trap:
**tagged files are invisible to `gofmt -l`-style checks only if your tooling forgets the tag** —
run `go vet -tags integration ./...` in CI too, or the file silently rots. See
[15.4](../15-tooling/04-build-tags.md) for the full constraint syntax.

## testdata/ and golden files

`testdata` is special to the go tool: it **ignores any directory named `testdata`** when matching
packages, so you can put arbitrary files there — including `.go` files that would not compile —
without affecting the build. Paths in a test are relative to the package directory, because `go
test` runs the binary with that as its working directory. So `testdata/input.json` just works.

A golden file test compares output against a checked-in file, with a flag to regenerate it. The
renderer under test:

```go
package report

import (
  "fmt"
  "strings"
)

type Row struct {
  Name  string
  Count int
}

func Render(title string, rows []Row) string {
  var b strings.Builder
  fmt.Fprintf(&b, "%s\n", title)
  fmt.Fprintf(&b, "%s\n", strings.Repeat("=", len(title)))
  total := 0
  for _, r := range rows {
    fmt.Fprintf(&b, "%-10s %4d\n", r.Name, r.Count)
    total += r.Count
  }
  fmt.Fprintf(&b, "%-10s %4d\n", "TOTAL", total)
  return b.String()
}
```

The test, with the `-update` flag:

```go
package report

import (
  "flag"
  "os"
  "path/filepath"
  "testing"
)

var update = flag.Bool("update", false, "rewrite the golden files")

func golden(t *testing.T, name, got string) {
  t.Helper()
  path := filepath.Join("testdata", name)
  if *update {
    if err := os.WriteFile(path, []byte(got), 0o600); err != nil {
      t.Fatalf("writing golden %s: %v", path, err)
    }
    t.Logf("updated %s", path)
    return
  }
  want, err := os.ReadFile(path)
  if err != nil {
    t.Fatalf("reading golden %s: %v (run with -update to create it)", path, err)
  }
  if got != string(want) {
    t.Errorf("output does not match %s\n--- got ---\n%s--- want ---\n%s", path, got, want)
  }
}

func TestRender(t *testing.T) {
  rows := []Row{{"alpha", 3}, {"beta", 14}, {"gamma", 1}}
  golden(t, "render.golden", Render("Daily Counts", rows))
}
```

A top-level `flag.Bool` in a test file works because `m.Run()` calls `flag.Parse()`, and `go test`
forwards unknown flags to the test binary.

**Step 1**, with no golden file yet — fails on purpose and tells you what to do.
`go test -count=1 -v .`. Output:

```
=== RUN   TestRender
    golden_test.go:33: reading golden testdata/render.golden: open testdata/render.golden: no such file or directory (run with -update to create it)
--- FAIL: TestRender (0.00s)
FAIL
FAIL	example.com/scratch	0.508s
FAIL
```

**Step 2**, generate it. `go test -count=1 -v -update .`. Output:

```
=== RUN   TestRender
    golden_test.go:33: updated testdata/render.golden
--- PASS: TestRender (0.00s)
PASS
ok  	example.com/scratch	0.197s
```

Both messages report line 33 — the `golden(t, ...)` call site, thanks to `t.Helper()`.

**Step 3**, the generated `testdata/render.golden`:

```
Daily Counts
============
alpha         3
beta         14
gamma         1
TOTAL        18
```

**Step 4**, a normal run now passes:

```
=== RUN   TestRender
--- PASS: TestRender (0.00s)
PASS
ok  	example.com/scratch	0.193s
```

**Step 5**, change the code — `"TOTAL"` becomes `"SUM"` — and the golden file catches it. Fails on
purpose:

```
=== RUN   TestRender
    golden_test.go:33: output does not match testdata/render.golden
        --- got ---
        Daily Counts
        ============
        alpha         3
        beta         14
        gamma         1
        SUM          18
        --- want ---
        Daily Counts
        ============
        alpha         3
        beta         14
        gamma         1
        TOTAL        18
--- FAIL: TestRender (0.00s)
FAIL
FAIL	example.com/scratch	0.503s
FAIL
```

**Step 6**, if the change was intended, `go test -count=1 -update .` rewrites the file and the diff
shows up in your next `git diff` for review.

That review step is the discipline that makes golden files worth having, and the danger if you skip
it: `-update` can rubber-stamp a regression. Keep the golden file small enough to read in a diff,
and never run `-update` without looking at what changed.

Golden files earn their keep for large formatted output — rendered templates, generated code,
pretty-printed JSON, CLI help text — where hand-writing `want` is impractical. For a three-field
struct, write the expectation inline.

## What is cleaned up for you

| Mechanism | Cleaned up automatically | Scope |
|---|---|---|
| `t.TempDir()` | yes — directory removed | per `*testing.T` (incl. subtests) |
| `t.Setenv(k, v)` | yes — previous value or unset restored | per test; forbids `t.Parallel` |
| `t.Chdir(dir)` | yes — original cwd and `PWD` restored | per test; forbids `t.Parallel` |
| `t.Cleanup(f)` | runs `f`, LIFO, after subtests | per `*testing.T` |
| `os.MkdirTemp` | **no** — you must `RemoveAll` | manual |
| `os.Setenv` | **no** — leaks into later tests | manual |
| `os.Create` / `os.Open` | **no** — you must `Close` | manual |
| a `net.Listener` you opened | **no** | manual |
| `httptest.NewServer` | no — call `srv.Close()`, usually via `t.Cleanup` | manual |
| package-level `var` you mutated | **no** — the classic flaky-test cause | manual |

The right-hand column is the list of things to wrap in a helper with `t.Cleanup`.

## Parallel isolation hazards

`t.Parallel()` turns every shared thing into a potential race. The four that actually cause trouble:

**Shared global state.** A package-level `var` written by two parallel tests is a data race, and
`go test -race` will say so ([11.5](../11-sync-context/05-race-detector.md)). Make the state
per-test and inject it ([14.8](08-test-doubles.md)).

**Shared files.** Two parallel tests writing `testdata/out.txt` interleave. `t.TempDir()` per test
fixes it, because each gets a distinct directory.

**`t.Setenv` and `t.Chdir`.** Process-global; the testing package panics rather than allow it. If a
parallel test needs a different working directory, open files by absolute path instead.

**Shared ports.** Two tests both binding `:8080` — one fails. Use `httptest.NewServer`, or listen on
`:0` and read back the assigned port ([16.11](../16-http/11-httptest.md)).

And the ordering rule from [14.2](02-table-driven.md), restated because it is the subtle one: the
parent test's serial body **returns before** parallel subtests run. So this is broken:

```go
func TestBroken(t *testing.T) {
  srv := httptest.NewServer(h)
  defer srv.Close()
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      t.Parallel()
      resp, err := http.Get(srv.URL)
      ...
    })
  }
}
```

`srv.Close()` runs when `TestBroken` returns, which is *before* any subtest body executes. Replace
`defer srv.Close()` with `t.Cleanup(srv.Close)` and it works, because cleanups wait for subtests.

## -shuffle catches order dependence

Tests must be independent. `-shuffle` is how you find out whether yours are. A cache with
package-level state, and two tests where the second depends on the first:

```go
package cache

var entries = map[string]string{}

func Set(k, v string) { entries[k] = v }

func Get(k string) (string, bool) {
  v, ok := entries[k]
  return v, ok
}
```

```go
package cache

import "testing"

func TestASet(t *testing.T) {
  Set("k", "v")
  if got, ok := Get("k"); !ok || got != "v" {
    t.Errorf("Get(k) = %q, %v, want %q, true", got, ok, "v")
  }
}

func TestBGet(t *testing.T) {
  got, ok := Get("k")
  if !ok || got != "v" {
    t.Errorf("Get(k) = %q, %v, want %q, true", got, ok, "v")
  }
}
```

In file order it passes, which is exactly why this bug survives in real suites.
`go test -count=1 -v .`. Output:

```
=== RUN   TestASet
--- PASS: TestASet (0.00s)
=== RUN   TestBGet
--- PASS: TestBGet (0.00s)
PASS
ok  	example.com/scratch	0.453s
```

Now `go test -count=1 -shuffle=on .` — the seed differs every run, and so may the outcome. A real
run. Output:

```
-test.shuffle 1793001498917629000
--- FAIL: TestBGet (0.00s)
    cache_test.go:15: Get(k) = "", false, want "v", true
FAIL
FAIL	example.com/scratch	0.189s
FAIL
```

`TestBGet` ran first and the cache was empty. The seed on the first line is the point: pass it back
to reproduce the exact order.

`go test -count=1 -v -shuffle=1793001498917629000 .`. Output:

```
-test.shuffle 1793001498917629000
=== RUN   TestBGet
    cache_test.go:15: Get(k) = "", false, want "v", true
--- FAIL: TestBGet (0.00s)
=== RUN   TestASet
--- PASS: TestASet (0.00s)
FAIL
FAIL	example.com/scratch	0.201s
FAIL
```

Deterministic for that seed, so you can iterate on the fix. `-shuffle` randomizes top-level tests
and benchmarks within each package; it does not reorder subtests.

The fix is never "make the ordering explicit". It is to give each test its own state:

```go
func TestBGet(t *testing.T) {
  Set("k", "v")
  ...
}
```

or better, drop the package-level map and have the test construct its own cache. **The rule: any
test must pass alone, in any order, and in parallel.** `go test -count=1 -shuffle=on -race ./...`
is the single command that checks all three.

## Quick Reference

| Call | Effect |
|---|---|
| `t.Cleanup(f)` | run `f` after the test and its subtests; LIFO; survives `Fatal` |
| `t.Helper()` | report failures at the caller's line, not this function's |
| `t.TempDir()` | fresh auto-removed directory, per `*testing.T` |
| `t.Setenv(k, v)` | set + auto-restore an env var; forbids `t.Parallel` |
| `t.Chdir(dir)` | change + auto-restore cwd and `PWD`; forbids `t.Parallel` |
| `t.Name()` | the test's full name, usable in fixture naming |
| `TestMain(m *testing.M)` | package-wide setup/teardown around `m.Run()` |
| `testing.TB` | the interface for helpers shared by `T`, `B` and `F` |

| Command | Effect |
|---|---|
| `go test -tags integration ./...` | compile and run build-tag-gated tests |
| `go test -update ./...` | your own flag, by convention, to rewrite golden files |
| `go test -shuffle=on ./...` | randomize order; prints the seed |
| `go test -shuffle=<seed> ./...` | reproduce one shuffled order |
| `go test -count=1 -shuffle=on -race ./...` | independence, order and race safety in one go |
| `go vet -tags integration ./...` | keep tagged files from rotting |

| Path convention | Meaning |
|---|---|
| `testdata/` | ignored by the go tool; holds fixtures |
| `testdata/*.golden` | expected-output files |
| cwd during a test | the package's own directory |

## Common Mistakes

**Using `defer` for teardown inside a setup helper.** It fires when the helper returns, destroying
the fixture before the test uses it — register `t.Cleanup` instead.

**Forgetting `t.Helper()` in an assertion helper.** Every failure then reports the same line inside
the helper, so you cannot tell which of twenty call sites broke.

**`defer os.RemoveAll(dir)` in `TestMain`.** `os.Exit` does not run deferred functions; clean up
between `m.Run()` and `os.Exit`, or wrap the body in a function that returns the code.

**Returning from `TestMain` without `os.Exit(code)`.** A failing suite can report success.

**`defer srv.Close()` in a parent whose subtests call `t.Parallel()`.** The parent returns before
the children run; use `t.Cleanup(srv.Close)`.

**Calling `os.Setenv` in a test.** It leaks into every test that runs afterwards; `t.Setenv`
restores it and refuses to let you combine it with `t.Parallel`.

**Registering a cleanup that observes another cleanup's effect, after it.** Cleanups are LIFO —
register yours *before* the mechanism whose teardown you want to observe.

**Relying on test order.** It passes locally and fails under `-shuffle` or when someone runs
`-run` on a single test; give each test its own state.

## Exercises

1. Write a `newTempStore(t *testing.T) *Store` helper that creates a temp directory, opens a store,
   and registers cleanup. Confirm with `t.Logf` that the directory is gone after the test, and that
   it still gets cleaned up when the test calls `t.Fatal`.
2. Build an assertion helper and call it from five places in one test. Capture the failure output
   with and without `t.Helper()`, and count how many distinct line numbers each version reports.
3. Write a test that calls `t.Setenv` and `t.Parallel`, capture the exact panic, then restructure
   the code under test so the configuration is injected and the test can be parallel.
4. Add a golden-file test for a function that renders JSON. Run the full cycle: fail with no golden,
   `-update` to create, change the code, see the diff, `-update` again. Then make the helper print a
   line-based diff instead of both whole files.
5. Write two tests that share a package-level variable and pass in file order. Find a `-shuffle`
   seed that fails, then fix the tests and confirm the same seed passes.
6. Gate a test behind `//go:build integration`. Prove with `-list` that it does not exist without
   the tag, then introduce a compile error in it and show that `go test ./...` still passes while
   `go vet -tags integration ./...` catches it.
7. Convert a `TestMain` that uses `defer` for teardown into the `os.Exit(run(m))` shape, and
   demonstrate with a deliberately failing test that the exit code still propagates.

---

[← 14.2 Table-Driven Tests and Subtests](02-table-driven.md) · [Course Index](../README.md) · [14.4 testify, Briefly →](04-testify.md)
