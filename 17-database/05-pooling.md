[← 17.4 Transactions](04-transactions.md) · [Course Index](../README.md) · [17.6 Migrations with golang-migrate →](06-migrations.md)

# 17.5 — Connection Pooling and Tuning

## What You'll Learn

- Every `pgxpool.Config` field with its **real** default printed, including the `MaxConns = max(4, NumCPU)` trap
- `pool.Stat()`'s nine counters, what each one tells you, and a real zero-state and saturation printout
- Why a bigger pool is usually slower, and the arithmetic that keeps `MaxConns × instances` under `max_connections`
- The real `context.DeadlineExceeded` from a saturated pool, and PgBouncer's incompatibility with prepared statements

The defaults, the stats and the acquire-timeout output here are real. The
under-load printout from a live server is labelled.

## Coming From Another Language

| Other | Go |
|---|---|
| `max: 20` in a node-pg config | `cfg.MaxConns = 20` on a parsed `*pgxpool.Config` |
| `pool_pre_ping=True` | `HealthCheckPeriod` plus `ShouldPing` |
| a pool that throws when exhausted | `Acquire` **blocks** until your `ctx` expires |
| `pool_recycle=3600` | `MaxConnLifetime`, plus `MaxConnLifetimeJitter` |
| a per-request connection | one pool for the process; never per request |
| default pool size 5 or 10 | `max(4, NumCPU)` — of the **client** machine |

The pool is the single most common production misconfiguration in a Go
backend, and it fails in two opposite directions: too small and your p99 is
all queueing, too large and you exhaust the server or make Postgres slower for
everyone. Both look like "the database is slow".

## The defaults, printed

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "runtime"
  "time"

  "github.com/jackc/pgx/v5/pgxpool"
)

func printStat(label string, s *pgxpool.Stat) {
  fmt.Printf("%s\n", label)
  fmt.Printf("  MaxConns=%d TotalConns=%d IdleConns=%d AcquiredConns=%d ConstructingConns=%d\n",
    s.MaxConns(), s.TotalConns(), s.IdleConns(), s.AcquiredConns(), s.ConstructingConns())
  fmt.Printf("  AcquireCount=%d NewConnsCount=%d EmptyAcquireCount=%d CanceledAcquireCount=%d\n",
    s.AcquireCount(), s.NewConnsCount(), s.EmptyAcquireCount(), s.CanceledAcquireCount())
  fmt.Printf("  AcquireDuration=%v EmptyAcquireWaitTime=%v\n",
    s.AcquireDuration(), s.EmptyAcquireWaitTime())
  fmt.Printf("  MaxLifetimeDestroyCount=%d MaxIdleDestroyCount=%d\n",
    s.MaxLifetimeDestroyCount(), s.MaxIdleDestroyCount())
}

func main() {
  fmt.Printf("runtime.NumCPU() = %d\n\n", runtime.NumCPU())

  cfg, err := pgxpool.ParseConfig("postgres://app@localhost:5432/shop?sslmode=disable")
  if err != nil {
    panic(err)
  }
  fmt.Printf("defaults from ParseConfig:\n")
  fmt.Printf("  MaxConns              = %d   (greater of 4 and NumCPU)\n", cfg.MaxConns)
  fmt.Printf("  MinConns              = %d\n", cfg.MinConns)
  fmt.Printf("  MinIdleConns          = %d\n", cfg.MinIdleConns)
  fmt.Printf("  MaxConnLifetime       = %v\n", cfg.MaxConnLifetime)
  fmt.Printf("  MaxConnLifetimeJitter = %v\n", cfg.MaxConnLifetimeJitter)
  fmt.Printf("  MaxConnIdleTime       = %v\n", cfg.MaxConnIdleTime)
  fmt.Printf("  HealthCheckPeriod     = %v\n", cfg.HealthCheckPeriod)
  fmt.Printf("  PingTimeout           = %v\n\n", cfg.PingTimeout)

  cfg.MaxConns = 8
  cfg.MinConns = 2
  cfg.MinIdleConns = 2
  cfg.MaxConnLifetime = 30 * time.Minute
  cfg.MaxConnLifetimeJitter = 5 * time.Minute
  cfg.MaxConnIdleTime = 5 * time.Minute
  cfg.HealthCheckPeriod = 30 * time.Second

  pool, err := pgxpool.NewWithConfig(context.Background(), cfg)
  if err != nil {
    panic(err)
  }
  defer pool.Close()

  printStat("zero state (nothing has connected yet):", pool.Stat())

  fmt.Println("\nacquire with a 50ms budget against a server that is not there:")
  ctx, cancel := context.WithTimeout(context.Background(), 50*time.Millisecond)
  defer cancel()
  conn, err := pool.Acquire(ctx)
  fmt.Printf("  Acquire err = %v\n", err)
  fmt.Printf("  errors.Is(err, context.DeadlineExceeded) = %v\n",
    errors.Is(err, context.DeadlineExceeded))
  if conn != nil {
    conn.Release()
  }

  fmt.Println()
  printStat("after the failed acquire:", pool.Stat())
}
```

Output:

```
runtime.NumCPU() = 12

