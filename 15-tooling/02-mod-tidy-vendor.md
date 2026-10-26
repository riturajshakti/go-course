[← 15.1 go build, run, install, and the Build Cache](01-build-run-install.md) · [Course Index](../README.md) · [15.3 go vet, staticcheck, and golangci-lint →](03-linters.md)

# 15.2 — go mod tidy, vendor, and Dependency Management

## What You'll Learn

- Every `go.mod` directive, including the three nobody reads: `exclude`, `retract`, `toolchain`
- `go mod tidy -diff` fails CI without touching the files
- The modern split: `go get` edits `go.mod`, `go install pkg@ver` installs binaries
- MVS picks the *minimum* version that satisfies everyone, never the newest

## Coming From Another Language

| Other | Go |
|---|---|
| `package.json` + lockfile | `go.mod` + `go.sum` — but `go.sum` is checksums, not a lockfile |
| npm resolves to the newest compatible | MVS resolves to the **oldest** version that satisfies all requirements |
| `node_modules/` per project | one global `$GOMODCACHE`, shared, read-only |
| `^1.2.0` ranges | exact versions only; there is no range syntax |
| `npm audit` / `pip install --require-hashes` | `go.sum` verification is always on, against a transparency log |
| `package.json` "overrides" | `replace` (local or remote), `exclude` |
| major version in the name: `lodash` vs `lodash-es` | major version in the import path: `pkg/v2` |

## Every go.mod directive

```
module example.com/app

go 1.27.1

toolchain go1.27.1

require (
	github.com/go-chi/chi/v5 v5.5.0
	github.com/prometheus/client_golang v1.25.0
)

require (
	github.com/beorn7/perks v1.0.1 // indirect
	golang.org/x/sys v0.48.0 // indirect
)

replace example.com/greet => ../greet

exclude github.com/bad/mod v1.4.0

retract (
	v0.2.0 // Published with a broken Hello; use v0.2.1.
	[v0.1.0, v0.1.3] // Accidentally exported internal API.
)

tool github.com/rakyll/hey
```

| Directive | Meaning |
|---|---|
| `module` | this module's import path; the `/vN` suffix is part of it |
| `go` | the **language version** this module is written against; gates syntax and stdlib symbols |
| `toolchain` | the minimum toolchain to use; triggers an automatic download if yours is older |
| `require` | a module and its minimum version; `// indirect` means no package of yours imports it directly |
| `replace` | substitute a module path or version, with a filesystem path or another module |
| `exclude` | never select this exact version; MVS skips to the next one up |
| `retract` | *this* module's own versions that should not be used (only meaningful for the module being published) |
| `tool` | record a tool dependency so `go tool` can run it (added by `go get -tool`) |

Two `require` blocks is not a style choice — `go mod tidy` writes direct
requirements in the first and indirect ones in the second. Do not merge them by
hand; tidy will split them again.

`go` and `toolchain` are different things. `go 1.27.1` says "this code uses Go
1.27 language features"; a Go 1.24 toolchain will refuse to build it.
`toolchain go1.27.1` says "use at least this compiler", and with `GOTOOLCHAIN`
set to its default `auto` the `go` command downloads it for you.

Edit any of them without a text editor:

```bash
go mod edit -require=golang.org/x/sys@v0.48.0
go mod edit -droprequire=github.com/old/dep
go mod edit -replace=example.com/greet=../greet
go mod edit -dropreplace=example.com/greet
go mod edit -go=1.27
go mod edit -json          # dump go.mod as JSON
```

`go mod edit` is purely textual: it does not resolve, download, or verify
anything. That makes it the right tool for scripts and the wrong tool for
"add this dependency" — use `go get`.

## go.sum: two hashes per module

```bash
cat go.sum
```

Output:

```
github.com/go-chi/chi/v5 v5.5.0 h1:xdKQ1dJmVJsmgahrXSKhUx8NyVPzpwdlyrrQFpMPJgg=
github.com/go-chi/chi/v5 v5.5.0/go.mod h1:d1IG7GtKPSjPyBP365PH8sAGVHk3qfB/IhjFNRH1wWM=
```

The first line hashes the **module zip** (all its source). The second, with the
`/go.mod` suffix, hashes **only that module's go.mod file**. Both exist because
version resolution needs to read a dependency's `go.mod` without downloading
its source: MVS walks the requirement graph via `/go.mod` hashes only, and
downloads zips just for the packages you actually import. That is why `go.sum`
usually has far more `/go.mod` lines than zip lines.

