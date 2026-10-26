[← 17.3 Queries, Scanning, and NULL Handling](03-queries-scanning.md) · [Course Index](../README.md) · [17.5 Connection Pooling and Tuning →](05-pooling.md)

# 17.4 — Transactions

## What You'll Learn

- The `withTx` helper that commits on success, rolls back on error, and rolls back on panic — written completely and **unit-tested for real against a fake `pgx.Tx`**
- Why `defer tx.Rollback(ctx)` is correct even on the success path, and what it returns then
- SQLSTATE handling: `*pgconn.PgError` with `errors.As`, the `40001` retry loop, and constraint codes mapped to domain errors — all runnable offline
- The `DBTX` interface that lets one repository method work with a pool **or** a transaction

The error-type handling, the retry predicate and the `withTx` helper are
exercised by real tests here. Blocks that need live rows to demonstrate
isolation are labelled.

## Coming From Another Language

| Other | Go |
|---|---|
| `BEGIN`/`COMMIT` as SQL strings | `pool.Begin(ctx)` returns a `pgx.Tx` you call methods on |
| `@transactional` / a decorator | a `withTx(ctx, pool, fn)` higher-order function |
| `with conn.transaction():` | `defer tx.Rollback(ctx)` plus an explicit `Commit` |
| an exception rolling back automatically | nothing is automatic; the `defer` is the mechanism |
| an ORM session tracking the tx for you | you pass the `tx` down explicitly, as an interface |
| `SAVEPOINT` as raw SQL | `tx.Begin(ctx)` on an existing tx **is** a savepoint |

## Begin, Commit, Rollback

```go
tx, err := pool.Begin(ctx)
if err != nil {
  return fmt.Errorf("begin: %w", err)
}
defer tx.Rollback(ctx)

if _, err := tx.Exec(ctx, `UPDATE accounts SET balance = balance - $2 WHERE id = $1`, from, amt); err != nil {
  return err
}
return tx.Commit(ctx)
```

`pgx.Tx` is an **interface**, which is what makes it mockable and what makes
savepoints possible. Its methods are the same as the pool's — `Exec`, `Query`,
`QueryRow`, `SendBatch`, `CopyFrom` — plus `Commit`, `Rollback`, `Begin` and
`Conn`.

### Why the deferred Rollback is correct after Commit

The `defer tx.Rollback(ctx)` runs on **every** path, including the one where
`Commit` already succeeded. That is not a bug, and the reason is worth stating
precisely: **once the transaction is closed, `Rollback` returns an error for
which `errors.Is(err, pgx.ErrTxClosed)` is true, and does nothing else.** No
SQL is sent, no state changes, the commit is not undone. pgx documents this
explicitly so that `defer tx.Rollback(ctx)` is a safe idiom.

What it is not is *tidy*: it discards an error unconditionally, so `errcheck`
and some linters complain, and it cannot roll back on a panic in a way you can
observe. The helper below fixes both.

## withTx: the helper worth writing once

```go
package main

import (
  "context"
  "errors"
  "fmt"

  "github.com/jackc/pgx/v5"
  "github.com/jackc/pgx/v5/pgxpool"
)

type Beginner interface {
  BeginTx(ctx context.Context, opts pgx.TxOptions) (pgx.Tx, error)
}

var _ Beginner = (*pgxpool.Pool)(nil)

func withTx(ctx context.Context, db Beginner, opts pgx.TxOptions, fn func(pgx.Tx) error) (err error) {
  tx, err := db.BeginTx(ctx, opts)
  if err != nil {
    return fmt.Errorf("begin: %w", err)
  }

  defer func() {
    if p := recover(); p != nil {
      _ = tx.Rollback(ctx)
      panic(p)
    }
    if err != nil {
      if rbErr := tx.Rollback(ctx); rbErr != nil && !errors.Is(rbErr, pgx.ErrTxClosed) {
        err = errors.Join(err, fmt.Errorf("rollback: %w", rbErr))
      }
      return
    }
    if cErr := tx.Commit(ctx); cErr != nil {
      err = fmt.Errorf("commit: %w", cErr)
    }
  }()

  return fn(tx)
}
```

Five deliberate decisions in those twenty lines.

**The named return `(err error)`** is what lets the deferred closure both read
and *replace* the error. Without the name, the deferred function cannot change
what the caller sees, so a commit failure would be invisible.

**The commit happens in the `defer`, not after `fn`.** This is the key move: it
means there is exactly one place that decides commit-or-rollback, and the
panic, error and success paths all flow through it.

**A panic rolls back and then re-panics.** Swallowing the panic would turn a
programming bug into a silent partial write. Re-panicking preserves the
behaviour your `Recoverer` middleware expects
([16.7](../16-http/07-chi.md)), with the transaction already cleaned up.

**`pgx.ErrTxClosed` is filtered out of the rollback error.** The transaction
may already be closed because Postgres aborted it, which is normal on a failed
statement; reporting it would bury the real error.

