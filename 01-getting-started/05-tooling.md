[← 1.4 Packages, Imports, and Visibility](04-packages-imports.md) · [Course Index](../README.md) · [1.6 How Go Thinks: A Mental Model →](06-mental-model.md)

# 1.5 — gofmt, go vet, and the Editor Setup

## What You'll Learn

- `gofmt` has no options and no config file; `gofmt -s` additionally simplifies code
- `goimports` is gofmt plus import management — the thing you actually bind to save
- `go vet` catches real bugs the compiler accepts, and `go test` runs a subset of it automatically
- `go doc` is the offline, authoritative reference for any package including your own

## Coming From Another Language

| Other | Go |
|---|---|
| `.prettierrc`, `.editorconfig`, 40 ESLint rules | zero config; `gofmt` is the format, full stop |
| Prettier is a dependency you install | `gofmt` ships with the compiler |
| `eslint --fix` mixes style and bugs | `gofmt` = style, `go vet` = bugs, strictly separate |
| `tsc --noEmit` as a separate typecheck pass | `go build` and `go vet` both typecheck |
| `mypy` catches `"%d" % "str"` sometimes | `go vet` catches Printf mismatches reliably |
| `npx jest` needs a runner config | `go test ./...` with no setup, and it vets first |

## gofmt has no options

```go
package main
import ("fmt"
"strings")
type User struct{
Name string
Age int
}
func main(){
u:=User{Name:"ada",Age : 36}
if u.Age>18 {
fmt.Println( strings.ToUpper(u.Name),"is an adult" )
}
}
```

```bash
gofmt -l .
```

Output:

```
messy.go
```

`-l` lists files whose formatting differs from canonical. That exit-code-friendly
form is the CI check. `-d` shows the unified diff:

```bash
gofmt -d messy.go
```

Output:

```
diff messy.go.orig messy.go
--- messy.go.orig
+++ messy.go
@@ -1,13 +1,18 @@
 package main
-import ("fmt"
-"strings")
-type User struct{
-Name string
-Age int
-}
-func main(){
-u:=User{Name:"ada",Age : 36}
-if u.Age>18 {
-fmt.Println( strings.ToUpper(u.Name),"is an adult" )
-}
+
+import (
+	"fmt"
+	"strings"
+)
+
+type User struct {
+	Name string
+	Age  int
+}
+
+func main() {
+	u := User{Name: "ada", Age: 36}
+	if u.Age > 18 {
+		fmt.Println(strings.ToUpper(u.Name), "is an adult")
+	}
 }
```

`gofmt -w messy.go` applies it. Note what it did beyond whitespace: it aligned
the struct field types into a column. That alignment uses **tabs**, not spaces,
which is why Go code renders correctly at any tab width and why you must not
convert tabs to spaces.

Indentation is tabs, alignment is tabs, braces are K&R with the opening brace
mandatory on the same line. There is no `--tab-width`, no `--semi`, no
`--single-quote`. The entire style debate is settled by the absence of flags, and
that is the point: every Go file on earth looks like this one.

The flags are `-l`, `-d`, `-w`, `-s`, and `-r`; the Quick Reference lists them.
`-s` is the one worth knowing about: not applied by default, nothing to do with
whitespace, and it removes redundancy gofmt alone leaves intact.

```bash
gofmt -l simp.go
gofmt -s -d simp.go
```

Output:

```
diff simp.go.orig simp.go
--- simp.go.orig
+++ simp.go
@@ -3,10 +3,10 @@
 import "fmt"
 
 func demo() {
-	m := map[string][]int{"a": []int{1, 2}}
-	s := [][]int{[]int{3}}
+	m := map[string][]int{"a": {1, 2}}
+	s := [][]int{{3}}
 	xs := []int{1, 2, 3}
-	for _ = range xs {
+	for range xs {
 	}
-	fmt.Println(m, s, xs[1:len(xs)])
+	fmt.Println(m, s, xs[1:])
 }
```

`gofmt -l` printed nothing — that file was already canonically formatted. The
redundant element types, the `_ =` in the range, and `len(xs)` as the upper bound
are all style-correct but needlessly verbose.

`go fmt ./...` is a thin wrapper that runs the `gofmt` binary across packages. It
does not accept gofmt's flags directly; pass them through with `go fmt -s` only
in recent versions, or just call `gofmt -s -w .` yourself.

## goimports: what you actually bind to save

gofmt will not add or remove imports. This file is perfectly formatted and does
not compile:

```go
package main

import (
  "os"
  "strings"
)

func main() {
  fmt.Println(strings.ToUpper("needs fmt"))
}
```

```bash
gofmt -l .
go build .
```

Output:

```
# example.com/goimp
./main.go:4:2: "os" imported and not used
./main.go:9:2: undefined: fmt
```

