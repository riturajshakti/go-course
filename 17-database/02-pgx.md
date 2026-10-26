[← 17.1 database/sql and the Driver Model](01-database-sql.md) · [Course Index](../README.md) · [17.3 Queries, Scanning, and NULL Handling →](03-queries-scanning.md)

# 17.2 — pgx: Connecting to Postgres

## What You'll Learn

- pgx's three usage modes and which one to pick, with a decision table
- `pgxpool.ParseConfig` on both DSN forms, with every parsed field and default printed for real
- How a connection failure actually surfaces: `pgxpool.New` succeeds, `Ping` fails — both captured, plus the real `context.DeadlineExceeded`
- The `docker compose` block that makes every labelled output in this module reproducible

Everything in this lesson except the Docker block and two labelled outputs was
executed against a machine with **no Postgres server running**, which turns out
to be most of it: config parsing and the failure paths are the interesting
parts.

## Coming From Another Language

| Other | Go |
|---|---|
| `new Pool(...)` then `await pool.query(...)` | `pgxpool.New` then `pool.Query(ctx, ...)` |
| `pool.connect()` returning a client you release | `pool.Acquire(ctx)` returning a conn you `Release()` |
| `psycopg2.connect()` opening a socket eagerly | `pgxpool.New` connects **lazily**; `Ping` is eager |
| node-postgres parsing `DATABASE_URL` for you | `pgxpool.ParseConfig` does, and you can inspect it |
| `sslmode` defaulting to off | pgx defaults to `prefer` — TLS if the server offers it |
| a pool size you forget to set | `MaxConns` defaults to `max(4, NumCPU)` (17.5) |

## The three modes

pgx is two libraries in one package tree: a native Postgres client, and a
`database/sql` driver built on top of it.

| Mode | Type | Concurrency | Use when |
|---|---|---|---|
| **`pgxpool`** (native) | `*pgxpool.Pool` | safe, pooled | **the default for a service** |
| `database/sql` adapter | `*sql.DB` via `_ ".../stdlib"` | safe, pooled | you need driver-agnostic code (17.1) |
| single connection | `*pgx.Conn` | **not** safe for concurrent use | migrations, CLI tools, `LISTEN`, `COPY` sessions |

```go
import (
  "github.com/jackc/pgx/v5"
  "github.com/jackc/pgx/v5/pgxpool"
  _ "github.com/jackc/pgx/v5/stdlib"
)

pool, err := pgxpool.New(ctx, dsn) // 1. native pool  <- use this
db, err := sql.Open("pgx", dsn)    // 2. database/sql
conn, err := pgx.Connect(ctx, dsn) // 3. one connection
```

The one rule that matters: **`*pgx.Conn` is not safe for concurrent use.** It
is a single protocol stream. Two goroutines querying one `*pgx.Conn` interleave
their messages and corrupt the connection. If you find yourself adding a mutex
around a `*pgx.Conn`, you have reimplemented `pgxpool` badly.

The versions used throughout this module:

```
github.com/jackc/pgx/v5 v5.11.0
github.com/golang-migrate/migrate/v4 v4.20.1
```

## DSN forms, parsed for real

pgx accepts libpq's URL form and its keyword/value form, and `ParseConfig`
turns either into an inspectable struct. This runs with no server:

```go
package main

import (
  "fmt"
  "runtime"

  "github.com/jackc/pgx/v5/pgxpool"
)

func describe(label, dsn string) {
  cfg, err := pgxpool.ParseConfig(dsn)
  if err != nil {
    fmt.Printf("%s\n  ParseConfig error: %v\n\n", label, err)
    return
  }
  cc := cfg.ConnConfig
  fmt.Printf("%s\n", label)
  fmt.Printf("  Host=%q Port=%d Database=%q User=%q Password=%q\n",
    cc.Host, cc.Port, cc.Database, cc.User, cc.Password)
  fmt.Printf("  TLSConfig==nil=%v  ConnectTimeout=%v  RuntimeParams=%v\n",
    cc.TLSConfig == nil, cc.ConnectTimeout, cc.RuntimeParams)
  fmt.Printf("  MaxConns=%d MinConns=%d MinIdleConns=%d\n",
    cfg.MaxConns, cfg.MinConns, cfg.MinIdleConns)
  fmt.Printf("  MaxConnLifetime=%v MaxConnLifetimeJitter=%v MaxConnIdleTime=%v HealthCheckPeriod=%v\n\n",
    cfg.MaxConnLifetime, cfg.MaxConnLifetimeJitter, cfg.MaxConnIdleTime, cfg.HealthCheckPeriod)
}

func main() {
  fmt.Printf("runtime.NumCPU() = %d\n\n", runtime.NumCPU())

  describe("1. URL form, minimal",
    "postgres://app:s3cret@db.internal:5432/shop?sslmode=disable")
  describe("2. URL form with pool params",
    "postgres://app@localhost/shop?sslmode=disable&pool_max_conns=20&pool_min_conns=2&pool_max_conn_lifetime=30m&pool_max_conn_idle_time=5m&pool_health_check_period=1m")
  describe("3. keyword/value form",
    "host=db.internal port=6432 user=app password=s3cret dbname=shop sslmode=disable connect_timeout=5")
  describe("4. malformed: bad scheme",
    "mysql://app:s3cret@localhost/shop")
  describe("5. malformed: non-numeric port",
    "postgres://app@localhost:notaport/shop")
  describe("6. malformed: bad pool param value",
    "postgres://app@localhost/shop?pool_max_conns=lots")
}
```

Output (on a 12-core machine; `MaxConns` tracks `NumCPU`):

```
runtime.NumCPU() = 12

1. URL form, minimal
  Host="db.internal" Port=5432 Database="shop" User="app" Password="s3cret"
  TLSConfig==nil=true  ConnectTimeout=0s  RuntimeParams=map[]
  MaxConns=12 MinConns=0 MinIdleConns=0
  MaxConnLifetime=1h0m0s MaxConnLifetimeJitter=0s MaxConnIdleTime=30m0s HealthCheckPeriod=1m0s

2. URL form with pool params
  Host="localhost" Port=5432 Database="shop" User="app" Password=""
  TLSConfig==nil=true  ConnectTimeout=0s  RuntimeParams=map[]
  MaxConns=20 MinConns=2 MinIdleConns=0
  MaxConnLifetime=30m0s MaxConnLifetimeJitter=0s MaxConnIdleTime=5m0s HealthCheckPeriod=1m0s

3. keyword/value form
  Host="db.internal" Port=6432 Database="shop" User="app" Password="s3cret"
  TLSConfig==nil=true  ConnectTimeout=5s  RuntimeParams=map[]
  MaxConns=12 MinConns=0 MinIdleConns=0
  MaxConnLifetime=1h0m0s MaxConnLifetimeJitter=0s MaxConnIdleTime=30m0s HealthCheckPeriod=1m0s

4. malformed: bad scheme
  ParseConfig error: cannot parse `mysql://app:xxxxxx@localhost/shop`: failed to parse as keyword/value (invalid keyword/value)

5. malformed: non-numeric port
  ParseConfig error: cannot parse `postgres://app@localhost:notaport/shop`: invalid port

6. malformed: bad pool param value
  ParseConfig error: cannot parse `postgres://app@localhost/shop?pool_max_conns=lots`: cannot parse pool_max_conns (strconv.ParseInt: parsing "lots": invalid syntax)
```

Several things worth reading off that.

**pgx redacts the password in its error messages.** Case 4's error shows
`mysql://app:xxxxxx@localhost/shop` — the password `s3cret` became `xxxxxx`
before the DSN went into an error. That matters, because connection errors end
up in logs, and [16.9](../16-http/09-error-responses.md) spent a whole lesson
on credentials leaking through error text. pgx is on your side here, but only
for the DSN it parsed; a DSN you format into your own error message yourself is
still your problem.

**Pool settings travel in the DSN.** `pool_max_conns`, `pool_min_conns`,
`pool_max_conn_lifetime`, `pool_max_conn_idle_time` and
`pool_health_check_period` are pgx extensions to the URL, which means a service
can be retuned by changing one environment variable with no redeploy. That is
genuinely useful in an incident.

**`ParseConfig` validates the pool params too** (case 6), so a typo in config
fails at startup rather than being silently ignored.

**Defaults are not zero.** `MaxConnLifetime` is 1 hour, `MaxConnIdleTime` is 30
minutes, `HealthCheckPeriod` is 1 minute, and `MaxConns` is `max(4, NumCPU)` of
the machine **running the client**. That last one is a trap worth naming now:
your pool size silently changes when you move to a bigger instance. 17.5 says
set it explicitly.

