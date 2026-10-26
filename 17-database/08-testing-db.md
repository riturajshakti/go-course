[← 17.7 The Repository Layer](07-repository.md) · [Course Index](../README.md) · [18.1 Project Layout: cmd/ and internal/ →](../18-production/01-project-layout.md)

# 17.8 — Testing Against a Real Database

## What You'll Learn

- The pyramid for data access: fast fake-backed unit tests, then integration tests against real Postgres — with **real** `go test` output for both halves
- `//go:build integration` gating, proven with `go list` showing which files the tag adds
- Per-test isolation by **transaction rollback** — the fastest real pattern — versus TRUNCATE versus per-test schemas
- Why mocking `database/sql` with sqlmock usually tests the wrong thing

The fake-backed tests, the build-tag gating and the `go list` comparison here
are real. The testcontainers harness compiles and vets clean under the
`integration` tag but needs a Docker daemon, which this environment does not
have, so its output is labelled.

## Coming From Another Language

| Other | Go |
|---|---|
| Jest + an in-memory mock of the ORM | an in-memory type implementing your own interface |
| `pytest` fixtures with `db` injected | `TestMain` for process-wide setup, `t.Cleanup` per test |
| `@pytest.mark.integration` | `//go:build integration` — the file is not even **compiled** |
| `transaction=True` rolling back each test | a `pgx.Tx` registered with `t.Cleanup(tx.Rollback)` |
| `testcontainers` in JS/Python | `testcontainers-go`, same idea |
| an in-memory SQLite substituting for Postgres | **do not**; the SQL dialects differ where it matters |

## The pyramid

| Layer | Tests | Speed | Catches | Misses |
|---|---|---|---|---|
| **unit, fake repository** | service logic, validation, error mapping | ~ms | business rules, transaction boundaries, cancellation | all SQL |
| **integration, real Postgres** | the repository's SQL | ~100ms + container start | syntax, constraints, types, NULLs, migrations | load behaviour |
| **end-to-end, HTTP + database** | a handful of critical paths | seconds | wiring, serialization | — |

Most of your coverage belongs at the top, and most of your *value* for the
repository layer is in the middle. The thing to avoid is a suite that is 90%
integration tests, because a four-minute test run stops being run.

## Why sqlmock is usually the wrong trade

`go-sqlmock` lets you assert that your code issued a particular SQL string and
return canned rows, with no database. That sounds like the best of both worlds,
and here is the honest problem: it tests the **string you wrote**, not the query
the database would run. A test asserting
`SELECT id, email FROM users WHERE id = \?` passes whether or not that SQL is
valid Postgres, whether or not the column exists, whether or not the index you
need is there, whether or not `email` is nullable, and whether or not a
migration dropped the table. You end up with a test that fails whenever you
reformat the SQL and passes whenever you break it — the exact inverse of what
you want. Worse, writing one couples your test to the query text, so the
refactoring the test is supposed to protect becomes the thing it blocks. For
service logic, a fake implementing your own interface is simpler *and* truer;
for SQL, only a real server tells you anything. The narrow case where sqlmock
earns its place is forcing a driver-level failure you cannot otherwise
provoke — a connection reset mid-`Rows`, say — and even then a fake of your own
interface usually gets you there (17.7's `statusFailingRepo` does it in four
lines).

## Layer one: the fake, for real

17.7's `InMemoryUserRepository` and the eleven tests against it run with no
database and no container. That is the suite that runs on every save:

```
$ go test ./internal/user/ -v
=== RUN   TestRegisterHappyPath
    service_test.go:32: created: {ID:1 OrgID:1 Email:ada@example.com Name:Ada Lovelace Bio:<nil> Status:active CreatedAt:2026-03-01 12:00:00 +0000 UTC}
--- PASS: TestRegisterHappyPath (0.00s)
=== RUN   TestRegisterRejectsDuplicateEmail
    service_test.go:46: err = CreateUser: user already exists
--- PASS: TestRegisterRejectsDuplicateEmail (0.00s)
=== RUN   TestRegisterRejectsUnknownOrg
    service_test.go:56: err = CreateUser: org not found
--- PASS: TestRegisterRejectsUnknownOrg (0.00s)
=== RUN   TestRegisterRollsBackWhenTheOrgIsFull
    service_test.go:81: err = Register: org 1 has 3 of 3 seats: org has reached its seat limit; count stayed at 3
--- PASS: TestRegisterRollsBackWhenTheOrgIsFull (0.00s)
=== RUN   TestRegisterIsAtomicAcrossTwoWrites
    service_test.go:98: err = deadlock detected; no partial row left behind
--- PASS: TestRegisterIsAtomicAcrossTwoWrites (0.00s)
=== RUN   TestActivateIsIdempotent
    service_test.go:132: status after two Activate calls: "active"
--- PASS: TestActivateIsIdempotent (0.00s)
=== RUN   TestActivateUnknownUser
    service_test.go:141: err = GetUser: user not found
--- PASS: TestActivateUnknownUser (0.00s)
=== RUN   TestPageWalksTheWholeSetWithACursor
    service_test.go:164: page 1: 2 users, next=2 more=true
    service_test.go:164: page 2: 2 users, next=4 more=true
    service_test.go:164: page 3: 1 users, next=5 more=false
    service_test.go:182: walked 5 users with no duplicates and no gaps
--- PASS: TestPageWalksTheWholeSetWithACursor (0.00s)
=== RUN   TestListUsersByIDsAvoidsNPlusOne
    service_test.go:207: one call returned 3 users; nil ids returned nil with no query
--- PASS: TestListUsersByIDsAvoidsNPlusOne (0.00s)
=== RUN   TestRepositoryHonoursContextCancellation
    service_test.go:220: every method checks ctx.Err() first, like a real driver would
--- PASS: TestRepositoryHonoursContextCancellation (0.00s)
=== RUN   TestCreateUserRejectsMissingEmail
    service_test.go:229: err = CreateUser: column "email" must not be null: user violates a database constraint
--- PASS: TestCreateUserRejectsMissingEmail (0.00s)
PASS
ok  	example.com/shop/internal/user	0.496s
```

Add 17.4's seven `withTx` tests, 17.4's error-mapping table and 17.7's
SQL-builder tests, and the whole suite is:

```
$ go test ./... -cover -count=1
ok  	example.com/shop/internal/user	0.527s	coverage: 73.0% of statements
```

```
$ go test ./... -race -count=1
ok  	example.com/shop/internal/user	1.623s
```

Clean under `-race`, which matters because the in-memory repository has a
mutex and the service may be called concurrently.

That is the base of the pyramid and it should stay that way: sub-second,
runnable offline, runnable on a plane.

## Layer two: gating integration tests

A build tag keeps the heavy tests out of the default run. The tag goes on the
first line, followed by a blank line:

```go
//go:build integration

package shoptest
```

The blank line is required — without it the comment is a doc comment for the
package and the constraint is silently ignored, which is a bug you will not
notice because the tests simply always run.

The payoff, with real output. A package with one unit test and one
tag-gated integration test:

```go
package tagdemo

import "testing"

func TestUnitAlwaysRuns(t *testing.T) {
  t.Log("no database, no Docker, no build tag")
}
```

```go
//go:build integration

package tagdemo

import (
  "os"
  "testing"
)

func TestIntegrationNeedsATag(t *testing.T) {
  if os.Getenv("TEST_DATABASE_URL") == "" {
    t.Skip("TEST_DATABASE_URL not set; start the compose stack or let testcontainers run")
  }
  t.Log("would run against a real Postgres")
}
```

The default run:

```
$ go test -v ./p_tagdemo
=== RUN   TestUnitAlwaysRuns
    unit_test.go:6: no database, no Docker, no build tag
--- PASS: TestUnitAlwaysRuns (0.00s)
PASS
ok  	example.com/m17scratch/p_tagdemo	0.497s
```

With the tag, and no DSN set:

```
$ go test -v -tags=integration ./p_tagdemo
=== RUN   TestIntegrationNeedsATag
    integration_test.go:12: TEST_DATABASE_URL not set; start the compose stack or let testcontainers run
--- SKIP: TestIntegrationNeedsATag (0.00s)
=== RUN   TestUnitAlwaysRuns
    unit_test.go:6: no database, no Docker, no build tag
--- PASS: TestUnitAlwaysRuns (0.00s)
PASS
ok  	example.com/m17scratch/p_tagdemo	0.492s
```

And the mechanism made visible:

```
$ go list -f '{{.TestGoFiles}}' ./p_tagdemo
[unit_test.go]
$ go list -tags=integration -f '{{.TestGoFiles}}' ./p_tagdemo
[integration_test.go unit_test.go]
```

