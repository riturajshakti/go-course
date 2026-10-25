[← 11.8 errgroup](../11-sync-context/08-errgroup.md) · [Course Index](../README.md) · [12.2 Fan-Out and Fan-In →](02-fan-out-fan-in.md)

# 12.1 — Worker Pools

## What You'll Learn

- The canonical jobs-channel + results-channel pool, and why `close(results)` must happen after `wg.Wait()`
- Picking a worker count: `NumCPU` for CPU-bound work, far higher for I/O-bound work
- Getting errors and per-job context cancellation out of workers
- Why `go doWork(item)` in a loop over a million items is not a pool

## Coming From Another Language

| Other | Go |
|---|---|
| JS `Promise.all(items.map(f))` — unbounded | a pool, or `errgroup.SetLimit(n)` |
| JS `p-limit(5)` | a buffered channel used as a semaphore |
| Python `ThreadPoolExecutor(max_workers=8)` | N goroutines ranging over one channel |
| Python `concurrent.futures` returns futures | Go returns values on a channel; you correlate them yourself |
| Rust `rayon::par_iter()` | no equivalent; you build the pool by hand (it is ~15 lines) |

The parts are from modules 10 and 11: a channel (10.2), `range` over a channel until
it closes (10.4), and `sync.WaitGroup` (11.1). A pool is just those three composed. What
is new here is the *shape* — who closes which channel, and when — because getting that
wrong is how you deadlock.

## The canonical pool

Jobs go in one channel, results come out another. Workers `range` over jobs, so closing
`jobs` is how you tell every worker there is no more work. Results arrive in completion
order, so collect and sort before printing.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
)

type job struct {
  id int
  n  int
}

type result struct {
  jobID int
  value int
}

func isPrime(n int) bool {
  if n < 2 {
    return false
  }
  for d := 2; d*d <= n; d++ {
    if n%d == 0 {
      return false
    }
  }
  return true
}

func countPrimes(limit int) int {
  c := 0
  for i := 2; i <= limit; i++ {
    if isPrime(i) {
      c++
    }
  }
  return c
}

func worker(jobs <-chan job, results chan<- result, wg *sync.WaitGroup) {
  defer wg.Done()
  for j := range jobs {
    results <- result{jobID: j.id, value: countPrimes(j.n)}
  }
}

func main() {
  const numWorkers = 4
  jobs := make(chan job)
  results := make(chan result)

  var wg sync.WaitGroup
  for i := 0; i < numWorkers; i++ {
    wg.Add(1)
    go worker(jobs, results, &wg)
  }

  go func() {
    for i, n := range []int{10000, 20000, 30000, 40000, 50000, 60000, 70000, 80000} {
      jobs <- job{id: i, n: n}
    }
    close(jobs)
  }()

  go func() {
    wg.Wait()
    close(results)
  }()

  var got []result
  for r := range results {
    got = append(got, r)
  }
  sort.Slice(got, func(i, j int) bool { return got[i].jobID < got[j].jobID })
  for _, r := range got {
    fmt.Printf("job %d -> %d primes\n", r.jobID, r.value)
  }
  fmt.Println("jobs done:", len(got))
}
```

Output:

```
job 0 -> 1229 primes
job 1 -> 2262 primes
job 2 -> 3245 primes
job 3 -> 4203 primes
job 4 -> 5133 primes
job 5 -> 6057 primes
job 6 -> 6935 primes
job 7 -> 7837 primes
jobs done: 8
```

Three rules are doing all the work:

1. The **producer** owns `jobs` and closes it. Workers only read it, which is why the
   parameter is `<-chan job`.
2. Each worker calls `wg.Done()` exactly once, via `defer`, when its `range` ends.
3. **`close(results)` happens in its own goroutine, after `wg.Wait()`.** Nobody else may
   close it, because no single worker knows it is the last one.

The directional types (`<-chan`, `chan<-`) are not decoration: they make `close(jobs)`
inside `worker` a compile error.

## Closing results too early deadlocks

Write the pool sequentially instead, and it looks fine until you run it.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  jobs := make(chan int)
  results := make(chan int)

  var wg sync.WaitGroup
  for i := 0; i < 2; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      for n := range jobs {
        results <- n * n
      }
    }()
  }

  for n := 1; n <= 5; n++ {
    jobs <- n
  }
  close(jobs)
  wg.Wait()
  close(results)

  for r := range results {
    fmt.Println(r)
  }
}
```

