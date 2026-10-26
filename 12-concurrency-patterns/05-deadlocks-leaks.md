[← 12.4 Rate Limiting and Semaphores](04-rate-limiting.md) · [Course Index](../README.md) · [12.6 The Go Memory Model in Practice →](06-memory-model.md)

# 12.5 — Deadlocks, Leaks, and How to Spot Them

## What You'll Learn

- The four deadlock shapes, each with its real `fatal error` and how to read the dump
- Why Go only reports a deadlock when **every** goroutine is blocked — and what to do about the ones that just hang
- Four leaks, each proven with `runtime.NumGoroutine` before and after, then fixed
- The diagnosis toolkit: SIGQUIT, `GOTRACEBACK`, `net/http/pprof`, and `goleak` in tests

## Coming From Another Language

| Other | Go |
|---|---|
| JS: a never-resolving Promise is silently collected | a blocked goroutine is **never** collected; it leaks until exit |
| Python `asyncio` warns "Task was destroyed but it is pending" | no warning at all unless you look |
| Rust: a leaked task needs `JoinHandle` discipline | same discipline, no compiler help |
| JS single thread: no lock-ordering deadlocks | real mutexes, so real AB/BA deadlocks |
| A thread dump via jstack | `kill -QUIT` or `/debug/pprof/goroutine` |

Go's runtime detects *one* condition — all goroutines blocked at once — and panics with a
dump that says exactly where. But a real service always has a runnable goroutine (an HTTP
listener, a ticker), so that detector essentially never fires in production and a stuck
subsystem just goes quiet. This lesson covers reading the dump when you get one, and
finding the problem when you do not.

## Deadlock 1: send with no receiver

```go
package main

import "fmt"

func main() {
  ch := make(chan int)
  fmt.Println("about to send")
  ch <- 1
  fmt.Println("never reached")
}
```

This program deadlocks on purpose. Output:

```
about to send
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/tmp/scratch/main.go:8 +0x70
exit status 2
```

`about to send` prints, so execution reached line 8. The state `[chan send]` plus
`/tmp/scratch/main.go:8` is the whole diagnosis: an unbuffered send blocks until someone
receives, and there is nobody else to do it.

## Deadlock 2: receive with no sender

The mirror image: the worker finishes and exits, so by the time `main` reaches `<-ch` it is
the only goroutine left.

```go
package main

import "fmt"

func main() {
  ch := make(chan int)
  done := make(chan struct{})
  go func() {
    defer close(done)
    fmt.Println("worker: waiting for a value")
  }()
  <-done
  v := <-ch
  fmt.Println("never reached", v)
}
```

This program deadlocks on purpose. Output:

```
worker: waiting for a value
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
	/tmp/scratch/main.go:13 +0xa4
exit status 2
```

`[chan receive]` at line 13. What makes this detectable is that the worker *exited*; had it
still been running, the runtime would have stayed quiet.

## Deadlock 3: two mutexes in opposite order

The classic AB/BA: one goroutine takes `a` then wants `b`, the other takes `b` then wants
`a`. The `time.Sleep` only makes it reliable — without it the deadlock is intermittent,
which is worse.

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

func main() {
  var a, b sync.Mutex
  var wg sync.WaitGroup
  wg.Add(2)

  go func() {
    defer wg.Done()
    a.Lock()
    time.Sleep(10 * time.Millisecond)
    b.Lock()
    b.Unlock()
    a.Unlock()
  }()

  go func() {
    defer wg.Done()
    b.Lock()
    time.Sleep(10 * time.Millisecond)
    a.Lock()
    a.Unlock()
    b.Unlock()
  }()

  wg.Wait()
  fmt.Println("never reached")
}
```

This program deadlocks on purpose. Output (truncated after the first two goroutines — the
real dump continues with goroutine 20 in the same shape):

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
sync.runtime_SemacquireWaitGroup(0x1049742f0?, 0x80?)
	GOROOT/src/runtime/sema.go:114 +0x38
sync.(*WaitGroup).Wait(0x697cdd710030)
	GOROOT/src/sync/waitgroup.go:206 +0xa4
main.main()
	/tmp/scratch/main.go:32 +0x110

goroutine 19 [sync.Mutex.Lock]:
internal/sync.runtime_SemacquireMutex(0x0?, 0x0?, 0x0?)
	GOROOT/src/runtime/sema.go:95 +0x28
internal/sync.(*Mutex).lockSlow(0x697cdd710028)
	GOROOT/src/internal/sync/mutex.go:149 +0x174
internal/sync.(*Mutex).Lock(...)
	GOROOT/src/internal/sync/mutex.go:70
sync.(*Mutex).Lock(...)
	GOROOT/src/sync/mutex.go:46
main.main.func1()
	/tmp/scratch/main.go:18 +0xfc
created by main.main in goroutine 1
...
exit status 2
```