`gofmt -l` printed nothing. `goimports` fixes both problems in one pass:

```bash
go install golang.org/x/tools/cmd/goimports@latest
goimports -d main.go
```

Output:

```
diff -u main.go.orig main.go
--- main.go.orig	2026-10-25 23:24:18
+++ main.go	2026-10-25 23:24:18
@@ -1,7 +1,7 @@
 package main
 
 import (
-	"os"
+	"fmt"
 	"strings"
 )
 
```

It dropped the unused `os` and added the needed `fmt`, in the correct group, in
sorted order.

```bash
goimports -w main.go
go run .
```

Output:

```
NEEDS FMT
```

goimports is a strict superset of gofmt, so there is no reason to run both.
Idiomatic: bind `goimports -w` (or gopls's organize-imports action, which does the
same thing) to file save, and check `gofmt -l .` in CI. The `-local` flag groups
your own module's imports into a third block:

```bash
goimports -local example.com/myapp -w .
```

## go vet: bugs the compiler allows

This program compiles with zero errors:

```go
package main

import (
  "fmt"
  "sync"
)

type Config struct {
  Host string `json:"host`
  Port int    `json:port"`
}

type Counter struct {
  mu sync.Mutex
  n  int
}

func (c Counter) Inc() {
  c.mu.Lock()
  defer c.mu.Unlock()
  c.n++
}

func main() {
  name := "ada"
  age := 36
  fmt.Printf("%s is %s\n", name, age)
  fmt.Printf("%d\n")
  fmt.Printf("%d\n", 1, 2)

  cfg := Config{Host: "localhost", Port: 5432}
  fmt.Println(cfg)

  var c Counter
  c.Inc()
  fmt.Println(c.n)
}
```

```bash
go vet ./...
```

Output:

```
main.go:18:9: Inc passes lock by value: example.com/vettour.Counter contains sync.Mutex
main.go:27:20: fmt.Printf format %s has arg age of wrong type int
main.go:28:14: fmt.Printf format %d reads arg #1, but call has 0 args
main.go:29:2: fmt.Printf call needs 1 arg but has 2 args
main.go:9:2: struct field tag `json:"host` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
main.go:10:2: struct field tag `json:port"` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
```

Six real defects. The order of vet's lines varies between runs because analyzers
execute in parallel, so match them by file position, not sequence. Walk them:

- **copylocks** — `Inc` has a *value* receiver, so it locks a copy of the mutex and
  increments a copy of `n`. The method is a no-op and the lock protects nothing.
  This is the single most expensive bug on the list and the compiler is silent.
- **printf** — `%s` against an `int` prints `%!s(int=36)`; too few args prints
  `%!d(MISSING)`; too many appends `%!(EXTRA int=2)`. All compile, all ship.
- **structtag** — the mangled backticks mean `encoding/json` falls back to the
  field name, so your API emits `Host` instead of `host`. Silent wire-format
  change.

A second set, from code that also compiles:

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "strings"
  "time"
)

func work(ctx context.Context) error {
  ctx, _ = context.WithTimeout(ctx, time.Second)
  <-ctx.Done()
  return ctx.Err()
}

func main() {
  s := "hello"
  strings.ToUpper(s)
  fmt.Errorf("wrapped: %w", errors.New("inner"))
  fmt.Println(work(context.Background()))
}
```

```bash
go vet ./...
```

Output:

```
main.go:20:2: result of fmt.Errorf call not used
main.go:12:7: the cancel function returned by context.WithTimeout should be called, not discarded, to avoid a context leak
```

**lostcancel** is the one that matters in services: discarding the `cancel`
function leaks the timer and the goroutine behind it until the deadline fires.
**unusedresult** flags discarded results from a hardcoded list of pure functions
(`fmt.Errorf`, `fmt.Sprintf`, `errors.New`, …) — note it did *not* flag
`strings.ToUpper(s)` on line 19, because that list is deliberately short to avoid
false positives. Do not rely on vet to catch every ignored return value.

## The analyzer list

```bash
go tool vet help
```

Output (abridged — 34 analyzers, all run by default):

```
Registered analyzers:

    appends      check for missing values after append
    assign       check for useless assignments
    atomic       check for common mistakes using the sync/atomic package
    composites   check for unkeyed composite literals
    copylocks    check for locks erroneously passed by value
    defers       report common mistakes in defer statements
    errorsas     report passing non-pointer or non-error values to errors.As
    httpresponse check for mistakes using HTTP responses
    ifaceassert  detect impossible interface-to-interface type assertions
    loopclosure  check references to loop variables from within nested functions
    lostcancel   check cancel func returned by context.WithCancel is called
    printf       check consistency of Printf format strings and arguments
    shift        check for shifts that equal or exceed the width of the integer
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
    unusedresult check for unused results of calls to some functions
    waitgroup    check for misuses of sync.WaitGroup
```

