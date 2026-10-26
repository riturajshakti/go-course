[← 17.8 Testing Against a Real Database](../17-database/08-testing-db.md) · [Course Index](../README.md) · [18.2 Configuration via Environment →](02-config.md)

# 18.1 — Project Layout: cmd/ and internal/

## What You'll Learn

- `cmd/<binary>/main.go` per executable, and why `main` should be a thin wiring function
- `internal/` is enforced by the **compiler**, not by convention — with the real error
- Organise `internal/` by **domain** (`internal/order/`), not by layer (`internal/models/` + `internal/handlers/`) — and the import cycle that proves it
- Hand-wiring dependencies in `main`, the `Application` struct, and why there is no `utils` package

## Coming From Another Language

| Other | Go |
|---|---|
| `src/` holding everything, `index.ts` as entry | no `src/`; the module root *is* the source root |
| `package.json` `"bin"` map | one directory per binary under `cmd/` |
| Python `__init__.py` + `_private` naming convention | `internal/` is a **compiler** rule; lowercase identifiers are the other half |
| `models/`, `views/`, `controllers/` by layer (Rails, Django, Nest) | by **domain**; the layer split causes import cycles Go will not allow |
| Rust `src/lib.rs` + `src/bin/*.rs` | `internal/...` + `cmd/*/main.go`; same shape, different names |
| Node: circular imports resolve to `undefined` at runtime | an import cycle is a **compile error**, always |

## There is no official layout