**A commit failure becomes the returned error.** `Commit` can fail — a
serialization failure at commit time is exactly how `SERIALIZABLE` reports a
conflict — and a helper that ignores it reports success for a transaction that
did not happen.

**`Beginner` is an interface, not `*pgxpool.Pool`.** That single change is what
makes the next section possible.

### Testing it for real

`pgx.Tx` is an interface, so a fake that records calls needs no database:

```go
package main

import (
  "context"
  "errors"
  "testing"

  "github.com/jackc/pgx/v5"
  "github.com/jackc/pgx/v5/pgconn"
)

type fakeTx struct {
  pgx.Tx
  log       *[]string
  commitErr error
  closed    bool
}

func (f *fakeTx) Commit(ctx context.Context) error {
  if f.closed {
    *f.log = append(*f.log, "Commit->ErrTxClosed")
    return pgx.ErrTxClosed
  }
  if f.commitErr != nil {
    *f.log = append(*f.log, "Commit->err")
    return f.commitErr
  }
  f.closed = true
  *f.log = append(*f.log, "Commit")
  return nil
}

func (f *fakeTx) Rollback(ctx context.Context) error {
  if f.closed {
    *f.log = append(*f.log, "Rollback->ErrTxClosed")
    return pgx.ErrTxClosed
  }
  f.closed = true
  *f.log = append(*f.log, "Rollback")
  return nil
}

func (f *fakeTx) Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error) {
  *f.log = append(*f.log, "Exec")
  return pgconn.NewCommandTag("UPDATE 1"), nil
}
```

```go
type fakeBeginner struct {
  log      []string
  beginErr error
  tx       *fakeTx
}

func (b *fakeBeginner) BeginTx(ctx context.Context, opts pgx.TxOptions) (pgx.Tx, error) {
  if b.beginErr != nil {
    b.log = append(b.log, "BeginTx->err")
    return nil, b.beginErr
  }
  b.log = append(b.log, "BeginTx(iso="+string(opts.IsoLevel)+",mode="+string(opts.AccessMode)+")")
  b.tx.log = &b.log
  return b.tx, nil
}

var _ Beginner = (*fakeBeginner)(nil)

func TestWithTxCommitsOnSuccess(t *testing.T) {
  b := &fakeBeginner{tx: &fakeTx{}}
  err := withTx(context.Background(), b, pgx.TxOptions{}, func(tx pgx.Tx) error {
    _, execErr := tx.Exec(context.Background(), "UPDATE users SET status = 'active'")
    return execErr
  })
  if err != nil {
    t.Fatalf("want nil, got %v", err)
  }
  assertLog(t, b.log, []string{"BeginTx(iso=,mode=)", "Exec", "Commit"})
}

func TestWithTxRollsBackOnError(t *testing.T) {
  boom := errors.New("business rule violated")
  b := &fakeBeginner{tx: &fakeTx{}}
  err := withTx(context.Background(), b, pgx.TxOptions{IsoLevel: pgx.Serializable}, func(tx pgx.Tx) error {
    tx.Exec(context.Background(), "UPDATE users SET status = 'active'")
    return boom
  })
  if !errors.Is(err, boom) {
    t.Fatalf("want the callback error back, got %v", err)
  }
  assertLog(t, b.log, []string{"BeginTx(iso=serializable,mode=)", "Exec", "Rollback"})
}

func TestWithTxSurfacesCommitError(t *testing.T) {
  cErr := errors.New("could not serialize access due to concurrent update")
  b := &fakeBeginner{tx: &fakeTx{commitErr: cErr}}
  err := withTx(context.Background(), b, pgx.TxOptions{}, func(tx pgx.Tx) error { return nil })
  if !errors.Is(err, cErr) {
    t.Fatalf("commit error must surface, got %v", err)
  }
  t.Logf("err = %v", err)
  assertLog(t, b.log, []string{"BeginTx(iso=,mode=)", "Commit->err"})
}

func assertLog(t *testing.T, got, want []string) {
  t.Helper()
  if len(got) != len(want) {
    t.Fatalf("call log = %v, want %v", got, want)
  }
  for i := range got {
    if got[i] != want[i] {
      t.Fatalf("call log = %v, want %v", got, want)
    }
  }
  t.Logf("call log: %v", got)
}
```

Four tests are omitted above for length but appear in the output below: the
deferred rollback after a commit returning `ErrTxClosed`, a `BeginTx` failure
not running the callback, the panic path (which `recover`s in a `defer`, checks
the call log, and lets the panic keep going), and the read-only/`RepeatableRead`
options reaching `BeginTx`.

The embedded `pgx.Tx` in `fakeTx` is the trick that keeps this short: it
satisfies the whole interface with nil methods, and you override only the four
you care about. Any method the test accidentally calls panics with a nil
pointer dereference, which is exactly the signal you want.

Output:

```
=== RUN   TestWithTxCommitsOnSuccess
    withtx_test.go:76: call log: [BeginTx(iso=,mode=) Exec Commit]
--- PASS: TestWithTxCommitsOnSuccess (0.00s)
=== RUN   TestWithTxRollsBackOnError
    withtx_test.go:89: call log: [BeginTx(iso=serializable,mode=) Exec Rollback]
--- PASS: TestWithTxRollsBackOnError (0.00s)
=== RUN   TestWithTxDeferredRollbackAfterCommitIsHarmless
    withtx_test.go:106: log after commit then an extra Rollback: [BeginTx(iso=,mode=) Commit Rollback->ErrTxClosed]
--- PASS: TestWithTxDeferredRollbackAfterCommitIsHarmless (0.00s)
=== RUN   TestWithTxSurfacesCommitError
    withtx_test.go:116: err = commit: could not serialize access due to concurrent update
    withtx_test.go:117: call log: [BeginTx(iso=,mode=) Commit->err]
--- PASS: TestWithTxSurfacesCommitError (0.00s)
=== RUN   TestWithTxBeginError
    withtx_test.go:134: err = begin: connection refused
--- PASS: TestWithTxBeginError (0.00s)
=== RUN   TestWithTxRollsBackOnPanic
    withtx_test.go:144: recovered: nil map write in the callback
    withtx_test.go:145: call log: [BeginTx(iso=,mode=) Exec Rollback]
--- PASS: TestWithTxRollsBackOnPanic (0.00s)
=== RUN   TestWithTxReadOnly
    withtx_test.go:159: call log: [BeginTx(iso=repeatable read,mode=read only) Commit]
--- PASS: TestWithTxReadOnly (0.00s)
PASS
```

Every branch of the helper is covered with no database, and the call logs make
the behaviour legible: `Begin → Exec → Commit` on success, `Begin → Exec →
Rollback` on error **and** on panic, and the third test shows the deferred
`Rollback` after a commit returning `ErrTxClosed` and changing nothing.

One real finding from writing those tests: the commit-error case logs
`[BeginTx, Commit->err]` with **no Rollback**. The helper does not roll back
after a failed commit, and it should not — Postgres has already aborted the
transaction by the time `COMMIT` returns an error, so there is nothing left to
roll back. `pgx.BeginFunc` in the pgx library does the same thing.

pgx ships `pgx.BeginFunc(ctx, db, fn)` and `pgx.BeginTxFunc(ctx, db, opts, fn)`
for the common case. Write your own when you want the panic handling, the
`errors.Join` on a rollback failure, or your own logging — usually all three.

## The DBTX interface

A repository method should not care whether it is running on the pool or inside
a transaction. The standard Go solution is to depend on the **intersection** of
the two:

```go
type DBTX interface {
  Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
  Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
  QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
  SendBatch(ctx context.Context, b *pgx.Batch) pgx.BatchResults
}

var (
  _ DBTX = (*pgxpool.Pool)(nil)
  _ DBTX = (pgx.Tx)(nil)
)
```

Those two assertions are the whole point, and they compile: `*pgxpool.Pool` and
`pgx.Tx` both satisfy `DBTX` without either knowing the interface exists —
[5.1](../05-interfaces/01-implicit-satisfaction.md)'s implicit satisfaction
doing real work. The repository takes a `DBTX`:

```go
type UserRepository struct {
  db DBTX
}

func NewUserRepository(db DBTX) *UserRepository {
  return &UserRepository{db: db}
}
```

and both call sites work unchanged — `NewUserRepository(pool)` outside a
transaction, and `NewUserRepository(tx)` inside one, with every method
identical:

```go
repo := NewUserRepository(pool)
u, err := repo.GetUser(ctx, 42)

err = withTx(ctx, pool, pgx.TxOptions{}, func(tx pgx.Tx) error {
  txRepo := NewUserRepository(tx)
  u, err := txRepo.GetUser(ctx, 42)
  if err != nil {
    return err
  }
  return txRepo.UpdateUserStatus(ctx, u.ID, "active")
})
```

Keep `DBTX` **small**. It deliberately excludes `Begin`, `Commit` and
`Rollback`, so a repository method physically cannot start or end a transaction
— the compiler enforces the layering that 17.7 argues for.

### Never use the pool inside a transaction

```go
// WRONG: this query runs on a DIFFERENT connection and cannot see the tx.
err = withTx(ctx, pool, pgx.TxOptions{}, func(tx pgx.Tx) error {
  if _, err := tx.Exec(ctx, `INSERT INTO users (email) VALUES ($1)`, email); err != nil {
    return err
  }
  var n int
  return pool.QueryRow(ctx, `SELECT count(*) FROM users`).Scan(&n)
})
```

A transaction lives on **one connection**. `pool.QueryRow` acquires a different
one, which is outside the transaction, so it cannot see the uncommitted insert
and will block if the insert holds a conflicting lock. With a pool of size 1,
that is a self-deadlock: the transaction holds the only connection and waits
for itself.

This is exactly the bug `DBTX` prevents — a repository that only ever holds a
`DBTX` has no pool to reach for.