The ones left out cover assembly, cgo, and build directives.

Disable one with `go vet -structtag=false ./...`, or run one
alone with `go vet -printf ./...`. Note what is *not* here: no style rules, no
complexity metrics, no naming conventions. Go vet reports likely mistakes only,
and it holds its false-positive rate near zero — which is why it is safe to make
it a hard CI gate.

## go test vets for you

`go test` runs a high-confidence subset of vet (`printf`, `tests`, `atomic` and a
few others) before compiling the test binary, and a finding is a build failure:

```bash
go test ./...
```

Output:

```
# example.com/vet3
# [example.com/vet3]
./main_test.go:9:14: fmt.Printf format %d has arg "string" of wrong type string
FAIL	example.com/vet3 [build failed]
FAIL
```

`go test -vet=off ./...` disables it; `-vet=all` runs the full set.

## go fix suggests rewrites

Go 1.27's vet also hosts "opportunity" analyzers whose findings come with
machine-applicable fixes. These surface through `go fix`, not `go vet`:

```go
package main

import (
  "fmt"
  "strings"
)

func main() {
  doc := "a\nb\nc"
  for _, line := range strings.Split(doc, "\n") {
    fmt.Println(line)
  }
}
```

```bash
go vet ./...
go fix -diff ./...
```

Output:

```
--- /tmp/scratch/main.go (old)
+++ /tmp/scratch/main.go (new)
@@ -7,7 +7,7 @@
 
 func main() {
 	doc := "a\nb\nc"
-	for _, line := range strings.Split(doc, "\n") {
+	for line := range strings.SplitSeq(doc, "\n") {
 		fmt.Println(line)
 	}
 }
```

`go vet` printed nothing — this is not a bug, just an allocation you can avoid.
`go fix ./...` applies the change. Review these; they are suggestions, and the
iterator form is only equivalent if you do not need the slice.

## staticcheck and golangci-lint, briefly

`go vet` is deliberately narrow. Two third-party tools go further:

```bash
go install honnef.co/go/tools/cmd/staticcheck@latest
staticcheck ./...
```

staticcheck adds several hundred checks across correctness (`SA`), simplification
(`S`), style (`ST`) and unused code (`U`) — dead code, impossible conditions,
misused `time.Duration` arithmetic, loops that can never execute. `golangci-lint`
is a runner that executes staticcheck plus dozens of other linters in parallel
with a shared `.golangci.yml`. Module 15.3 covers both properly, including which
checks are worth enabling.

One honest caveat observed while writing this lesson: the staticcheck release
installed here (2025.1.1) predates Go 1.27 and cannot read its export data, so it
fails on this toolchain with `export data version 4 is greater than maximum
supported version 2` rather than producing findings. That is the normal cost of
third-party analyzers — they lag each Go release by weeks. `go vet` never does,
because it ships with the compiler. Keep vet as the gate and the others as
advisory.

## gopls and the editor

`gopls` is the official language server; every editor integration is a thin
client over it.

```bash
go install golang.org/x/tools/gopls@latest
gopls version
```

Output:

```
golang.org/x/tools/gopls v0.23.0
```

The settings that matter, whatever the editor:

| Setting | Value |
|---|---|
| format on save | on, via gopls (`source.organizeImports` + format) |
| `gopls.staticcheck` | `true` if you want staticcheck findings inline |
| `gopls.analyses` | enable `unusedparams`, `shadow` if you like them |
| tab size | 4 for display; the file contains tabs regardless |
| `gopls.hints` | inlay hints for parameter names and composite literal fields |

In VS Code, the official `golang.go` extension fetches gopls, delve, and
goimports into `$GOBIN` on first prompt. Neovim uses `gopls` via
`nvim-lspconfig`. GoLand has its own engine and does not use gopls.

gopls needs a module: open the directory containing `go.mod`, not its parent.
That is the single most common "autocomplete is broken" cause.

## go doc

The reference ships with the toolchain and works offline.

```bash
go doc strings.ToUpper
```

Output:

```
package strings // import "strings"

func ToUpper(s string) string
    ToUpper returns s with all Unicode letters mapped to their upper case.
```

A type shows its exported fields and its full method set:

```bash
go doc strings.Builder
```

Output:

```
package strings // import "strings"

type Builder struct {
	// Has unexported fields.
}
    A Builder is used to efficiently build a string using Builder.Write methods.
    It minimizes memory copying. The zero value is ready to use. Do not copy a
    non-zero Builder.

func (b *Builder) Cap() int
func (b *Builder) Grow(n int)
func (b *Builder) Len() int
func (b *Builder) Reset()
func (b *Builder) String() string
func (b *Builder) Write(p []byte) (int, error)
func (b *Builder) WriteByte(c byte) error
func (b *Builder) WriteRune(r rune) (int, error)
func (b *Builder) WriteString(s string) (int, error)
```