Three goroutines, three states: `main` in `[sync.WaitGroup.Wait]` at line 32, and both
workers in `[sync.Mutex.Lock]` at lines 18 and 27. Those two line numbers *are* the bug
report: line 18 is `b.Lock()` in the first goroutine, line 27 is `a.Lock()` in the second.
Each holds what the other wants.

The fix is not a smarter lock, it is a **global lock order**: every path that needs both
mutexes takes them in the same order, which makes a cycle impossible. Better still, do not
hold two locks at once — mutexes that always travel together should be one mutex.

## Deadlock 4: WaitGroup.Wait with a missing Done

An early `return` that skips `wg.Done()`: the `defer` sits after it, so it never registers for `i == 2`.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var wg sync.WaitGroup
  for i := 0; i < 3; i++ {
    wg.Add(1)
    go func() {
      if i == 2 {
        return
      }
      defer wg.Done()
      fmt.Println("worked", i)
    }()
  }
  wg.Wait()
  fmt.Println("never reached")
}
```

This program deadlocks on purpose. Output:

```
worked 0
worked 1
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
sync.runtime_SemacquireWaitGroup(0x104dac200?, 0x30?)
	GOROOT/src/runtime/sema.go:114 +0x38
sync.(*WaitGroup).Wait(0x404315c1e020)
	GOROOT/src/sync/waitgroup.go:206 +0xa4
main.main()
	/tmp/scratch/main.go:20 +0xb8
exit status 2
```

`worked 0` and `worked 1` print, then `main` hangs in `[sync.WaitGroup.Wait]` at line 20
forever: the counter is 1 and nothing will decrement it. `defer wg.Done()` belongs on the
**first** line of the goroutine, before any branch that can return.

## How to read the dump

Every dump has the same structure, and three fields do the work:

| Field | Meaning |
|---|---|
| `fatal error: all goroutines are asleep` | the runtime's deadlock detector fired |
| `goroutine 1` | goroutine ID; 1 is always `main` |
| `[chan send]` | **the state — read this first.** Why it is blocked |
| `main.main()` + `file:line` | where it is blocked; read bottom-up for the call chain |
| `created by ... in goroutine N` | who launched it, for non-main goroutines |

The states you will actually see:

| State | Means |
|---|---|
| `[chan send]` | blocked sending; no receiver, or buffer full |
| `[chan receive]` | blocked receiving; no sender and not closed |
| `[select]` / `[select (no cases)]` | no case ready / `select {}`, blocked by construction |
| `[sync.WaitGroup.Wait]` / `[sync.Mutex.Lock]` | counter never hit zero / someone holds it |
| `[IO wait]` / `[sleep]` | runnable — **not** a deadlock |

Method: find every goroutine in a blocking state, note the file:line, look for the cycle.
With four goroutines that takes a minute; with four hundred, use pprof.

## The crucial limitation: Go only detects a TOTAL deadlock

`fatal error: all goroutines are asleep` means exactly what it says. If even one goroutine
is runnable — a `time.Sleep`, an HTTP listener, a ticker — the runtime sees a live program
and **says nothing**, even though a subsystem is permanently wedged. That is a *partial
deadlock*, and it is what you will actually meet in production.

The program below wedges a sender permanently while `main` sits in `select {}`, and the
runtime never complains. To bound it I added a **watchdog goroutine that sleeps 2 seconds,
dumps all stacks with `runtime.Stack(buf, true)`, and calls `os.Exit(3)`** — so the output
below comes from a program deliberately killed, not one that finished.

```go
package main

import (
  "fmt"
  "os"
  "runtime"
  "time"
)