## Isolation levels

| Level | Prevents | Still possible | Cost |
|---|---|---|---|
| `ReadUncommitted` | — | everything | in Postgres this **is** `ReadCommitted` |
| `ReadCommitted` (**default**) | dirty reads | non-repeatable reads, phantoms, lost updates | none |
| `RepeatableRead` | dirty + non-repeatable reads, phantoms | write skew | aborts with `40001` on conflict |
| `Serializable` | everything | — | aborts with `40001`; needs a retry loop |

```go
opts := pgx.TxOptions{
  IsoLevel:       pgx.Serializable,
  AccessMode:     pgx.ReadOnly,
  DeferrableMode: pgx.Deferrable,
}
tx, err := pool.BeginTx(ctx, opts)
```

Postgres specifics that differ from the SQL standard and from MySQL:

- **`ReadUncommitted` does not exist.** Postgres silently treats it as
  `ReadCommitted`; there are no dirty reads to be had.
- **`RepeatableRead` in Postgres is snapshot isolation,** which also prevents
  phantom reads — stronger than the standard requires. What it does not prevent
  is **write skew**: two transactions each read a consistent snapshot, each
  decides a constraint is satisfied, and together they violate it.
- **`Serializable` uses SSI**, which detects conflicts rather than locking. It
  does not block; it aborts one transaction at commit time with SQLSTATE
  `40001`. **That means every `Serializable` transaction needs a retry loop**, or
  you have just moved your race condition into an occasional 500.
- **The level is set per transaction**, and cannot be changed after the first
  statement.

The practical advice: stay on `ReadCommitted`, and reach for `Serializable`
specifically for the invariants a single statement cannot express — "at most
three seats per org", "no overlapping bookings". For everything else, a
`SELECT ... FOR UPDATE` or a database constraint is cheaper and does not need
retries.

Expected output (requires a running Postgres; not executed in this environment):

```
tx A: read seats = 2
tx B: read seats = 2
tx A: commit OK
tx B: commit -> ERROR: could not serialize access due to read/write dependencies among transactions (SQLSTATE 40001)
```

## pgconn.PgError: the structured Postgres error

Every error Postgres reports arrives as a `*pgconn.PgError` with the full
protocol error fields. This runs offline:

```go
package main

import (
  "errors"
  "fmt"

  "github.com/jackc/pgx/v5"
  "github.com/jackc/pgx/v5/pgconn"
)

func main() {
  pe := &pgconn.PgError{
    Severity:       "ERROR",
    Code:           "23505",
    Message:        `duplicate key value violates unique constraint "users_email_key"`,
    Detail:         `Key (email)=(ada@example.com) already exists.`,
    SchemaName:     "public",
    TableName:      "users",
    ConstraintName: "users_email_key",
    File:           "nbtinsert.c",
    Line:           664,
    Routine:        "_bt_check_unique",
  }

  wrapped := fmt.Errorf("creating user: %w", pe)

  fmt.Printf("Error():  %v\n\n", pe)
  fmt.Printf("wrapped:  %v\n\n", wrapped)

  var got *pgconn.PgError
  fmt.Printf("errors.As through the wrap = %v\n", errors.As(wrapped, &got))
  fmt.Printf("  Code=%q\n  Message=%q\n  Detail=%q\n  Table=%q\n  Constraint=%q\n  Severity=%q\n",
    got.Code, got.Message, got.Detail, got.TableName, got.ConstraintName, got.Severity)
  fmt.Printf("  SQLState()=%q\n", got.SQLState())

  fmt.Println("\n--- pgx sentinel errors ---")
  fmt.Printf("pgx.ErrNoRows           = %v\n", pgx.ErrNoRows)
  fmt.Printf("pgx.ErrTxClosed         = %v\n", pgx.ErrTxClosed)
  fmt.Printf("pgx.ErrTxCommitRollback = %v\n", pgx.ErrTxCommitRollback)
  fmt.Printf("pgx.ErrTooManyRows      = %v\n", pgx.ErrTooManyRows)
}
```

Output:

```
Error():  ERROR: duplicate key value violates unique constraint "users_email_key" (SQLSTATE 23505)

wrapped:  creating user: ERROR: duplicate key value violates unique constraint "users_email_key" (SQLSTATE 23505)

errors.As through the wrap = true
  Code="23505"
  Message="duplicate key value violates unique constraint \"users_email_key\""
  Detail="Key (email)=(ada@example.com) already exists."
  Table="users"
  Constraint="users_email_key"
  Severity="ERROR"
  SQLState()="23505"

--- pgx sentinel errors ---
pgx.ErrNoRows           = no rows in result set
pgx.ErrTxClosed         = tx is closed
pgx.ErrTxCommitRollback = commit unexpectedly resulted in rollback
pgx.ErrTooManyRows      = too many rows in result set
```

