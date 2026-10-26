[← 18.5 Graceful Shutdown](05-graceful-shutdown.md) · [Course Index](../README.md) · [18.7 Metrics with Prometheus →](07-prometheus.md)

# 18.6 — Health Checks and Readiness

## What You'll Learn

- Liveness = restart me, readiness = do not send me traffic, startup = still booting — conflating them causes outages
- `/healthz` checks **nothing** but the process; `/readyz` checks dependencies in parallel with per-check timeouts
- Hard versus soft dependencies: a failing cache **degrades** but stays ready; a failing database does not
- Caching check results so a health endpoint cannot be used to hammer your database, plus the build-info endpoint with real `-ldflags -X` values

## Coming From Another Language

| Other | Go |
|---|---|
| a `/health` route returning `{"ok": true}` | three distinct endpoints with three distinct meanings |
| Spring Boot Actuator's `/actuator/health` | you write it; 60 lines, and you control exactly what it checks |
| framework-managed readiness | an `atomic.Bool` you flip in your shutdown path (18.5) |
| `Promise.all` over checks | a `WaitGroup` over goroutines, each with its own `context.WithTimeout` |

## The three probes, and the outage from conflating them

| Probe | Question | Failure action | Should it check dependencies? |
|---|---|---|---|
| **liveness** | is this process wedged? | **kill and restart the container** | **no** |
| **readiness** | can this instance serve traffic right now? | remove from the load balancer | **yes** |
| **startup** | is it still booting? | keep waiting; suppress the other two | yes, but leniently |

The concrete failure story, which happens constantly: a team points `livenessProbe` at an
endpoint that pings Postgres. Postgres has a 20-second blip — a failover, a long lock, a
network hiccup. Every pod's liveness probe fails simultaneously, Kubernetes restarts **every
pod at once**, and the fleet comes back cold with empty connection pools and caches, hammering
the database that was already struggling. A 20-second degradation became a 10-minute outage,
and the cause was one URL in a YAML file.

The rule: **liveness must depend on nothing outside the process.** A dependency being down is
not a reason to restart you — restarting does not fix the dependency, and it loses your
in-flight requests. It is a reason to stop *receiving traffic*, which is readiness.

## /healthz: the liveness endpoint

```go
func (h *Health) Livez(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "text/plain; charset=utf-8")
  w.Header().Set("Cache-Control", "no-store")
  io.WriteString(w, "ok\n")
}
```

That is the correct implementation and it is not a simplification. If this handler runs, the
process is alive, the scheduler is scheduling, the listener is accepting, and the HTTP stack
works. There is nothing else liveness can usefully tell you. It catches exactly what it should:
a deadlocked process ([12.5](../12-concurrency-patterns/05-deadlocks-leaks.md)), a goroutine
leak that has exhausted memory, a listener that has stopped accepting.

If you want it to detect more, the honest version is a watchdog your own goroutines update:

```go
mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
  if time.Since(lastLoopTick.Load().(time.Time)) > 60*time.Second {
    http.Error(w, "event loop stalled", http.StatusServiceUnavailable)
    return
  }
  io.WriteString(w, "ok\n")
})
```

That is still process-internal, which is the test. "Can I reach Postgres" is not.

## /readyz: the complete implementation

A pluggable `Checker` interface, checks run in parallel with a per-check timeout, hard/soft
dependencies, and a JSON body with per-check status and duration.

