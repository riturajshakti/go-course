# The Go Tour

A hands-on Go course for developers who already ship software in another language.
Every lesson is short, example-driven, and shows real program output.

- **Target toolchain:** Go 1.27.1, modules (no GOPATH)
- **Backend stack taught:** net/http + chi + pgx + Postgres
- **Format:** 18 modules, 143 lessons. Read in order; each lesson stands alone.

Check the `Done` column to track your progress as files land.

---

## Module 01 — Getting Started

| # | Lesson | Done |
|---|--------|------|
| 1.1 | [Installing Go and the Toolchain](01-getting-started/01-install-toolchain.md) | ✅ |
| 1.2 | [Hello, Modules: Your First Project](01-getting-started/02-hello-modules.md) | ✅ |
| 1.3 | [go.mod, go.sum, and the Module Graph](01-getting-started/03-go-mod.md) | ✅ |
| 1.4 | [Packages, Imports, and Visibility](01-getting-started/04-packages-imports.md) | ✅ |
| 1.5 | [gofmt, go vet, and the Editor Setup](01-getting-started/05-tooling.md) | ✅ |
| 1.6 | [How Go Thinks: A Mental Model](01-getting-started/06-mental-model.md) | ✅ |

## Module 02 — Syntax Basics

| # | Lesson | Done |
|---|--------|------|
| 2.1 | [Variables, Zero Values, and Declaration Forms](02-syntax-basics/01-variables.md) | ✅ |
| 2.2 | [Numeric Types, Overflow, and Conversion](02-syntax-basics/02-numeric-types.md) | ✅ |
| 2.3 | [Booleans, Operators, and Precedence](02-syntax-basics/03-operators.md) | ✅ |
| 2.4 | [Constants, Typed vs Untyped, and iota](02-syntax-basics/04-constants-iota.md) | ✅ |
| 2.5 | [Functions, Multiple Returns, and Named Results](02-syntax-basics/05-functions.md) | ✅ |
| 2.6 | [Variadic Functions and Closures](02-syntax-basics/06-variadic-closures.md) | ✅ |
| 2.7 | [Comments, Doc Comments, and go doc](02-syntax-basics/07-comments-docs.md) | ✅ |

## Module 03 — Control Flow

| # | Lesson | Done |
|---|--------|------|
| 3.1 | [if, else, and the if-with-statement Form](03-control-flow/01-if.md) | ✅ |
| 3.2 | [Every for Loop Form](03-control-flow/02-for-loops.md) | ✅ |
| 3.3 | [Every switch Form](03-control-flow/03-switch.md) | ✅ |
| 3.4 | [goto, Labels, break, and continue](03-control-flow/04-labels-goto.md) | ✅ |
| 3.5 | [defer: Order, Arguments, and Gotchas](03-control-flow/05-defer.md) | ✅ |
| 3.6 | [panic and recover](03-control-flow/06-panic-recover.md) | ✅ |

## Module 04 — Types, Structs, and Methods

| # | Lesson | Done |
|---|--------|------|
| 4.1 | [Pointers and the Absence of Pointer Arithmetic](04-types-structs/01-pointers.md) | ✅ |
| 4.2 | [Defining and Instantiating Structs](04-types-structs/02-structs.md) | ✅ |
| 4.3 | [Struct Comparison, Copying, and Memory Layout](04-types-structs/03-struct-semantics.md) | ✅ |
| 4.4 | [Methods: Value vs Pointer Receivers](04-types-structs/04-methods.md) | ✅ |
| 4.5 | [Struct Embedding and Promotion](04-types-structs/05-embedding.md) | ✅ |
| 4.6 | [Named Types, Type Definitions, and Aliases](04-types-structs/06-named-types.md) | ✅ |
| 4.7 | [Struct Tags](04-types-structs/07-struct-tags.md) | ✅ |
| 4.8 | [Constructors and the Functional Options Pattern](04-types-structs/08-constructors-options.md) | ✅ |

## Module 05 — Interfaces

| # | Lesson | Done |
|---|--------|------|
| 5.1 | [Implicit Satisfaction](05-interfaces/01-implicit-satisfaction.md) | ✅ |
| 5.2 | [Interface Values, nil, and the Typed-nil Trap](05-interfaces/02-interface-values-nil.md) | ✅ |
| 5.3 | [Type Assertions and Type Switches](05-interfaces/03-assertions-type-switches.md) | ✅ |
| 5.4 | [The Empty Interface and any](05-interfaces/04-empty-interface.md) | ✅ |
| 5.5 | [Interface Composition and Small Interfaces](05-interfaces/05-composition.md) | ✅ |
| 5.6 | [Standard Library Interfaces Worth Knowing](05-interfaces/06-stdlib-interfaces.md) | ✅ |