`errors.As` reached the `*pgconn.PgError` through a `%w` wrap, which is the
whole mechanism. You branch on `Code` — a five-character SQLSTATE defined by
the standard and by Postgres — and never on `Message`, because messages are
localised and change between versions.

`ConstraintName` is the field that makes good error messages possible: it tells
you *which* unique index was violated, so a user with both a unique email and a
unique external ID gets the right message.

## Mapping SQLSTATE to domain errors

| SQLSTATE | Name | Means | Domain error | HTTP (16.9) |
|---|---|---|---|---|
| `23505` | `unique_violation` | duplicate key | `ErrDuplicateUser` | **409** |
| `23503` | `foreign_key_violation` | referenced row missing | `ErrOrgNotFound` | **422** or 404 |
| `23502` | `not_null_violation` | required column was NULL | `ErrInvalidUser` | **422** |
| `23514` | `check_violation` | a `CHECK` failed | `ErrInvalidUser` | **422** |
| `23P01` | `exclusion_violation` | an `EXCLUDE` constraint failed | `ErrConflict` | **409** |
| `40001` | `serialization_failure` | SSI conflict | retry, then `ErrConflict` | **409** or 503 |
| `40P01` | `deadlock_detected` | deadlock | retry, then `ErrConflict` | **409** |
| `22P02` | `invalid_text_representation` | bad enum/uuid/number literal | `ErrInvalidUser` | **422** |
| `53300` | `too_many_connections` | pool/server exhausted (17.5) | infrastructure | **503** |
| `57014` | `query_canceled` | `statement_timeout` or a cancel | `context.Canceled` | 499/504 |

The mapping function, with real tests:

```go
package main

import (
  "errors"
  "fmt"

  "github.com/jackc/pgx/v5"
  "github.com/jackc/pgx/v5/pgconn"
)

var (
  ErrUserNotFound  = errors.New("user not found")
  ErrDuplicateUser = errors.New("user already exists")
  ErrOrgNotFound   = errors.New("org not found")
  ErrInvalidUser   = errors.New("user violates a database constraint")
  ErrConflict      = errors.New("concurrent update, retry")
)

const (
  codeUniqueViolation     = "23505"
  codeForeignKeyViolation = "23503"
  codeNotNullViolation    = "23502"
  codeCheckViolation      = "23514"
  codeSerializationFail   = "40001"
  codeDeadlockDetected    = "40P01"
)

func mapUserError(op string, err error) error {
  if err == nil {
    return nil
  }
  if errors.Is(err, pgx.ErrNoRows) {
    return fmt.Errorf("%s: %w", op, ErrUserNotFound)
  }
  var pg *pgconn.PgError
  if !errors.As(err, &pg) {
    return fmt.Errorf("%s: %w", op, err)
  }
  switch pg.Code {
  case codeUniqueViolation:
    if pg.ConstraintName == "users_email_key" {
      return fmt.Errorf("%s: %w", op, ErrDuplicateUser)
    }
    return fmt.Errorf("%s: constraint %s: %w", op, pg.ConstraintName, ErrDuplicateUser)
  case codeForeignKeyViolation:
    if pg.ConstraintName == "users_org_id_fkey" {
      return fmt.Errorf("%s: %w", op, ErrOrgNotFound)
    }
    return fmt.Errorf("%s: constraint %s: %w", op, pg.ConstraintName, ErrInvalidUser)
  case codeNotNullViolation:
    return fmt.Errorf("%s: column %q must not be null: %w", op, pg.ColumnName, ErrInvalidUser)
  case codeCheckViolation:
    return fmt.Errorf("%s: check %q failed: %w", op, pg.ConstraintName, ErrInvalidUser)
  case codeSerializationFail, codeDeadlockDetected:
    return fmt.Errorf("%s: sqlstate %s: %w", op, pg.Code, ErrConflict)
  }
  return fmt.Errorf("%s: sqlstate %s: %w", op, pg.Code, err)
}
```

The default case is the important one: an unrecognised SQLSTATE keeps the
original error reachable with `%w`, so it classifies as a 500 and the real
cause stays in the log. Silently turning an unknown database error into a 404
is how an outage becomes invisible.

```go
func TestMapUserError(t *testing.T) {
  tests := []struct {
    name string
    in   error
    want error
  }{
    {"nil stays nil", nil, nil},
    {"no rows", pgx.ErrNoRows, ErrUserNotFound},
    {
      "duplicate email",
      &pgconn.PgError{Code: "23505", ConstraintName: "users_email_key",
        Message: `duplicate key value violates unique constraint "users_email_key"`},
      ErrDuplicateUser,
    },
    {
      "missing org",
      &pgconn.PgError{Code: "23503", ConstraintName: "users_org_id_fkey"},
      ErrOrgNotFound,
    },
    {
      "not null",
      &pgconn.PgError{Code: "23502", ColumnName: "email", TableName: "users"},
      ErrInvalidUser,
    },
    {"serialization failure", &pgconn.PgError{Code: "40001"}, ErrConflict},
  }

  for _, tc := range tests {
    t.Run(tc.name, func(t *testing.T) {
      got := mapUserError("CreateUser", tc.in)
      if tc.want == nil {
        if got != nil {
          t.Fatalf("want nil, got %v", got)
        }
        return
      }
      if !errors.Is(got, tc.want) {
        t.Fatalf("errors.Is(%v, %v) = false", got, tc.want)
      }
      t.Logf("%v", got)
    })
  }
}

```

