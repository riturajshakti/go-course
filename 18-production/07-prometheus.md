[← 18.6 Health Checks and Readiness](06-health-checks.md) · [Course Index](../README.md) · [18.8 Docker Multi-Stage Builds →](08-docker.md)

# 18.7 — Metrics with Prometheus

## What You'll Learn

- The four metric types, and why you almost always want a Histogram
- A **custom registry** instead of the global default, and why that matters for tests
- **Cardinality** — the one operational lesson that actually takes systems down, shown as 1000 time series versus 1
- The golden-signals middleware recording `http_requests_total` and `http_request_duration_seconds`, with the real `/metrics` text exposition

## Coming From Another Language

| Other | Go |
|---|---|
| `prom-client` with a default global registry | `prometheus.NewRegistry()` of your own; the global is a convenience, not the design |
| StatsD: fire-and-forget UDP, server aggregates | **pull** model: you hold the numbers in memory and Prometheus scrapes `/metrics` |
| `histogram.observe(ms)` | `Observe(seconds)` — base units, always seconds |
| a decorator that times a function | a `func(http.Handler) http.Handler` reusing 16.6's `ResponseWriter` wrapper |
| dynamic tags from request data | a fixed label **set** declared up front; a dynamic label is a cardinality bomb |

## The four metric types

| Type | Can it go down? | Use for | Exposed as |
|---|---|---|---|
| **Counter** | no, monotonically increasing | requests served, errors, bytes written, jobs processed | one series, `_total` suffix |
| **Gauge** | yes | in-flight requests, pool connections, queue depth, temperature | one series |
| **Histogram** | n/a | distributions: latency, response size | `_bucket` per boundary + `_sum` + `_count` |
| **Summary** | n/a | distributions with **client-side** quantiles | `_sum`, `_count`, `{quantile="..."}` |

**Prefer Histogram over Summary.** A Summary computes quantiles per process, and quantiles
cannot be averaged — so you can never answer "p99 across all ten pods", which is the question
you actually have. A Histogram ships bucket counts, which *are* addable, so
`histogram_quantile()` works across any aggregation. A counter resetting to zero on restart is
fine: `rate()` detects it — which is why a counter must never be decremented, or `rate()` sees
a reset that did not happen.

## A custom registry, not the global default

```go
reg := prometheus.NewRegistry()
reg.MustRegister(m.Requests, m.Duration, m.InFlight)
admin.Handle("GET /metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{}))
```

Two concrete reasons. **Test isolation:** metrics on `prometheus.DefaultRegisterer` are
process-global, so two tests constructing your metrics panic on the second `MustRegister`, and a
test asserting a counter equals 3 fails because another test already incremented it. **No
surprise collectors:** `promhttp.Handler()` serves the default registry, which already holds the
Go and process collectors plus whatever any transitive dependency registered at `init` time.

`promauto` registers on construction, which hides the registry; bind it to yours with
`promauto.With(reg)` rather than the package-level functions.

```go
package main

import (
  "errors"
  "fmt"

  "github.com/prometheus/client_golang/prometheus"
  "github.com/prometheus/client_golang/prometheus/promauto"
)

func main() {
  fmt.Println("prometheus.DefBuckets =", prometheus.DefBuckets)
  fmt.Println("ExponentialBuckets(0.001, 4, 7) =", prometheus.ExponentialBuckets(0.001, 4, 7))
  fmt.Println("LinearBuckets(0, 0.05, 6) =", prometheus.LinearBuckets(0, 0.05, 6))

  fmt.Println("\n=== registering the same metric twice on one registry ===")
  reg := prometheus.NewRegistry()
  c1 := prometheus.NewCounter(prometheus.CounterOpts{Name: "orders_total", Help: "h"})
  c2 := prometheus.NewCounter(prometheus.CounterOpts{Name: "orders_total", Help: "h"})
  fmt.Println("first  Register:", reg.Register(c1))
  err := reg.Register(c2)
  fmt.Println("second Register:", err)
  var dup prometheus.AlreadyRegisteredError
  fmt.Println("is AlreadyRegisteredError:", errors.As(err, &dup))
  func() {
    defer func() { fmt.Println("MustRegister on a duplicate panics:", recover()) }()
    reg.MustRegister(c2)
  }()

  fmt.Println("\n=== promauto bound to YOUR registry, not the global default ===")
  reg2 := prometheus.NewRegistry()
  orders := promauto.With(reg2).NewCounterVec(prometheus.CounterOpts{
    Name: "orders_created_total",
    Help: "Orders created.",
  }, []string{"plan"})
  orders.WithLabelValues("free").Inc()
  orders.WithLabelValues("pro").Add(3)
  fmt.Println("time series on reg2:", countSeries(reg2))

  fmt.Println("\n=== an inconsistent label cardinality is a runtime error ===")
  v := prometheus.NewCounterVec(prometheus.CounterOpts{Name: "x_total", Help: "h"},
    []string{"a", "b"})
  _, err = v.GetMetricWithLabelValues("only-one")
  fmt.Println(" ", err)
}

func countSeries(reg *prometheus.Registry) int {
  mf, _ := reg.Gather()
  n := 0
  for _, f := range mf {
    n += len(f.GetMetric())
  }
  return n
}
```

Output:

```
prometheus.DefBuckets = [0.005 0.01 0.025 0.05 0.1 0.25 0.5 1 2.5 5 10]
ExponentialBuckets(0.001, 4, 7) = [0.001 0.004 0.016 0.064 0.256 1.024 4.096]
LinearBuckets(0, 0.05, 6) = [0 0.05 0.1 0.15000000000000002 0.2 0.25]

=== registering the same metric twice on one registry ===
first  Register: <nil>
second Register: duplicate metrics collector registration attempted
is AlreadyRegisteredError: true
MustRegister on a duplicate panics: duplicate metrics collector registration attempted

=== promauto bound to YOUR registry, not the global default ===
time series on reg2: 2

=== an inconsistent label cardinality is a runtime error ===
  inconsistent label cardinality: expected 2 label values but got 1 in []string{"only-one"}
```

Note the floating-point artefact in `LinearBuckets`: `0.15000000000000002`. Bucket boundaries
are `float64`, so prefer explicit literal slices for latency buckets where the exact boundary
appears in your queries ([2.2](../02-syntax-basics/02-numeric-types.md)).
`ExponentialBucketsRange(min, max, n)` spreads `n` buckets geometrically between two values.

`WithLabelValues` **panics** on the wrong number of labels; `GetMetricWithLabelValues` returns
the error above instead. Use the panicking form in middleware where the label count is a
constant you can see, and the error form when the labels come from a map.

## Naming conventions

| Rule | Right | Wrong |
|---|---|---|
| `_total` suffix on counters | `http_requests_total` | `http_requests` |
| base units: **seconds**, not ms | `http_request_duration_seconds` | `http_request_duration_ms` |
| base units: **bytes**, not KB/MB | `response_size_bytes` | `response_size_kb` |
| unit suffix, singular | `..._seconds`, `..._bytes` | `..._secs`, `..._byte_count` |
| `snake_case`, app prefix | `orders_created_total` | `OrdersCreated` |
| no label value in the name | `http_requests_total{status="500"}` | `http_500_requests_total` |

These are not aesthetics. Every Grafana dashboard and alerting-rule library assumes them:
`histogram_quantile` returns milliseconds labelled as seconds if you store milliseconds, and a
community dashboard silently shows nothing. Store seconds as a float — `0.0012` is perfectly
precise — and let the UI format it.

## Cardinality: the lesson that matters

A **time series** is one metric name plus one unique combination of label values. Prometheus
holds every active series in memory. Cardinality is the **product** of each label's distinct
value count, so one careless label multiplies your entire series count.

```go
package main

import (
  "fmt"
  "strconv"

  "github.com/prometheus/client_golang/prometheus"
  "github.com/prometheus/client_golang/prometheus/testutil"
)

func main() {
  bad := prometheus.NewCounterVec(prometheus.CounterOpts{
    Name: "http_requests_total",
    Help: "bad labels",
  }, []string{"method", "path", "user_id", "status"})
  good := prometheus.NewCounterVec(prometheus.CounterOpts{
    Name: "http_requests_total",
    Help: "good labels",
  }, []string{"method", "route", "status"})

  badReg, goodReg := prometheus.NewRegistry(), prometheus.NewRegistry()
  badReg.MustRegister(bad)
  goodReg.MustRegister(good)

  for user := 1; user <= 200; user++ {
    for order := 1; order <= 5; order++ {
      bad.WithLabelValues("GET", "/orders/"+strconv.Itoa(order*1000+user),
        strconv.Itoa(user), "200").Inc()
      good.WithLabelValues("GET", "/orders/{id}", "200").Inc()
    }
  }

  fmt.Println("same 1000 requests, two label schemes:")
  fmt.Printf("  raw path + user_id  -> %d time series\n", testutil.CollectAndCount(bad))
  fmt.Printf("  route pattern only  -> %d time series\n", testutil.CollectAndCount(good))
}
```

Output:

```
same 1000 requests, two label schemes:
  raw path + user_id  -> 1000 time series
  route pattern only  -> 1 time series
```

**1000 series versus 1, for the identical traffic.** That is the whole lesson, and it compounds:
cardinality is the **product** of each label's distinct value count, so `method` (5) × `route`
(40) × `status` (8) is a comfortable 1,600 series, while swapping `route` for a raw path with IDs
in it and adding `user_id` turns the same metric into billions. Each of those
series costs memory in your Prometheus server forever (until it ages out), and none of them is
useful — a counter with one observation tells you nothing about a rate.

Labels that must **never** carry unbounded values: user ID, order ID, email, session ID, trace
ID, raw URL path, raw error message, timestamp, SQL query text. If the possible values are not a
short list you could write down, it is not a label — that belongs in a **log line**
([18.4](04-service-logging.md)) or a **trace**, both designed for high-cardinality data.

The fix for paths is to use the **route pattern**, which chi exposes:

```go
route := chi.RouteContext(r.Context()).RoutePattern()
if route == "" {
  route = "unmatched"
}
```

`RoutePattern()` returns `/orders/{id}`, not `/orders/12345` ([16.7](../16-http/07-chi.md)). The
`""` fallback matters: an unmatched request has no pattern, and labelling it with `r.URL.Path`
reintroduces unbounded cardinality through your 404s — exactly where a scanner hitting random
URLs attacks it. With the stdlib mux, `mux.Handler(r)` returns the matched pattern
([16.2](../16-http/02-servemux-routing.md)).