## Module 06 — Strings, Runes, and Bytes

| # | Lesson | Done |
|---|--------|------|
| 6.1 | [string, []byte, and []rune](06-strings/01-string-bytes-runes.md) | |
| 6.2 | [UTF-8, Indexing, and Iteration](06-strings/02-utf8-iteration.md) | ✅ |
| 6.3 | [The strings Package Reference](06-strings/03-strings-package.md) | ✅ |
| 6.4 | [strings.Builder, Replacer, and Reader](06-strings/04-builder-replacer.md) | ✅ |
| 6.5 | [The bytes Package Reference](06-strings/05-bytes-package.md) | ✅ |
| 6.6 | [strconv: Parsing and Formatting](06-strings/06-strconv.md) | ✅ |
| 6.7 | [fmt: Every Verb and Flag](06-strings/07-fmt-verbs.md) | ✅ |
| 6.8 | [unicode, utf8, and Text Edge Cases](06-strings/08-unicode-utf8.md) | ✅ |

## Module 07 — Arrays, Slices, and Maps

| # | Lesson | Done |
|---|--------|------|
| 7.1 | [Arrays: Fixed Size, Value Semantics](07-slices-maps/01-arrays.md) | ✅ |
| 7.2 | [Slices: len, cap, and the Slice Header](07-slices-maps/02-slices-basics.md) | ✅ |
| 7.3 | [append Semantics and Reallocation](07-slices-maps/03-append.md) | ✅ |
| 7.4 | [The Shared Backing Array Trap](07-slices-maps/04-shared-backing.md) | ✅ |
| 7.5 | [copy, Three-Index Slicing, and Deletion](07-slices-maps/05-copy-three-index.md) | ✅ |
| 7.6 | [Maps: Every Operation](07-slices-maps/06-maps.md) | ✅ |
| 7.7 | [Map Keys, Nested Maps, and Sets](07-slices-maps/07-map-patterns.md) | ✅ |
| 7.8 | [The slices Package Reference](07-slices-maps/08-slices-package.md) | ✅ |
| 7.9 | [The maps Package Reference](07-slices-maps/09-maps-package.md) | ✅ |
| 7.10 | [sort: Before and Alongside Generics](07-slices-maps/10-sort.md) | ✅ |

## Module 08 — Errors

| # | Lesson | Done |
|---|--------|------|
| 8.1 | [The error Interface and errors.New](08-errors/01-error-interface.md) | ✅ |
| 8.2 | [Wrapping with fmt.Errorf and %w](08-errors/02-wrapping.md) | ✅ |
| 8.3 | [errors.Is, errors.As, and Unwrap](08-errors/03-is-as-unwrap.md) | ✅ |
| 8.4 | [Sentinel Errors and Custom Error Types](08-errors/04-sentinels-custom.md) | ✅ |
| 8.5 | [errors.Join and Multiple Errors](08-errors/05-join.md) | ✅ |
| 8.6 | [Error Handling Strategy for Libraries vs Services](08-errors/06-strategy.md) | ✅ |

## Module 09 — Generics

| # | Lesson | Done |
|---|--------|------|
| 9.1 | [Type Parameters on Functions](09-generics/01-type-parameters.md) | ✅ |
| 9.2 | [Constraints, comparable, and cmp.Ordered](09-generics/02-constraints.md) | ✅ |
| 9.3 | [Generic Types and Methods](09-generics/03-generic-types.md) | ✅ |
| 9.4 | [Type Inference and Its Limits](09-generics/04-inference.md) | ✅ |
| 9.5 | [When Generics Are the Wrong Tool](09-generics/05-when-not-to.md) | ✅ |

## Module 10 — Goroutines and Channels

| # | Lesson | Done |
|---|--------|------|
| 10.1 | [Goroutines and the Scheduler](10-concurrency-basics/01-goroutines.md) | ✅ |
| 10.2 | [Unbuffered Channels](10-concurrency-basics/02-unbuffered-channels.md) | ✅ |
| 10.3 | [Buffered Channels](10-concurrency-basics/03-buffered-channels.md) | ✅ |
| 10.4 | [Closing, range over a Channel, and the comma-ok Receive](10-concurrency-basics/04-closing-range.md) | ✅ |
| 10.5 | [Directional Channels and nil Channels](10-concurrency-basics/05-directional-nil.md) | ✅ |
| 10.6 | [select, default, and Timeouts](10-concurrency-basics/06-select.md) | ✅ |

