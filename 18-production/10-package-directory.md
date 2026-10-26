[← 18.9 CI with GitHub Actions](09-ci.md) · [Course Index](../README.md)

# 18.10 — The Essential Package Directory

## What You'll Learn

- The stdlib-first default, stated concretely: what the standard library already covers, and the short list of gaps worth a dependency
- A curated directory of Go backend packages by purpose, with the stdlib alternative named for every entry
- How to evaluate a dependency before you add it — including the real dependency-graph cost, measured
- What to build next, and where to read further

Every module path below was verified against the real module proxy with
`go list -m <path>@latest`: **57 paths, 0 failures.** Resolved versions are quoted where they
are useful; they will have moved on by the time you read this.

## Coming From Another Language

| Other | Go |
|---|---|
| `package.json` with 40 direct and 900 transitive dependencies | a service with 5-15 direct dependencies is normal; 900 would alarm a reviewer |
| a framework chosen first, then the app written inside it | `net/http` plus a router; the "framework" is usually the standard library |
| `requirements.txt` pinned by a tool, resolved at install | `go.mod` + `go.sum` with checksums, and minimal version selection (1.3) |
| a new library for every need (dates, UUIDs, HTTP, env) | the stdlib already covers most of those; a dependency needs a reason |
| `left-pad` culture — small packages are free | "a little copying is better than a little dependency" is a real cultural norm |

## Stdlib first, and what that actually means

Go's standard library covers more of a backend service than most ecosystems' do, and the
community treats reaching past it as a decision that needs a reason. That is not purism — it is
a response to three real costs: every dependency is code you did not review but do ship, a
supply-chain surface, and a thing that can go unmaintained while your service still needs it.

Concretely, the stdlib already gives you a production HTTP server and client with routing
(`net/http`, 16.1-16.3), JSON (`encoding/json`), structured logging (`log/slog`), SQL access
(`database/sql`), templating (`html/template`), a test framework with benchmarks, fuzzing and
coverage (`testing`), TLS, flags, profiling (`net/http/pprof`), and since 1.22 a `ServeMux`
with method matching and path wildcards. Modules 13 through 17 of this course are almost
entirely standard library.

So the question for each entry below is narrow: **what does this do that the stdlib does not,
and is that gap worth the cost?** For most of them the answer is yes for *some* projects and no
for yours.

## Routing

| Package | What it adds | Reach for it when |
|---|---|---|
| `net/http` (`ServeMux`) | method matching, `{wildcards}`, `{rest...}` since 1.22 | always your first choice; enough for many services |
| `github.com/go-chi/chi/v5` | middleware chains, `Group`/`Route`, `URLParam`, `RoutePattern()`; 100% `net/http` compatible | you want route groups and per-group middleware — **this course's choice** |
| `github.com/gin-gonic/gin` | own `Context`, binding, validation, rendering; fast | you want a batteries-included framework and accept the non-stdlib handler type |
| `github.com/labstack/echo/v4` | similar scope to gin, different API taste | same trade-off as gin |
| `github.com/gorilla/mux` | regex routes, host matching, subrouters | legacy codebases; `ServeMux` 1.22+ covers most of its reason to exist |

**chi, restated in one line:** it is the only one of these whose handlers are plain
`http.Handler` and whose middleware is plain `func(http.Handler) http.Handler`, so every
example in module 16 works unchanged and you can remove chi later without rewriting handlers —
which is exactly why this course picked it ([16.7](../16-http/07-chi.md)).

A framework with its own `Context` type is not wrong, but it is a commitment: your handlers no
longer fit anywhere else, and every middleware you want must exist for that framework.

## Database

