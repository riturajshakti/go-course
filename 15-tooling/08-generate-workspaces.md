[← 15.7 Delve, Briefly](07-delve.md) · [Course Index](../README.md) · [16.1 Your First net/http Server →](../16-http/01-first-server.md)

# 15.8 — go generate and Workspaces

## What You'll Learn

- `go generate` **never** runs automatically — not on build, not on test, not on vet
- `$GOFILE`, `$GOLINE`, `$GOPACKAGE`, `$DOLLAR`, and a complete working generator
- `go work` replaces a relative `replace` for local multi-module development
- Why `go.work` usually does not get committed

## Coming From Another Language

| Other | Go |
|---|---|
| `npm run build` hooks into every install | `go generate` runs only when you type it |
| codegen as a webpack/vite plugin | a comment in the source, invoked by a separate command |
| generated files in `.gitignore` | generated files are **committed**, marked `DO NOT EDIT` |
| `npm link` / `pnpm workspace:` | `go work use`, and it needs no `go.mod` change at all |
| `cargo` workspace in `Cargo.toml`, committed | `go.work` is local, usually gitignored |
| decorators / metaprogramming at runtime | generation at build time, or generics (9.x), or reflect (13.15) |

## The one fact that matters

`go generate` is **not** part of the build. `go build`, `go test`, `go vet`,
and `go install` never invoke it. Proof — a directive whose only job is to
leave a file behind:

```go
package shapes

//go:generate sh -c "echo ran > out.txt"
```

```bash
rm -f out.txt
go build ./... && go vet ./... && go test ./...
ls out.txt
```

Output:

```
?   	example.com/scratch	[no test files]
ls: out.txt: No such file or directory
```

Nothing ran. The consequence is the convention: **generated code is committed
to the repository**, because a fresh clone must build with no tools installed
and no generation step. `go generate ./...` is something a developer runs after
changing the source of truth, and something CI *verifies* (by regenerating and
checking the diff is empty) rather than performs.

## Directive syntax

```
//go:generate command arg1 arg2
```

No space after `//`, it must be at the start of a line, and it can appear
anywhere in a file — unlike `//go:build` (15.4) it is not restricted to the
header. The command is run with the package directory as the working directory,
and it is **not** a shell: no pipes, no globs, no redirection, no variable
expansion beyond the generator variables below. Use `sh -c "..."` when you need
shell features.

| Variable | Value |
|---|---|
| `$GOFILE` | the file containing the directive |
| `$GOLINE` | its line number |
| `$GOPACKAGE` | the package name |
| `$GOARCH`, `$GOOS` | the target architecture and OS |
| `$GOROOT`, `$GOPATH` | as `go env` reports them |
| `$DOLLAR` | a literal `$`, since `$` itself is substituted |

```go
package shapes

//go:generate echo "GOFILE=$GOFILE GOLINE=$GOLINE GOPACKAGE=$GOPACKAGE GOARCH=$GOARCH literal=$DOLLAR HOME"
//go:generate sh -c "echo generating for $GOPACKAGE from $GOFILE:$GOLINE"
```

```bash
go generate ./...
```

Output:

```
GOFILE=doc.go GOLINE=3 GOPACKAGE=shapes GOARCH=arm64 literal=$ HOME
generating for shapes from doc.go:4
```

`$DOLLAR HOME` printed `$ HOME`, not the home directory — that is the escape
hatch for passing a literal `$` through to a generator.

Three flags you will use:

```bash
go generate -n ./...        # print the commands, run nothing
go generate -x ./...        # print each command as it runs
go generate -run 'stringer' ./...   # only directives matching this regexp
```

```bash
go generate -n ./...
```

Output:

```
echo GOFILE=doc.go GOLINE=3 GOPACKAGE=shapes GOARCH=arm64 literal=$ HOME
sh -c echo generating for shapes from doc.go:4
```

`-n` is how you check what a repo's generation step would do before letting it
write to your tree. `-skip` is the inverse of `-run`.

## A complete real generator