Idiomatic: parse the DSN from the environment with `ParseConfig`, then override
the pool fields in code so they are reviewable, and use `NewWithConfig`.

### Environment variables

With an empty or partial DSN, pgx falls back to libpq's variables:

| Variable | Fills in | Default if unset |
|---|---|---|
| `PGHOST` | host | a local Unix socket directory |
| `PGPORT` | port | `5432` |
| `PGUSER` | user | the OS user name |
| `PGPASSWORD` | password | empty, then `~/.pgpass` |
| `PGDATABASE` | database | the user name |
| `PGSSLMODE` | TLS mode | `prefer` |
| `PGCONNECT_TIMEOUT` | dial timeout | none |
| `PGAPPNAME` | `application_name` | empty |

Precedence, highest first: **explicit DSN fields**, then **environment
variables**, then **`~/.pgpass` and the service file**, then **built-in
defaults**. So `postgres:///shop` with `PGHOST=db.internal` connects to
`db.internal`, while `postgres://localhost/shop` ignores `PGHOST` entirely.

For a service, pass one `DATABASE_URL` and do not rely on the `PG*` variables —
they are convenient for `psql` and confusing in a deployment manifest
([18.2](../18-production/02-config.md)).

## Connection failure, for real

This is the behaviour to internalise: `pgxpool.New` **does not connect**, in
exactly the way `sql.Open` does not. Run against a machine with nothing on 5432:

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "net"

  "github.com/jackc/pgx/v5/pgxpool"
)

const dsn = "postgres://app:s3cret@localhost:5432/shop?sslmode=disable"

func main() {
  ctx := context.Background()

  fmt.Println("=== 1. pgxpool.New with nothing listening ===")
  pool, err := pgxpool.New(ctx, dsn)
  fmt.Printf("  pgxpool.New err=%v  pool!=nil=%v\n", err, pool != nil)
  fmt.Printf("  Stat: Total=%d Idle=%d Acquired=%d Max=%d New=%d\n",
    pool.Stat().TotalConns(), pool.Stat().IdleConns(),
    pool.Stat().AcquiredConns(), pool.Stat().MaxConns(),
    pool.Stat().NewConnsCount())

  fmt.Println("\n=== 2. pool.Ping is what actually connects ===")
  err = pool.Ping(ctx)
  fmt.Printf("  pool.Ping err:\n%v\n", err)

  fmt.Println("\n=== 3. classifying the error ===")
  var netErr *net.OpError
  fmt.Printf("  errors.As(*net.OpError)=%v\n", errors.As(err, &netErr))
  if errors.As(err, &netErr) {
    fmt.Printf("  Op=%q Net=%q Addr=%v Err=%v\n", netErr.Op, netErr.Net, netErr.Addr, netErr.Err)
  }
  fmt.Printf("  errors.Is(err, context.DeadlineExceeded)=%v\n",
    errors.Is(err, context.DeadlineExceeded))

  fmt.Println("\n=== 4. the first query connects too (and fails the same way) ===")
  var n int
  qerr := pool.QueryRow(ctx, "SELECT 1").Scan(&n)
  fmt.Printf("  QueryRow err is non-nil: %v\n", qerr != nil)
  pool.Close()
  fmt.Printf("  after Close, Stat.TotalConns=%d\n", pool.Stat().TotalConns())
}
```

Output:

```
=== 1. pgxpool.New with nothing listening ===
  pgxpool.New err=<nil>  pool!=nil=true
  Stat: Total=0 Idle=0 Acquired=0 Max=12 New=0

=== 2. pool.Ping is what actually connects ===
  pool.Ping err:
failed to connect to `user=app database=shop`:
	[::1]:5432 (localhost): dial error: dial tcp [::1]:5432: connect: connection refused
	127.0.0.1:5432 (localhost): dial error: dial tcp 127.0.0.1:5432: connect: connection refused

=== 3. classifying the error ===
  errors.As(*net.OpError)=true
  Op="dial" Net="tcp" Addr=[::1]:5432 Err=connect: connection refused
  errors.Is(err, context.DeadlineExceeded)=false

=== 4. the first query connects too (and fails the same way) ===
  QueryRow err is non-nil: true
  after Close, Stat.TotalConns=0
