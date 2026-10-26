[← 14.8 Test Doubles with Interfaces](../14-testing/08-test-doubles.md) · [Course Index](../README.md) · [15.2 go mod tidy, vendor, and Dependency Management →](02-mod-tidy-vendor.md)

# 15.1 — go build, run, install, and the Build Cache

## What You'll Learn

- The three commands and where each one puts its output: nowhere, here, `$GOBIN`
- The build cache is why the second build takes 0.18s instead of 4.16s
- `-ldflags` for version injection and binary size, `-gcflags=-m` for escape analysis
- `go version -m` reads the module graph back out of a shipped binary

## Coming From Another Language

| Other | Go |
|---|---|
| `node script.js` interprets | `go run .` compiles to a temp dir, executes, deletes |
| `tsc && node dist/x.js` two steps, two artifacts | `go build` produces one static-ish binary, no runtime needed |
| `npm install -g cli` installs from a registry | `go install pkg@v1.2.3` compiles from source into `$GOBIN` |
| `cargo build` writes to `target/` | `go build` writes to the current directory; the cache lives elsewhere |
| `webpack --mode=production` strips | `-ldflags "-s -w"` strips; it is a linker flag, not a mode |
| `process.env.VERSION` at runtime | `-ldflags "-X main.version=1.4.2"` at link time |

## The program used throughout

```go
package main

import (
  "fmt"
  "runtime"
)

var version = "dev"

func main() {
  fmt.Printf("build-demo %s (%s/%s)\n", version, runtime.GOOS, runtime.GOARCH)
}
```

The module is `example.com/scratch`, with two extra packages for the directory
examples:

```
go.mod
main.go
mathx/mathx.go          package mathx
cmd/app/main.go         package main
```

## go run compiles to a temp directory and throws it away

```bash
go run .
```

Output:

```
build-demo dev (darwin/arm64)
```

There is no binary in the directory afterwards. `-x` shows where it actually
went:

```bash
go run -x . 2>&1 | tail -6
```

Output (the `WORK` path and the cache hash differ every run):

```
WORK=/var/folders/3l/gf781gh95p75bmw55xppgfhh0000gn/T/go-build2946412618
/Users/raj/Library/Caches/go-build/e5/e5e2264a25630f2454f00fc36cff33e18678940e762fb68e445b952042ccb873-d/scratch
build-demo dev (darwin/arm64)
```

The link step writes `$WORK/b001/exe/scratch`, a copy goes into the build cache,
and `$WORK` is deleted when the process exits. So `go run` is not an
interpreter — it is a full compile and link whose artifact you never see. The
*compilation* is cached (that is why the second `go run` is instant), but the
linked executable is not kept where you can reach it.

Three forms, three meanings:

```bash
go run .            # the main package in the current directory
go run main.go      # exactly these files, as package "command-line-arguments"
go run ./cmd/app    # the main package in that directory
```

Output:

```
build-demo dev (darwin/arm64)
build-demo dev (darwin/arm64)
cmd/app
```

`go run main.go` happens to work here because the program is one file. The
moment you split it, that form breaks while `go run .` keeps working — the file
list form only compiles the files you name. Idiomatic: always `go run .` or
`go run ./cmd/app`, never a file list.

Arguments after the package go to the program, not to `go`:

```bash
go run . -flag=value   # -flag=value reaches os.Args
go run -race . -v      # -race is go's, -v is the program's
```

## go build and its output naming rules

```bash
go build .
ls
```

Output:

```
cmd
go.mod
main.go
mathx
scratch
```

The binary is named after the **directory's import path base**, not after
`main.go`. The module is `example.com/scratch` in a directory called `l1`, and
the binary came out `scratch` — the import path wins. With `./cmd/app` you get
`app`.

`-o` overrides, and creates the parent directory if needed:

```bash
go build -o bin/demo .
ls -l bin/demo
```

Output (size varies with Go version):

```
-rwxr-xr-x@ 1 raj  staff  2429922 Oct 26 13:20 bin/demo
```

Building a **non-main** package type-checks it and discards the result:

