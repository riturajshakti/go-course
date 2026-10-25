[← 1.2 Hello, Modules: Your First Project](02-hello-modules.md) · [Course Index](../README.md) · [1.4 Packages, Imports, and Visibility →](04-packages-imports.md)

# 1.3 — go.mod, go.sum, and the Module Graph

## What You'll Learn

- All six `go.mod` directives: `module`, `go`, `toolchain`, `require`, `replace`, `exclude`, `retract`
- `go.sum` is not a lockfile — `go.mod` is; `go.sum` is a checksum ledger, and tampering produces a `SECURITY ERROR`
- `// indirect` marks a requirement no package of yours imports directly
- Minimal version selection: you get the *oldest* version that satisfies everyone, so builds are reproducible without a lockfile
- Major version ≥ 2 goes in the import path (`/v2`), and the compiler enforces it

## Coming From Another Language

| Other | Go |
|---|---|
| npm resolves the *newest* matching semver range | MVS resolves the *oldest* version that satisfies all requirements |
| `package-lock.json` pins the tree | `go.mod` already pins it; `go.sum` only verifies bytes |
| `^1.2.0` ranges | exact versions only — no range syntax exists |
| `pip install -U` mutates your env | `go get pkg@version` edits `go.mod`, nothing global |
| Cargo: `semver` major bump is metadata | Go: major bump changes the **import path** |
| `node_modules/` per project | one read-only `$GOMODCACHE` shared by every project |

## Adding a dependency

`go mod tidy` reads your imports, finds the modules that provide them, and writes
both files. You never hand-edit `require` lines in practice.

```go
package main

import (
  "fmt"

  "github.com/go-chi/chi/v5"
)

func main() {
  r := chi.NewRouter()
  fmt.Printf("%T\n", r)
}
```

```bash
go mod tidy
```

Output:

```
go: finding module for package github.com/go-chi/chi/v5
go: downloading github.com/go-chi/chi/v5 v5.5.0
go: downloading github.com/go-chi/chi v1.5.5
go: found github.com/go-chi/chi/v5 in github.com/go-chi/chi/v5 v5.5.0
```

It downloaded `chi` v1 as well — only its `go.mod`, to confirm v1 does not also
claim that import path. Resolution needs the graph, not just the code.

```bash
cat go.mod
```

Output:

```
module example.com/modtour

go 1.27.1

require github.com/go-chi/chi/v5 v5.5.0
```

```bash
go run .
```

Output:

```
*chi.Mux
```

## go.sum: what is actually in it

```bash
cat go.sum
```

Output:

```
github.com/go-chi/chi/v5 v5.5.0 h1:xdKQ1dJmVJsmgahrXSKhUx8NyVPzpwdlyrrQFpMPJgg=
github.com/go-chi/chi/v5 v5.5.0/go.mod h1:d1IG7GtKPSjPyBP365PH8sAGVHk3qfB/IhjFNRH1wWM=
```

Two lines per module version. The first is the hash of the *source zip*; the
`/go.mod` one is the hash of just that module's `go.mod`. The second exists so
the graph can be computed from `go.mod` files alone, verified, without
downloading any source.

`h1:` is the algorithm tag (SHA-256 over a sorted file manifest), not a version.
Entries accumulate and are never garbage-collected automatically — `go mod tidy`
is what prunes them, and entries for versions you no longer use but that still
appear in the graph legitimately remain.

Corrupt one hash and the build stops hard:

```bash
go build .
```

This fails on purpose, after editing one `h1:` line in `go.sum`. Output:

```
go: downloading github.com/jackc/pgx/v5 v5.11.0
go: downloading github.com/jackc/pgpassfile v1.0.0
verifying github.com/jackc/pgpassfile@v1.0.0: checksum mismatch
	downloaded: h1:/6Hmqy13Ss2zCq62VdNG8tM1wchn8zjSGOBJ6icpsIM=
	go.sum:     h1:AAAAy13Ss2zCq62VdNG8tM1wchn8zjSGOBJ6icpsIM=

SECURITY ERROR
This download does NOT match an earlier download recorded in go.sum.
The bits may have been replaced on the origin server, or an attacker may
have intercepted the download attempt.
```

