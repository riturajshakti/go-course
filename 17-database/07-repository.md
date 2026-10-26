[← 17.6 Migrations with golang-migrate](06-migrations.md) · [Course Index](../README.md) · [17.8 Testing Against a Real Database →](08-testing-db.md)

# 17.7 — The Repository Layer

## What You'll Learn

- The interface belongs in the **service** package that consumes it, not the database package that implements it
- Domain-shaped methods (`GetUser`, `ListUsersByOrg`), never a `Query(sql string)` passthrough
- Transactions are owned by the **service**, not the repository — the most commonly botched design in Go backends
- Keyset pagination, the N+1 fix, and an in-memory implementation that makes the whole layer testable — with real `go test` output

Every test in this lesson runs for real against the in-memory implementation.
The `PostgresUserRepository` compiles and vets clean but needs a server to
execute, which is exactly the problem the in-memory one solves.

## Coming From Another Language

| Other | Go |
|---|---|
| a `UserRepository` base class | an interface; the struct does not declare it implements anything |
| the interface living next to its implementation | the interface lives with the **consumer** |
| `@Repository` + dependency injection container | a struct field set by the constructor in `main` |
| a generic `Repository<T>` with `findAll`, `save` | small, named, domain-shaped methods |
| `session.begin()` inside the repository | the service calls `withTx` and passes the `tx` down |
| an ORM lazily loading relations | you write the join, or you have an N+1 |

## Where the interface goes

This is the decision that determines whether the layer helps or hurts.

```
internal/
  user/                     <- the SERVICE package
    service.go              <- UserService, and the UserRepository interface
    user.go                 <- the User domain type
    errors.go               <- ErrUserNotFound, ErrDuplicateUser, ...
    memory.go               <- InMemoryUserRepository (for tests)
  postgres/
    users.go                <- PostgresUserRepository — implements user.UserRepository
    tx.go                   <- withTx, PgxTxManager
```

The `postgres` package imports `user`. The `user` package does **not** import
`postgres` — it does not import pgx at all. That direction is the whole point:

- The interface has exactly the methods **this** consumer needs
  ([5.5](../05-interfaces/05-composition.md)), not whatever the driver can do.
- Swapping Postgres for an in-memory fake, or adding a caching decorator, needs
  no change in `user`.
- You can read `service.go` and know every database operation the service
  performs without opening the SQL.
- The domain is reusable from a CLI or a gRPC server, exactly as
  [16.9](../16-http/09-error-responses.md) argued for keeping status codes out
  of it.

"Accept interfaces, return structs"
([5.1](../05-interfaces/01-implicit-satisfaction.md)) is the slogan;
`NewPostgresUserRepository` returns a `*PostgresUserRepository`, and
`NewUserService` accepts a `UserRepository`.

The failure mode to avoid is declaring the interface in `postgres` and having
the service import it: the service then depends on the database package, the
interface grows to whatever Postgres can do, and you have added a file without
decoupling anything.

### Keep it small and domain-shaped

```go
type UserRepository interface {
  GetUser(ctx context.Context, id int64) (User, error)
  GetUserByEmail(ctx context.Context, email string) (User, error)
  CreateUser(ctx context.Context, u User) (User, error)
  UpdateUserStatus(ctx context.Context, id int64, status string) error
  ListUsers(ctx context.Context, f ListFilter) ([]User, error)
  ListUsersByIDs(ctx context.Context, ids []int64) ([]User, error)
  CountUsersByOrg(ctx context.Context, orgID int64) (int64, error)
}
```

Every method takes `ctx` first (16.12), names a domain operation, and speaks in
domain types. Compare the anti-pattern:

```go
// WRONG: this is not a repository, it is a database handle with extra steps.
type Repository interface {
  Query(ctx context.Context, sql string, args ...any) (pgx.Rows, error)
  Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
}
```

That interface leaks `pgx` into every consumer, cannot be faked without writing
a SQL engine, and lets any handler run any query. It is strictly worse than
passing the pool around, because it adds indirection and removes nothing.

The other direction is just as wrong: a generic
`Repository[T]` with `FindAll`, `FindByID`, `Save` and `Delete`. `FindAll` on a
users table is a production incident, and `Save` cannot express "insert, and
tell me the generated ID" differently from "update these three columns".
[9.5](../09-generics/05-when-not-to.md) covers why generics do not help here:
the operations differ per entity, and that difference is the information you
want in the type.

## The domain type and the row type