## The golden signals middleware

Rate, errors and duration all fall out of two metrics plus a gauge for saturation. This is the
centerpiece of the lesson, and the `/metrics` output below is real.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strconv"
  "strings"
  "time"

  "github.com/go-chi/chi/v5"
  "github.com/prometheus/client_golang/prometheus"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

type Metrics struct {
  Requests  *prometheus.CounterVec
  Duration  *prometheus.HistogramVec
  InFlight  prometheus.Gauge
  Registry  *prometheus.Registry
  BuildInfo *prometheus.GaugeVec
}

func NewMetrics() *Metrics {
  reg := prometheus.NewRegistry()
  m := &Metrics{
    Registry: reg,
    Requests: prometheus.NewCounterVec(prometheus.CounterOpts{
      Name: "http_requests_total",
      Help: "Total number of HTTP requests.",
    }, []string{"method", "route", "status"}),
    Duration: prometheus.NewHistogramVec(prometheus.HistogramOpts{
      Name:    "http_request_duration_seconds",
      Help:    "HTTP request latency in seconds.",
      Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1, 2.5},
    }, []string{"method", "route"}),
    InFlight: prometheus.NewGauge(prometheus.GaugeOpts{
      Name: "http_requests_in_flight",
      Help: "Number of HTTP requests currently being served.",
    }),
    BuildInfo: prometheus.NewGaugeVec(prometheus.GaugeOpts{
      Name: "build_info",
      Help: "Build information; always 1.",
    }, []string{"version", "commit", "goversion"}),
  }
  reg.MustRegister(m.Requests, m.Duration, m.InFlight, m.BuildInfo)
  return m
}

func (m *Metrics) Middleware(elapsed func(route string) time.Duration) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      m.InFlight.Inc()
      defer m.InFlight.Dec()

      rw := &recorder{ResponseWriter: w}
      next.ServeHTTP(rw, r)

      route := chi.RouteContext(r.Context()).RoutePattern()
      if route == "" {
        route = "unmatched"
      }
      m.Requests.WithLabelValues(r.Method, route, strconv.Itoa(rw.status)).Inc()
      m.Duration.WithLabelValues(r.Method, route).Observe(elapsed(route).Seconds())
    })
  }
}

func main() {
  m := NewMetrics()
  m.BuildInfo.WithLabelValues("1.4.2", "9f3a1c7", "go1.27.1").Set(1)

  fixed := map[string]time.Duration{
    "/orders/{id}": 8 * time.Millisecond,
    "/orders":      140 * time.Millisecond,
    "/slow":        1200 * time.Millisecond,
    "/boom":        3 * time.Millisecond,
    "unmatched":    1 * time.Millisecond,
  }
  elapsed := func(route string) time.Duration { return fixed[route] }

  r := chi.NewRouter()
  r.Use(m.Middleware(elapsed))
  r.Get("/orders/{id}", func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, `{"id":"`+chi.URLParam(r, "id")+`"}`)
  })
  r.Post("/orders", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusCreated)
    io.WriteString(w, `{"id":"ord_new"}`)
  })
  r.Get("/slow", func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "slow")
  })
  r.Get("/boom", func(w http.ResponseWriter, r *http.Request) {
    http.Error(w, "boom", http.StatusInternalServerError)
  })

  admin := http.NewServeMux()
  admin.Handle("GET /metrics", promhttp.HandlerFor(m.Registry, promhttp.HandlerOpts{}))

  api := httptest.NewServer(r)
  defer api.Close()
  adm := httptest.NewServer(admin)
  defer adm.Close()

  type req struct{ method, path string }
  traffic := []req{
    {"GET", "/orders/1"}, {"GET", "/orders/2"}, {"GET", "/orders/3"},
    {"GET", "/orders/4"}, {"GET", "/orders/5"},
    {"POST", "/orders"}, {"POST", "/orders"},
    {"GET", "/slow"},
    {"GET", "/boom"}, {"GET", "/boom"},
    {"GET", "/does-not-exist"},
  }
  for _, q := range traffic {
    rq, _ := http.NewRequest(q.method, api.URL+q.path, strings.NewReader("{}"))
    resp, err := http.DefaultClient.Do(rq)
    if err != nil {
      fmt.Println("client err", err)
      continue
    }
    io.ReadAll(resp.Body)
    resp.Body.Close()
  }
  fmt.Printf("drove %d requests across 5 route patterns\n\n", len(traffic))

  resp, err := http.Get(adm.URL + "/metrics")
  if err != nil {
    panic(err)
  }
  body, _ := io.ReadAll(resp.Body)
  resp.Body.Close()
  fmt.Printf("GET /metrics -> %s  Content-Type: %s\n\n", resp.Status, resp.Header.Get("Content-Type"))
  fmt.Print(string(body))
}
```

The `recorder` `ResponseWriter` wrapper is 16.6's, unchanged and in a second file — the
`WriteHeader`/`Write`/`Unwrap` trio that captures the status code
([16.6](../16-http/06-middleware.md)). The elapsed-time function is **injected** with a fixed
duration per route so the histogram buckets and `_sum` values are reproducible; in production it
is `time.Since(start)`.

Output:

```
drove 11 requests across 5 route patterns

