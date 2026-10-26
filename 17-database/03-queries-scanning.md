[← 17.2 pgx: Connecting to Postgres](02-pgx.md) · [Course Index](../README.md) · [17.4 Transactions →](04-transactions.md)

# 17.3 — Queries, Scanning, and NULL Handling

## What You'll Learn

- `pgx.CollectRows` with `RowToStructByName` — the row-to-struct helper that removes the hand-written `Scan` list
- NULL handling in depth: `sql.NullString`, the generic `sql.Null[T]`, `*string`, and `pgtype` — all exercised by **real unit tests with no database**
- `pgx.ErrNoRows` is **not** `sql.ErrNoRows` but `errors.Is` bridges them in one direction only — the real output
- SQL injection shown working, and placeholders as the fix; plus `NamedArgs`, `Batch` and `CopyFrom`

The NULL, type-mapping, named-argument and SQL-building sections here run for
real. The blocks that need live rows are labelled; nothing else is.

## Coming From Another Language

| Other | Go |
|---|---|
| `rows.map(r => r.email)` | `pgx.CollectRows(rows, pgx.RowTo[string])` |
| an ORM materialising objects | `pgx.RowToStructByName[T]` — reflection, opt-in, one call |
| `null` collapsing into `undefined` | NULL needs a type that can **hold** NULL, or it is a scan error |
| `Optional[str]` / `str \| None` | `*string`, `sql.Null[string]`, or `pgtype.Text` — you choose |
| `cursor.execute(q % params)` | never; `$1` placeholders only |
| `Decimal` for money | `pgtype.Numeric`, never `float64` |

## The three query calls

```go
rows, err := pool.Query(ctx, sql, args...) // (pgx.Rows, error)
row := pool.QueryRow(ctx, sql, args...)    // pgx.Row — error deferred to Scan
tag, err := pool.Exec(ctx, sql, args...)   // (pgconn.CommandTag, error)
```

Two differences from `database/sql` worth noting immediately. There is no
`QueryContext`/`QueryRowContext` split, because **`ctx` is the first parameter
on every pgx method** — the non-context forms do not exist, so 16.12's rule is
enforced by the compiler. And `pgx.Rows` is an interface, not a struct, which
is what makes the collect helpers and test fakes possible.

`pgconn.CommandTag` carries `RowsAffected()`, plus `Insert()`, `Update()`,
`Delete()` and `Select()` predicates and the raw `String()`.

## CollectRows: stop writing Scan lists

The hand-written loop still works and is still correct:

```go
func manualScan(ctx context.Context, pool *pgxpool.Pool) ([]User, error) {
  rows, err := pool.Query(ctx,
    `SELECT id, email, name, bio, created_at, deleted_at FROM users ORDER BY id`)
  if err != nil {
    return nil, err
  }
  defer rows.Close()

  var out []User
  for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Email, &u.Name, &u.Bio, &u.CreatedAt, &u.DeletedAt); err != nil {
      return nil, err
    }
    out = append(out, u)
  }
  return out, rows.Err()
}
```

Six columns, six `&u.` expressions, in an order you must keep in sync with the
SELECT by hand. Add a column in the middle and nothing warns you — it compiles
and silently puts `created_at` into `Name` if the types happen to agree.

pgx v5's collect helpers replace all of it:

```go
package main

import (
  "context"
  "fmt"
  "time"

  "github.com/jackc/pgx/v5"
  "github.com/jackc/pgx/v5/pgxpool"
)

type User struct {
  ID        int64      `db:"id"`
  Email     string     `db:"email"`
  Name      string     `db:"name"`
  Bio       *string    `db:"bio"`
  CreatedAt time.Time  `db:"created_at"`
  DeletedAt *time.Time `db:"deleted_at"`
}

func listByName(ctx context.Context, pool *pgxpool.Pool, orgID int64) ([]User, error) {
  rows, err := pool.Query(ctx,
    `SELECT id, email, name, bio, created_at, deleted_at
     FROM users WHERE org_id = $1 ORDER BY id`, orgID)
  if err != nil {
    return nil, err
  }
  return pgx.CollectRows(rows, pgx.RowToStructByName[User])
}

func listPointers(ctx context.Context, pool *pgxpool.Pool) ([]*User, error) {
  rows, err := pool.Query(ctx, `SELECT id, email, name, bio, created_at, deleted_at FROM users`)
  if err != nil {
    return nil, err
  }
  return pgx.CollectRows(rows, pgx.RowToAddrOfStructByName[User])
}

func listMaps(ctx context.Context, pool *pgxpool.Pool) ([]map[string]any, error) {
  rows, err := pool.Query(ctx, `SELECT id, email FROM users`)
  if err != nil {
    return nil, err
  }
  return pgx.CollectRows(rows, pgx.RowToMap)
}

func listIDs(ctx context.Context, pool *pgxpool.Pool) ([]int64, error) {
  rows, err := pool.Query(ctx, `SELECT id FROM users ORDER BY id`)
  if err != nil {
    return nil, err
  }
  return pgx.CollectRows(rows, pgx.RowTo[int64])
}

func oneUser(ctx context.Context, pool *pgxpool.Pool, id int64) (User, error) {
  rows, err := pool.Query(ctx,
    `SELECT id, email, name, bio, created_at, deleted_at FROM users WHERE id = $1`, id)
  if err != nil {
    return User{}, err
  }
  return pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User])
}

func forEach(ctx context.Context, pool *pgxpool.Pool) error {
  rows, err := pool.Query(ctx, `SELECT id, email FROM users ORDER BY id`)
  if err != nil {
    return err
  }
  var (
    id    int64
    email string
  )
  _, err = pgx.ForEachRow(rows, []any{&id, &email}, func() error {
    fmt.Printf("  %d %s\n", id, email)
    return nil
  })
  return err
}

func main() {
  fns := []any{listByName, listPointers, listMaps, listIDs, oneUser, forEach}
  fmt.Printf("%d query functions compiled\n", len(fns))
}
```