```bash
go build ./mathx
ls
```

Output:

```
cmd
go.mod
main.go
mathx
scratch
```

No `mathx` file appeared. For library packages `go build` is purely a
compile check, which is exactly why `go build ./...` is a useful CI step:

```bash
go build ./...
```

It prints nothing on success and — verified on Go 1.27 — writes **no**
binaries at all, not even for the main packages it finds:

```bash
go build ./...
ls | grep -E '^(scratch|app)$' || echo "NO BINARIES WRITTEN"
```

Output:

```
NO BINARIES WRITTEN
```

A multi-package pattern discards every result; only a single named package
(`go build .`) produces a file. So `go build ./...` is purely a compile check,
and it is the cheapest useful CI step there is.

## The build cache

```bash
go env GOCACHE
```

Output:

```
/Users/raj/Library/Caches/go-build
```

Every compiled package, keyed by a hash of its source, its dependencies'
hashes, the compiler version and the build flags, lands in there. Change
nothing and the compiler is never invoked. Add some heavy stdlib imports to
make the difference visible:

```go
package main

import (
  _ "crypto/tls"
  _ "encoding/json"
  _ "net/http"
  _ "os/exec"
  _ "regexp"
  _ "text/template"
)
```

```bash
go clean -cache
echo COLD; /usr/bin/time -p go build -o /dev/null .
echo WARM; /usr/bin/time -p go build -o /dev/null .
```

Output (absolute times depend on the machine; the *ratio* is the point):

```
COLD
real 4.16
user 14.66
sys 3.91
WARM
real 0.18
user 0.20
sys 0.46
```

23x. Note `user` 14.66s against `real` 4.16s on the cold build — the toolchain
compiles packages in parallel across cores.

`-a` forces a rebuild of everything, ignoring the cache for compilation but
still writing results back:

```bash
/usr/bin/time -p go build -a -o /dev/null .
```

Output:

```
real 2.97
user 14.00
sys 2.89
```

Same `user` time as a cold build, slightly less `real` — `-a` is almost never
what you want. If you think the cache is lying to you it is far more likely a
stale generated file or a `replace` pointing somewhere unexpected. `go clean
-cache` is the honest reset.

`go clean` has four scopes:

| Command | Removes |
|---|---|
| `go clean` | binaries and object files in the named packages' directories |
| `go clean -cache` | the entire build cache (`$GOCACHE`) |
| `go clean -testcache` | only the cached *test results* |
| `go clean -modcache` | the downloaded module source in `$GOMODCACHE` |
| `go clean -fuzzcache` | stored fuzzing corpora |

`-n` prints what it would do without doing it. `go clean -n -cache` dumps 256
`rm -rf` lines, one per cache shard directory — the cache is a two-hex-digit
fan-out tree.

`-modcache` is the slow one to regret: it forces every dependency in every
project to be re-downloaded.

## Seeing the actual commands: -x and -work

```bash
go clean -cache
go build -x -o /dev/null . 2>&1 | head -6
```

Output (truncated):

```
WORK=/var/folders/3l/gf781gh95p75bmw55xppgfhh0000gn/T/go-build737292661
mkdir -p $WORK/b006/
mkdir -p $WORK/b012/
mkdir -p $WORK/b017/
mkdir -p $WORK/b015/
mkdir -p $WORK/b008/
```

Filtering to the real compiler invocations:

```bash
go build -x -o /dev/null . 2>&1 | grep compile | head -1
```

Output (one line, wrapped here):

```
/opt/homebrew/Cellar/go/1.27.1/libexec/pkg/tool/darwin_arm64/compile -o $WORK/b015/_pkg_.a
 -trimpath "$WORK/b015=>" -p internal/coverage/rtcov -lang=go1.27 -std -complete -buildid ...
```

`go build` is a driver: it computes the package graph, then shells out to
`compile`, `asm`, `pack` and `link` in `$GOROOT/pkg/tool/$GOOS_$GOARCH`.

`-work` prints the work directory and **does not delete it**, so you can go
read the generated import configs and archives:

```bash
go build -work -o /dev/null .
```

Output:

```
WORK=/var/folders/3l/gf781gh95p75bmw55xppgfhh0000gn/T/go-build2363048376
```

Clean it up yourself afterwards.

## go install and $GOBIN

`go install` builds and moves the binary into `$GOBIN`, falling back to
`$GOPATH/bin`:

```bash
go env GOBIN GOPATH
```

Output:

```
/Users/raj/go/bin
/Users/raj/go
```

```bash
go install .
ls $(go env GOPATH)/bin
```

Output:

```
air
dlv
goimports
gopls
hello
scratch
staticcheck
```

That directory must be on your `PATH` or installing is pointless:

```bash
export PATH="$PATH:$(go env GOPATH)/bin"
```

If `GOBIN` is empty, `$GOPATH/bin` is used; `$GOPATH` itself defaults to
`$HOME/go`. This is the only place GOPATH still matters in a modules world.

## go install pkg@version for tools

The version-suffixed form builds a tool **outside** any module — your `go.mod`
is not consulted and not modified:

```bash
go install github.com/rakyll/hey@latest
```

Output:

```
go: downloading github.com/rakyll/hey v0.1.5
go: downloading golang.org/x/net v0.48.0
go: downloading golang.org/x/text v0.33.0
```

```bash
ls -l /Users/raj/go/bin/hey
```

Output:

```
-rwxr-xr-x@ 1 raj  staff  12589602 Oct 26 13:21 /Users/raj/go/bin/hey
```

This is how every Go CLI tool is installed: `goimports`, `staticcheck`, `dlv`,
`golangci-lint`, `migrate`. Pin the version in CI (`@v0.1.5`), use `@latest`
only interactively. `go install` with a version **requires** a main package —
it refuses a library.

## Reading a binary back: go version -m

The linker embeds the module graph and the build settings:

```bash
go version -m /Users/raj/go/bin/hey
```

Output:

```
/Users/raj/go/bin/hey: go1.27.1
	path	github.com/rakyll/hey
	mod	github.com/rakyll/hey	v0.1.5	h1:oc3QhpT8ETXcr5xIE2xgWYNSNA/Z52XA20ku9hWCchY=
	dep	golang.org/x/net	v0.48.0	h1:zyQRTTrjc33Lhh0fBgT/H3oZq9WuvRR5gPC70xpDiQU=
	dep	golang.org/x/text	v0.33.0	h1:B3njUFyqtHDUI5jMn1YIr5B0IE2U0qck04r6d4KPAxE=
	build	-buildmode=exe
	build	-compiler=gc
	build	DefaultGODEBUG=containermaxprocs=0,cryptocustomrand=1,...
	build	CGO_ENABLED=1
	build	CGO_CFLAGS=
	build	CGO_CPPFLAGS=
	build	CGO_CXXFLAGS=
	build	CGO_LDFLAGS=
	build	GOARCH=arm64
	build	GOOS=darwin
	build	GOARM64=v8.0
```

`mod` is the main module with its checksum, each `dep` is a dependency with
its own, and the `build` lines are the flags and environment that produced it.
This is what vulnerability scanners read (`govulncheck` consumes exactly this),
and how you answer "which commit is actually running in prod" from a binary
alone. The same program built from a dirty working tree with VCS info shows
`vcs.revision` and `vcs.modified` lines too.

The *same* data is available to the program itself through
`runtime/debug.ReadBuildInfo()`.

## -ldflags: size and version injection

Two linker flags matter in practice. `-s -w` drops the symbol table and the
DWARF debug info:

```bash
go build -o bin/demo .
go build -o bin/demo-sw -ldflags "-s -w" .
ls -l bin/demo bin/demo-sw
```

Output:

```
-rwxr-xr-x@ 1 raj  staff  5452178 Oct 26 13:21 bin/demo
-rwxr-xr-x@ 1 raj  staff  3665874 Oct 26 13:21 bin/demo-sw
```

33% smaller, and it costs you readable stack traces' symbol names plus the
ability to use a debugger — 15.7 shows delve failing on a stripped binary.

