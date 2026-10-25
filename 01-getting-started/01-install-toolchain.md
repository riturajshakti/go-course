[Course Index](../README.md) · [1.2 Hello, Modules: Your First Project →](02-hello-modules.md)

# 1.1 — Installing Go and the Toolchain

## What You'll Learn

- The `go` command is the whole toolchain: compiler, formatter, test runner, dependency manager, no config files
- `go env` is the single source of truth for where things live — GOROOT, GOMODCACHE, GOBIN
- GOTOOLCHAIN makes the installed `go` download and run a *different* Go version when a module asks for one
- Installed binaries land in `GOBIN` (default `$GOPATH/bin`), which you must add to PATH yourself

## Coming From Another Language

| Other | Go |
|---|---|
| `node` + `npm` + `prettier` + `eslint` + `jest` as five tools | one `go` binary does all of it |
| `nvm use 20` to switch runtime version | `go` reads `go.mod` and switches itself (GOTOOLCHAIN) |
| `pip install -g`, `npm -g` → version-specific dirs | `go install` → one `$GOPATH/bin`, always |
| `rustup component add rustfmt` | `gofmt` ships in the box, non-optional |
| `python --version` varies per venv | the toolchain version is a property of the *module*, not the shell |

## What is installed

```bash
go version
```

Output:

```
go version go1.27.1 darwin/arm64
```

Everything in this course was run against exactly this. The `go` binary is ~70 MB
and self-contained; there is no separate compiler, linker, or package manager to
install.

## Install routes

Pick one. They all give you the same `go` binary.

```bash
brew install go

go version
which -a go gofmt
```

Output:

```
go version go1.27.1 darwin/arm64
/opt/homebrew/bin/go
/opt/homebrew/bin/gofmt
```

Homebrew symlinks into `/opt/homebrew/bin`, with the real tree under
`/opt/homebrew/Cellar/go/1.27.1/libexec`. On Linux, or when you want a version
Homebrew does not carry, use the official tarball:

```bash
curl -LO https://go.dev/dl/go1.27.1.darwin-arm64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.27.1.darwin-arm64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

The tarball always extracts to a directory literally named `go`, which is why the
`rm -rf` comes first — untarring over an old tree leaves stale files behind. Do
not use your distro's `apt install golang`; those packages lag badly and
sometimes patch the module behaviour.

The third route is installing a toolchain *with* Go, once you already have any Go
≥ 1.21:

```bash
go install golang.org/dl/go1.26.0@latest
go1.26.0 download
go1.26.0 version
```

This gives you a `go1.26.0` shim command alongside your main `go`. In practice
GOTOOLCHAIN (below) has made this route mostly unnecessary.

## go env: where everything lives

```bash
go env GOROOT GOPATH GOMODCACHE GOOS GOARCH GOTOOLCHAIN GOBIN CGO_ENABLED
```

Output:

```
/opt/homebrew/Cellar/go/1.27.1/libexec
/Users/raj/go
/Users/raj/go/pkg/mod
darwin
arm64
auto
/Users/raj/go/bin
1
```

| Var | Means | You should |
|---|---|---|
| `GOROOT` | where the toolchain and stdlib source live | never set it; the binary finds itself |
| `GOPATH` | the one shared dir for the module cache and installed binaries | leave at `~/go`; it is **not** where your code goes any more |
| `GOMODCACHE` | downloaded, read-only module sources | know it exists for `go clean -modcache` |
| `GOOS` / `GOARCH` | build target | override per-command to cross-compile |
| `GOTOOLCHAIN` | toolchain switching policy | leave at `auto` |
| `GOBIN` | where `go install` puts binaries | add it to PATH |
| `CGO_ENABLED` | link against libc | set `0` for truly static Linux builds |

GOPATH is the biggest trap for anyone who read a pre-2019 Go tutorial. In the
module era your project lives wherever you want. `GOPATH` is now just a cache
directory plus a bin directory.

`go env` with no args prints all ~60 variables. `go env -w KEY=value` writes a
persistent override to a config file rather than your shell profile:

```bash
go env GOENV
```

Output:

```
/Users/raj/Library/Application Support/go/env
```

## Binaries and PATH

`go install` compiles a `main` package and drops the executable in `GOBIN`:

```bash
go install golang.org/x/example/hello@latest
ls -l $(go env GOBIN)/hello
```

Output:

```
go: downloading golang.org/x/example/hello v0.0.0-20250915201037-7f05d217867b
go: downloading golang.org/x/example v0.0.0-20250915201037-7f05d217867b
-rwxr-xr-x@ 1 raj  staff  2571538 Oct 25 23:26 /Users/raj/go/bin/hello
```

Nothing adds that directory to PATH for you. Add this to your shell profile:

```bash
export PATH=$PATH:$(go env GOPATH)/bin
```

The `name@version` form is how you install Go CLI tools. It ignores the current
module entirely — the tool's own `go.mod` decides its dependencies, so installing
a linter cannot perturb your project. `@latest` resolves the newest release tag.

## GOTOOLCHAIN: the go command switches Go versions for you

This one is genuinely Go-specific and surprises everyone. Since 1.21, a `go.mod`
declaring `go 1.30.0` does not fail against an older `go` binary — the installed
`go` downloads the requested toolchain from the module proxy, caches it in
`GOMODCACHE`, and re-executes itself under it. The `go` on your PATH is a
bootstrap, not a pin. That is why `GOTOOLCHAIN=auto` is the default.

Watch it actually happen:

```bash
GOTOOLCHAIN=go1.26.0 go version
```

Output:

```
go: downloading go1.26.0 (darwin/arm64)
go version go1.26.0 darwin/arm64
```

The downloaded toolchain is an ordinary module:

```bash
ls -d $(go env GOMODCACHE)/golang.org/toolchain@*
```

Output:

```
/Users/raj/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.26.0.darwin-arm64
```

A module can only push the toolchain *up*. A `go.mod` asking for a version older
than your local `go` is satisfied by the local one, which is why this prints
1.27.1 even with `toolchain go1.26.0` in the file:

```bash
cat go.mod
go run .
```

Output:

```
module example.com/tc2