```

Read that error text closely, because you will see it a lot.

**`pgxpool.New` returned a nil error and a usable pool against a dead server.**
The pool is lazy by design: it has nothing to connect *for* yet. This is the
same lesson as `sql.Open`, and the same fix — ping at startup.

**`localhost` resolved to two addresses and pgx tried both.** The error lists
`[::1]:5432` and `127.0.0.1:5432` separately. That is why "connection refused"
appears twice, and it is the usual reason a container that binds only IPv4
gives a confusing half-failure: pgx tries IPv6 first, gets refused, falls back.
If you see only the `[::1]` line in a timeout, your IPv6 attempt is being
dropped rather than refused, and `127.0.0.1` in the DSN is the quick diagnosis.

**The error wraps a `*net.OpError`,** so `errors.As` gets you structured access
to `Op`, `Net` and `Addr`. In practice you rarely need it — "cannot reach the
database" is not a condition your handler can do anything clever about, and it
maps to a 503 ([16.9](../16-http/09-error-responses.md)) — but it is how a
health check distinguishes "refused" from "timed out".

**The first query connects too**, with the same error. There is no state where
the pool is "broken"; each acquire tries again.

### The real DeadlineExceeded

Connection *refused* is instant, so a short timeout does not change the error —
a surprise worth stating, because the obvious experiment appears to disprove
the deadline. To see `context.DeadlineExceeded` you need a dial that **hangs**,
which a non-routable address provides:

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "time"

  "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
  const dsn = "postgres://app:s3cret@10.255.255.1:5432/shop?sslmode=disable"

  pool, err := pgxpool.New(context.Background(), dsn)
  if err != nil {
    panic(err)
  }
  defer pool.Close()

  ctx, cancel := context.WithTimeout(context.Background(), 300*time.Millisecond)
  defer cancel()

  start := time.Now()
  err = pool.Ping(ctx)
  fmt.Printf("gave up after ~%dms\n", time.Since(start).Round(100*time.Millisecond).Milliseconds())
  fmt.Printf("err = %v\n", err)
  fmt.Printf("errors.Is(err, context.DeadlineExceeded) = %v\n",
    errors.Is(err, context.DeadlineExceeded))
  fmt.Printf("errors.Is(err, context.Canceled)         = %v\n",
    errors.Is(err, context.Canceled))
}
```

Output:

```
gave up after ~300ms
err = context deadline exceeded
errors.Is(err, context.DeadlineExceeded) = true
errors.Is(err, context.Canceled)         = false
```

The context won at exactly its deadline, and `errors.Is` distinguishes a
timeout from a cancellation — which is how 16.12's handler chose 504 over 499.
That distinction survives all the way from `pool.Ping` up to the HTTP status,
with no code in between doing anything special.

Note the error here is **bare** `context deadline exceeded`, with no DSN and no
address, because the dial never got far enough to produce one. A timeout tells
you less than a refusal does.

## Startup: the shape to copy

```go
func OpenPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
  cfg, err := pgxpool.ParseConfig(dsn)
  if err != nil {
    return nil, fmt.Errorf("parsing DATABASE_URL: %w", err)
  }

  cfg.MaxConns = 20
  cfg.MinConns = 2
  cfg.MinIdleConns = 2
  cfg.MaxConnLifetime = 30 * time.Minute
  cfg.MaxConnLifetimeJitter = 5 * time.Minute
  cfg.MaxConnIdleTime = 5 * time.Minute
  cfg.HealthCheckPeriod = 30 * time.Second
  cfg.ConnConfig.RuntimeParams["application_name"] = "shop-api"

  pool, err := pgxpool.NewWithConfig(ctx, cfg)
  if err != nil {
    return nil, fmt.Errorf("creating pool: %w", err)
  }

  pingCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
  defer cancel()
  if err := pool.Ping(pingCtx); err != nil {
    pool.Close()
    return nil, fmt.Errorf("connecting to database: %w", err)
  }
  return pool, nil
}
```

Four deliberate choices. Pool fields are set **in code**, not left to the DSN,
so a code reviewer can see them. `application_name` is set because it shows up
in `pg_stat_activity` and turns "some client is holding a lock" into "the
shop-api is holding a lock". The ping has **its own** timeout, so a hung DNS
lookup cannot hang your boot forever. And a failed ping **closes the pool**
before returning, because a `*pgxpool.Pool` you drop on the floor keeps its
health-check goroutine running.