| Package | What it is | Note |
|---|---|---|
| `database/sql` | the stdlib driver-agnostic interface | portable, connection pooling, `NULL` handling via `sql.Null*` |
| `github.com/jackc/pgx/v5` | the Postgres driver **and** a native API | **this course's choice**; native mode gives Postgres types, `COPY`, batching, `LISTEN/NOTIFY`, better performance ([17.2](../17-database/02-pgx.md)) |
| `github.com/jmoiron/sqlx` | `database/sql` plus struct scanning | removes the `rows.Scan(&a, &b, &c)` boilerplate without hiding SQL |
| `sqlc` (a code generator) | generates typed Go from your `.sql` files | you want compile-time-checked queries and hand-written SQL; generates for pgx |
| `gorm.io/gorm` | full ORM: migrations, associations, hooks | see below |
| `entgo.io/ent` | schema-as-Go-code, generated typed client, graph traversals | a large relational model where generated type safety pays |

**The honest line on ORMs.** An ORM is worth it when you have many entities with many
relationships and most queries are CRUD — it removes a great deal of repetitive code. It is not
worth it when your queries are the interesting part of your application, because you then spend
your time fighting the query builder to emit the SQL you could have written directly, and the
SQL it does emit is hard to predict and therefore hard to optimise. The middle path that most
experienced Go teams land on is **plain SQL plus a scanning helper** (pgx, or sqlc for
generated types), which is what module 17 teaches.

## Migrations

| Package | Style |
|---|---|
| `github.com/golang-migrate/migrate/v4` | numbered `up`/`down` `.sql` files, CLI + library — **this course's choice** ([17.6](../17-database/06-migrations.md)) |
| `github.com/pressly/goose/v3` | `.sql` **or** Go migrations; embeddable; very commonly used |
| `ariga.io/atlas` | declarative — you describe the desired schema, it computes the diff |
| `github.com/jackc/tern/v2` | minimal, pgx-native |

No stdlib alternative; this is a genuine gap. Whichever you pick, run migrations as a **separate
step** (a `cmd/migrate` binary or an init container), not on service startup — otherwise five
replicas racing to migrate at once is your deploy process.

## Configuration

| Package | Note |
|---|---|
| `os.LookupEnv` + `flag` | the stdlib answer; 60 lines covers a service completely ([18.2](02-config.md)) |
| `github.com/kelseyhightower/envconfig` | struct tags → env vars; tiny, stable, effectively finished |
| `github.com/knadh/koanf/v2` | several sources merged (env + file + consul) with a modest dependency tree |
| `github.com/spf13/viper` | the kitchen sink: file watching, remote config; heavy (see the graph measurement below) |
| `github.com/joho/godotenv` | `.env` parsing for local development only |

## Logging

| Package | Note |
|---|---|
| `log/slog` | **the answer for new code** since Go 1.21 — structured, levelled, pluggable handlers ([13.12](../13-stdlib/12-slog.md), [18.4](04-service-logging.md)) |
| `github.com/rs/zerolog` | zero-allocation JSON; faster than slog in microbenchmarks |
| `go.uber.org/zap` | the pre-slog standard; still excellent, large API |

`slog` ended the logging debate for new Go code. It is in the standard library, so every
dependency can log through it without agreeing on a package, and `slog.Handler` lets you keep a
zap or zerolog backend if you have measured that you need the speed. Do not migrate a working
zap codebase for its own sake; do not start a new service on anything else.

## Errors

| Package | Note |
|---|---|
| `errors` + `fmt.Errorf("%w")` | the whole answer: `Is`, `As`, `Unwrap`, `Join` (module 8) |
| `github.com/pkg/errors` | **obsolete and archived.** Its `%w`-equivalent and stack traces were absorbed into the stdlib in 1.13; new code must not use it |

If you want stack traces on errors, a 20-line custom error type capturing `runtime.Callers`
beats a dependency — and the wrapping chain from `%w` usually tells you more than a stack trace
does anyway ([8.6](../08-errors/06-strategy.md)).

## Validation

| Package | Style |
|---|---|
| hand-written `Validate() error` | the stdlib answer; `errors.Join` for field errors ([16.8](../16-http/08-validation.md)) |
| `github.com/go-playground/validator/v10` | struct tags (`validate:"required,email,gte=18"`); the de-facto standard, used inside gin and echo |
| `github.com/go-ozzo/ozzo-validation/v4` | programmatic rules, no tags; easier for conditional validation |

Tags are concise for simple field rules and awkward the moment a rule depends on another field;
most codebases end up with tags plus a hand-written method for the cross-field cases.