GET /metrics -> 200 OK  Content-Type: text/plain; version=0.0.4; charset=utf-8; escaping=underscores

# HELP build_info Build information; always 1.
# TYPE build_info gauge
build_info{commit="9f3a1c7",goversion="go1.27.1",version="1.4.2"} 1
# HELP http_request_duration_seconds HTTP request latency in seconds.
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",route="/boom",le="0.01"} 2
http_request_duration_seconds_bucket{method="GET",route="/boom",le="0.05"} 2
http_request_duration_seconds_bucket{method="GET",route="/boom",le="0.1"} 2
http_request_duration_seconds_bucket{method="GET",route="/boom",le="0.5"} 2
http_request_duration_seconds_bucket{method="GET",route="/boom",le="1"} 2
http_request_duration_seconds_bucket{method="GET",route="/boom",le="2.5"} 2
http_request_duration_seconds_bucket{method="GET",route="/boom",le="+Inf"} 2
http_request_duration_seconds_sum{method="GET",route="/boom"} 0.006
http_request_duration_seconds_count{method="GET",route="/boom"} 2
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="0.01"} 5
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="0.05"} 5
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="0.1"} 5
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="0.5"} 5
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="1"} 5
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="2.5"} 5
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="+Inf"} 5
http_request_duration_seconds_sum{method="GET",route="/orders/{id}"} 0.04
http_request_duration_seconds_count{method="GET",route="/orders/{id}"} 5
http_request_duration_seconds_bucket{method="GET",route="/slow",le="0.01"} 0
http_request_duration_seconds_bucket{method="GET",route="/slow",le="0.05"} 0
http_request_duration_seconds_bucket{method="GET",route="/slow",le="0.1"} 0
http_request_duration_seconds_bucket{method="GET",route="/slow",le="0.5"} 0
http_request_duration_seconds_bucket{method="GET",route="/slow",le="1"} 0
http_request_duration_seconds_bucket{method="GET",route="/slow",le="2.5"} 1
http_request_duration_seconds_bucket{method="GET",route="/slow",le="+Inf"} 1
http_request_duration_seconds_sum{method="GET",route="/slow"} 1.2
http_request_duration_seconds_count{method="GET",route="/slow"} 1
http_request_duration_seconds_bucket{method="GET",route="unmatched",le="0.01"} 1
http_request_duration_seconds_bucket{method="GET",route="unmatched",le="0.05"} 1
http_request_duration_seconds_bucket{method="GET",route="unmatched",le="0.1"} 1
http_request_duration_seconds_bucket{method="GET",route="unmatched",le="0.5"} 1
http_request_duration_seconds_bucket{method="GET",route="unmatched",le="1"} 1
http_request_duration_seconds_bucket{method="GET",route="unmatched",le="2.5"} 1
http_request_duration_seconds_bucket{method="GET",route="unmatched",le="+Inf"} 1
http_request_duration_seconds_sum{method="GET",route="unmatched"} 0.001
http_request_duration_seconds_count{method="GET",route="unmatched"} 1
http_request_duration_seconds_bucket{method="POST",route="/orders",le="0.01"} 0
http_request_duration_seconds_bucket{method="POST",route="/orders",le="0.05"} 0
http_request_duration_seconds_bucket{method="POST",route="/orders",le="0.1"} 0
http_request_duration_seconds_bucket{method="POST",route="/orders",le="0.5"} 2
http_request_duration_seconds_bucket{method="POST",route="/orders",le="1"} 2
http_request_duration_seconds_bucket{method="POST",route="/orders",le="2.5"} 2
http_request_duration_seconds_bucket{method="POST",route="/orders",le="+Inf"} 2
http_request_duration_seconds_sum{method="POST",route="/orders"} 0.28
http_request_duration_seconds_count{method="POST",route="/orders"} 2
# HELP http_requests_in_flight Number of HTTP requests currently being served.
# TYPE http_requests_in_flight gauge
http_requests_in_flight 0
# HELP http_requests_total Total number of HTTP requests.
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/boom",status="500"} 2
http_requests_total{method="GET",route="/orders/{id}",status="200"} 5
http_requests_total{method="GET",route="/slow",status="200"} 1
http_requests_total{method="GET",route="unmatched",status="404"} 1
http_requests_total{method="POST",route="/orders",status="201"} 2
```

### Reading that exposition

**Buckets are cumulative, and `le` means "less than or equal".** Look at `/slow`, whose single
observation was 1.2 seconds: every bucket from `le="0.01"` through `le="1"` is **0**, and
`le="2.5"` and `le="+Inf"` are both **1**. Each bucket counts observations *at or below* its
boundary, so counts are non-decreasing and the top one equals `_count`. A bucket is **not** a
count of observations *in* that range — subtract adjacent buckets for that, which is what
`histogram_quantile()` does internally.

**`/orders/{id}` shows 5 requests collapsed into one series.** Five distinct URLs, one route
label — the cardinality fix working in the real middleware. And `/does-not-exist` landed on
`route="unmatched", status="404"`, bounded.

**`_sum` and `_count` are the cheap average.** `0.04 / 5 = 8ms` for `/orders/{id}`. The average
is a poor latency statistic but a fine sanity check, and `rate(..._sum[5m]) /
rate(..._count[5m])` is the query for it.

**`http_requests_in_flight` is 0** because every request finished — `defer m.InFlight.Dec()`
doing its job. This is the "saturation" signal, a gauge precisely because it goes down too.

**`build_info` is a gauge fixed at 1** whose value is irrelevant; the *labels* are the payload,
so a dashboard can annotate "latency rose at the 1.4.2 deploy"
([18.6](06-health-checks.md) has the `-ldflags` injection). The content type
`text/plain; version=0.0.4` is the Prometheus text exposition format.

### What you query it with

| Question | PromQL |
|---|---|
| request **rate** | `sum(rate(http_requests_total[5m])) by (route)` |
| **error** ratio | `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| p99 **duration** | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))` |
| **saturation** | `http_requests_in_flight` |

`rate()` turns a counter into a per-second rate and handles restarts; `histogram_quantile()`
interpolates within a bucket from the cumulative `_bucket` series — which is why buckets must
`sum by (le)` first, and why summaries cannot do this at all.

## Choosing latency buckets

`prometheus.DefBuckets` is `[.005 .01 .025 .05 .1 .25 .5 1 2.5 5 10]` — eleven buckets from 5ms
to 10s, a reasonable web-service default. Tune it when your SLO boundary is not in the list:
promise p99 under 300ms and you need a boundary **at** 0.3, because `histogram_quantile`
interpolates linearly inside a bucket.

| Workload | Buckets |
|---|---|
| in-memory cache, sub-ms | `{0.0001, 0.00025, 0.0005, 0.001, 0.0025, 0.005, 0.01}` |
| typical JSON API | `DefBuckets`, or add your SLO boundary |
| database query | `{0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5}` |
| batch job, minutes | `prometheus.ExponentialBuckets(1, 2, 12)` |

Each bucket is a series per label combination, so eleven buckets × 40 routes × 5 methods is
2,200 series from one histogram. Ten to fifteen buckets is the practical range.

Native histograms (experimental) replace fixed buckets with a sparse exponential schema and
remove the choice entirely — worth watching, not yet the default.

## The built-in collectors and a custom one

The Go and process collectors give you runtime metrics for two lines, and a custom `Collector`
lets you expose numbers that already live somewhere else — like a connection pool's stats —
without duplicating them into a gauge you have to remember to update.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "regexp"
  "sort"
  "strings"

  "github.com/prometheus/client_golang/prometheus"
  "github.com/prometheus/client_golang/prometheus/collectors"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

type PoolStat struct {
  TotalConns    int32
  IdleConns     int32
  AcquiredConns int32
  AcquireCount  int64
}

type PoolStatter interface{ Stat() PoolStat }

type fakePool struct{ s PoolStat }

func (f fakePool) Stat() PoolStat { return f.s }

type PoolCollector struct {
  pool PoolStatter

  total    *prometheus.Desc
  idle     *prometheus.Desc
  acquired *prometheus.Desc
  acquires *prometheus.Desc
}

func NewPoolCollector(p PoolStatter, dbName string) *PoolCollector {
  labels := prometheus.Labels{"db": dbName}
  return &PoolCollector{
    pool: p,
    total: prometheus.NewDesc("pgxpool_conns_total",
      "Total connections currently in the pool.", nil, labels),
    idle: prometheus.NewDesc("pgxpool_conns_idle",
      "Idle connections in the pool.", nil, labels),
    acquired: prometheus.NewDesc("pgxpool_conns_acquired",
      "Connections currently checked out.", nil, labels),
    acquires: prometheus.NewDesc("pgxpool_acquires_total",
      "Cumulative successful acquires.", nil, labels),
  }
}

func (c *PoolCollector) Describe(ch chan<- *prometheus.Desc) {
  ch <- c.total
  ch <- c.idle
  ch <- c.acquired
  ch <- c.acquires
}

func (c *PoolCollector) Collect(ch chan<- prometheus.Metric) {
  s := c.pool.Stat()
  ch <- prometheus.MustNewConstMetric(c.total, prometheus.GaugeValue, float64(s.TotalConns))
  ch <- prometheus.MustNewConstMetric(c.idle, prometheus.GaugeValue, float64(s.IdleConns))
  ch <- prometheus.MustNewConstMetric(c.acquired, prometheus.GaugeValue, float64(s.AcquiredConns))
  ch <- prometheus.MustNewConstMetric(c.acquires, prometheus.CounterValue, float64(s.AcquireCount))
}

func main() {
  reg := prometheus.NewRegistry()
  reg.MustRegister(
    collectors.NewGoCollector(),
    collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}),
  )
  pool := fakePool{PoolStat{
    TotalConns: 12, IdleConns: 7, AcquiredConns: 5, AcquireCount: 48213,
  }}
  reg.MustRegister(NewPoolCollector(pool, "orders"))

  mux := http.NewServeMux()
  mux.Handle("GET /metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{}))
  srv := httptest.NewServer(mux)
  defer srv.Close()

  resp, _ := http.Get(srv.URL + "/metrics")
  body, _ := io.ReadAll(resp.Body)
  resp.Body.Close()

  fmt.Println("=== the DB pool collector (fixed fake Stat, so reproducible) ===")
  for _, line := range strings.Split(string(body), "\n") {
    if strings.HasPrefix(line, "pgxpool") || strings.HasPrefix(line, "# TYPE pgxpool") {
      fmt.Println(line)
    }
  }

  fmt.Println("\n=== built-in collectors: how many families, and the ones worth a panel ===")
  nameRe := regexp.MustCompile(`^# TYPE (\S+) (\S+)$`)
  var names []string
  for _, line := range strings.Split(string(body), "\n") {
    if m := nameRe.FindStringSubmatch(line); m != nil && !strings.HasPrefix(m[1], "pgxpool") {
      names = append(names, fmt.Sprintf("%-34s %s", m[1], m[2]))
    }
  }
  sort.Strings(names)
  fmt.Printf("  %d built-in metric families, for two lines of registration\n", len(names))
  notable := []string{"go_goroutines", "go_threads", "go_gc_duration_seconds",
    "go_gc_gomemlimit_bytes", "go_memstats_heap_inuse_bytes", "go_memstats_alloc_bytes_total",
    "process_cpu_seconds_total", "process_resident_memory_bytes",
    "process_open_fds", "process_max_fds"}
  for _, n := range names {
    for _, w := range notable {
      if strings.HasPrefix(n, w+" ") {
        fmt.Println("   ", n)
      }
    }
  }

  fmt.Println("\n=== a few real sampled values (these DO vary run to run) ===")
  for _, line := range strings.Split(string(body), "\n") {
    for _, want := range []string{"go_goroutines ", "go_threads ", "go_memstats_alloc_bytes ",
      "go_gc_duration_seconds_count", "process_open_fds "} {
      if strings.HasPrefix(line, want) {
        fmt.Println(" ", line)
      }
    }
  }
}
```

The pool is a fake with a fixed `Stat()` so the `pgxpool_*` values are reproducible; with real
pgx the method is `pgxpool.Pool.Stat()` and the fields are the same names
([17.5](../17-database/05-pooling.md)). The Go and process metric **values** vary every run —
the sampled numbers at the end are from one real scrape and will differ on yours.

Output:

```
=== the DB pool collector (fixed fake Stat, so reproducible) ===
# TYPE pgxpool_acquires_total counter
pgxpool_acquires_total{db="orders"} 48213
# TYPE pgxpool_conns_acquired gauge
pgxpool_conns_acquired{db="orders"} 5
# TYPE pgxpool_conns_idle gauge
pgxpool_conns_idle{db="orders"} 7
# TYPE pgxpool_conns_total gauge
pgxpool_conns_total{db="orders"} 12