## Module 11 — Synchronization and Context

| # | Lesson | Done |
|---|--------|------|
| 11.1 | [sync.WaitGroup](11-sync-context/01-waitgroup.md) | ✅ |
| 11.2 | [sync.Mutex and sync.RWMutex](11-sync-context/02-mutex.md) | |
| 11.3 | [sync.Once, sync.Pool, and sync.Map](11-sync-context/03-once-pool-map.md) | |
| 11.4 | [sync/atomic](11-sync-context/04-atomic.md) | |
| 11.5 | [The Race Detector](11-sync-context/05-race-detector.md) | |
| 11.6 | [context: Cancellation and Timeouts](11-sync-context/06-context-cancel.md) | |
| 11.7 | [context Values and Propagation](11-sync-context/07-context-values.md) | |
| 11.8 | [errgroup](11-sync-context/08-errgroup.md) | |

## Module 12 — Concurrency Patterns

| # | Lesson | Done |
|---|--------|------|
| 12.1 | [Worker Pools](12-concurrency-patterns/01-worker-pools.md) | ✅ |
| 12.2 | [Fan-Out and Fan-In](12-concurrency-patterns/02-fan-out-fan-in.md) | |
| 12.3 | [Pipelines and Cancellation](12-concurrency-patterns/03-pipelines.md) | |
| 12.4 | [Rate Limiting and Semaphores](12-concurrency-patterns/04-rate-limiting.md) | |
| 12.5 | [Deadlocks, Leaks, and How to Spot Them](12-concurrency-patterns/05-deadlocks-leaks.md) | |
| 12.6 | [The Go Memory Model in Practice](12-concurrency-patterns/06-memory-model.md) | |

## Module 13 — Standard Library Tour

| # | Lesson | Done |
|---|--------|------|
| 13.1 | [os: Files, Args, and Environment](13-stdlib/01-os.md) | |
| 13.2 | [io: Reader, Writer, and the Helpers](13-stdlib/02-io.md) | |
| 13.3 | [bufio: Scanner, Reader, and Writer](13-stdlib/03-bufio.md) | |
| 13.4 | [time: Durations, Instants, and Formatting](13-stdlib/04-time.md) | |
| 13.5 | [time: Timers, Tickers, and After](13-stdlib/05-timers.md) | |
| 13.6 | [encoding/json: Marshal and Unmarshal](13-stdlib/06-json.md) | |
| 13.7 | [encoding/json: Streaming, Tags, and Custom Marshalers](13-stdlib/07-json-advanced.md) | |
| 13.8 | [regexp](13-stdlib/08-regexp.md) | |
| 13.9 | [math, math/big, and math/rand](13-stdlib/09-math-rand.md) | |
| 13.10 | [path/filepath and io/fs](13-stdlib/10-filepath-fs.md) | |
| 13.11 | [flag: Command-Line Parsing](13-stdlib/11-flag.md) | |
| 13.12 | [log/slog: Structured Logging](13-stdlib/12-slog.md) | |
| 13.13 | [embed: Embedding Files in the Binary](13-stdlib/13-embed.md) | |
| 13.14 | [iter: Range-over-Func Iterators](13-stdlib/14-iter.md) | |
| 13.15 | [reflect: The Basics](13-stdlib/15-reflect.md) | |

## Module 14 — Testing

| # | Lesson | Done |
|---|--------|------|
| 14.1 | [go test and the Basics](14-testing/01-go-test.md) | |
| 14.2 | [Table-Driven Tests and Subtests](14-testing/02-table-driven.md) | |
| 14.3 | [Fixtures, Helpers, TestMain, and Cleanup](14-testing/03-fixtures-helpers.md) | |
| 14.4 | [testify, Briefly](14-testing/04-testify.md) | |
| 14.5 | [Benchmarks](14-testing/05-benchmarks.md) | |
| 14.6 | [Fuzzing](14-testing/06-fuzzing.md) | |
| 14.7 | [Examples as Tests and Coverage](14-testing/07-examples-coverage.md) | |
| 14.8 | [Test Doubles with Interfaces](14-testing/08-test-doubles.md) | |

## Module 15 — Tooling and Builds