defaults from ParseConfig:
  MaxConns              = 12   (greater of 4 and NumCPU)
  MinConns              = 0
  MinIdleConns          = 0
  MaxConnLifetime       = 1h0m0s
  MaxConnLifetimeJitter = 0s
  MaxConnIdleTime       = 30m0s
  HealthCheckPeriod     = 1m0s
  PingTimeout           = 0s

zero state (nothing has connected yet):
  MaxConns=8 TotalConns=0 IdleConns=0 AcquiredConns=0 ConstructingConns=0
  AcquireCount=0 NewConnsCount=0 EmptyAcquireCount=0 CanceledAcquireCount=0
  AcquireDuration=0s EmptyAcquireWaitTime=0s
  MaxLifetimeDestroyCount=0 MaxIdleDestroyCount=0

acquire with a 50ms budget against a server that is not there:
  Acquire err = failed to connect to `user=app database=shop`:
	[::1]:5432 (localhost): dial error: dial tcp [::1]:5432: connect: connection refused
	127.0.0.1:5432 (localhost): dial error: dial tcp 127.0.0.1:5432: connect: connection refused
  errors.Is(err, context.DeadlineExceeded) = false

after the failed acquire:
  MaxConns=8 TotalConns=0 IdleConns=0 AcquiredConns=0 ConstructingConns=0
  AcquireCount=0 NewConnsCount=3 EmptyAcquireCount=0 CanceledAcquireCount=0
  AcquireDuration=0s EmptyAcquireWaitTime=0s
  MaxLifetimeDestroyCount=0 MaxIdleDestroyCount=0