```go
package main

import (
  "context"
  "encoding/json"
  "errors"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "sync"
  "sync/atomic"
  "time"
)

type Status string

const (
  StatusUp       Status = "up"
  StatusDegraded Status = "degraded"
  StatusDown     Status = "down"
)

type Checker interface {
  Name() string
  Critical() bool
  Check(ctx context.Context) error
}

type CheckFunc struct {
  CheckName string
  Hard      bool
  Fn        func(ctx context.Context) error
}

func (c CheckFunc) Name() string                    { return c.CheckName }
func (c CheckFunc) Critical() bool                  { return c.Hard }
func (c CheckFunc) Check(ctx context.Context) error { return c.Fn(ctx) }

type Result struct {
  Status     Status `json:"status"`
  Error      string `json:"error,omitempty"`
  Critical   bool   `json:"critical"`
  DurationMs int64  `json:"duration_ms"`
}

type Report struct {
  Status Status            `json:"status"`
  Checks map[string]Result `json:"checks"`
}

type Health struct {
  checks   []Checker
  timeout  time.Duration
  cacheTTL time.Duration
  clock    func() time.Time
  ready    atomic.Bool
  durFn    func(name string) time.Duration

  mu        sync.Mutex
  cached    *Report
  cachedAt  time.Time
  cacheHits atomic.Int64
  runs      atomic.Int64
}

func NewHealth(timeout, cacheTTL time.Duration, clock func() time.Time,
  durFn func(string) time.Duration, checks ...Checker) *Health {
  h := &Health{checks: checks, timeout: timeout, cacheTTL: cacheTTL,
    clock: clock, durFn: durFn}
  h.ready.Store(true)
  return h
}

func (h *Health) SetReady(v bool) { h.ready.Store(v) }

func (h *Health) Report(ctx context.Context) (*Report, bool) {
  h.mu.Lock()
  if h.cached != nil && h.clock().Sub(h.cachedAt) < h.cacheTTL {
    r := h.cached
    h.mu.Unlock()
    h.cacheHits.Add(1)
    return r, true
  }
  h.mu.Unlock()

  h.runs.Add(1)
  results := make([]Result, len(h.checks))
  var wg sync.WaitGroup
  for i, c := range h.checks {
    wg.Add(1)
    go func(i int, c Checker) {
      defer wg.Done()
      cctx, cancel := context.WithTimeout(ctx, h.timeout)
      defer cancel()
      err := c.Check(cctx)
      res := Result{Critical: c.Critical(), DurationMs: h.durFn(c.Name()).Milliseconds()}
      switch {
      case err == nil:
        res.Status = StatusUp
      case c.Critical():
        res.Status, res.Error = StatusDown, err.Error()
      default:
        res.Status, res.Error = StatusDegraded, err.Error()
      }
      results[i] = res
    }(i, c)
  }
  wg.Wait()

  rep := &Report{Status: StatusUp, Checks: make(map[string]Result, len(h.checks))}
  for i, c := range h.checks {
    rep.Checks[c.Name()] = results[i]
    switch results[i].Status {
    case StatusDown:
      rep.Status = StatusDown
    case StatusDegraded:
      if rep.Status == StatusUp {
        rep.Status = StatusDegraded
      }
    }
  }

  h.mu.Lock()
  h.cached, h.cachedAt = rep, h.clock()
  h.mu.Unlock()
  return rep, false
}

func (h *Health) Livez(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "text/plain; charset=utf-8")
  w.Header().Set("Cache-Control", "no-store")
  io.WriteString(w, "ok\n")
}

func (h *Health) Readyz(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "application/json")
  w.Header().Set("Cache-Control", "no-store")

  if !h.ready.Load() {
    w.WriteHeader(http.StatusServiceUnavailable)
    json.NewEncoder(w).Encode(Report{Status: StatusDown,
      Checks: map[string]Result{"shutdown": {Status: StatusDown, Critical: true}}})
    return
  }
  rep, hit := h.Report(r.Context())
  w.Header().Set("X-Health-Cache", map[bool]string{true: "hit", false: "miss"}[hit])
  if rep.Status == StatusDown {
    w.WriteHeader(http.StatusServiceUnavailable)
  }
  json.NewEncoder(w).Encode(rep)
}

type BuildInfo struct {
  Version   string `json:"version"`
  Commit    string `json:"commit"`
  BuildTime string `json:"build_time"`
  GoVersion string `json:"go_version"`
}

func main() {
  fixedNow := time.Date(2026, 1, 1, 12, 0, 0, 0, time.UTC)
  clock := func() time.Time { return fixedNow }
  durs := map[string]time.Duration{
    "postgres": 3 * time.Millisecond,
    "redis":    1 * time.Millisecond,
    "payments": 11 * time.Millisecond,
  }
  durFn := func(n string) time.Duration { return durs[n] }

  var dbErr, cacheErr, upstreamErr atomic.Pointer[error]
  setErr := func(p *atomic.Pointer[error], e error) {
    if e == nil {
      p.Store(nil)
      return
    }
    p.Store(&e)
  }
  load := func(p *atomic.Pointer[error]) error {
    if e := p.Load(); e != nil {
      return *e
    }
    return nil
  }

  db := CheckFunc{"postgres", true, func(ctx context.Context) error { return load(&dbErr) }}
  cache := CheckFunc{"redis", false, func(ctx context.Context) error { return load(&cacheErr) }}
  upstream := CheckFunc{"payments", false, func(ctx context.Context) error { return load(&upstreamErr) }}

  h := NewHealth(2*time.Second, 0, clock, durFn, db, cache, upstream)

  build := BuildInfo{Version: "1.4.2", Commit: "9f3a1c7", BuildTime: "2026-01-01T12:00:00Z",
    GoVersion: "go1.27.1"}

  mux := http.NewServeMux()
  mux.HandleFunc("GET /healthz", h.Livez)
  mux.HandleFunc("GET /readyz", h.Readyz)
  mux.HandleFunc("GET /startupz", func(w http.ResponseWriter, r *http.Request) {
    h.Readyz(w, r)
  })
  mux.HandleFunc("GET /buildinfo", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(build)
  })

  srv := httptest.NewServer(mux)
  defer srv.Close()

  show := func(label, path string) {
    resp, err := http.Get(srv.URL + path)
    if err != nil {
      fmt.Printf("%-26s client err %v\n", label, err)
      return
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    cache := resp.Header.Get("X-Health-Cache")
    if cache != "" {
      cache = "  cache=" + cache
    }
    fmt.Printf("%-26s %-8s %d%s\n   %s", label, path, resp.StatusCode, cache, b)
  }

  fmt.Println("=== all healthy ===")
  show("liveness", "/healthz")
  show("readiness", "/readyz")
  show("build info", "/buildinfo")

  fmt.Println("\n=== soft dependency failing: DEGRADED but still READY (200) ===")
  setErr(&cacheErr, errors.New("dial tcp 10.0.0.9:6379: connect: connection refused"))
  show("readiness", "/readyz")

  fmt.Println("\n=== hard dependency failing: DOWN and NOT ready (503) ===")
  setErr(&dbErr, errors.New("dial tcp 10.0.0.5:5432: connect: connection refused"))
  show("readiness", "/readyz")
  show("liveness still ok", "/healthz")

  fmt.Println("\n=== recovered ===")
  setErr(&dbErr, nil)
  setErr(&cacheErr, nil)
  show("readiness", "/readyz")

  fmt.Println("\n=== a check that exceeds its per-check timeout ===")
  slow := CheckFunc{"postgres", true, func(ctx context.Context) error {
    select {
    case <-time.After(time.Second):
      return nil
    case <-ctx.Done():
      return fmt.Errorf("ping: %w", ctx.Err())
    }
  }}
  h2 := NewHealth(50*time.Millisecond, 0, clock, durFn, slow, cache)
  mux2 := http.NewServeMux()
  mux2.HandleFunc("GET /readyz", h2.Readyz)
  srv2 := httptest.NewServer(mux2)
  defer srv2.Close()
  resp, _ := http.Get(srv2.URL + "/readyz")
  b, _ := io.ReadAll(resp.Body)
  resp.Body.Close()
  fmt.Printf("%-26s %-8s %d\n   %s", "readiness (timeout)", "/readyz", resp.StatusCode, b)

  fmt.Println("\n=== caching: 5 requests, TTL 5s, frozen clock ===")
  h3 := NewHealth(time.Second, 5*time.Second, clock, durFn, db, cache, upstream)
  mux3 := http.NewServeMux()
  mux3.HandleFunc("GET /readyz", h3.Readyz)
  srv3 := httptest.NewServer(mux3)
  defer srv3.Close()
  for i := 1; i <= 5; i++ {
    resp, _ := http.Get(srv3.URL + "/readyz")
    io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("  request %d -> %d  X-Health-Cache=%s\n", i, resp.StatusCode,
      resp.Header.Get("X-Health-Cache"))
  }
  fmt.Printf("  dependency checks actually run: %d (cache hits: %d)\n",
    h3.runs.Load(), h3.cacheHits.Load())

  fmt.Println("\n=== readiness flipped during shutdown (18.5) ===")
  h.SetReady(false)
  show("readiness while draining", "/readyz")
  show("liveness while draining", "/healthz")
}
```