Start with **one** struct used for both, and split only when they genuinely
diverge:

```go
type User struct {
  ID        int64
  OrgID     int64
  Email     string
  Name      string
  Bio       *string
  Status    string
  CreatedAt time.Time
}
```

Separate them when the database shape stops matching the domain shape — a
column you store but never expose, a `jsonb` blob you unpack into several
domain fields, a join that produces a shape no table has, or soft-delete
bookkeeping the domain should not see. Until then, two structs plus a mapper is
pure cost: every new column means three edits, the mapper is untested
boilerplate, and its bugs are silent field drops. Split when it hurts.

What the domain type must **not** have is `db:"..."` tags for columns it does
not own, `pgtype` fields, or a `Scan` method — those belong on a row type in
the `postgres` package if you need one.

## The Postgres implementation

```go
package postgres

import (
  "context"
  "fmt"
  "strings"

  "github.com/jackc/pgx/v5"
  "github.com/jackc/pgx/v5/pgconn"
  "github.com/jackc/pgx/v5/pgxpool"
)

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

type PostgresUserRepository struct {
  db DBTX
}

func NewPostgresUserRepository(db DBTX) *PostgresUserRepository {
  return &PostgresUserRepository{db: db}
}

const userColumns = `id, org_id, email, name, bio, status, created_at`

func (r *PostgresUserRepository) GetUser(ctx context.Context, id int64) (User, error) {
  rows, err := r.db.Query(ctx,
    `SELECT `+userColumns+` FROM users WHERE id = $1`, id)
  if err != nil {
    return User{}, mapUserError("GetUser", err)
  }
  u, err := pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User])
  if err != nil {
    return User{}, mapUserError("GetUser", err)
  }
  return u, nil
}

func (r *PostgresUserRepository) CreateUser(ctx context.Context, u User) (User, error) {
  rows, err := r.db.Query(ctx,
    `INSERT INTO users (org_id, email, name, bio, status)
     VALUES ($1, $2, $3, $4, $5)
     RETURNING `+userColumns,
    u.OrgID, u.Email, u.Name, u.Bio, u.Status)
  if err != nil {
    return User{}, mapUserError("CreateUser", err)
  }
  created, err := pgx.CollectExactlyOneRow(rows, pgx.RowToStructByName[User])
  if err != nil {
    return User{}, mapUserError("CreateUser", err)
  }
  return created, nil
}

var _ UserRepository = (*PostgresUserRepository)(nil)
```

Five things in that code worth naming.

**`db` is a `DBTX`, not a `*pgxpool.Pool`** (17.4). Every method works
unchanged inside a transaction, and the narrow interface means a method
physically cannot call `Begin`.

**Every error goes through `mapUserError`** (17.4), so SQLSTATE never escapes
this package and the service only ever sees `ErrUserNotFound`,
`ErrDuplicateUser` and friends.

**`UpdateUserStatus`, omitted above, checks `RowsAffected() == 0`** and returns
`ErrUserNotFound`. An UPDATE against a nonexistent row succeeds silently;
without that check your API returns 204 for a user that does not exist.

**`ListUsersByIDs` and `CountUsersByOrg` are omitted above** —
`SELECT ... WHERE id = ANY($1) ORDER BY id` collected with `CollectRows`, and
`SELECT count(*) ... WHERE org_id = $1` scanned into an `int64`. The first
returns early on an empty slice, because `id = ANY('{}')` matches nothing so
the query is correct but pointless.

`GetUserByEmail` is omitted above: it is `GetUser` with
`WHERE lower(email) = lower($1)`, which is what matches the functional unique
index from 17.6.

**`var _ UserRepository = (*PostgresUserRepository)(nil)`** fails the build the
moment the interface and the implementation drift. In the layout above this
assertion lives in the `postgres` package, which is the package that imports
`user` — the right direction.

## A ListFilter and safe dynamic SQL

Optional filters mean the SQL varies. The placeholders must still be
placeholders:

```go
type ListFilter struct {
  OrgID  int64
  Status string
  Search string
  Limit  int32
  After  int64
}

func buildListUsers(f ListFilter) (string, []any) {
  where := []string{"org_id = $1"}
  args := []any{f.OrgID}

  next := func() int { return len(args) + 1 }

  if f.Status != "" {
    where = append(where, fmt.Sprintf("status = $%d", next()))
    args = append(args, f.Status)
  }
  if f.Search != "" {
    n := next()
    where = append(where, fmt.Sprintf("(name ILIKE $%d OR email ILIKE $%d)", n, n))
    args = append(args, "%"+f.Search+"%")
  }
  if f.After > 0 {
    where = append(where, fmt.Sprintf("id > $%d", next()))
    args = append(args, f.After)
  }

  limit := f.Limit
  if limit <= 0 || limit > 100 {
    limit = 50
  }
  q := fmt.Sprintf("SELECT %s FROM users WHERE %s ORDER BY id ASC LIMIT $%d",
    userColumns, strings.Join(where, " AND "), next())
  args = append(args, limit)
  return q, args
}

func (r *PostgresUserRepository) ListUsers(ctx context.Context, f ListFilter) ([]User, error) {
  q, args := buildListUsers(f)
  rows, err := r.db.Query(ctx, q, args...)
  if err != nil {
    return nil, mapUserError("ListUsers", err)
  }
  users, err := pgx.CollectRows(rows, pgx.RowToStructByName[User])
  if err != nil {
    return nil, mapUserError("ListUsers", err)
  }
  return users, nil
}
```

The `Sprintf` only ever formats **integers that this function computed**.
Nothing from the filter reaches the SQL text. `next()` reading `len(args)+1`
keeps the numbering contiguous however many clauses fire, and the search clause
reuses one placeholder in two positions — which positional arguments alone
cannot do.

The `limit` clamp is not decoration. Without it, a client sends `?limit=1000000`
and your service tries to materialise a million rows.

Splitting `buildListUsers` out as a pure function is what makes it testable
with no database. The suite is a table over filter combinations asserting the
exact SQL and args, plus the one that matters most:

```go
func TestBuildListUsersNeverInterpolates(t *testing.T) {
  evil := "'; DROP TABLE users; --"
  q, args := buildListUsers(ListFilter{OrgID: 1, Search: evil, Status: evil})
  if strings.Contains(q, "DROP") {
    t.Fatalf("user input reached the SQL text: %s", q)
  }
  t.Logf("SQL:  %s", q)
  t.Logf("args: %#v", args)
}
```

Output:

```
=== RUN   TestBuildListUsers
--- PASS: TestBuildListUsers (0.00s)
    --- PASS: TestBuildListUsers/org_only,_default_limit (0.00s)
    --- PASS: TestBuildListUsers/status_filter (0.00s)
    --- PASS: TestBuildListUsers/search_reuses_one_placeholder_twice (0.00s)
    --- PASS: TestBuildListUsers/keyset_cursor (0.00s)
    --- PASS: TestBuildListUsers/everything_at_once (0.00s)
    --- PASS: TestBuildListUsers/an_absurd_limit_is_clamped (0.00s)
=== RUN   TestBuildListUsersNeverInterpolates
    buildsql_test.go:85: SQL:  SELECT id, org_id, email, name, bio, status, created_at FROM users WHERE org_id = $1 AND status = $2 AND (name ILIKE $3 OR email ILIKE $3) ORDER BY id ASC LIMIT $4
    buildsql_test.go:86: args: []interface {}{1, "'; DROP TABLE users; --", "%'; DROP TABLE users; --%", 50}
--- PASS: TestBuildListUsersNeverInterpolates (0.00s)
PASS
```

The second test is the one to keep: `DROP TABLE users` is sitting in `args`,
where it is a harmless string, and the SQL text is unchanged. That is a
regression test against the vulnerability from 17.3, and it costs nothing to
run.

The six table cases named in that output cover org-only, status, search,
keyset, all-filters-at-once, and an absurd limit; each asserts the exact SQL
string and the exact args slice. When the filter
combinatorics outgrow hand-built strings, a query builder
(squirrel, goqu) is a reasonable next step — but note you have just traded
reviewable SQL for a DSL, and the bugs move from the string concatenation to
the builder's semantics.

## Pagination: keyset beats OFFSET

The obvious version:

```sql
SELECT id, org_id, email, name, bio, status, created_at
FROM users WHERE org_id = $1
ORDER BY id ASC
LIMIT $2 OFFSET $3
```

Two problems, and the second is worse than the first.

**`OFFSET n` makes the database read and discard `n` rows.** Page 1 is fast,
page 1000 reads a million rows to return fifty. Latency grows linearly with
page number, so the slowest requests are the ones from your most engaged users
and from crawlers.