The canonical generation task is a `String()` method for an iota enum. Here is
one written from scratch, using `go/parser` and `go/ast` (13.15's neighbours) —
real code, really run.

The source of truth, carrying the directive:

```go
package main

//go:generate go run gen_enum.go -type=Status -out=status_string.go

type Status int

const (
  StatusPending Status = iota
  StatusActive
  StatusSuspended
  StatusClosed
)
```

The generator, in the same directory but behind an `ignore` tag so it is never
part of the package (15.4):

```go
//go:build ignore

package main

import (
  "flag"
  "fmt"
  "go/ast"
  "go/parser"
  "go/token"
  "os"
  "strings"
)

func main() {
  typeName := flag.String("type", "", "enum type name")
  out := flag.String("out", "", "output file")
  flag.Parse()

  fset := token.NewFileSet()
  pkgs, err := parser.ParseDir(fset, ".", nil, 0)
  if err != nil {
    panic(err)
  }

  var names []string
  for _, pkg := range pkgs {
    for _, f := range pkg.Files {
      for _, d := range f.Decls {
        gd, ok := d.(*ast.GenDecl)
        if !ok || gd.Tok != token.CONST {
          continue
        }
        current := ""
        for _, spec := range gd.Specs {
          vs := spec.(*ast.ValueSpec)
          if id, ok := vs.Type.(*ast.Ident); ok {
            current = id.Name
          }
          if current != *typeName {
            continue
          }
          for _, nm := range vs.Names {
            names = append(names, nm.Name)
          }
        }
      }
    }
  }

  var b strings.Builder
  fmt.Fprintf(&b, "// Code generated by gen_enum.go; DO NOT EDIT.\n\npackage %s\n\n", os.Getenv("GOPACKAGE"))
  fmt.Fprintf(&b, "func (v %s) String() string {\n\tswitch v {\n", *typeName)
  for i, n := range names {
    fmt.Fprintf(&b, "\tcase %d:\n\t\treturn %q\n", i, strings.ToLower(strings.TrimPrefix(n, *typeName)))
  }
  fmt.Fprintf(&b, "\t}\n\treturn \"unknown\"\n}\n")

  if err := os.WriteFile(*out, []byte(b.String()), 0o644); err != nil {
    panic(err)
  }
  fmt.Printf("gen_enum: wrote %s (%d values of %s)\n", *out, len(names), *typeName)
}
```

The `current := ""` carried across specs is the detail that matters: in a
grouped `const` block only the **first** spec has an explicit type, and the
rest inherit it implicitly along with `iota`. A naive version that only looks
at `vs.Type` finds one constant instead of four.

```bash
go generate ./...
```

Output:

```
gen_enum: wrote status_string.go (4 values of Status)
```

```bash
cat status_string.go
```

Output:

```
// Code generated by gen_enum.go; DO NOT EDIT.

package main

func (v Status) String() string {
	switch v {
	case 0:
		return "pending"
	case 1:
		return "active"
	case 2:
		return "suspended"
	case 3:
		return "closed"
	}
	return "unknown"
}
```

And it works — `Status` now satisfies `fmt.Stringer`, so `%s` and `%v` pick it
up automatically (5.6):

```go
package main

import "fmt"

func main() {
  for _, s := range []Status{StatusPending, StatusActive, StatusSuspended, StatusClosed, Status(9)} {
    fmt.Printf("%d=%s ", int(s), s)
  }
  fmt.Println()
}
```

```bash
go run .
```

Output:

```
0=pending 1=active 2=suspended 3=closed 9=unknown 
```

## stringer, the canonical tool

Nobody writes that generator in production; `stringer` already exists and is
better:

```bash
go install golang.org/x/tools/cmd/stringer@latest
```

```go
package shapes

//go:generate stringer -type=Status -linecomment

type Status int

const (
  StatusPending Status = iota // pending
  StatusActive                // active
  StatusClosed                // closed
)
```

```bash
go generate ./...
cat status_string.go
```

Real output:

```
// Code generated by "stringer -type=Status -linecomment"; DO NOT EDIT.

package shapes

import "strconv"

func _() {
	// An "invalid array index" compiler error signifies that the constant values have changed.
	// Re-run the stringer command to generate them again.
	var x [1]struct{}
	_ = x[StatusPending-0]
	_ = x[StatusActive-1]
	_ = x[StatusClosed-2]
}

const _Status_name = "pendingactiveclosed"

var _Status_index = [...]uint8{0, 7, 13, 19}

func (i Status) String() string {
	idx := int(i) - 0
	if i < 0 || idx >= len(_Status_index)-1 {
		return "Status(" + strconv.FormatInt(int64(i), 10) + ")"
	}
	return _Status_name[_Status_index[idx]:_Status_index[idx+1]]
}
```

Two things worth stealing from that output. The strings are one concatenated
constant plus an index table — zero allocations and no switch. And that `func
_()` is a **staleness guard**: if someone inserts a constant into the middle of
the block, the array index goes negative and the *package stops compiling* with
an "invalid array index" error pointing at the generated file. Generated code
that detects its own staleness is the difference between generation you can
trust and generation that silently drifts.

`-linecomment` is what made the names lowercase from the trailing comments.
Other widely used generators in the same shape: `mockgen` (test doubles, 14.8),
`protoc-gen-go`, `sqlc`, `go-enum`, `wire`.

## Conventions

**Mark it.** The first line of every generated file must match this regexp, per
the official convention:

```
^// Code generated .* DO NOT EDIT\.$
```

Tooling relies on it: `gopls` suppresses some diagnostics, code review tools
collapse the diff, `golangci-lint` skips it by default (15.3), and coverage
reports can exclude it.

**Commit it.** A fresh clone must build. The CI check is regeneration, not
generation:

```bash
go generate ./...
git diff --exit-code || { echo "generated code is stale; run go generate ./..."; exit 1; }
```

**Pin the generator.** An unpinned `@latest` means two developers generate
different output from the same source. Use the `tool` directive (15.2) and
`go tool`:

```bash
go get -tool golang.org/x/tools/cmd/stringer
```

```go
//go:generate go tool stringer -type=Status
```

**Put a `//go:generate` on the type, not in a Makefile.** The directive sits
next to the thing it generates from, so it is obvious what regenerating
touches.

## Generation vs reflection vs generics

- **Generics** (9.5): when the *shape* is the same and only the type varies.
  Compile-time safe, zero cost, no files. First choice when it fits.
- **Reflection** (13.15): when the types are not knowable at compile time at
  all — a JSON decoder, a struct-tag-driven validator. Flexible, slow,
  errors surface at runtime.
- **Generation**: when you need code generics cannot express — a `String()`
  method per enum, a mock implementing an interface, a typed accessor per
  struct field, protobuf messages. Zero runtime cost, real stack traces, and
  you can read the output.

The usable rule: reach for generics first, generation when generics cannot
express it, reflection only when neither can.

## Workspaces: the problem

Back to 15.2's two modules. `app` imports `greet`, neither is published:

```
ws/
  greet/go.mod        module example.com/greet
  greet/greet.go
  app/go.mod          module example.com/app
  app/main.go
```

```go
package greet

import "fmt"

func Hello(name string) string {
  return fmt.Sprintf("hello, %s", name)
}
```

```go
package main

import (
  "fmt"

  "example.com/greet"
)

func main() {
  fmt.Println(greet.Hello("workspaces"))
}
```

```bash
cd app && go build .
```

This is an intentional failure. Output:

```
main.go:6:3: missing go.sum entry for module providing package example.com/greet (imported by example.com/app); to add:
	go get example.com/app
```

15.2's answer was a `require` plus a relative `replace`. It works, and it has
one fatal flaw: **it lives in a committed file**. Push
`replace example.com/greet => ../greet` and every CI job and every teammate
whose directory layout differs is broken. The workspace is the same
redirection, stored outside the repository.

## go work

```bash
go work init ./greet ./app
cat go.work
```

Output:

```
go 1.27.1

use (
	./app
	./greet
)
```

```bash
cd app && go run .
```

Output:

```
hello, workspaces
```

```bash
cat app/go.mod
```

Output:

```
module example.com/app

go 1.27.1
```

That is the headline: **no `require`, no `replace`, nothing.** `app/go.mod` is
completely untouched. A workspace works by overriding module resolution for the
`go` command itself — every module in `use` is resolved from its directory,
ahead of anything `go.mod` says. Compare with the `replace` approach, where
`app/go.mod` must carry both a placeholder `require example.com/greet v0.0.0`
and the `replace` line, both of which you must remember to remove before
publishing.

Adding a third module is one command:

```bash
go work use ./logx
cat go.work
```

Output:

```
go 1.27.1

use (
	./app
	./greet
	./logx
)
```

```go
package main

import (
  "example.com/greet"
  "example.com/logx"
)

func main() {
  logx.Log(greet.Hello("workspaces"))
}
```

```bash
go run .
```

Output:

```
[log] hello, workspaces
```

`app` now imports `logx` with no requirement recorded anywhere. That is
convenient and it is also the trap: the module does not build outside the
workspace.

```bash
GOWORK=off go build .
```

This is an intentional failure. Output:

```
main.go:4:3: no required module provides package example.com/greet; to add it:
	go get example.com/greet
main.go:5:3: no required module provides package example.com/logx; to add it:
	go get example.com/logx
```

`GOWORK=off` disables the workspace for one command, which is the check to run
before you push: it tells you whether each module's own `go.mod` is still
complete. Put it in CI — a workspace that hides a missing requirement is
exactly the failure it is supposed to prevent.

```bash
go work use -r .        # recursively add every module under a directory
go work edit -dropuse=./logx
go work edit -use=./logx
go work edit -go=1.27
go work edit -replace=example.com/x=./x
go work edit -json
```

`go work edit -json` dumps the parsed file:

```
{
	"Go": "1.27.1",
	"Use": [
		{"DiskPath": "./app"},
		{"DiskPath": "./greet"},
		{"DiskPath": "./logx"}
	]
}
```

Inside a workspace, module-aware commands see the union:

```bash
go list -m all
```

Output:

```
example.com/app
example.com/greet
example.com/logx
github.com/go-chi/chi/v5 v5.5.0
```

One thing that does **not** work from the workspace root:

```bash
go build ./...
```

This is an intentional failure. Output:

```
pattern ./...: directory prefix . does not contain modules listed in go.work or their selected dependencies
```

`./...` is relative to a module, and the workspace root is not one. Use
`go build ./app/... ./greet/...`, or `go build all` for the whole workspace, or
just `cd` into a module.

## go work sync and go.work.sum

`go work sync` pushes the workspace's *resolved* build list back into each
member's `go.mod`, so the modules agree on dependency versions when built
alone. It is the "I am done, make this real" command.

Honest result here: on these modules it does nothing useful, and said so
clearly in the attempt. `go work sync` resolves each `require` through the
proxy, and `example.com/greet` is not published, so with a placeholder
`require example.com/greet v0.0.0` left in `app/go.mod` it fails:

```
go: example.com/greet@v0.0.0: unrecognized import path "example.com/greet": reading https://example.com/greet?go-get=1: 404 Not Found
```

Once those placeholder requires are dropped — which is the whole point of using
a workspace instead of `replace` — `go work sync` exits 0 silently, because
there are no shared external versions to reconcile. It earns its keep in a
workspace of several modules that each depend on the same third-party library
at different versions.

`go.work.sum` holds checksums for modules the workspace resolves that are not
in any member's `go.sum` — the workspace's own `go.sum`. It is only created
when such a module exists; none of the runs above produced one, since every
external dependency (`chi`) was already recorded in a member's `go.sum`.

## Should go.work be committed?

**Usually no.** Add it to `.gitignore`:

```
go.work
go.work.sum
```

The reasons: it encodes *your* directory layout, which differs from everyone
else's; it silently masks missing requirements in member `go.mod` files, so CI
passes and a consumer's build fails; and the thing a published module must get
right is its own `go.mod`, which the workspace bypasses by design. CI should
build with `GOWORK=off`.

**Commit it when** the repository *is* the whole world: a monorepo containing
every module, where there is no "building one module alone", and where a
checked-in `go.work` is a convenience that saves every developer the same `go
work init`. Then use relative paths only, never absolute, and still run a
`GOWORK=off` job for any module you publish.

| Approach | `go.mod` touched | Committed | Use for |
|---|---|---|---|
| `replace` with a relative path | yes (`require` + `replace`) | yes, and that is the problem | nothing; it is the legacy answer |
| `replace` to a fork | yes | yes, deliberately | pinning around an upstream bug |
| `go work` | no | usually not | local multi-module development |

## Quick Reference

| Command | Does |
|---|---|
| `go generate ./...` | run every directive in every package |
| `go generate -n ./...` | print the commands, run nothing |
| `go generate -x ./...` | print each command as it runs |
| `go generate -run RE ./...` | only matching directives |
| `go generate -skip RE ./...` | skip matching directives |

| Directive variable | Value |
|---|---|
| `$GOFILE` | the containing file |
| `$GOLINE` | the directive's line |
| `$GOPACKAGE` | the package name |
| `$GOOS`, `$GOARCH` | target platform |
| `$DOLLAR` | a literal `$` |

| Command | Does |
|---|---|
| `go work init [dirs...]` | create `go.work` |
| `go work use DIR` | add a module |
| `go work use -r DIR` | add every module under DIR |
| `go work edit -dropuse=DIR` | remove one |
| `go work edit -replace=A=B` | a workspace-scoped replace |
| `go work edit -json` | dump the parsed file |
| `go work sync` | push resolved versions into member `go.mod`s |
| `go work vendor` | vendor the whole workspace |
| `GOWORK=off go build ./...` | ignore the workspace |
| `go env GOWORK` | which `go.work` is active |

## Common Mistakes

**Assuming `go generate` runs during the build.** It never does; generated code
must be committed, and CI should verify it with `git diff --exit-code`.

**Omitting the `// Code generated ... DO NOT EDIT.` line.** Linters and review
tools then treat the file as hand-written, and someone eventually edits it.

**Using `@latest` for a generator.** Two developers produce different output
from the same source; pin it with `go get -tool` and invoke via `go tool`.

**Putting a generator in the package without an `ignore` build tag.** A second
`func main` in `package main`, or a dependency on `go/ast` in your production
binary.

**Committing `go.work`.** It encodes your directory layout and hides missing
requirements in member `go.mod` files, so CI goes green and a consumer's build
fails. Gitignore it and run CI with `GOWORK=off`.

**Running `go build ./...` from the workspace root.** `./...` is
module-relative; the root is not a module. Use `go build all` or name the
member directories.

## Exercises

1. Write a generator that emits a `MarshalJSON` method for an iota enum,
   producing the lowercase name as a JSON string. Include a staleness guard in
   the style of stringer's `func _()` and prove it fires when you insert a
   constant.
2. Add a `//go:generate` directive using `$GOFILE` and `$GOLINE` to write a
   comment naming its own source location into the generated file. Confirm with
   `go generate -n` that the substitution happens before the command runs.
3. Set up the CI staleness check (`go generate ./...` then
   `git diff --exit-code`). Verify it fails when you edit the source of truth
   without regenerating, and passes afterwards.
4. Convert the `replace`-based two-module setup from 15.2 to a workspace. Diff
   both `go.mod` files before and after, and confirm with `GOWORK=off` that the
   modules still build standalone.
5. Build a three-module workspace where two members require *different* minor
   versions of the same third-party library. Run `go work sync` and record what
   it wrote into each `go.mod`.
6. Add a package import across workspace members without adding the `require`.
   Confirm it builds in the workspace and fails under `GOWORK=off`, then write
   the CI step that catches it.

---

[← 15.7 Delve, Briefly](07-delve.md) · [Course Index](../README.md) · [16.1 Your First net/http Server →](../16-http/01-first-server.md)
