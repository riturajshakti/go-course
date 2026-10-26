[← 18.4 Structured Logging in a Service](04-service-logging.md) · [Course Index](../README.md) · [18.6 Health Checks and Readiness →](06-health-checks.md)

# 18.5 — Graceful Shutdown

## What You'll Learn

- `signal.NotifyContext` plus `srv.Shutdown(ctx)` — the whole pattern in fifteen lines, run for real with an actual `SIGTERM`
- `http.ErrServerClosed` is the **expected** return from `ListenAndServe`, not a failure
- `Shutdown` drains in-flight requests; `Close` kills them — both demonstrated, with the client's view of each
- The teardown **order**, and flipping readiness to NOT-ready *before* shutting down — the detail almost everyone misses

## Coming From Another Language

| Other | Go |
|---|---|
| `server.close(cb)` + `process.on('SIGTERM')` | `signal.NotifyContext` + `srv.Shutdown(ctx)`; no callbacks |
| gunicorn/uvicorn handles SIGTERM for you | **you** handle it; nothing is automatic |
| `atexit` / `finally` running on exit | `os.Exit` skips every `defer`, so only `main` may call it |
| a framework "graceful" plugin | it is in the standard library: `(*http.Server).Shutdown` |
| killing a task/thread from outside | impossible for a goroutine; you cancel a context and the goroutine must cooperate |

## Why it matters

A container runtime sends `SIGTERM`, waits a grace period (Kubernetes defaults to 30
seconds), then sends `SIGKILL`. If your process dies on `SIGTERM`:

- every in-flight request becomes a connection reset — a 502 in the client's logs, during an
  ordinary deploy, for no reason
- a transaction mid-commit is abandoned
- buffered metrics and logs are lost, so the last seconds before the shutdown are invisible —
  exactly the seconds you want when debugging a bad deploy

Rolling deploys mean this happens on every single release. Getting it right is a one-time
fifteen-line cost that removes a permanent source of error noise.

## signal.NotifyContext

Lead with this. It gives you a `context.Context` cancelled on the first matching signal:

```go
ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
defer stop()
```

`stop()` restores the default signal behaviour, which matters: after you call it, a **second**
`SIGTERM` kills the process immediately. That is the behaviour you want — an operator pressing
Ctrl-C twice because shutdown is hanging should get an exit, not be ignored. So call `stop()`
as soon as the signal arrives, not only via `defer`.

The older form, still worth recognising in existing code:

```go
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
sig := <-sigCh
log.Info("signal received", "signal", sig.String())
```

The channel **must be buffered** — `signal.Notify` sends non-blockingly, so an unbuffered
channel with no ready receiver drops the signal silently. `NotifyContext` exists precisely so
you cannot get that wrong, and it composes with every other context-aware API in your service.

Which signals: `SIGTERM` is what orchestrators send, `SIGINT` is Ctrl-C. Do **not** trap
`SIGKILL` (impossible by design) and think twice before trapping `SIGQUIT`, whose default
behaviour of dumping all goroutine stacks is genuinely useful when a shutdown wedges.

## Shutdown versus Close

| | `Shutdown(ctx)` | `Close()` |
|---|---|---|
| Listeners | closed immediately | closed immediately |
| Idle keep-alive connections | closed immediately | closed immediately |
| **In-flight requests** | allowed to finish | connection closed under them |
| Returns when | all connections idle, or `ctx` expires | immediately |
| On `ctx` expiry | returns `ctx.Err()`, connections stay open | n/a |
| Client sees | its complete response | a reset / `EOF` |

Both, with a request in flight:

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
  "sync"
  "time"
)