func main() {
  go func() {
    time.Sleep(2 * time.Second)
    fmt.Println("watchdog: still running after 2s, so this is NOT a full deadlock")
    fmt.Println("watchdog: goroutines =", runtime.NumGoroutine())
    buf := make([]byte, 1<<16)
    n := runtime.Stack(buf, true)
    fmt.Printf("watchdog: dumping all goroutine stacks\n%s", buf[:n])
    os.Exit(3)
  }()

  results := make(chan int)
  go func() {
    for i := 0; i < 5; i++ {
      results <- i * i
    }
  }()

  fmt.Println("got:", <-results)
  fmt.Println("got:", <-results)

  select {}
}
```

This program would hang forever; the watchdog kills it after 2 seconds. Output:

```
got: 0
got: 1
watchdog: still running after 2s, so this is NOT a full deadlock
watchdog: goroutines = 3
watchdog: dumping all goroutine stacks
goroutine 5 [running]:
main.main.func1()
	/tmp/scratch/main.go:16 +0x144
created by main.main in goroutine 1
	/tmp/scratch/main.go:11 +0x28

goroutine 1 [select (no cases)]:
main.main()
	/tmp/scratch/main.go:31 +0x144

goroutine 6 [chan send]:
main.main.func2()
	/tmp/scratch/main.go:24 +0x3c
created by main.main in goroutine 1
	/tmp/scratch/main.go:22 +0x74
exit status 3
```

Three goroutines, no fatal error. `goroutine 20 [chan send]` at line 24 is the stuck
producer: it sent two of five values and nobody will receive the third. `goroutine 1` is
`main`, blocked but not counted as a deadlock because the watchdog stayed runnable.

Without the watchdog this process would sit at 0% CPU forever, no log line and no error.
That is the failure mode to recognise: **not a crash, just silence.**

| Situation | Does Go report it? |
|---|---|
| all goroutines blocked on channels/mutexes | yes — `fatal error: all goroutines are asleep` |
| one goroutine in `time.Sleep`, rest deadlocked | **no** |
| a single goroutine leaked in an otherwise healthy service | **no** |
| two mutexes taken in opposite order, nothing else running | yes |

The detector is a development-time luxury. In production you need the toolkit below.

## Leak 1: blocked forever on a channel nobody writes

Each demo runs as its own process (`go run . 1leak`) so every count starts from a clean
baseline. A goroutine blocked on a channel is **not garbage** — it is live, it holds its
stack and everything its closure references, and `runtime.GC()` never reclaims it.

`count(label)` below sleeps 50ms, calls `runtime.GC()`, then prints
`runtime.NumGoroutine()`:

```go
func leakBlockedReceive() {
  fmt.Println("1. goroutine blocked on a channel nobody writes:")
  count("before")
  for i := 0; i < 3; i++ {
    ch := make(chan int)
    go func() {
      <-ch
    }()
  }
  count("after 3 goroutines")
}

func fixBlockedReceive() {
  for i := 0; i < 3; i++ {
    ch := make(chan int)
    go func() {
      <-ch
    }()
    close(ch)
  }
}
```

Output:

```
1. goroutine blocked on a channel nobody writes:
  before                       1
  after 3 goroutines           4
   fixed with a close:
  before                       1
  after 3 goroutines           1
```

Three goroutines created, three still there. Each `fix*` function below makes the same
`count` calls as its `leak*` twin; those lines are elided. `close(ch)` fixes this one
because a receive on a closed channel returns immediately (10.4).

## Leak 2: a goroutine that ignores its context

Having a `ctx` is not the same as using it: `cancel()` fires and the workers never notice, because `range work` has no `select`.

```go
func leakIgnoredCtx() {
  fmt.Println("2. goroutine ignoring ctx:")
  count("before")
  ctx, cancel := context.WithCancel(context.Background())
  work := make(chan int)
  for i := 0; i < 3; i++ {
    go func() {
      for v := range work {
        _ = v
      }
    }()
  }
  cancel()
  _ = ctx
  count("after cancel")
}

func fixIgnoredCtx() {
  ctx, cancel := context.WithCancel(context.Background())
  work := make(chan int)
  for i := 0; i < 3; i++ {
    go func() {
      for {
        select {
        case <-ctx.Done():
          return
        case v := <-work:
          _ = v
        }
      }
    }()
  }
  cancel()
}
```

Output:

```
2. goroutine ignoring ctx:
  before                       1
  after cancel                 4
   fixed by selecting on ctx.Done():
  before                       1
  after cancel                 1
