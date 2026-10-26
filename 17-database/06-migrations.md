[← 17.5 Connection Pooling and Tuning](05-pooling.md) · [Course Index](../README.md) · [17.7 The Repository Layer →](07-repository.md)

# 17.6 — Migrations with golang-migrate

## What You'll Learn

- golang-migrate as a **library** with `//go:embed`, so migrations ship inside the binary
- The source driver parsing a **real** migration directory offline: the version list, and the genuine errors for a malformed filename, a duplicate version and a missing down file
- The `schema_migrations` table and the `dirty` flag — what sets it, why `Force` is the only recovery, and how to avoid needing it
- Expand/contract for a zero-downtime column rename, and the `CREATE INDEX CONCURRENTLY` gotcha

The source-driver parsing here is real, run against a real three-pair
migrations directory. The `migrate` CLI is not installed in this environment
and nothing needs a server to apply DDL, so those blocks are labelled.

## Coming From Another Language

| Other | Go |
|---|---|
| `knex migrate:latest`, `prisma migrate deploy` | a Go binary you build, or the `migrate` CLI |
| Alembic autogenerating from models | **you write the SQL**; nothing is generated |
| Rails `rake db:migrate` reading `db/migrate/` | `//go:embed migrations/*.sql` — in the binary |
| a framework running migrations on boot | a deliberate choice; the recommendation is a separate step |
| `.rollback()` on a migration object | a separate `.down.sql` file you write by hand |
| a migrations table with a row per migration | one row holding the **current version** and a `dirty` flag |

## File naming

```
migrations/
  000001_create_users.up.sql
  000001_create_users.down.sql
  000002_create_orgs.up.sql
  000002_create_orgs.down.sql
  000003_add_users_status.up.sql
  000003_add_users_status.down.sql
```

The convention is `{version}_{name}.{up|down}.sql`. The version is an unsigned
integer, must be **unique**, and determines the order. Two schemes are in use:
a zero-padded sequence (`000001`) and a UTC timestamp
(`20260301120000`). Sequence numbers read better and collide in pull requests;
timestamps never collide and sort fine. For a team, use timestamps; for a
tutorial, sequences. Either way, **never renumber a migration that has been
applied anywhere** — the version in `schema_migrations` is the only record of
what ran.

## The real migrations

Three real pairs, used for every output in this lesson.

`000001_create_users.up.sql`:

```sql
CREATE TABLE users (
  id         bigserial PRIMARY KEY,
  org_id     bigint      NOT NULL,
  email      text        NOT NULL,
  name       text        NOT NULL,
  bio        text,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX users_email_key ON users (lower(email));
CREATE INDEX users_org_id_idx ON users (org_id);
```

`000001_create_users.down.sql`:

```sql
DROP TABLE users;
```

`000002_create_orgs.up.sql`:

```sql
CREATE TABLE orgs (
  id   bigserial PRIMARY KEY,
  name text NOT NULL
);

ALTER TABLE users
  ADD CONSTRAINT users_org_id_fkey FOREIGN KEY (org_id) REFERENCES orgs (id);
```

`000002_create_orgs.down.sql`:

```sql
ALTER TABLE users DROP CONSTRAINT users_org_id_fkey;
DROP TABLE orgs;
```

`000003_add_users_status.up.sql`:

```sql
CREATE TYPE user_status AS ENUM ('pending', 'active', 'suspended');

ALTER TABLE users
  ADD COLUMN status user_status NOT NULL DEFAULT 'pending';
```

`000003_add_users_status.down.sql`:

```sql
ALTER TABLE users DROP COLUMN status;
DROP TYPE user_status;
```

Note the unique index on `lower(email)` rather than on `email`. That is what
makes `GetUserByEmail` case-insensitive in 17.7, and it is the constraint whose
name — `users_email_key` — 17.4's error mapper branches on.

## The source driver, offline

The source driver reads and orders migrations with no database involved, which
means your migration directory is **testable**:

```go
package main

import (
  "embed"
  "fmt"
  "io"
  "testing/fstest"

  "github.com/golang-migrate/migrate/v4/source"
  "github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

func walk(label string, d source.Driver, err error) {
  fmt.Printf("=== %s ===\n", label)
  if err != nil {
    fmt.Printf("  open error: %v\n\n", err)
    return
  }
  v, err := d.First()
  if err != nil {
    fmt.Printf("  First() error: %v\n\n", err)
    return
  }
  for {
    up, upName, upErr := d.ReadUp(v)
    down, downName, downErr := d.ReadDown(v)
    upLen, downLen := -1, -1
    if upErr == nil {
      b, _ := io.ReadAll(up)
      up.Close()
      upLen = len(b)
    }
    if downErr == nil {
      b, _ := io.ReadAll(down)
      down.Close()
      downLen = len(b)
    }
    fmt.Printf("  v=%-3d up=%-18s (%3d bytes)  down=%-18s (%3d bytes)\n",
      v, upName, upLen, downName, downLen)
    if downErr != nil {
      fmt.Printf("        ReadDown error: %v\n", downErr)
    }
    next, err := d.Next(v)
    if err != nil {
      fmt.Printf("  Next(%d): %v  (end of list)\n", v, err)
      break
    }
    v = next
  }
  fmt.Println()
}

func main() {
  d, err := iofs.New(migrationsFS, "migrations")
  walk("iofs + embed.FS: the real migrations directory", d, err)

  fmt.Println("=== source.Parse on individual filenames ===")
  for _, name := range []string{
    "000001_create_users.up.sql",
    "2_add_index.down.sql",
    "20260301120000_create_users.up.sql",
    "000001_create_users.up.SQL",
    "create_users.up.sql",
    "000004_no_direction.sql",
    "000005_bad.sideways.sql",
  } {
    m, err := source.Parse(name)
    if err != nil {
      fmt.Printf("  %-36s -> error: %v\n", name, err)
      continue
    }
    fmt.Printf("  %-36s -> Version=%d Identifier=%q Direction=%q\n",
      name, m.Version, m.Identifier, m.Direction)
  }

  fmt.Println("\n=== two files claiming the same version ===")
  dup := fstest.MapFS{
    "m/000001_create_users.up.sql":   {Data: []byte("CREATE TABLE users ();")},
    "m/000001_create_users.down.sql": {Data: []byte("DROP TABLE users;")},
    "m/000001_create_orgs.up.sql":    {Data: []byte("CREATE TABLE orgs ();")},
    "m/000001_create_orgs.down.sql":  {Data: []byte("DROP TABLE orgs;")},
  }
  _, err = iofs.New(dup, "m")
  fmt.Printf("  iofs.New error: %v\n\n", err)

  missing := fstest.MapFS{
    "m/000001_create_users.up.sql":   {Data: []byte("CREATE TABLE users ();")},
    "m/000001_create_users.down.sql": {Data: []byte("DROP TABLE users;")},
    "m/000002_add_bio.up.sql":        {Data: []byte("ALTER TABLE users ADD bio text;")},
  }
  md, err := iofs.New(missing, "m")
  walk("a version with no down file", md, err)

  ed, err := iofs.New(fstest.MapFS{"m/.keep": {Data: nil}}, "m")
  walk("an empty directory", ed, err)

  fmt.Printf("=== registered source drivers === %v\n", source.List())
}
```

Output:

```
=== iofs + embed.FS: the real migrations directory ===
  v=1   up=create_users       (343 bytes)  down=create_users       ( 18 bytes)
  v=2   up=create_orgs        (171 bytes)  down=create_orgs        ( 70 bytes)
  v=3   up=add_users_status   (147 bytes)  down=add_users_status   ( 61 bytes)
  Next(3): next for version 3 migrations: file does not exist  (end of list)

=== source.Parse on individual filenames ===
  000001_create_users.up.sql           -> Version=1 Identifier="create_users" Direction="up"
  2_add_index.down.sql                 -> Version=2 Identifier="add_index" Direction="down"
  20260301120000_create_users.up.sql   -> Version=20260301120000 Identifier="create_users" Direction="up"
  000001_create_users.up.SQL           -> Version=1 Identifier="create_users" Direction="up"
  create_users.up.sql                  -> error: no match
  000004_no_direction.sql              -> error: no match
  000005_bad.sideways.sql              -> error: no match

=== two files claiming the same version ===
  iofs.New error: failed to init driver with path m: duplicate migration file: 000001_create_users.down.sql

=== a version with no down file ===
  v=1   up=create_users       ( 22 bytes)  down=create_users       ( 17 bytes)
  v=2   up=add_bio            ( 31 bytes)  down=                   ( -1 bytes)
        ReadDown error: read down for version 2 m: file does not exist
  Next(2): next for version 2 m: file does not exist  (end of list)

=== an empty directory ===
  First() error: first m: file does not exist

=== registered source drivers === []
```

Six findings from that, all of them things you would otherwise discover during a
deploy.

**The zero padding is cosmetic.** `000001` and `2_add_index` both parse;
`Version` is 1 and 2. Padding only makes `ls` sort correctly.