**It skips and duplicates rows under concurrent writes.** If a row is inserted
before your position between page 2 and page 3, every subsequent row shifts
down by one and you never see the one at the boundary. If a row is deleted, one
row appears on two pages. The window is not a snapshot.

Keyset (cursor) pagination fixes both by remembering **where you were** rather
than **how many you skipped**:

```sql
SELECT id, org_id, email, name, bio, status, created_at
FROM users WHERE org_id = $1 AND id > $2
ORDER BY id ASC
LIMIT $3
```

Constant time at any depth, because the index seeks straight to `id > cursor`,
and stable under inserts and deletes. The cost is that you cannot jump to page
1000 — which is a feature, since nobody does that except crawlers.

The service turns it into a page with a next cursor by asking for one row more
than it needs and trimming:

```go
type Page struct {
  Users      []User
  NextCursor int64
  HasMore    bool
}

func (s *UserService) Page(ctx context.Context, f ListFilter) (Page, error) {
  probe := f
  probe.Limit = f.Limit + 1

  users, err := s.repo.ListUsers(ctx, probe)
  if err != nil {
    return Page{}, err
  }
  p := Page{Users: users}
  if int32(len(users)) > f.Limit {
    p.Users = users[:f.Limit]
    p.HasMore = true
  }
  if n := len(p.Users); n > 0 {
    p.NextCursor = p.Users[n-1].ID
  }
  return p, nil
}
```

Asking for `Limit + 1` and trimming is how you know `HasMore` without a second
`COUNT(*)` query. A `COUNT(*)` over a filtered set is often more expensive than
the page itself.

Two refinements for real use. **Order by something unique**, or add a
tiebreaker — `ORDER BY created_at DESC, id DESC` with the cursor comparing the
pair, because two rows sharing a `created_at` make a single-column cursor skip
one. And **opaque cursors**, base64 of the key values, so clients cannot
construct one and you can change the key without breaking them.

## The N+1 problem

The shape: a loop over org IDs calling `s.orgs.GetOrg(ctx, id)` and
`s.repo.ListUsers(ctx, ListFilter{OrgID: id})` for each one.

Fifty orgs is 101 round trips. At 1ms each that is 101ms of pure latency, and
it scales with the input. It is easy to miss in review because the loop looks
innocent, and in development with three rows it is fast.

**Fix 1: one query with `= ANY($1)`.** Postgres arrays make a single
parameterised query over a set, with no SQL built from the slice length:

```go
func (r *PostgresUserRepository) ListUsersByOrgs(ctx context.Context, orgIDs []int64) (map[int64][]User, error) {
  if len(orgIDs) == 0 {
    return nil, nil
  }
  rows, err := r.db.Query(ctx,
    `SELECT `+userColumns+`
     FROM users WHERE org_id = ANY($1) ORDER BY org_id, id`, orgIDs)
  if err != nil {
    return nil, mapUserError("ListUsersByOrgs", err)
  }
  users, err := pgx.CollectRows(rows, pgx.RowToStructByName[User])
  if err != nil {
    return nil, mapUserError("ListUsersByOrgs", err)
  }
  byOrg := make(map[int64][]User, len(orgIDs))
  for _, u := range users {
    byOrg[u.OrgID] = append(byOrg[u.OrgID], u)
  }
  return byOrg, nil
}
```

Grouping in Go rather than in SQL is deliberate: `array_agg` into a `jsonb`
column works, but then you are parsing JSON to rebuild structs you already have
a scanner for.

`= ANY($1)` is strictly better than building `IN ($1, $2, $3, ...)`: one
placeholder, one query plan regardless of the list length, and no string
building. Note Postgres caps bind parameters at 65535, which `= ANY` sidesteps
entirely since the whole array is one parameter.

**Fix 2: a join, when you want the rows together.** One query selecting
`u.*, o.name AS org_name` with `JOIN orgs o ON o.id = u.org_id WHERE
u.org_id = ANY($1)`, collected with `pgx.CollectRows(rows,
pgx.RowToStructByName[UserWithOrg])`.

`UserWithOrg` is the case where a separate row type is justified: its shape
exists only as a query result, no table has it, and it belongs in the
`postgres` package:

```go
type UserWithOrg struct {
  ID        int64     `db:"id"`
  OrgID     int64     `db:"org_id"`
  Email     string    `db:"email"`
  Name      string    `db:"name"`
  Bio       *string   `db:"bio"`
  Status    string    `db:"status"`
  CreatedAt time.Time `db:"created_at"`
  OrgName   string    `db:"org_name"`
}
```

