[← 15.2 go mod tidy, vendor, and Dependency Management](02-mod-tidy-vendor.md) · [Course Index](../README.md) · [15.4 Build Tags and Conditional Compilation →](04-build-tags.md)

# 15.3 — go vet, staticcheck, and golangci-lint

## What You'll Learn

- All 36 analyzers `go vet` ships, and ten of them firing on real code
- `shadow` is **not** built in; it needs `-vettool` from `x/tools`
- `go test` fails the build on a vet subset, and how to turn that off
- Where vet stops: unchecked errors need errcheck, which needs golangci-lint

1.5 introduced `go vet` and showed six findings. This lesson is the full
analyzer inventory, the three tools that sit above vet, and which of them
belong in CI versus on your keyboard.

## Coming From Another Language

| Other | Go |
|---|---|
| ESLint: style + bugs + 200 plugins | `gofmt` is style, `go vet` is bugs, and they never overlap |
| `.eslintrc` is mandatory configuration | `go vet` has no config file; the analyzer set is fixed |
| `tsc --strict` catches unchecked results | nothing in the toolchain catches an ignored `error`; that is errcheck |
| `clippy` ships with rustup | `staticcheck` is third-party and lags each Go release |
| `ruff`/`mypy` need version pinning per project | `go vet` is pinned to your compiler by construction |
| `pytest` runs tests only | `go test` runs a vet subset first and fails on it |

## The analyzer inventory

```bash
go tool vet help
```

Output (the preamble, then all 36 registered analyzers):

```
vet is a tool for static analysis of Go programs.

vet examines Go source code and reports diagnostics for
suspicious constructs or opportunities for improvement.
Diagnostics may include suggested fixes.

...

Registered analyzers:

    appends      check for missing values after append
    asmdecl      report mismatches between assembly files and Go declarations
    assign       check for useless assignments
    atomic       check for common mistakes using the sync/atomic package
    bools        check for common mistakes involving boolean operators
    buildtag     check //go:build and // +build directives
    cgocall      detect some violations of the cgo pointer passing rules
    composites   check for unkeyed composite literals
    copylocks    check for locks erroneously passed by value
    defers       report common mistakes in defer statements
    directive    check Go toolchain directives such as //go:debug
    errorsas     report passing non-pointer or non-error values to errors.As
    framepointer report assembly that clobbers the frame pointer before saving it
    hostport     check format of addresses passed to net.Dial
    httpresponse check for mistakes using HTTP responses
    ifaceassert  detect impossible interface-to-interface type assertions
    loopclosure  check references to loop variables from within nested functions
    lostcancel   check cancel func returned by context.WithCancel is called
    nilfunc      check for useless comparisons between functions and nil
    printf       check consistency of Printf format strings and arguments
    shift        check for shifts that equal or exceed the width of the integer
    sigchanyzer  check for unbuffered channel of os.Signal
    slog         check for invalid structured logging calls
    stdmethods   check signature of methods of well-known interfaces
    stdversion   report uses of too-new standard library symbols
    stringintconv check for string(int) conversions
    structtag    check that struct field tags conform to reflect.StructTag.Get
    testinggoroutine report calls to (*testing.T).Fatal from goroutines started by a test
    tests        check for common mistaken usages of tests and examples
    timeformat   check for calls of (time.Time).Format or time.Parse with 2006-02-01
    unmarshal    report passing non-pointer or non-interface values to unmarshal
    unreachable  check for unreachable code
    unsafeptr    check for invalid conversions of uintptr to unsafe.Pointer
    unusedresult check for unused results of calls to some functions
    waitgroup    check for misuses of sync.WaitGroup
```

Note the preamble's distinction, new since Go 1.26: analyzers of the *first*
kind report suspicious constructs and run under `go vet`; analyzers of the
*second* kind report opportunities with machine-applicable fixes and run under
`go fix -fixtool=`. The `strings.SplitSeq` suggestion 1.5 showed is the second
kind. Six of the 36 above (`asmdecl`, `cgocall`, `framepointer`, `unsafeptr`,
`buildtag`, `directive`) only fire on assembly, cgo, `unsafe`, or directive
comments.

## Ten analyzers on one file

This program compiles cleanly. Every line in it is a real bug:

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "net/http"
  "sync"
  "sync/atomic"
  "time"
)