**The extension is case-insensitive but the structure is not.**
`...up.SQL` parsed fine. `create_users.up.sql` (no version),
`000004_no_direction.sql` (no direction) and `000005_bad.sideways.sql` all
failed with `no match`, which is `source.ErrParse`. A file that does not parse
is **silently skipped** by the directory scan rather than being an error — so a
typo like `000004_add_index.sql` means that migration never runs and nothing
tells you. That is the single best argument for the test at the end of this
section.

**A duplicate version is a hard error at open time.** `iofs.New` refused to
construct the driver at all: `duplicate migration file`. This is what catches
two developers both numbering their migration `000007`, and it fails at startup
rather than halfway through a deploy.

**A missing down file is not an error until you need it.** Version 2 listed
fine and `ReadUp` worked; only `ReadDown` failed. So a migration with no down
file deploys happily and then cannot be rolled back — discovered at the worst
possible moment. Test for it.

**An empty directory fails at `First()`,** not at `New`. A broken embed pattern
(or a `migrations/` directory that did not get copied into your container) looks
like this.

**`source.List()` is empty** because this program imported only `iofs`, which
deliberately does not register a URL scheme. Importing
`_ "github.com/golang-migrate/migrate/v4/source/file"` adds `file`.

### Testing your migration directory

Everything above is a test you can run in CI without a database:

```go
package migrations_test

import (
  "testing"

  "github.com/golang-migrate/migrate/v4/source"

  "example.com/shop/internal/migrations"
)

func TestEveryFileParses(t *testing.T) {
  entries, err := migrations.FS.ReadDir("migrations")
  if err != nil {
    t.Fatal(err)
  }
  for _, e := range entries {
    if _, err := source.Parse(e.Name()); err != nil {
      t.Errorf("%s does not match {version}_{name}.{up|down}.sql: %v", e.Name(), err)
    }
  }
  if len(entries)%2 != 0 {
    t.Errorf("%d files is odd; every version needs an up and a down", len(entries))
  }
}
```

A companion test walks the driver with `First`/`Next`, asserting each version
is greater than the last and that both `ReadUp` and `ReadDown` succeed for
every one — which is what catches the missing down file.

`TestEveryFileParses` is the one that earns its keep: it catches the silently
skipped typo that the directory scan ignores. Run both in CI
([18.9](../18-production/09-ci.md)) and a malformed migration never reaches a
deploy.

## Embedding migrations in the binary

This is the production choice. One artifact contains the code and the schema it
expects, so they cannot drift and there is no "did the migrations directory get
into the image" question
([13.13](../13-stdlib/13-embed.md), [18.8](../18-production/08-docker.md)).

```go
package migrations

import "embed"

//go:embed migrations/*.sql
var FS embed.FS
```

Two `//go:embed` rules that bite here. The directive must be **immediately
above** the variable with no blank line, and the paths are relative to the
**package directory** — so `migrations/*.sql` requires a `migrations/`
subdirectory next to the `.go` file. The pattern also silently matches nothing
rather than erroring if the directory is empty, which is why the empty-directory
case above is worth a test.

The alternative is the `file://` driver, which reads from disk at runtime:

```go
import _ "github.com/golang-migrate/migrate/v4/source/file"

m, err := migrate.New("file://./migrations", dbURL)
```

Useful for local development and for the CLI. In a container it means shipping
the SQL files alongside the binary and getting the relative path right, which
is one more thing to get wrong.

## The migrator

```go
package main

import (
  "embed"
  "errors"
  "fmt"
  "log/slog"
  "os"

  "github.com/golang-migrate/migrate/v4"
  _ "github.com/golang-migrate/migrate/v4/database/pgx/v5"
  "github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

var log = slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
  ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
    if a.Key == slog.TimeKey {
      return slog.Attr{}
    }
    return a
  },
}))

func newMigrator(dbURL string) (*migrate.Migrate, error) {
  src, err := iofs.New(migrationsFS, "migrations")
  if err != nil {
    return nil, fmt.Errorf("loading embedded migrations: %w", err)
  }
  m, err := migrate.NewWithSourceInstance("iofs", src, dbURL)
  if err != nil {
    return nil, fmt.Errorf("opening migrator: %w", err)
  }
  return m, nil
}

func migrateUp(dbURL string) error {
  m, err := newMigrator(dbURL)
  if err != nil {
    return err
  }
  defer func() {
    if srcErr, dbErr := m.Close(); srcErr != nil || dbErr != nil {
      log.Error("closing migrator", "source_err", srcErr, "db_err", dbErr)
    }
  }()

  before, dirty, err := m.Version()
  switch {
  case errors.Is(err, migrate.ErrNilVersion):
    log.Info("no migrations applied yet")
  case err != nil:
    return fmt.Errorf("reading version: %w", err)
  default:
    log.Info("current schema", "version", before, "dirty", dirty)
  }
  if dirty {
    return fmt.Errorf("schema is dirty at version %d: inspect it, then Force(%d)", before, before)
  }

  switch err := m.Up(); {
  case errors.Is(err, migrate.ErrNoChange):
    log.Info("schema already up to date")
  case err != nil:
    return fmt.Errorf("applying migrations: %w", err)
  }

  after, _, err := m.Version()
  if err != nil {
    return fmt.Errorf("reading version: %w", err)
  }
  log.Info("migrations applied", "version", after)
  return nil
}

func main() {
  if err := migrateUp("pgx5://app:s3cret@localhost:5432/shop?sslmode=disable"); err != nil {
    log.Error("migrate failed", "err", err)
  }
}
```