That last pair is the point. Without the tag, `integration_test.go` is **not
part of the package** — it is not compiled, so its imports
(`testcontainers-go`, and everything that pulls in) are not built either. A
`t.Skip` at the top of an untagged file would still compile the file and its
dependency tree, which is slow and means a broken integration test breaks your
unit test build.

Note the two layers of guard, which do different jobs:

- the **build tag** decides whether the code is compiled at all
- the **`t.Skip`** decides, at runtime, whether a compiled test can actually
  run

Keep both. The tag is for "do not slow down the normal run"; the skip is for
"the tag is on, but there is no database here", which is what makes
`-tags=integration` safe to run anywhere. Note `-tags` requires a rebuild, so
the first tagged run is slower ([15.4](../15-tooling/04-build-tags.md)).

```bash
go test ./...
go test -tags=integration ./...
go test -tags=integration -run TestCreateAndGetUser ./internal/postgres/
TEST_DATABASE_URL='postgres://app:s3cret@localhost:5432/shop?sslmode=disable' \
  go test -tags=integration ./...
```

## Isolation: three strategies

Every integration test must start from a known state. Three ways, and the first
is the one to use:

| Strategy | Speed | Isolation | Parallel-safe | Catches |
|---|---|---|---|---|
| **transaction rollback** | **fastest** — no I/O to undo | per test | **yes**, each tx is independent | everything except commit-time behaviour |
| `TRUNCATE` between tests | fast-ish; one statement per table | per test | **no** — one test truncates another's rows | everything, including commits and triggers |
| a schema per test | slow — DDL per test | per test | yes | everything |
| a database per test | slowest | total | yes | everything |

### Transaction rollback

Each test runs inside a transaction that is **always** rolled back. Nothing is
ever committed, so there is nothing to clean up:

```go
func inTx(t *testing.T) pgx.Tx {
  t.Helper()
  ctx := context.Background()
  tx, err := testPool.Begin(ctx)
  if err != nil {
    t.Fatalf("begin: %v", err)
  }
  t.Cleanup(func() {
    if err := tx.Rollback(ctx); err != nil && !errors.Is(err, pgx.ErrTxClosed) {
      t.Errorf("rollback: %v", err)
    }
  })
  return tx
}
```

Eleven lines, and they are the most valuable eleven in the lesson. Four
details:

**`t.Cleanup` rather than `defer`** ([14.3](../14-testing/03-fixtures-helpers.md)),
so the helper can be called from a helper and the rollback still runs at the
end of the *test*, not at the end of `inTx`.

**`t.Helper()`** so a failure inside it is reported at the caller's line.

**`pgx.ErrTxClosed` is tolerated,** because a test may legitimately have
committed or the transaction may have been aborted by a failed statement (17.4).

**The `tx` is what the test uses** — and because the repository takes a `DBTX`
(17.7), it slots straight in:

```go
func TestRepositoryGetUser(t *testing.T) {
  tx := inTx(t)
  repo := postgres.NewPostgresUserRepository(tx)
  ctx := context.Background()

  orgID := seedOrg(t, tx, "Acme")
  created, err := repo.CreateUser(ctx, user.User{
    OrgID: orgID, Email: "ada@example.com", Name: "Ada", Status: "pending",
  })
  if err != nil {
    t.Fatal(err)
  }

  got, err := repo.GetUser(ctx, created.ID)
  if err != nil {
    t.Fatal(err)
  }
  if got.Email != "ada@example.com" {
    t.Fatalf("Email = %q", got.Email)
  }
  if got.CreatedAt.IsZero() {
    t.Error("CreatedAt should be set by the database default")
  }
}
```

That `DBTX` parameter from 17.4, which existed so the service could own
transactions, turns out to be what makes the repository testable. Worth
noticing: designing for one reason paid off for another.

**The limits, stated honestly.** Nothing is committed, so this pattern cannot
test deferred constraints, `AFTER` triggers that fire at commit, `NOTIFY`
delivery, `SERIALIZABLE` conflict detection (which happens at commit), or
anything involving two concurrent transactions. For those, use TRUNCATE
isolation and a non-parallel test.

### TRUNCATE between tests

```go
func truncateAll(t *testing.T) {
  t.Helper()
  ctx := context.Background()
  _, err := testPool.Exec(ctx,
    `TRUNCATE users, orgs RESTART IDENTITY CASCADE`)
  if err != nil {
    t.Fatalf("truncating: %v", err)
  }
}

func TestCommitBehaviour(t *testing.T) {
  truncateAll(t)
  t.Cleanup(func() { truncateAll(t) })
  // ... real commits here
}
```