The column order in the struct must match the SELECT for
`RowToStructByPos`; with `RowToStructByName` only the names matter, which is
why the `db:"org_name"` tag on the aliased column is the load-bearing line.

Choose by cardinality: a join duplicates the parent's columns once per child,
so for one org with ten thousand users the `= ANY` plus grouping moves far less
data. For a handful of children each, the join is simpler.

**Fix 3: a batch, when the queries genuinely differ.** `pgx.Batch` (17.3) is
one round trip for N different statements, which is the fallback for when one
query genuinely cannot express the work. For *this* example a single
`SELECT org_id, count(*) FROM users WHERE org_id = ANY($1) GROUP BY org_id`
beats the batch, and that is the general rule: reach for `GROUP BY` or
`= ANY` first, and batch only when the statements are actually different.

## Transactions belong to the service

This is the single most commonly botched piece of layering in Go backends.

The broken version puts the transaction inside the repository — a
`CreateUserAndAudit` method that calls `r.pool.Begin(ctx)` itself.

Once a repository method owns a transaction, any operation spanning two
methods needs a third method that does both — so the repository grows a method
per use case, each with its own transaction, and the SQL duplicates. You have
pushed business logic into the database layer, which is exactly what the layer
was supposed to prevent.

The correct split: **the repository knows how to run one operation on whatever
`DBTX` it was given; the service decides what is atomic.**

```go
type TxManager interface {
  WithTx(ctx context.Context, fn func(UserRepository) error) error
}
```

That interface lives in the **service** package next to `UserRepository`, and
it is deliberately shaped around `UserRepository` rather than exposing
`pgx.Tx` — so the service never imports pgx. The Postgres implementation:

```go
type PgxTxManager struct {
  pool *pgxpool.Pool
  opts pgx.TxOptions
}

func NewPgxTxManager(pool *pgxpool.Pool, opts pgx.TxOptions) *PgxTxManager {
  return &PgxTxManager{pool: pool, opts: opts}
}

func (m *PgxTxManager) WithTx(ctx context.Context, fn func(UserRepository) error) error {
  return withTx(ctx, m.pool, m.opts, func(tx pgx.Tx) error {
    return fn(NewPostgresUserRepository(tx))
  })
}

var _ TxManager = (*PgxTxManager)(nil)
```

Four lines of body, and they are the hinge of the whole design: `withTx`
(17.4) opens the transaction, and the callback gets a **repository bound to
that `tx`**. Every method the service calls on it runs inside the transaction,
because `DBTX` is satisfied by `pgx.Tx`.

The service:

```go
type UserService struct {
  repo UserRepository
  tx   TxManager
}

func NewUserService(repo UserRepository, tx TxManager) *UserService {
  return &UserService{repo: repo, tx: tx}
}

var ErrOrgFull = errors.New("org has reached its seat limit")

const seatLimit = 3

func (s *UserService) Register(ctx context.Context, orgID int64, email, name string) (User, error) {
  email = strings.TrimSpace(strings.ToLower(email))
  if email == "" {
    return User{}, fmt.Errorf("Register: email required: %w", ErrInvalidUser)
  }
  var created User
  err := s.tx.WithTx(ctx, func(repo UserRepository) error {
    n, err := repo.CountUsersByOrg(ctx, orgID)
    if err != nil {
      return err
    }
    if n >= seatLimit {
      return fmt.Errorf("Register: org %d has %d of %d seats: %w",
        orgID, n, seatLimit, ErrOrgFull)
    }
    created, err = repo.CreateUser(ctx, User{
      OrgID:  orgID,
      Email:  email,
      Name:   name,
      Status: "pending",
    })
    if err != nil {
      return err
    }
    return repo.UpdateUserStatus(ctx, created.ID, "active")
  })
  if err != nil {
    return User{}, err
  }
  created.Status = "active"
  return created, nil
}
```

Read what `Register` does: **three repository calls, one transaction, and the
business rule between them.** The seat check and the insert are atomic, so two
concurrent registrations cannot both pass the check. The repository knows
nothing about seat limits; the service knows nothing about SQL.

`Activate`, not shown, uses `s.repo` directly because it needs no atomicity —
read the status, return early if it is already active, otherwise update. Not
everything needs a transaction, and wrapping single statements in one is just
overhead.

