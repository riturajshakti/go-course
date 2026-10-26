[← 15.3 go vet, staticcheck, and golangci-lint](03-linters.md) · [Course Index](../README.md) · [15.5 Cross-Compilation and Static Binaries →](05-cross-compilation.md)

# 15.4 — Build Tags and Conditional Compilation

## What You'll Learn

- `//go:build` with `&&`, `||`, `!` and parens; `// +build` is the deprecated form gofmt keeps in sync
- Filename suffixes (`_linux.go`, `_amd64.go`) are constraints too, and need no comment
- `go list -tags` shows exactly which files the toolchain selected
- A typo'd tag silently excludes a file — and the two ways that shows up

## Coming From Another Language

| Other | Go |
|---|---|
| `#ifdef` / `#if defined(X)` in C | per-**file** constraints; there is no in-function conditional compilation |
| `process.env.NODE_ENV === 'production'` at runtime | a tag decides at compile time; the other branch is not in the binary |
| `#[cfg(target_os = "linux")]` on items | `//go:build linux` on whole files, or a `_linux.go` suffix |
| webpack `DefinePlugin` + tree shaking | the excluded file is never parsed, so dead code cannot exist |
| `sys.platform ==` checks everywhere | one implementation per file, one symbol, no branching |

## The directive and where it goes

```go
//go:build debug

package main

func mode() string { return "debug (verbose, assertions on)" }
```

Three rules, all enforced:

1. The line must appear **before** the `package` clause.
2. It must be followed by a **blank line** (otherwise it is a doc comment for
   the package, not a directive).
3. Only `//` comments and blank lines may precede it.

Break rule 1 and the compiler says so:

```go
package main

//go:build debug

func helper() string { return "x" }
```

This is an intentional failure. Output:

```
# example.com/scratch
./bad.go:3:3: misplaced compiler directive
```

`go vet` reports it too, via the `buildtag` analyzer:

```
bad.go:3:1: misplaced //go:build comment
```

Break rule 2 and you get the dangerous case — no error at all:

```go
//go:build debug
package main

func helper() string { return "x" }
```

```bash
go build .
```

This is an intentional failure. Output:

```
# example.com/scratch
./main.go:5:27: undefined: helper
```

The constraint *was* honoured (the file is excluded without the `debug` tag),
so the symptom is a missing symbol somewhere else entirely. gofmt fixes it:

```bash
gofmt -d bad.go
```

Output:

```
diff bad.go.orig bad.go
--- bad.go.orig
+++ bad.go
@@ -1,4 +1,5 @@
 //go:build debug
+
 package main
 
 func helper() string { return "x" }
```

## The expression grammar

| Form | Means |
|---|---|
| `//go:build linux` | one term |
| `//go:build linux && amd64` | both |
| `//go:build linux \|\| darwin` | either |
| `//go:build !windows` | negation |
| `//go:build (linux \|\| darwin) && amd64 && !nocgo` | parens, any nesting |

`&&` binds tighter than `||`, as everywhere else. gofmt adds the clarifying
parens whether you asked or not:

```go
//go:build linux && !arm64 || darwin
```

```bash
gofmt -d new.go
```

Output:

```
diff new.go.orig new.go
--- new.go.orig
+++ new.go
@@ -1,4 +1,4 @@
-//go:build linux && !arm64 || darwin
+//go:build (linux && !arm64) || darwin
 
 package main
 
```

Exactly one `//go:build` line per file. Two lines is an error, not an implicit
`&&` — that was the old form's behaviour.

## The deprecated // +build form

Before Go 1.17 the syntax was a space/comma soup: space meant OR, comma meant
AND, `!` meant NOT, and multiple lines meant AND.

```go
// +build linux,!arm64 darwin

package main

func legacy() {}
```

gofmt synthesises the modern line above it:

```bash
gofmt -d old.go
```

Output:

```
diff old.go.orig old.go
--- old.go.orig
+++ old.go
@@ -1,3 +1,4 @@
+//go:build (linux && !arm64) || darwin
 // +build linux,!arm64 darwin
 
 package main
```

`linux,!arm64 darwin` → `(linux && !arm64) || darwin`. gofmt keeps the two in
sync in both directions and errors if they disagree, which is the migration
path: run `gofmt -w .` once, then delete the `// +build` lines when you no
longer support Go 1.16. If you are writing new code today you write only
`//go:build`.

## Filename-based constraints

No comment needed: a filename ending in `_GOOS.go`, `_GOARCH.go`, or
`_GOOS_GOARCH.go` is constrained implicitly. One function, three
implementations, zero `if`:

```go
package main

import "fmt"

func main() {
  fmt.Println("platform:", platform())
  fmt.Println("mode:", mode())
}
```

```go
package main

func platform() string { return "macOS" }
```

```go
package main

func platform() string { return "Linux" }
```

```go
package main

func platform() string { return "Windows" }
```

Those three are `plat_darwin.go`, `plat_linux.go`, `plat_windows.go`. Plus a
pair driven by a custom tag, `mode_debug.go` and `mode_release.go`:

```go
//go:build debug

package main

func mode() string { return "debug (verbose, assertions on)" }
```

```go
//go:build !debug

package main

func mode() string { return "release" }
```

```bash
go run .
```

Output (on darwin/arm64):

```
platform: macOS
mode: release
```

```bash
go run -tags debug .
```

Output:

```
platform: macOS
mode: debug (verbose, assertions on)
```

Two things to take from this. The `!debug` file is what makes it compile at
all — without it, a plain build has no `mode` at all, so **every** tag you
introduce needs a default implementation or a stub. And no `platform` function
was selected by a tag: the filename did it.

The recognised suffixes are every valid `GOOS` and `GOARCH` value, plus `_test`
(test files), `_unix` (any Unix-like GOOS, since Go 1.19), and `_js`/`_wasm`.
A leading underscore or dot excludes a file from every build:
`_scratch.go` is never compiled, which is where `go generate` templates often
live.

One trap: `fmt_windows.go` is constrained to Windows whether you meant it or
not. A file named for a *concept* that happens to end in a GOOS name gets
silently excluded. `http_darwin_helpers.go` is fine (the suffix must be the
last component); `helpers_darwin.go` is constrained.

## Which files were selected

`go list` answers this definitively, and it is the first thing to run when a
symbol is mysteriously undefined:

```bash
go list -f '{{.GoFiles}}' .
go list -tags debug -f '{{.GoFiles}}' .
GOOS=linux go list -f '{{.GoFiles}}' .
GOOS=windows go list -f '{{.GoFiles}}' .
```

Output:

```
[main.go mode_release.go plat_darwin.go]
[main.go mode_debug.go plat_darwin.go]
[main.go mode_release.go plat_linux.go]
[main.go mode_release.go plat_windows.go]
```

Four selections from the same six files. The complement is as useful:

```bash
go list -f 'ignored: {{.IgnoredGoFiles}}' .
GOOS=linux go list -f 'ignored: {{.IgnoredGoFiles}}' .
```

Output:

```
ignored: [mode_debug.go plat_linux.go plat_windows.go]
ignored: [mode_debug.go plat_darwin.go plat_windows.go]
```

Other fields worth knowing: `.TestGoFiles`, `.XTestGoFiles`, `.CgoFiles`,
`.IgnoredOtherFiles`. `go list -json .` dumps all of them.

## GOOS, GOARCH, and version constraints

Any `GOOS` and any `GOARCH` value is a tag, always, with no `-tags` flag
needed. So are these:

| Tag | Set when |
|---|---|
| `linux`, `darwin`, `windows`, `js`, `wasip1`, … | `GOOS` matches |
| `amd64`, `arm64`, `386`, `riscv64`, … | `GOARCH` matches |
| `unix` | GOOS is any Unix-like system |
| `cgo` | cgo is enabled **and** a C toolchain is available |
| `race`, `msan`, `asan` | built with the corresponding `-race`/`-msan`/`-asan` |
| `gc`, `gccgo` | the compiler in use |
| `go1.1` … `go1.27` | the language version is **at least** that |
| `goexperiment.*` | that GOEXPERIMENT is on |

`go1.N` is the one with surprising semantics: it is `>=`, not `==`, and there
is no `go1.21.3`. Only minor versions. That gives you the stdlib-migration
pattern:

```go
//go:build go1.21

package main

import (
  "fmt"
  "slices"
)

func impl() string {
  xs := []int{3, 1, 2}
  slices.Sort(xs)
  return fmt.Sprint(xs)
}
```

```go
//go:build !go1.21

package main

import (
  "fmt"
  "sort"
)

func impl() string {
  xs := []int{3, 1, 2}
  sort.Ints(xs)
  return fmt.Sprint(xs)
}
```

```bash
go list -f '{{.GoFiles}}' .
go run .
```

Output:

```
[main.go new_api.go]
[1 2 3]
```

This is how a library supports a range of Go versions while using new stdlib
APIs on new toolchains. It is far less common now that modules carry a `go`
directive, which refuses to build on an older compiler anyway.