So `go.sum` must be committed. Deleting it does not break the build — Go
re-fetches and re-verifies against the public checksum database at
`sum.golang.org` — but it discards your project's own record of what it has seen
before.

## Indirect requirements

A dependency with its own dependencies produces a second `require` block:

```go
package main

import (
  "fmt"

  "github.com/jackc/pgx/v5"
)

func main() {
  cfg, err := pgx.ParseConfig("postgres://user:pw@localhost:5432/app")
  if err != nil {
    fmt.Println("error:", err)
    return
  }
  fmt.Println(cfg.Host, cfg.Port, cfg.Database)
}
```

```bash
go mod tidy
cat go.mod
```

Output:

```
module example.com/indirect

go 1.27.1

require github.com/jackc/pgx/v5 v5.11.0

require (
	github.com/jackc/pgpassfile v1.0.0 // indirect
	github.com/jackc/pgservicefile v0.0.0-20240606120523-5a60cdf6a761 // indirect
	golang.org/x/text v0.29.0 // indirect
)
```

```bash
go run .
```

Output:

```
localhost 5432 app
```

`// indirect` means "no package in this module imports this directly". The
grouping into two blocks is purely `go mod tidy`'s convention; the directive is
identical.

Why list them at all, rather than trusting pgx's own `go.mod`? Because the build
list must be reproducible from *your* `go.mod` alone. Go records the full
transitive set so that resolution never needs to walk the graph at build time.

That long `v0.0.0-20240606120523-5a60cdf6a761` is a **pseudo-version**: a
timestamp plus commit hash, generated for a commit with no semver tag. The
structure is `v0.0.0-<UTC yyyymmddhhmmss>-<12-char hash>`, and it sorts below any
real `v0.0.1`.

## go.sum grows faster than go.mod

```bash
wc -l < go.sum
head -3 go.sum
```

Output:

```
      26
github.com/davecgh/go-spew v1.1.0/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
github.com/davecgh/go-spew v1.1.1 h1:vj9j/u1bqnvCEfJOwUhtlOARqs3+rkHYY13jYWTU97c=
github.com/davecgh/go-spew v1.1.1/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
```

Four requirements, 26 `go.sum` lines. `go-spew` is a test dependency of pgx's
test dependency — present in the graph, hashed, never linked into your binary.
Note `v1.1.0/go.mod` with no matching `h1:` line: Go verified that version's
`go.mod` while computing the graph but never needed its source.

## Reading the graph

`go list -m all` is the build list: exactly what MVS selected.

```bash
go list -m all
```

Output, for the `golang.org/x/sync` module:

```
example.com/graphtour
golang.org/x/sync v0.24.0
```

`go mod graph` prints the raw requirement edges, one `from to` pair per line:

```bash
go mod graph
```

Output:

```
example.com/indirect github.com/jackc/pgpassfile@v1.0.0
example.com/indirect github.com/jackc/pgservicefile@v0.0.0-20240606120523-5a60cdf6a761
example.com/indirect github.com/jackc/pgx/v5@v5.11.0
example.com/indirect go@1.27.1
example.com/indirect golang.org/x/text@v0.29.0
github.com/jackc/pgpassfile@v1.0.0 github.com/stretchr/testify@v1.3.0
github.com/jackc/pgservicefile@v0.0.0-20240606120523-5a60cdf6a761 github.com/stretchr/testify@v1.7.0
github.com/jackc/pgx/v5@v5.11.0 github.com/jackc/pgpassfile@v1.0.0
github.com/jackc/pgx/v5@v5.11.0 github.com/jackc/pgservicefile@v0.0.0-20240606120523-5a60cdf6a761
github.com/jackc/pgx/v5@v5.11.0 github.com/jackc/puddle/v2@v2.2.2
github.com/jackc/pgx/v5@v5.11.0 github.com/stretchr/testify@v1.11.1
github.com/jackc/pgx/v5@v5.11.0 golang.org/x/sync@v0.17.0
github.com/jackc/pgx/v5@v5.11.0 golang.org/x/text@v0.29.0
github.com/jackc/pgx/v5@v5.11.0 github.com/davecgh/go-spew@v1.1.1
github.com/jackc/pgx/v5@v5.11.0 github.com/kr/pretty@v0.3.0
github.com/jackc/pgx/v5@v5.11.0 github.com/pmezard/go-difflib@v1.0.0
github.com/jackc/pgx/v5@v5.11.0 gopkg.in/check.v1@v1.0.0-20201130134442-10cb98267c6c
github.com/jackc/pgx/v5@v5.11.0 gopkg.in/yaml.v3@v3.0.1
github.com/jackc/pgx/v5@v5.11.0 go@1.25.0
go@1.27.1 toolchain@go1.27.1
```