## Auth and authorization

| Package | Purpose |
|---|---|
| `github.com/golang-jwt/jwt/v5` | JWT signing and validation — **this course's choice** ([18.3](03-auth-jwt.md)) |
| `golang.org/x/crypto/bcrypt` | password hashing; `argon2` in the same repo |
| `crypto/rand` + `crypto/subtle` | stdlib: session tokens, constant-time comparison |
| `github.com/casbin/casbin/v2` | policy-based authorization (RBAC/ABAC) with a policy DSL |
| `github.com/volatiletech/authboss/v3` | a full auth module: registration, confirm, recover, lock |

For a single service, a `sessions` table plus `crypto/rand` and an httpOnly cookie needs no
dependency at all, and 18.3 argues that is often the better choice.

## HTTP clients

| Package | Note |
|---|---|
| `net/http` | the stdlib client: connection pooling, timeouts, redirects ([16.10](../16-http/10-http-client.md)) |
| `github.com/go-resty/resty/v2` | fluent API with built-in retries and JSON marshalling |

Resty saves real boilerplate for a client hitting many endpoints of one API. The thing to not
skip either way: **always set a `Timeout` on your `http.Client`**, because the zero value waits
forever and that is how one slow upstream exhausts your goroutines.

## JSON

| Package | Note |
|---|---|
| `encoding/json` | the answer ([13.6](../13-stdlib/06-json.md), [13.7](../13-stdlib/07-json-advanced.md)) |
| `github.com/bytedance/sonic` | SIMD-accelerated; amd64/arm64 only, uses assembly and unsafe |
| `github.com/json-iterator/go` | drop-in faster replacement; largely superseded |

Use these **only after profiling** shows JSON encoding is your bottleneck
([15.6](../15-tooling/06-pprof.md)). It usually is not — the database or the network is. Go
1.25's `encoding/json/v2` work is also closing much of the gap in the stdlib itself.

## Testing

| Package | Purpose |
|---|---|
| `testing` | tables, subtests, benchmarks, fuzzing, coverage, `t.Cleanup`, `t.Setenv` (module 14) |
| `net/http/httptest` | the reason every HTTP example in this course is runnable ([16.11](../16-http/11-httptest.md)) |
| `github.com/stretchr/testify` | `assert`/`require` and `mock`; ubiquitous ([14.4](../14-testing/04-testify.md)) |
| `github.com/google/go-cmp` | `cmp.Diff` for deep comparison with readable diffs — **the one to add first** |
| `github.com/testcontainers/testcontainers-go` | real Postgres/Redis in Docker per test run ([17.8](../17-database/08-testing-db.md)) |
| `go.uber.org/goleak` | fails a test that leaked a goroutine ([12.5](../12-concurrency-patterns/05-deadlocks-leaks.md)) |
| `github.com/gavv/httpexpect/v2` | fluent HTTP API assertions |

If you add exactly one: **go-cmp**. `cmp.Diff(want, got)` on two structs produces a readable
field-level diff, which `reflect.DeepEqual` plus `%+v` does not, and it handles unexported
fields and `time.Time` correctly via options. **goleak** is the second, because a leaked
goroutine is otherwise invisible until production memory climbs.

testify is fine and you will meet it everywhere; this course uses stdlib `testing` because
`if got != want { t.Errorf(...) }` needs no explanation and the failure messages you write are
better than generic ones.

## Concurrency

| Package | Purpose |
|---|---|
| `sync`, `sync/atomic`, `context` | the foundation (module 11) |
| `golang.org/x/sync/errgroup` | a `WaitGroup` that collects the first error and cancels the group ([11.8](../11-sync-context/08-errgroup.md)) |
| `golang.org/x/sync/semaphore` | weighted concurrency limiting |
| `golang.org/x/sync/singleflight` | collapse duplicate concurrent calls for the same key — excellent cache-stampede protection |
| `golang.org/x/time/rate` | token-bucket rate limiting ([12.4](../12-concurrency-patterns/04-rate-limiting.md)) |