`h1:` is the hash algorithm prefix — a base64 SHA-256 of a sorted file-hash
list. There has never been an `h2`.

`go.sum` is **not** a lockfile. It does not decide which version you get
(`go.mod` does); it only asserts that the version you got has the bytes it had
the first time anyone fetched it. Commit it always.

```bash
go mod verify
```

Output:

```
all modules verified
```

## The checksum security error

Tamper with one character of a hash:

```bash
sed -i '' 's|h1:xdKQ1dJmVJ...|h1:AAAAAAAA...|' go.sum
go clean -modcache
go build .
```

This is an intentional failure. Output:

```
go: downloading github.com/go-chi/chi/v5 v5.5.0
verifying github.com/go-chi/chi/v5@v5.5.0: checksum mismatch
	downloaded: h1:xdKQ1dJmVJsmgahrXSKhUx8NyVPzpwdlyrrQFpMPJgg=
	go.sum:     h1:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=

SECURITY ERROR
This download does NOT match an earlier download recorded in go.sum.
The bits may have been replaced on the origin server, or an attacker may
have intercepted the download attempt.

For more information, see 'go help module-auth'.
```

That is a hard stop — there is no `--force`. Legitimate causes are rare: a
module author force-pushed a tag, or someone hand-edited `go.sum`. The fix is
never `go mod tidy` over the top; it is to find out which of those happened.
There is no override flag. If you genuinely must accept new bytes for a
version, delete that module's two lines from `go.sum` and let the toolchain
re-record them — and understand that you have just chosen to trust whatever the
proxy served you.

## go mod tidy: exactly what it adds and removes

Start from a module whose `main.go` imports `chi` and `jwt`, but whose `go.mod`
requires `chi` and `prometheus/client_golang`. So there is one **missing**
requirement and one **unused** one:

```go
package main

import (
  "fmt"

  "github.com/go-chi/chi/v5"
  "github.com/golang-jwt/jwt/v5"
)

func main() {
  r := chi.NewRouter()
  t := jwt.New(jwt.SigningMethodHS256)
  fmt.Println(r != nil, t.Method.Alg())
}
```

```bash
cat go.mod
```

Output:

```
module example.com/scratch

go 1.27.1

require github.com/go-chi/chi/v5 v5.5.0

require github.com/prometheus/client_golang v1.25.0 // indirect
```

`go mod tidy -diff` shows the fix without applying it:

```bash
go mod tidy -diff
```

Output:

```
diff current/go.mod tidy/go.mod
--- current/go.mod
+++ tidy/go.mod
@@ -2,6 +2,7 @@
 
 go 1.27.1
 
-require github.com/go-chi/chi/v5 v5.5.0
-
-require github.com/prometheus/client_golang v1.25.0 // indirect
+require (
+	github.com/go-chi/chi/v5 v5.5.0
+	github.com/golang-jwt/jwt/v5 v5.3.1
+)

diff current/go.sum go.sum
--- current/go.sum
+++ tidy/go.sum
@@ -1,4 +1,4 @@
 github.com/go-chi/chi/v5 v5.5.0 h1:xdKQ1dJmVJsmgahrXSKhUx8NyVPzpwdlyrrQFpMPJgg=
 github.com/go-chi/chi/v5 v5.5.0/go.mod h1:d1IG7GtKPSjPyBP365PH8sAGVHk3qfB/IhjFNRH1wWM=
-github.com/prometheus/client_golang v1.25.0 h1:cDtW1TcFsMi1RJc8khwCKB4VhzwV04aKqbYtNOXuiK0=
-github.com/prometheus/client_golang v1.25.0/go.mod h1:j2X/4/kaQHLPnmyEU8zjmkndo1T317VpCIQEml8LTKk=
+github.com/golang-jwt/jwt/v5 v5.3.1 h1:kYf81DTWFe7t+1VvL7eS+jKFVWaUnK9cB1qbwn63YCY=
+github.com/golang-jwt/jwt/v5 v5.3.1/go.mod h1:fxCRLWMO43lRc8nhHWY6LGqRcf+1gQWArsqaEUEa5bE=
```