=== built-in collectors: how many families, and the ones worth a panel ===
  36 built-in metric families, for two lines of registration
    go_gc_duration_seconds             summary
    go_gc_gomemlimit_bytes             gauge
    go_goroutines                      gauge
    go_memstats_alloc_bytes_total      counter
    go_memstats_heap_inuse_bytes       gauge
    go_threads                         gauge
    process_cpu_seconds_total          counter
    process_max_fds                    gauge
    process_open_fds                   gauge
    process_resident_memory_bytes      gauge

=== a few real sampled values (these DO vary run to run) ===
  go_gc_duration_seconds_count 0
  go_goroutines 9
  go_memstats_alloc_bytes 203680
  go_threads 7
  process_open_fds 8
```

The ten printed above are the ones that earn a dashboard panel. `go_goroutines` climbing
monotonically is a goroutine leak ([12.5](../12-concurrency-patterns/05-deadlocks-leaks.md));
`process_resident_memory_bytes` is what the OOM killer looks at and is always higher than heap
in-use; `process_open_fds` climbing against `process_max_fds` means you are leaking response
bodies or files.

The `Collector` interface is two methods: `Describe` sends the `*Desc`s, `Collect` builds
metrics on demand. Writing one is right whenever the numbers already exist somewhere — a pool,
a cache, a queue client — because `Collect` reads the live value at scrape time with no
background goroutine keeping a gauge in sync. `collectors.NewDBStatsCollector(db, name)` ships
one for `database/sql` ([17.1](../17-database/01-database-sql.md)).

Also register `collectors.NewBuildInfoCollector()` for module version metrics, and
`collectors.NewGoCollector(collectors.WithGoCollectorRuntimeMetrics(...))` for the newer
`runtime/metrics`-based series.

## Exposing /metrics, and whether to authenticate it

```go
admin := http.NewServeMux()
admin.Handle("GET /metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{
  ErrorHandling:     promhttp.HTTPErrorOnError,
  EnableOpenMetrics: true,
}))
admin.HandleFunc("GET /healthz", h.Livez)
admin.Handle("GET /debug/pprof/", http.DefaultServeMux)