`golang.org/x/...` is maintained by the Go team under a compatibility policy slightly weaker
than the standard library's. Treat `errgroup` as essentially stdlib: it is in nearly every
serious Go service, and `errgroup.WithContext` is the right tool for "do these five things in
parallel, fail fast".

## Observability

| Package | Purpose |
|---|---|
| `net/http/pprof` | CPU, heap, goroutine, block and mutex profiles over HTTP ([15.6](../15-tooling/06-pprof.md)) |
| `runtime/metrics`, `expvar` | stdlib runtime counters |
| `github.com/prometheus/client_golang` | metrics — **this course's choice** ([18.7](07-prometheus.md)) |
| `go.opentelemetry.io/otel` | vendor-neutral traces, metrics and logs; the tracing answer |

Traces are the thing metrics and logs cannot give you: where the time went inside one request
across service boundaries. Add OTel tracing when you have more than two services; before that,
the request ID from 18.4 plus a canonical log line answers most of the same questions.

## CLI

| Package | Note |
|---|---|
| `flag` | the stdlib answer; enough for any service binary ([13.11](../13-stdlib/11-flag.md)) |
| `github.com/spf13/cobra` | subcommands, completions, generated help; what `kubectl` and `gh` use |
| `github.com/urfave/cli/v2` | similar scope, lighter, simpler API |

`flag` is right for `cmd/server` and `cmd/migrate`. Reach for cobra when you have a real
multi-command CLI (`app users create`, `app db migrate`) and want shell completion.

## Templates

| Package | Note |
|---|---|
| `html/template` | contextual auto-escaping, which is the security property that matters |
| `github.com/a-h/templ` | templates as Go code, compiled and type-checked; a generator step |

`html/template` escapes according to context (HTML body, attribute, JS, URL), which is why you
should not hand-roll HTML with `fmt.Sprintf`. templ gives you compile-time errors for a typo in
a field name, at the cost of `go generate` in your build.

## Task queues and background jobs

| Package | Backend |
|---|---|
| `github.com/riverqueue/river` | **Postgres** — transactional job enqueue in the same transaction as your write |
| `github.com/hibiken/asynq` | Redis; mature, good web UI |
| `github.com/RichardKnop/machinery/v2` | multiple brokers; older design |

River is worth singling out: enqueuing a job inside the same database transaction as the row it
is about removes the "wrote the row, crashed before enqueuing" failure mode, with no extra
infrastructure. If you already run Postgres, start there. A `time.Ticker` plus a goroutine is
genuinely sufficient for simple periodic work ([13.5](../13-stdlib/05-timers.md)).

## Caching

| Package | Note |
|---|---|
| `map` + `sync.RWMutex`, or `sync.Map` | the stdlib answer for a small bounded cache |
| `github.com/dgraph-io/ristretto/v2` | high-throughput in-memory cache with admission policy and TTL |
| `github.com/redis/go-redis/v9` | the Redis client, when the cache must be shared between instances |

A `map` guarded by an `RWMutex` with a size cap handles more cases than people expect. The
reason to move to ristretto is eviction quality under memory pressure; the reason to move to
Redis is sharing state across instances — a different problem, not a faster version of the same
one. Pair either with `singleflight` so a cache miss does not send a thousand concurrent
requests to your database.

## IDs

| Package | Produces |
|---|---|
| `github.com/google/uuid` | UUID v4 (random) and v7 (time-ordered) |
| `github.com/oklog/ulid/v2` | ULID: lexicographically sortable, 26 chars, Crockford base32 |
| `crypto/rand` | stdlib: any opaque token you want, e.g. 16 random bytes hex-encoded |

Prefer **time-ordered** IDs (UUIDv7 or ULID) for database primary keys: random v4 keys scatter
B-tree inserts across the whole index and hurt write throughput, while time-ordered ones append.
For a non-database token, `crypto/rand` plus `base64.RawURLEncoding` needs no dependency.

## Time