Four table cases and two test functions are omitted above for length; all
appear in the output below. `TestMapUserErrorPassesThroughUnknown` asserts an
unrecognised error stays reachable with `errors.Is` and matches **none** of the
five sentinels; `TestMapUserErrorUnknownSQLState` asserts a `53300` keeps its
`*pgconn.PgError` reachable and does not classify as a not-found.

Output:

```
=== RUN   TestMapUserError
=== RUN   TestMapUserError/nil_stays_nil
=== RUN   TestMapUserError/no_rows
    errmap_test.go:63: CreateUser: user not found
=== RUN   TestMapUserError/wrapped_no_rows
    errmap_test.go:63: CreateUser: user not found
=== RUN   TestMapUserError/duplicate_email
    errmap_test.go:63: CreateUser: user already exists
=== RUN   TestMapUserError/some_other_unique_index
    errmap_test.go:63: CreateUser: constraint users_external_id_key: user already exists
=== RUN   TestMapUserError/missing_org
    errmap_test.go:63: CreateUser: org not found
=== RUN   TestMapUserError/not_null
    errmap_test.go:63: CreateUser: column "email" must not be null: user violates a database constraint
=== RUN   TestMapUserError/check_violation
    errmap_test.go:63: CreateUser: check "users_email_lowercase" failed: user violates a database constraint
=== RUN   TestMapUserError/serialization_failure
    errmap_test.go:63: CreateUser: sqlstate 40001: concurrent update, retry
=== RUN   TestMapUserError/deadlock
    errmap_test.go:63: CreateUser: sqlstate 40P01: concurrent update, retry
--- PASS: TestMapUserError (0.00s)
    --- PASS: TestMapUserError/nil_stays_nil (0.00s)
    --- PASS: TestMapUserError/no_rows (0.00s)
    --- PASS: TestMapUserError/wrapped_no_rows (0.00s)
    --- PASS: TestMapUserError/duplicate_email (0.00s)
    --- PASS: TestMapUserError/some_other_unique_index (0.00s)
    --- PASS: TestMapUserError/missing_org (0.00s)
    --- PASS: TestMapUserError/not_null (0.00s)
    --- PASS: TestMapUserError/check_violation (0.00s)
    --- PASS: TestMapUserError/serialization_failure (0.00s)
    --- PASS: TestMapUserError/deadlock (0.00s)
=== RUN   TestMapUserErrorPassesThroughUnknown
    errmap_test.go:79: GetUser: connection reset by peer
--- PASS: TestMapUserErrorPassesThroughUnknown (0.00s)
=== RUN   TestMapUserErrorUnknownSQLState
    errmap_test.go:92: GetUser: sqlstate 53300: : too many connections for role "app" (SQLSTATE 53300)
--- PASS: TestMapUserErrorUnknownSQLState (0.00s)
PASS
```

### Connecting it to the HTTP layer

16.9's `classify` function already handles sentinels. Add three cases and a
duplicate email becomes a 409 with no code in between knowing about SQLSTATE:

```go
case errors.Is(err, ErrDuplicateUser):
  return APIError{http.StatusConflict, "conflict",
    "That email address is already registered.", nil, 0}
case errors.Is(err, ErrOrgNotFound):
  return APIError{http.StatusUnprocessableEntity, "unknown_org",
    "That organization does not exist.", nil, 0}
case errors.Is(err, ErrConflict):
  return APIError{http.StatusConflict, "conflict",
    "The resource changed, please retry.", nil, 0}
```

That is the layering working as designed: the repository knows SQLSTATE, the
domain knows sentinels, the transport knows status codes, and nobody knows two
of the three. Note what the client gets for a duplicate — "That email address
is already registered." — while the log keeps
`constraint users_email_key`, which reveals your schema and stays internal.

## Retrying on 40001

A `Serializable` transaction that is not retried is a transaction that
occasionally fails for no reason the user can understand.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "math/rand/v2"
  "time"

  "github.com/jackc/pgx/v5"
  "github.com/jackc/pgx/v5/pgconn"
  "github.com/jackc/pgx/v5/pgxpool"
)

func isRetryable(err error) bool {
  var pg *pgconn.PgError
  if !errors.As(err, &pg) {
    return false
  }
  return pg.Code == "40001" || pg.Code == "40P01"
}

func withRetry(ctx context.Context, attempts int, fn func(context.Context) error) error {
  var err error
  for i := range attempts {
    err = fn(ctx)
    if err == nil || !isRetryable(err) {
      return err
    }
    backoff := time.Duration(1<<i)*10*time.Millisecond +
      time.Duration(rand.N(10*time.Millisecond))
    select {
    case <-ctx.Done():
      return errors.Join(err, ctx.Err())
    case <-time.After(backoff):
    }
  }
  return fmt.Errorf("after %d attempts: %w", attempts, err)
}

