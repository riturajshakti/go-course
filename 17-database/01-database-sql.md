[← 16.12 Context Propagation Through Handlers](../16-http/12-context-propagation.md) · [Course Index](../README.md) · [17.2 pgx: Connecting to Postgres →](02-pgx.md)

# 17.1 — database/sql and the Driver Model

## What You'll Learn

- `sql.Open` does **not** connect — it validates a driver name and returns a pool; `Ping` is what connects, proven with a bogus DSN
- `*sql.DB` is a connection **pool**, safe for concurrent use, and must be a long-lived singleton — never per request
- The `*sql.Rows` contract: you **must** check `rows.Err()` after the loop and `Close` the rows, with the silent data-loss bug shown for real
- Why this module leaves `database/sql` behind for pgx, and the narrow case where `database/sql` is still the right answer

## Coming From Another Language

| Other | Go |
|---|---|
| `new Pool()` from `pg`, `mysql2` | `sql.Open` returns the pool; there is no separate client type |
| `await pool.connect()` to get a connection | you rarely touch a connection; the pool hands one out per call |
| Python DB-API `connect()` opening a socket | `sql.Open` opens nothing — `PingContext` does |
| sqlx/SQLAlchemy mapping rows to objects | `Scan` into addresses, field by field; no reflection in the stdlib |
| `?` placeholders everywhere | placeholder syntax is the **driver's**: `$1` for Postgres, `?` for MySQL |
| exceptions on a failed query | `err` on every call, and `rows.Err()` *after* the loop |

`database/sql` is not a database client. It is a generic pool and row-scanning
layer over a driver interface, and it ships with **no drivers at all**. That
split is the whole design, and everything surprising about the package follows
from it.

## The driver registry

A driver is a type implementing `driver.Driver`, registered under a name at
init time. `sql.Drivers()` lists what has been registered.

```go
package main

import (
  "context"
  "database/sql"
  "database/sql/driver"
  "errors"
  "fmt"
  "time"
)

type nopDriver struct{}

func (nopDriver) Open(name string) (driver.Conn, error) {
  return nil, errors.New("nop: not a real driver")
}

func main() {
  fmt.Println("drivers before:", sql.Drivers())
  sql.Register("nop", nopDriver{})
  fmt.Println("drivers after: ", sql.Drivers())

  db, err := sql.Open("nop", "this-dsn-is-complete-nonsense")
  fmt.Printf("sql.Open err=%v  db!=nil=%v\n", err, db != nil)

  ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
  defer cancel()
  fmt.Printf("db.PingContext err=%v\n", db.PingContext(ctx))

  _, err = sql.Open("nonexistent-driver", "")
  fmt.Printf("unknown driver err=%v\n", err)

  st := db.Stats()
  fmt.Printf("stats: open=%d inuse=%d idle=%d maxopen=%d waitcount=%d\n",
    st.OpenConnections, st.InUse, st.Idle, st.MaxOpenConnections, st.WaitCount)
  fmt.Printf("db.Close err=%v\n", db.Close())

  defer func() {
    fmt.Printf("recovered: %v\n", recover())
  }()
  sql.Register("nop", nopDriver{})
}
```

Output:

```
drivers before: []
drivers after:  [nop]
sql.Open err=<nil>  db!=nil=true
db.PingContext err=nop: not a real driver
unknown driver err=sql: unknown driver "nonexistent-driver" (forgotten import?)
stats: open=0 inuse=0 idle=0 maxopen=0 waitcount=0
db.Close err=<nil>
recovered: sql: Register called twice for driver nop
```

Four facts in that output, all of them load-bearing.

**`sql.Open` succeeded with a DSN that is pure garbage.** It never asked the
driver to parse it. All it did was look up `"nop"` in the registry and build a
pool around it.

**`PingContext` is what actually called `Open` on the driver** and surfaced its
error. This is the single most misunderstood thing about the package, and 17.2
shows the same split with a real Postgres driver.

**An unknown driver name fails at `sql.Open`** with a message that names the
real cause: a forgotten blank import.

**Registering the same name twice panics.** `sql.Register` is designed to be
called exactly once, from a package's `init`, which is why registration is a
side effect of importing.

### The blank import

Drivers register themselves in `init`, so you import them for the side effect
only. The driver name you pass to `sql.Open` is a string the driver chose, not
the import path:

```go
import (
  "database/sql"

  _ "github.com/jackc/pgx/v5/stdlib"
)
```