`time` is complete. You need `time.Time`, `time.Duration`, `time.Ticker`, `time.AfterFunc` and
the layout-based formatting, and that is genuinely all — the one habit to adopt is injecting a
`func() time.Time` clock into anything you want to test, which is what made half the outputs in
this module reproducible ([13.4](../13-stdlib/04-time.md)).

## gRPC and protobuf

| Package | Purpose |
|---|---|
| `google.golang.org/grpc` | the gRPC implementation |
| `google.golang.org/protobuf` | protobuf runtime and `protoc-gen-go` |
| `github.com/bufbuild/buf` | schema management, linting, breaking-change detection, codegen without `protoc` |

Use gRPC for service-to-service calls where you control both ends and want a typed contract and
streaming; keep JSON over HTTP for anything a browser or a third party calls. `buf` replaces the
`protoc` plugin-path misery and is the first thing to adopt if you go down this road.

## Message queues

| Package | Broker |
|---|---|
| `github.com/segmentio/kafka-go` | Kafka, pure Go — no CGO, so it fits the static-binary story from 18.8 |
| `github.com/confluentinc/confluent-kafka-go/v2` | Kafka via librdkafka (**CGO**); more features, breaks `CGO_ENABLED=0` |
| `github.com/nats-io/nats.go` | NATS / NATS JetStream; very light to operate |
| `github.com/rabbitmq/amqp091-go` | RabbitMQ (the official successor to `streadway/amqp`) |

The CGO distinction is a real deployment consequence, not trivia: the confluent client needs
`libc` in your final image, so distroless `static` and `scratch` are out
([18.8](08-docker.md)).

## Cloud SDKs

`github.com/aws/aws-sdk-go-v2` (modular — import only the services you use),
`cloud.google.com/go/...` and `github.com/Azure/azure-sdk-for-go/sdk/...`; all are large,
generated, and the only practical option for their platform, so the only real decision is to
import the sub-packages you need rather than the umbrella module.

## Dependency injection

| Approach | Note |
|---|---|
| hand-wiring in `main` | **idiomatic and usually enough** ([18.1](01-project-layout.md)) |
| `github.com/google/wire` | compile-time codegen; the generated code is readable Go, no runtime reflection |
| `go.uber.org/fx` | runtime DI with lifecycle hooks; powerful, and a framework you adopt wholesale |

Hand-wiring stops being pleasant somewhere past fifty dependencies. Until then, a `run()`
function that calls constructors in order is readable top to bottom, has no magic, and no new
contributor has to learn anything.

## How to evaluate a Go dependency

Before `go get`, spend two minutes:

**1. Is it in the stdlib or `golang.org/x/...` already?** This catches more candidates than you
would expect — `slog`, `slices`, `maps`, `cmp`, `errors.Join`, `ServeMux` wildcards and
`min`/`max` all landed recently and obsoleted popular packages.

**2. Last commit, and the shape of the issue tracker.** No commits in two years can mean
abandoned *or* finished — `envconfig` is finished. Read the open issues: unanswered bug reports
are the signal, not the count.

**3. Is it v1 or later, and does it respect semver?** A `v0.x` module promises nothing about
compatibility, and Go's import-path rule means a `v2` is a different import path — so a v0
dependency may demand a code change on any minor release.

**4. Count the transitive dependencies.** Measured for real, on two modules whose only
difference is one direct dependency:

```bash
go mod graph | wc -l
go list -m all | wc -l
```

For a module importing only `net/http` and chi:

Output:

```
go mod graph | wc -l : 4
go list -m all | wc -l: 2
```

For a module whose single direct dependency is viper:

Output:

```
go mod graph | wc -l : 65
go list -m all | wc -l: 27
go.sum lines        : 47
```

**One import, 26 additional modules.** Each is code in your binary, in your vulnerability
surface, and in your `go.sum`, and each can independently break or go unmaintained. That is not
an argument against viper specifically — it is a configuration library that merges many sources
and that costs dependencies — but it is the number you should look at before deciding whether
you need those sources. chi's 2 is the other end of the scale and is a large part of why this
course chose it.

**5. License.** MIT, BSD and Apache-2.0 are routine. GPL or AGPL in a service you distribute is
a question for someone other than you.