Output:

```
6 query functions compiled
```

That is a compile check, not a demonstration — the functions need rows. But it
is the compile check that matters, because these are generic functions whose
type errors are not obvious.

| Helper | Produces | Notes |
|---|---|---|
| `pgx.RowToStructByName[T]` | `T` | matches **by column name**, case-insensitive, `db:"..."` tag override, `db:"-"` to skip |
| `pgx.RowToStructByNameLax[T]` | `T` | same, but tolerates struct fields with no matching column |
| `pgx.RowToStructByPos[T]` | `T` | matches by **position**; no tags needed, brittle to column reordering |
| `pgx.RowToAddrOfStructByName[T]` | `*T` | for `[]*T` |
| `pgx.RowTo[T]` | `T` | exactly one column, scanned into a scalar |
| `pgx.RowToMap` | `map[string]any` | dynamic shapes, ad-hoc tooling |
| `pgx.CollectRows(rows, fn)` | `([]T, error)` | **closes rows automatically**; returns `rows.Err()` |
| `pgx.CollectOneRow(rows, fn)` | `(T, error)` | `pgx.ErrNoRows` if empty; extra rows ignored |
| `pgx.CollectExactlyOneRow(rows, fn)` | `(T, error)` | `pgx.ErrNoRows` if empty, **`pgx.ErrTooManyRows`** if more than one |
| `pgx.ForEachRow(rows, dests, fn)` | `(pgconn.CommandTag, error)` | streaming; no slice is built |

Three things to internalise.

**`CollectRows` closes the rows and checks `rows.Err()` for you.** 17.1's
silent-truncation bug is structurally impossible when you use it — which is the
best reason to prefer it over the manual loop.

**`ByName` is strict about both directions by default.** `RowToStructByName`
requires the struct and the row to have the same number of fields. Select four
columns into a six-field struct and you get an error, not a half-filled struct.
Use `...Lax` when you deliberately select a subset.

**`CollectExactlyOneRow` is the right call for a "get by ID".** `CollectOneRow`
silently ignores extra rows, which hides a broken WHERE clause.

Idiomatic: `CollectRows` + `RowToStructByName` for lists, `CollectExactlyOneRow`
for single-row gets, `ForEachRow` when the result set is too big to hold in
memory, and the manual loop only when you are scanning into something that is
not a struct field.

## pgx.ErrNoRows and sql.ErrNoRows

They are different values, and the relationship between them is asymmetric.
This runs for real:

```go
package main

import (
  "database/sql"
  "errors"
  "fmt"

  "github.com/jackc/pgx/v5"
)

func main() {
  fmt.Printf("pgx.ErrNoRows == sql.ErrNoRows        : %v\n", pgx.ErrNoRows == sql.ErrNoRows)
  fmt.Printf("errors.Is(pgx.ErrNoRows, sql.ErrNoRows): %v\n", errors.Is(pgx.ErrNoRows, sql.ErrNoRows))
  fmt.Printf("errors.Is(sql.ErrNoRows, pgx.ErrNoRows): %v\n", errors.Is(sql.ErrNoRows, pgx.ErrNoRows))
  fmt.Printf("pgx.ErrNoRows.Error() : %q\n", pgx.ErrNoRows.Error())
  fmt.Printf("sql.ErrNoRows.Error() : %q\n", sql.ErrNoRows.Error())

  wrapped := fmt.Errorf("loading user 42: %w", pgx.ErrNoRows)
  fmt.Printf("\nwrapped: %v\n", wrapped)
  fmt.Printf("errors.Is(wrapped, pgx.ErrNoRows): %v\n", errors.Is(wrapped, pgx.ErrNoRows))
  fmt.Printf("errors.Is(wrapped, sql.ErrNoRows): %v\n", errors.Is(wrapped, sql.ErrNoRows))
}
```

Output:

```
pgx.ErrNoRows == sql.ErrNoRows        : false
errors.Is(pgx.ErrNoRows, sql.ErrNoRows): true
errors.Is(sql.ErrNoRows, pgx.ErrNoRows): false
pgx.ErrNoRows.Error() : "no rows in result set"
sql.ErrNoRows.Error() : "sql: no rows in result set"

wrapped: loading user 42: no rows in result set
errors.Is(wrapped, pgx.ErrNoRows): true
errors.Is(wrapped, sql.ErrNoRows): true
```