```

In that last block `TotalConns` and `ConstructingConns` vary run to run: the
`MinConns: 2` background connects are racing the printout, so you will also see
`TotalConns=1 ConstructingConns=1`. `NewConnsCount=3` is stable — two attempts
for `MinConns` plus the one the failed `Acquire` triggered, all of which failed
to connect, which is why `TotalConns` settles back to zero.

### MaxConns is the trap

`MaxConns` defaulted to **12** on this machine because it has 12 cores. Deploy
the identical binary to a 4-core container and it is 4; to a 64-core node and
it is 64. Nothing in your configuration changed, and your database's connection
count quadrupled.

Worse, in a container the value pgx sees is the **host's** core count unless
`GOMAXPROCS` is set from the cgroup limit, so a 0.5-CPU pod on a 64-core node
gets `MaxConns = 64`. Go 1.25 made `GOMAXPROCS` cgroup-aware, which fixes the
common case, but `runtime.NumCPU()` is still the machine's count.

**Always set `MaxConns` explicitly.** This is the one tuning parameter you must
not leave to a default.

The other defaults are more defensible: a 1-hour lifetime, a 30-minute idle
timeout, a 1-minute health check. The two zeroes to think about are `MinConns`
(no connections are kept warm, so the first request after a quiet period pays
the connect cost) and `MaxConnLifetimeJitter`, covered below.

## The config fields

| Field | Default | What it does | Set it to |
|---|---|---|---|
| `MaxConns` | `max(4, NumCPU)` | hard ceiling on connections | **always explicit**; see the arithmetic below |
| `MinConns` | `0` | pool floor, replenished by the health check | 2–5 for a latency-sensitive service |
| `MinIdleConns` | `0` | minimum **idle** connections kept ready | preferred over `MinConns` for warm capacity |
| `MaxConnLifetime` | `1h` | age at which a connection is retired | 30m–1h |
| `MaxConnLifetimeJitter` | `0` | random extra lifetime, per connection | 10–20% of `MaxConnLifetime` |
| `MaxConnIdleTime` | `30m` | idle age at which a connection is reaped | 5–15m |
| `HealthCheckPeriod` | `1m` | how often idle connections are checked | 15–60s |
| `PingTimeout` | `0` (none) | budget for a health-check ping | 2–5s |
| `ConnConfig.ConnectTimeout` | none | dial timeout | 5s |

`MinIdleConns` is newer than `MinConns` and is the better knob: `MinConns`
counts *all* connections, so a pool with 5 busy connections and `MinConns: 5`
keeps zero idle and the next request still waits for a new connect.
`MinIdleConns: 2` guarantees two are actually available.

## pool.Stat()

| Accessor | Type | Meaning | Watch for |
|---|---|---|---|
| `MaxConns()` | `int32` | the configured ceiling | — |
| `TotalConns()` | `int32` | acquired + idle + constructing | at `MaxConns` means saturated |
| `IdleConns()` | `int32` | available right now | persistently 0 under load |
| `AcquiredConns()` | `int32` | checked out right now | **== MaxConns is the alarm** |
| `ConstructingConns()` | `int32` | mid-connect | high means connect latency |
| `AcquireCount()` | `int64` | successful acquires, cumulative | your query rate |
| `NewConnsCount()` | `int64` | connections ever created | churn; should flatten out |
| `EmptyAcquireCount()` | `int64` | acquires that had to **wait** | **the pool-too-small signal** |
| `EmptyAcquireWaitTime()` | `Duration` | total time spent waiting | divide by `EmptyAcquireCount` |
| `CanceledAcquireCount()` | `int64` | acquires that gave up | **requests you dropped** |
| `AcquireDuration()` | `Duration` | total time in `Acquire` | divide by `AcquireCount` |
| `MaxLifetimeDestroyCount()` | `int64` | retired for age | should track `NewConnsCount` |
| `MaxIdleDestroyCount()` | `int64` | reaped for idleness | high means bursty traffic |

The two that matter most are derived, not raw:

```go
func poolPressure(s *pgxpool.Stat) (waitRatio float64, meanWait time.Duration) {
  if s.AcquireCount() > 0 {
    waitRatio = float64(s.EmptyAcquireCount()) / float64(s.AcquireCount())
  }
  if s.EmptyAcquireCount() > 0 {
    meanWait = s.EmptyAcquireWaitTime() / time.Duration(s.EmptyAcquireCount())
  }
  return waitRatio, meanWait
}
```

A `waitRatio` near zero means the pool is big enough. A rising `waitRatio` with
a small `meanWait` means brief contention, which is fine. A rising
`waitRatio` with a `meanWait` in the tens of milliseconds means you are
queueing, and `CanceledAcquireCount() > 0` means you are **dropping** requests.

### A saturated pool, for real

When every connection is checked out, `Acquire` blocks. It does not error, and
it does not return a nil connection — it waits for a free connection or for
your context to expire. With a hanging server and `MaxConns = 2`:

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
  cfg, err := pgxpool.ParseConfig("postgres://app@10.255.255.1:5432/shop?sslmode=disable")
  if err != nil {
    panic(err)
  }
  cfg.MaxConns = 2

  pool, err := pgxpool.NewWithConfig(context.Background(), cfg)
  if err != nil {
    panic(err)
  }
  defer pool.Close()

  ctx, cancel := context.WithTimeout(context.Background(), 200*time.Millisecond)
  defer cancel()

  start := time.Now()
  conn, err := pool.Acquire(ctx)
  fmt.Printf("Acquire returned after ~%dms\n", time.Since(start).Round(100*time.Millisecond).Milliseconds())
  fmt.Printf("err = %v\n", err)
  fmt.Printf("errors.Is(err, context.DeadlineExceeded) = %v\n",
    errors.Is(err, context.DeadlineExceeded))
  if conn != nil {
    conn.Release()
  }

  s := pool.Stat()
  fmt.Printf("\nStat: MaxConns=%d TotalConns=%d AcquiredConns=%d\n",
    s.MaxConns(), s.TotalConns(), s.AcquiredConns())
  fmt.Printf("      AcquireCount=%d CanceledAcquireCount=%d EmptyAcquireCount=%d\n",
    s.AcquireCount(), s.CanceledAcquireCount(), s.EmptyAcquireCount())
}
```

Output:

```
Acquire returned after ~200ms
err = context deadline exceeded
errors.Is(err, context.DeadlineExceeded) = false

Stat: MaxConns=2 TotalConns=1 AcquiredConns=0
      AcquireCount=0 CanceledAcquireCount=1 EmptyAcquireCount=0
```

The acquire burned the entire 200ms budget and gave up with
`context deadline exceeded`, and **`CanceledAcquireCount` went to 1**. That
counter is the one to alert on: it is a direct count of requests your service
abandoned because it could not get a connection.

Note `errors.Is(err, context.DeadlineExceeded)` is **false** here even though
the message is `context deadline exceeded` — puddle, the pool library under
`pgxpool`, returns its own error value rather than wrapping the context's. So
in a handler, do not rely on `errors.Is` to catch pool exhaustion; compare
against the acquire path, or treat any error from `Acquire` as a 503. This
differs from `pool.Ping`, which **does** wrap it (17.2), and it is the sort of
asymmetry worth knowing before you write the classifier.

What this means for your handler, in one line: a saturated pool turns into
**queueing**, and queueing plus a per-request deadline turns into 503s. There
is no error that says "the pool is full"; there is only a request that ran out
of time.

Expected output (requires a running Postgres; not executed in this environment):

```
50 concurrent SELECT pg_sleep(0.05) against MaxConns=5:
  Stat: MaxConns=5 TotalConns=5 IdleConns=0 AcquiredConns=5
        AcquireCount=50 NewConnsCount=5 EmptyAcquireCount=45
        EmptyAcquireWaitTime=2.1s  mean wait 46ms
        CanceledAcquireCount=0
  waitRatio=0.90 meanWait=46ms  <- the pool is the bottleneck
```

That is the shape to recognise: `AcquireCount` is 50, `EmptyAcquireCount` is 45,
so 90% of requests waited, and the mean wait is roughly one query duration
because they queued one deep.

## Sizing the pool

The instinct is that more connections means more throughput. It is wrong, and
the reason is worth one tight paragraph: a Postgres connection is an **OS
process** with its own memory (`work_mem` per sort, plus overhead), and every
one of them contends for the same finite resources — the buffer pool, the WAL
writer, the lock manager's shared hash tables, and ultimately a fixed number of
cores and disk queues. Past the point where the server's cores and disks are
busy, additional concurrency adds context switching, lock-manager contention
and `work_mem` pressure without adding work done, so **total throughput falls
and every query gets slower**. The database becomes the queue, and a queue
inside Postgres is far worse than a queue in your pool, because Postgres cannot
time it out, cannot prioritise it, and cannot tell you about it.

A starting point, not a formula to trust:

```
MaxConns per instance  ≈  (cores_on_the_db_host × 2) + effective_spindles
                          ───────────────────────────────────────────────
                                      number_of_app_instances
```

Then sanity-check it against the server:

```
Σ (MaxConns × instances)  +  migrations  +  admin/psql  +  monitoring
     must stay below  max_connections − superuser_reserved_connections
```

| Setting | Typical | Note |
|---|---|---|
| `max_connections` | 100 (default), 200–500 tuned | a server-side cost, not free |
| `superuser_reserved_connections` | 3 | you cannot use these |
| migration job | 1–2 | runs at deploy, spikes the count (17.6) |
| monitoring exporter | 1–3 | easy to forget |
| a human with `psql` | 1–2 | always leave room |

A worked example. A 4-core database with `max_connections = 100`, eight API
replicas, two background workers, a migration job and an exporter:

- usable: `100 − 3 = 97`
- reserve: `2` (migrations) `+ 3` (exporter) `+ 2` (humans) `= 7`
- for the apps: `90`
- `90 / 10 instances = 9` per instance → set `MaxConns = 8` for headroom

Eight connections per replica feels small to people used to a default of 100.
It is correct: 4 cores cannot usefully run more than about 10 concurrent
queries, and 80 connections to a 4-core box is strictly worse than 32.

Three things that break the arithmetic in practice:

**Autoscaling.** Your replica count is the multiplier, so a scale-out event
multiplies your connection count. Size for `max_replicas`, not for today's
count, or put PgBouncer in front.

**Read replicas.** They have their own `max_connections`, so a separate
read-only pool is separate arithmetic — and it is usually the better design
than one pool doing both, because a read pool can be bigger and more tolerant.