type Config struct {
  Host string `json:"host`
  Port int    `json:"port"`
}

type Counter struct {
  mu sync.Mutex
  n  int64
}

func (c Counter) Inc() {
  c.mu.Lock()
  defer c.mu.Unlock()
  c.n++
}

type MyErr struct{ Code int }

func (e *MyErr) Error() string { return fmt.Sprint(e.Code) }

func work(ctx context.Context) error {
  ctx, _ = context.WithTimeout(ctx, time.Millisecond)
  <-ctx.Done()
  return ctx.Err()
}

func fetch(url string) error {
  resp, err := http.Get(url)
  defer resp.Body.Close()
  if err != nil {
    return err
  }
  return nil
}

func main() {
  age := 36
  fmt.Printf("%s is %d\n", age, "ada")

  var cnt int64
  cnt = atomic.AddInt64(&cnt, 1)

  if work == nil {
    fmt.Println("never")
  }

  var target MyErr
  errors.As(errors.New("x"), &target)

  fmt.Errorf("lost: %w", errors.New("inner"))

  fmt.Println(string(65), Config{}, cnt)
  fmt.Println(work(context.Background()), fetch("http://x"))

  var c Counter
  c.Inc()
}
```

```bash
go build ./...   # succeeds, silently
go vet .
```

Output (analyzers run in parallel, so the line order varies between runs —
match by file position):

```
main.go:60:3: second argument to errors.As must be a non-nil pointer to either a type that implements error, or to any interface type
main.go:41:9: using resp before checking for errors
main.go:55:6: comparison of function work == nil is always false
main.go:50:15: fmt.Printf format %s has arg age of wrong type int
main.go:53:3: direct assignment to atomic value
main.go:34:8: the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
main.go:64:15: conversion from untyped int to string yields a string of one rune, not a string of digits
main.go:14:3: struct field tag `json:"host` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
main.go:62:3: result of fmt.Errorf call not used
main.go:23:9: Inc passes lock by value: example.com/scratch.Counter contains sync.Mutex
```

Which analyzer produced which, and why each one matters:

| Line | Analyzer | The actual bug |
|---|---|---|
| 14 | `structtag` | the unterminated quote means `encoding/json` ignores the tag and emits `Host`, silently changing your wire format |
| 23 | `copylocks` | value receiver, so `Inc` locks a *copy* of the mutex and increments a *copy* of `n` — the method does nothing |
| 34 | `lostcancel` | the discarded `cancel` leaks a timer and its goroutine until the deadline fires; in a request handler that is a per-request leak |
| 41 | `httpresponse` | `defer resp.Body.Close()` before the `err != nil` check panics on nil `resp` whenever the request fails |
| 50 | `printf` | `%s` against an `int` prints `%!s(int=36)`, `%d` against a string prints `%!d(string=ada)` |
| 53 | `atomic` | `cnt = atomic.AddInt64(&cnt, 1)` does a non-atomic store of the atomic result, reintroducing the race the atomic was there to prevent |
| 55 | `nilfunc` | a declared func is never nil; the comparison is dead code hiding a typo (you meant to call it, or compare a func *variable*) |
| 60 | `errorsas` | `&target` is `*MyErr` but `error` is implemented on `*MyErr`, so `errors.As` needs `**MyErr`; as written it panics at runtime |
| 62 | `unusedresult` | `fmt.Errorf` has no side effects; the wrapped error was thrown away |
| 64 | `stringintconv` | `string(65)` is `"A"`, not `"65"` — the single most common Go-from-Python mistake; use `strconv.Itoa` |

Four more that need their own shapes:

```go
package main

import (
  "fmt"
  "sync"
  "testing"
)

type reader interface{ Read([]byte) (int, error) }
type badReader interface{ Read() string }

func impossible(r reader) {
  if b, ok := r.(badReader); ok {
    fmt.Println(b)
  }
}

func loopy() {
  var wg sync.WaitGroup
  for _, s := range []string{"a", "b"} {
    wg.Go(func() { fmt.Println(s) })
  }
  wg.Wait()
}

func fromGoroutine(t *testing.T) {
  go func() { t.Fatal("boom") }()
}