`-X importpath.name=value` sets a **string variable** at link time. This is the
canonical Go way to stamp a version, and it is genuinely useful:

```go
package main

import (
  "fmt"
  "runtime"
)

var version = "dev"

func main() {
  fmt.Printf("build-demo %s (%s/%s)\n", version, runtime.GOOS, runtime.GOARCH)
}
```

```bash
go build -o bin/demo-ver -ldflags "-X main.version=1.4.2" .
./bin/demo-ver
```

Output:

```
build-demo 1.4.2 (darwin/arm64)
```

The real release form stamps three values from git:

```bash
go build -ldflags "
  -s -w
  -X main.version=$(git describe --tags --always)
  -X main.commit=$(git rev-parse --short HEAD)
  -X main.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)
" -o bin/app ./cmd/app
```

Rules for `-X`: the target must be a package-level `var` of type `string`, it
must not be a `const`, the path is the full import path (`main.version`, or
`example.com/app/internal/build.Version`), and if the variable is otherwise
unused the linker may have already eliminated it — give it a default value and
actually print it.

## -gcflags=-m: escape analysis

`-gcflags` passes flags to the compiler instead of the linker. `-m` reports
inlining and escape decisions (the mechanics are 4.1):

```go
package main

import "fmt"

type point struct{ x, y int }

func stack() point {
  p := point{1, 2}
  return p
}

func heap() *point {
  p := point{3, 4}
  return &p
}

func main() {
  fmt.Println(stack(), *heap())
}
```

```bash
go build -gcflags="-m" .
```

Output:

```
# example.com/esc
./main.go:7:6: can inline stack
./main.go:12:6: can inline heap
./main.go:18:20: inlining call to stack
./main.go:18:29: inlining call to heap
./main.go:18:14: inlining call to fmt.Println
./main.go:13:3: moved to heap: p
./main.go:18:14: ... argument does not escape
./main.go:18:20: ~r0 escapes to heap
./main.go:18:24: *(~r0) escapes to heap
```

`moved to heap: p` on line 13 is `heap()`'s local, which must outlive the call
because its address is returned. `stack()`'s `p` is not mentioned: it stays on
the stack. `-gcflags="-m -m"` doubles the verbosity and explains *why*.

Two more worth knowing: `-gcflags="all=-N -l"` disables optimisation and
inlining (what `dlv` needs, 15.7), and `-gcflags="-d=ssa/check_bce/debug=1"`
reports where bounds checks were *not* eliminated.

Note the `all=` prefix: without it the flags apply only to the packages named
on the command line, with it they apply to every package including the stdlib.

## -trimpath

By default the binary embeds absolute source paths, which leaks your home
directory into stack traces and makes builds unreproducible across machines:

```bash
go build -trimpath -o bin/demo-tp .
go version -m bin/demo-tp | grep trimpath
```

Output:

```
	build	-trimpath=true
```

Paths become `example.com/scratch/main.go` instead of
`/Users/raj/.../l1/main.go`. Always on for release builds; 15.5 ties it to
reproducible builds.

## Build modes, briefly

```bash
go help buildmode | head -8
```

Output:

```
The 'go build' and 'go install' commands take a -buildmode argument which
indicates which kind of object file is to be built. Currently supported values
are:

	-buildmode=archive
		Build the listed non-main packages into .a files. Packages named
		main are ignored.

```

The default is `exe` (or `pie` on some platforms — note the `-buildmode=pie`
in the earlier `-x` link line on darwin). The ones worth knowing exist:
`c-archive` and `c-shared` to be called from C, and `plugin` for
`plugin.Open` at runtime. All three are niche, platform-limited, and break
cross-compilation; do not reach for them to build a plugin system when an
interface and a registry will do.

## GOFLAGS

Flags you want on every invocation go in the environment rather than in a
Makefile for every command:

```bash
export GOFLAGS="-trimpath -mod=readonly"
go build ./...
```

`GOFLAGS` is parsed as if typed on the command line and silently ignored by
subcommands that do not accept a given flag. Common contents: `-mod=mod`,
`-mod=vendor`, `-trimpath`, `-count=1` (to defeat test caching), `-tags=...`.
Explicit command-line flags win over `GOFLAGS`.

