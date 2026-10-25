[← 1.3 go.mod, go.sum, and the Module Graph](03-go-mod.md) · [Course Index](../README.md) · [1.5 gofmt, go vet, and the Editor Setup →](05-tooling.md)

# 1.4 — Packages, Imports, and Visibility

## What You'll Learn

- One package per directory, enforced by the compiler; files in a package see each other with no imports
- An initial capital letter is the *entire* visibility system — there is no `private`, `protected`, or `export`
- The four import forms: plain, aliased, dot, blank — and the real reason only two of them are acceptable
- Import cycles are a compile error, not a runtime hazard; `internal/` enforces package privacy by directory
- `init()` order is deterministic: imports depth-first, then package vars, then every `init()` in file order

## Coming From Another Language

| Other | Go |
|---|---|
| `export`/`export default` per symbol | capital first letter, nothing else |
| `import { a, b } from "./x"` | `import "m/x"` then `x.A` — no named imports, ever |
| Python: a file is a module | a **directory** is a package; files are just shards of it |
| Python: circular imports work if you are careful | hard compile error, no exceptions |
| `_private` convention in Python | lowercase is enforced by the compiler |
| TS `paths` aliases in tsconfig | import paths are literal module-path-relative, no mapping |
| Rust `mod`/`pub`/`use` tree | no nesting declaration; the filesystem *is* the tree |

## A directory is a package

```go
package mathx

const Pi = 3.14159

var callCount int

func Double(n int) int {
  callCount++
  return n * 2
}

func halve(n int) int {
  return n / 2
}

func Calls() int {
  return callCount
}
```

```go
package main

import (
  "fmt"

  "example.com/pkgtour/mathx"
)

func main() {
  fmt.Println(mathx.Double(21))
  fmt.Println(mathx.Pi)
  fmt.Println(mathx.Calls())
}
```

Output:

```
42
3.14159
1
```

The import path is `<module path>/<directory path>`. The *package name* is
whatever the `package` clause says, and that is what you type at the call site —
they usually match, but nothing enforces it. `gopls` inserts the real package
name, which is why `github.com/jackc/pgx/v5` is referenced as `pgx` and not `v5`.

There is no barrel file, no index, and no re-export. A package's API is the set of
capitalised identifiers across all its files.

## Visibility is one bit

Capitalised = exported outside the package. Lowercase = package-private. This
applies uniformly to functions, types, methods, struct fields, constants, and
package-level variables. There is no middle tier.

```go
package main

import (
  "fmt"

  "example.com/pkgtour/mathx"
)

func main() {
  fmt.Println(mathx.halve(10))
  fmt.Println(mathx.callCount)
}
```

This does not compile. Output:

```
main.go:6:3: no required module provides package example.com/pkgtour/mathx; to add it:
	go get example.com/pkgtour/mathx
```

Two different messages for the same rule. The first is reported as `undefined`
because `halve` is a function and the compiler cannot even resolve the selector;
the second gets the explicit "not exported" phrasing. Both mean the same thing.

Within a package there is no privacy at all — `callCount` is freely readable and
writable from any file of `mathx`. The package, not the type, is the unit of
encapsulation. This is why Go code tends toward smaller packages than Java or C#
code tends toward smaller classes.

## Files in one package see each other

No imports, no `#include`, no ordering requirement. Declaration order across files
is irrelevant.

```go
package main

import "fmt"

func main() {
  fmt.Println(greeting(), version)
  fmt.Println(farewell())
}
```

```go
package main

const version = "1.0"

func greeting() string { return "hello" }
```

```go
package main

func farewell() string { return "bye from " + version }
```

Output:

```
hello 1.0
bye from 1.0
```

`bye.go` uses `version` from `greet.go` and `main.go` uses both, with nothing
linking them but the shared `package main` clause and the directory.

## Two packages in one directory is an error

```bash
go build ./mixed
```

This does not compile, with `one.go` declaring `package one` and `two.go`
declaring `package two`. Output:

```
found packages one (one.go) and two (two.go) in /tmp/scratch/mixed
```

The one exception is the `_test` suffix: a directory may hold `package foo` and
`package foo_test` together, for black-box tests (lesson 14.1).

## Import cycles

```go
package a

import "example.com/pkgtour/b"

func A() string { return "a" + b.B() }
```

```go
package b

import "example.com/pkgtour/a"

func B() string { return "b" + a.A() }
```

This does not compile. Output:

```
package example.com/pkgtour/a
	imports example.com/pkgtour/b from a.go
	imports example.com/pkgtour/a from b.go: import cycle not allowed
```