They are **not equal**, and their messages differ (`sql:` prefix). But
`pgx.ErrNoRows` *wraps* `sql.ErrNoRows`, so `errors.Is(err, sql.ErrNoRows)`
succeeds on a pgx error — and **not** the other way round.

The practical rule: in pgx code, match `pgx.ErrNoRows`. If you are migrating a
codebase from `database/sql`, your existing `errors.Is(err, sql.ErrNoRows)`
checks keep working unchanged, which is a deliberate kindness from pgx's
authors. What breaks is any surviving `err == sql.ErrNoRows`, because the
identity is now false — another reason 17.1 insisted on `errors.Is`.

```go
u, err := pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User])
switch {
case errors.Is(err, pgx.ErrNoRows):
  return User{}, fmt.Errorf("user %d: %w", id, ErrUserNotFound)
case errors.Is(err, pgx.ErrTooManyRows):
  return User{}, fmt.Errorf("user %d: %w", id, ErrDataCorrupt)
case err != nil:
  return User{}, fmt.Errorf("querying user %d: %w", id, err)
}
```

## NULL handling

This is the heart of the lesson, and all of it runs with no database at all,
because `Scan` and `Value` are ordinary methods you can call directly.

The problem in one line: **a Postgres NULL cannot be scanned into a `string`,
an `int64`, a `time.Time`, or any other non-nilable Go type.** The column type
is not the question; whether the *column* is nullable is. You need a target
that can represent "no value".

Four options, in rough order of how often you should reach for them:

| Option | Zero state | JSON | Use when |
|---|---|---|---|
| `*string`, `*int64`, `*time.Time` | `nil` | `null` automatically | **the default** — domain types, API responses |
| `sql.Null[T]` (Go 1.22+) | `{V: zero, Valid: false}` | needs a custom marshaller | generic code; you want the flag explicit |
| `sql.NullString` etc. | `{String: "", Valid: false}` | needs a custom marshaller | pre-generics code, or a `database/sql` codebase |
| `pgtype.Text`, `pgtype.Int4` … | `{..., Valid: false}` | needs a custom marshaller | Postgres-specific types: `numeric`, `uuid`, ranges, `jsonb` |

### Exercising them for real

```go
package nulltest

import (
  "database/sql"
  "testing"
)

func TestNullStringRoundTrip(t *testing.T) {
  var ns sql.NullString

  if ns.Valid {
    t.Fatalf("zero value must be invalid, got %+v", ns)
  }

  if err := ns.Scan("hello"); err != nil {
    t.Fatal(err)
  }
  if !ns.Valid || ns.String != "hello" {
    t.Fatalf("after Scan: %+v", ns)
  }
  v, err := ns.Value()
  if err != nil {
    t.Fatal(err)
  }
  if v != "hello" {
    t.Fatalf("Value() = %#v, want \"hello\"", v)
  }

  if err := ns.Scan(nil); err != nil {
    t.Fatal(err)
  }
  if ns.Valid || ns.String != "" {
    t.Fatalf("after Scan(nil): %+v", ns)
  }
  v, err = ns.Value()
  if err != nil {
    t.Fatal(err)
  }
  if v != nil {
    t.Fatalf("Value() = %#v, want nil", v)
  }
  t.Log("Scan(nil) clears the value AND the flag; Value() emits a real NULL")
}
```

The ten tests omitted for length — they bring in `encoding/json`, `strings`,
`time` and `pgtype` — all appear in the output below: `""` scanning as `Valid: true`; two failing `Scan`s; the generic `sql.Null[T]` over `int32`, a named string type and
`time.Time`; `*string` marshalling to JSON `null`; the five `pgtype` ones
(`Text`/`Int4`, `Timestamptz`, `UUID`, `Numeric`, arrays and JSON); and the
table-driven `NULL` versus `''` comparison.

`go test ./... -v` output:

```
=== RUN   TestNullStringRoundTrip
    null_test.go:47: Scan(nil) clears the value AND the flag; Value() emits a real NULL
--- PASS: TestNullStringRoundTrip (0.00s)
=== RUN   TestTheEmptyStringIsNotNull
    null_test.go:62: Scan("") -> {String: Valid:true}, Value() -> ""
--- PASS: TestTheEmptyStringIsNotNull (0.00s)
=== RUN   TestNullScanFailures
    null_test.go:74: NullInt64.Scan("abc") -> converting driver.Value type string ("abc") to a int64: invalid syntax
    null_test.go:81: NullFloat64.Scan(true) -> converting driver.Value type bool ("true") to a float64: invalid syntax
--- PASS: TestNullScanFailures (0.00s)
=== RUN   TestGenericNull
    null_test.go:99: sql.Null[int32] after Scan(int64(7)): {V:7 Valid:true}, Value() = 7
    null_test.go:109: sql.Null[Status] works on a named string type: {V:active Valid:true}
    null_test.go:116: sql.Null[time.Time]: Valid=true V=2026-03-01T12:00:00Z
--- PASS: TestGenericNull (0.00s)
=== RUN   TestPointerFieldsAsTheAlternative
    null_test.go:138: Bio == nil  -> {"name":"Ada","bio":null}
    null_test.go:139: Bio != nil  -> {"name":"Ada","bio":"mathematician"}
    null_test.go:140: a *string needs no .Valid check and needs no custom MarshalJSON
--- PASS: TestPointerFieldsAsTheAlternative (0.00s)
=== RUN   TestPgtypeTextAndInt4
    null_test.go:158: pgtype.Text: {String:hi Valid:true}, Value() = "hi"
    null_test.go:174: pgtype.Int4 stores an int32, scans from an int64: {Int32:99 Valid:true}
--- PASS: TestPgtypeTextAndInt4 (0.00s)
=== RUN   TestPgtypeTimestamptz
    null_test.go:189: pgtype.Timestamptz: Valid=true InfinityModifier=finite Time=2026-03-01T12:00:00Z
--- PASS: TestPgtypeTimestamptz (0.00s)
=== RUN   TestPgtypeUUID
    null_test.go:205: pgtype.UUID stores [16]byte, Value() = 6ba7b810-9dad-11d1-80b4-00c04fd430c8
    null_test.go:211: bad input -> cannot parse UUID not-a-uuid
--- PASS: TestPgtypeUUID (0.00s)
=== RUN   TestPgtypeNumericKeepsPrecision
    null_test.go:229: Int=12345600 Exp=-4 -> Value() = "1234.5600" (scale survives the round trip)
    null_test.go:235: Float64Value() = {Float64:1234.56 Valid:true}  <- lossy; never use this for money
--- PASS: TestPgtypeNumericKeepsPrecision (0.00s)
=== RUN   TestPgtypeArrayAndJSON
    null_test.go:243: pgtype.FlatArray[int32] = [1 2 3] (maps to int4[])
    null_test.go:257: jsonb scanned straight into a struct: {Theme:dark Emails:true}
--- PASS: TestPgtypeArrayAndJSON (0.00s)
=== RUN   TestNullAndEmptyAreDistinctInSQLSemantics
=== RUN   TestNullAndEmptyAreDistinctInSQLSemantics/NULL
    null_test.go:280: input <nil> -> Valid=false String="" trimmed-empty=true
=== RUN   TestNullAndEmptyAreDistinctInSQLSemantics/empty_string
    null_test.go:280: input "" -> Valid=true String="" trimmed-empty=true
=== RUN   TestNullAndEmptyAreDistinctInSQLSemantics/whitespace
    null_test.go:280: input " " -> Valid=true String=" " trimmed-empty=true
=== RUN   TestNullAndEmptyAreDistinctInSQLSemantics/a_value
    null_test.go:280: input "ada" -> Valid=true String="ada" trimmed-empty=false
--- PASS: TestNullAndEmptyAreDistinctInSQLSemantics (0.00s)
    --- PASS: TestNullAndEmptyAreDistinctInSQLSemantics/NULL (0.00s)
    --- PASS: TestNullAndEmptyAreDistinctInSQLSemantics/empty_string (0.00s)
    --- PASS: TestNullAndEmptyAreDistinctInSQLSemantics/whitespace (0.00s)
    --- PASS: TestNullAndEmptyAreDistinctInSQLSemantics/a_value (0.00s)
PASS
ok  	example.com/m17scratch/p_nulltest	0.482s
```

The payoff of writing them this way is that **your NULL handling has unit
tests** — no container, no fixtures, 0.48 seconds. 17.8 builds on exactly this
idea.

Seven things that output establishes.

**`sql.Null[T]` exists and works on Go 1.27.** Verified above on `int32`, on a
named string type, and on `time.Time`. It is the modern form and makes
`NullString`/`NullInt64`/`NullBool`/`NullFloat64`/`NullTime` redundant for new
code — though all five remain and you will keep meeting them.

**`Scan(nil)` resets the value as well as the flag.** `ns.String` went back to
`""`. So you cannot stash a value, scan a NULL over it, and read it back.

**`""` is not NULL.** `Scan("")` gives `Valid: true`. This is the single most
common NULL bug in a Go backend: a nullable `text` column where some rows hold
`NULL` and some hold `''`, and your code treats them as the same thing until a
`WHERE bio IS NOT NULL` disagrees with your Go code. Pick one representation in
a migration (usually: make the column `NOT NULL DEFAULT ''`, or forbid `''` with
a `CHECK`) and the problem disappears.

**A failed scan leaves `Valid: false`,** so you cannot distinguish "was NULL"
from "failed to convert" by looking at the struct. Check the error.

**`pgtype.Int4` holds an `Int32`** while scanning from an `int64`. The pgtype
names are the *Postgres* type names (`Int4` is `int4`, 32 bits), not the Go
ones. `Int8` holds an `Int64`.

**`pgtype.Numeric` preserves scale exactly.** `"1234.5600"` round-tripped with
its trailing zeroes, stored as `Int=12345600, Exp=-4`. `Float64Value()` gave
`1234.56` and lost them. For money, use `numeric` in Postgres and
`pgtype.Numeric` in Go, and never let a `float64` touch it.