**6. Single-maintainer risk.** One person's side project is fine for a small, finished library
and a problem for something central to your architecture. Check whether anyone else has merge
rights.

**7. Does it have a smaller alternative — or none at all?** A dependency you can replace with
40 lines you understand is usually the better trade. The Go proverb is **"a little copying is
better than a little dependency"**: if you need one function from a package, copy it (with its
license header and a comment saying where it came from) rather than taking on the module.

Also run `govuln`: `go run golang.org/x/vuln/cmd/govulncheck@latest ./...` reports known
vulnerabilities in your actual call paths, not just in your module graph — and wire it into CI
([18.9](09-ci.md)).

## What to build next

Three projects, increasing in scope, each exercising this course's stack. Build them in order;
each one teaches something the previous one let you avoid.

**1. A URL shortener with analytics.** `POST /links` returns a short code, `GET /{code}`
redirects, `GET /links/{code}/stats` reports hit counts by day. Postgres for storage, chi for
routing, JWT for the admin endpoints, Prometheus for request metrics, graceful shutdown, a
multi-stage Dockerfile, CI.

Small enough to finish in a weekend and complete enough to hit the real decisions: what the
primary key is (time-ordered, not v4), how you count hits without a write per request
(in-memory counters flushed periodically — which forces you to think about what graceful
shutdown must flush), and whether redirect latency belongs in the same histogram as the API.

**2. A job-processing service.** An HTTP API that accepts work, a `cmd/worker` that processes it
from a Postgres queue (river, or `SELECT ... FOR UPDATE SKIP LOCKED` by hand), with retries and
exponential backoff, a dead-letter table, per-job metrics, and an admin UI to inspect and retry
failures.

This is where concurrency stops being theoretical: a worker pool draining on context
cancellation (12.1), at-least-once delivery and therefore idempotency, a job that outlives your
shutdown timeout, and a `SELECT FOR UPDATE SKIP LOCKED` that you can prove works by running two
workers. Write the test that starts two workers and asserts no job runs twice.

**3. A multi-tenant SaaS API.** Organisations, users, roles, invitations; row-level tenant
isolation; per-tenant rate limits; an audit log; webhooks out to customer endpoints with retries
and signature verification; OpenAPI spec and a generated client; traces across the API and the
worker.

The scope where architecture decisions start to bind. Tenant isolation enforced in one place
(and the test that proves a cross-tenant read is impossible), authorization beyond a role
string, outbound webhooks that must not let one slow customer endpoint block everyone, and
metric cardinality when "per tenant" is tempting and wrong (18.7).

## Where to read next