go func() {
  adminSrv := &http.Server{Addr: ":9090", Handler: admin}
  adminSrv.ListenAndServe()
}()
```

Serve it on a **separate port** your ingress does not route, alongside the health endpoints and
pprof ([15.6](../15-tooling/06-pprof.md)). That beats authentication: `/metrics` reveals your
route list, traffic volumes, error rates, pool sizes and build version — a precise map of your
service — and it does not belong on the public internet at all.

Keep `/metrics` out of the metrics middleware itself, or the scrape shows up in your request
rate.

## Testing metrics

`prometheus/testutil` gives you the two assertions you need.

```go
package main

import (
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
  "testing"
  "time"

  "github.com/go-chi/chi/v5"
  "github.com/prometheus/client_golang/prometheus/testutil"
)

func newRouter(m *Metrics, d time.Duration) http.Handler {
  r := chi.NewRouter()
  r.Use(m.Middleware(func() time.Duration { return d }))
  r.Get("/orders/{id}", func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "ok")
  })
  r.Get("/boom", func(w http.ResponseWriter, r *http.Request) {
    http.Error(w, "boom", http.StatusInternalServerError)
  })
  return r
}

func hit(t *testing.T, h http.Handler, method, path string) {
  t.Helper()
  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, httptest.NewRequest(method, path, nil))
}