`-all` dumps the entire package with every doc comment — the fastest way to learn
an unfamiliar stdlib package:

```bash
go doc -all strings | head -11
```

Output:

```
package strings // import "strings"

Package strings implements simple functions to manipulate UTF-8 encoded strings.

For information about UTF-8 strings in Go, see https://blog.golang.org/strings.

FUNCTIONS

func Clone(s string) string
    Clone returns a fresh copy of s. It guarantees to make a copy of s into a
    new allocation, which can be important when retaining only a small substring
```

It works on your own packages too, which is the fastest review of a package's
public surface:

```bash
go doc ./mathx
```

Output:

```
package mathx // import "example.com/pkgtour/mathx"

const Pi = 3.14159
func Calls() int
func Double(n int) int
```

Three exported names, no doc comments — which `go doc` makes obvious. Writing
them is lesson 2.7. `go doc -u` additionally shows unexported identifiers.

## Quick Reference

| Command | Does |
|---|---|
| `gofmt -l .` | list misformatted files; silent on success |
| `gofmt -d FILE` | print the formatting diff |
| `gofmt -w .` | rewrite in place |
| `gofmt -s -w .` | rewrite, also simplifying |
| `gofmt -r 'a[b:len(a)] -> a[b:]' -w .` | apply a rewrite rule |
| `go fmt ./...` | run gofmt over packages |
| `goimports -w .` | gofmt plus add/remove/sort imports |
| `goimports -local MOD -w .` | group your own module's imports separately |
| `go vet ./...` | all analyzers over all packages |
| `go vet -printf ./...` | one analyzer only |
| `go vet -structtag=false ./...` | all but one |
| `go tool vet help` | the analyzer list and flags |
| `go fix -diff ./...` | preview machine-applicable suggestions |
| `go test -vet=off ./...` | skip the automatic vet pass |
| `go doc PKG` | exported API of a package |
| `go doc PKG.Symbol` | one symbol, with its doc comment |
| `go doc -all PKG` | everything, with all doc comments |
| `go doc -u PKG` | include unexported identifiers |
| `go doc ./localpkg` | your own package |

| Tool | Scope | Ships with Go |
|---|---|---|
| `gofmt` | formatting only | yes |
| `goimports` | formatting + imports | no (`x/tools`) |
| `go vet` | likely bugs, near-zero false positives | yes |
| `go fix` | mechanical modernisations | yes |
| `staticcheck` | hundreds of correctness + style checks | no |
| `golangci-lint` | runner for many linters | no |
| `gopls` | language server | no (`x/tools`) |

The CI minimum, in order:

```bash
gofmt -l . | tee /dev/stderr | (! read)
go vet ./...
go test ./... -race
```

## Common Mistakes

**Configuring your editor to format Go with Prettier or a generic formatter.**
Only gofmt produces canonical Go; anything else creates diff churn against every
other contributor.

**Converting tabs to spaces because the diff looked odd.** Go's alignment depends
on tabs; spaces break column alignment at other tab widths and `gofmt -l` will
flag every file.

**Relying on `gofmt` to fix a missing import.** It never touches the import
block's contents — that is goimports or gopls.

**Treating `go vet` findings as style suggestions.** vet reports likely bugs with
a very low false-positive rate; a `copylocks` or `lostcancel` hit is almost always
a real defect.

**Assuming `go vet` catches every ignored return value.** `unusedresult` checks a
short hardcoded list; `strings.ToUpper(s)` discarded passes silently.

**Opening the repository root in your editor when `go.mod` is in a
subdirectory.** gopls needs the module root, and the symptom is total loss of
completion and jump-to-definition.

## Exercises

1. Write a file that `gofmt -l` accepts but `gofmt -s -d` wants to change in at
   least three distinct ways. Capture the diff.
2. Reproduce all six vet findings from this lesson in a fresh module, confirm
   `go build` succeeds, then fix each one and confirm vet goes quiet.
3. Write a struct with a `sync.Mutex` and a value-receiver method that increments
   a counter. Call it 1000 times, print the counter, and explain the number with
   reference to the `copylocks` finding.
4. Add a `gofmt -l` gate to a shell script that exits non-zero when any file is
   misformatted, and verify it fails on a misformatted file and passes otherwise.
5. Run `go doc -all` on `errors`, `io`, and `context` and write down, for each,
   the three symbols you expect to use most.
6. Install staticcheck and run it on a package that `go vet` passes. Classify
   each finding as a real bug, a simplification, or a false positive.

---

[← 1.4 Packages, Imports, and Visibility](04-packages-imports.md) · [Course Index](../README.md) · [1.6 How Go Thinks: A Mental Model →](06-mental-model.md)