func main() {
  fmt.Println("=== Close() vs Shutdown() on an in-flight request ===")
  for _, mode := range []string{"Shutdown", "Close"} {
    started := make(chan struct{})
    mux := http.NewServeMux()
    mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
      close(started)
      select {
      case <-time.After(400 * time.Millisecond):
        io.WriteString(w, "completed")
      case <-r.Context().Done():
        fmt.Printf("  [%s] handler saw r.Context() done: %v\n", mode, r.Context().Err())
      }
    })

    srv := httptest.NewUnstartedServer(mux)
    srv.Start()

    var wg sync.WaitGroup
    wg.Add(1)
    var clientErr error
    var body string
    go func() {
      defer wg.Done()
      resp, err := http.Get(srv.URL + "/slow")
      if err != nil {
        clientErr = err
        return
      }
      b, rErr := io.ReadAll(resp.Body)
      resp.Body.Close()
      body, clientErr = string(b), rErr
    }()

    <-started
    time.Sleep(50 * time.Millisecond)
    if mode == "Shutdown" {
      err := srv.Config.Shutdown(context.Background())
      fmt.Printf("  [%s] returned %v\n", mode, err)
    } else {
      err := srv.Config.Close()
      fmt.Printf("  [%s] returned %v\n", mode, err)
    }
    wg.Wait()
    errStr := "<nil>"
    if clientErr != nil {
      errStr = strings.Replace(clientErr.Error(), srv.URL, "SRV", 1)
    }
    fmt.Printf("  [%s] client: body=%q err=%s\n", mode, body, errStr)
    srv.Close()
  }

  fmt.Println("\n=== http.ErrServerClosed is the EXPECTED return ===")
  srv := &http.Server{Addr: "127.0.0.1:18414"}
  go func() {
    time.Sleep(100 * time.Millisecond)
    srv.Shutdown(context.Background())
  }()
  err := srv.ListenAndServe()
  fmt.Printf("ListenAndServe -> %v\n", err)
  fmt.Printf("err == http.ErrServerClosed      : %v\n", err == http.ErrServerClosed)
  fmt.Printf("errors.Is(err, ErrServerClosed)  : %v\n", errors.Is(err, http.ErrServerClosed))
  fmt.Printf("treating it as a failure would be a false alarm\n")
}
```

The httptest server's port varies, so the client error has it substituted with `SRV`.

Output:

```
=== Close() vs Shutdown() on an in-flight request ===
  [Shutdown] returned <nil>
  [Shutdown] client: body="completed" err=<nil>
  [Close] returned <nil>
  [Close] handler saw r.Context() done: context canceled
  [Close] client: body="" err=Get "SRV/slow": EOF