The error prints the full chain, which is what makes long cycles tractable. There
is no lazy-import escape hatch as in Python, no forward declaration as in C. The
cycle must be broken, and the three standard moves are: extract the shared types
into a third package that both import; define the interface in the *consumer*
package rather than the producer (Go's implicit satisfaction makes this free —
lesson 5.1); or invert the dependency with a callback.

This constraint is load-bearing for Go's compile speed: package dependencies form
a DAG, so the compiler can build packages in parallel and cache each one.

## internal/

Any directory named `internal` is importable only by code rooted at its parent.
This is the one structural privacy mechanism beyond capitalisation.

```go
package secret

func Token() string { return "s3cr3t" }
```

From inside the same module, at `example.com/pkgtour/internal/secret`:

```go
package main

import (
  "fmt"

  "example.com/pkgtour/internal/secret"
)

func main() {
  fmt.Println(secret.Token())
}
```

Output:

```
s3cr3t
```

From a *different* module:

```bash
go build .
```

This does not compile. Output:

```
package example.com/outside
	main.go:6:2: use of internal package example.com/pkgtour/internal/secret not allowed
```

The rule is purely about tree position, not modules. A nested `internal` restricts
to *its* parent, so `svc/internal/store` is off limits to a sibling `other/`
package in the same module:

```bash
go build ./other
```

This does not compile. Output:

```
package example.com/pkgtour/other
	other/other.go:3:8: use of internal package example.com/pkgtour/svc/internal/store not allowed
```

Idiomatic: put everything in `internal/` by default and promote to a public
package only when you have decided to support it as API. Module 18.1 covers the
`cmd/` + `internal/` layout properly.

## The four import forms

```go
package main

import (
  "fmt"
  mrand "math/rand"
  "strings"

  _ "example.com/importtour/driver"
  "example.com/importtour/registry"
)
```

| Form | Effect |
|---|---|
| `import "strings"` | bind the package's own name |
| `import mrand "math/rand"` | bind under your name |
| `import . "strings"` | dump all exported names into this file's scope |
| `import _ "x/driver"` | run its `init()`, bind nothing |

Aliasing is for genuine collisions — two `rand` packages, or `v1 "api/v1"` next to
`v2 "api/v2"`. The blank import exists for registration side effects: database
drivers, image format decoders, `net/http/pprof` handlers.

The dot import works and should be treated as off limits:

```go
package main

import (
  . "fmt"
  . "strings"
)

func main() {
  Println(ToUpper("x"))
  Println(Split("a,b", ","))
}
```

Output:

```
X
[a b]
```

Nothing in that `main` tells a reader which package `Split` came from, and adding
a function to either package upstream can break your file with a redeclaration
error. Its only legitimate use is inside a `_test` package. Idiomatic: plain
imports; alias only on collision; `_` only for registration; never `.`.

Note the two import groups in the earlier example separated by a blank line.
gofmt sorts within a group but never merges groups, and the universal convention
is stdlib first, then everything else.

## init() and initialization order

```go
package registry

import "fmt"

var drivers []string

func Register(name string) {
  drivers = append(drivers, name)
}

func List() []string {
  return drivers
}

func init() {
  fmt.Println("registry init")
}
```

```go
package driver

import (
  "fmt"

  "example.com/importtour/registry"
)

func init() {
  fmt.Println("driver init")
  registry.Register("pgx")
}
```

```go
package main

import (
  "fmt"
  mrand "math/rand"
  "strings"

  _ "example.com/importtour/driver"
  "example.com/importtour/registry"
)

var topLevel = initTopLevel()

func initTopLevel() string {
  fmt.Println("package-level var init")
  return "ready"
}

func init() {
  fmt.Println("main init 1")
}

func init() {
  fmt.Println("main init 2")
}

func main() {
  fmt.Println("main:", topLevel)
  fmt.Println(registry.List())
  fmt.Println(strings.ToUpper("aliased and plain"))
  mrand.New(mrand.NewSource(1))
  fmt.Println("done")
}
```

Output:

```
registry init
driver init
package-level var init
main init 1
main init 2
main: ready
[pgx]
ALIASED AND PLAIN
done
```

The order is fully determined:

1. Every imported package is initialized first, depth-first, each exactly once.
   `registry` precedes `driver` because `driver` imports it.
2. Then this package's package-level variables, in *dependency* order — not
   source order. A var whose initializer reads another var waits for it.
3. Then every `init()` in the package, in the order the files are presented to
   the compiler (alphabetical, in practice) and top-to-bottom within a file.
4. Then `main()`.

`init` is the one function you may declare more than once per package, even in
one file. It takes no arguments, returns nothing, and cannot be called. The
`_ "driver"` import is what made `registry.List()` return `[pgx]` — the whole
registration pattern rests on step 1.