**`Timestamptz` has an `InfinityModifier`.** Postgres timestamps can literally
be `infinity` and `-infinity`, which no `time.Time` can represent. The field is
`Finite` here; if you allow infinite timestamps, you must check it.

### Which one to pick

For the repository layer in 17.7, pointers:

```go
type User struct {
  ID        int64
  Email     string
  Name      string
  Bio       *string
  DeletedAt *time.Time
}
```

`*string` reads naturally (`if u.Bio != nil`), marshals to JSON `null` with no
extra code, and makes the nullability visible in the type. Its one cost is that
you cannot write a literal inline — `Bio: &"text"` is not legal Go, so you need
a helper:

```go
func ptr[T any](v T) *T { return &v }

u := User{Email: "ada@example.com", Bio: ptr("mathematician")}
```

Use `pgtype` when the Postgres type has no Go equivalent: `numeric`, `uuid`,
ranges, `jsonb` with a dynamic shape, `inet`, arrays with NULL elements. Use
`sql.Null[T]` in generic code where `*T` would mean another allocation per row.

### What a NULL into a non-nilable target looks like

Attempting to scan a NULL into a `string` is a scan error, not a zero value:

Expected output (requires a running Postgres; not executed in this environment):

```
can't scan NULL into *string
```

That error, on a column you thought was `NOT NULL`, almost always means your Go
struct and your migrations have drifted apart. The fix is in the schema or the
struct, not in the scan.

## Custom Scanner and Valuer

Any type implementing `sql.Scanner` and `driver.Valuer` can be a scan target.
This is how you turn a Postgres enum into a Go type that cannot hold an invalid
value:

```go
package scanner

import (
  "database/sql/driver"
  "fmt"
)

type Status string

const (
  StatusPending   Status = "pending"
  StatusActive    Status = "active"
  StatusSuspended Status = "suspended"
)

func (s Status) Valid() bool {
  switch s {
  case StatusPending, StatusActive, StatusSuspended:
    return true
  }
  return false
}

func (s *Status) Scan(src any) error {
  switch v := src.(type) {
  case nil:
    *s = StatusPending
    return nil
  case string:
    *s = Status(v)
  case []byte:
    *s = Status(v)
  default:
    return fmt.Errorf("scanning %T into Status: want string, []byte or nil", src)
  }
  if !s.Valid() {
    return fmt.Errorf("scanning %q into Status: not a known status", string(*s))
  }
  return nil
}

func (s Status) Value() (driver.Value, error) {
  if !s.Valid() {
    return nil, fmt.Errorf("storing Status %q: not a known status", string(s))
  }
  return string(s), nil
}
```

Note the receivers: `Scan` needs a **pointer** receiver because it mutates;
`Value` takes a value receiver. Getting that backwards is a silent failure —
the interface is simply not satisfied, and your column scans as a plain string.
Assert it at compile time:

```go
var (
  _ sql.Scanner   = (*Status)(nil)
  _ driver.Valuer = Status("")
)
```

And test it with no database — a table over `Scan` inputs (a string, bytes,
`nil`, an unknown value, a wrong type) plus the write side:

```go
func TestStatusValue(t *testing.T) {
  v, err := StatusActive.Value()
  if err != nil {
    t.Fatal(err)
  }
  if v != "active" {
    t.Fatalf("Value() = %#v", v)
  }
  t.Logf("StatusActive.Value() = %#v", v)

  if _, err := Status("deleted").Value(); err == nil {
    t.Fatal("a bogus Status must refuse to be written")
  } else {
    t.Logf("Status(\"deleted\").Value() -> %v", err)
  }
}
```

Output:

```
=== RUN   TestStatusScan
=== RUN   TestStatusScan/from_a_string
    status_test.go:44: "active" -> "active"
=== RUN   TestStatusScan/from_bytes
    status_test.go:44: []byte{0x73, 0x75, 0x73, 0x70, 0x65, 0x6e, 0x64, 0x65, 0x64} -> "suspended"
=== RUN   TestStatusScan/NULL_becomes_the_zero_state
    status_test.go:44: <nil> -> "pending"
=== RUN   TestStatusScan/an_unknown_value_is_rejected
    status_test.go:35: "deleted" -> scanning "deleted" into Status: not a known status
=== RUN   TestStatusScan/a_wrong_type_is_rejected
    status_test.go:35: 42 -> scanning int into Status: want string, []byte or nil
--- PASS: TestStatusScan (0.00s)
    --- PASS: TestStatusScan/from_a_string (0.00s)
    --- PASS: TestStatusScan/from_bytes (0.00s)
    --- PASS: TestStatusScan/NULL_becomes_the_zero_state (0.00s)
    --- PASS: TestStatusScan/an_unknown_value_is_rejected (0.00s)
    --- PASS: TestStatusScan/a_wrong_type_is_rejected (0.00s)
=== RUN   TestStatusValue
    status_test.go:57: StatusActive.Value() = "active"
    status_test.go:62: Status("deleted").Value() -> storing Status "deleted": not a known status
--- PASS: TestStatusValue (0.00s)
PASS
```

Handling **both** `string` and `[]byte` in `Scan` is not optional: which one
arrives depends on the driver, the protocol mode, and the column type, and a
type that only handles `string` works in tests and fails in production.