=== http.ErrServerClosed is the EXPECTED return ===
ListenAndServe -> http: Server closed
err == http.ErrServerClosed      : true
errors.Is(err, ErrServerClosed)  : true
treating it as a failure would be a false alarm
```

`Shutdown` gave the client `"completed"`. `Close` gave it `EOF` and cancelled the handler's
request context. That is the entire difference, and it is why `Close` is the fallback after a
`Shutdown` timeout, never the first move.

### http.ErrServerClosed

`ListenAndServe` returns `http.ErrServerClosed` after `Shutdown` or `Close` — a *successful*
shutdown. The near-universal bug:

```go
if err := srv.ListenAndServe(); err != nil {
  log.Error("server failed", "err", err)
  os.Exit(1)
}
```

That logs an error and exits non-zero on every clean shutdown, which makes your crash-loop
alerts fire on every deploy and trains everyone to ignore them. Always:

```go
if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
  return fmt.Errorf("listen: %w", err)
}
```

## The canonical main, run for real

Start the server in a goroutine, wait on the signal context, flip readiness, then `Shutdown`
with a timeout. Here it is complete, and then genuinely executed with a real `kill -TERM`.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "io"
  "log/slog"
  "net/http"
  "os"
  "os/signal"
  "sync/atomic"
  "syscall"
  "time"
)

func main() {
  if err := run(); err != nil {
    slog.Error("fatal", "err", err)
    os.Exit(1)
  }
  slog.Info("exited cleanly")
}

func run() error {
  log := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      if a.Key == slog.TimeKey && len(g) == 0 {
        return slog.Attr{}
      }
      return a
    },
  }))
  slog.SetDefault(log)

  ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
  defer stop()

  var ready atomic.Bool
  ready.Store(true)

  mux := http.NewServeMux()
  mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
    if !ready.Load() {
      http.Error(w, "draining", http.StatusServiceUnavailable)
      return
    }
    io.WriteString(w, "ready")
  })
  mux.HandleFunc("GET /slow", func(w http.ResponseWriter, r *http.Request) {
    log.Info("slow handler: started", "path", r.URL.Path)
    select {
    case <-time.After(2 * time.Second):
      log.Info("slow handler: finished its work")
      io.WriteString(w, "slow response completed\n")
    case <-r.Context().Done():
      log.Warn("slow handler: request context cancelled", "err", r.Context().Err())
    }
  })
  mux.HandleFunc("GET /fast", func(w http.ResponseWriter, r *http.Request) {
    io.WriteString(w, "fast\n")
  })

  srv := &http.Server{
    Addr:              "127.0.0.1:18411",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
  }

  errCh := make(chan error, 1)
  go func() {
    log.Info("server listening", "addr", srv.Addr)
    err := srv.ListenAndServe()
    log.Info("ListenAndServe returned", "err", err,
      "is_ErrServerClosed", errors.Is(err, http.ErrServerClosed))
    if err != nil && !errors.Is(err, http.ErrServerClosed) {
      errCh <- err
      return
    }
    errCh <- nil
  }()

  select {
  case err := <-errCh:
    return err
  case <-ctx.Done():
    stop()
    log.Info("signal received, beginning graceful shutdown")
  }

  ready.Store(false)
  log.Info("readiness flipped to NOT ready", "grace", "2s pre-stop delay would go here")

  shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
  defer cancel()

  start := time.Now()
  err := srv.Shutdown(shutdownCtx)
  log.Info("Shutdown returned", "err", err, "drained_in_under", "10s timeout")
  _ = start

  if serveErr := <-errCh; serveErr != nil {
    return fmt.Errorf("serve: %w", serveErr)
  }
  return err
}
```

Built and started in the background, then driven with `curl` and a real signal:

```bash
./srv > server.log 2>&1 &
PID=$!
curl -s http://127.0.0.1:18411/fast
curl -s -o /dev/null -w "readyz -> %{http_code}\n" http://127.0.0.1:18411/readyz

curl -s http://127.0.0.1:18411/slow &     # 2-second handler, in flight
sleep 0.5
kill -TERM $PID                            # a genuine SIGTERM, mid-request
sleep 0.4
curl -s http://127.0.0.1:18411/fast || echo "new request failed: curl exit=$?"
wait $PID; echo "server exit=$?"
```

The server's log:

Output:

```
level=INFO msg="server listening" addr=127.0.0.1:18411
level=INFO msg="slow handler: started" path=/slow
level=INFO msg="signal received, beginning graceful shutdown"
level=INFO msg="readiness flipped to NOT ready" grace="2s pre-stop delay would go here"
level=INFO msg="ListenAndServe returned" err="http: Server closed" is_ErrServerClosed=true
level=INFO msg="slow handler: finished its work"
level=INFO msg="Shutdown returned" err=<nil> drained_in_under="10s timeout"
level=INFO msg="exited cleanly"
```

The client side:

Output:

```
--- baseline fast request ---
fast
--- readyz before shutdown ---
readyz -> 200
--- sending SIGTERM while /slow is in flight ---
--- new request AFTER SIGTERM (listener closed) ---
new request failed: curl exit=7
readyz -> 000
readyz failed: curl exit=7
```

and the in-flight request:

Output:

```
slow response completed
slow curl exit=0
```

Read those three captures together, because every claim in this lesson is in them.