`-tags` is the one flag here that deserves a lesson of its own — that is 15.4.

## Quick Reference

| Command | Does |
|---|---|
| `go run .` | compile to a temp dir, run, discard |
| `go run ./cmd/app` | same, for a nested main package |
| `go run main.go` | compile only the named files (`command-line-arguments`) |
| `go build .` | binary named after the import path base, in the cwd |
| `go build -o bin/app .` | explicit output path |
| `go build ./...` | compile-check everything; writes **no** binaries |
| `go build ./pkg` | type-check a library, discard the result |
| `go install .` | binary into `$GOBIN` or `$GOPATH/bin` |
| `go install pkg@v1.2.3` | build a tool, ignoring the current `go.mod` |
| `go version -m BIN` | module graph and build flags embedded in a binary |

| Flag | Effect |
|---|---|
| `-a` | rebuild all packages, ignoring cached compilations |
| `-x` | print every command the driver runs |
| `-work` | print `$WORK` and keep it |
| `-n` | print commands without running them |
| `-v` | print package names as they compile |
| `-race` | race detector (11.5) |
| `-trimpath` | strip absolute paths from the binary |
| `-tags a,b` | enable build tags (15.4) |
| `-ldflags "-s -w"` | strip symbol table and DWARF |
| `-ldflags "-X path.Var=val"` | set a string var at link time |
| `-gcflags="-m"` | escape analysis and inlining report |
| `-gcflags="all=-N -l"` | disable optimisation and inlining, for delve |
| `-buildmode=MODE` | exe, pie, archive, c-archive, c-shared, plugin |

| Env | Meaning |
|---|---|
| `GOCACHE` | build cache location |
| `GOMODCACHE` | downloaded module source |
| `GOBIN` | where `go install` puts binaries |
| `GOFLAGS` | flags injected into every `go` command |

## Common Mistakes

**Using `go run main.go` in a project with more than one file in `package
main`.** It compiles only that file and fails on every symbol defined in a
sibling; use `go run .`.

**Expecting `go build ./pkg` to produce something.** Non-main packages are
type-checked and discarded — the only artifact is a cache entry.

**Reaching for `go build -a` when a build looks wrong.** The cache is keyed on
content and is almost never the culprit; `go clean -cache` is the right reset
and `-a` just burns CPU.

**Running `go clean -modcache` to fix one bad dependency.** That re-downloads
every module for every project on the machine; delete the one module's
directory or use `go mod download` for the module you actually care about.

**`-ldflags "-X main.Version=..."` against a `const` or a non-string.** The
linker silently does nothing; the target must be a package-level `var` of type
`string`.

**Shipping release binaries without `-trimpath`.** Your home directory ends up
in panic traces, and two machines produce different bytes from identical source.

## Exercises

1. Build the same main package as `go build .`, `go build ./`, and
   `go build -o x .` from both the package directory and its parent. Record the
   resulting filenames and explain each from the import path.
2. Time a cold and a warm build of a package that imports `net/http`. Then
   change one comment in your own file and time it again — explain why that
   third build is fast.
3. Add `version`, `commit`, and `buildDate` vars to a program, write a shell
   script that stamps all three from git with one `-ldflags`, and verify with
   both `./app -version` and `go version -m app`.
4. Build a binary with and without `-ldflags "-s -w"` and with and without
   `-trimpath`. Record the four sizes and use `go version -m` to show which
   flags are recorded in each.
5. Find a function in your own code that `-gcflags=-m` reports as escaping to
   the heap, change it so it does not, and confirm with the flag plus a
   `-benchmem` benchmark showing fewer allocations.
6. Set `GOFLAGS=-tags=nonexistent` and run `go build`, `go test`, and `go doc`.
   Note which commands accept it and which ignore it.

---

[← 14.8 Test Doubles with Interfaces](../14-testing/08-test-doubles.md) · [Course Index](../README.md) · [15.2 go mod tidy, vendor, and Dependency Management →](02-mod-tidy-vendor.md)