`pool.Close()` is the matching shutdown call. It waits for checked-out
connections to be released, so call it after your HTTP server has drained
([18.5](../18-production/05-graceful-shutdown.md)) — closing it first makes
every in-flight request fail on its way out.

### Hooks

`pgxpool.Config` has callbacks for per-connection setup:

| Hook | Signature | Use for |
|---|---|---|
| `BeforeConnect` | `func(ctx, *pgx.ConnConfig) error` | rotating credentials, IAM auth tokens |
| `AfterConnect` | `func(ctx, *pgx.Conn) error` | `SET` session params, registering custom types |
| `PrepareConn` | `func(ctx, *pgx.Conn) (bool, error)` | validating a connection before it is handed out |
| `AfterRelease` | `func(*pgx.Conn) bool` | returning `false` to discard a dirtied connection |
| `BeforeClose` | `func(*pgx.Conn)` | logging churn |

```go
cfg.AfterConnect = func(ctx context.Context, conn *pgx.Conn) error {
  _, err := conn.Exec(ctx, "SET statement_timeout = '5s'")
  return err
}
```

A server-side `statement_timeout` is a good belt-and-braces companion to your
context deadlines, because it stops a runaway query even if the client
goroutine has wandered off. `BeforeAcquire` still exists but is deprecated in
v5.11 in favour of `PrepareConn`, which can return an error rather than only a
bool.

## TLS and sslmode

| `sslmode` | Encrypted | Server cert verified | Hostname verified |
|---|---|---|---|
| `disable` | no | — | — |
| `allow` | only if the server insists | no | no |
| `prefer` (**pgx default**) | if offered | no | no |
| `require` | yes | **no** | no |
| `verify-ca` | yes | yes | no |
| `verify-full` | yes | yes | yes |

The trap is `require`: it encrypts but verifies nothing, so it stops passive
sniffing and does nothing against an active man-in-the-middle. For a managed
Postgres over the public internet, `verify-full` with the provider's CA bundle
is the only mode worth having:

```
postgres://app:s3cret@db.example.com:5432/shop?sslmode=verify-full&sslrootcert=/etc/ssl/rds-ca.pem
```

`disable` is fine inside a private network and is what the local Docker setup
below uses. Note that the **default is `prefer`**, so a DSN with no `sslmode`
silently gives you unverified encryption — pick one explicitly.

## Running Postgres locally

This is the block that makes the labelled outputs in the rest of the module
reproducible. Save as `docker-compose.yml`:

```yaml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: s3cret
      POSTGRES_DB: shop
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d shop"]
      interval: 2s
      timeout: 3s
      retries: 15
    command:
      - postgres
      - -c
      - log_statement=all
      - -c
      - max_connections=100

volumes:
  pgdata:
```

```bash
docker compose up -d
docker compose exec db pg_isready -U app -d shop

export DATABASE_URL='postgres://app:s3cret@localhost:5432/shop?sslmode=disable'
go run .

docker compose exec db psql -U app -d shop -c '\dt'
docker compose logs -f db
docker compose down -v
```

The healthcheck matters more than it looks: without it, `docker compose up -d`
returns as soon as the container starts, and Postgres takes a second or two
more to accept connections — so your first `go run .` fails with the
connection-refused error above and you conclude your DSN is wrong.
`pg_isready` is the gate.

For the migration CLI used in 17.6:

```bash
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@v4.20.1
```

And for a throwaway server with no compose file:

```bash
docker run --rm -d --name pg17 \
  -e POSTGRES_PASSWORD=s3cret -e POSTGRES_USER=app -e POSTGRES_DB=shop \
  -p 5432:5432 postgres:17-alpine
```

## pool.Stat in its zero state

`pool.Stat()` is a snapshot, and it is what you export as metrics in
[18.7](../18-production/07-prometheus.md). Here it is before anything has
connected — real output from the lazy pool above:

```
Stat: Total=0 Idle=0 Acquired=0 Max=12 New=0
```

`Total=0` with `Max=12` is the signature of a pool nobody has used yet. 17.5
walks through every field and what each one tells you under load.

## Quick Reference