`cgo` in one line: `//go:build cgo` selects a C-backed implementation and
`//go:build !cgo` its pure-Go fallback — the pattern the `net` and `os/user`
packages use, which 15.5 covers under `CGO_ENABLED=0`.

Combining them reads exactly as you would hope:

```go
//go:build (linux || darwin) && amd64 && !nocgo

package main

func combo() {}
```

```bash
go list -f '{{.IgnoredGoFiles}}' .
GOARCH=amd64 go list -f '{{.GoFiles}}' .
```

Output:

```
[combo.go mode_debug.go plat_linux.go plat_windows.go]
[combo.go main.go mode_release.go plat_darwin.go]
```

Ignored on `darwin/arm64`, selected on `darwin/amd64`.

## Custom tags with -tags

```bash
go build -tags debug .
go build -tags "debug,verbose" .
go build -tags debug,verbose .
go test -tags integration ./...
```

Comma and space both separate tags in the `-tags` value (the space form needs
quoting). The tags are additive: `-tags debug,verbose` sets both, and a file
needing `debug && verbose` is then included.

Unknown tags are **not** an error:

```bash
go build -tags thisdoesnotexist -o /dev/null .
```

That succeeds silently. There is no registry of valid tags, no `-tags` typo
check, and `go vet` will not help you. Which brings us to the real hazard.

## The silent-nothing failure

A package whose only file is excluded does not vanish quietly — but a package
whose *one interesting* file is excluded does. The loud version first:

```go
//go:build intergation

package sub

func Helper() string { return "x" }
```

Note the typo. `sub/` now has no files at all for a default build:

```bash
go build ./sub
```

This is an intentional failure. Output (the absolute path is this scratch
directory; yours will differ):

```
package example.com/scratch/sub: build constraints exclude all Go files in /tmp/scratch/sub
```

"build constraints exclude all Go files" is a clear message, and it appears
from `go build`, `go vet`, and `go list` alike.

The quiet version is a package with several files where one is wrongly tagged.
Nothing errors; a function is just missing, and you get `undefined: Helper` at
whichever call site happens to reference it. Or worse, if a `!debug` default
also exists, the *wrong* implementation is silently selected and the build
succeeds.

Three detection techniques, in order of usefulness:

```bash
go list -f '{{.ImportPath}} {{.IgnoredGoFiles}}' ./... | grep -v '\[\]'
```

Every package with an excluded file, which you then eyeball. In CI, build the
tag combinations you actually support:

```bash
for t in "" debug integration "debug,integration"; do
  echo "== tags='$t'"
  go build -tags "$t" ./... || exit 1
  go vet -tags "$t" ./... || exit 1
done
```

And for a whole-matrix check:

```bash
for os in linux darwin windows; do
  GOOS=$os go vet ./... || exit 1
done
```

That third loop is the one people skip and then discover at release time that
`_windows.go` has not compiled in six months. 15.5 turns it into a real
cross-compilation matrix.

## The ignore tag convention

`ignore` is not special to the toolchain — it is just a tag nobody ever passes,
which is exactly what makes it useful. Two established uses.

A generator you run with `go run`, which must not be part of the package:

```go
//go:build ignore

package main

func main() {
  // writes generated code; invoked via go:generate (15.8)
}
```

And the pre-`tool`-directive way of pinning tool dependencies:

```go
//go:build ignore

package main

import _ "golang.org/x/tools/cmd/stringer"
```

```bash
go list -f '{{.GoFiles}}' ./tools
```

Output:

```
package example.com/scratch/tools: build constraints exclude all Go files in /tmp/scratch/tools
```

The file is never compiled, but `go mod tidy` *does* read its imports, so the
tool's version lands in `go.mod`. Since Go 1.24 the `tool` directive and
`go get -tool` (15.2) replace this, and new code should use those.

## Build-tagged tests

The standard pattern for tests that need a database, a network, or minutes of
runtime:

```go
//go:build integration

package main

import "testing"

func TestSlow(t *testing.T) { t.Log("hit a real database") }
```

```bash
go test -v ./...
```

Output:

```
?   	example.com/scratch	[no test files]
```

The package reports "no test files" — the tagged file is invisible. With the
tag:

```bash
go test -tags integration -v -run TestSlow .
```

Output:

```
=== RUN   TestSlow
    slow_test.go:7: hit a real database
--- PASS: TestSlow (0.00s)
PASS
ok  	example.com/scratch	0.503s
```