```

This is the leak that survives code review, because the function *takes* a context and
looks cancellable. The fix is the `select` on `ctx.Done()`, exactly as in 12.3.

## Leak 3: a forgotten time.Tick

`time.Tick` has no `Stop` — the documentation says outright that it leaks the underlying
ticker, so it suits only programs that tick until they exit.

```go
func leakTick() {
  fmt.Println("3. forgotten time.Tick:")
  count("before")
  for i := 0; i < 3; i++ {
    go func() {
      for range time.Tick(10 * time.Millisecond) {
      }
    }()
  }
  count("after 3 tickers")
}

func fixTick() {
  done := make(chan struct{})
  for i := 0; i < 3; i++ {
    go func() {
      t := time.NewTicker(10 * time.Millisecond)
      defer t.Stop()
      for {
        select {
        case <-done:
          return
        case <-t.C:
        }
      }
    }()
  }
  close(done)
}
```

Output:

```
3. forgotten time.Tick:
  before                       1
  after 3 tickers              4
   fixed with NewTicker + Stop + done:
  before                       1
  after close(done)            1
```

`time.NewTicker` with `defer t.Stop()` and a `done` channel in the `select` is the correct
form. Use `time.Tick` only in a `main` that runs forever.

## Leak 4: an http.Response.Body never closed

The most expensive one in a real service, because every unclosed body also holds a
connection out of the pool. `fixBody` below uses the same `httptest` server and loop as
`leakBody`; only the two body-handling lines differ.

```go
func leakBody() {
  fmt.Println("4. http.Response.Body never closed:")
  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "hello")
  }))
  defer srv.Close()

  count("before")
  for i := 0; i < 5; i++ {
    resp, err := http.Get(srv.URL)
    if err != nil {
      return
    }
    _ = resp
  }
  count("after 5 unclosed bodies")
}

func fixBody() {
  for i := 0; i < 5; i++ {
    resp, err := http.Get(srv.URL)
    if err != nil {
      return
    }
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
  }
}
```

Output:

```
4. http.Response.Body never closed:
  before                       2
  after 5 unclosed bodies      17
   fixed with ReadAll + Close:
  before                       2
  after 5 closed bodies        5
```

Five unclosed bodies took the process from 2 goroutines to 17 — a reader and writer
goroutine per connection, none returned to the idle pool. The fixed version lands on 5, and
those are reusable pooled connections rather than leaks.

Both halves matter: `io.Copy(io.Discard, resp.Body)` **then** `resp.Body.Close()`. Closing
without draining works but prevents connection reuse; draining without closing is the leak
above. And `defer resp.Body.Close()` must come *after* the error check — `resp` is nil when
`err != nil`.

A fifth leak, the abandoned pipeline stage, is covered in 12.2 and 12.3: `break` out of a
`range` over a channel and everything upstream blocks on its next send. Taking 1 value of
1000 from a two-stage pipeline leaks both stages — 1 goroutine before, 3 after.

## Toolkit 1: NumGoroutine assertions in tests

The cheapest leak detector you can write. Snapshot the count, run the code, then poll with
a timeout — teardown is asynchronous, so a bare `NumGoroutine()` straight afterwards is
flaky. Two constructors to test against, reused by `goleak` below:

```go
package worker

import "context"

func StartLeaky(items <-chan int) {
  go func() {
    for v := range items {
      _ = v
    }
  }()
}

func StartClean(ctx context.Context, items <-chan int) {
  go func() {
    for {
      select {
      case <-ctx.Done():
        return
      case v, ok := <-items:
        if !ok {
          return
        }
        _ = v
      }
    }
  }()
}
```

The helper plus both tests:

```go
package worker

import (
  "context"
  "runtime"
  "testing"
  "time"
)

func assertNoLeak(t *testing.T, before int) {
  t.Helper()
  for i := 0; i < 50; i++ {
    if runtime.NumGoroutine() <= before {
      return
    }
    time.Sleep(10 * time.Millisecond)
  }
  buf := make([]byte, 1<<14)
  n := runtime.Stack(buf, true)
  t.Fatalf("leaked %d goroutines (before %d, now %d)\n%s",
    runtime.NumGoroutine()-before, before, runtime.NumGoroutine(), buf[:n])
}