## timestamptz vs timestamp

Postgres has two timestamp types and only one of them is correct for an event.
`timestamptz` stores an absolute instant (internally UTC) and converts to the
session `TimeZone` on the way out; `timestamp` ("without time zone") stores
wall-clock digits with no zone at all, so the same value means a different
instant depending on who reads it. Go's `time.Time` always carries a location,
so scanning a `timestamp` gives you a `time.Time` whose zone is a fiction —
usually UTC — and the bug shows up as events an hour off twice a year.
**Use `timestamptz` for everything that happened**, use `date` for a birthday,
and reserve `timestamp` for the rare "9am local, wherever that is" case. In Go,
compare with `.Equal()` rather than `==`, because two `time.Time` values for
the same instant in different zones are not `==`.

## pgx.NamedArgs

Long queries with eight positional placeholders are hard to review. `NamedArgs`
rewrites `@name` into `$n` before the query is sent, and it is fully offline:

```go
package main

import (
  "fmt"

  "github.com/jackc/pgx/v5"
)

func main() {
  args := pgx.NamedArgs{
    "email":  "ada@example.com",
    "org_id": 7,
    "active": true,
  }
  sqlIn := `SELECT id, email FROM users
WHERE email = @email AND org_id = @org_id AND active = @active`

  out, rewritten, err := args.RewriteQuery(nil, nil, sqlIn, nil)
  fmt.Printf("err=%v\n", err)
  fmt.Printf("rewritten SQL:\n%s\n\n", out)
  fmt.Printf("args (positional order): %v\n", rewritten)

  fmt.Println("\n--- a name used twice gets ONE placeholder ---")
  out2, args2, _ := pgx.NamedArgs{"q": "ada"}.RewriteQuery(nil, nil,
    `SELECT * FROM users WHERE email ILIKE @q OR name ILIKE @q`, nil)
  fmt.Printf("%s\n%v\n", out2, args2)

  fmt.Println("\n--- a missing name becomes NULL ---")
  out3, args3, err3 := pgx.NamedArgs{"a": 1}.RewriteQuery(nil, nil,
    `SELECT @a, @b`, nil)
  fmt.Printf("%s\n%v  err=%v\n", out3, args3, err3)

  fmt.Println("\n--- @ inside a string literal is left alone ---")
  out4, args4, _ := pgx.NamedArgs{"x": 1}.RewriteQuery(nil, nil,
    `SELECT 'user@example.com', @x`, nil)
  fmt.Printf("%s\n%v\n", out4, args4)
}
```

Output:

```
err=<nil>
rewritten SQL:
SELECT id, email FROM users
WHERE email = $1 AND org_id = $2 AND active = $3

args (positional order): [ada@example.com 7 true]

--- a name used twice gets ONE placeholder ---
SELECT * FROM users WHERE email ILIKE $1 OR name ILIKE $1
[ada]

--- a missing name becomes NULL ---
SELECT $1, $2
[1 <nil>]  err=<nil>

--- @ inside a string literal is left alone ---
SELECT 'user@example.com', $1
[1]
```

In real code you never call `RewriteQuery` yourself — you pass the `NamedArgs`
as the single argument and pgx rewrites it:

```go
rows, err := pool.Query(ctx,
  `SELECT id, email FROM users WHERE org_id = @org AND status = @status`,
  pgx.NamedArgs{"org": orgID, "status": "active"})
```

Three things that output proves. The argument order follows **appearance in the
SQL**, not Go map order, so it is deterministic. A name used twice becomes one
placeholder with one value — which positional args cannot do as cleanly. The
`@` in `'user@example.com'` was correctly left alone, because the rewriter
understands string literals.

And one genuine hazard: **a missing name silently becomes NULL.** `@b` with no
`"b"` key produced `$2` bound to `nil` with **no error**. A typo'd key turns
`WHERE id = @userId` into `WHERE id = NULL`, which matches nothing and reports
nothing. That is the price of named args, and it is why they are a readability
tool for long queries rather than a blanket default.

## SQL injection

The vulnerable version, so you can recognise it:

```go
// WRONG. Never do this. This is a SQL injection vulnerability.
func findUserBroken(ctx context.Context, pool *pgxpool.Pool, email string) (User, error) {
  q := fmt.Sprintf("SELECT id, email, name FROM users WHERE email = '%s'", email)
  rows, err := pool.Query(ctx, q)
  if err != nil {
    return User{}, err
  }
  return pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User])
}
```

Call it with `email = "x' OR '1'='1"` and the SQL becomes:

```sql
SELECT id, email, name FROM users WHERE email = 'x' OR '1'='1'
```

which returns every user. Call it with `email = "x'; DROP TABLE users; --"` and
the statement becomes two statements. The correct version differs by one line:

```go
func findUser(ctx context.Context, pool *pgxpool.Pool, email string) (User, error) {
  rows, err := pool.Query(ctx,
    `SELECT id, email, name FROM users WHERE email = $1`, email)
  if err != nil {
    return User{}, err
  }
  return pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User])
}
```