**Serverless and per-request runtimes.** Lambda-style deployments defeat pooling
entirely: a thousand concurrent invocations means a thousand pools of size one.
That is what PgBouncer or a provider's data proxy exists for.

Idiomatic: bound your **application** concurrency (a semaphore, a worker pool
from [12.4](../12-concurrency-patterns/04-rate-limiting.md)) rather than relying
on the connection pool to do it. The pool's queue has no priority and no
backpressure to your caller; a semaphore in front of your handler can shed load
with a fast 503.

## Connection lifetime

`MaxConnLifetime` retires healthy connections on a timer. That sounds
counterproductive until you list what it fixes:

- **Load balancer and failover churn.** After a failover, long-lived
  connections point at the old primary, or at a node that is now a replica. A
  lifetime guarantees the fleet rediscovers DNS within that window.
- **Server-side memory.** A Postgres backend's memory footprint only grows:
  the relation cache, the catalog cache and the plan cache accumulate.
  Recycling bounds it.
- **Rolling upgrades.** The old connections drain on their own.
- **Credential rotation.** An IAM auth token or a rotated password takes effect
  within one lifetime instead of at the next restart.

30 minutes to 1 hour is the usual range. Much shorter and you pay the TLS
handshake and `AfterConnect` cost too often.

### The jitter, and why it is not optional

`MaxConnLifetime` alone creates a thundering herd. Every connection in a pool
is created at roughly the same moment — the first traffic burst after a deploy
— so every connection reaches its lifetime at roughly the same moment, and the
entire pool reconnects at once. For a moment the pool is empty, every request
queues, and your p99 spikes on a 30-minute cycle. Multiply by every replica,
which all deployed together, and the whole fleet reconnects simultaneously.

```go
cfg.MaxConnLifetime = 30 * time.Minute
cfg.MaxConnLifetimeJitter = 5 * time.Minute
```

Each connection now lives 30 to 35 minutes, chosen independently, so
retirements spread out. 10–20% of the lifetime is a good default. The
`MaxLifetimeDestroyCount` counter is how you confirm it: a smooth climb is
healthy, a staircase means your jitter is too small.

## PgBouncer and prepared statements

PgBouncer multiplexes many client connections onto few server connections. In
**transaction pooling** mode — the mode worth using — a server connection is
assigned per *transaction*, so consecutive statements from one client can land
on different server connections.

That breaks anything with per-connection server-side state, and pgx's automatic
statement cache is exactly that: pgx prepares a statement on connection A, then
PgBouncer routes the execute to connection B, which has never heard of it.

```
ERROR: prepared statement "stmt_1" does not exist (SQLSTATE 26000)
```

The fix is to tell pgx not to use named prepared statements:

```go
cfg, err := pgxpool.ParseConfig(dsn)
if err != nil {
  return nil, err
}
cfg.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeExec
cfg.ConnConfig.StatementCacheCapacity = 0
cfg.ConnConfig.DescriptionCacheCapacity = 0
```

or in the DSN:

```
postgres://app:s3cret@pgbouncer:6432/shop?sslmode=disable&default_query_exec_mode=exec
```

| `QueryExecMode` | Protocol | Placeholders | PgBouncer transaction mode |
|---|---|---|---|
| `QueryExecModeCacheStatement` (**default**) | extended, named prepare, cached | server-side | **broken** |
| `QueryExecModeCacheDescribe` | extended, cached descriptions | server-side | **broken** |
| `QueryExecModeDescribeExec` | extended, describe each time | server-side | works, one extra round trip |
| `QueryExecModeExec` | extended, unnamed prepare | server-side | **works** |
| `QueryExecModeSimpleProtocol` | simple | **client-side interpolation** | works |

Prefer `QueryExecModeExec`: it keeps server-side parameter binding, so you keep
the injection safety and the type handling, and only lose the plan cache.

`QueryExecModeSimpleProtocol` is the one to be careful with. It interpolates
arguments into the SQL string **in the client**, using pgx's own escaping. pgx
does it correctly, but it means your values become SQL text, it loses binary
encoding for most types, and it cannot send a statement with no arguments
through the same path. Use it only when something downstream cannot speak the
extended protocol at all.

Two more PgBouncer consequences. `session`-scoped features break in transaction
mode: `SET` (outside a transaction), advisory locks held across statements,
`LISTEN`/`NOTIFY`, and `WITH HOLD` cursors. And your pool arithmetic changes —
with PgBouncer in front, `MaxConns` in each app is a bound on *client* slots,
which are cheap, while PgBouncer's `default_pool_size` is the real bound on
server connections.