Run with no server, this produces a genuine error:

```
level=ERROR msg="migrate failed" err="opening migrator: failed to open database: failed to connect to `user=app database=shop`:\n\t[::1]:5432 (localhost): dial error: dial tcp [::1]:5432: connect: connection refused\n\t127.0.0.1:5432 (localhost): dial error: dial tcp 127.0.0.1:5432: connect: connection refused"
```

The down, goto and force variants are the same shape over `newMigrator`:
`m.Steps(-1)`, `m.Migrate(version)` and `m.Force(version)`, each tolerating
`migrate.ErrNoChange`.

That output is worth a note, because it contradicts what 17.2 established about
pgx. **`migrate.NewWithSourceInstance` connects eagerly**, unlike
`pgxpool.New`. The error arrived at construction, not at `Up()`. Good: a
migration tool that cannot reach the database should fail immediately.

Also note the URL scheme is **`pgx5://`**, not `postgres://`. golang-migrate
selects its database driver from the scheme, and the pgx v5 driver registers
`pgx5`. Using `postgres://` picks the older `lib/pq`-based driver, which is also
bundled and works, but then you have two Postgres drivers in your binary.
`pgx5://` is the one to use.

| Method | Does |
|---|---|
| `m.Up()` | apply every pending migration; `ErrNoChange` if none |
| `m.Down()` | roll back **everything**; almost never what you want |
| `m.Steps(n)` | `n` migrations up, or `-n` down |
| `m.Migrate(v)` | go to exactly version `v`, up or down |
| `m.Version()` | `(version uint, dirty bool, err error)`; `ErrNilVersion` if none applied |
| `m.Force(v)` | set the version and **clear `dirty`**, running nothing |
| `m.Drop()` | delete every table in the database; for tests only |
| `m.Close()` | returns **two** errors: `(sourceErr, dbErr)` |

`m.Close()` returning two errors is easy to get wrong — `defer m.Close()`
compiles and discards both.

| Sentinel | `Error()` text | Means |
|---|---|---|
| `migrate.ErrNoChange` | `no change` | already at the target version; **not** a failure |
| `migrate.ErrNilVersion` | `no migration` | `schema_migrations` is empty |
| `migrate.ErrLocked` | `database locked` | another migrator holds the advisory lock |
| `migrate.ErrLockTimeout` | `timeout: can't acquire database lock` | waited out `DefaultLockTimeout` (15s) |
| `migrate.ErrInvalidVersion` | `version must be >= -1` | bad argument to `Force` |
| `migrate.ErrShortLimit` | — | `Steps(n)` ran out of migrations |
| `migrate.ErrDirty` | `Dirty database version 3. Fix and force version.` | carries `Version` |

Those strings are the real `Error()` values on v4.20.1.

Always treat `ErrNoChange` as success. Code that returns it as an error makes
every redeploy fail.

## The schema_migrations table and the dirty flag

golang-migrate keeps exactly one row:

```sql
CREATE TABLE schema_migrations (
  version bigint  NOT NULL PRIMARY KEY,
  dirty   boolean NOT NULL
);
```

One row, not one per migration. `version` is the **current** version and
`dirty` says whether the last attempt completed.

The sequence for each migration is:

1. set `version = N, dirty = true`
2. run `000N_*.up.sql`
3. set `dirty = false`

So `dirty = true` means **step 2 did not finish**: the process was killed, the
container was OOM-killed, the deploy timed out, the connection dropped, or the
SQL failed partway through a multi-statement file.