func TestCounterUsesRoutePatternNotPath(t *testing.T) {
  m := NewMetrics()
  h := newRouter(m, 5*time.Millisecond)
  for _, id := range []string{"1", "2", "3"} {
    hit(t, h, "GET", "/orders/"+id)
  }

  got := testutil.ToFloat64(m.Requests.WithLabelValues("GET", "/orders/{id}", "200"))
  if got != 3 {
    t.Errorf("counter = %v, want 3", got)
  }
  if n := testutil.CollectAndCount(m.Requests); n != 1 {
    t.Errorf("got %d time series, want 1 (three IDs must collapse into one)", n)
  }
}

func TestErrorsGetTheirOwnStatusLabel(t *testing.T) {
  m := NewMetrics()
  h := newRouter(m, 3*time.Millisecond)
  hit(t, h, "GET", "/orders/1")
  hit(t, h, "GET", "/boom")
  hit(t, h, "GET", "/boom")

  want := strings.NewReader(`
# HELP http_requests_total Total number of HTTP requests.
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/boom",status="500"} 2
http_requests_total{method="GET",route="/orders/{id}",status="200"} 1
`)
  if err := testutil.CollectAndCompare(m.Requests, want, "http_requests_total"); err != nil {
    t.Error(err)
  }
}

func TestHistogramBucketsAreCumulative(t *testing.T) {
  m := NewMetrics()
  h := newRouter(m, 50*time.Millisecond)
  hit(t, h, "GET", "/orders/1")

  want := strings.NewReader(`
# HELP http_request_duration_seconds HTTP request latency in seconds.
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="0.01"} 0
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="0.1"} 1
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="1"} 1
http_request_duration_seconds_bucket{method="GET",route="/orders/{id}",le="+Inf"} 1
http_request_duration_seconds_sum{method="GET",route="/orders/{id}"} 0.05
http_request_duration_seconds_count{method="GET",route="/orders/{id}"} 1
`)
  if err := testutil.CollectAndCompare(m.Duration, want, "http_request_duration_seconds"); err != nil {
    t.Error(err)
  }
}

func TestUnmatchedRouteIsBounded(t *testing.T) {
  m := NewMetrics()
  h := newRouter(m, time.Millisecond)
  for _, p := range []string{"/nope", "/also-nope", "/wat/1/2/3"} {
    hit(t, h, "GET", p)
  }
  if got := testutil.ToFloat64(m.Requests.WithLabelValues("GET", "unmatched", "404")); got != 3 {
    t.Errorf("unmatched counter = %v, want 3", got)
  }
  if n := testutil.CollectAndCount(m.Requests); n != 1 {
    t.Errorf("got %d series, want 1", n)
  }
}
```

These tests use a trimmed `Metrics` with a three-bucket histogram and an `elapsed func()
time.Duration` taking no route argument, so the expected exposition stays short:

```go
func NewMetrics() *Metrics {
  reg := prometheus.NewRegistry()
  m := &Metrics{
    Registry: reg,
    Requests: prometheus.NewCounterVec(prometheus.CounterOpts{
      Name: "http_requests_total",
      Help: "Total number of HTTP requests.",
    }, []string{"method", "route", "status"}),
    Duration: prometheus.NewHistogramVec(prometheus.HistogramOpts{
      Name:    "http_request_duration_seconds",
      Help:    "HTTP request latency in seconds.",
      Buckets: []float64{0.01, 0.1, 1},
    }, []string{"method", "route"}),
  }
  reg.MustRegister(m.Requests, m.Duration)
  return m
}