func transferSeats(ctx context.Context, pool *pgxpool.Pool, from, to int64, n int32) error {
  return withRetry(ctx, 4, func(ctx context.Context) error {
    return withTx(ctx, pool, pgx.TxOptions{IsoLevel: pgx.Serializable}, func(tx pgx.Tx) error {
      var available int32
      err := tx.QueryRow(ctx,
        `SELECT seats FROM orgs WHERE id = $1`, from).Scan(&available)
      if err != nil {
        return err
      }
      if available < n {
        return fmt.Errorf("org %d has %d seats, needs %d", from, available, n)
      }
      if _, err := tx.Exec(ctx,
        `UPDATE orgs SET seats = seats - $2 WHERE id = $1`, from, n); err != nil {
        return err
      }
      _, err = tx.Exec(ctx,
        `UPDATE orgs SET seats = seats + $2 WHERE id = $1`, to, n)
      return err
    })
  })
}
```

Four properties make this correct rather than merely present.

**The retry wraps the whole transaction**, not a statement inside it. A
`40001` aborts the entire transaction; retrying one statement inside an aborted
transaction gives you `25P02 in_failed_sql_transaction`.

**The callback must be idempotent and must re-read.** It starts a fresh
transaction with a fresh snapshot, so every value it read is re-read. A closure
that captured a row from the first attempt is a bug.

**Backoff with jitter**, because the whole point is that two transactions
conflicted — retrying both instantly reproduces the conflict. `rand.N` from
`math/rand/v2` is fine here; this is not a security decision.

**It respects `ctx`.** The `select` on `ctx.Done()` means a client who gave up
does not keep your retries running (16.12), and the `errors.Join` keeps both
the database error and the cancellation reason.

The predicate is testable offline:

```
  isRetryable(serialization failure ) = true
  isRetryable(deadlock              ) = true
  isRetryable(unique violation      ) = false
  isRetryable(plain error           ) = false
  isRetryable(wrapped 40001         ) = true
```

Note the last line: the `40001` was wrapped in `fmt.Errorf("commit: %w", ...)`
by `withTx`, and `errors.As` still found it. That matters because a
serialization failure usually surfaces **at commit time**, not at the statement
that caused it — so a retry predicate that only inspects statement errors will
never fire.

**Never retry a unique violation.** It is deterministic: the row exists, and it
will exist on the next attempt too. Retrying it just delays the 409 by your
backoff budget.

## Savepoints: nested Begin

pgx implements `tx.Begin(ctx)` on an existing transaction as a **savepoint**.
Commit releases the savepoint, rollback rolls back to it, and the outer
transaction continues:

```go
err := withTx(ctx, pool, pgx.TxOptions{}, func(tx pgx.Tx) error {
  for _, email := range emails {
    sp, err := tx.Begin(ctx)
    if err != nil {
      return err
    }
    _, err = sp.Exec(ctx,
      `INSERT INTO users (org_id, email, name, status)
       VALUES (1, $1, $1, 'pending')`, email)
    if err != nil {
      if rbErr := sp.Rollback(ctx); rbErr != nil {
        return rbErr
      }
      var pg *pgconn.PgError
      if errors.As(err, &pg) && pg.Code == "23505" {
        continue
      }
      return err
    }
    if err := sp.Commit(ctx); err != nil {
      return err
    }
    imported++
  }
  return nil
})
```

The savepoint is what makes "skip the duplicates, import the rest, atomically"
possible. Without it, the **first** failed INSERT aborts the whole transaction
and every subsequent statement fails with `25P02` — which is the behaviour that
surprises people coming from MySQL, where a failed statement does not poison
the transaction.

Savepoints are not free. Each one is a round trip and server-side state, so a
per-row savepoint over a million rows is slow. For a bulk import, prefer
`INSERT ... ON CONFLICT DO NOTHING`, which does the same job in one statement:

```sql
INSERT INTO users (org_id, email, name, status)
VALUES ($1, $2, $3, 'pending')
ON CONFLICT (email) DO NOTHING
RETURNING id
```

Expected output (requires a running Postgres; not executed in this environment):

```
imported 2 of 3 (1 duplicate skipped), transaction committed
```

## Read-only transactions

```go
err := withTx(ctx, pool,
  pgx.TxOptions{IsoLevel: pgx.RepeatableRead, AccessMode: pgx.ReadOnly},
  func(tx pgx.Tx) error {
    if err := tx.QueryRow(ctx, `SELECT count(*) FROM users`).Scan(&users); err != nil {
      return err
    }
    return tx.QueryRow(ctx, `SELECT count(*) FROM orgs`).Scan(&orgs)
  })