func TestCleanNoLeak(t *testing.T) {
  before := runtime.NumGoroutine()
  ctx, cancel := context.WithCancel(context.Background())
  items := make(chan int)
  StartClean(ctx, items)
  items <- 1
  cancel()
  assertNoLeak(t, before)
}

func TestLeakyDetected(t *testing.T) {
  before := runtime.NumGoroutine()
  items := make(chan int)
  StartLeaky(items)
  items <- 1
  assertNoLeak(t, before)
}
```

Running the leaky one gives a real failure:

```
--- FAIL: TestLeakyDetected (0.58s)
    leak_test.go:39: leaked 1 goroutines (before 2, now 3)
        goroutine 35 [running]:
        example.com/worker.assertNoLeak(0x392c5e6e8248, 0x2)
        	/tmp/scratch/leak_test.go:19 +0xfc
        example.com/worker.TestLeakyDetected(0x392c5e6e8248)
        	/tmp/scratch/leak_test.go:39 +0xc8
...
```

Truncated — the real failure includes every goroutine's stack. Note the weakness: the
*leaked* goroutine is in there somewhere, but the dump leads with the asserting one.
`goleak` solves exactly that.

## Toolkit 2: SIGQUIT dumps every goroutine stack

`kill -QUIT <pid>` makes a Go process dump all goroutine stacks and exit. It works on any
Go binary with no instrumentation, which makes it the first thing to try on a wedged
process.

```go
package main

import (
  "fmt"
  "os"
  "time"
)

func stuckSender(ch chan int) {
  ch <- 42
}

func stuckReceiver(ch chan string) {
  <-ch
}

func main() {
  go stuckSender(make(chan int))
  go stuckReceiver(make(chan string))

  fmt.Fprintln(os.Stderr, "pid", os.Getpid(), "ready")
  time.Sleep(30 * time.Second)
}
```

Run it in the background, then signal it:

```bash
go build -o prog . && ./prog &
sleep 1
kill -QUIT $!
```

Output (truncated: the header, then the two stuck goroutines with their `runtime.*`
frames elided. The real dump also lists `main`, the GC and finalizer goroutines — 9 in
total — each with full register detail):

```
SIGQUIT: quit
PC=0x180787fc4 m=0 sigcode=0

...

goroutine 19 gp=0x469e65484780 m=nil [chan send]:
main.stuckSender(...)
	/tmp/scratch/main.go:10
main.main.gowrap1()
	/tmp/scratch/main.go:18 +0x28 fp=0x469e653c97d0 sp=0x469e653c97b0 pc=0x1028cba68
created by main.main in goroutine 1
	/tmp/scratch/main.go:18 +0x74

goroutine 20 gp=0x469e65484960 m=nil [chan receive]:
main.stuckReceiver(...)
	/tmp/scratch/main.go:14
main.main.gowrap2()
	/tmp/scratch/main.go:19 +0x24 fp=0x469e653c9fd0 sp=0x469e653c9fb0 pc=0x1028cba24
created by main.main in goroutine 1
	/tmp/scratch/main.go:19 +0xd0