Both halves in one diff: jwt added because the code imports it, prometheus
removed because nothing does. `-diff` exists on Go 1.27 and **exits 1** when
there is a difference, which makes it the right CI gate — no working-tree
mutation, no `git diff --exit-code` dance:

```bash
go mod tidy -diff   # exit 0 if tidy, exit 1 with the diff if not
```

What tidy does, precisely:

- adds a `require` for every module providing a package imported by this
  module's packages **or their tests**
- removes `require` lines nothing needs, transitively
- adds `// indirect` markers and the full indirect requirement set
- adds every `go.sum` entry needed to build and test all of them, and removes
  the rest
- resolves the newest version of anything missing

That "or their tests" clause is why a test-only dependency like testify shows
up as a direct requirement. And `-diff` aside, tidy is *not* read-only: run it
before you commit, never as part of a build.

## Indirect requirements and why they are listed

```bash
go mod tidy
cat go.mod
```

Output:

```
module example.com/scratch

go 1.27.1

require (
	github.com/go-chi/chi/v5 v5.5.0
	github.com/prometheus/client_golang v1.25.0
)

require (
	github.com/beorn7/perks v1.0.1 // indirect
	github.com/cespare/xxhash/v2 v2.3.0 // indirect
	github.com/munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822 // indirect
	github.com/prometheus/client_model v0.6.3 // indirect
	github.com/prometheus/common v0.72.0 // indirect
	github.com/prometheus/procfs v0.22.0 // indirect
	golang.org/x/sys v0.48.0 // indirect
	google.golang.org/protobuf v1.36.12 // indirect
)
```

Since the `go 1.17` module graph pruning change, the main module's `go.mod`
lists **every** module needed to build it, direct or not — the complete,
flattened build list. The benefit is that resolving your build no longer
requires reading every dependency's `go.mod` transitively, so `go build` can
avoid network access entirely. The cost is a longer file that churns on every
upgrade. Do not delete the indirect block; tidy will put it back.

`munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822` is a **pseudo-version**:
a synthesised semver for a commit with no tag. Its shape is
`v0.0.0-<UTC commit time>-<12-char commit hash>`, and it sorts below any real
tag so a later real release always wins. Variants exist for commits after a
tag (`v1.2.4-0.2019...`) and after a prerelease. You never type one by hand —
`go get module@<branch-or-sha>` writes it for you.

## go get vs go install vs go mod download

This is the thing most people still have wrong:

| Command | Edits `go.mod` | Installs a binary | Needs a module |
|---|---|---|---|
| `go get pkg@ver` | yes | **no** | yes |
| `go install pkg@ver` | no | yes | no |
| `go mod download` | only `go.sum` | no | yes |
| `go get -tool pkg` | yes (adds `tool`) | no | yes |

`go get` has not installed binaries since Go 1.18. It only changes
requirements:

```bash
go get github.com/rakyll/hey
```

Output:

```
go: added github.com/rakyll/hey v0.1.5
go: added golang.org/x/net v0.48.0
go: added golang.org/x/text v0.33.0
```

Nothing was installed anywhere; three requirements were added to `go.mod`.
To get the `hey` binary you want `go install github.com/rakyll/hey@latest`
(15.1).

`go get -tool` records a tool dependency so the version is pinned in `go.mod`
and shared across the team:

```bash
go get -tool github.com/rakyll/hey
```

The resulting `go.mod` gains a `tool` line, and `go tool hey` then builds and
runs it at the pinned version. This replaces the old `tools.go` file with a
blank-import list behind an `ignore` build tag (15.4 covers that convention).

`go mod download` populates the module cache and `go.sum` without building:

```bash
go mod download -json github.com/go-chi/chi/v5
```

Output (paths are machine-specific):

```
{
	"Path": "github.com/go-chi/chi/v5",
	"Version": "v5.5.0",
	"Info": "/Users/raj/go/pkg/mod/cache/download/github.com/go-chi/chi/v5/@v/v5.5.0.info",
	"GoMod": "/Users/raj/go/pkg/mod/cache/download/github.com/go-chi/chi/v5/@v/v5.5.0.mod",
	"Zip": "/Users/raj/go/pkg/mod/cache/download/github.com/go-chi/chi/v5/@v/v5.5.0.zip",
	"Dir": "/Users/raj/go/pkg/mod/github.com/go-chi/chi/v5@v5.5.0",
	"Sum": "h1:xdKQ1dJmVJsmgahrXSKhUx8NyVPzpwdlyrrQFpMPJgg=",
	"GoModSum": "h1:d1IG7GtKPSjPyBP365PH8sAGVHk3qfB/IhjFNRH1wWM="
}
```