func main() {
  impossible(nil)
  loopy()
  fmt.Printf("%d\n")
  fmt.Printf("%d\n", 1, 2)
}
```

```bash
go vet .
```

Output:

```
main.go:27:15: call to (*testing.T).Fatal from a non-test goroutine
main.go:13:18: impossible type assertion: no type can implement both example.com/scratch.reader and example.com/scratch.badReader (conflicting types for Read method)
main.go:33:15: fmt.Printf format %d reads arg #1, but call has 0 args
main.go:34:3: fmt.Printf call needs 1 arg but has 2 args
```

- `ifaceassert` (line 13) — both interfaces declare `Read` with incompatible
  signatures, so the assertion can never succeed and the branch is unreachable.
- `testinggoroutine` (line 27) — `t.Fatal` calls `runtime.Goexit`, which from a
  non-test goroutine terminates *that* goroutine and leaves the test to pass.
- `printf` again for too few and too many arguments.

Note what did **not** fire: `loopclosure`. The closure captures `s` from the
range statement, which before Go 1.22 meant every goroutine saw the last value.
Since 1.22 each iteration gets a fresh variable, so the code is correct and
the analyzer correctly stays quiet. `loopclosure` reads the language version
from your `go.mod` — the same code under `go 1.21` is a bug and gets flagged:

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var wg sync.WaitGroup
  for _, s := range []string{"a", "b"} {
    wg.Add(1)
    go func() { defer wg.Done(); fmt.Println(s) }()
  }
  wg.Wait()
}
```

```bash
go mod edit -go=1.21 && go vet .
go mod edit -go=1.27 && go vet . ; echo "exit=$?"
```

Output:

```
main.go:12:46: loop variable s captured by func literal
exit=0
```

One analyzer, two verdicts, same source — driven entirely by the `go` line in
`go.mod` (15.2).

## Running one analyzer, or all but one

```bash
go vet -printf .
```

Output:

```
main.go:50:15: fmt.Printf format %s has arg age of wrong type int
```

Naming any analyzer switches to opt-in mode: only the named ones run.
To subtract instead, set one to false:

```bash
go vet -printf=false -copylocks .
```

Output:

```
main.go:23:9: Inc passes lock by value: example.com/scratch.Counter contains sync.Mutex
```

And across the module, which is the form you actually type:

```bash
go vet ./...
```

vet exits 1 when it reports anything, which is all a CI gate needs.

## shadow is not built in

The most-requested analyzer is not in the list above. This function returns the
wrong error:

```go
package main

import (
  "fmt"
  "os"
)

func shadowed() error {
  err := os.Remove("/tmp/nope")
  if err != nil {
    err := os.Remove("/tmp/other")
    _ = err
  }
  return err
}

func unchecked() {
  f, _ := os.Create("/tmp/scratch-demo.txt")
  f.WriteString("hello")
  f.Close()
  os.Remove("/tmp/scratch-demo.txt")
}

func main() {
  fmt.Println(shadowed())
  unchecked()
}
```

```bash
go vet .
echo "exit=$?"
```

Output:

```
exit=0
```

Nothing. The inner `err :=` on line 11 declares a *new* variable, the cleanup's
error is discarded, and `return err` returns the outer one. `shadow` lives in
`x/tools` and plugs in through `-vettool`:

```bash
go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest
go vet -vettool=$(go env GOPATH)/bin/shadow .
```

Output:

```
main.go:11:5: declaration of "err" shadows declaration at line 9
```

`-vettool=PROGRAM` replaces vet's analyzer set entirely with that program's, so
this run checked *only* shadowing. Any binary built with
`golang.org/x/tools/go/analysis/unitchecker` works this way — that is the plugin
interface, and it is how `golangci-lint` and `nilaway` hook in.

Shadowing is deliberately out of the default set because its false-positive
rate is high: `if err := f(); err != nil` inside a function that already has an
`err` is idiomatic Go, and `shadow` flags a chunk of it. Run it, read the
findings, do not gate on it.

## Where vet stops: unchecked errors

`unchecked()` above has three ignored errors — `f.WriteString`, `f.Close`, and
`os.Remove` — and vet said nothing. `unusedresult` only covers a short
hardcoded list of pure functions (`fmt.Errorf`, `fmt.Sprintf`, `errors.New`,
`sort.Reverse`, …), chosen so its false-positive rate stays at zero. Anything
that returns an `error` alongside a side effect is out of scope.