...
exit status 2
```

`goroutine 19 [chan send]` and `goroutine 20 [chan receive]`, each with
`main.stuckSender` / `main.stuckReceiver` and a `created by` line pointing at the launch
site. Read past the `runtime.*` frames: the first `main.*` frame is your code.

A SIGQUIT dump carries `gp=0x...` pointers and `fp`/`sp`/`pc` registers. Ignore all of
that: the state in brackets and the first `main.` frame are what matter.

## Toolkit 3: GOTRACEBACK

`GOTRACEBACK` controls how much a crash prints. Measured against the two-mutex deadlock
above:

| Value | Effect | Measured |
|---|---|---|
| `none` | no stacks at all | just `fatal error: ...` + `exit status 2` |
| `all` (default for a deadlock) | all user goroutines | 3 goroutines in the dump |
| `system` | user plus runtime goroutines | 8 goroutines in the dump |

With `GOTRACEBACK=none` the entire output is:

```
fatal error: all goroutines are asleep - deadlock!
exit status 2
```

Set `GOTRACEBACK=all` explicitly in containers — some base images set `none`, which turns
a diagnosable crash into a mystery. Try `system` when a runtime or cgo deadlock is
suspected.

## Toolkit 4: net/http/pprof goroutine profile

For a running service this is the best tool, because it **groups identical stacks with a
count**: four hundred goroutines stuck in the same place appear as one entry reading
`400 @`.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "net/http/pprof"
  "runtime"
  "strings"
  "time"
)

func leakyHandler(stuck chan int) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {
    go func() {
      stuck <- 1
    }()
    fmt.Fprint(w, "ok")
  }
}

func main() {
  mux := http.NewServeMux()
  mux.Handle("/work", leakyHandler(make(chan int)))
  mux.HandleFunc("/debug/pprof/goroutine", pprof.Handler("goroutine").ServeHTTP)

  srv := httptest.NewServer(mux)
  defer srv.Close()

  for i := 0; i < 4; i++ {
    resp, err := http.Get(srv.URL + "/work")
    if err != nil {
      panic(err)
    }
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
  }
  time.Sleep(100 * time.Millisecond)
  fmt.Println("NumGoroutine after 4 requests:", runtime.NumGoroutine())

  resp, err := http.Get(srv.URL + "/debug/pprof/goroutine?debug=1")
  if err != nil {
    panic(err)
  }
  defer resp.Body.Close()
  b, _ := io.ReadAll(resp.Body)

  for _, l := range strings.Split(string(b), "\n") {
    if strings.HasPrefix(l, "goroutine profile:") || strings.Contains(l, "main.leakyHandler") || strings.HasPrefix(l, "4 @") {
      fmt.Println(l)
    }
  }
}
```

Output:

```
NumGoroutine after 4 requests: 9
goroutine profile: total 10
4 @ 0x100acc748 0x100a5d8a4 0x100a5d4b8 0x100cad938 0x100ad3784
#	0x100cad937	main.leakyHandler.func1.1+0x27	/tmp/scratch/main.go:17
```

`4 @ ...` means four goroutines share that identical stack, and the `#` line names
`main.leakyHandler.func1` at line 17. In a real service, import the package for its side
effects and the handlers register on `DefaultServeMux`:

```go
import _ "net/http/pprof"

go func() {
  http.ListenAndServe("localhost:6060", nil)
}()
```

Then:

```bash
curl 'localhost:6060/debug/pprof/goroutine?debug=1'
curl 'localhost:6060/debug/pprof/goroutine?debug=2'
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

`debug=1` is the grouped form above; `debug=2` gives per-goroutine stacks with states and
wait durations, like SIGQUIT. **Never expose this port publicly.** 15.6 covers pprof's CPU
and heap profiles.

Diagnosing a leak is a two-sample job: fetch, wait, fetch again, and look for the count
that grew. A steady-state service has a stable goroutine profile; a leaking one has an
entry whose count tracks request volume.

## Toolkit 5: goleak in tests

`go.uber.org/goleak` (v1.3.0) does this properly: it ignores the runtime's own goroutines,
retries with a backoff, and reports **the leaked goroutine's** stack.

```go
func TestMain(m *testing.M) {
  goleak.VerifyTestMain(m)
}
```

The test file, over the same `StartLeaky`/`StartClean` from Toolkit 1:

```go
package worker

import (
  "context"
  "testing"
  "time"

  "go.uber.org/goleak"
)

func TestMain(m *testing.M) {
  goleak.VerifyTestMain(m)
}

func TestStartLeaky(t *testing.T) {
  items := make(chan int)
  StartLeaky(items)
  items <- 1
  time.Sleep(10 * time.Millisecond)
}