go 1.27.1

toolchain go1.26.0
go1.27.1
```

Set `GOTOOLCHAIN=local` to forbid switching entirely. You then get a hard error
instead of a download — useful in CI and air-gapped builds where you want the
pinned image's Go and nothing else:

```bash
GOTOOLCHAIN=local go build .
```

This fails on purpose, against a module whose `go.mod` says `go 1.30.0`. Output:

```
go: go.mod requires go >= 1.30.0 (running go 1.27.1; GOTOOLCHAIN=local)
```

With the default `auto`, the same module tries the download and fails only
because that version does not exist:

```bash
go build .
```

Output:

```
go: downloading go1.30.0 (darwin/arm64)
go: download go1.30.0 for darwin/arm64: toolchain not available
```

Idiomatic: leave `GOTOOLCHAIN=auto` locally, set `GOTOOLCHAIN=local` in CI, and
pin the real version with the `go` and `toolchain` directives in `go.mod`
(lesson 1.3).

One sharp edge: once a `go.mod` names an unavailable toolchain, *every* `go`
command in that directory fails — including `go mod edit`, which you need to fix
it. Prefix with `GOTOOLCHAIN=local` to get out.

## The subcommands

```bash
go help
```

Output:

```
Go is a tool for managing Go source code.

Usage:

	go <command> [arguments]

The commands are:

	bug         start a bug report
	build       compile packages and dependencies
	clean       remove object files and cached files
	doc         show documentation for package or symbol
	env         print Go environment information
	fix         apply fixes suggested by static checkers
	fmt         gofmt (reformat) package sources
	generate    generate Go files by processing source
	get         add dependencies to current module and install them
	install     compile and install packages and dependencies
	list        list packages or modules
	mod         module maintenance
	run         compile and run Go program
	telemetry   manage telemetry data and settings
	test        test packages
	tool        run specified go tool
	version     print Go version
	vet         report likely mistakes in packages
	work        workspace maintenance

Use "go help <command>" for more information about a command.

Additional help topics:

	buildconstraint build constraints
	buildjson       build -json encoding
	buildmode       build modes
	c               calling between Go and C
	cache           build and test caching
	environment     environment variables
	filetype        file types
	goauth          GOAUTH environment variable
	go.mod          the go.mod file
	gopath          GOPATH environment variable
	goproxy         module proxy protocol
	importpath      import path syntax
	modules         modules, module versions, and more
	module-auth     module authentication using go.sum
	packages        package lists and patterns
	private         configuration for downloading non-public code
	testflag        testing flags
	testfunc        testing functions
	vcs             controlling version control with GOVCS

