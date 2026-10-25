[← 1.1 Installing Go and the Toolchain](01-install-toolchain.md) · [Course Index](../README.md) · [1.3 go.mod, go.sum, and the Module Graph →](03-go-mod.md)

# 1.2 — Hello, Modules: Your First Project

## What You'll Learn

- `go mod init <path>` creates the two-line file that makes a directory a module
- `go run .` builds the whole package; `go run main.go` builds only that file and breaks the moment you add a second
- `go build` leaves a binary named after the module's last path element; `go run` throws it away; `go install` puts it in GOBIN
- The real compiler errors for a non-`main` package, a missing `main` function, an unused import, and a missing `go.mod`

## Coming From Another Language

| Other | Go |
|---|---|
| `npm init` writes 20 fields | `go mod init` writes two lines |
| `node index.js` interprets one file | `go run .` compiles the whole package first |
| `package.json` `"name": "myapp"` | the module path is a URL-shaped import prefix, not a display name |
| `__pycache__`, `node_modules` in your tree | nothing; artifacts live in a global cache |
| `python -c` for a scratch script | `go run .` is already a 90 ms round trip |

## go mod init

The argument is a *module path*: the string other code would import you by. It is
conventionally a repo URL without the scheme, because that is what makes
`go get github.com/you/thing` work. Nothing resolves it for a private program, so
`example.com/anything` is fine for scratch work.

```bash
mkdir greeter && cd greeter
go mod init example.com/greeter
```

Output:

```
go: creating new go.mod: module example.com/greeter
```

Omit the argument and Go tries to infer the path from a git remote. Outside a
repo it refuses rather than guessing:

```bash
go mod init
```

This fails on purpose. Output:

```
go: cannot determine module path for source directory /tmp/scratch (outside GOPATH, module path must be specified)
```

That is the whole setup. One file now exists:

```bash
cat go.mod
```

Output:

```
module example.com/greeter

go 1.27.1
```

The `go` directive is the minimum language version this module requires, and as
lesson 1.1 showed, it is also what drives toolchain switching.

## Hello world

```go
package main

import "fmt"

func main() {
  fmt.Println("hello, modules")
}
```

```bash
go run .
```

Output:

```
hello, modules
```

Three non-negotiables are visible here: the package must be called `main`, the
entry point must be `func main()` taking and returning nothing, and the import
must be used. Go has no command-line arg in `main`'s signature — `os.Args` serves
that role.

```bash
find . -type f | sort
```

Output:

```
./go.mod
./main.go
```

Two files. No lockfile yet (that is `go.sum`, and it appears only when you add a
dependency), no build directory, no config.

## go run . vs go run main.go

`go run .` means "the package in this directory". `go run main.go` means "a
synthetic package made of exactly these files". With one file they agree. Add a
second and they diverge:

```go
package main

func tag() string { return "v1" }
```

```go
package main

import "fmt"

func main() {
  fmt.Println("hello,", tag())
}
```

```bash
go run .
```

Output:

```
hello, v1
```

```bash
go run main.go
```

This fails on purpose. Output:

```
# command-line-arguments
./main.go:6:24: undefined: tag
```

`command-line-arguments` is the fake package name Go uses for a file list. You
can name every file and it works again, but there is no reason to:

```bash
go run main.go tag.go
```

Output:

```
hello, v1
```

Idiomatic: always `go run .` (or `go run ./cmd/api`). Treat `go run file.go` as a
mistake — it is the single most common cause of "undefined: X" in a beginner's
first multi-file program.

Arguments after the package go to your program, not to `go`:

```go
package main

import (
  "fmt"
  "os"
)

func main() {
  fmt.Println(os.Args)
}
```

```bash
go run . alpha beta
```

Output:

```
[/var/folders/3l/gf781gh95p75bmw55xppgfhh0000gn/T/go-build955524156/b001/exe/greeter alpha beta]
```

`os.Args[0]` is the temporary binary `go run` built and will delete. That path
changes every run — never depend on it.

## build, run, install

```bash
go build
ls -1
./greeter
```

Output:

```
go.mod
greeter
main.go
hello, modules
```

`go build` with no args builds the current package and, because it is `main`,
writes an executable named after the **last element of the module path** — not
after `main.go`. Override with `-o`:

```bash
go build -o bin/greeter .
./bin/greeter
```

Output:

```
hello, modules
```

`-o /dev/null` is the idiom for "just typecheck and compile, keep nothing".

```bash
go install .
ls -l $(go env GOBIN)/greeter
```

Output:

```
-rwxr-xr-x@ 1 raj  staff  2429714 Oct 25 23:22 /Users/raj/go/bin/greeter
```