func TestStartClean(t *testing.T) {
  ctx, cancel := context.WithCancel(context.Background())
  items := make(chan int)
  StartClean(ctx, items)
  items <- 1
  cancel()
  time.Sleep(10 * time.Millisecond)
}
```

`go test ./...` really fails:

```
PASS
goleak: Errors on successful test run: found unexpected goroutines:
[Goroutine 36 in state chan receive, with example.com/worker.StartLeaky.func1 on top of the stack:
example.com/worker.StartLeaky.func1()
	/tmp/scratch/worker.go:7 +0x34
created by example.com/worker.StartLeaky in goroutine 35
	/tmp/scratch/worker.go:6 +0x64
]
FAIL	example.com/worker	0.981s
FAIL
```

That is the whole diagnosis in four lines: state `chan receive`, function
`StartLeaky.func1`, leaked from `worker.go:7`, launched at `worker.go:6`. Note that `PASS`
prints *before* the failure — the tests themselves passed, and `goleak` failed the package
afterwards.

Running only the clean test passes:

```
ok  	example.com/worker	0.310s
```

`goleak.VerifyNone(t)` with `defer` checks a single test instead of the whole package, and
`goleak.IgnoreTopFunction(...)` silences known-noisy third-party goroutines without giving
up the check.

## Quick Reference

| Tool | Use | Cost |
|---|---|---|
| `fatal error: all goroutines are asleep` | total deadlock, dev time | free, but never fires in production |
| `runtime.NumGoroutine()` | a quick before/after in a test | free; flaky without a poll loop |
| `kill -QUIT <pid>` | wedged process, no instrumentation | free; kills the process |
| `net/http/pprof` goroutine profile | running service, grouped counts | one import plus a port |
| `go.uber.org/goleak` | catch leaks in CI | one dependency |

| Symptom | First move |
|---|---|
| `fatal error: all goroutines are asleep` | read the states, find the cycle |
| process at 0% CPU, no output | `kill -QUIT`, read the `[chan ...]` states |

## Common Mistakes

**Assuming no `fatal error` means no deadlock.** Go reports a deadlock only when *every*
goroutine is blocked. A partial deadlock in a live service is completely silent.

**`defer wg.Done()` placed after a conditional `return`.** It never registers on that path
and `Wait` hangs forever. It belongs on the first line of the goroutine.

**Taking two mutexes in whatever order reads nicely.** Pick one global order and follow it
everywhere, or merge the two mutexes into one.

**`defer resp.Body.Close()` before checking `err`.** `resp` is nil on error, so the defer
panics. Check the error, then defer the close.

**Trusting `runtime.NumGoroutine()` immediately after cancelling.** Teardown is
asynchronous; poll with a timeout, or use `goleak`.

**Using `time.Tick` anywhere but `main`.** It cannot be stopped, so every call leaks a
ticker. `time.NewTicker` plus `defer Stop()`.

**Treating `[IO wait]` or `[sleep]` goroutines as leaks.** Those are runnable. Look for
`[chan send]`, `[chan receive]`, `[select]` and `[sync.*]`.

## The checklist

Most of this module's failure modes are prevented by seven habits:

1. **`defer` the release immediately after the acquire** — `wg.Done()`, `mu.Unlock()`,
   `<-sem`, `resp.Body.Close()`, `ticker.Stop()`. First line, before any branch.
2. **Every goroutine has a defined way to exit.** If you cannot name it, it is a leak.
3. **Every send on a channel whose lifetime you do not own is a `select` with
   `ctx.Done()`.**
4. **Whoever creates a channel closes it**, exactly once, and never a worker.
5. **One global lock order** — or better, never hold two locks at once.
6. **`defer cancel()` on every context you create**, plus an explicit `cancel()` before an
   early `break`.
7. **`goleak` in CI and `-race` on every test run.** Both catch what review misses.

## Exercises

1. Take deadlock 4 and fix it three ways: move the `defer`, call `wg.Add` after the branch,
   and switch to `errgroup`. Which one still works if the goroutine panics?
2. Add a watchdog to the partial-deadlock program that dumps stacks but does *not* exit,
   printing only goroutines whose state contains `chan`.
3. Build an `httptest` server whose handler leaks one goroutine per request. Diff
   `/debug/pprof/goroutine?debug=1` before and after 50 requests, and write an assertion
   that fails when any stack's count grows.
4. Add `goleak.VerifyTestMain` to a package of your own and fix every leak it reports.
   For any you cannot (a third-party background goroutine), use `IgnoreTopFunction`.
5. Reproduce the two-mutex deadlock with the `time.Sleep` calls removed and run it 100
   times. How often does it actually deadlock, and what does that say about catching this
   class of bug in CI?

---

[← 12.4 Rate Limiting and Semaphores](04-rate-limiting.md) · [Course Index](../README.md) · [12.6 The Go Memory Model in Practice →](06-memory-model.md)