The clock and the per-check durations are **injected** so the output is reproducible — in
production the duration is `time.Since(start)` per check. There is no Postgres or Redis in
this environment, so each check is a function whose error is set from the test driver; the
error strings are the genuine `net` dial-failure texts you would see.

Output:

```
=== all healthy ===
liveness                   /healthz 200
   ok
readiness                  /readyz  200  cache=miss
   {"status":"up","checks":{"payments":{"status":"up","critical":false,"duration_ms":11},"postgres":{"status":"up","critical":true,"duration_ms":3},"redis":{"status":"up","critical":false,"duration_ms":1}}}
build info                 /buildinfo 200
   {"version":"1.4.2","commit":"9f3a1c7","build_time":"2026-01-01T12:00:00Z","go_version":"go1.27.1"}

=== soft dependency failing: DEGRADED but still READY (200) ===
readiness                  /readyz  200  cache=miss
   {"status":"degraded","checks":{"payments":{"status":"up","critical":false,"duration_ms":11},"postgres":{"status":"up","critical":true,"duration_ms":3},"redis":{"status":"degraded","error":"dial tcp 10.0.0.9:6379: connect: connection refused","critical":false,"duration_ms":1}}}

=== hard dependency failing: DOWN and NOT ready (503) ===
readiness                  /readyz  503  cache=miss
   {"status":"down","checks":{"payments":{"status":"up","critical":false,"duration_ms":11},"postgres":{"status":"down","error":"dial tcp 10.0.0.5:5432: connect: connection refused","critical":true,"duration_ms":3},"redis":{"status":"degraded","error":"dial tcp 10.0.0.9:6379: connect: connection refused","critical":false,"duration_ms":1}}}
liveness still ok          /healthz 200
   ok

=== recovered ===
readiness                  /readyz  200  cache=miss
   {"status":"up","checks":{"payments":{"status":"up","critical":false,"duration_ms":11},"postgres":{"status":"up","critical":true,"duration_ms":3},"redis":{"status":"up","critical":false,"duration_ms":1}}}

=== a check that exceeds its per-check timeout ===
readiness (timeout)        /readyz  503
   {"status":"down","checks":{"postgres":{"status":"down","error":"ping: context deadline exceeded","critical":true,"duration_ms":3},"redis":{"status":"up","critical":false,"duration_ms":1}}}

=== caching: 5 requests, TTL 5s, frozen clock ===
  request 1 -> 200  X-Health-Cache=miss
  request 2 -> 200  X-Health-Cache=hit
  request 3 -> 200  X-Health-Cache=hit
  request 4 -> 200  X-Health-Cache=hit
  request 5 -> 200  X-Health-Cache=hit
  dependency checks actually run: 1 (cache hits: 4)

=== readiness flipped during shutdown (18.5) ===
readiness while draining   /readyz  503
   {"status":"down","checks":{"shutdown":{"status":"down","critical":true,"duration_ms":0}}}
liveness while draining    /healthz 200
   ok
```