| Resource | Why |
|---|---|
| [go.dev/blog](https://go.dev/blog) | release notes and design rationale from the team; the generics, slog and error-handling posts are essential |
| [Effective Go](https://go.dev/doc/effective_go) | the original idiom document; dated in places, still the best statement of *why* Go code looks like it does |
| [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) | the community's accumulated review feedback, in list form — read it once a year |
| [The Go Memory Model](https://go.dev/ref/mem) | what is and is not guaranteed about concurrent access; short, precise, worth the hour ([12.6](../12-concurrency-patterns/06-memory-model.md)) |
| [The Go Spec](https://go.dev/ref/spec) | genuinely readable, about 90 pages, and settles arguments definitively |
| [pkg.go.dev](https://pkg.go.dev) | the docs for every public module; read the stdlib source from here — it is the best Go you will read |
| `go doc <pkg>` / `go doc <pkg>.<Sym>` | the same, offline, from your terminal ([2.7](../02-syntax-basics/07-comments-docs.md)) |

The single highest-return habit: when you are unsure how something behaves, **read the standard
library's implementation of it**. `net/http`, `database/sql` and `sync` are all approachable,
heavily commented, and written by people solving the problem you are solving.

## Quick Reference

| Need | Stdlib | If the stdlib is not enough |
|---|---|---|
| routing | `net/http.ServeMux` | `chi` |
| SQL | `database/sql` | `pgx` (+ `sqlc` or `sqlx`) |
| migrations | — | `golang-migrate`, `goose` |
| config | `os` + `flag` | `envconfig`, `koanf` |
| logging | `log/slog` | `zerolog`, `zap` |
| errors | `errors`, `%w` | nothing — **never** `pkg/errors` |
| validation | hand-written | `go-playground/validator` |
| JWT / passwords | — / — | `golang-jwt/jwt/v5`, `x/crypto/bcrypt` |
| HTTP client | `net/http` | `resty` |
| JSON | `encoding/json` | `sonic` (only after profiling) |
| tests | `testing`, `httptest` | `go-cmp`, `goleak`, `testcontainers-go` |
| parallel work | `sync`, `context` | `x/sync/errgroup`, `x/sync/singleflight` |
| rate limiting | — | `x/time/rate` |
| metrics / traces | `expvar`, `pprof` | `prometheus/client_golang`, `otel` |
| CLI | `flag` | `cobra` |
| templates | `html/template` | `templ` |
| job queue | `time.Ticker` | `river` (Postgres), `asynq` (Redis) |
| cache | `map` + `sync.RWMutex` | `ristretto`, `go-redis` |
| IDs | `crypto/rand` | `google/uuid` (v7), `oklog/ulid` |
| DI | hand-wiring in `main` | `wire`, `fx` |

| Evaluation check | Command / question |
|---|---|
| already in stdlib? | `go doc <stdlib pkg>`; check recent release notes |
| transitive cost | `go mod graph \| wc -l`, `go list -m all \| wc -l` |
| vulnerabilities | `go run golang.org/x/vuln/cmd/govulncheck@latest ./...` |
| why is it in my build? | `go mod why <module>` |
| stability | is it v1+? does it tag releases? |
| maintenance | last commit, unanswered issues, number of maintainers |
| alternative | could 40 lines I understand replace it? |

## Common Mistakes

**Adding a web framework before trying `net/http` plus chi.** You inherit a non-stdlib handler
type and a middleware ecosystem you must stay inside, for routing features you may not need.

**Using `github.com/pkg/errors` in new code.** It is archived and its features are in the
standard library; use `errors` and `%w`.

**Reaching for an ORM because writing SQL feels repetitive.** The repetition is in the
*scanning*, which sqlc or sqlx removes without hiding the queries.

**Adding a faster JSON library before profiling.** The bottleneck is almost always the database
or the network, and you have taken on assembly and `unsafe` for nothing.

**Running migrations from your service's startup path.** Five replicas race on every deploy; run
them as a separate step.

**Random UUIDv4 as a database primary key at scale.** Inserts scatter across the B-tree; use
UUIDv7 or ULID.

**Choosing a CGO-based client (confluent-kafka-go) while targeting a `scratch` image.** The
binary is no longer static and the container will not start.

**Taking a 27-module dependency for one helper function.** Copy the function with its license,
or write the forty lines.

## Exercises

1. Audit a real Go project of yours: run `go list -m all | wc -l`, then `go mod why` on each
   direct dependency. For every one, write down what it does that the stdlib does not, and
   remove any where you cannot answer. Report the before/after module count and binary size.
2. Replace one dependency with stdlib code — a UUID generator with `crypto/rand`, a config
   library with `os.LookupEnv`, an assertion library with plain `if` statements — and measure
   the change in `go.sum` lines, `go mod graph` edges and build time.
3. Run `govulncheck` on a module pinned to a deliberately old dependency version, capture the
   real finding, then use `go mod why` to explain the path by which it entered your build.
4. Build project 1 (the URL shortener) end to end with this module's stack: 18.1's layout,
   18.2's config, 18.3's auth, 18.4's logging, 18.5's shutdown, 18.6's probes, 18.7's metrics,
   18.8's image and 18.9's CI. Make every gate in CI green before you add the second feature.
5. Read the standard library's `net/http.(*ServeMux).ServeHTTP` and `sync.Once.Do`, then write
   a one-page explanation of each to someone who has finished this course. If you cannot, read
   them again — that is the skill this course was pointing at.

---

[← 18.9 CI with GitHub Actions](09-ci.md) · [Course Index](../README.md)