```

Two reasons to bother. `ReadOnly` makes an accidental write a database error
rather than a data-corruption incident — useful for a reporting endpoint.
`RepeatableRead` means both counts come from the **same snapshot**, so a report
cannot show 100 users in one number and 101 in another. Without the
transaction, two `pool.QueryRow` calls are two snapshots and the numbers can
disagree.

Add `DeferrableMode: pgx.Deferrable` with `Serializable` + `ReadOnly` and
Postgres waits for a safe snapshot and then guarantees the transaction will
never be aborted by `40001` — the right setting for a long analytical read.

## Long transactions hold locks

A transaction left open holds its locks, keeps its snapshot alive, and blocks
`VACUUM` from reclaiming rows newer than it, so an idle-in-transaction
connection bloats tables and queues other writers behind it. Never hold one
open across an HTTP call or a message publish; do the external work before
`Begin` or after `Commit`.

## Quick Reference

| Call | Note |
|---|---|
| `pool.Begin(ctx)` | `(pgx.Tx, error)` with default options |
| `pool.BeginTx(ctx, opts)` | with `pgx.TxOptions` |
| `tx.Commit(ctx)` | `ErrTxClosed` if already closed |
| `tx.Rollback(ctx)` | safe to `defer`; `ErrTxClosed` after commit |
| `tx.Begin(ctx)` | a **savepoint**, not a real nested transaction |
| `tx.Conn()` | the underlying `*pgx.Conn` |
| `pgx.BeginFunc(ctx, db, fn)` | the library's own commit/rollback wrapper |
| `pgx.BeginTxFunc(ctx, db, opts, fn)` | same, with options |

| `pgx.TxOptions` field | Values |
|---|---|
| `IsoLevel` | `pgx.ReadCommitted`, `pgx.RepeatableRead`, `pgx.Serializable`, `pgx.ReadUncommitted` |
| `AccessMode` | `pgx.ReadWrite`, `pgx.ReadOnly` |
| `DeferrableMode` | `pgx.Deferrable`, `pgx.NotDeferrable` |
| `BeginQuery` | a raw statement to run instead of `BEGIN` |

| Sentinel | Means |
|---|---|
| `pgx.ErrNoRows` | no rows; wraps `sql.ErrNoRows` |
| `pgx.ErrTooManyRows` | `CollectExactlyOneRow` got more than one |
| `pgx.ErrTxClosed` | `Commit`/`Rollback` on a closed tx — the harmless case |
| `pgx.ErrTxCommitRollback` | `COMMIT` resulted in a rollback |

## Common Mistakes

**Forgetting the deferred rollback.** An early return leaves the transaction
open and its connection checked out until the pool reaps it; `defer` it or use
a `withTx` helper.

**Ignoring the error from `Commit`.** A serialization failure surfaces at commit
time, so ignoring it reports success for a transaction that never happened.

**Using the pool inside a transaction.** The query runs on a different
connection, cannot see uncommitted rows, and self-deadlocks with a small pool;
pass a `DBTX`.

**Using `Serializable` without a retry loop.** `40001` becomes an occasional
unexplained 500; wrap the whole transaction in a retry with backoff.

**Retrying a statement instead of the transaction.** The transaction is already
aborted, so every statement fails with `25P02`; restart from `Begin`.

**Retrying a unique violation.** It is deterministic and will fail identically;
map `23505` to a 409 immediately.

**Branching on `err.Error()` text.** Postgres messages are localised and change
between versions; branch on `pg.Code` via `errors.As`.
**Expecting a failed statement not to poison the transaction.** Unlike MySQL,
Postgres aborts the whole transaction; use a savepoint or `ON CONFLICT`.

**Holding a transaction open across a network call.** It pins locks and blocks
`VACUUM`; do the external work outside the transaction.

## Exercises

1. Extend `withTx` to accept an `after func(error)` hook so the caller can log
   commit-versus-rollback with a duration, then add tests to the fake-`Tx`
   suite proving the hook fires exactly once on each of the four paths.
2. Reproduce a `40001` for real against the compose stack: two goroutines,
   `Serializable`, each reading a seat count and inserting. Capture the real
   error, then add the retry loop and show it succeeding, with a counter of how
   many retries fired.
3. Reproduce a `40P01` deadlock: two transactions updating the same two rows in
   opposite orders. Capture the real error including the `Detail` field, then
   fix it by ordering the updates consistently.
4. Write a `mapError` for a second table with its own constraints and a test
   asserting all twelve SQLSTATEs from this lesson's table map to the status
   code the 16.9 table specifies. Assert on status codes, not strings.
5. Implement an "at most N seats per org" invariant three ways — `Serializable`
   plus retry, `SELECT ... FOR UPDATE`, and a deferred `CHECK` on a summary
   table — and benchmark all three under 20 concurrent writers. Report which
   you would ship.

---

[← 17.3 Queries, Scanning, and NULL Handling](03-queries-scanning.md) · [Course Index](../README.md) · [17.5 Connection Pooling and Tuning →](05-pooling.md)