`RESTART IDENTITY` resets the sequences so IDs are predictable;
`CASCADE` handles the foreign keys. Truncating at the **start** as well as in
cleanup makes a test robust against a previous test that crashed.

This cannot run in parallel: one test's `TRUNCATE` deletes another's rows.

### A schema per test

```go
func freshSchema(t *testing.T) string {
  t.Helper()
  ctx := context.Background()
  name := "test_" + strings.ReplaceAll(strings.ToLower(t.Name()), "/", "_")

  if _, err := testPool.Exec(ctx,
    `DROP SCHEMA IF EXISTS `+pgx.Identifier{name}.Sanitize()+` CASCADE`); err != nil {
    t.Fatal(err)
  }
  if _, err := testPool.Exec(ctx,
    `CREATE SCHEMA `+pgx.Identifier{name}.Sanitize()); err != nil {
    t.Fatal(err)
  }
  t.Cleanup(func() {
    testPool.Exec(context.Background(),
      `DROP SCHEMA `+pgx.Identifier{name}.Sanitize()+` CASCADE`)
  })
  return name
}
```

Full isolation including commits, parallel-safe, and you pay for the migrations
per schema. `pgx.Identifier{}.Sanitize()` (17.3) is doing real work here,
because the schema name comes from `t.Name()`.

## Layer two, assembled: testcontainers-go

`TestMain` ([14.3](../14-testing/03-fixtures-helpers.md)) is where process-wide
setup goes: one container, one migration run, one pool, shared by every test in
the package.

```go
//go:build integration

package shoptest

import (
  "context"
  "embed"
  "errors"
  "fmt"
  "os"
  "strings"
  "testing"
  "time"

  "github.com/golang-migrate/migrate/v4"
  _ "github.com/golang-migrate/migrate/v4/database/pgx/v5"
  "github.com/golang-migrate/migrate/v4/source/iofs"
  "github.com/jackc/pgx/v5/pgxpool"
  "github.com/testcontainers/testcontainers-go"
  "github.com/testcontainers/testcontainers-go/modules/postgres"
  "github.com/testcontainers/testcontainers-go/wait"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

var testPool *pgxpool.Pool

func TestMain(m *testing.M) {
  code, err := run(m)
  if err != nil {
    fmt.Fprintf(os.Stderr, "integration setup failed: %v\n", err)
    os.Exit(1)
  }
  os.Exit(code)
}

func run(m *testing.M) (int, error) {
  ctx := context.Background()

  dsn := os.Getenv("TEST_DATABASE_URL")
  if dsn == "" {
    container, err := postgres.Run(ctx, "postgres:17-alpine",
      postgres.WithDatabase("shop_test"),
      postgres.WithUsername("app"),
      postgres.WithPassword("s3cret"),
      testcontainers.WithWaitStrategy(
        wait.ForLog("database system is ready to accept connections").
          WithOccurrence(2).
          WithStartupTimeout(60*time.Second),
      ),
    )
    if err != nil {
      return 0, fmt.Errorf("starting postgres: %w", err)
    }
    defer func() {
      if err := testcontainers.TerminateContainer(container); err != nil {
        fmt.Fprintf(os.Stderr, "terminating container: %v\n", err)
      }
    }()

    dsn, err = container.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
      return 0, fmt.Errorf("connection string: %w", err)
    }
  }

  if err := migrateUp(dsn); err != nil {
    return 0, err
  }

  pool, err := pgxpool.New(ctx, dsn)
  if err != nil {
    return 0, fmt.Errorf("pool: %w", err)
  }
  defer pool.Close()

  pingCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
  defer cancel()
  if err := pool.Ping(pingCtx); err != nil {
    return 0, fmt.Errorf("ping: %w", err)
  }
  testPool = pool

  return m.Run(), nil
}

func migrateUp(dsn string) error {
  src, err := iofs.New(migrationsFS, "migrations")
  if err != nil {
    return fmt.Errorf("loading migrations: %w", err)
  }
  dbURL := "pgx5" + strings.TrimPrefix(dsn, "postgres")
  mig, err := migrate.NewWithSourceInstance("iofs", src, dbURL)
  if err != nil {
    return fmt.Errorf("migrator: %w", err)
  }
  defer mig.Close()
  if err := mig.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
    return fmt.Errorf("migrating: %w", err)
  }
  return nil
}
```