Go ships no project scaffold and the standard library has no opinion. The widely-cited
[golang-standards/project-layout](https://github.com/golang-standards/project-layout) repo
is **not** official and not endorsed by the Go team — it is one person's survey, and its
`pkg/` advice in particular is widely disputed. What *is* real is enforced by the toolchain:
`cmd/`, `internal/`, `testdata/`, and `go.mod`'s module path. Everything else is taste, and
small projects should stay flat.

Start here, and only add directories when a file has nowhere good to go:

```
myservice/
  go.mod
  go.sum
  main.go
  handlers.go
  store.go
  store_test.go
```

That is a legitimate layout for a few thousand lines. Promote to the structured version
below when you grow a second binary or a second domain.

## The single-binary service layout

This is the realistic shape for one deployed service. Each entry has a job:

```
github.com/acme/orders/
  go.mod
  go.sum
  cmd/
    server/main.go         # the HTTP binary: parse config, wire, run
  internal/
    order/                 # DOMAIN package: type, service, repo, handler
      order.go
      service.go
      postgres.go
      handler.go
    user/
    platform/              # cross-cutting infrastructure, no domain logic
      httpx/
      postgres/
      config/
  api/
    openapi.yaml           # the spec; generated clients/servers go here too
  migrations/
    0001_init.up.sql
  web/
    static/                # assets, usually //go:embed-ed (13.13)
  scripts/
    dev.sh
  deploy/
    k8s/
    Dockerfile
  testdata/                # ignored by the go tool, any module
```

Three of those are toolchain-meaningful. `internal/` restricts imports (below).
`testdata/` is skipped by `go build` and every package-listing command, so you can put
malformed files in it without breaking builds. `cmd/` is pure convention, but universal
enough that deviating costs you reader comprehension for nothing.

## cmd/ and a thin main

One directory per binary; the directory name becomes the binary name from
`go build ./cmd/server` ([15.1](../15-tooling/01-build-run-install.md)).

```
cmd/
  server/main.go       -> ./server
  worker/main.go       -> ./worker
  migrate/main.go      -> ./migrate
```

`main` does exactly three things: read configuration, construct dependencies, call `run`.
No business logic lives there, because nothing in `package main` can be imported or tested
by anything else.

```go
package main

import (
  "fmt"
  "os"

  "github.com/acme/orders/internal/order"
  "github.com/acme/orders/internal/platform/httpx"
)

func main() {
  if err := run(); err != nil {
    fmt.Fprintln(os.Stderr, "fatal:", err)
    os.Exit(1)
  }
}

func run() error {
  svc := order.NewService(order.NewMemoryRepo())
  srv := httpx.NewServer(svc)
  fmt.Println("wired:", srv.Routes())
  return nil
}
```

Output:

```
wired: [GET /orders/{id}]
```

The `main`/`run` split is the idiomatic testable shape: `run` returns an error, `main` is the
only place that calls `os.Exit`. [18.5](05-graceful-shutdown.md) develops this into the full
production `run()` with signal handling, and the reason it matters there is that `os.Exit`
skips every `defer` — so `main` must own it and nothing else.

## internal/ is a compiler rule

A package under any directory named `internal` is importable only by code rooted at that
`internal` directory's **parent**. This is not a lint; it is in the build system, and it is
the only access control Go gives you above the identifier level
([1.4](../01-getting-started/04-packages-imports.md) covers the lowercase half).

Here is a second module trying to import `github.com/acme/orders/internal/order`:

```go
package main

import (
  "fmt"

  "github.com/acme/orders/internal/order"
)

func main() {
  fmt.Println(order.ErrNotFound)
}
```

This does not compile. Output:

```
package example.com/outsider
	main.go:6:3: use of internal package github.com/acme/orders/internal/order not allowed
```

That error is the single most useful structural tool in Go. Put **everything** in
`internal/` by default: you can then refactor any of it freely, forever, because no external
consumer can possibly depend on it. Promote a package out of `internal/` only when you have
decided to support its API.

The rule is positional, not global — `internal/` directories nest. In

```
internal/platform/postgres/     <- importable anywhere in the module
internal/order/internal/sql/    <- importable only under internal/order/
```

the second form lets a large domain package hide its own guts from its siblings. Rarely
needed, but it exists.

## pkg/ and why you probably do not need it

`pkg/foo/` means "public library code". It adds a path segment, and `github.com/acme/orders/pkg/order` reads no better than `github.com/acme/orders/order` — the Go team does not use it in the standard library or in `golang.org/x/...`. Use it only if you genuinely publish libraries from the same repo as your service and want the boundary visible in the import path; otherwise put public packages at the module root and everything else in `internal/`.

## Domain packages, not layer packages

This is the decision that determines whether the codebase stays pleasant. The instinct from
MVC frameworks is to split by layer:

```
internal/
  models/       user.go, order.go, invoice.go
  handlers/     user.go, order.go, invoice.go
  services/     user.go, order.go, invoice.go
  repositories/ user.go, order.go, invoice.go
```

Every feature now touches four directories, every file is named after its layer's view of
the same noun, and — the fatal part — the packages want to import each other. `handlers`
needs `models`. And the moment a model needs a response shape, or a repository needs a
service's validation rule, you get this:

```go
package models

import "example.com/scratch/internal/handlers"

type User struct {
  ID   string
  View handlers.UserView
}
```

This does not compile. Output:

```
package example.com/scratch/internal/handlers
	imports example.com/scratch/internal/models from user.go
	imports example.com/scratch/internal/handlers from user.go: import cycle not allowed
```

Go has no cycle escape hatch. Not a forward declaration, not a lazy import, not an interface
file you can stick in the middle without creating a fifth package. In TypeScript or Python
the equivalent cycle resolves at runtime (to `undefined` or a partially-initialised module)
and you never notice; in Go the build stops.

The fix is to cut along the **domain** instead:

```
internal/
  order/        order.go  service.go  postgres.go  handler.go
  user/         user.go   service.go  postgres.go  handler.go
  invoice/
  platform/
    httpx/
    postgres/
```

`internal/order` now contains the type, the business rules, the SQL, and the HTTP handler for
orders. Those four files change together when a feature changes, they can see each other's
unexported identifiers, and there is no inter-layer import to cycle. Cross-domain
dependencies become a **directed** graph you choose deliberately — `invoice` imports `order`,
`order` does not import `invoice` — and if that ever becomes a cycle, it is telling you the
domain boundary is wrong, which is information worth having.

The real skeleton built for this lesson, verified with `gofmt -l`, `go vet ./...` and
`go build ./...` (all clean):

Output:

```
.
./api
./api/openapi.yaml
./cmd
./cmd/server
./cmd/server/main.go
./cmd/worker
./cmd/worker/main.go
./deploy
./deploy/k8s.yaml
./go.mod
./internal
./internal/order
./internal/order/handler.go
./internal/order/memory.go
./internal/order/order.go
./internal/order/service.go
./internal/platform
./internal/platform/httpx
./internal/platform/httpx/server.go
./internal/platform/postgres
./internal/platform/postgres/db.go
./internal/user
./internal/user/user.go
./migrations
./migrations/0001_init.up.sql
./scripts
./scripts/dev.sh
./testdata
./testdata/orders.json
```

## Where modules 16 and 17 live in this

The transport, service and repository code you have already written slots in like this:

| Layer | From | Lives in | Depends on |
|---|---|---|---|
| HTTP handlers, routing, middleware | 16.6, 16.7, 16.9 | `internal/order/handler.go`, `internal/platform/httpx/` | the service interface |
| business rules, orchestration | — | `internal/order/service.go` | the `Repository` interface |
| SQL, pgx, scanning | 17.3, 17.7 | `internal/order/postgres.go` | the pool from `platform/postgres` |
| pool construction, DSN handling | 17.2, 17.5 | `internal/platform/postgres/` | config only |

The direction of every arrow is inward: HTTP knows about the service, the service knows about
an interface, the Postgres implementation satisfies that interface. The `Repository` interface
is declared in `internal/order` — **next to its consumer**, not next to its implementation —
which is the Go convention that makes `order` testable with a fake and keeps it ignorant of
pgx entirely ([14.8](../14-testing/08-test-doubles.md)).

```go
package order

import "errors"

var ErrNotFound = errors.New("order not found")

type Order struct {
  ID    string
  Total int64
}

type Repository interface {
  Get(id string) (Order, error)
  Put(o Order) error
}
```

## The multi-binary monorepo layout

Several binaries sharing domain code. The only structural change is more entries under `cmd/`
and a `platform` tree worth its keep:

```
github.com/acme/platform/
  go.mod
  cmd/
    api/main.go              # public HTTP API
    worker/main.go           # queue consumer
    cron/main.go             # scheduled jobs
    migrate/main.go          # migration runner
    adminctl/main.go         # operator CLI
  internal/
    order/                   # shared by api, worker, cron
    user/
    billing/
    notify/
    platform/
      config/
      postgres/
      httpx/
      queue/
      telemetry/
  api/openapi.yaml
  migrations/
  deploy/
    api/Dockerfile
    worker/Dockerfile
    k8s/
```

Every binary links only what it imports, so `cmd/cron` does not carry the HTTP router. When
one binary genuinely needs a different dependency *version* from another — rare — that is the
signal to split into separate modules or a `go.work` workspace
([15.8](../15-tooling/08-generate-workspaces.md)), not to contort the layout.

## Wiring by hand in main

Construct everything explicitly, innermost first, and pass it down. Nothing clever:

```go
func run(ctx context.Context) error {
  cfg, err := config.Load()
  if err != nil {
    return fmt.Errorf("config: %w", err)
  }

  logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: cfg.LogLevel}))

  pool, err := postgres.Open(ctx, cfg.DatabaseURL)
  if err != nil {
    return fmt.Errorf("database: %w", err)
  }
  defer pool.Close()

  orders := order.NewService(order.NewPostgresRepo(pool), logger)
  users := user.NewService(user.NewPostgresRepo(pool), logger)

  app := &httpx.Application{
    Logger: logger,
    Orders: orders,
    Users:  users,
    Config: cfg,
  }
  return app.Serve(ctx)
}
```

That is the whole dependency-injection story for most Go services: a function that calls
constructors in order. `google/wire` (compile-time codegen) and `uber-go/fx` (runtime
reflection) exist and are reasonable above roughly fifty dependencies; hand-wiring is
idiomatic and usually enough, and it has the property that the dependency graph is readable
top to bottom in one function.

## The Application struct instead of globals

The alternative to package-level `var db *pgxpool.Pool` is one struct that holds the
dependencies, with handlers as methods on it:

```go
package httpx

type Application struct {
  Logger *slog.Logger
  Orders *order.Service
  Users  *user.Service
  Config *config.Config
}

func (a *Application) Routes() http.Handler {
  r := chi.NewRouter()
  r.Get("/orders/{id}", a.getOrder)
  r.Post("/orders", a.createOrder)
  return r
}

func (a *Application) getOrder(w http.ResponseWriter, r *http.Request) {
  o, err := a.Orders.Get(r.Context(), chi.URLParam(r, "id"))
  ...
}
```

Globals make every test share mutable state and make `t.Parallel()` unsafe; the struct makes
each test construct its own `Application` with fakes. This is the same pattern as the `*API`
receiver in 16.5 and 16.9, scaled to a whole service.

## Do not create a utils package

`internal/utils`, `internal/common`, `internal/helpers`, `internal/shared`: all the same
mistake. The name describes nothing, so nobody can predict what is inside, so everyone adds
one more unrelated function, so eventually every package imports it — and now it is a
cycle magnet and a dependency bottleneck. Go's own standard library has no `utils`: a
string helper goes in `strings`, a time helper in `time`.

Name packages for what they *are*: `internal/platform/money`, `internal/platform/pagination`,
`internal/platform/validate`. If a helper has exactly one caller, it belongs in that caller's
package as an unexported function. Package names should be short, lowercase, no underscores,
and no plurals — `order` not `orders`, because the call site reads `order.Service`.

## Import cycles drive Go package design

In most languages the package boundary is organisational. In Go it is a build constraint with
teeth: the import graph must be a DAG, and the only ways to break a cycle are to merge the two
packages, move the shared type into a third package that both import, or invert the dependency
with an interface declared in the consumer. That is why Go codebases end up with fewer, larger,
domain-shaped packages than the same system written in TypeScript or Java — splitting a package
costs you an edge in a graph that must stay acyclic. When you feel the urge to split, ask which
direction the import will point first; if the answer is "both", you have one package, not two.

## The module path

```bash
go mod init github.com/acme/orders
```

The module path is a) the import prefix for every package in the module and b) where `go get`
would fetch it. Use the real repository URL even for a private service — it costs nothing and
it means the path is correct if the code ever moves or gets imported. Avoid a bare name like
`orders`: it is legal, but it is unfetchable and it renders badly in a dependency graph. The
major version lives in the path from v2 onward (`github.com/acme/orders/v2`), which is
[1.3](../01-getting-started/03-go-mod.md).