Wiring it up in `main` ([18.1](../18-production/01-project-layout.md)) is the
only place that knows both the interface and the implementation:
`postgres.NewPostgresUserRepository(pool)`,
`postgres.NewPgxTxManager(pool, pgx.TxOptions{})`, then
`user.NewUserService(repo, txm)`. That is the dependency injection; there is no
container.

## The in-memory implementation

The same interface, backed by a map. This is what makes the whole layer
testable without Docker, and it is what 17.8 builds on:

```go
type InMemoryUserRepository struct {
  mu     sync.Mutex
  nextID int64
  users  map[int64]User
  orgs   map[int64]bool
  now    func() time.Time
}

func NewInMemoryUserRepository(orgIDs ...int64) *InMemoryUserRepository {
  orgs := make(map[int64]bool, len(orgIDs))
  for _, id := range orgIDs {
    orgs[id] = true
  }
  fixed := time.Date(2026, 3, 1, 12, 0, 0, 0, time.UTC)
  return &InMemoryUserRepository{
    nextID: 1,
    users:  map[int64]User{},
    orgs:   orgs,
    now:    func() time.Time { return fixed },
  }
}

func (r *InMemoryUserRepository) CreateUser(ctx context.Context, u User) (User, error) {
  if err := ctx.Err(); err != nil {
    return User{}, err
  }
  r.mu.Lock()
  defer r.mu.Unlock()
  if u.Email == "" {
    return User{}, fmt.Errorf("CreateUser: column %q must not be null: %w", "email", ErrInvalidUser)
  }
  if len(r.orgs) > 0 && !r.orgs[u.OrgID] {
    return User{}, fmt.Errorf("CreateUser: %w", ErrOrgNotFound)
  }
  for _, existing := range r.users {
    if strings.EqualFold(existing.Email, u.Email) {
      return User{}, fmt.Errorf("CreateUser: %w", ErrDuplicateUser)
    }
  }
  u.ID = r.nextID
  r.nextID++
  if u.Status == "" {
    u.Status = "pending"
  }
  u.CreatedAt = r.now()
  r.users[u.ID] = u
  return u, nil
}

func (r *InMemoryUserRepository) WithTx(ctx context.Context, fn func(UserRepository) error) error {
  r.mu.Lock()
  snapshot := maps.Clone(r.users)
  savedID := r.nextID
  r.mu.Unlock()

  if err := fn(r); err != nil {
    r.mu.Lock()
    r.users = snapshot
    r.nextID = savedID
    r.mu.Unlock()
    return err
  }
  return nil
}

var (
  _ UserRepository = (*InMemoryUserRepository)(nil)
  _ TxManager      = (*InMemoryUserRepository)(nil)
)
```

`sortedLocked()` returns the map's values sorted by ID, and `matches` is a
case-insensitive substring test over name and email.

Six more methods are omitted above for length. `GetUser` is a map lookup returning
`ErrUserNotFound` on a miss; `CountUsersByOrg` counts matching rows.
`GetUserByEmail` scans
`sortedLocked()` with `strings.EqualFold`, mirroring the `lower(email)` index.
`UpdateUserStatus` looks the row up and returns `ErrUserNotFound` if it is
missing, matching the Postgres version's `RowsAffected() == 0` check.
`ListUsers` walks `sortedLocked()`
applying the same four filters `buildListUsers` turns into SQL — org, status,
`After` cursor and search — and stops at `limit`, clamped identically.
`ListUsersByIDs` is `sortedLocked()` filtered by `slices.Contains(ids, u.ID)`,
with the same early return on an empty slice as the Postgres version. Both
check `ctx.Err()` first.

Six design notes, because a fake that lies is worse than no fake at all.

**It implements both interfaces.** `WithTx` snapshots the map and restores it on
error, so rollback is real — which is what lets the whole service, atomicity
included, be tested with no database.

**Every method checks `ctx.Err()` first,** like a real driver does. Without
that, a test for cancellation passes against Postgres and fails against the
fake, or worse, the reverse.

**It returns the **same** sentinel errors** as the Postgres version:
`ErrUserNotFound`, `ErrDuplicateUser`, `ErrOrgNotFound`, `ErrInvalidUser`. A
fake that returns different errors tests nothing about your error handling.

**It enforces the same constraints.** Case-insensitive unique email (matching
the `UNIQUE INDEX ON users (lower(email))` from 17.6), a non-empty email
mirroring `NOT NULL`, and a foreign key on `org_id`. These are the rules the
database would enforce, so the fake must too or your tests pass and production
409s.