The `_` is required: nothing in your code references the package, so a normal
import would be a compile error. Other common ones are `_ "github.com/lib/pq"`
(registers `"postgres"`), `_ "github.com/go-sql-driver/mysql"` (`"mysql"`), and
`_ "modernc.org/sqlite"` (`"sqlite"`).

pgx ships a `database/sql` adapter in its `stdlib` subpackage, and it registers
two names:

```go
package main

import (
  "database/sql"
  "errors"
  "fmt"

  _ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
  fmt.Println("registered drivers:", sql.Drivers())

  db, err := sql.Open("pgx", "postgres://nobody:nothing@127.0.0.1:1/nosuchdb")
  fmt.Printf("sql.Open: err=%v\n", err)

  err = db.Ping()
  fmt.Printf("db.Ping: non-nil=%v\n", err != nil)
  fmt.Printf("is ErrNoRows: %v\n", errors.Is(err, sql.ErrNoRows))
  fmt.Printf("db.Close: err=%v\n", db.Close())
}
```

Output:

```
registered drivers: [pgx pgx/v5]
sql.Open: err=<nil>
db.Ping: non-nil=true
is ErrNoRows: false
db.Close: err=<nil>
```

Again: `sql.Open` succeeded against a port with nothing on it. The failure
arrived at `Ping`. The real error text is in 17.2.

## *sql.DB is a pool, and a singleton

`*sql.DB` holds a set of connections, hands one out for each call, and returns
it afterwards. Three consequences that people get wrong:

**It is safe for concurrent use from many goroutines.** No mutex, no
per-request copy. That is its purpose.

**It must be created once, at startup, and shared.** A `sql.Open` per HTTP
request creates a new pool per request, so you get no reuse, and every pool's
connections stay open until GC — which is how a service exhausts
`max_connections` under load while looking idle.

**`Close` is a shutdown operation, not a per-query one.** You call it once,
when the process is shutting down ([18.5](../18-production/05-graceful-shutdown.md)).

```go
type App struct {
  db *sql.DB
}

func New(ctx context.Context, dsn string) (*App, error) {
  db, err := sql.Open("pgx", dsn)
  if err != nil {
    return nil, fmt.Errorf("opening db: %w", err)
  }
  db.SetMaxOpenConns(25)
  db.SetMaxIdleConns(25)
  db.SetConnMaxLifetime(30 * time.Minute)
  db.SetConnMaxIdleTime(5 * time.Minute)

  pingCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
  defer cancel()
  if err := db.PingContext(pingCtx); err != nil {
    db.Close()
    return nil, fmt.Errorf("connecting to db: %w", err)
  }
  return &App{db: db}, nil
}
```

Because `sql.Open` is lazy, that `PingContext` is the only thing standing
between you and a service that starts healthy and 500s on its first request.
Idiomatic: **always ping at startup**, with a timeout, and fail the boot.

## The API surface

Every method comes in a `Context` and a non-`Context` form. The non-`Context`
forms call `context.Background()` internally, which is why 16.12 insisted you
use the `Context` variants exclusively — one plain call breaks cancellation for
everything below it.

| Method | Returns | Use for |
|---|---|---|
| `QueryContext(ctx, q, args...)` | `(*sql.Rows, error)` | many rows |
| `QueryRowContext(ctx, q, args...)` | `*sql.Row` | exactly one row; error deferred to `Scan` |
| `ExecContext(ctx, q, args...)` | `(sql.Result, error)` | INSERT/UPDATE/DELETE/DDL |
| `PrepareContext(ctx, q)` | `(*sql.Stmt, error)` | a statement reused many times |
| `BeginTx(ctx, opts)` | `(*sql.Tx, error)` | a transaction (17.4) |
| `Conn(ctx)` | `(*sql.Conn, error)` | pinning one connection for session state |
| `PingContext(ctx)` | `error` | liveness; the only thing that connects eagerly |
| `Close()` | `error` | shutdown only |
| `Stats()` | `sql.DBStats` | pool gauges for metrics (17.5) |
| `SetMaxOpenConns(n)` | — | hard ceiling; `0` means unlimited |
| `SetMaxIdleConns(n)` | — | keep-warm count; default **2** |
| `SetConnMaxLifetime(d)` | — | recycle age; `0` means forever |
| `SetConnMaxIdleTime(d)` | — | idle reap age; `0` means never |

`Query` and `QueryRow` also exist without `Context`, as do `Exec`, `Prepare`
and `Begin`. Treat them as deprecated in service code.