This program deadlocks on purpose. Output (goroutine dump truncated):

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/tmp/scratch/main.go:24 +0x100

goroutine 35 [chan send]:
main.main.func1()
	/tmp/scratch/main.go:18 +0x6c
created by main.main in goroutine 1
	/tmp/scratch/main.go:15 +0x6c
...
exit status 2
```

`main` is at line 24 (`jobs <- n`) and the workers are at line 18 (`results <- n*n`).
Nobody is reading `results`, because the only reader is the loop at the bottom of `main`
— which `main` cannot reach while it is blocked feeding `jobs`. Hence the rule: the
goroutine that collects results must not also be the one feeding jobs, or `results` must
be buffered large enough to hold every result.

Idiomatic: feed in a goroutine, collect in `main`, and close results in a third goroutine
after `wg.Wait()`.

## Choosing the worker count

CPU-bound work saturates at roughly `runtime.NumCPU()` — there is nothing left to
parallelise once every core is busy, and more goroutines just add scheduling churn.
I/O-bound work is the opposite: a goroutine waiting on a socket costs a few KB of stack
and zero CPU, so N workers give N-way concurrency well past the core count. The ceiling
there is not the CPU, it is the remote service's connection limit and your own file
descriptors. Rule of thumb: `NumCPU()` for compute, hundreds for network calls, and
measure.

```go
package main

import (
  "fmt"
  "runtime"
  "sync"
  "time"
)

func runPool(numWorkers, numJobs int, work func()) time.Duration {
  jobs := make(chan int)
  var wg sync.WaitGroup
  for i := 0; i < numWorkers; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      for range jobs {
        work()
      }
    }()
  }
  start := time.Now()
  for i := 0; i < numJobs; i++ {
    jobs <- i
  }
  close(jobs)
  wg.Wait()
  return time.Since(start)
}

func spin(d time.Duration) {
  end := time.Now().Add(d)
  for time.Now().Before(end) {
  }
}

func main() {
  fmt.Println("NumCPU:", runtime.NumCPU())

  fmt.Println("CPU-bound, 64 jobs of 10ms busy-spin:")
  for _, n := range []int{1, 2, 4, 8, 16, 64} {
    d := runPool(n, 64, func() { spin(10 * time.Millisecond) })
    fmt.Printf("  %2d workers: %v\n", n, d.Round(10*time.Millisecond))
  }

  fmt.Println("I/O-bound, 64 jobs of 10ms sleep:")
  for _, n := range []int{1, 2, 4, 8, 16, 64} {
    d := runPool(n, 64, func() { time.Sleep(10 * time.Millisecond) })
    fmt.Printf("  %2d workers: %v\n", n, d.Round(10*time.Millisecond))
  }
}
```

Output (on a 12-core machine; absolute timings depend on your hardware):

```
NumCPU: 12
CPU-bound, 64 jobs of 10ms busy-spin:
   1 workers: 640ms
   2 workers: 320ms
   4 workers: 160ms
   8 workers: 80ms
  16 workers: 60ms
  64 workers: 60ms
I/O-bound, 64 jobs of 10ms sleep:
   1 workers: 700ms
   2 workers: 350ms
   4 workers: 180ms
   8 workers: 90ms
  16 workers: 40ms
  64 workers: 10ms