Idiomatic in a Dockerfile: copy `go.mod`/`go.sum`, run `go mod download`, then
copy the source. The download layer caches until your dependencies change
(18.8).

## Upgrading and downgrading

All four forms, run in sequence on the same module:

```bash
go get github.com/go-chi/chi/v5@v5.0.10
go get github.com/go-chi/chi/v5@latest
go get github.com/go-chi/chi/v5@v5.0.10
go get -u=patch ./...
go get -u ./...
```

Output:

```
go: added github.com/go-chi/chi/v5 v5.0.10
go: upgraded github.com/go-chi/chi/v5 v5.0.10 => v5.5.0
go: downgraded github.com/go-chi/chi/v5 v5.5.0 => v5.0.10
go: upgraded github.com/go-chi/chi/v5 v5.0.10 => v5.0.14
go: upgraded github.com/go-chi/chi/v5 v5.0.14 => v5.5.0
```

Read the last two carefully. From `v5.0.10`, `-u=patch` went to `v5.0.14` — the
newest **patch** of the same minor. Plain `-u` went to `v5.5.0` — the newest
minor within the same major. Neither crosses a major version, ever: `v6`
is a different import path, so upgrading across a major is an edit to your
source, not a flag.

`@none` removes a requirement, `@upgrade` and `@patch` are the per-module
equivalents of `-u` and `-u=patch`, and a branch or SHA gets you a
pseudo-version:

```bash
go get github.com/old/dep@none
go get github.com/x/y@master
go get github.com/x/y@a1b2c3d
```

See what is available before you jump:

```bash
go list -m -u all
```

Output (truncated; `[...]` is the available upgrade):

```
example.com/scratch
github.com/alecthomas/kingpin/v2 v2.4.0
github.com/alecthomas/units v0.0.0-20240927000941-0f3dac36c52b
github.com/beorn7/perks v1.0.1
github.com/cespare/xxhash/v2 v2.3.0
github.com/go-chi/chi/v5 v5.5.0
github.com/golang-jwt/jwt/v5 v5.3.1
github.com/golang/protobuf v1.5.0 [v1.5.4] (deprecated)
github.com/google/go-cmp v0.7.0
github.com/google/uuid v1.6.0
github.com/jpillora/backoff v1.0.0 [v1.1.0]
github.com/json-iterator/go v1.1.12
github.com/julienschmidt/httprouter v1.3.0
github.com/klauspost/compress v1.20.1 [v1.20.3]
```

Three things to notice. Modules with no bracket are current. `(deprecated)`
comes from a `Deprecated:` comment in the upstream `go.mod` — the toolchain
surfaces it for you. And this list is longer than `go.mod`'s: `all` includes
the whole module graph, not just the build list.

`go list -m -versions pkg` lists every published version of one module, and
`-retracted` includes the retracted ones.

## MVS in one paragraph

Minimal version selection: for each module in the graph, Go selects the
**maximum of the minimums** that anything requires — and nothing more. Real
example from the module above:

```bash
go mod graph | grep 'golang.org/x/sys@'
```

Output:

```
example.com/scratch golang.org/x/sys@v0.48.0
github.com/prometheus/client_golang@v1.25.0 golang.org/x/sys@v0.48.0
github.com/prometheus/common@v0.72.0 golang.org/x/sys@v0.48.0
github.com/prometheus/procfs@v0.22.0 golang.org/x/sys@v0.47.0
golang.org/x/sys@v0.48.0 go@1.26.0
```

`procfs` asks for `v0.47.0`, `client_golang` and `common` ask for `v0.48.0`.
Selected:

```bash
go list -m golang.org/x/sys
```

Output:

```
golang.org/x/sys v0.48.0
```

`v0.48.0` — the highest *required* version, not the highest *published* one
(`go list -m -u` above shows upgrades available for several modules; MVS
ignores them). The consequence is the property npm lacks: the same `go.mod`
produces the same build list today and in three years, with no lockfile,
because nothing in the algorithm consults "latest". A version only changes when
someone edits a `go.mod`.