14.3 covers this alongside the alternative, `testing.Short()` with
`-test.short`. The split is: a tag when the test cannot even **compile** without
something (a cgo dependency, a driver, a generated fixture), `testing.Short()`
when it compiles fine and you only want to skip it. The tag is the stronger
tool and the one that stops a missing dependency from breaking `go build ./...`.

One line on embedded assets: `//go:embed` (13.13) is a *different* directive
with the same syntax shape. It is not a build constraint and does not go before
the package clause — it goes immediately above the variable it fills. Build
tags and `//go:embed` compose, so `//go:build dev` plus a `//go:embed` of a
live directory, against a `!dev` file embedding the built assets, is the
standard dev/prod split for a Go server serving a frontend.

## Quick Reference

| Syntax | Means |
|---|---|
| `//go:build tag` | include only when `tag` is set |
| `//go:build !tag` | include only when it is not |
| `//go:build a && b` | both |
| `//go:build a \|\| b` | either |
| `//go:build (a \|\| b) && !c` | grouped |
| `// +build a,b c` | deprecated: `(a && b) \|\| c` |
| `name_linux.go` | GOOS=linux only |
| `name_amd64.go` | GOARCH=amd64 only |
| `name_linux_amd64.go` | both |
| `name_unix.go` | any Unix-like GOOS |
| `name_test.go` | test files only |
| `_name.go`, `.name.go` | never compiled |

| Command | Does |
|---|---|
| `go build -tags a,b .` | build with tags set |
| `go test -tags integration ./...` | include tagged test files |
| `go run -tags debug .` | run with tags set |
| `go vet -tags debug ./...` | vet the tagged selection |
| `go list -f '{{.GoFiles}}' .` | files selected |
| `go list -f '{{.IgnoredGoFiles}}' .` | files excluded |
| `go list -tags t -f '{{.GoFiles}}' .` | selection under a tag |
| `GOOS=linux go list -f '{{.GoFiles}}' .` | selection for another platform |
| `GOFLAGS=-tags=debug` | set tags for every command (15.1) |

| Implicit tag | Source |
|---|---|
| `linux`, `darwin`, `windows`, … | `GOOS` |
| `amd64`, `arm64`, … | `GOARCH` |
| `unix` | Unix-like GOOS |
| `cgo` | `CGO_ENABLED=1` and a C toolchain present |
| `race`, `msan`, `asan` | the matching build flag |
| `go1.21` | language version **>=** 1.21 |
| `gc`, `gccgo` | the compiler |

## Common Mistakes

**No blank line between `//go:build` and `package`.** It becomes a doc comment
and the constraint is... still applied, but the symptom surfaces as
`undefined:` at an unrelated call site. Run `gofmt -w .`.

**Adding a `//go:build debug` file with no `!debug` counterpart.** The default
build then has no implementation of that symbol; every tag needs a default or
a stub.

**Naming a file after a concept that ends in a GOOS or GOARCH name.**
`cache_windows.go` is Windows-only whether you meant it or not — put the
concept first: `windows_cache.go`.

**Trusting that a typo'd tag will be reported.** Unknown tags are legal and
silent; `go list -f '{{.IgnoredGoFiles}}'` is the only check.

**Only ever building for your own GOOS.** A `_windows.go` file that nobody
compiles rots; add `GOOS=windows go vet ./...` to CI (15.5).

**Using a build tag where `testing.Short()` would do.** A tag makes the test
invisible to `go build ./...` and to your editor; reserve it for tests that
genuinely cannot compile without something.

## Exercises

1. Build the `platform`/`mode` example from this lesson, then add an `arm64`
   and an `amd64` variant of a third function. Use `go list -tags` plus
   `GOOS`/`GOARCH` to print the selected file set for six combinations.
2. Write a file whose `//go:build` line and `// +build` line disagree. Capture
   what gofmt and the compiler each say about it.
3. Introduce a tag typo in a package with three files so that the build
   succeeds but the wrong implementation is selected. Then write the one-line
   `go list` command that would have caught it.
4. Add a `debug` tag that enables an expensive invariant check in a hot
   function. Benchmark the function with and without the tag and report the
   difference.
5. Write a CI script that builds and vets your module for every tag
   combination you support, crossed with three values of GOOS. Make it fail
   fast and print which combination broke.
6. Convert a `tools.go` file with an `ignore` tag to the Go 1.24 `tool`
   directive. Confirm `go tool <name>` works and that `go mod tidy` still pins
   the version.

---

[← 15.3 go vet, staticcheck, and golangci-lint](03-linters.md) · [Course Index](../README.md) · [15.5 Cross-Compilation and Static Binaries →](05-cross-compilation.md)