```

The CPU-bound column flattens at 60ms — 64 jobs of 10ms over 12 cores is ~53ms of work,
so 16 workers already reaches the floor and 64 buys nothing. The I/O-bound column keeps
halving all the way to 64 workers, where all 64 sleeps overlap and the whole batch takes
one sleep's worth of time.

## Buffering the jobs channel

A buffered `jobs` channel decouples the producer from the workers: the producer runs
ahead and returns early. It does not make the work finish sooner. Replacing
`make(chan int)` with `make(chan int, N)` in a 2-worker pool over 20 jobs of 10ms:

```
buffer  0: producer returned after 100ms, all work done after 110ms
buffer 10: producer returned after  40ms, all work done after 110ms
buffer 20: producer returned after    0s, all work done after 110ms
```

Total is 110ms in all three cases. Unbuffered means the producer blocks until a worker
takes each job, which is backpressure — usually what you want, since a queue otherwise
hides the fact that you produce faster than you consume. Buffer when the producer has
other work to do, or when job production is bursty.

## Errors out of workers

A worker cannot `return err`; it has no caller. Put the error in the result struct.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "sort"
  "strings"
  "sync"
)

type fetched struct {
  path string
  size int
  err  error
}

func main() {
  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    switch r.URL.Path {
    case "/missing":
      http.Error(w, "nope", http.StatusNotFound)
    default:
      fmt.Fprint(w, strings.Repeat("x", 10*len(r.URL.Path)))
    }
  }))
  defer srv.Close()

  paths := []string{"/a", "/bb", "/ccc", "/dddd", "/missing", "/ffffff", "/ggggggg"}

  jobs := make(chan string)
  results := make(chan fetched)

  var wg sync.WaitGroup
  for i := 0; i < 3; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      for p := range jobs {
        results <- fetch(srv.URL, p)
      }
    }()
  }

  go func() {
    for _, p := range paths {
      jobs <- p
    }
    close(jobs)
  }()
  go func() { wg.Wait(); close(results) }()

  var got []fetched
  for r := range results {
    got = append(got, r)
  }
  sort.Slice(got, func(i, j int) bool { return got[i].path < got[j].path })
  for _, r := range got {
    if r.err != nil {
      fmt.Printf("%-9s ERROR %v\n", r.path, r.err)
      continue
    }
    fmt.Printf("%-9s %d bytes\n", r.path, r.size)
  }
}

func fetch(base, path string) fetched {
  resp, err := http.Get(base + path)
  if err != nil {
    return fetched{path: path, err: err}
  }
  defer resp.Body.Close()
  if resp.StatusCode != http.StatusOK {
    return fetched{path: path, err: fmt.Errorf("status %s", resp.Status)}
  }
  b, err := io.ReadAll(resp.Body)
  return fetched{path: path, size: len(b), err: err}
}
```

Output:

```
/a        20 bytes
/bb       30 bytes
/ccc      40 bytes
/dddd     50 bytes
/ffffff   70 bytes
/ggggggg  80 bytes
/missing  ERROR status 404 Not Found
```

The server is a real `httptest.NewServer`, so these are real HTTP round-trips over
loopback. Note `defer resp.Body.Close()` — a pool that skips it leaks a goroutine per
request (12.5).

Idiomatic: a result struct carrying `err` when you want *all* results including the
failures. Use `errgroup` when the first error should abort everything.

## The errgroup version

`errgroup.SetLimit(n)` turns `errgroup` (11.8) into a bounded pool with error propagation
built in. Write results into a pre-sized slice at the item's own index — each goroutine
owns one slot, so there is no race and no sorting.

```go
package main

import (
  "context"
  "fmt"
  "strings"

  "golang.org/x/sync/errgroup"
)

func main() {
  words := []string{"go", "channel", "select", "", "mutex", "atomic"}
  lengths := make([]int, len(words))

  g, ctx := errgroup.WithContext(context.Background())
  g.SetLimit(3)
  for i, w := range words {
    g.Go(func() error {
      if w == "" {
        return fmt.Errorf("word %d is empty", i)
      }
      select {
      case <-ctx.Done():
        return ctx.Err()
      default:
      }
      lengths[i] = len(strings.ToUpper(w))
      return nil
    })
  }
  err := g.Wait()
  fmt.Println("err:", err)
  fmt.Println("lengths:", lengths)
}
```

Output:

```
err: word 3 is empty
lengths: [2 7 6 0 5 6]
```

`g.Wait()` returns the first non-nil error and waits for every goroutine regardless, so
the other five slots are filled. Slot 3 keeps its zero value. `go run -race` is clean:
distinct indices of a slice are distinct memory.

This is roughly 20 lines shorter than the channel version. Reach for it unless you need
streaming results or a long-lived pool.

## A pool that runs until the context is cancelled

For a pool draining a queue of unknown length, the shutdown signal is a context, not
`close(jobs)`. Each worker checks `ctx.Done()` before starting a job.