**The in-flight request completed.** `/slow` was 500ms into a 2-second handler when `SIGTERM`
arrived. The handler logged `finished its work` *after* `ListenAndServe` had already returned
`http: Server closed`, and the client got the full `slow response completed` with exit 0. No
reset, no 502.

**New connections were refused immediately.** `curl exit=7` is "failed to connect" — the
listener was closed the instant `Shutdown` was called, 1.5 seconds before the process exited.
That is the correct behaviour: stop taking new work, finish the work you have.

**`ListenAndServe` returned `http: Server closed` and that was fine.** `is_ErrServerClosed=true`,
and `run` returned nil, and the process exited 0.

**`Shutdown` returned `<nil>`** because everything drained inside the 10-second budget.

**Readiness flipped first**, before `Shutdown` — which is the next section.

## When the shutdown timeout expires

A handler that ignores `r.Context()` cannot be drained. `Shutdown` waits, the deadline passes,
and you have to decide what to do.

```go
package main

import (
  "context"
  "errors"
  "io"
  "log/slog"
  "net/http"
  "os"
  "os/signal"
  "syscall"
  "time"
)

func main() {
  log := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      if a.Key == slog.TimeKey && len(g) == 0 {
        return slog.Attr{}
      }
      return a
    },
  }))

  ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
  defer stop()

  mux := http.NewServeMux()
  mux.HandleFunc("GET /stuck", func(w http.ResponseWriter, r *http.Request) {
    log.Info("stuck handler: started, ignores r.Context()")
    time.Sleep(30 * time.Second)
    log.Info("stuck handler: would finish here, but we never get this far")
    io.WriteString(w, "done\n")
  })

  srv := &http.Server{Addr: "127.0.0.1:18412", Handler: mux}
  go func() {
    log.Info("server listening", "addr", srv.Addr)
    if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
      log.Error("listen", "err", err)
    }
  }()

  <-ctx.Done()
  stop()
  log.Info("signal received, shutting down with a 2s timeout")

  shutdownCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
  defer cancel()

  err := srv.Shutdown(shutdownCtx)
  log.Warn("Shutdown returned",
    "err", err,
    "is_DeadlineExceeded", errors.Is(err, context.DeadlineExceeded))

  if err != nil {
    log.Warn("timeout expired: forcing connections closed with Close()")
    log.Info("Close returned", "err", srv.Close())
  }
  log.Info("process exiting while the stuck handler goroutine is still running")
}
```

Run the same way — start it, fire `/stuck`, `kill -TERM`:

Output:

```
level=INFO msg="server listening" addr=127.0.0.1:18412
level=INFO msg="stuck handler: started, ignores r.Context()"
level=INFO msg="signal received, shutting down with a 2s timeout"
level=WARN msg="Shutdown returned" err="context deadline exceeded" is_DeadlineExceeded=true
level=WARN msg="timeout expired: forcing connections closed with Close()"
level=INFO msg="Close returned" err=<nil>
level=INFO msg="process exiting while the stuck handler goroutine is still running"
```

The client got `curl exit=52` — "empty reply from server", the connection closed without a
response.

Three facts in that capture. `Shutdown` returned `context.DeadlineExceeded`, so you can detect
the situation and log it loudly — a shutdown timing out is a bug worth fixing, usually a
handler not selecting on `r.Context().Done()`. **`Shutdown` does not force connections closed
when it gives up**, which is why the explicit `Close()` follows. And the `time.Sleep(30s)`
goroutine was still running when the process exited: Go cannot kill a goroutine, so
`os.Exit` is the only thing that ends it.

Size the timeout **below** the orchestrator's grace period — 25 seconds against Kubernetes'
30-second default — so your own `Close()` runs before `SIGKILL` arrives and you get the log
line that tells you shutdown timed out.

## Teardown order, and why

Shutting resources down in the wrong order produces errors during every deploy:

| # | Step | Why here |
|---|---|---|
| 1 | readiness → NOT ready | the load balancer stops sending new requests *before* you stop accepting them |
| 2 | HTTP `Shutdown(ctx)` | stop accepting, drain in-flight handlers |
| 3 | stop background workers | they may still be using the DB; drain their queues |
| 4 | close the DB pool | nothing needs it any more — closing earlier makes draining requests fail |
| 5 | flush metrics and logs | last, so steps 1-4 are themselves recorded |

Close the database first and every draining request fails with "pool closed" — you have turned
a graceful shutdown into a burst of 500s. The rule is **outside-in**: stop the inbound edge,
then drain, then release what the drain depended on.

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "io"
  "log/slog"
  "net/http"
  "os"
  "os/signal"
  "sync"
  "sync/atomic"
  "syscall"
  "time"
)

type DBPool struct {
  log    *slog.Logger
  closed atomic.Bool
}

func (p *DBPool) Close() error {
  p.log.Info("db pool: draining in-flight queries and closing connections")
  p.closed.Store(true)
  return nil
}

type MetricsFlusher struct{ log *slog.Logger }

func (m *MetricsFlusher) Flush(ctx context.Context) error {
  m.log.Info("metrics: flushing final batch to the collector")
  return nil
}

type Workers struct {
  log  *slog.Logger
  jobs chan int
  wg   sync.WaitGroup
}

func NewWorkers(log *slog.Logger, n int) *Workers {
  w := &Workers{log: log, jobs: make(chan int, 16)}
  for i := 1; i <= n; i++ {
    w.wg.Add(1)
    go w.worker(i)
  }
  return w
}

func (w *Workers) worker(id int) {
  defer w.wg.Done()
  for job := range w.jobs {
    time.Sleep(100 * time.Millisecond)
    w.log.Info("worker: job done", "worker", id, "job", job)
  }
  w.log.Info("worker: job channel closed, exiting", "worker", id)
}

func (w *Workers) Submit(job int) { w.jobs <- job }

func (w *Workers) Shutdown(ctx context.Context) error {
  close(w.jobs)
  done := make(chan struct{})
  go func() { w.wg.Wait(); close(done) }()
  select {
  case <-done:
    w.log.Info("workers: all drained")
    return nil
  case <-ctx.Done():
    return fmt.Errorf("workers did not drain: %w", ctx.Err())
  }
}

type App struct {
  log     *slog.Logger
  srv     *http.Server
  db      *DBPool
  metrics *MetricsFlusher
  workers *Workers
  ready   atomic.Bool
}

func (a *App) teardown(ctx context.Context) error {
  var errs []error

  a.log.Info("teardown 1/5: readiness -> NOT ready (load balancer stops sending traffic)")
  a.ready.Store(false)

  a.log.Info("teardown 2/5: http server Shutdown (stop accepting, drain in-flight)")
  if err := a.srv.Shutdown(ctx); err != nil {
    errs = append(errs, fmt.Errorf("http shutdown: %w", err))
  }

  a.log.Info("teardown 3/5: worker pool drain")
  if err := a.workers.Shutdown(ctx); err != nil {
    errs = append(errs, fmt.Errorf("workers: %w", err))
  }

  a.log.Info("teardown 4/5: database pool close")
  if err := a.db.Close(); err != nil {
    errs = append(errs, fmt.Errorf("db close: %w", err))
  }

  a.log.Info("teardown 5/5: metrics flush")
  if err := a.metrics.Flush(ctx); err != nil {
    errs = append(errs, fmt.Errorf("metrics flush: %w", err))
  }

  return errors.Join(errs...)
}

func main() {
  if err := run(); err != nil {
    slog.Error("fatal", "err", err)
    os.Exit(1)
  }
  slog.Info("process exiting with status 0")
}