Three things to notice. `testify` appears at v1.3.0, v1.7.0 and v1.11.1 — MVS
will pick v1.11.1, the maximum of the minimums. The toolchain itself is a node
(`go@1.27.1`, `toolchain@go1.27.1`), which is how the `go` directive participates
in version selection. And the graph lists far more than the build list: edges
exist for modules never selected.

`go mod why` explains *why* a module is in the graph, as an import chain:

```bash
go mod why github.com/jackc/puddle/v2
```

Output:

```
# github.com/jackc/puddle/v2
example.com/indirect
github.com/jackc/pgx/v5
github.com/jackc/pgx/v5.test
github.com/jackc/pgx/v5/stdlib
github.com/jackc/pgx/v5/pgxpool
github.com/jackc/puddle/v2
```

Read it top-down: your module imports pgx, whose *test* binary imports its own
`stdlib` package, which reaches `pgxpool`, which needs `puddle`. That chain is
why `go mod tidy` keeps it.

When nothing imports a module's root package you get the negative form:

```bash
go mod why golang.org/x/sync
```

Output:

```
# golang.org/x/sync
(main module does not need package golang.org/x/sync)
```

This is not a contradiction — `x/sync` was required for `x/sync/errgroup`, a
subpackage. `go mod why -m golang.org/x/sync` asks about the module instead of
the package.

## Minimal version selection, in one paragraph

Every module declares the *minimum* version of each dependency it works with.
MVS computes the build list by taking, for each module path in the graph, the
maximum of all those minimums — nothing more. Adding a dependency that requires
`testify v1.11.1` upgrades testify to exactly v1.11.1 and does not touch
anything else. This is why Go needs no lockfile and no SAT solver: the algorithm
is a single pass over the graph, the result is a pure function of the `go.mod`
files, and a new upstream release cannot change your build until someone edits a
`go.mod`. The cost is that you never get patch upgrades for free; `go get -u`
exists precisely because upgrades must be explicit.

## go mod download and the cache

```bash
go mod download -json github.com/jackc/pgpassfile
```

Output:

```
{
	"Path": "github.com/jackc/pgpassfile",
	"Version": "v1.0.0",
	"Info": "/Users/raj/go/pkg/mod/cache/download/github.com/jackc/pgpassfile/@v/v1.0.0.info",
	"GoMod": "/Users/raj/go/pkg/mod/cache/download/github.com/jackc/pgpassfile/@v/v1.0.0.mod",
	"Zip": "/Users/raj/go/pkg/mod/cache/download/github.com/jackc/pgpassfile/@v/v1.0.0.zip",
	"Dir": "/Users/raj/go/pkg/mod/github.com/jackc/pgpassfile@v1.0.0",
	"Sum": "h1:/6Hmqy13Ss2zCq62VdNG8tM1wchn8zjSGOBJ6icpsIM=",
	"GoModSum": "h1:CEx0iS5ambNFdcRtxPj5JhEz+xB6uRky5eyVu/W2HEg="
}
```

`Dir` is the extracted, **read-only** source. Editing it is how people
accidentally break every project on the machine; `replace` is the supported way
to patch a dependency. `go mod download` with no args fetches everything in the
build list, which is the standard Docker layer-caching step.

Check for available upgrades without applying them:

```bash
go list -m -u github.com/jackc/pgpassfile
```

Output:

```
github.com/jackc/pgpassfile v1.0.0
```

No bracketed `[v1.1.0]` suffix means it is already current.

## replace: pointing a path somewhere else

The common case is developing two modules together. `replace` with a relative
path makes the dependency resolve to a directory, with no version, no download,
and no `go.sum` entry.

```go
package libx

func Answer() int { return 42 }
```

```go
package main

import (
  "fmt"

  "example.com/libx"
)

func main() {
  fmt.Println(libx.Answer())
}
```

```bash
go mod edit -require=example.com/libx@v0.0.0 -replace=example.com/libx=../libx
cat go.mod
go run .
```

Output:

```
module example.com/app

go 1.27.1

require example.com/libx v0.0.0

replace example.com/libx => ../libx
42
```

Two things to know. A `replace` in a **dependency's** `go.mod` is ignored — only
the main module's replaces apply, so a library cannot force its consumers'
resolution. And for the multi-module local-development case specifically, `go
work` (lesson 15.8) is the modern answer: it keeps the redirect out of `go.mod`
so you cannot accidentally commit it.

## exclude and retract

`exclude` says "never select this exact version", pushing MVS to the next one up.
It is for a known-broken release you cannot avoid transitively. `retract` is the
mirror image, declared by the *author* in their own `go.mod` to disown a version
they published.

```bash
go mod edit -require=github.com/go-chi/chi/v5@v5.5.0
go mod edit -replace=github.com/go-chi/chi/v5=github.com/go-chi/chi/v5@v5.2.1
go mod edit -exclude=github.com/go-chi/chi/v5@v5.1.0
go mod edit -retract=v0.1.0
cat go.mod
```

Output:

```
module example.com/edittour

go 1.27.1

require github.com/go-chi/chi/v5 v5.5.0

replace github.com/go-chi/chi/v5 => github.com/go-chi/chi/v5 v5.2.1

exclude github.com/go-chi/chi/v5 v5.1.0

retract v0.1.0
```

Note `retract` takes a bare version — it only ever refers to the module it is
written in. Once published, `go get` warns consumers and `go list -m -versions`
hides retracted versions. A retracted version is not deleted; the proxy is
append-only, so retraction is advisory metadata.

`go mod edit -json` gives the file as structured data, which is what you script
against:

```bash
go mod edit -json
```

Output:

```
{
	"Module": {
		"Path": "example.com/edittour"
	},
	"Go": "1.27.1",
	"Require": [
		{
			"Path": "github.com/go-chi/chi/v5",
			"Version": "v5.5.0"
		}
	],
	"Exclude": [
		{
			"Path": "github.com/go-chi/chi/v5",
			"Version": "v5.1.0"
		}
	],
	"Replace": [
		{
			"Old": {
				"Path": "github.com/go-chi/chi/v5"
			},
			"New": {
				"Path": "github.com/go-chi/chi/v5",
				"Version": "v5.2.1"
			}
		}
	],
	"Retract": [
		{
			"Low": "v0.1.0",
			"High": "v0.1.0"
		}
	]
}
```

## Semantic import versioning: the /v2 rule

For major version 2 and above, the major version is part of the import path.
`chi` v5 is imported as `github.com/go-chi/chi/v5`, and that is a *different
module path* from `github.com/go-chi/chi`. The consequence is that v1 and v5 can
coexist in one build — the diamond dependency problem simply does not arise,
because they are not the same module.

v0 and v1 share the unsuffixed path. The compiler enforces the boundary:

```bash
go build .
```

This fails on purpose, with `require github.com/go-chi/chi v5.5.0` — a v5 version
under the v1 path. Output:

```
go: errors parsing go.mod:
go.mod:5: require github.com/go-chi/chi: version "v5.5.0" invalid: should be v0 or v1, not v5
```

So publishing a breaking change means moving your own code to `/v2`: tag `v2.0.0`
and either add a `/v2` suffix to the `module` line on the main branch, or create a
`v2/` subdirectory. Downstream upgrades become an explicit import-path edit,
never a silent break.