## Quick Reference

| Directory | Toolchain-aware | Purpose |
|---|---|---|
| `cmd/<name>/` | no (convention) | one `main` package per binary; name becomes the binary |
| `internal/` | **yes** | importable only within the parent subtree |
| `testdata/` | **yes** | skipped by build and package listing |
| `pkg/` | no | public library code; usually unnecessary |
| `api/` | no | OpenAPI/proto specs and generated code |
| `migrations/` | no | SQL migrations (17.6) |
| `web/` | no | static assets, templates; often `//go:embed`-ed |
| `scripts/` | no | dev and build shell scripts |
| `deploy/` | no | Dockerfiles, k8s manifests, compose files |
| `vendor/` | **yes** | `go mod vendor` output; used automatically if present |

| Rule | Why |
|---|---|
| default everything into `internal/` | the compiler guarantees you can refactor it |
| one domain per package, not one layer | the layer split forces import cycles |
| declare interfaces beside the consumer | lets the consumer be tested without the implementation |
| `main` parses config, wires, calls `run() error` | only `main` calls `os.Exit`, which skips defers |
| dependencies on a struct, never package globals | parallel tests, no shared mutable state |
| no `utils`/`common`/`helpers` | unnameable packages become cycle magnets |

## Common Mistakes

**Splitting `internal/` by layer.** `models/` + `handlers/` + `services/` wants to be a cycle
and Go will refuse to build it; split by domain and keep the layers as files.