```go
package main

import (
  "context"
  "fmt"
  "sort"
  "sync"
  "sync/atomic"
  "time"
)

func main() {
  ctx, cancel := context.WithTimeout(context.Background(), 120*time.Millisecond)
  defer cancel()

  jobs := make(chan int)
  var mu sync.Mutex
  var done []int
  var abandoned atomic.Int64

  var wg sync.WaitGroup
  for i := 0; i < 4; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      for n := range jobs {
        select {
        case <-ctx.Done():
          abandoned.Add(1)
          continue
        default:
        }
        time.Sleep(20 * time.Millisecond)
        mu.Lock()
        done = append(done, n)
        mu.Unlock()
      }
    }()
  }

  for n := 1; n <= 40; n++ {
    jobs <- n
  }
  close(jobs)
  wg.Wait()

  sort.Ints(done)
  fmt.Println("ctx err:", ctx.Err())
  fmt.Println("completed:", len(done))
  fmt.Println("abandoned:", abandoned.Load())
  fmt.Println("completed+abandoned:", len(done)+int(abandoned.Load()))
}
```

Output:

```
ctx err: context deadline exceeded
completed: 24
abandoned: 16
completed+abandoned: 40
```

Workers keep ranging over `jobs` after cancellation but skip the work, so the producer
never blocks and `close(jobs)` still ends the loop cleanly. The two counts always sum to
40: every job is accounted for.

That is **graceful drain**: finish the job in hand, skip the rest, exit tidily. The
alternative is **abandon** — `return` instead of `continue` on `ctx.Done()`. Abandoning is
faster to shut down but leaves the producer blocked on a send nobody will receive, so if
you abandon, the producer must select on `ctx.Done()` too.

## A semaphore is the lighter alternative

When you have a fixed slice of items and no need for a long-lived pool, a buffered
channel used as a counting semaphore bounds concurrency in four lines. Prove the bound
with an atomic high-water mark.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
  "sync/atomic"
  "time"
)

func main() {
  items := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
  const limit = 3

  sem := make(chan struct{}, limit)
  var inFlight, maxInFlight atomic.Int64
  out := make([]int, len(items))

  var wg sync.WaitGroup
  for i, v := range items {
    wg.Add(1)
    sem <- struct{}{}
    go func() {
      defer wg.Done()
      defer func() { inFlight.Add(-1); <-sem }()
      cur := inFlight.Add(1)
      for {
        old := maxInFlight.Load()
        if cur <= old || maxInFlight.CompareAndSwap(old, cur) {
          break
        }
      }
      time.Sleep(10 * time.Millisecond)
      out[i] = v * v
    }()
  }
  wg.Wait()

  sort.Ints(out)
  fmt.Println("squares:", out)
  fmt.Println("max concurrent:", maxInFlight.Load(), "<= limit", limit)
}
```

Output:

```
squares: [1 4 9 16 25 36 49 64 81 100]
max concurrent: 3 <= limit 3
```

Acquire is a send, release is a receive, and the buffer size is the limit. Note `sem <-`
is *before* `go`, not inside it — otherwise you launch all N goroutines immediately and
they queue up on the acquire, which bounds the work but not the goroutines. 12.4 covers
this properly, including `golang.org/x/sync/semaphore` for weighted acquires.

## One goroutine per item is not a pool

The reason any of this exists: `go f(item)` in a loop has no bound, and goroutine stacks
are not free.

```go
package main

import (
  "fmt"
  "runtime"
  "sync"
)

func sysMiB() float64 {
  var m runtime.MemStats
  runtime.ReadMemStats(&m)
  return float64(m.Sys) / (1 << 20)
}