| Call | Note |
|---|---|
| `pgxpool.New(ctx, dsn)` | parses, returns a pool, **does not connect** |
| `pgxpool.NewWithConfig(ctx, cfg)` | same, from a `*pgxpool.Config` |
| `pgxpool.ParseConfig(dsn)` | offline; validates the DSN and pool params |
| `pool.Ping(ctx)` | the eager connect; use at startup with a timeout |
| `pool.Acquire(ctx)` | `(*pgxpool.Conn, error)`; **must** `Release()` |
| `pool.Stat()` | `*pgxpool.Stat` snapshot |
| `pool.Close()` | waits for released conns; call once, at shutdown |
| `pool.Config()` | a copy of the config in use |
| `pgx.Connect(ctx, dsn)` | one `*pgx.Conn`; **not** concurrency-safe |
| `pgx.ParseConfig(dsn)` | the connection half, without pool fields |

| DSN parameter | Meaning |
|---|---|
| `sslmode` | `disable`/`prefer`/`require`/`verify-ca`/`verify-full` |
| `connect_timeout` | dial timeout in seconds |
| `application_name` | shows in `pg_stat_activity` |
| `pool_max_conns` | `MaxConns` |
| `pool_min_conns` | `MinConns` |
| `pool_max_conn_lifetime` | `MaxConnLifetime` |
| `pool_max_conn_idle_time` | `MaxConnIdleTime` |
| `pool_health_check_period` | `HealthCheckPeriod` |

| Error you will see | Means |
|---|---|
| `connect: connection refused` | nothing is listening on that host:port |
| `context deadline exceeded` (bare) | the dial hung; wrong host or a dropping firewall |
| `password authentication failed for user "app"` | credentials |
| `database "shop" does not exist` | connected and authenticated; wrong dbname |
| `server error: FATAL: sorry, too many clients already` | `max_connections` exhausted (17.5) |
| `failed to parse as keyword/value` | malformed DSN; check the scheme |

## Common Mistakes

**Assuming `pgxpool.New` connected.** It is lazy exactly like `sql.Open`, so a
wrong host passes startup and fails on the first request; `Ping` with its own
timeout and fail the boot.

**Sharing a `*pgx.Conn` between goroutines.** It is a single protocol stream and
will corrupt; use `pgxpool`.

**Acquiring a connection and forgetting `Release()`.** Each leak permanently
shrinks the pool until every acquire blocks; `defer conn.Release()`.

**Leaving `sslmode` unset.** The default is `prefer`, which encrypts without
verifying anything; choose `disable` or `verify-full` deliberately.

**Leaving `MaxConns` at its default.** It is `max(4, NumCPU)` of the client
machine, so your pool size changes when you resize the instance (17.5).

**Creating a pool per request.** Each one starts a health-check goroutine and
its own connections; build one at startup and inject it.

**Calling `pool.Close()` before the HTTP server has drained.** In-flight
requests fail on their way out; close the pool after `srv.Shutdown` returns.

**Putting the password in your own error messages.** pgx redacts the DSN in
*its* errors; `fmt.Errorf("connecting to %s: %w", dsn, err)` undoes that.

## Exercises

1. Write a `ParseConfig`-based validator that rejects a `DATABASE_URL` at
   startup if `sslmode` is missing or is `prefer`/`allow`, and prints which
   field was at fault. Show it accepting one DSN and rejecting three.
2. Use `AfterConnect` to `SET statement_timeout = '100ms'` and
   `SET application_name`, then query `pg_stat_activity` to prove both took
   effect on every pooled connection. (Needs the compose stack.)
3. Capture the three distinct failure modes as separate real errors: wrong port
   (refused), unroutable host (deadline), and a hostname that does not resolve.
   Write a `classifyConnErr(error) string` that distinguishes all three with
   `errors.As`/`errors.Is` and no string matching.
4. Compare `pgx.Connect` and `pgxpool.New` under concurrency: run 50 goroutines
   issuing `SELECT pg_sleep(0.01)` against each, with `-race`. Report what
   happens to the single connection and explain the output.
5. Build a readiness handler ([18.6](../18-production/06-health-checks.md)) that
   reports `pool.Stat()` as JSON and returns 503 if `Ping` fails within 1s.
   Prove it returns 503 with no server and 200 with the compose stack up.

---

[← 17.1 database/sql and the Driver Model](01-database-sql.md) · [Course Index](../README.md) · [17.3 Queries, Scanning, and NULL Handling →](03-queries-scanning.md)