### Hard versus soft, made a property of the check

The two middle sections are the design payoff. Redis down gave **200 with
`"status":"degraded"`**; Postgres down gave **503 with `"status":"down"`**. Both appear in the
same body with their own per-check status, so an operator looking at the JSON sees exactly
what is wrong even when the aggregate is fine.

Criticality is `Critical() bool` on the `Checker`, not a decision the aggregator makes, which
means the person adding a dependency declares its criticality at the point of adding it. The
question to ask is: **can I serve a useful response without this?** If the cache is down you
serve slower; if the database is down you serve nothing. Getting this wrong in the strict
direction is expensive — marking an optional analytics upstream as critical means its outage
takes your entire fleet out of the load balancer, which is a self-inflicted outage.

`degraded` is also the state worth alerting on at low urgency: it never pulls a pod from the
pool, so without an alert nobody notices Redis has been down for a week.

### Parallel, with per-check timeouts

Each check gets its own `context.WithTimeout` and its own goroutine, so three checks take as
long as the slowest, not the sum. The timeout matters more than it looks: without it a
`conn.Ping` against a host that is dropping packets (rather than refusing connections) blocks
for the OS TCP timeout — minutes — and your readiness probe itself times out, which looks
identical to the process being wedged.

The timeout output shows the mechanism working: `"error":"ping: context deadline exceeded"`,
with `redis` still reported `up` because its check ran concurrently and was unaffected.