## Quick Reference

| Directive | Form | Meaning |
|---|---|---|
| `module` | `module example.com/x` | this module's import path |
| `go` | `go 1.27.1` | minimum language version; drives toolchain selection |
| `toolchain` | `toolchain go1.27.1` | preferred toolchain, if newer than local |
| `require` | `require path v1.2.3` | minimum version of a dependency |
| `require` + `// indirect` | | nothing here imports it directly |
| `replace` | `replace a => b v1.0.0` | redirect a path/version; main module only |
| `replace` | `replace a => ../dir` | redirect to a local directory |
| `exclude` | `exclude path v1.2.3` | never select that exact version |
| `retract` | `retract v0.1.0` | author disowns this version of *this* module |
| `retract` | `retract [v1.0.0, v1.0.5]` | a disowned range |

| Command | Does |
|---|---|
| `go mod init PATH` | create `go.mod` |
| `go mod tidy` | sync `require`/`go.sum` with actual imports |
| `go get pkg@v1.2.3` | add or change one requirement |
| `go get -u ./...` | upgrade dependencies of these packages |
| `go get pkg@none` | drop a requirement |
| `go list -m all` | the build list MVS selected |
| `go list -m -u all` | build list with available upgrades in brackets |
| `go list -m -versions PATH` | every published version |
| `go mod graph` | raw requirement edges, `from to` per line |
| `go mod why PATH` | the import chain that pulls it in |
| `go mod why -m PATH` | same, asking about the module not the package |
| `go mod download` | populate the module cache from the build list |
| `go mod download -json PATH` | cache paths and hashes as JSON |
| `go mod edit -require=...` | mechanical edit, no resolution |
| `go mod edit -json` | the file as JSON |
| `go mod verify` | re-hash the cache against `go.sum` |

| File | Commit it? | Is it a lockfile? |
|---|---|---|
| `go.mod` | yes | yes — it pins exact versions |
| `go.sum` | yes | no — it is a checksum ledger |

## Common Mistakes

**Treating `go.sum` as the lockfile and `go.mod` as a range spec.** `go.mod`
holds exact versions and pins the build; `go.sum` only proves the bytes were not
swapped.

**Deleting `go.sum` to fix a checksum error.** That silences the one mechanism
designed to catch a tampered dependency — investigate the mismatch instead.

**Hand-editing a `require` line to upgrade.** Use `go get pkg@version`, which
updates the indirect set and `go.sum` too; a raw edit leaves the graph
inconsistent.

**Expecting `replace` in your library to affect its consumers.** Only the main
module's `replace` directives apply; publish a proper version instead.

**Importing `github.com/foo/bar` when the module is at v3.** v2+ requires the
`/v3` suffix in the import path, and the unsuffixed path is a different module.

**Editing files under `$GOMODCACHE` to patch a dependency.** That tree is
read-only and shared machine-wide; use `replace` pointing at a local checkout.

## Exercises

1. Start a fresh module, add `github.com/go-chi/chi/v5`, and diff `go.mod` and
   `go.sum` before and after `go mod tidy`. Explain every line added to `go.sum`.
2. Use `go mod graph` to find a module that appears at three different versions,
   then confirm with `go list -m all` which one MVS selected and why.
3. Add a dependency, then remove the import but not the `require` line. Show what
   `go mod tidy` does, and what `go build` does without it.
4. Create two local modules where A requires B via a relative `replace`. Then
   convert the setup to `go work` and show that `go.mod` no longer mentions B's
   location.
5. Pick any module at v2 or above and deliberately import it without the version
   suffix. Capture the error, then find the published version list with
   `go list -m -versions`.
6. Tamper with one byte of an `h1:` hash in `go.sum`, run `go clean -modcache`,
   and capture the full `SECURITY ERROR`. Then recover with `go mod tidy`.

---

[← 1.2 Hello, Modules: Your First Project](02-hello-modules.md) · [Course Index](../README.md) · [1.4 Packages, Imports, and Visibility →](04-packages-imports.md)