The default `SetMaxIdleConns(2)` is a real trap: with `MaxOpenConns(25)` and
the default idle count, a burst opens 25 connections and then closes 23 of them
the moment it subsides, so the next burst reconnects 23 times. Set them equal.

## The *sql.Rows contract

This is the part that silently loses data. A driver can fail **mid-iteration**:
`Next` returns `false`, your loop exits, and it looks exactly like a normal
end of results. The error is parked on `rows.Err()`.

```go
package main

import (
  "context"
  "database/sql"
  "database/sql/driver"
  "errors"
  "fmt"
  "io"
)

type memDriver struct{}
type memConn struct{}
type memStmt struct{ n int }
type memRows struct {
  i, n int
  fail bool
}

func (memDriver) Open(dsn string) (driver.Conn, error) { return memConn{}, nil }

func (memConn) Prepare(q string) (driver.Stmt, error) {
  switch q {
  case "SELECT id FROM t":
    return memStmt{n: 3}, nil
  case "SELECT id FROM t WHERE id = 99":
    return memStmt{n: 0}, nil
  case "SELECT id FROM broken":
    return memStmt{n: -1}, nil
  }
  return nil, errors.New(`syntax error at or near "` + q + `"`)
}
func (memConn) Close() error              { return nil }
func (memConn) Begin() (driver.Tx, error) { return nil, errors.New("no transactions") }

func (s memStmt) Close() error  { return nil }
func (s memStmt) NumInput() int { return 0 }
func (s memStmt) Exec(args []driver.Value) (driver.Result, error) {
  return driver.RowsAffected(2), nil
}
func (s memStmt) Query(args []driver.Value) (driver.Rows, error) {
  if s.n < 0 {
    return &memRows{n: 2, fail: true}, nil
  }
  return &memRows{n: s.n}, nil
}

func (r *memRows) Columns() []string { return []string{"id"} }
func (r *memRows) Close() error      { return nil }
func (r *memRows) Next(dest []driver.Value) error {
  if r.fail && r.i == 1 {
    return errors.New("connection reset by peer")
  }
  if r.i >= r.n {
    return io.EOF
  }
  dest[0] = int64(10 * (r.i + 1))
  r.i++
  return nil
}

func main() {
  sql.Register("mem", memDriver{})
  db, _ := sql.Open("mem", "")
  defer db.Close()
  ctx := context.Background()

  fmt.Println("--- 1. the full Rows loop ---")
  rows, err := db.QueryContext(ctx, "SELECT id FROM t")
  if err != nil {
    panic(err)
  }
  defer rows.Close()
  cols, _ := rows.Columns()
  fmt.Printf("columns: %v\n", cols)
  var ids []int64
  for rows.Next() {
    var id int64
    if err := rows.Scan(&id); err != nil {
      panic(err)
    }
    ids = append(ids, id)
  }
  fmt.Printf("ids=%v  rows.Err()=%v\n", ids, rows.Err())
  fmt.Printf("rows.Close()=%v\n", rows.Close())

  fmt.Println("\n--- 2. the loop that ends in an error ---")
  bad, err := db.QueryContext(ctx, "SELECT id FROM broken")
  if err != nil {
    panic(err)
  }
  var got []int64
  for bad.Next() {
    var id int64
    bad.Scan(&id)
    got = append(got, id)
  }
  fmt.Printf("ids=%v  (looks complete)\n", got)
  fmt.Printf("bad.Err()=%v   <-- the loop LIED\n", bad.Err())
  bad.Close()

  fmt.Println("\n--- 3. QueryRow with no rows ---")
  var id int64
  err = db.QueryRowContext(ctx, "SELECT id FROM t WHERE id = 99").Scan(&id)
  fmt.Printf("err=%v\n", err)
  fmt.Printf("errors.Is(err, sql.ErrNoRows)=%v\n", errors.Is(err, sql.ErrNoRows))
  fmt.Printf("err == sql.ErrNoRows        =%v\n", err == sql.ErrNoRows)

  fmt.Println("\n--- 4. Exec and Result ---")
  res, err := db.ExecContext(ctx, "SELECT id FROM t")
  if err != nil {
    panic(err)
  }
  n, err := res.RowsAffected()
  fmt.Printf("RowsAffected=%d err=%v\n", n, err)
  last, err := res.LastInsertId()
  fmt.Printf("LastInsertId=%d err=%v\n", last, err)

  fmt.Println("\n--- 5. a query error ---")
  _, err = db.QueryContext(ctx, "SELCT oops")
  fmt.Printf("err=%v\n", err)

  fmt.Println("\n--- 6. scan type mismatch ---")
  r2, _ := db.QueryContext(ctx, "SELECT id FROM t")
  r2.Next()
  var s struct{}
  fmt.Printf("Scan into *struct{}: %v\n", r2.Scan(&s))
  var tm bool
  fmt.Printf("Scan int64 into *bool: %v\n", r2.Scan(&tm))
  r2.Close()

  fmt.Printf("\nfinal stats: open=%d inuse=%d idle=%d\n",
    db.Stats().OpenConnections, db.Stats().InUse, db.Stats().Idle)
}
```