The argument never becomes part of the SQL text. The server receives the query
and the value as separate protocol messages, so there is no string for the
attacker to escape out of. No escaping function, no allow-list, no sanitising —
placeholders are a different mechanism, not a safer string.

The rule that covers every case: **a value goes in a placeholder; an
identifier is a constant in your source.** Table names, column names and
`ORDER BY` directions cannot be placeholders, so if they must vary, map from a
closed set:

```go
var sortColumns = map[string]string{
  "created": "created_at",
  "email":   "email",
  "name":    "name",
}

func listSorted(ctx context.Context, pool *pgxpool.Pool, sortKey, dir string) (pgx.Rows, error) {
  col, ok := sortColumns[sortKey]
  if !ok {
    return nil, fmt.Errorf("sort key %q: %w", sortKey, ErrBadInput)
  }
  order := "ASC"
  if dir == "desc" {
    order = "DESC"
  }
  return pool.Query(ctx, fmt.Sprintf(
    `SELECT id, email, name FROM users ORDER BY %s %s LIMIT $1`, col, order), 50)
}
```

That `Sprintf` is safe because `col` and `order` can only ever be values you
wrote yourself. The user's string selects a key; it never reaches the SQL. 17.7
builds a full filter struct on this principle, with a test that proves user
input never lands in the query text.

## pgx.Batch

One round trip for many statements:

```go
func insertBatch(ctx context.Context, pool *pgxpool.Pool, users []NewUser) ([]int64, error) {
  batch := &pgx.Batch{}
  for _, u := range users {
    batch.Queue(
      `INSERT INTO users (org_id, email, name, status)
       VALUES ($1, $2, $3, 'pending') RETURNING id`,
      u.OrgID, u.Email, u.Name)
  }

  results := pool.SendBatch(ctx, batch)
  defer results.Close()

  ids := make([]int64, 0, len(users))
  for i := range users {
    var id int64
    if err := results.QueryRow().Scan(&id); err != nil {
      return nil, fmt.Errorf("batch item %d (%s): %w", i, users[i].Email, err)
    }
    ids = append(ids, id)
  }
  return ids, results.Close()
}
```

You must read **exactly one result per queued statement**, in order, and then
`Close`. Reading fewer leaves the connection with unread results; reading more
errors. The `defer results.Close()` plus the explicit final `results.Close()`
is intentional — `Close` is idempotent, the deferred one covers the error
returns, and the explicit one surfaces a close error on the success path.

v5 also lets you attach a handler per queued statement, which keeps the
error message next to the row it came from:

```go
func insertBatchWithQueuedFuncs(ctx context.Context, pool *pgxpool.Pool, users []NewUser) error {
  batch := &pgx.Batch{}
  for _, u := range users {
    batch.Queue(
      `INSERT INTO users (org_id, email, name, status)
       VALUES ($1, $2, $3, 'pending')`,
      u.OrgID, u.Email, u.Name,
    ).Exec(func(tag pgconn.CommandTag) error {
      if tag.RowsAffected() != 1 {
        return fmt.Errorf("%s: inserted %d rows", u.Email, tag.RowsAffected())
      }
      return nil
    })
  }
  return pool.SendBatch(ctx, batch).Close()
}
```

A batch is sent **inside an implicit transaction** when it is not already in
one: all the statements succeed or all fail. That is usually what you want, and
it is worth knowing before you batch a million rows into one call.

Expected output (requires a running Postgres; not executed in this environment):

```
inserted 3 users: [1 2 3]
```

## CopyFrom for bulk loads

`COPY` is Postgres's bulk ingest protocol and is dramatically faster than
row-by-row inserts for large loads. pgx exposes it directly:

```go
func copyUsers(ctx context.Context, pool *pgxpool.Pool, users []NewUser) (int64, error) {
  return pool.CopyFrom(ctx,
    pgx.Identifier{"users"},
    []string{"org_id", "email", "name"},
    pgx.CopyFromSlice(len(users), func(i int) ([]any, error) {
      u := users[i]
      if u.Email == "" {
        return nil, errors.New("email is required")
      }
      return []any{u.OrgID, u.Email, u.Name}, nil
    }),
  )
}

func copyFromRows(ctx context.Context, pool *pgxpool.Pool, rows [][]any) (int64, error) {
  return pool.CopyFrom(ctx,
    pgx.Identifier{"public", "users"},
    []string{"org_id", "email", "name"},
    pgx.CopyFromRows(rows),
  )
}
```

`pgx.Identifier` quotes correctly, which you can check offline:

```go
fmt.Printf("pgx.Identifier{\"public\",\"users\"}.Sanitize() = %s\n",
  pgx.Identifier{"public", "users"}.Sanitize())
fmt.Printf("pgx.Identifier{`weird\"name`}.Sanitize()       = %s\n",
  pgx.Identifier{`weird"name`}.Sanitize())