That gap is the entire reason to install a second tool. `errcheck` reports
exactly this, and it is the highest-value linter outside the toolchain:
an ignored `f.Close()` on a *written* file loses data silently, because the
flush happens in `Close`. Idiomatic Go for the deliberate case is an explicit
`_ =`, which errcheck accepts and a human reviewer can see:

```go
defer func() { _ = f.Close() }()
```

## go test runs vet for you

`go test` runs a high-confidence subset (`printf`, `tests`, `atomic`,
`bools`, `buildtag`, `errorsas`, `ifaceassert`, `nilfunc`, `stringintconv`) on
the package *and its test files* before compiling, and a finding is a build
failure, not a warning:

```go
package main

import "testing"

func TestDouble(t *testing.T) {
  if got := Double(3); got != 6 {
    t.Errorf("got %d want %d", "6", got)
  }
}
```

```bash
go test ./...
echo "exit=$?"
```

Output:

```
# example.com/scratch
# [example.com/scratch]
./main_test.go:7:19: (*testing.common).Errorf format %d has arg "6" of wrong type string
FAIL	example.com/scratch [build failed]
FAIL
exit=1
```

Note that vet knows `t.Errorf` is printf-like — the printf analyzer tracks
wrappers, including your own if you name the format parameter conventionally.
The two control flags:

```bash
go test -vet=off ./...
```

Output:

```
ok  	example.com/scratch	0.503s
```

```bash
go test -vet=all ./...
```

Output:

```
# example.com/scratch
# [example.com/scratch]
./main_test.go:7:19: (*testing.common).Errorf format %d has arg "6" of wrong type string
FAIL	example.com/scratch [build failed]
FAIL
```

`-vet=off` is for one specific situation: a vet false positive blocking an
urgent test run. It is not a CI setting.

## staticcheck

```bash
go install honnef.co/go/tools/cmd/staticcheck@latest
staticcheck ./...
```

staticcheck adds several hundred checks in four families:

| Family | Covers | Examples |
|---|---|---|
| `SA` | correctness — real bugs | `SA1019` deprecated API, `SA4006` value never read, `SA9003` empty branch, `SA1006` `Printf` with a dynamic format string, `SA6002` `sync.Pool` storing non-pointers |
| `S` (gosimple) | simplification | `S1000` `select` with one case → a plain receive, `S1021` merge declaration and assignment, `S1008` `if cond { return true }; return false` |
| `ST` (stylecheck) | style and naming | `ST1003` naming conventions, `ST1005` error strings should not be capitalised or end in punctuation, `ST1016` inconsistent receiver names |
| `QF` | quickfixes, machine-applicable | `QF1003` `if/else` chain → `switch`, `QF1012` `WriteString(fmt.Sprintf(...))` → `Fprintf` |

The `SA` family is the one worth gating on. `ST` and `QF` are opinions; enable
the handful you agree with and silence the rest.

Honest result in this environment — staticcheck **is** installed but cannot run
on Go 1.27:

```bash
staticcheck -version
staticcheck ./...
```

Output:

```
staticcheck 2025.1.1 (0.6.1)
-: internal error in importing "internal/cpu" (cannot decode "internal/cpu", export data version 4 is greater than maximum supported version 2); please report an issue (compile)
-: internal error in importing "internal/goarch" (cannot decode "internal/goarch", export data version 4 is greater than maximum supported version 2); please report an issue (compile)
-: internal error in importing "math/bits" (cannot decode "math/bits", export data version 4 is greater than maximum supported version 2); please report an issue (compile)
```

That is the standard third-party-analyzer problem, stated plainly. Every tool
that type-checks Go reads the compiler's **export data** — the serialised type
information in each compiled package — using a vendored copy of
`go/internal/gcimporter`. Go 1.27 bumped that format to version 4; staticcheck
2025.1.1 was built against a toolchain that knows up to version 2. There is no
flag for it. You wait for the next staticcheck release, or you pin your
toolchain.

So: **no staticcheck findings are shown in this lesson, because none were
produced.** I am not going to invent them. The lesson is the one the error
teaches: `go vet` ships with the compiler and therefore can never lag, which is
why it is the right thing to make a hard CI gate. staticcheck, golangci-lint,
and every other external analyzer are advisory tools that will break for a few
weeks after each February and August Go release. Plan for that: pin the linter
version in CI, pin the Go version alongside it, and bump them as a pair.