That is a complete, registered `database/sql` driver in about 50 lines, which
is also a useful sense of how thin the driver interface is. Output:

```
--- 1. the full Rows loop ---
columns: [id]
ids=[10 20 30]  rows.Err()=<nil>
rows.Close()=<nil>

--- 2. the loop that ends in an error ---
ids=[10]  (looks complete)
bad.Err()=connection reset by peer   <-- the loop LIED

--- 3. QueryRow with no rows ---
err=sql: no rows in result set
errors.Is(err, sql.ErrNoRows)=true
err == sql.ErrNoRows        =true

--- 4. Exec and Result ---
RowsAffected=2 err=<nil>
LastInsertId=0 err=LastInsertId is not supported by this driver

--- 5. a query error ---
err=syntax error at or near "SELCT oops"

--- 6. scan type mismatch ---
Scan into *struct{}: sql: Scan error on column index 0, name "id": unsupported Scan, storing driver.Value type int64 into type *struct {}
Scan int64 into *bool: sql: Scan error on column index 0, name "id": sql/driver: couldn't convert 10 into type bool

final stats: open=1 inuse=0 idle=1
```

Case 2 is the bug. The loop collected one row out of two and exited cleanly.
Every variable looks fine. `got` is a plausible result set. If you do not check
`Err()`, you have just served a truncated list to a client and logged nothing.

The correct loop, every time:

```go
rows, err := db.QueryContext(ctx, q, args...)
if err != nil {
  return nil, err
}
defer rows.Close()

var out []Thing
for rows.Next() {
  var t Thing
  if err := rows.Scan(&t.ID, &t.Name); err != nil {
    return nil, err
  }
  out = append(out, t)
}
return out, rows.Err()
```

Four obligations: check the `Query` error, `defer rows.Close()`, check each
`Scan` error, and **return `rows.Err()`**. `return out, rows.Err()` is the
idiomatic last line of a list method, and it is the whole fix.

### Why Close matters

While `rows` is open, its connection is checked out of the pool. Forgetting
`Close` leaks that connection until GC. A handler that returns early on an
error path without closing will, under load, pin every connection in the pool
and then block forever on acquire — a hang that looks like the database is
slow when the database is idle.

`Close` is idempotent and safe after a completed loop (the final output shows
`rows.Close()=<nil>`), so `defer rows.Close()` immediately after the error
check costs nothing and is always right. Draining the loop to completion
closes the rows automatically, but your early returns do not.

## sql.ErrNoRows

`QueryRow` defers its error to `Scan`, and "no rows" is an error, not an empty
result:

```go
var u User
err := db.QueryRowContext(ctx, `SELECT id, email FROM users WHERE id = $1`, id).
  Scan(&u.ID, &u.Email)
switch {
case errors.Is(err, sql.ErrNoRows):
  return User{}, fmt.Errorf("user %d: %w", id, ErrUserNotFound)
case err != nil:
  return User{}, fmt.Errorf("querying user %d: %w", id, err)
}
```

Match it with `errors.Is`, never `==` ([8.3](../08-errors/03-is-as-unwrap.md)).
The bare comparison works only until some layer wraps with `%w`, and then a
404 silently becomes a 500 — exactly the failure
[16.9](../16-http/09-error-responses.md) catalogued.

Mapping it to a **domain** sentinel at the repository boundary, rather than
letting `sql.ErrNoRows` escape, is what keeps the service layer from importing
`database/sql`. That is 17.7's job.

## Result: RowsAffected, and the LastInsertId trap

`sql.Result` has exactly two methods, and on Postgres only one of them works:

```
RowsAffected=2 err=<nil>
LastInsertId=0 err=LastInsertId is not supported by this driver
```

`LastInsertId` is a MySQL/SQLite idea. Postgres has no such protocol field, so
the driver returns an error. The Postgres way is `RETURNING`, which is better
anyway because it gives you every server-generated column at once:

```go
var id int64
var createdAt time.Time
err := db.QueryRowContext(ctx,
  `INSERT INTO users (email, name) VALUES ($1, $2) RETURNING id, created_at`,
  email, name).Scan(&id, &createdAt)
```

Note that an INSERT with `RETURNING` is a **`QueryRow`**, not an `Exec`.

`RowsAffected` is reliable on Postgres and is how you detect a no-op UPDATE:

```go
tag, err := db.ExecContext(ctx, `UPDATE users SET name = $2 WHERE id = $1`, id, name)
if err != nil {
  return err
}
n, err := tag.RowsAffected()
if err != nil {
  return err
}
if n == 0 {
  return fmt.Errorf("user %d: %w", id, ErrUserNotFound)
}
```

Without that check, updating a row that does not exist succeeds silently.

## Placeholders are driver-specific

`database/sql` does not parse your SQL. The placeholder syntax is whatever the
server speaks:

| Driver | Placeholder | Example |
|---|---|---|
| Postgres (pgx, lib/pq) | `$1`, `$2` | `WHERE id = $1 AND org = $2` |
| MySQL | `?` | `WHERE id = ? AND org = ?` |
| SQLite | `?` or `$1` or `:name` | both accepted |
| Oracle | `:1` | `WHERE id = :1` |

So "driver-agnostic" code is not actually portable at the SQL level without
effort, which undercuts the main argument for `database/sql` in the first place.
Postgres's numbered placeholders have one real advantage: you can reference the
same argument twice, as `$1` in two clauses, with one value in the args list.

What is **not** negotiable: the values go through placeholders, never through
string formatting. `fmt.Sprintf` into SQL is an injection hole; 17.3 shows one
working.

## Prepared statements, and the one the driver makes for you

`PrepareContext` returns a `*sql.Stmt` you can reuse:

```go
stmt, err := db.PrepareContext(ctx, `SELECT id, email FROM users WHERE org_id = $1`)
if err != nil {
  return err
}
defer stmt.Close()

for _, org := range orgs {
  rows, err := stmt.QueryContext(ctx, org)
  // ...
}
```

A `*sql.Stmt` from `*sql.DB` is pool-aware: it re-prepares itself on whatever
connection it lands on, and tracks one server-side statement per connection.
That also makes it leaky if you keep many around — each one holds state on
every connection it has touched.

Here is the thing worth knowing: **on Postgres you usually do not need this.**
Every `QueryContext` with arguments already uses the extended protocol —
prepare, bind, execute — so you get the injection safety and the plan reuse
without a `*sql.Stmt`. pgx goes further and caches the prepared statement per
connection automatically (17.5 covers the exception, PgBouncer).

Idiomatic: do not reach for `Prepare` as an optimisation. Reach for it only
when you are running one statement in a tight loop and have measured that it
helps. Prefer `pgx.Batch` or `CopyFrom` (17.3) for bulk work.

## Why this module uses pgx directly

`database/sql` costs you something real on Postgres, because its interface is
the intersection of every database's features:

- **Types.** Everything crosses the boundary as one of six `driver.Value`
  kinds. Postgres arrays, `jsonb`, `numeric`, `uuid`, `tstzrange`, enums,
  composite types and `hstore` all arrive as `[]byte` for you to parse.
  pgx knows them natively (17.3).
- **`COPY`.** Postgres's bulk-load protocol has no `database/sql` expression
  at all. pgx exposes `CopyFrom`, which is an order of magnitude faster than
  row-by-row inserts.
- **`LISTEN`/`NOTIFY`.** Needs a pinned connection and an async message
  channel. pgx has `conn.WaitForNotification`; `database/sql` has nothing.
- **Errors.** `database/sql` gives you an opaque `error`. pgx gives you
  `*pgconn.PgError` with `Code`, `Detail`, `ConstraintName` and `ColumnName`,
  which is what makes "duplicate email becomes a 409" a three-line function
  instead of string matching (17.4).
- **Speed.** pgx's native path skips the `driver.Value` conversion and the
  `database/sql` bookkeeping, and uses the binary protocol for most types.
- **Batching.** `pgx.Batch` pipelines many statements in one round trip.

And the honest counterweight, in one line: **`database/sql` remains the right
choice when your code must work against more than one database engine** — a
library, a tool that supports both Postgres and SQLite, or a test suite that
runs on SQLite in CI. If that is you, use `database/sql`, and use pgx's
`stdlib` adapter as your Postgres driver so you can drop to the native API
later with `stdlib.AcquireConn`.