## Health checks

Every `HealthCheckPeriod`, pgx checks idle connections: it destroys ones past
`MaxConnIdleTime` or `MaxConnLifetime`, creates connections to satisfy
`MinConns`/`MinIdleConns`, and pings connections that have been idle long
enough. `PingTimeout` bounds that ping; left at zero, a hung ping can hold the
health-check goroutine indefinitely, so set it:

```go
cfg.PingTimeout = 3 * time.Second
cfg.ShouldPing = func(ctx context.Context, p pgxpool.ShouldPingParams) bool {
  return p.IdleDuration > 5*time.Second
}
```

The default `ShouldPing` pings connections idle for at least a second, which is
reasonable. The health check is also what repopulates the pool after a database
restart, so a service with `MinConns > 0` recovers without any requests having
to fail first.

## Exposing the stats as metrics

```go
func RegisterPoolMetrics(reg prometheus.Registerer, pool *pgxpool.Pool) {
  gauge := func(name, help string, f func(*pgxpool.Stat) float64) {
    reg.MustRegister(prometheus.NewGaugeFunc(
      prometheus.GaugeOpts{Name: name, Help: help},
      func() float64 { return f(pool.Stat()) },
    ))
  }
  gauge("pgxpool_conns_max", "configured maximum connections",
    func(s *pgxpool.Stat) float64 { return float64(s.MaxConns()) })
  gauge("pgxpool_conns_total", "current total connections",
    func(s *pgxpool.Stat) float64 { return float64(s.TotalConns()) })
  gauge("pgxpool_conns_idle", "current idle connections",
    func(s *pgxpool.Stat) float64 { return float64(s.IdleConns()) })
  gauge("pgxpool_conns_acquired", "currently checked out",
    func(s *pgxpool.Stat) float64 { return float64(s.AcquiredConns()) })
  gauge("pgxpool_acquires_total", "cumulative successful acquires",
    func(s *pgxpool.Stat) float64 { return float64(s.AcquireCount()) })
  gauge("pgxpool_acquires_empty_total", "acquires that had to wait",
    func(s *pgxpool.Stat) float64 { return float64(s.EmptyAcquireCount()) })
  gauge("pgxpool_acquires_canceled_total", "acquires that gave up",
    func(s *pgxpool.Stat) float64 { return float64(s.CanceledAcquireCount()) })
  gauge("pgxpool_acquire_wait_seconds_total", "cumulative wait time",
    func(s *pgxpool.Stat) float64 { return s.EmptyAcquireWaitTime().Seconds() })
}
```

`GaugeFunc` reads `pool.Stat()` at scrape time, so there is no background
goroutine and no staleness. The counters are monotonic, so a `rate()` over
`pgxpool_acquires_empty_total` is your contention signal, and
`pgxpool_conns_acquired / pgxpool_conns_max` is your saturation gauge. Full
treatment in [18.7](../18-production/07-prometheus.md).

The three alerts worth having:

| Alert | Condition | Means |
|---|---|---|
| pool saturated | `acquired / max > 0.9` for 5m | too small, or queries too slow |
| requests dropped | `rate(acquires_canceled_total) > 0` | you are shedding load |
| connection churn | `rate(pgxpool_conns_new_total)` high and flat | lifetime too short, or the server is killing connections |

## A tuning checklist

1. **Set `MaxConns` explicitly.** Never ship the `NumCPU` default.
2. Compute `Σ (MaxConns × max_instances) + overhead < max_connections − 3`.
3. Set `MinIdleConns` to 2–5 so the first request after a quiet period is fast.
4. Set `MaxConnLifetime` to 30m–1h **and** `MaxConnLifetimeJitter` to 10–20% of
   it.
5. Set `MaxConnIdleTime` to 5–15m.
6. Set `ConnectTimeout` (5s) and `PingTimeout` (3s) so nothing hangs forever.
7. Set `application_name` so `pg_stat_activity` names your service.
8. Add a server-side `statement_timeout` via `AfterConnect` as a backstop to
   your context deadlines.
9. Export `pool.Stat()` and alert on `CanceledAcquireCount` and saturation.
10. Bound concurrency in the **application** with a semaphore, not by making the
    pool the queue.