Use "go help <topic>" for more information about that topic.
```

That is the entire surface. There is no `go lint`, no `go format` (it is `go fmt`,
a thin wrapper over the separate `gofmt` binary), and no plugin system. The
`go help <topic>` entries are real reference documentation — `go help modules` and
`go help importpath` are worth reading once.

## What a built binary remembers

Go stamps build metadata into the executable, readable without running it:

```bash
go version -m hello
```

Output:

```
hello: go1.27.1
	path	example.com/hello
	mod	example.com/hello	(devel)	
	build	-buildmode=exe
	build	-compiler=gc
	build	CGO_ENABLED=1
	build	CGO_CFLAGS=
	build	CGO_CPPFLAGS=
	build	CGO_CXXFLAGS=
	build	CGO_LDFLAGS=
	build	GOARCH=arm64
	build	GOOS=darwin
	build	GOARM64=v8.0
```

For a binary with dependencies this also lists every module and its `h1:` hash.
`(devel)` means built from a module with no version tag. This is how you audit a
deployed binary for a vulnerable dependency without access to its source — and
what `govulncheck` reads.

## Verifying the install from Go itself

```go
package main

import (
  "fmt"
  "runtime"
)

func main() {
  fmt.Println(runtime.Version(), runtime.GOOS, runtime.GOARCH, runtime.NumCPU())
}
```

Output:

```
go1.27.1 darwin arm64 12
```

`runtime.Version()` reports the toolchain that *compiled* this binary, so it will
differ from `go version` on your PATH if GOTOOLCHAIN switched.

## Quick Reference

| Command | Does |
|---|---|
| `go version` | toolchain version and platform |
| `go version -m BINARY` | build info and dependency hashes baked into a binary |
| `go env` | all environment variables |
| `go env VAR1 VAR2` | just those, one per line |
| `go env -w VAR=value` | persist an override to the go env file |
| `go env -u VAR` | remove a persisted override |
| `go install pkg@version` | build and install a tool into GOBIN, ignoring the current module |
| `go clean -cache` | wipe the build cache |
| `go clean -modcache` | wipe downloaded module sources |
| `go help <topic>` | real reference docs, offline |

| Variable | Default here | Effect |
|---|---|---|
| `GOTOOLCHAIN=auto` | yes | download and switch to the version `go.mod` asks for |
| `GOTOOLCHAIN=local` | | never switch; error instead |
| `GOTOOLCHAIN=go1.26.0` | | always use that version |
| `GOBIN` | `~/go/bin` | `go install` destination |
| `GOFLAGS` | empty | flags implicitly added to every `go` command |
| `GOPROXY` | `https://proxy.golang.org,direct` | where modules are fetched from |
| `GOPRIVATE` | empty | module path globs to fetch direct, skipping proxy and sumdb |

## Common Mistakes

**Putting your project under `~/go/src` because an old tutorial said to.** That
was GOPATH mode; with modules your code lives anywhere, and `~/go` is only a
cache.

**Forgetting to add `$(go env GOPATH)/bin` to PATH, then concluding `go install`
is broken.** The binary is there; your shell just cannot see it.

**Setting `GOROOT` by hand.** The `go` binary locates its own stdlib; an
explicit `GOROOT` pointing at an old install produces baffling version mismatches.

**Assuming `go version` tells you which compiler built your project.** With
`GOTOOLCHAIN=auto` the module's `go` directive can silently select a different
one; check `runtime.Version()` or `go version -m`.

**Running `go install ./...` expecting it to install dependencies.** `go install`
installs *binaries*; dependencies are resolved automatically by `go build` and
recorded by `go mod tidy`.

## Exercises

1. Print every `go env` variable, then diff it against `GOTOOLCHAIN=local go env`
   to find which values are toolchain-dependent.
2. Create a module whose `go.mod` says `go 1.26.0`, build it, and use
   `go version -m` on the result to confirm which toolchain compiled it. Repeat
   with `GOTOOLCHAIN=local` and explain the difference.
3. Set `GOFLAGS=-mod=readonly` with `go env -w`, then try `go get` on a new
   dependency. Capture the error and explain what `-mod=readonly` protects.
4. Cross-compile the `runtime.Version()` program for `linux/amd64` and
   `windows/amd64` without installing anything, then confirm the targets with
   `file`.
5. Wipe the build cache with `go clean -cache` and time a cold vs warm
   `go build` of a program importing `net/http`. Report both numbers.

---

[Course Index](../README.md) · [1.2 Hello, Modules: Your First Project →](02-hello-modules.md)