## golangci-lint

**Not installed in this environment.** No output below this heading was
executed here; it is the documented invocation and a configuration I would
actually ship.

```bash
go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.6.1
golangci-lint --version
```

golangci-lint is a *runner*: it loads each package once, hands the type-checked
AST to every enabled linter in parallel, and caches the results. Running
errcheck, staticcheck, and revive separately means three full type-check
passes; golangci-lint means one. That is the whole value proposition, and on a
large repo it is the difference between 20 seconds and three minutes.

A `.golangci.yml` worth starting from (v2 schema):

```yaml
version: "2"

linters:
  default: none
  enable:
    - errcheck
    - govet
    - staticcheck
    - unused
    - ineffassign
    - revive
    - gosec
  settings:
    errcheck:
      check-type-assertions: true
      check-blank: false
      exclude-functions:
        - (*bytes.Buffer).WriteString
        - (*strings.Builder).WriteString
    govet:
      enable-all: true
      disable:
        - fieldalignment
    staticcheck:
      checks:
        - all
        - -ST1000
        - -ST1003
        - -ST1020
        - -ST1021
    revive:
      rules:
        - name: exported
          disabled: true
        - name: unused-parameter
        - name: early-return
    gosec:
      excludes:
        - G104
  exclusions:
    rules:
      - path: _test\.go
        linters:
          - gosec
          - errcheck

formatters:
  enable:
    - gofmt
    - goimports
  settings:
    goimports:
      local-prefixes:
        - example.com/myapp

issues:
  max-issues-per-linter: 0
  max-same-issues: 0
```

What each enabled linter buys you:

| Linter | Catches |
|---|---|
| `errcheck` | unchecked `error` returns — the gap `go vet` leaves |
| `govet` | the 36 analyzers above, plus the non-default ones via `enable-all` |
| `staticcheck` | the SA/S/ST/QF families (gosimple and stylecheck are folded in since v2) |
| `unused` | unexported functions, types, fields, and constants nothing references |
| `ineffassign` | assignments whose value is never read |
| `revive` | configurable style rules; the maintained replacement for golint |
| `gosec` | hardcoded credentials, weak crypto, unsanitised SQL, `exec` with tainted input |

Two settings there matter more than they look. `default: none` makes the
enabled list exhaustive, so a golangci-lint upgrade cannot silently add linters
and break your build. And `fieldalignment` is disabled because it demands you
reorder struct fields by size for padding — technically correct, and it will
generate hundreds of findings nobody will act on.

The CI invocation:

```bash
golangci-lint run --timeout=5m
```

Common additions: `--new-from-rev=origin/main` reports only issues on lines you
changed, which is how you adopt the tool on an existing codebase without a
10,000-line cleanup PR; `--out-format=github-actions` gets inline PR
annotations; `--fix` applies what can be applied.

## gofmt and goimports as CI checks

`gofmt -l` prints misformatted filenames and **exits 0 either way**, so you
cannot use it directly as a gate:

```bash
gofmt -l .
echo "exit=$?"
```

Output:

```
bad.go
exit=0
```

The idiomatic wrapper turns "printed anything" into a non-zero exit:

```bash
gofmt -l . | tee /dev/stderr | (! read)
echo "exit=$?"
```

Output:

```
bad.go
exit=1
```

And with the file removed:

```bash
gofmt -l . | (! read)
echo "exit=$?"
```

Output:

```
exit=0
```

`tee /dev/stderr` is there so you see *which* files failed. The same pattern
works for `goimports -l .`, and since goimports is a strict superset of gofmt
(1.5), checking goimports alone is enough.

## What to actually run

| Tool | Locally | In CI | Gate? |
|---|---|---|---|
| `goimports -w` (or gopls) | on save | — | — |
| `goimports -l` / `gofmt -l` | — | yes | **hard** |
| `go build ./...` | — | yes | **hard** |
| `go vet ./...` | before commit | yes | **hard** |
| `go test ./... -race` | before commit | yes | **hard** |
| `go mod tidy -diff` | — | yes | **hard** |
| `staticcheck ./...` (SA only) | — | yes | hard, pinned version |
| `golangci-lint run` | occasionally | yes | hard on new code (`--new-from-rev`) |
| `shadow` via `-vettool` | occasionally | — | no |
| `govulncheck ./...` | — | yes | advisory, or hard on HIGH |