Six things that make this work rather than nearly work.

**`TestMain` delegates to `run` so the `defer`s fire.** `os.Exit` skips
deferred functions, so `TestMain` must not `defer` anything itself — the
container teardown and `pool.Close()` live in `run`, whose return value
`TestMain` then exits with. Getting this wrong leaks a container per test run,
which you discover when `docker ps` has forty of them.

**`TEST_DATABASE_URL` short-circuits the container.** The same tests then run
against the compose stack locally (fast, no container start) and against a CI
service container, with no code change. This is the single most useful line in
the harness.

**The wait strategy uses `WithOccurrence(2)`.** The Postgres image logs
"ready to accept connections" **twice**: once after its internal init run, once
when it is genuinely listening. Waiting for the first occurrence gives you an
intermittent connection-refused on a fast machine, and it is the classic
testcontainers flake.

**Migrations run in setup**, from the same embedded `iofs` source the service
uses (17.6), so the test schema cannot drift from production's. The DSN is
rewritten from `postgres://` to `pgx5://` because golang-migrate picks its
driver from the scheme.

**`Ping` with its own timeout** confirms the database is actually reachable
before any test runs, so a failure is one clear message rather than N confusing
ones.

**The container is created once** for the package. Container start is 1–3
seconds; doing it per test is what makes people give up on integration tests.

And the tests themselves, with fixture helpers:

```go
//go:build integration

package shoptest

import (
  "context"
  "errors"
  "testing"

  "github.com/jackc/pgx/v5"
)

func inTx(t *testing.T) pgx.Tx {
  t.Helper()
  ctx := context.Background()
  tx, err := testPool.Begin(ctx)
  if err != nil {
    t.Fatalf("begin: %v", err)
  }
  t.Cleanup(func() {
    if err := tx.Rollback(ctx); err != nil && !errors.Is(err, pgx.ErrTxClosed) {
      t.Errorf("rollback: %v", err)
    }
  })
  return tx
}

func seedOrg(t *testing.T, tx pgx.Tx, name string) int64 {
  t.Helper()
  var id int64
  err := tx.QueryRow(context.Background(),
    `INSERT INTO orgs (name) VALUES ($1) RETURNING id`, name).Scan(&id)
  if err != nil {
    t.Fatalf("seeding org %q: %v", name, err)
  }
  return id
}

func seedUser(t *testing.T, tx pgx.Tx, orgID int64, email string) int64 {
  t.Helper()
  var id int64
  err := tx.QueryRow(context.Background(),
    `INSERT INTO users (org_id, email, name, status)
     VALUES ($1, $2, $2, 'pending') RETURNING id`, orgID, email).Scan(&id)
  if err != nil {
    t.Fatalf("seeding user %q: %v", email, err)
  }
  return id
}

func TestCreateAndGetUser(t *testing.T) {
  tx := inTx(t)
  ctx := context.Background()

  orgID := seedOrg(t, tx, "Acme")
  id := seedUser(t, tx, orgID, "ada@example.com")

  var email, status string
  err := tx.QueryRow(ctx,
    `SELECT email, status FROM users WHERE id = $1`, id).Scan(&email, &status)
  if err != nil {
    t.Fatal(err)
  }
  if email != "ada@example.com" || status != "pending" {
    t.Fatalf("got %q/%q", email, status)
  }
  t.Logf("user %d created inside a transaction that will be rolled back", id)
}

func TestUniqueEmailIndexIsCaseInsensitive(t *testing.T) {
  tx := inTx(t)
  ctx := context.Background()

  orgID := seedOrg(t, tx, "Acme")
  seedUser(t, tx, orgID, "ada@example.com")

  _, err := tx.Exec(ctx,
    `INSERT INTO users (org_id, email, name, status)
     VALUES ($1, 'ADA@EXAMPLE.COM', 'Impostor', 'pending')`, orgID)
  if err == nil {
    t.Fatal("want a unique violation; the index is on lower(email)")
  }
  t.Logf("real constraint error: %v", err)
}

func TestForeignKeyRejectsUnknownOrg(t *testing.T) {
  tx := inTx(t)
  _, err := tx.Exec(context.Background(),
    `INSERT INTO users (org_id, email, name, status)
     VALUES (999999, 'x@example.com', 'X', 'pending')`)
  if err == nil {
    t.Fatal("want a foreign key violation")
  }
  t.Logf("real constraint error: %v", err)
}
```