Note `results[i] = res` with each goroutine writing its own index: distinct slice elements, no
mutex needed ([12.6](../12-concurrency-patterns/06-memory-model.md)). Also note the readiness
handler passes `r.Context()` as the parent, so a probe that gives up cancels the checks it
started instead of leaving them running.

### Caching, so the probe cannot become an attack

Five requests, one actual round of dependency checks. With a 5-second TTL and a probe every 5
seconds you do roughly one real check per interval — but `/readyz` is usually reachable, and
without the cache anyone (or any misconfigured monitoring system) can issue thousands of
requests per second and turn your health endpoint into a database load generator. One to five
seconds of TTL costs you nothing in probe accuracy, since `periodSeconds` is already in that
range.

The cache deliberately stores the whole `*Report` under a mutex and returns the shared
immutable pointer, so concurrent probes never run overlapping check rounds.

### Readiness flipped during shutdown

The last section is 18.5's integration: `SetReady(false)` makes `/readyz` return 503 while
`/healthz` keeps returning 200. That distinction is load-bearing — a draining pod must be
removed from the load balancer and must **not** be restarted. If your shutdown failed liveness,
Kubernetes would kill the pod mid-drain and you would lose the in-flight requests the whole
shutdown dance exists to protect.

## Exclude the health endpoints from auth, logging and metrics

Three exclusions, each for a concrete reason:

```go
r := chi.NewRouter()

r.Group(func(r chi.Router) {
  r.Get("/healthz", h.Livez)
  r.Get("/readyz", h.Readyz)
})

r.Group(func(r chi.Router) {
  r.Use(RequestLogger(log), Metrics(reg), auth.Middleware)
  r.Mount("/v1", apiRoutes())
})
```

**No auth**, because the kubelet cannot present a bearer token and will not follow your login
flow. These endpoints must be unauthenticated — which is why their bodies must not leak
anything: note that the output above exposes `dial tcp 10.0.0.5:5432` on the DB failure.
Internal hostnames and ports in a public readiness body are an information leak; on an
internet-facing service, return per-check status without the error string, or bind the probes
to a second listener on a port that is not routed externally. That second-listener pattern is
the cleanest answer and also solves the next two problems.

**No request logging**, because a probe every 5 seconds per pod is 17,000 log lines per pod per
day that say nothing. Filter them out, or you pay to store them.

**No metrics**, because `/healthz` and `/readyz` would otherwise appear as routes in your
`http_requests_total` series and dominate your request-rate graphs
([18.7](07-prometheus.md)).

**Must be cheap, and must not cascade.** A readiness check that calls another service's
readiness check builds a chain where one leaf failure takes down everything upstream — the
classic cascading outage. Check your *own* ability to serve: can I reach my database, is my
queue connection alive. Do not ask your dependencies whether they consider themselves healthy.

## Build info, with real ldflags values

Knowing which build is in a pod is worth an endpoint. Inject at link time
([15.1](../15-tooling/01-build-run-install.md)):