The four hard gates at the top need no third-party tools, no config file, and
no version pinning. Get those green first; everything below them is
incremental.

```bash
gofmt -l . | tee /dev/stderr | (! read)
go build ./...
go vet ./...
go mod tidy -diff
go test ./... -race
golangci-lint run --timeout=5m
```

## Quick Reference

| Command | Does |
|---|---|
| `go vet ./...` | all default analyzers, all packages |
| `go vet -printf ./...` | opt in to one analyzer only |
| `go vet -structtag=false ./...` | all except one |
| `go tool vet help` | the analyzer list |
| `go vet -vettool=BIN ./...` | replace vet's analyzers with BIN's |
| `go test ./...` | vet subset first; a finding fails the build |
| `go test -vet=off ./...` | skip the automatic vet pass |
| `go test -vet=all ./...` | run the full analyzer set on tests |
| `gofmt -l .` | list misformatted files; always exits 0 |
| `gofmt -l . \| (! read)` | the same as a non-zero gate |
| `goimports -l .` | the same, including import problems |
| `staticcheck ./...` | several hundred SA/S/ST/QF checks |
| `staticcheck -checks=SA ./...` | correctness family only |
| `golangci-lint run` | all enabled linters, one type-check pass |
| `golangci-lint run --new-from-rev=origin/main` | only issues on changed lines |
| `golangci-lint run --fix` | apply the applicable fixes |

| Install | |
|---|---|
| `go install golang.org/x/tools/cmd/goimports@latest` | goimports |
| `go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow@latest` | shadow vettool |
| `go install honnef.co/go/tools/cmd/staticcheck@latest` | staticcheck |
| `go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.6.1` | golangci-lint |
| `go install golang.org/x/vuln/cmd/govulncheck@latest` | govulncheck |

## Common Mistakes

**Assuming `go vet` catches unchecked errors.** `unusedresult` covers a short
list of pure functions; a discarded `f.Close()` passes silently and loses your
last buffered write. That is errcheck's job.

**Expecting `shadow` in the default set.** It is not registered; it needs
`-vettool` with the `x/tools` binary, and its false-positive rate is why.

**Setting `-vet=off` in CI because one finding was inconvenient.** You have
just disabled the only free bug-finder in the toolchain for the whole repo.

**Enabling every golangci-lint linter on an existing codebase.** You get
thousands of findings and the team disables the tool; start with
`default: none`, seven linters, and `--new-from-rev`.

**Pinning a Go version without pinning the linter, or vice versa.** External
analyzers read compiler export data and break on a format bump — the exact
`export data version 4 is greater than maximum supported version 2` failure
above. Bump the pair together.

**Using `gofmt -l .` as a gate without checking its output.** It exits 0 even
when it lists files; pipe it through `(! read)`.

## Exercises

1. Reproduce all ten findings from the first program, confirm `go build ./...`
   succeeds, then fix them one at a time and watch the list shrink. For each
   fix, name the analyzer.
2. Write a function where `errors.As` panics at runtime, capture the panic, then
   show that `go vet` predicted it.
3. Write a `printf`-style wrapper of your own (`func logf(format string, args
   ...any)`) with a mismatched call site. Determine experimentally what vet
   requires in order to recognise it as printf-like.
4. Install the `shadow` vettool and run it over a real project. Classify the
   first ten findings as real bugs or idiomatic `if err :=` shadowing, and
   decide whether you would gate on it.
5. Write a file where an ignored `Close()` on a buffered writer actually loses
   data. Prove the data loss, then show that `go vet` is silent about it.
6. Build the five-command CI gate from this lesson as a shell script that stops
   at the first failure, and verify each gate fires by breaking one thing at a
   time.
7. Take the `.golangci.yml` above, run it against a project you know, and move
   one linter from the enabled list to disabled with a one-sentence
   justification for the change.

---

[← 15.2 go mod tidy, vendor, and Dependency Management](02-mod-tidy-vendor.md) · [Course Index](../README.md) · [15.4 Build Tags and Conditional Compilation →](04-build-tags.md)