func run() error {
  log := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      if a.Key == slog.TimeKey && len(g) == 0 {
        return slog.Attr{}
      }
      return a
    },
  }))
  slog.SetDefault(log)

  ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
  defer stop()

  app := &App{
    log:     log,
    db:      &DBPool{log: log},
    metrics: &MetricsFlusher{log: log},
    workers: NewWorkers(log, 2),
  }
  app.ready.Store(true)

  mux := http.NewServeMux()
  mux.HandleFunc("GET /readyz", func(w http.ResponseWriter, r *http.Request) {
    if !app.ready.Load() {
      http.Error(w, "draining", http.StatusServiceUnavailable)
      return
    }
    io.WriteString(w, "ready\n")
  })
  mux.HandleFunc("GET /work", func(w http.ResponseWriter, r *http.Request) {
    app.workers.Submit(7)
    app.workers.Submit(8)
    io.WriteString(w, "queued\n")
  })
  app.srv = &http.Server{
    Addr:              "127.0.0.1:18413",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
  }

  serveErr := make(chan error, 1)
  go func() {
    log.Info("server listening", "addr", app.srv.Addr)
    err := app.srv.ListenAndServe()
    if errors.Is(err, http.ErrServerClosed) {
      log.Info("ListenAndServe returned the expected http.ErrServerClosed")
      err = nil
    }
    serveErr <- err
  }()

  select {
  case err := <-serveErr:
    if err != nil {
      return fmt.Errorf("listen: %w", err)
    }
    return nil
  case <-ctx.Done():
    stop()
    log.Info("SIGTERM received; starting ordered teardown")
  }

  tdCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
  defer cancel()

  tdErr := app.teardown(tdCtx)
  if err := <-serveErr; err != nil {
    tdErr = errors.Join(tdErr, fmt.Errorf("listen: %w", err))
  }
  return tdErr
}
```

Started, hit `/readyz` and `/work` (which queues two jobs), then `kill -TERM`:

Output:

```
level=INFO msg="server listening" addr=127.0.0.1:18413
level=INFO msg="SIGTERM received; starting ordered teardown"
level=INFO msg="teardown 1/5: readiness -> NOT ready (load balancer stops sending traffic)"
level=INFO msg="teardown 2/5: http server Shutdown (stop accepting, drain in-flight)"
level=INFO msg="ListenAndServe returned the expected http.ErrServerClosed"
level=INFO msg="teardown 3/5: worker pool drain"
level=INFO msg="worker: job done" worker=2 job=8
level=INFO msg="worker: job channel closed, exiting" worker=2
level=INFO msg="worker: job done" worker=1 job=7
level=INFO msg="worker: job channel closed, exiting" worker=1
level=INFO msg="workers: all drained"
level=INFO msg="teardown 4/5: database pool close"
level=INFO msg="db pool: draining in-flight queries and closing connections"
level=INFO msg="teardown 5/5: metrics flush"
level=INFO msg="metrics: flushing final batch to the collector"
level=INFO msg="process exiting with status 0"
```

Which worker picks up which job, and in what order the two `worker:` lines interleave, varies
run to run — that is normal goroutine scheduling
([10.1](../10-concurrency-basics/01-goroutines.md)). Everything else above is deterministic.

**Both queued jobs ran to completion** before `workers: all drained`. The worker shutdown is
the standard pattern: `close(jobs)` makes every `for range w.jobs` loop terminate once the
buffer empties, and the `WaitGroup` tells you when the last one has returned
([12.1](../12-concurrency-patterns/01-worker-pools.md),
[11.1](../11-sync-context/01-waitgroup.md)). The `select` over `done` and `ctx.Done()` bounds
the wait, because `wg.Wait()` alone can hang forever.

**`errors.Join` collects failures without stopping the teardown.** Shutting down is exactly
when you must not bail on the first error — skipping the metrics flush because the DB close
failed loses the evidence of the DB failure ([8.5](../08-errors/05-join.md)).

**Goroutines you started must also stop.** A goroutine blocked on a channel send, or looping
without a `ctx.Done()` case, keeps the process alive past `Shutdown` or gets killed
mid-operation. Every long-lived goroutine takes a context and selects on it
([11.6](../11-sync-context/06-context-cancel.md)); a goroutine with no exit path is a leak
([12.5](../12-concurrency-patterns/05-deadlocks-leaks.md)).

## The pre-stop delay, and why almost everyone gets it wrong

This is the subtlest part of the lesson, and the reason graceful shutdown is still followed by
502s in services that implement everything above correctly.

The sequence looks like it should work: `SIGTERM` arrives, you flip readiness to false, the
load balancer sees the failing probe and removes you from the pool, you drain. The problem is
**timing**. The load balancer is not watching continuously — it polls every
`periodSeconds` (often 5 or 10) and needs `failureThreshold` consecutive failures. So for
several seconds after `SIGTERM`, the load balancer is still routing new requests to you — and
by then you have already called `Shutdown`, your listener is closed, and those requests are
connection-refused. A 502 per request, on every deploy, caused by shutting down too *fast*.

In Kubernetes the endpoint removal is also asynchronous and racy: the pod's `SIGTERM` and the
Endpoints update are two independent propagations, and kube-proxy on each node updates its
rules whenever it gets around to it.

The fix is a deliberate delay between flipping readiness and calling `Shutdown`:

```go
case <-ctx.Done():
  stop()
  log.Info("signal received")
}