func main() {
  const n = 1_000_000
  fmt.Printf("start:         %7d goroutines, Sys %6.1f MiB\n", runtime.NumGoroutine(), sysMiB())

  release := make(chan struct{})
  var wg sync.WaitGroup
  for i := 0; i < n; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      <-release
    }()
  }
  fmt.Printf("1M launched:   %7d goroutines, Sys %6.1f MiB\n", runtime.NumGoroutine(), sysMiB())
  close(release)
  wg.Wait()
  runtime.GC()
  fmt.Printf("after drain:   %7d goroutines, Sys %6.1f MiB\n", runtime.NumGoroutine(), sysMiB())
}
```

Output (the MiB figures vary by a few MiB between runs):

```
start:               1 goroutines, Sys    7.5 MiB
1M launched:   1000001 goroutines, Sys 2639.0 MiB
after drain:         1 goroutines, Sys 2639.6 MiB
```

2.6 GiB of address space for a million goroutines that do nothing. Real work carries real
stacks, so the number is worse. And if each of those goroutines opened a socket, you
would hit `too many open files` long before the memory mattered. Note that `Sys` does not
drop after the drain — the runtime keeps the memory for reuse — so the peak is what you
have to budget for.

A pool caps this at `numWorkers` stacks regardless of job count, which is the entire
point.

## Quick Reference

| Piece | Form | Rule |
|---|---|---|
| jobs channel | `make(chan Job)` | producer owns it and closes it |
| worker signature | `func(jobs <-chan J, out chan<- R, wg *sync.WaitGroup)` | directional types prevent a stray `close` |
| worker loop | `for j := range jobs { ... }` | ends when `jobs` closes |
| done signal | `defer wg.Done()` first line of worker | exactly once per worker |
| results channel | `make(chan Result)` | closed by `go func(){ wg.Wait(); close(out) }()` |
| collect | `for r := range results` in `main` | must not be the goroutine feeding jobs |
| ordering | sort by a job ID, or write to `out[i]` | results arrive in completion order |
| error | field in the result struct | or `errgroup` for first-error-aborts |
| cancel | `select { case <-ctx.Done(): ... default: }` per job | drain (`continue`) or abandon (`return`) |

| Workload | Worker count |
|---|---|
| CPU-bound (hashing, parsing, compression) | `runtime.NumCPU()` |
| I/O-bound (HTTP, DB, disk) | tens to hundreds; bounded by the remote limit, not cores |
| mixed | measure; start at `NumCPU()*2` |

| Tool | Use when |
|---|---|
| jobs+results channels | long-lived pool, streaming results, unknown job count |
| `errgroup` + `SetLimit` | fixed set of items, first error should abort |
| buffered-channel semaphore | fixed set of items, no error aggregation needed |
| `go f(x)` unbounded | essentially never |

## Common Mistakes

**Calling `close(results)` on the same line as `wg.Wait()` in `main`.** `main` cannot
both feed jobs and drain results; close it from a goroutine that does nothing but
`wg.Wait(); close(out)`.

**Calling `wg.Add(1)` inside the worker goroutine.** `Wait` can return before the
goroutine has started; always `Add` in the launching loop.

**Closing `results` from a worker.** No worker knows it is the last; the second close
panics with `close of closed channel`.

**Printing results in arrival order and calling it the answer.** Completion order is
nondeterministic; sort by job ID or write into `out[i]`.

**Assuming more workers is always faster.** Past `NumCPU` on CPU-bound work it is
strictly worse; the win only continues for work that blocks.

## Exercises

1. Take the canonical pool and remove the `go` from the producer closure so `main` feeds
   jobs directly. Find the smallest `results` buffer size that makes it work for 8 jobs,
   and explain why one less deadlocks.
2. Add a `workerID` field to `result` and print how many jobs each of 4 workers handled
   for 100 jobs of random duration. Run it three times — the distribution is not even.
   Explain why, and make it even.
3. Write `func Map[T, R any](in []T, workers int, f func(T) (R, error)) ([]R, error)` as
   a pool that preserves input order and returns the first error. Then write it again
   with `errgroup` and compare line counts.
4. Build a pool where a job can itself enqueue more jobs (a crawler). Make it terminate:
   `close(jobs)` no longer works, so you need a pending-job counter. Prove with
   `runtime.NumGoroutine()` that nothing leaks.
5. Instrument the context-cancelled pool with `atomic.Int64` counters and confirm that
   switching `continue` to `return` on `ctx.Done()` makes the producer block forever. Then
   fix the producer with a `select`.

---

[← 11.8 errgroup](../11-sync-context/08-errgroup.md) · [Course Index](../README.md) · [12.2 Fan-Out and Fan-In →](02-fan-out-fan-in.md)