Setting a requirement *lower* than the graph demands simply has no effect:

```bash
go mod edit -require=golang.org/x/sys@v0.20.0
go list -m golang.org/x/sys
```

Output:

```
golang.org/x/sys v0.48.0
```

MVS took the maximum and ignored your downgrade. To actually force a lower
version you need `replace` (or to downgrade whatever requires the higher one,
which `go get pkg@lower` will offer to do for you).

## go mod graph and go mod why

`go mod graph` prints the edges of the module requirement graph, one
`requirer requiree@version` pair per line:

```bash
go mod graph | head -10
```

Output:

```
example.com/scratch github.com/beorn7/perks@v1.0.1
example.com/scratch github.com/cespare/xxhash/v2@v2.3.0
example.com/scratch github.com/go-chi/chi/v5@v5.5.0
example.com/scratch github.com/munnerz/goautoneg@v0.0.0-20191010083416-a7dc8b61c822
example.com/scratch github.com/prometheus/client_golang@v1.25.0
example.com/scratch github.com/prometheus/client_model@v0.6.3
example.com/scratch github.com/prometheus/common@v0.72.0
example.com/scratch github.com/prometheus/procfs@v0.22.0
example.com/scratch go@1.27.1
example.com/scratch golang.org/x/sys@v0.48.0
```

`go mod why` answers the question you actually have — *why is this in my
build?* — with an import path chain:

```bash
go mod why github.com/cespare/xxhash/v2
```

Output:

```
# github.com/cespare/xxhash/v2
example.com/scratch
github.com/prometheus/client_golang/prometheus/promhttp
github.com/prometheus/client_golang/prometheus
github.com/cespare/xxhash/v2
```

Read it top-down: my module imports `promhttp`, which imports `prometheus`,
which imports `xxhash`. `-m` takes module paths instead of package paths and
reports the shortest chain to a *package* in that module:

```bash
go mod why -m golang.org/x/sys
```

Output:

```
# golang.org/x/sys
example.com/scratch
github.com/prometheus/client_golang/prometheus/promhttp
github.com/prometheus/client_golang/prometheus
golang.org/x/sys/unix
```

When `go mod why` prints `(main module does not need module X)`, tidy will
remove it.

## replace for local development

Two modules side by side. `greet` is a library, `app` consumes it, and neither
is published anywhere:

```
rep/
  greet/go.mod       module example.com/greet
  greet/greet.go
  app/go.mod         module example.com/app
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
  fmt.Println(greet.Hello("tooling"))
}
```

Without a `replace`, `go mod tidy` goes looking on the internet:

```bash
cd app && go mod tidy
```

This is an intentional failure. Output:

```
go: finding module for package example.com/greet
go: example.com/app imports
	example.com/greet: cannot find module providing package example.com/greet: unrecognized import path "example.com/greet": reading https://example.com/greet?go-get=1: 404 Not Found
```

A `require` plus a relative `replace` fixes it:

```bash
go mod edit -require=example.com/greet@v0.0.0 -replace=example.com/greet=../greet
cat go.mod
```

Output:

```
module example.com/app

go 1.27.1

require example.com/greet v0.0.0

replace example.com/greet => ../greet
```

```bash
go run .
```

Output:

```
hello, tooling
```

Three details that trip people up. You need **both** a `require` and the
`replace` — the replace only redirects a requirement that already exists, and
the `v0.0.0` version is a placeholder that is never looked up. `go.sum` stays
**empty**: a filesystem replacement has no checksum to verify. And a `replace`
in a dependency's `go.mod` is **ignored** — only the main module's replaces
apply, which is deliberate, and which is why a library must never rely on one.

The other `replace` forms:

```
replace example.com/greet => ../greet                        # path, any version
replace example.com/greet v0.1.0 => ../greet                 # only that version
replace example.com/greet => example.com/fork/greet v1.2.0    # a fork
replace example.com/greet v0.1.0 => example.com/greet v0.1.1  # pin around a bad release
```

A relative `replace` must not be committed to a shared branch — it breaks every
other checkout and every CI job. That is precisely the problem workspaces solve,
and 15.8 shows the same two modules wired up with `go work` instead.

## exclude and retract

`exclude` blacklists one exact version, so MVS skips to the next one that
satisfies the graph:

```
exclude github.com/bad/mod v1.4.0
```