For a service that targets Postgres and only Postgres — which is this course's
stack — there is no reason to accept the lowest common denominator. 17.2
switches to `pgxpool`.

## Quick Reference

| Call | Note |
|---|---|
| `sql.Register(name, drv)` | from `init`; panics on a duplicate name |
| `sql.Drivers()` | sorted list of registered names |
| `sql.Open(name, dsn)` | **does not connect**; errors only on an unknown driver |
| `db.PingContext(ctx)` | the first thing that actually connects |
| `db.QueryContext` | `(*sql.Rows, error)` |
| `db.QueryRowContext` | `*sql.Row`; error surfaces at `Scan` |
| `db.ExecContext` | `(sql.Result, error)` |
| `rows.Next()` | `false` on both end-of-rows **and** error |
| `rows.Err()` | **must** be checked after the loop |
| `rows.Close()` | idempotent; `defer` it right after the error check |
| `sql.ErrNoRows` | from `QueryRow().Scan()`; match with `errors.Is` |
| `res.RowsAffected()` | works on Postgres |
| `res.LastInsertId()` | **errors** on Postgres; use `RETURNING` |

| Pool setting | Default | Set it to |
|---|---|---|
| `SetMaxOpenConns` | unlimited | a real number, under the server limit (17.5) |
| `SetMaxIdleConns` | **2** | the same as MaxOpenConns |
| `SetConnMaxLifetime` | forever | 30m–1h |
| `SetConnMaxIdleTime` | never | a few minutes |

| Symptom | Cause |
|---|---|
| `sql: unknown driver "postgres"` | missing blank import |
| service starts fine, first query fails | no `PingContext` at startup |
| truncated result sets, no error logged | `rows.Err()` not checked |
| handler hangs under load, database idle | leaked `*sql.Rows` — missing `Close` |
| `LastInsertId is not supported` | Postgres; use `RETURNING` |
| 404 became a 500 | `err == sql.ErrNoRows` instead of `errors.Is` |
| reconnect storm after every burst | default `MaxIdleConns` of 2 |

## Common Mistakes

**Assuming `sql.Open` connected.** It validates a driver name and nothing else,
so a typo'd host surfaces on the first query in production; `PingContext` with
a timeout at startup.

**Calling `sql.Open` per request.** You get a new pool each time and leak its
connections until GC; create one `*sql.DB` at startup and share it.

**Not checking `rows.Err()` after the loop.** A mid-iteration driver failure
ends the loop exactly like success, so you silently serve truncated data.

**Forgetting `defer rows.Close()`.** An early return pins a pool connection
until GC, and enough of them hang the service on acquire.

**Comparing with `err == sql.ErrNoRows`.** One `%w` wrap anywhere upstream
turns your 404 into a 500; use `errors.Is`.

**Calling `LastInsertId` on Postgres.** It is unsupported by the protocol; add
`RETURNING id` and use `QueryRow`.

**Leaving `MaxIdleConns` at its default of 2.** Every traffic burst is followed
by a reconnect storm; set it equal to `MaxOpenConns`.

**Using `db.Query` instead of `db.QueryContext`.** The plain form passes
`context.Background()`, so cancellation stops at your repository (16.12).

## Exercises

1. Extend the `mem` driver in this lesson to implement `driver.QueryerContext`
   and `driver.ExecerContext` directly, skipping the `Prepare` step. Prove with
   real output that `Prepare` is no longer called, and explain what
   `NumInput() == -1` means for argument checking.
2. Write a repository method `ListUsers` that commits the `rows.Err()` bug on
   purpose against the `broken` query, and a test that fails because of it.
   Then fix the method and show the same test passing.
3. Build a leak reproduction: a loop of 30 queries against a pool with
   `SetMaxOpenConns(5)` that never calls `rows.Close()`. Capture the hang with
   a `context.WithTimeout` on the acquire, then print `db.Stats()` to show
   where the connections went.
4. Write a wrapper type implementing `driver.Driver` that logs every query and
   its duration, then delegates to pgx's `stdlib` driver. Register it as
   `"pgx-logged"` and show the log lines from three queries.
5. Compare `db.QueryRowContext(...).Scan(&n)` for a row that does not exist
   against `db.QueryContext` plus a `rows.Next()` that returns `false`
   immediately. Show that one is an error and the other is not, and say which
   you would use in a repository and why.

---

[← 16.12 Context Propagation Through Handlers](../16-http/12-context-propagation.md) · [Course Index](../README.md) · [17.2 pgx: Connecting to Postgres →](02-pgx.md)