ready.Store(false)
log.Info("readiness now failing; waiting for load balancers to notice")
time.Sleep(cfg.PreStopDelay)        // 5-15s: > periodSeconds * failureThreshold

shutdownCtx, cancel := context.WithTimeout(context.Background(), cfg.ShutdownTimeout)
defer cancel()
return srv.Shutdown(shutdownCtx)
```

During that sleep you are **still serving normally** — readiness fails, but the listener is
open and requests succeed. That is the point: you keep accepting traffic until everyone has
agreed to stop sending it. Size the delay above `periodSeconds × failureThreshold` of your
readiness probe ([18.6](06-health-checks.md) has the probe YAML) and keep
`PreStopDelay + ShutdownTimeout` safely under `terminationGracePeriodSeconds`.

The alternative placement is a Kubernetes `preStop` hook, which runs **before** `SIGTERM` is
sent and delays it:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]
terminationGracePeriodSeconds: 45
```

`terminationGracePeriodSeconds` is the total budget from `SIGTERM` (or `preStop`) to `SIGKILL`.
Either mechanism works; the in-process delay is better because it is visible in your code and
your logs, and it works identically outside Kubernetes.

## The testable run()

Everything above uses the same shape, so make it explicit as the thing to copy:

```go
func main() {
  if err := run(context.Background(), os.Args[1:], os.Stdout); err != nil {
    fmt.Fprintln(os.Stderr, "fatal:", err)
    os.Exit(1)
  }
}

func run(ctx context.Context, args []string, out io.Writer) error {
  cfg, err := config.Load(args)
  if err != nil {
    return fmt.Errorf("config: %w", err)
  }
  ctx, stop := signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)
  defer stop()

  app, err := newApp(ctx, cfg, out)
  if err != nil {
    return err
  }
  return app.Serve(ctx)
}
```

Three properties make this worth the indirection. **Only `main` calls `os.Exit`**, so every
`defer` in `run` actually runs — `os.Exit` skips them all
([3.5](../03-control-flow/05-defer.md)), which is the bug behind "my cleanup never happens".
**`run` returns an error**, so failures compose with `%w` instead of calling `log.Fatal` from
four levels down. And **`run` is callable from a test**: pass a cancellable context and a
`bytes.Buffer`, cancel it, and assert the process shut down cleanly — an integration test for
your shutdown path, which is otherwise the least-tested code in the service.

## Quick Reference