Use it when a specific release is broken and you cannot wait for whoever
requires it to bump. It is rare; `replace` with a version pair is usually
clearer because it names the version you *do* want.

`retract` is the other direction: it is how **you**, as a module author, say
"do not use these versions of my module". You add it, tag a *new* version
(because the retraction lives in the retracted module's own `go.mod`, which
means it must be published after the fact), and push:

```
retract (
	v0.2.0 // Published with a broken Hello; use v0.2.1.
	[v0.1.0, v0.1.3] // Accidentally exported internal API.
)
```

The comment is not decoration — the toolchain prints it to consumers. A
consumer already pinned to a retracted version keeps building (retraction
never breaks anyone), but `go get -u` and `go list -m -u` stop offering it, and
`go list -m -retracted module@version` reports it with your reason.

Verified honestly: I could not capture that consumer-side message here. A
filesystem `replace` bypasses the retraction check entirely —
`go list -m -retracted` against a replaced local module reports an empty
retraction list — and the retraction has to be served by a module proxy from a
real published tag to be seen. The directive syntax above is what you write;
the consumer behaviour is from `go help mod edit` and the module reference, not
from a run in this environment.

## Vendoring

`go mod vendor` copies every package needed to build and test the main module
into `vendor/`:

```bash
go mod vendor
find vendor -maxdepth 2 -type d | sort
```

Output:

```
vendor
vendor/github.com
vendor/github.com/beorn7
vendor/github.com/cespare
vendor/github.com/go-chi
vendor/github.com/munnerz
vendor/github.com/prometheus
vendor/golang.org
vendor/golang.org/x
vendor/google.golang.org
vendor/google.golang.org/protobuf
```

Only the **packages** you need, not whole modules — no tests, no unused
subpackages. `vendor/modules.txt` is the manifest:

```bash
head -22 vendor/modules.txt
```

Output:

```
# github.com/beorn7/perks v1.0.1
## explicit; go 1.11
github.com/beorn7/perks/quantile
# github.com/cespare/xxhash/v2 v2.3.0
## explicit; go 1.11
github.com/cespare/xxhash/v2
# github.com/go-chi/chi/v5 v5.5.0
## explicit; go 1.24
github.com/go-chi/chi/v5
# github.com/munnerz/goautoneg v0.0.0-20191010083416-a7dc8b61c822
## explicit
github.com/munnerz/goautoneg
# github.com/prometheus/client_golang v1.25.0
## explicit; go 1.26.0
github.com/prometheus/client_golang/internal/github.com/golang/gddo/httputil
github.com/prometheus/client_golang/internal/github.com/golang/gddo/httputil/header
github.com/prometheus/client_golang/prometheus
github.com/prometheus/client_golang/prometheus/internal
github.com/prometheus/client_golang/prometheus/promhttp
github.com/prometheus/client_golang/prometheus/promhttp/internal
# github.com/prometheus/client_model v0.6.3
## explicit; go 1.23
```

`#` lines are modules with versions, `##` lines carry metadata (`explicit`
means it is a direct requirement in your `go.mod`), and bare lines are the
vendored packages. The toolchain cross-checks this against `go.mod` on every
build.

**`vendor/` is used automatically when it exists** — confirmed on Go 1.27: with
a `vendor/` directory present and `go.mod` declaring `go 1.14` or later, plain
`go build` resolves from it and never touches the network or the module cache.
`go build -x` shows 74 references to `vendor/` paths and zero downloads. You do
not need `-mod=vendor`.

| Setting | Effect |
|---|---|
| (default, `vendor/` present) | `-mod=vendor` |
| (default, no `vendor/`) | `-mod=readonly` |
| `-mod=vendor` | resolve from `vendor/`, error if inconsistent |
| `-mod=mod` | ignore `vendor/`, resolve from the cache; may edit `go.mod` |
| `-mod=readonly` | resolve from the cache; **error** rather than edit `go.mod` |

`-mod=readonly` being the default since 1.16 is why `go build` now tells you to
run `go get` instead of silently adding a requirement.

A `vendor/` that disagrees with `go.mod` is a hard error, not a warning:

```bash
go mod edit -require=github.com/golang-jwt/jwt/v5@v5.3.1
go build .
```

This is an intentional failure. Output (the absolute path is this scratch
directory; yours will differ):