func (m *Metrics) Middleware(elapsed func() time.Duration) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      rw := &recorder{ResponseWriter: w}
      next.ServeHTTP(rw, r)
      route := chi.RouteContext(r.Context()).RoutePattern()
      if route == "" {
        route = "unmatched"
      }
      m.Requests.WithLabelValues(r.Method, route, strconv.Itoa(rw.status)).Inc()
      m.Duration.WithLabelValues(r.Method, route).Observe(elapsed().Seconds())
    })
  }
}
```

`go test -v` (the elapsed time varies run to run):

Output:

```
=== RUN   TestCounterUsesRoutePatternNotPath
--- PASS: TestCounterUsesRoutePatternNotPath (0.00s)
=== RUN   TestErrorsGetTheirOwnStatusLabel
--- PASS: TestErrorsGetTheirOwnStatusLabel (0.00s)
=== RUN   TestHistogramBucketsAreCumulative
--- PASS: TestHistogramBucketsAreCumulative (0.00s)
=== RUN   TestUnmatchedRouteIsBounded
--- PASS: TestUnmatchedRouteIsBounded (0.00s)
PASS
ok  	example.com/scratch	0.545s
```

`ToFloat64` reads one metric's value — "did this counter go up". `CollectAndCompare` asserts a
collector's **entire exposition** against expected text, catching a label you did not mean to
add. `CollectAndCount` asserts the series count, which is how you write a **regression test
against cardinality** — worth putting in every service, because it fails the build when someone
adds a user-ID label. A fresh `NewMetrics()` per test is what makes these independent, which is
the custom-registry argument made concrete.

## OpenTelemetry, in one line

OTel metrics are the vendor-neutral alternative — one SDK for metrics, traces and logs, with an
exporter that still speaks Prometheus — worth adopting when you want distributed tracing anyway,
and otherwise more moving parts for the same dashboards.

## Quick Reference

| Constructor | Type |
|---|---|
| `prometheus.NewCounter(CounterOpts{...})` | single counter |
| `prometheus.NewCounterVec(opts, []string{"label"})` | counter family |
| `prometheus.NewGauge` / `NewGaugeVec` | gauge |
| `prometheus.NewHistogram(HistogramOpts{Buckets: ...})` / `NewHistogramVec` | histogram |
| `prometheus.NewSummary` / `NewSummaryVec` | summary (prefer histogram) |
| `NewDesc(name, help, varLabels, constLabels)` + `MustNewConstMetric(desc, type, v, labels...)` | a custom collector's `Describe`/`Collect` |

| Operation | Call |
|---|---|
| increment | `c.Inc()`, `c.Add(n)` |
| set / adjust a gauge | `g.Set(v)`, `g.Inc()`, `g.Dec()`, `g.SetToCurrentTime()` |
| record an observation | `h.Observe(seconds)` |
| pick a labelled child | `vec.WithLabelValues("GET", "/x")` (panics on arity) or `GetMetricWithLabelValues` |
| drop a child series | `vec.DeleteLabelValues(...)`, `vec.Reset()` |
| time a block | `t := prometheus.NewTimer(h); defer t.ObserveDuration()` |
| instrument a client | `promhttp.InstrumentRoundTripperDuration(hv, rt)` |

| Registry / handler | Call |
|---|---|
| new registry | `prometheus.NewRegistry()` |
| register | `reg.MustRegister(...)` / `reg.Register(...)` → `AlreadyRegisteredError` |
| auto-register factory | `promauto.With(reg).NewCounter(...)` |
| serve | `promhttp.HandlerFor(reg, promhttp.HandlerOpts{})` |
| Go/process metrics | `collectors.NewGoCollector()`, `collectors.NewProcessCollector(opts)` |
| `database/sql` pool | `collectors.NewDBStatsCollector(db, "orders")` |

| Test helper | Asserts |
|---|---|
| `testutil.ToFloat64(m)` | one metric's current value |
| `testutil.CollectAndCompare(c, expected, names...)` | the full exposition text |
| `testutil.CollectAndCount(c)` | the number of time series (cardinality) |
| `testutil.GatherAndLint(reg)` | naming-convention violations |

## Common Mistakes

**A raw URL path, user ID or trace ID as a label.** Cardinality explodes and Prometheus runs
out of memory; use the route pattern and put the detail in logs or traces.

**No fallback for the empty route pattern.** 404s from a scanner each create a series; map
unmatched requests to one `"unmatched"` label.

**Milliseconds in a `_seconds` metric, or no `_total` on a counter.** Every community
dashboard and alerting rule silently breaks; follow the naming conventions.

**Using the default global registry.** Tests panic on duplicate registration and see each
other's values; create a registry per `NewMetrics()`.

**Summary instead of Histogram for latency.** You can never aggregate a quantile across pods,
which is the only question you have in production.

**Reading buckets as per-range counts.** They are cumulative `le` counts; subtract adjacent
buckets, or just use `histogram_quantile`.

**Decrementing a counter.** `rate()` interprets the drop as a process restart and your graph
goes wrong; use a gauge for anything that can go down.

**`/metrics` on the public port, unauthenticated.** It discloses your routes, traffic, error
rates and version; serve it on a non-routed admin port.

**Instrumenting `/metrics` and `/healthz` with your own middleware.** Probes and scrapes then
dominate your request-rate graphs; exclude them.

## Exercises

1. Add a `response_size_bytes` histogram and an `http_requests_in_flight` gauge **by route** to
   the middleware, drive 20 mixed requests, and paste the real `/metrics` output. Then compute,
   by hand from the bucket counts, the p50 and p95 for one route and check your arithmetic
   against `histogram_quantile`.
2. Write a `CollectAndCount` regression test that fails if any metric family exceeds 50 series,
   then deliberately add a `user_id` label and show the test failing with its real message.
3. Instrument the outbound HTTP client from 16.10 with
   `promhttp.InstrumentRoundTripperDuration` and `...Counter`, labelled by the **upstream
   name** (not URL). Show the exposition for two upstreams, one of which returns 503.
4. Write a custom `Collector` for a bounded in-memory cache exposing size, capacity, hits and
   misses, and prove with `testutil.CollectAndCompare` that `Collect` reads live values at
   scrape time rather than a stale snapshot.
5. Run `testutil.GatherAndLint` over your registry after deliberately naming a metric
   `requestDurationMs`, capture the real lint output, then fix the name and show it clean.

---

[← 18.6 Health Checks and Readiness](06-health-checks.md) · [Course Index](../README.md) · [18.8 Docker Multi-Stage Builds →](08-docker.md)