```go
package main

import (
  "encoding/json"
  "os"
  "runtime"
  "runtime/debug"
)

var (
  version   = "dev"
  commit    = "none"
  buildTime = "unknown"
)

type BuildInfo struct {
  Version   string `json:"version"`
  Commit    string `json:"commit"`
  BuildTime string `json:"build_time"`
  GoVersion string `json:"go_version"`
  VCSRev    string `json:"vcs_revision,omitempty"`
  VCSDirty  string `json:"vcs_dirty,omitempty"`
}

func Info() BuildInfo {
  bi := BuildInfo{
    Version:   version,
    Commit:    commit,
    BuildTime: buildTime,
    GoVersion: runtime.Version(),
  }
  if di, ok := debug.ReadBuildInfo(); ok {
    for _, s := range di.Settings {
      switch s.Key {
      case "vcs.revision":
        bi.VCSRev = s.Value
      case "vcs.modified":
        bi.VCSDirty = s.Value
      }
    }
  }
  return bi
}

func main() {
  enc := json.NewEncoder(os.Stdout)
  enc.SetIndent("", "  ")
  enc.Encode(Info())
}
```

Without flags, the defaults:

Output:

```
{
  "version": "dev",
  "commit": "none",
  "build_time": "unknown",
  "go_version": "go1.27.1"
}
```

With `go run -ldflags="-X main.version=1.4.2 -X main.commit=9f3a1c7 -X main.buildTime=2026-01-01T12:00:00Z" .`:

Output:

```
{
  "version": "1.4.2",
  "commit": "9f3a1c7",
  "build_time": "2026-01-01T12:00:00Z",
  "go_version": "go1.27.1"
}
```

`-X importpath.name=value` only works on a **package-level `string` variable** — not a
constant, not an int, not a struct field. The build command in CI
([18.9](09-ci.md), [18.8](08-docker.md)):

```bash
go build -trimpath -ldflags="-s -w \
  -X main.version=$(git describe --tags --always) \
  -X main.commit=$(git rev-parse --short HEAD) \
  -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -o server ./cmd/server
```

`debug.ReadBuildInfo()` gives you some of this for free — `vcs.revision`, `vcs.modified`,
module versions — with no flags at all, as long as you build from inside a git working tree
(`-buildvcs` is on by default). It is empty here because the scratch module is not a git
repository, which is also why it is empty inside a Docker build that copies only the source.
Explicit `-ldflags -X` is the version that works everywhere, which is why both appear above.

Expose it at `/buildinfo`, log it once at startup, and emit it as a metric
(`build_info{version="1.4.2",commit="9f3a1c7"} 1`) so a dashboard can correlate a latency
change with a deploy.

## Kubernetes probe YAML

Real manifest, verified to parse as YAML. Not applied here — there is no cluster in this
environment.