```
go: inconsistent vendoring in /tmp/scratch:
	github.com/golang-jwt/jwt/v5@v5.3.1: is explicitly required in go.mod, but not marked as explicit in vendor/modules.txt

	To ignore the vendor directory, use -mod=readonly or -mod=mod.
	To sync the vendor directory, run:
		go mod vendor
```

So vendoring is a commitment: every dependency change is now a two-step
(`go get` then `go mod vendor`) and a large diff in review.

When is it still worth it? Builds in an air-gapped network. A hard requirement
that the repo contain every byte that ships. Regulatory source review. Very
large monorepos that want zero module-cache variance across thousands of CI
jobs. For most services a proxy plus committed `go.sum` gives you the same
reproducibility without the diff noise — `go mod vendor` is no longer the
default advice it was in 2018.

## GOPROXY, GOPRIVATE, GOSUMDB

```bash
go env GOPROXY GOPRIVATE GOSUMDB GOMODCACHE
```

Output (GOPRIVATE is empty here):

```
https://proxy.golang.org,direct

sum.golang.org
/Users/raj/go/pkg/mod
```

| Variable | Meaning |
|---|---|
| `GOPROXY` | comma-separated proxy list; `direct` fetches from the VCS; `off` forbids any download |
| `GOPRIVATE` | glob list of module prefixes that are neither proxied nor checksum-verified |
| `GONOPROXY` | glob list to fetch directly, bypassing the proxy; defaults to `GOPRIVATE` |
| `GONOSUMDB` | glob list to skip checksum-database lookup for; defaults to `GOPRIVATE` |
| `GOSUMDB` | the checksum transparency log; `off` disables the log lookup entirely |
| `GONOSUMCHECK` | **not a thing in modules mode** — a GOPATH-era variable; it is silently ignored |
| `GOMODCACHE` | where downloaded module source lives |
| `GOFLAGS=-mod=mod` | let `go build` edit `go.mod` again (the pre-1.16 behaviour) |

The practical setup for a company with private repos is one line:

```bash
go env -w GOPRIVATE=github.com/mycorp/*
```

`GONOPROXY` and `GONOSUMDB` both default to `GOPRIVATE`, so
`github.com/mycorp/...` is fetched directly over your git credentials and never
sent to `proxy.golang.org` or `sum.golang.org`. Everything public still goes
through the proxy and is still verified. Do **not** reach for `GOFLAGS=-insecure`
or `GOSUMDB=off` globally — that turns verification off for your public
dependencies too.

`GOPROXY=off` is the hermetic-build switch: any attempt to download fails, so
the build must be satisfiable from the module cache or `vendor/`.

## Semantic import versioning and /v2

Major version 2 and above goes **in the import path**:

```
github.com/golang-jwt/jwt      → v0.x, v1.x
github.com/golang-jwt/jwt/v5   → v5.x
```

The consequence is that `v1` and `v5` of the same library are two different
packages and can coexist in one build — which is how a large dependency tree
survives a breaking upgrade of a shared library. Import the wrong path and the
resolver simply does not find it:

```go
package main

import (
  "fmt"

  "github.com/golang-jwt/jwt"
)

func main() {
  fmt.Println(jwt.SigningMethodHS256.Alg())
}
```

This is an intentional failure, with `jwt/v5` already required. Output:

```
main.go:6:3: no required module provides package github.com/golang-jwt/jwt; to add it:
	go get github.com/golang-jwt/jwt
```

And asking for a v5 version at the unsuffixed path gives the diagnostic that
names the rule:

```bash
go get github.com/golang-jwt/jwt@v5.3.1
```

This is an intentional failure. Output:

```
go: github.com/golang-jwt/jwt@v5.3.1: invalid version: go.mod has post-v5 module path "github.com/golang-jwt/jwt/v5" at revision v5.3.1
```

"go.mod has post-v5 module path" is the toolchain telling you the tag exists
but the module declares itself at `/v5`, so that is the path you must import.

`v0` and `v1` share the unsuffixed path, and `v0` carries no compatibility
promise at all. `+incompatible` versions (you saw
`github.com/gofrs/uuid v4.4.0+incompatible` in an earlier resolution) are
pre-modules repos tagged `v2`+ with no `go.mod` — the toolchain tolerates them
rather than endorsing them.

## Quick Reference