11. Behind PgBouncer transaction pooling, set
    `DefaultQueryExecMode = pgx.QueryExecModeExec`.
12. Load-test with a realistic `MaxConns` and compare p99 at two or three sizes
    — the right number is measured, not derived.

When something is slow, the diagnosis order is: `AcquiredConns == MaxConns`
means the pool is the bottleneck; `EmptyAcquireCount` flat but queries slow
means the *database* is the bottleneck and a bigger pool will make it worse.
Those two look identical from the outside and have opposite fixes.

## Quick Reference

| Config field | Default | Recommended |
|---|---|---|
| `MaxConns` | `max(4, NumCPU)` | **explicit**, from the arithmetic |
| `MinConns` | `0` | 2–5 |
| `MinIdleConns` | `0` | 2–5 (better than `MinConns`) |
| `MaxConnLifetime` | `1h` | 30m–1h |
| `MaxConnLifetimeJitter` | `0` | 10–20% of lifetime |
| `MaxConnIdleTime` | `30m` | 5–15m |
| `HealthCheckPeriod` | `1m` | 15–60s |
| `PingTimeout` | `0` | 3s |
| `ConnConfig.ConnectTimeout` | none | 5s |
| `ConnConfig.DefaultQueryExecMode` | `CacheStatement` | `Exec` behind PgBouncer |

| Stat | Alarm when |
|---|---|
| `AcquiredConns() / MaxConns()` | `> 0.9` sustained |
| `EmptyAcquireCount()` | rising fast relative to `AcquireCount()` |
| `CanceledAcquireCount()` | `> 0` at all |
| `EmptyAcquireWaitTime() / EmptyAcquireCount()` | tens of ms |
| `NewConnsCount()` | climbing steadily after warm-up |
| `MaxLifetimeDestroyCount()` | in steps rather than smoothly (jitter too small) |

## Common Mistakes

**Leaving `MaxConns` at the default.** It is `max(4, NumCPU)` of the client
machine, so your connection count changes when you resize instances.

**Sizing the pool by app concurrency instead of database capacity.** 200
connections to a 4-core database is slower than 32; size against the server.

**Forgetting to multiply by replica count.** Eight replicas with `MaxConns: 50`
is 400 connections against a `max_connections` of 100.

**Setting `MaxConnLifetime` without jitter.** The whole pool retires at once
every lifetime and your p99 spikes on a cycle.

**Expecting `Acquire` to fail fast when the pool is full.** It blocks until
your context expires, so without a deadline it blocks forever.

**Assuming `errors.Is(err, context.DeadlineExceeded)` catches pool
exhaustion.** `Acquire` returns puddle's own error, not a wrapped context error;
treat any `Acquire` failure as a 503.

**Running pgx's default exec mode behind PgBouncer transaction pooling.** You
get `prepared statement "stmt_1" does not exist`; set `QueryExecModeExec`.

**Forgetting to `Release()` an acquired connection.** Each leak shrinks the pool
permanently until every acquire times out.

**Leaving `PingTimeout` at zero.** A hung health-check ping has no deadline.

## Exercises

1. Instrument a handler with `pool.Stat()` before and after, and load-test it
   at `MaxConns` 2, 8 and 32 against the compose stack with 100 concurrent
   clients. Plot p50/p99 and `EmptyAcquireCount` for each, and state which size
   you would ship and why.
2. Reproduce the thundering herd: `MaxConnLifetime: 10s` with
   `MaxConnLifetimeJitter: 0`, under steady load, logging `NewConnsCount` every
   second. Show the staircase, then add jitter and show it smooth out.
3. Write a `/debug/pool` handler returning `pool.Stat()` as JSON plus the
   derived `waitRatio` and `meanWait`, with a table-driven test over synthetic
   `Stat` values. (You will need to think about how to construct a `Stat` —
   explain what you did.)
4. Deliberately leak connections: acquire without releasing in a loop with
   `MaxConns: 3`. Capture the real error from the fourth acquire with a 1s
   deadline, and show `AcquiredConns` stuck at 3.
5. Set `max_connections=10` in the compose file and simulate four replicas with
   `MaxConns: 4` each. Capture the real `53300 too_many_connections` error, then
   fix it with the arithmetic from this lesson.

---

[← 17.4 Transactions](04-transactions.md) · [Course Index](../README.md) · [17.6 Migrations with golang-migrate →](06-migrations.md)