| # | Lesson | Done |
|---|--------|------|
| 15.1 | [go build, run, install, and the Build Cache](15-tooling/01-build-run-install.md) | |
| 15.2 | [go mod tidy, vendor, and Dependency Management](15-tooling/02-mod-tidy-vendor.md) | |
| 15.3 | [go vet, staticcheck, and golangci-lint](15-tooling/03-linters.md) | |
| 15.4 | [Build Tags and Conditional Compilation](15-tooling/04-build-tags.md) | |
| 15.5 | [Cross-Compilation and Static Binaries](15-tooling/05-cross-compilation.md) | |
| 15.6 | [pprof Profiling](15-tooling/06-pprof.md) | |
| 15.7 | [Delve, Briefly](15-tooling/07-delve.md) | |
| 15.8 | [go generate and Workspaces](15-tooling/08-generate-workspaces.md) | |

## Module 16 — HTTP Services

| # | Lesson | Done |
|---|--------|------|
| 16.1 | [Your First net/http Server](16-http/01-first-server.md) | |
| 16.2 | [ServeMux Routing: Methods and Wildcards](16-http/02-servemux-routing.md) | |
| 16.3 | [Handlers, HandlerFunc, and the Interface](16-http/03-handlers.md) | |
| 16.4 | [Request Parsing: Path, Query, Headers, Body](16-http/04-request-parsing.md) | |
| 16.5 | [JSON Requests and Responses](16-http/05-json-api.md) | |
| 16.6 | [Middleware from Scratch](16-http/06-middleware.md) | |
| 16.7 | [chi: Routing, Groups, and Middleware](16-http/07-chi.md) | |
| 16.8 | [Request Validation](16-http/08-validation.md) | |
| 16.9 | [HTTP Errors and Problem Responses](16-http/09-error-responses.md) | |
| 16.10 | [The net/http Client](16-http/10-http-client.md) | |
| 16.11 | [Testing Handlers with httptest](16-http/11-httptest.md) | |
| 16.12 | [Context Propagation Through Handlers](16-http/12-context-propagation.md) | |

## Module 17 — Databases

| # | Lesson | Done |
|---|--------|------|
| 17.1 | [database/sql and the Driver Model](17-database/01-database-sql.md) | |
| 17.2 | [pgx: Connecting to Postgres](17-database/02-pgx.md) | |
| 17.3 | [Queries, Scanning, and NULL Handling](17-database/03-queries-scanning.md) | |
| 17.4 | [Transactions](17-database/04-transactions.md) | |
| 17.5 | [Connection Pooling and Tuning](17-database/05-pooling.md) | |
| 17.6 | [Migrations with golang-migrate](17-database/06-migrations.md) | |
| 17.7 | [The Repository Layer](17-database/07-repository.md) | |
| 17.8 | [Testing Against a Real Database](17-database/08-testing-db.md) | |

## Module 18 — Production Go

| # | Lesson | Done |
|---|--------|------|
| 18.1 | [Project Layout: cmd/ and internal/](18-production/01-project-layout.md) | |
| 18.2 | [Configuration via Environment](18-production/02-config.md) | |
| 18.3 | [Authentication with JWT](18-production/03-auth-jwt.md) | |
| 18.4 | [Structured Logging in a Service](18-production/04-service-logging.md) | |
| 18.5 | [Graceful Shutdown](18-production/05-graceful-shutdown.md) | |
| 18.6 | [Health Checks and Readiness](18-production/06-health-checks.md) | |
| 18.7 | [Metrics with Prometheus](18-production/07-prometheus.md) | |
| 18.8 | [Docker Multi-Stage Builds](18-production/08-docker.md) | |
| 18.9 | [CI with GitHub Actions](18-production/09-ci.md) | |
| 18.10 | [The Essential Package Directory](18-production/10-package-directory.md) | |

---

## Lesson Format

Every lesson follows the same shape so you always know where to look:

1. **What You'll Learn** — the 3-4 takeaways
2. **Coming From Another Language** — how this maps from JS/TS, Python, or Rust
3. **Topic sections** — named for what they show, code first
4. **Quick Reference** — a table of the syntax and functions covered
5. **Common Mistakes** — the errors you will hit, and the fix
6. **Exercises** — a few problems to confirm it stuck

Every code block that produces output is followed by that program's real output.
All examples were run on Go 1.27.1 and checked with `gofmt` and `go vet`.

## How to Use This Course

```bash
# Create a scratch module to try the examples in
mkdir go-practice && cd go-practice
go mod init example.com/practice

# Run examples as you read
go run .

# Keep these on while you learn
gofmt -l .
go vet ./...
go test ./... -race
```

Work the modules in order. Modules 01-09 are the foundation. Modules 10-12 are
concurrency, which is the part of Go that most repays slow reading even if you
have used async elsewhere. Modules 16-18 are where your full-stack experience
starts paying off directly.