| Command | Compiles | Writes the binary to | Runs it |
|---|---|---|---|
| `go run .` | yes | a temp dir, then deletes it | yes |
| `go build .` | yes | `./<last path element>` | no |
| `go build -o X .` | yes | `X` | no |
| `go build -o /dev/null .` | yes | nowhere | no |
| `go install .` | yes | `$GOBIN` | no |
| `go vet ./...` | typechecks only | nowhere | no |

All of them share one build cache, so the second invocation of any of them is
near-instant. Add the built binary name to `.gitignore`; Go will not do it for
you and `greeter` looks like a source directory to a reviewer.

## The errors you will hit in the first ten minutes

Each of these was compiled to capture the real text.

A file whose package is not `main`:

```go
package greeting

import "fmt"

func main() {
  fmt.Println("hi")
}
```

This does not compile. Output:

```
package example.com/wrongpkg is not a main package
```

Note it comes from `go run .`. The same file via `go run greet.go` names the fake
package instead:

```
package command-line-arguments is not a main package
```

A `main` package with no `main` function. This one gets past the typechecker and
fails at link time, which is why the error mentions `runtime`:

```go
package main

import "fmt"

func greet() {
  fmt.Println("hi")
}
```

This does not compile. Output:

```
# example.com/nomain
runtime.main_main·f: function main is undeclared in the main package
```

An unused import and an unused local. Both are hard compile errors, not warnings
— there is no flag to downgrade them:

```go
package main

import (
  "fmt"
  "os"
)

func main() {
  x := 1
  fmt.Println("hi")
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:5:3: "os" imported and not used
./main.go:9:3: declared and not used: x
```

Coming from JS or Python this feels hostile for about a week, then you realise it
means dead code never accumulates. The escape hatches are `_ = x` for a value and
`_ "path"` for an import you want only for its side effects (lesson 1.4).

Unused *package-level* variables and unused function parameters are both fine —
the rule applies only to imports and function-local variables.

Running outside a module:

```bash
go run .
```

This fails on purpose, in a directory with `main.go` but no `go.mod`. Output:

```
go: go.mod file not found in current directory or any parent directory; see 'go help modules'
```

"or any parent directory" matters: the `go` command walks up looking for
`go.mod`, so a subdirectory of your module needs no file of its own.

## Quick Reference

| Command | Does |
|---|---|
| `go mod init <path>` | create `go.mod` with that module path |
| `go run .` | build and run the package in this directory |
| `go run ./cmd/api` | build and run that package |
| `go run . args...` | pass `args` to your program as `os.Args[1:]` |
| `go build` | binary named after the module path's last element |
| `go build -o NAME .` | binary at `NAME` |
| `go build -o /dev/null .` | compile-check, keep nothing |
| `go install .` | binary into `$GOBIN` |
| `go clean` | remove binaries `go build` left in this directory |

| Requirement | Rule |
|---|---|
| executable package | must be `package main` |
| entry point | `func main()`, no params, no results |
| imports | every one must be used, or blanked with `_` |
| local variables | every one must be read, or assigned to `_` |
| module root | a `go.mod` in this dir or any ancestor |

## Common Mistakes

**Running `go run main.go` in a multi-file package.** It compiles only that file
and reports your other functions as undefined; use `go run .`.

**Expecting `go build` to produce `main` or `main.exe`.** The name comes from the
last element of the module path; pass `-o` when you care.

**Committing the binary `go build` dropped next to your source.** Add it to
`.gitignore` on day one, or build with `-o bin/` and ignore `bin/`.

**Leaving a debug import in place and fighting the compiler.** Unused imports are
errors by design — delete it, or write `_ "net/http/pprof"` if you genuinely want
the side effect.

**Giving the module a bare name like `greeter`.** It compiles, but the moment
anyone wants to import it the path is unfetchable; use a URL-shaped path such as
`github.com/you/greeter`.

## Exercises

1. Create a module, split `main` across three files in the same directory, and
   confirm `go run .` works while `go run main.go` fails. Capture both outputs.
2. Make `go build` produce a binary named `api` from a module path ending in
   `service`, two different ways.
3. Write a program that prints `os.Args` and run it via `go run .`, then via
   `go build` plus `./binary`, with the same trailing arguments. Explain why
   `os.Args[0]` differs.
4. Deliberately trigger all four errors from this lesson and record the exact
   text. Then fix each one with the minimum edit.
5. Create a subdirectory inside your module with its own `main.go` under
   `cmd/tool/`, and run it without adding a second `go.mod`. What is the binary
   called when you `go build ./cmd/tool`?

---

[← 1.1 Installing Go and the Toolchain](01-install-toolchain.md) · [Course Index](../README.md) · [1.3 go.mod, go.sum, and the Module Graph →](03-go-mod.md)