| Command | Does |
|---|---|
| `go mod init PATH` | create `go.mod` |
| `go mod tidy` | add missing, drop unused, sync `go.sum` |
| `go mod tidy -diff` | print the diff, exit 1 if not tidy; changes nothing |
| `go mod verify` | re-hash the module cache against `go.sum` |
| `go mod download` | fetch modules into the cache, update `go.sum` |
| `go mod download -json MOD` | cache paths and hashes as JSON |
| `go mod graph` | every requirement edge |
| `go mod why PKG` | import chain explaining why a package is in the build |
| `go mod why -m MOD` | the same, for a module path |
| `go mod vendor` | copy needed packages into `vendor/` |
| `go mod edit -require=M@V` | textual edit, no resolution |
| `go mod edit -replace=A=B` | add a replace |
| `go mod edit -json` | `go.mod` as JSON |

| Command | Does |
|---|---|
| `go get pkg@v1.2.3` | pin an exact version |
| `go get pkg@latest` | newest release |
| `go get pkg@none` | remove the requirement |
| `go get pkg@master` | a branch, as a pseudo-version |
| `go get -u ./...` | upgrade minors+patches within the same major |
| `go get -u=patch ./...` | upgrade patches only |
| `go get -tool pkg` | add a pinned `tool` dependency |
| `go install pkg@v1.2.3` | build a binary into `$GOBIN`; ignores `go.mod` |
| `go list -m all` | the whole module graph |
| `go list -m -u all` | with available upgrades and deprecations |
| `go list -m -versions pkg` | every published version |
| `go list -m -retracted pkg@v` | retraction status and reason |

## Common Mistakes

**Treating `go.sum` as a lockfile and deleting it to "re-resolve".** It is a
checksum database; `go.mod` decides versions. Deleting it discards your
supply-chain verification and tidy will happily re-record whatever it now
downloads.

**Hand-merging the two `require` blocks or deleting the `// indirect` one.**
Tidy re-splits and re-adds them; the indirect block is the pruned module
graph your build needs.

**Committing a relative `replace` to a shared branch.** It breaks every other
checkout and CI; use a workspace (15.8) for local multi-module work.

**Expecting `go get` to install a CLI.** Since Go 1.18 it only edits `go.mod` —
`go install pkg@latest` installs, `go get -tool` pins.

**Running `go mod tidy` inside the build step of a Dockerfile or CI job.** It
mutates `go.mod`/`go.sum` and masks the drift you wanted to catch; run
`go mod tidy -diff` as a check instead.

**Assuming `-u` will cross a major version.** `v2+` lives at a different import
path, so moving from `pkg` to `pkg/v2` is a source edit, not a flag.

**Setting `GOSUMDB=off` or `GOFLAGS=-insecure` to make one private repo work.**
That disables verification for all your public dependencies too; `GOPRIVATE`
scopes it to the prefixes you own.

## Exercises

1. Build a module that imports two packages, then hand-add an unused `require`
   and delete one import. Predict the output of `go mod tidy -diff`, run it,
   and check its exit code with `echo $?`.
2. Find a module in a real project's `go.mod` whose purpose you cannot guess.
   Use `go mod why` and `go mod why -m` on it and write down the chain. Then
   remove the direct dependency that pulls it in and confirm tidy drops it.
3. Construct an MVS conflict on purpose: two dependencies requiring different
   minor versions of the same third module. Use `go mod graph` to find both
   edges and `go list -m` to show which won.
4. Wire up the `greet`/`app` pair from this lesson with a relative `replace`.
   Then add a method to `greet`, use it in `app`, and confirm no `go get` was
   needed. Check whether `go.sum` gained any lines.
5. Vendor a module, then upgrade one dependency with `go get` without
   re-vendoring. Capture the exact error, then fix it. Measure how many files
   `go mod vendor` adds to `git status`.
6. Tamper with one character of one `go.sum` hash, clear the module cache, and
   capture the full security error. Restore it and confirm `go mod verify`
   passes.
7. Set `GOPROXY=off` and run `go build` in a module whose cache you have just
   cleared. Then vendor it and try again with `GOPROXY=off` still set.

---

[← 15.1 go build, run, install, and the Build Cache](01-build-run-install.md) · [Course Index](../README.md) · [15.3 go vet, staticcheck, and golangci-lint →](03-linters.md)