| Call | Purpose |
|---|---|
| `signal.NotifyContext(ctx, sigs...)` | `(ctx, stop)`; ctx cancelled on first signal |
| `signal.Notify(ch, sigs...)` | older form; `ch` **must** be buffered |
| `signal.Reset(sigs...)` / `stop()` | restore default behaviour, so a second signal kills |
| `srv.Shutdown(ctx)` | close listeners, drain in-flight, wait or return `ctx.Err()` |
| `srv.Close()` | close listeners **and** all connections immediately |
| `srv.RegisterOnShutdown(f)` | run `f` when `Shutdown` is called (e.g. close WebSockets) |
| `http.ErrServerClosed` | the expected `ListenAndServe` return after shutdown |
| `srv.SetKeepAlivesEnabled(false)` | stop new keep-alive reuse without closing the listener |
| `errors.Join(errs...)` | collect teardown failures without stopping |

| Signal | Sent by | Trap it? |
|---|---|---|
| `SIGTERM` | Kubernetes, systemd, `docker stop` | **yes** |
| `SIGINT` | Ctrl-C | yes |
| `SIGHUP` | terminal close; conventionally "reload config" | optionally |
| `SIGQUIT` | Ctrl-\\ | no — the default goroutine dump is useful |
| `SIGKILL` | the orchestrator after the grace period | impossible |

| Order | Step |
|---|---|
| 1 | readiness → NOT ready |
| 2 | wait `PreStopDelay` while still serving |
| 3 | `srv.Shutdown(ctx)` with a timeout |
| 4 | stop and drain workers |
| 5 | close the DB pool |
| 6 | flush metrics and logs |

## Common Mistakes

**Treating `http.ErrServerClosed` as a failure.** Every clean shutdown then logs an error and
exits non-zero; check it with `errors.Is` and return nil.

**Calling `Shutdown` immediately after flipping readiness.** The load balancer has not noticed
yet, so in-flight-bound requests get connection-refused; wait longer than
`periodSeconds × failureThreshold` first.

**Closing the DB pool before `Shutdown` returns.** Draining requests fail with "pool closed";
close it after the HTTP server has drained.

**An unbuffered channel with `signal.Notify`.** The signal is delivered non-blockingly and
dropped; use a buffered channel, or `NotifyContext`.

**A shutdown timeout longer than `terminationGracePeriodSeconds`.** `SIGKILL` arrives first and
you lose both the drain and the log line explaining why; keep it comfortably under.

**Handlers that ignore `r.Context()`.** They cannot be drained, so `Shutdown` always times out;
select on `r.Context().Done()` and pass the context to every downstream call.

**Calling `os.Exit` or `log.Fatal` deep in the code.** Every `defer` is skipped, including
`pool.Close()` and your metrics flush; return an error to `main`.

**Forgetting background goroutines.** They keep running or die mid-operation; give each one a
context and wait for it in teardown.

## Exercises

1. Add a `PreStopDelay` config value to the canonical `run()` and prove the point for real:
   with the delay at zero, send `SIGTERM` and immediately fire a request, capturing the
   connection-refused error; with the delay at 2 seconds, show the same request succeeding.
2. Write a test that calls `run(ctx, ...)` with a cancellable context, waits for the server to
   be listening, cancels, and asserts `run` returns nil within the timeout. No signals, no
   subprocess.
3. Make the worker pool in the teardown example drain on `ctx` rather than on `close(jobs)`,
   so a 30-second job is abandoned when the deadline passes. Show the `errors.Join` output
   containing both the worker failure and a successful DB close.
4. Register a `RegisterOnShutdown` callback that closes all open WebSocket connections with a
   normal-closure frame, and verify a connected client receives it rather than an abrupt reset.
5. Instrument shutdown: a counter for shutdowns by outcome (`drained`/`timed_out`) and a
   histogram of drain duration ([18.7](07-prometheus.md)). Flush them before exiting and prove
   the flush happens even when `Shutdown` times out.

---

[← 18.4 Structured Logging in a Service](04-service-logging.md) · [Course Index](../README.md) · [18.6 Health Checks and Readiness →](06-health-checks.md)