**Putting business logic in `package main`.** Nothing can import it, so nothing can test it;
move it to `internal/` and leave `main` with wiring only.

**Creating `pkg/` for everything.** It adds a path segment and implies a public API you did not
intend to support; use `internal/` until you deliberately publish something.

**Using package-level globals for the DB pool and logger.** Tests then share mutable state and
cannot run in parallel; hang them off an `Application` struct.

**Declaring the `Repository` interface next to its Postgres implementation.** The consumer then
imports the database package to get the interface; declare it in the consuming package.

**Naming a package `utils`.** It attracts unrelated code until every package imports it; name
packages for what they contain.

**A module path that is not a URL.** `go mod init orders` builds fine but is unfetchable and
breaks the moment another module needs it.

## Exercises

1. Take the 16.5 CRUD API and the 17.7 repository and restructure them into
   `cmd/server/main.go` + `internal/item/` (type, service, postgres, handler) +
   `internal/platform/httpx/`. Confirm `go build ./...` and `go vet ./...` are clean and that
   `main.go` contains no business logic.
2. Deliberately create the `models` ↔ `handlers` cycle from this lesson in your own module,
   capture the exact compiler error, then fix it three different ways (merge the packages,
   extract a third package, invert with an interface) and write one paragraph on which you
   would ship.
3. Add a second binary `cmd/worker` that imports `internal/item` but not the HTTP packages.
   Build both and compare `ls -l` sizes to confirm the worker did not link the router.
4. Publish a package out of `internal/` to the module root, then write a throwaway second
   module that imports it, to see the before/after of the `internal` restriction for real.
5. Audit a project you already have in another language for the layer-vs-domain split. List
   every file you would touch to add one field to one entity under each layout, and compare
   the counts.

---

[← 17.8 Testing Against a Real Database](../17-database/08-testing-db.md) · [Course Index](../README.md) · [18.2 Configuration via Environment →](02-config.md)