**`now` is injectable,** so `CreatedAt` is deterministic and assertions on it
do not flake.

**Results are sorted by ID,** because map iteration order is random
([7.6](../07-slices-maps/06-maps.md)) and a fake with nondeterministic ordering
produces tests that fail one run in ten.


The honest limits: no isolation levels, a non-reentrant `WithTx`, substring
search rather than `ILIKE` semantics, and no way to catch a SQL syntax error or
a missing index. Those are what 17.8's integration tests are for. The fake
tests your **logic**; only a real database tests your **SQL**.

## The service tested against it

```go
func newFixture(t *testing.T) (*InMemoryUserRepository, *UserService) {
  t.Helper()
  repo := NewInMemoryUserRepository(1, 2)
  return repo, NewUserService(repo, repo)
}

func TestRegisterRollsBackWhenTheOrgIsFull(t *testing.T) {
  ctx := context.Background()
  repo, svc := newFixture(t)

  for _, e := range []string{"a@x.com", "b@x.com", "c@x.com"} {
    if _, err := svc.Register(ctx, 1, e, e); err != nil {
      t.Fatal(err)
    }
  }
  before, _ := repo.CountUsersByOrg(ctx, 1)

  _, err := svc.Register(ctx, 1, "d@x.com", "Dan")
  if !errors.Is(err, ErrOrgFull) {
    t.Fatalf("want ErrOrgFull, got %v", err)
  }
  after, _ := repo.CountUsersByOrg(ctx, 1)
  if before != after {
    t.Fatalf("the transaction must have rolled back: %d -> %d", before, after)
  }
  if _, err := repo.GetUserByEmail(ctx, "d@x.com"); !errors.Is(err, ErrUserNotFound) {
    t.Fatalf("the rejected user must not exist, got %v", err)
  }
  t.Logf("err = %v; count stayed at %d", err, after)
}

type statusFailingRepo struct {
  *InMemoryUserRepository
}

func (r *statusFailingRepo) UpdateUserStatus(ctx context.Context, id int64, status string) error {
  return errors.New("deadlock detected")
}

func (r *statusFailingRepo) WithTx(ctx context.Context, fn func(UserRepository) error) error {
  return r.InMemoryUserRepository.WithTx(ctx, func(UserRepository) error { return fn(r) })
}

func TestRegisterIsAtomicAcrossTwoWrites(t *testing.T) {
  ctx := context.Background()
  repo := NewInMemoryUserRepository(1)
  failing := &statusFailingRepo{InMemoryUserRepository: repo}
  svc := NewUserService(failing, failing)

  _, err := svc.Register(ctx, 1, "ada@example.com", "Ada")
  if err == nil {
    t.Fatal("want the UpdateUserStatus failure to surface")
  }
  n, _ := repo.CountUsersByOrg(ctx, 1)
  if n != 0 {
    t.Fatalf("the INSERT must have rolled back with the UPDATE, count = %d", n)
  }
  t.Logf("err = %v; no partial row left behind", err)
}


```

Shown above are the two tests that justify the whole design. The other nine
appear in the output below: happy path, duplicate email, unknown org,
idempotent activate, unknown user, the cursor walk, the `= ANY` batch load, the
cancelled-context check, and the missing-email rejection.

Output:

```
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
```

Eleven tests, no database, under half a second. Three of them are the ones that
justify the whole design.

**`TestRegisterRollsBackWhenTheOrgIsFull`** proves the seat-limit check and the
insert are atomic: the count did not change and the rejected user does not
exist. That is a *transaction* assertion made without a transaction engine.

**`TestRegisterIsAtomicAcrossTwoWrites`** injects a failure into the second
write by embedding the fake and overriding one method
([4.5](../04-types-structs/05-embedding.md)) — four lines to turn the fake into
a fault injector — and proves the first write rolled back.

**`TestRepositoryHonoursContextCancellation`** — not shown, but in the output —
cancels a context up front and asserts every method returns
`context.Canceled`. It is the one people skip, and it is the behaviour 16.12
said nothing would warn you about.

Every assertion uses `errors.Is` against a sentinel, never a string compare, so
the tests survive any change to the message wording.

## Caching, as a decorator

Because the service depends on an interface, a cache is a new implementation
rather than a change to either side. Embedding the interface means the methods
you do not override pass straight through:

```go
import lru "github.com/hashicorp/golang-lru/v2"

type CachedUserRepository struct {
  UserRepository
  cache *lru.Cache[int64, User]
}

func (c *CachedUserRepository) GetUser(ctx context.Context, id int64) (User, error) {
  if u, ok := c.cache.Get(id); ok {
    return u, nil
  }
  u, err := c.UserRepository.GetUser(ctx, id)
  if err != nil {
    return User{}, err
  }
  c.cache.Add(id, u)
  return u, nil
}
```

`UpdateUserStatus` is the other half: call through, then `c.cache.Remove(id)`.
The same shape works for metrics, tracing and retries, which is why the
interface pays for itself even when there is only one real implementation.

One warning: a cached repository used **inside** a transaction will serve rows
the transaction cannot see, and will cache rows a rollback then discards. Bind
the cache to the pool-backed repository only, never to the one `WithTx`
creates.

## Quick Reference

| Piece | Where it lives |
|---|---|
| `User`, `ListFilter` | the service package |
| `UserRepository`, `TxManager` interfaces | the service package (**the consumer**) |
| `ErrUserNotFound`, `ErrDuplicateUser`… | the service package |
| `PostgresUserRepository`, `DBTX`, `mapUserError` | the `postgres` package |
| `withTx`, `PgxTxManager` | the `postgres` package |
| `InMemoryUserRepository` | the service package, or a `usertest` package |
| wiring | `main` / `cmd` (18.1) |

| Rule | Why |
|---|---|
| `ctx` first on every method | 16.12; cancellation must reach the driver |
| methods named for the domain | a `Query(sql)` passthrough decouples nothing |
| take a `DBTX`, not a pool | the same method works in a transaction |
| map SQLSTATE to domain errors here | nothing above imports pgx |
| the **service** owns transactions | atomicity is a business decision |
| check `RowsAffected() == 0` on UPDATE | a no-op UPDATE is not an error to Postgres |
| keyset, not OFFSET | constant time, and stable under writes |
| `= ANY($1)` over a loop | 1 query instead of N |
| return `rows.Err()` / use `CollectRows` | 17.1's silent truncation |

## Common Mistakes

**Declaring the interface in the database package.** The service then depends on
the database layer and the interface grows to whatever the driver can do; put
it with the consumer.

**A generic `Query(sql string)` method.** It leaks the driver into every caller
and cannot be faked; use domain-shaped methods.

**Beginning transactions inside repository methods.** Two operations can then
never be atomic, so the repository grows a method per use case; the service
owns the transaction.

**Taking a `*pgxpool.Pool` in the repository.** The method cannot then run in a
transaction; take a `DBTX`.

**Letting `pgx.ErrNoRows` or `*pgconn.PgError` escape.** Every layer above now
imports pgx; map to domain sentinels at this boundary.

**`OFFSET` pagination.** Latency grows with page depth and concurrent writes
skip and duplicate rows; use a keyset cursor.

**A loop of `GetX` calls.** That is an N+1; one `= ANY($1)` or a join.

**Not clamping `Limit`.** A client asks for a million rows and gets them.

**An in-memory fake that does not enforce the database's constraints.** Your
tests pass and production returns 409s; mirror the unique indexes, the
`NOT NULL`s and the foreign keys.

**A fake that iterates a map without sorting.** Order is random, so the tests
flake one run in ten.

## Exercises

1. Add `SoftDeleteUser` and make every read exclude soft-deleted rows, in both
   implementations, then write one test table that runs against **both** and
   asserts identical behaviour. Report any divergence you found.
2. Replace the single-column cursor with `(created_at, id)` so ties are stable,
   base64-encode it into an opaque string, and add tests proving that ten rows
   sharing one `created_at` paginate with no skips at `Limit: 3`.
3. Build the N+1 for real against the compose stack with 200 orgs, measure it,
   then replace it with `= ANY($1)` and with a join. Report all three timings
   and the bytes transferred.
4. Write a `MetricsUserRepository` decorator recording a Prometheus histogram
   per method, by embedding the interface. Prove with a test that all seven
   methods are observed and that a failing call is labelled as an error.
5. Extend the in-memory `WithTx` to be reentrant (a nested `WithTx` inside one
   behaves like a savepoint) and write tests for commit-inner/rollback-outer and
   rollback-inner/commit-outer. Compare your semantics with pgx's nested
   `tx.Begin` from 17.4.

---

[← 17.6 Migrations with golang-migrate](06-migrations.md) · [Course Index](../README.md) · [17.8 Testing Against a Real Database →](08-testing-db.md)