And here is the important part: **golang-migrate refuses to do anything while
dirty.** Every subsequent `Up` fails with `*migrate.ErrDirty`. This is correct
and deliberate — the tool does not know how much of migration N applied, so
re-running it could fail on the half that succeeded, and skipping it would
leave the schema wrong. It stops and makes a human decide.

Expected output (requires a running Postgres; not executed in this environment):

```
error: Dirty database version 3. Fix and force version.
```

### Recovering

**`Force` is the recovery, and it only sets the version number.** It runs no
SQL. So the procedure is:

1. **Look at the actual schema.** `\d users`, `\d+ schema_migrations`. Compare
   it to what `000003_*.up.sql` was supposed to do.
2. **Decide which version the schema really is at.** Either migration 3 fully
   applied (the process died between step 2 and step 3), or it partly applied.
3. **Make the schema match a clean version by hand.** If 3 partly applied,
   either finish it manually or undo the part that landed.
4. **Then** `m.Force(3)` if the schema now matches version 3, or `m.Force(2)` if
   you undid it.
5. Re-run `Up`.

```go
func recoverDirty(dbURL string) error {
  m, err := newMigrator(dbURL)
  if err != nil {
    return err
  }
  defer m.Close()

  v, dirty, err := m.Version()
  if err != nil {
    return err
  }
  if !dirty {
    log.Info("not dirty, nothing to do", "version", v)
    return nil
  }
  log.Warn("schema is dirty; inspect it before forcing", "version", v)
  return fmt.Errorf("manual intervention required at version %d", v)
}
```

Note what that function does **not** do: it does not call `Force`
automatically. An automated `Force` is how a half-applied migration becomes a
permanently wrong schema that every later migration builds on. `Force` belongs
in a runbook, run by a human who has looked at the schema.

### How to not need it

**Lean on Postgres's implicit transaction.** golang-migrate's Postgres driver
sends the whole `.up.sql` file as **one** `Exec`, and Postgres wraps a
multi-statement simple-protocol query in a single implicit transaction. So a
statement that *fails* rolls the whole file back and the schema stays clean —
`dirty` is still set, but the schema matches the previous version, which makes
the recovery easy. The dirty flag with a genuinely half-applied schema comes
from the **process dying mid-file**. The mitigations:

- **One logical change per migration file.** A file with eight `ALTER`s is eight
  times more likely to be interrupted midway.
- **No long-running statements in a migration.** A `CREATE INDEX` on a large
  table, or a backfill `UPDATE` over a million rows, is exactly what a deploy
  timeout kills. Backfills belong in a separate job, not a migration.
- **Generous timeouts on the migration step.** Longer than your longest
  migration, by a lot.
- **Never two migrators at once.** golang-migrate takes a Postgres advisory
  lock, so the second one gets `ErrLocked` rather than corrupting anything —
  but a `Force` run concurrently with an `Up` is unprotected.
- **Do not enable `x-multi-statement`.** That DSN option splits the file and
  runs each statement in its own `Exec`, which throws away the implicit
  transaction and makes a half-applied file possible on a plain SQL error. You
  only need it for statements that genuinely cannot share a transaction.

## Startup versus a separate deploy step

Running migrations from your application's `main` at startup is tempting: one
artifact, no extra pipeline step, impossible to forget. It is also wrong at
more than one replica, and the reasons compound. With *N* replicas rolling out
together, *N* processes race to migrate; the advisory lock means one wins and
the rest block, so your rollout serialises behind the slowest migration and a
readiness probe may kill a pod that is legitimately waiting. Worse, the *old*
code is still serving traffic while the new schema lands, so a migration that
drops a column breaks the running version — and if the migration fails, every
replica crash-loops instead of one job failing loudly with a log you can read.
Separating it gives you one place to look when it breaks, the ability to run it
and *stop* before deploying code, and a schema change you can roll out ahead of
the code that needs it.

**Recommendation: a separate step that runs to completion before any new code
starts.** A Kubernetes `Job` with the same image and a different entrypoint, a
`release` phase, or a CI step. One process, one exit code.

```go
func main() {
  migrateOnly := flag.Bool("migrate", false, "apply migrations and exit")
  flag.Parse()

  dbURL := os.Getenv("DATABASE_URL")
  if *migrateOnly {
    if err := migrateUp(dbURL); err != nil {
      log.Error("migration failed", "err", err)
      os.Exit(1)
    }
    return
  }
  // ... normal service startup
}
```

One binary, two modes. The deploy runs `./shop -migrate` as a job, waits for
exit 0, then rolls out `./shop`.

The one case where startup migrations are fine: a single-replica service, or a
local dev stack, or integration tests (17.8 does exactly this). If you do it,
guard it so only one replica tries:

```go
if os.Getenv("RUN_MIGRATIONS") == "true" {
  if err := migrateUp(dbURL); err != nil {
    return fmt.Errorf("migrating: %w", err)
  }
}
```

Expected output (requires a running Postgres; not executed in this environment):

```
level=INFO msg="no migrations applied yet"
level=INFO msg="migrations applied" version=3
```

And a second run:

```
level=INFO msg="current schema" version=3 dirty=false
level=INFO msg="schema already up to date"
level=INFO msg="migrations applied" version=3
```

## The CLI

Not installed in this environment, so the install command is real and the usage
below is the documented form rather than a captured run:

```bash
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@v4.20.1
```

```bash
export POSTGRESQL_URL='pgx5://app:s3cret@localhost:5432/shop?sslmode=disable'

migrate create -ext sql -dir migrations -seq add_users_last_login

migrate -database "$POSTGRESQL_URL" -path migrations up
migrate -database "$POSTGRESQL_URL" -path migrations down 1
migrate -database "$POSTGRESQL_URL" -path migrations goto 2
migrate -database "$POSTGRESQL_URL" -path migrations version
migrate -database "$POSTGRESQL_URL" -path migrations force 3
```

`migrate create -seq` is genuinely useful — it writes both files with the next
sequence number, so you cannot accidentally reuse a version. Use
`-format 20060102150405` instead of `-seq` for timestamps.

The CLI and the library read the same files and the same `schema_migrations`
table, so using the CLI locally and the library in production is a normal
arrangement.

## Writing reversible migrations

A down migration should return the schema to the previous version. Most can:

| Up | Down | Reversible? |
|---|---|---|
| `CREATE TABLE t` | `DROP TABLE t` | yes |
| `ADD COLUMN c` | `DROP COLUMN c` | **schema yes, data no** |
| `DROP COLUMN c` | `ADD COLUMN c` | **no — the data is gone** |
| `CREATE INDEX i` | `DROP INDEX i` | yes |
| `ADD CONSTRAINT k` | `DROP CONSTRAINT k` | yes |
| `ALTER COLUMN ... TYPE text` | back to `varchar(50)` | **no — long values truncate** |
| `RENAME COLUMN a TO b` | rename back | yes |
| `CREATE TYPE ... AS ENUM` | `DROP TYPE` | yes |
| adding an enum value | **nothing** | **no — Postgres cannot remove one** |
| `UPDATE` (a backfill) | no honest inverse | **no** |

The two that catch people: **dropping a column deletes its data permanently**,
so the down migration recreates an empty column and the rows are gone; and
**Postgres cannot remove an enum value**, so `ALTER TYPE ... ADD VALUE` is
one-way (which is a good argument for a `text` column with a `CHECK` instead of
an enum).

Write the down file anyway, and say so when it is lossy:

```sql
-- 000007_drop_users_legacy_id.down.sql
-- Not a true rollback: the legacy_id values cannot be recovered.
ALTER TABLE users ADD COLUMN legacy_id bigint;
```

That is a comment the course's style rules permit, because the file cannot
carry the warning any other way and a future reader would otherwise assume the
rollback is complete.

In production, the honest position is that **you roll forward, not back.** A
down migration that drops a column you have been writing to for three hours
destroys three hours of data. Downs are for local development and for CI,
where recreating the database is free. If a migration was wrong, write
migration N+1 that fixes it.

## Expand and contract

The pattern for a change that cannot be done atomically, because old and new
code must both work during the rollout. Renaming `users.name` to
`users.full_name`, in three deploys:

**Deploy 1 — expand.** Add the new column; write to both, read from the old.

```sql
-- 000010_add_users_full_name.up.sql
ALTER TABLE users ADD COLUMN full_name text;
```

```sql
-- 000010_add_users_full_name.down.sql
ALTER TABLE users DROP COLUMN full_name;
```

```go
_, err := db.Exec(ctx,
  `INSERT INTO users (email, name, full_name) VALUES ($1, $2, $2)`, email, name)
```

Then backfill, in a **separate job** with batching so it does not lock the
table or time out the deploy:

```sql
UPDATE users SET full_name = name
WHERE full_name IS NULL AND id IN (
  SELECT id FROM users WHERE full_name IS NULL ORDER BY id LIMIT 10000
);
```

Loop that until it affects zero rows. A single unbatched `UPDATE` over a large
table holds row locks for its whole duration and bloats the table.

**Deploy 2 — migrate reads.** Every replica is now on code that writes both
columns, so switch reads to `full_name` and add the constraint:

```sql
-- 000011_users_full_name_not_null.up.sql
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;
```

Add `NOT NULL` only after the backfill is confirmed complete — on a big table,
prefer a `CHECK (full_name IS NOT NULL) NOT VALID` followed by
`VALIDATE CONSTRAINT`, which does not need a full-table `ACCESS EXCLUSIVE` lock.

**Deploy 3 — contract.** Nothing reads or writes `name` any more, so drop it:

```sql
-- 000012_drop_users_name.up.sql
ALTER TABLE users DROP COLUMN name;
```

Three deploys, each one safe with both versions of the code running. The rule
that generates the pattern: **a schema change and the code change that needs it
must never be in the same deploy.** Additive changes (new nullable column, new
index, new table) go out *before* the code; destructive ones (drop column, drop
table, tighten a constraint) go out *after*.

| Change | Safe? | Note |
|---|---|---|
| `ADD COLUMN` nullable, no default | **safe** | metadata only |
| `ADD COLUMN` with a constant default | **safe** (PG 11+) | no table rewrite |
| `ADD COLUMN` with a volatile default | **rewrites the table** | avoid |
| `ADD COLUMN NOT NULL` with no default | **fails** if rows exist | expand/contract |
| `DROP COLUMN` | safe but **destructive** | contract phase only |
| `CREATE INDEX` | **locks writes** | use `CONCURRENTLY` |
| `ALTER COLUMN TYPE` | usually **rewrites** | some widenings are free |
| `ADD CONSTRAINT ... NOT VALID` | safe | then `VALIDATE CONSTRAINT` |
| `ADD FOREIGN KEY` | locks both tables | `NOT VALID` then validate |
| `RENAME COLUMN` | instant, but **breaks old code** | expand/contract |
| `ALTER TYPE ... ADD VALUE` | safe, **irreversible** | cannot be removed |

## Transactional DDL, and CREATE INDEX CONCURRENTLY

Postgres can do DDL inside a transaction, which is genuinely unusual — MySQL
cannot. golang-migrate's pgx driver does not issue an explicit `BEGIN`; it
sends the file as one query and relies on Postgres's **implicit** transaction
around a multi-statement simple query. The effect is the same: a failing
statement rolls the whole file back.

The exception that will bite you:

```sql
CREATE INDEX CONCURRENTLY users_email_idx ON users (lower(email));
```

`CONCURRENTLY` builds the index without taking a write lock, which is how you
add an index to a live table. It **cannot run inside a transaction block**:

Expected output (requires a running Postgres; not executed in this environment):

```
ERROR: CREATE INDEX CONCURRENTLY cannot run inside a transaction block (SQLSTATE 25001)
```

golang-migrate has **no per-file "no transaction" directive** — that is goose
and dbmate syntax, and writing `-- migrate:no-transaction` in a golang-migrate
file does nothing but add a comment. What golang-migrate gives you instead is
the implicit transaction being *per query*, so a file containing exactly one
statement is already running outside any multi-statement block and
`CONCURRENTLY` works:

```sql
-- 000013_add_users_email_idx.up.sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS users_email_idx ON users (lower(email));
```

```sql
-- 000013_add_users_email_idx.down.sql
DROP INDEX CONCURRENTLY IF EXISTS users_email_idx;
```

**One statement per file is therefore mandatory for `CONCURRENTLY`**, not just
advisable: add a second statement and Postgres opens an implicit transaction
around both, and you get the `25001` error. Keep these migrations to a single
line, with no trailing semicolon-separated extras.

Three consequences of running outside a transaction, all of which matter:

**A failure leaves `dirty = true` with no rollback.** There is no transaction
to undo, which is exactly the dirty-flag scenario — another reason these files
hold one statement.

**A failed `CREATE INDEX CONCURRENTLY` leaves an invalid index behind.** It is
not rolled back, it does not work, and it still costs write throughput. You
must drop it by hand before retrying:

```sql
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
DROP INDEX CONCURRENTLY users_email_idx;
```

The `IF NOT EXISTS` above does **not** save you, because an invalid index
exists.

**It takes a long time on a large table** and does two table scans. This is the
statement most likely to be killed by a deploy timeout, which is the best
argument for running index creation as its own deploy step.

The same applies to `ALTER TYPE ... ADD VALUE` before PG 12, `DROP INDEX
CONCURRENTLY`, `REINDEX CONCURRENTLY`, and `VACUUM`: one statement, its own
file.

## Seeding data

Reference data that the application requires — roles, plan tiers, country codes
— belongs in a migration, written idempotently:

```sql
-- 000004_seed_plans.up.sql
INSERT INTO plans (code, name, seat_limit) VALUES
  ('free', 'Free',       3),
  ('team', 'Team',      25),
  ('ent',  'Enterprise', 0)
ON CONFLICT (code) DO UPDATE
  SET name = EXCLUDED.name, seat_limit = EXCLUDED.seat_limit;
```

`ON CONFLICT DO UPDATE` makes it re-runnable, which matters when the migration
is re-applied after a `Force`.

Development and test *fixtures* do not belong in migrations — they would run in
production. Keep those in a separate seed command, or in test helpers (17.8).

## Alternatives, in one line

[goose](https://github.com/pressly/goose) supports Go migrations as well as SQL
and embeds just as well; [atlas](https://atlasgo.io) does declarative schema
diffing and can generate migrations; [tern](https://github.com/jackc/tern) is
by pgx's author and is pleasantly minimal. golang-migrate is the course default
for being the most widely deployed.

## Quick Reference

| Call | Note |
|---|---|
| `iofs.New(fsys, "migrations")` | `embed.FS` source; the production choice |
| `source.Open("file://./migrations")` | disk source; needs the `file` driver import |
| `migrate.New(sourceURL, dbURL)` | both from URLs |
| `migrate.NewWithSourceInstance(name, src, dbURL)` | embedded source, URL database |
| `migrate.NewWithDatabaseInstance(srcURL, name, drv)` | reuse an existing `*sql.DB` |
| `migrate.NewWithInstance(...)` | both as instances |
| `source.Parse(filename)` | offline filename validation |
| `m.Close()` | returns **two** errors |

| URL scheme | Driver |
|---|---|
| `pgx5://` | pgx v5 — **use this** |
| `pgx://` | pgx v4 |
| `postgres://`, `postgresql://` | lib/pq |

| `schema_migrations` | |
|---|---|
| `version` | the current version; **one row only** |
| `dirty` | the last attempt did not complete |
| `Force(v)` | sets `version` and clears `dirty`; **runs no SQL** |

## Common Mistakes

**Treating `migrate.ErrNoChange` as a failure.** Every redeploy then fails; it
means "already up to date".

**Running migrations from every replica at startup.** They race on the advisory
lock, your rollout serialises, and a failure crash-loops every pod; use a
separate job.

**Calling `Force` without inspecting the schema.** You pin a version number to a
half-applied schema and every later migration builds on the wrong base.

**Editing a migration that has already been applied.** The version in
`schema_migrations` is unchanged, so your environments silently diverge; write
a new migration.

**Reusing a version number.** `iofs.New` refuses to open the directory at all,
which at least fails loudly.

**A filename that does not parse.** It is silently skipped, so the migration
never runs; test every filename with `source.Parse`.

**Omitting the down file.** It deploys fine and fails only when you need to roll
back; test that every version has both.

**`CREATE INDEX` without `CONCURRENTLY` on a live table.** It blocks writes for
the duration; add `CONCURRENTLY` plus `-- migrate:no-transaction`.

**Backfilling in a migration.** A million-row `UPDATE` holds locks and gets
killed by the deploy timeout; batch it in a separate job.

**`defer m.Close()` ignoring both return values.** It returns
`(sourceErr, dbErr)` and discards them.

## Exercises

1. Add the two tests from this lesson to a real migrations package and
   deliberately break the directory three ways — a duplicate version, a
   filename typo, a missing down file. Show each test catching its case with
   the real failure output.
2. Write the three-deploy expand/contract sequence for renaming `users.bio` to
   `users.about`, including a batched backfill loop in Go that reports progress
   and is safe to re-run. Prove with the compose stack that old-shape reads
   keep working at every step.
3. Simulate the dirty flag: start a migration whose up file contains a
   `pg_sleep(30)` followed by an `ALTER`, kill the process mid-flight, then
   capture the real `ErrDirty`. Write the recovery runbook you would actually
   follow and execute it.
4. Add a `CREATE INDEX CONCURRENTLY` migration without the
   `-- migrate:no-transaction` comment and capture the real `25001` error. Fix
   it, then force a failure mid-build and show the invalid index in `pg_index`.
5. Build a `-migrate` flag into a service binary, plus a `docker compose`
   service that runs it as a one-shot job with `depends_on` the database's
   healthcheck. Show the job exiting 0, the service starting after it, and the
   job exiting 1 on a deliberately broken migration.

---

[← 17.5 Connection Pooling and Tuning](05-pooling.md) · [Course Index](../README.md) · [17.7 The Repository Layer →](07-repository.md)