The two constraint tests are the whole justification for this layer. The
in-memory fake *claims* email uniqueness is case-insensitive and that `org_id`
has a foreign key, because 17.7 wrote it that way — these tests check that the
**migration actually created those constraints**. A fake and a schema that
disagree is the bug neither layer can catch alone.

Expected output (requires a running Docker daemon; not executed in this environment):

```
$ go test -tags=integration -v ./internal/postgres/
=== RUN   TestCreateAndGetUser
    repo_test.go:67: user 1 created inside a transaction that will be rolled back
--- PASS: TestCreateAndGetUser (0.02s)
=== RUN   TestUniqueEmailIndexIsCaseInsensitive
    repo_test.go:84: real constraint error: ERROR: duplicate key value violates unique constraint "users_email_key" (SQLSTATE 23505)
--- PASS: TestUniqueEmailIndexIsCaseInsensitive (0.01s)
=== RUN   TestForeignKeyRejectsUnknownOrg
    repo_test.go:95: real constraint error: ERROR: insert or update on table "users" violates foreign key constraint "users_org_id_fkey" (SQLSTATE 23503)
--- PASS: TestForeignKeyRejectsUnknownOrg (0.01s)
PASS
ok  	example.com/shop/internal/postgres	3.482s
```

The ~3.5s is almost entirely container start; the tests themselves are
hundredths of a second each, which is what the rollback pattern buys you.

`ory/dockertest` is the older alternative with the same shape and a more manual
API; testcontainers-go is the one to reach for now.

## Asserting on domain errors

```go
// WRONG: brittle, and it tests the message rather than the behaviour.
if !strings.Contains(err.Error(), "duplicate key") {
  t.Fatal("expected a duplicate")
}
```

```go
// Right: errors.Is against the domain sentinel.
if !errors.Is(err, user.ErrDuplicateUser) {
  t.Fatalf("want ErrDuplicateUser, got %v", err)
}
```

The string version breaks when Postgres reword a message, when a locale
changes, or when a layer adds a prefix — and it passes for any error whose text
happens to contain the substring. `errors.Is` sees through `%w` wrapping
([8.3](../08-errors/03-is-as-unwrap.md)) and tests the thing you care about.

For the typed case, `errors.As` plus a field check:

```go
var pg *pgconn.PgError
if !errors.As(err, &pg) {
  t.Fatalf("want a PgError, got %T: %v", err, err)
}
if pg.Code != "23505" {
  t.Errorf("SQLSTATE = %s, want 23505", pg.Code)
}
if pg.ConstraintName != "users_email_key" {
  t.Errorf("constraint = %s", pg.ConstraintName)
}
```

Asserting on `ConstraintName` is the right level of specificity for an
integration test: it proves the right index fired, which is what 17.4's mapper
branches on.

## Parallel tests against one database

`t.Parallel()` is safe with the **rollback** pattern and unsafe with TRUNCATE:

```go
func TestSomethingIsolated(t *testing.T) {
  t.Parallel()
  tx := inTx(t)
  // each parallel test has its own transaction and its own snapshot
}
```

Three hazards even then.

**Pool size.** N parallel tests each hold a connection for the whole test, so
`MaxConns` must exceed your parallelism or tests queue on acquire and
eventually time out (17.5). `go test` defaults `-parallel` to `GOMAXPROCS`, so
set `MaxConns` above that.

**Shared sequences.** `bigserial` values are allocated outside the transaction
and are **not** rolled back, so parallel tests see gaps and
non-deterministic IDs. Never assert `id == 1`; assert on what you inserted.

**Lock contention.** Two parallel tests touching the same seeded row block each
other. Give each test its own rows — seed with `t.Name()` in the email, say —
rather than sharing fixtures.

```go
func seedUniqueUser(t *testing.T, tx pgx.Tx, orgID int64) int64 {
  t.Helper()
  email := strings.ToLower(t.Name()) + "@example.test"
  return seedUser(t, tx, orgID, email)
}
```

Deriving fixture data from `t.Name()` is a small habit that removes a whole
class of parallel flake.

## Golden-file testing of query results

For a query whose shape is the thing under test — a report, an aggregation —
comparing against a checked-in golden file beats hand-written assertions:

```go
var update = flag.Bool("update", false, "rewrite golden files")

func TestOrgReportGolden(t *testing.T) {
  tx := inTx(t)
  ctx := context.Background()
  seedReportFixtures(t, tx)

  rows, err := tx.Query(ctx, orgReportSQL)
  if err != nil {
    t.Fatal(err)
  }
  got, err := pgx.CollectRows(rows, pgx.RowToMap)
  if err != nil {
    t.Fatal(err)
  }

  buf, err := json.MarshalIndent(got, "", "  ")
  if err != nil {
    t.Fatal(err)
  }
  buf = append(buf, '\n')

  golden := filepath.Join("testdata", t.Name()+".json")
  if *update {
    if err := os.MkdirAll("testdata", 0o755); err != nil {
      t.Fatal(err)
    }
    if err := os.WriteFile(golden, buf, 0o644); err != nil {
      t.Fatal(err)
    }
    t.Logf("wrote %s", golden)
    return
  }

  want, err := os.ReadFile(golden)
  if err != nil {
    t.Fatalf("reading golden file (run with -update to create): %v", err)
  }
  if !bytes.Equal(buf, want) {
    t.Errorf("result differs from %s\n got: %s\nwant: %s", golden, buf, want)
  }
}
```

```bash
go test -tags=integration -run TestOrgReportGolden ./internal/postgres/ -update
```

Two requirements, or the golden file is a flake generator. **The query must be
fully ordered** — add `ORDER BY` on enough columns to make the result unique,
because Postgres makes no ordering promise otherwise. And **the data must be
deterministic**: no `now()`, no sequence-generated IDs in the output, no
floating-point aggregates. Exclude those columns from the query, or normalise
them out of the map before marshalling — a raw `bytes.Equal` against a golden
file has no tolerance at all.

The real benefit is reviewability: a schema or query change shows up as a diff
in the pull request, which a reviewer can read.

## CI with a Postgres service container

For CI, a service container beats testcontainers: no Docker-in-Docker, and the
database starts in parallel with your build.

```yaml
name: test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: s3cret
          POSTGRES_DB: shop_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U app -d shop_test"
          --health-interval 2s
          --health-timeout 3s
          --health-retries 15

    steps:
      - uses: actions/checkout@v5

      - uses: actions/setup-go@v6
        with:
          go-version: '1.27.1'
          cache: true

      - name: Verify formatting
        run: test -z "$(gofmt -l .)"

      - name: Vet
        run: go vet ./...

      - name: Vet tagged files too
        run: go vet -tags=integration ./...

      - name: Unit tests
        run: go test ./... -race -count=1

      - name: Integration tests
        env:
          TEST_DATABASE_URL: postgres://app:s3cret@localhost:5432/shop_test?sslmode=disable
        run: go test -tags=integration ./... -count=1
```

Five things that matter in that file.

**The `options:` healthcheck.** Without it the job races the database's startup
and fails intermittently — the same `pg_isready` gate as 17.2's compose file.

**`TEST_DATABASE_URL` is set**, so the harness skips testcontainers entirely and
uses the service container. One harness, two environments.

**`go vet -tags=integration`** is a separate step, because the default `vet`
does not see the tagged files at all — as the `go list` output above proved. An
integration test that does not compile would otherwise pass CI until someone
ran it.

**`-count=1`** disables the test result cache, so a green run means the tests
actually ran.

**`gofmt -l` as a gate**: `test -z "$(gofmt -l .)"` fails when anything is
unformatted ([15.3](../15-tooling/03-linters.md)).

Full CI treatment in [18.9](../18-production/09-ci.md).

## The decision table

| Situation | Use |
|---|---|
| service logic, validation, business rules | **the in-memory fake** |
| transaction boundaries and atomicity | **the fake's `WithTx`** |
| error mapping, SQL building, `Scan`/`Value` | **plain unit tests**, no database |
| does my SQL parse and return the right columns | **integration**, tx rollback |
| do my constraints and indexes exist | **integration**, tx rollback |
| do my migrations apply cleanly | **integration**, `migrateUp` in `TestMain` |
| NULL handling against real columns | **integration**, tx rollback |
| commit-time behaviour, deferred constraints, triggers | **integration**, TRUNCATE, not parallel |
| `SERIALIZABLE` conflicts, deadlocks | **integration**, TRUNCATE, two goroutines |
| `LISTEN`/`NOTIFY` | **integration**, dedicated connections |
| local developer loop | compose stack + `TEST_DATABASE_URL` |
| CI | **service container** + `TEST_DATABASE_URL` |
| a one-off on a laptop with no compose stack | **testcontainers** fallback |
| pool behaviour under load | a load test, not `go test` |
| asserting on a SQL string | **nothing** — delete the test |