```

Output:

```
pgx.Identifier{"public","users"}.Sanitize() = "public"."users"
pgx.Identifier{`weird"name`}.Sanitize()       = "weird""name"
```

`CopyFromSlice` streams, so it does not buffer your whole dataset, and an error
from your callback aborts the copy. The limits worth knowing: `COPY` does not
support `ON CONFLICT`, does not give you `RETURNING`, and fires row triggers
but not statement triggers. The standard pattern for a large upsert is `COPY`
into an unlogged staging table, then one `INSERT ... SELECT ... ON CONFLICT`
from it.

Expected output (requires a running Postgres; not executed in this environment):

```
copied 50000 rows
```

## Quick Reference

| Call | Returns |
|---|---|
| `pool.Query(ctx, sql, args...)` | `(pgx.Rows, error)` |
| `pool.QueryRow(ctx, sql, args...)` | `pgx.Row`; error at `Scan` |
| `pool.Exec(ctx, sql, args...)` | `(pgconn.CommandTag, error)` |
| `pool.SendBatch(ctx, batch)` | `pgx.BatchResults` |
| `pool.CopyFrom(ctx, table, cols, src)` | `(int64, error)` |
| `tag.RowsAffected()` | `int64` |

| NULL target | Zero state | Note |
|---|---|---|
| `*T` | `nil` | the default; free JSON `null` |
| `sql.Null[T]` | `{V: zero, Valid: false}` | Go 1.22+; generic |
| `sql.NullString`/`Int64`/`Bool`/`Float64`/`Time` | `{_, Valid: false}` | pre-generics |
| `pgtype.Text`/`Int4`/`Int8`/`Bool`/`Float8` | `{_, Valid: false}` | Postgres-native |
| `pgtype.Timestamptz` | `{_, Valid: false}` | has `InfinityModifier` |
| `pgtype.UUID` | `{Bytes: [16]byte{}, Valid: false}` | `Scan` accepts the dashed string |
| `pgtype.Numeric` | `{_, Valid: false}` | exact scale; **use for money** |
| `pgtype.JSONB` / a struct | — | `jsonb` scans into a struct directly |
| `pgtype.FlatArray[T]`, `[]T` | `nil` | maps to `T[]` |

| Postgres type | Scan into |
|---|---|
| `text`, `varchar` | `string`, `*string` |
| `int4` / `int8` | `int32` / `int64` |
| `numeric` | `pgtype.Numeric` (**never `float64`**) |
| `bool` | `bool` |
| `timestamptz` | `time.Time` |
| `timestamp` | `time.Time` — but prefer `timestamptz` in the schema |
| `date` | `time.Time`, `pgtype.Date` |
| `uuid` | `pgtype.UUID`, `[16]byte`, or `string` |
| `json` / `jsonb` | any struct, `map[string]any`, `[]byte` |
| `int4[]` | `[]int32` |
| `text[]` | `[]string` |
| `bytea` | `[]byte` |
| an enum | `string` or a custom `Scanner` |

## Common Mistakes

**Scanning a nullable column into `string` or `time.Time`.** It is a scan error,
not a zero value; use `*T`, `sql.Null[T]`, or a `pgtype`.

**Treating `''` and `NULL` as the same value.** `Scan("")` is `Valid: true`, so
your Go code and your `IS NOT NULL` clauses disagree; normalise it in a
migration.

**Using `float64` for money.** `numeric` to `float64` loses precision silently;
use `pgtype.Numeric`.

**Writing `Scan` with a value receiver.** The interface is not satisfied, with
no compile error at the call site; use a pointer receiver and assert
`var _ sql.Scanner = (*T)(nil)`.

**Handling only `string` in a custom `Scan`.** `[]byte` arrives depending on
protocol mode and column type, so it works in tests and fails in production.

**Formatting values into SQL with `Sprintf`.** That is an injection hole; values
go in `$1` placeholders, and varying identifiers come from a closed map.

**Typing a `NamedArgs` key.** The missing name becomes NULL with no error, so
the query silently matches nothing.

**Using `timestamp` instead of `timestamptz`.** It stores no zone, so the same
value means different instants to different readers.

**Not reading one result per queued batch statement.** The connection is left
with unread results and the next query on it fails.

## Exercises

1. Convert a hand-written six-column `rows.Scan` loop to
   `pgx.CollectRows(rows, pgx.RowToStructByName[T])`, then deliberately add a
   column to the SELECT and show the error you get. Repeat with
   `RowToStructByNameLax` and explain the difference.
2. Write a `Money` type over `pgtype.Numeric` with `Scan`, `Value`,
   `MarshalJSON` and `String`, and a table-driven test proving `"0.10"` plus
   `"0.20"` is exactly `"0.30"` — then show the same arithmetic with `float64`
   failing.
3. Build the injection vulnerability from this lesson against the compose
   stack, prove it returns every row with `x' OR '1'='1`, then fix it and show
   the same input returning nothing.
4. Write a `ListFilter` with four optional fields and a test asserting that for
   every combination (16 cases) the generated SQL contains no user-supplied
   bytes and the placeholder numbering is contiguous from `$1`.
5. Benchmark three ways of inserting 10,000 rows — a loop of `Exec`, one
   `pgx.Batch`, and `CopyFrom` — with `testing.B` against the compose stack.
   Report ns/op and explain the ordering.

---

[← 17.2 pgx: Connecting to Postgres](02-pgx.md) · [Course Index](../README.md) · [17.4 Transactions →](04-transactions.md)