Expected usage (requires a Kubernetes cluster; not executed in this environment):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: server
          image: ghcr.io/acme/orders:1.4.2
          ports:
            - containerPort: 8080
          startupProbe:
            httpGet:
              path: /startupz
              port: 8080
            periodSeconds: 2
            timeoutSeconds: 2
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 2
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
```

| Field | Meaning | How to choose it |
|---|---|---|
| `initialDelaySeconds` | wait this long before the first probe | **0** if you have a `startupProbe`; that is what it is for |
| `periodSeconds` | interval between probes | readiness 5s (fast removal), liveness 10s (restarting is expensive) |
| `timeoutSeconds` | probe gives up after this | **above** your per-check timeout, or the probe times out before your handler answers |
| `successThreshold` | consecutive passes to be ready again | 1 for readiness; must be 1 for liveness |
| `failureThreshold` | consecutive failures before acting | readiness 2 (remove quickly), liveness 3 (do not restart on a blip) |

The relationships that matter:

**`startupProbe` makes `initialDelaySeconds` obsolete.** `periodSeconds: 2` ×
`failureThreshold: 30` gives a 60-second boot budget, and while the startup probe is running
the liveness and readiness probes are **suppressed entirely**. Without it you must pick an
`initialDelaySeconds` that is long enough for the slowest cold start and therefore wastes that
time on every fast one — or, worse, is too short and liveness kills the pod before it finishes
booting, forever.

**Liveness `failureThreshold × periodSeconds` is your restart latency.** 3 × 10s = 30 seconds
of wedged process before a restart. Lower it and a GC pause or a slow disk gets you restarted;
raise it and a real deadlock persists longer.

**Readiness `failureThreshold × periodSeconds` sets your pre-stop delay.** 2 × 5s = 10 seconds
is the worst case for a load balancer to notice you are draining, so 18.5's `PreStopDelay`
must exceed it. This is the single most common cause of 502s during deploys in an otherwise
correct service.

**`terminationGracePeriodSeconds` ≥ `PreStopDelay` + `ShutdownTimeout`.** 45 ≥ 10 + 25, with
room to spare.

## Quick Reference

| Endpoint | Checks | Returns | Who calls it |
|---|---|---|---|
| `/healthz` | nothing beyond the process | 200 always (or 503 if a watchdog stalled) | kubelet liveness |
| `/readyz` | all dependencies, in parallel | 200 up/degraded, 503 down | kubelet readiness, load balancer |
| `/startupz` | dependencies, leniently | 200 / 503 | kubelet startup |
| `/buildinfo` | — | version, commit, build time | humans, deploy tooling |

| Piece | Shape |
|---|---|
| a check | `Name() string`, `Critical() bool`, `Check(ctx) error` |
| per-check timeout | `context.WithTimeout(ctx, h.timeout)` inside each goroutine |
| parallel execution | `sync.WaitGroup`, each goroutine writing `results[i]` |
| aggregation | any critical down → `down`; any soft down → `degraded`; else `up` |
| readiness flag | `atomic.Bool`, flipped in the shutdown path (18.5) |
| result cache | `*Report` + timestamp under a mutex, 1-5s TTL |
| version injection | `-ldflags "-X main.version=..."` on a package-level `string` |

## Common Mistakes

**Checking the database in the liveness probe.** A database blip restarts every pod
simultaneously and turns a short degradation into an outage; liveness checks only the process.

**One `/health` endpoint used for both probes.** You get either liveness that restarts on
dependency failures or readiness that never removes a broken pod; they answer different
questions.

**No per-check timeout.** A dependency dropping packets blocks the check for the OS TCP timeout
and the probe itself times out; wrap each check in `context.WithTimeout`.

**Marking every dependency critical.** An optional upstream's outage pulls your whole fleet
from the load balancer; make criticality a property of each check.

**No caching on `/readyz`.** The endpoint is unauthenticated, so anyone can use it to load-test
your database; cache the report for a few seconds.

**Requiring auth on the probes.** The kubelet cannot authenticate, so every probe fails and
the pod never becomes ready.

**Returning dependency error strings on a public probe.** You leak internal hostnames and
ports; omit them, or serve the probes on a non-routed port.

**A readiness check that calls a dependency's readiness check.** One leaf failure cascades
upward and takes down everything; check only your own ability to serve.

**`timeoutSeconds` lower than your per-check timeout.** The probe gives up before your handler
answers, so a slow dependency looks like a wedged process.

## Exercises

1. Write real `Checker` implementations for `pgxpool.Pool` (`Ping`), an HTTP upstream (a `GET`
   on its `/healthz` with a 1-second timeout), and free disk space via `syscall.Statfs`. Mark
   them hard/soft and show the three-check report for every combination of up and down.
2. Add a `Checker` whose `Check` returns an error for the first 5 seconds after process start,
   wire it only into `/startupz`, and prove with `httptest` that `/startupz` goes 503 → 200
   while `/healthz` is 200 throughout.
3. Extend the cache so a **failing** report is cached for 1 second and a **passing** report for
   5. Explain in one sentence why the asymmetry is the right default, and test both TTLs with
   an injected clock.
4. Make `/readyz` support `?verbose=0` (status only, no per-check detail) and have it omit error
   strings unless the request comes from a private IP range. Test both forms.
5. Serve `/healthz`, `/readyz`, `/metrics` and `/debug/pprof` on a **second** `http.Server` on
   port 9090 with its own mux, and shut both servers down in the right order on `SIGTERM`
   (18.5). Prove the admin server stops accepting only after the main one has drained.

---

[← 18.5 Graceful Shutdown](05-graceful-shutdown.md) · [Course Index](../README.md) · [18.7 Metrics with Prometheus →](07-prometheus.md)