The arrangement that works: **one harness, three ways in.** `TestMain` prefers
`TEST_DATABASE_URL`, falls back to testcontainers, and the build tag keeps the
whole thing out of the default run.

## Quick Reference

| Tool | For |
|---|---|
| `//go:build integration` + blank line | excluding the file from compilation |
| `t.Skip` | runtime "no database here" |
| `TestMain(m)` + a `run(m)` helper | one container, one pool, deferred teardown |
| `t.Cleanup(f)` | per-test teardown that survives helper nesting |
| `t.Helper()` | failures reported at the caller |
| `t.Parallel()` | safe with tx rollback, unsafe with TRUNCATE |
| `testPool.Begin` + `t.Cleanup(tx.Rollback)` | the isolation pattern |
| `TRUNCATE ... RESTART IDENTITY CASCADE` | when you need real commits |
| `pgx.Identifier{}.Sanitize()` | a schema name from `t.Name()` |
| `postgres.Run(ctx, "postgres:17-alpine", ...)` | testcontainers-go |
| `wait.ForLog(...).WithOccurrence(2)` | the Postgres image logs ready twice |
| `go list -tags=X -f '{{.TestGoFiles}}'` | confirming what a tag includes |
| `errors.Is` / `errors.As` | assertions; **never** string matching |

| Command | |
|---|---|
| `go test ./...` | unit only |
| `go test -tags=integration ./...` | plus integration |
| `go vet -tags=integration ./...` | vet the tagged files too |
| `go test -tags=integration -run TestX ./pkg/` | one test |
| `go test ./... -race -count=1` | no cache, race detector |
| `go test ... -update` | rewrite golden files |

## Common Mistakes

**Forgetting the blank line after `//go:build integration`.** It becomes a
package doc comment and the constraint is silently ignored, so the tests always
run.

**`defer` inside `TestMain`.** `os.Exit` skips deferred calls, so the container
leaks; put the work in a helper that returns the exit code.

**Waiting for the first "ready to accept connections".** The Postgres image logs
it twice; use `WithOccurrence(2)` or get intermittent refusals.

**Starting a container per test.** Container start dominates; one per package in
`TestMain`.

**Using TRUNCATE isolation with `t.Parallel()`.** One test deletes another's
rows mid-test.

**Asserting on generated IDs.** Sequences are not rolled back, so IDs differ
between runs and with parallelism.

**Asserting on `err.Error()` substrings.** It breaks on a reworded message and
passes for the wrong error; use `errors.Is`.

**A golden file over an unordered query.** Postgres makes no ordering promise,
so the test flakes; add `ORDER BY` over a unique key.

**Not running `go vet -tags=integration` in CI.** A tagged file that does not
compile passes every check until someone runs it.

**Substituting SQLite for Postgres in tests.** The dialects differ exactly where
your bugs are — types, NULLs, constraints, `RETURNING`, `jsonb`.

**Testing SQL strings with sqlmock.** The test fails on reformatting and passes
on a broken query.

## Exercises

1. Build the full three-layer suite for the repository in 17.7: the fake-backed
   unit tests, a tag-gated integration package with `TestMain`, and one
   end-to-end test through an `httptest` server (16.11). Report the wall-clock
   time of each layer.
2. Write one table-driven test body that runs against **both** the in-memory and
   the Postgres repository — the same cases, selected by build tag — and report
   every behavioural divergence you find. Then fix the fake.
3. Prove the rollback pattern's limit: write a test with a deferred constraint
   (`ADD CONSTRAINT ... DEFERRABLE INITIALLY DEFERRED`) that passes under
   rollback isolation and fails under TRUNCATE isolation. Explain why.
4. Convert the integration suite to `t.Parallel()` with rollback isolation,
   measure the speedup at `-parallel 8`, and find the `MaxConns` value below
   which it starts timing out. Report both numbers.
5. Add a CI job that runs the migrations **up, then all the way down, then up
   again** against a fresh container, and prove every down migration from 17.6
   actually works. Report anything that failed.

---

[← 17.7 The Repository Layer](07-repository.md) · [Course Index](../README.md) · [18.1 Project Layout: cmd/ and internal/ →](../18-production/01-project-layout.md)