Keep `init` rare. It runs before flags are parsed and before any config is
loaded, it cannot return an error (only `panic`), and it makes tests order-
dependent. Explicit constructor calls from `main` are almost always better. The
genuine use cases are driver registration and compiling a `regexp` or `template`
once at startup.

## Package naming

```go
package mathx
```

Short, lowercase, one word, no underscores, no camelCase. It becomes a prefix at
every call site, so the name should not repeat in its members: `http.Server`, not
`http.HTTPServer`; `bytes.Buffer`, not `bytes.ByteBuffer`.

| Avoid | Why |
|---|---|
| `util`, `common`, `helpers`, `misc` | no coherent API; grows into a dumping ground |
| `models`, `types`, `interfaces` | organised by what things *are*, not what they do — guarantees import cycles |
| `myPackage`, `my_package` | gofmt won't fix it; convention is one lowercase word |
| a plural like `strings` for a single type | fine for stdlib precedent, odd for `users` holding one `User` |

`mathx` shows the standard escape when the obvious name is taken by the stdlib:
suffix `x`. The alternative is aliasing at the import site.

## package main vs a library package

`package main` compiles to an executable and cannot be imported — nothing can
import a main package, by any path. Everything else compiles to an archive that
only ever gets linked into some binary. That is the entire difference; there is no
"library mode" flag.

```bash
go build ./lib
```

This does not compile, where `lib` imports `cmd/app`, a main package. Output:

```
lib/lib.go:3:8: import "example.com/impmain/cmd/app" is a program, not an importable package
```

That error is why `cmd/` directories stay thin: anything in a main package is
untestable from elsewhere and unreusable, so logic belongs in `internal/`.

A library package needs no `main` function and `go build` of one produces no file
— it just typechecks and caches. This is why `go build ./...` on a library
repository appears to do nothing and succeeds.

## Quick Reference

| Rule | Detail |
|---|---|
| package per directory | one, plus an optional `foo_test` |
| import path | `<module path>/<dir path>`, always |
| identifier binding | the `package` clause, not the directory name |
| exported | first rune is an uppercase letter |
| unexported | anything else, including `_` and non-cased scripts |
| intra-package visibility | total; no privacy between files |
| import cycles | compile error, full chain printed |
| `internal/X` | importable only from the subtree rooted at `internal`'s parent |
| `package main` | not importable by anything |

| Import form | Use it for |
|---|---|
| `import "path"` | everything |
| `import alias "path"` | name collisions only |
| `import _ "path"` | side-effect registration only |
| `import . "path"` | `_test` packages, at most |

| Init stage | Order |
|---|---|
| imported packages | depth-first, once each |
| package-level vars | dependency order, not source order |
| `init()` funcs | file order, then top-to-bottom |
| `main()` | last |

## Common Mistakes

**Expecting `import { Foo } from` style named imports.** Go imports whole
packages and qualifies every use; there is no way to pull one symbol into scope
except the discouraged dot import.

**Lowercasing a struct field that `encoding/json` must see.** Unexported fields
are invisible to reflection, so they silently vanish from marshalled output.

**Creating a `models` or `types` package.** Organising by kind rather than by
responsibility puts the type below everything that uses it and makes cycles
inevitable; keep types in the package that owns their behaviour.

**Breaking an import cycle by merging the two packages.** Define the interface in
the consumer instead — implicit satisfaction means the producer needs no import
back.

**Doing real work in `init()`.** It runs before flag parsing, cannot return an
error, and makes tests order-dependent; call a constructor from `main`.

**Naming the directory `userService` and expecting that to be the package name.**
The `package` clause decides; a mismatch compiles but confuses every reader and
tool.

## Exercises

1. Build a module with packages `store` and `handler` where each needs a type
   from the other. Capture the cycle error, then break it three ways: a shared
   third package, a consumer-side interface, and a callback.
2. Create `internal/db` and prove two things: that a sibling package in the same
   module can import it, and that a second module with a `replace` directive
   cannot. Capture both results.
3. Add a `plugin` package that registers itself via `init()` into a `registry`,
   imported only as `_`. Then remove the blank import and show what the registry
   returns.
4. Write a package with four files, each declaring one `init()` printing its
   filename, plus a package-level var whose initializer prints. Predict the
   output, then run it. Rename a file so it sorts first and re-run.
5. Put a `User` struct with a lowercase `password` field through
   `json.Marshal` and `json.Unmarshal`. Explain the output, then fix it with a
   struct tag and an exported field.
6. Try to import a `package main` from another package. Capture the error and
   explain why `cmd/` directories are conventionally thin.

---

[← 1.3 go.mod, go.sum, and the Module Graph](03-go-mod.md) · [Course Index](../README.md) · [1.5 gofmt, go vet, and the Editor Setup →](05-tooling.md)
