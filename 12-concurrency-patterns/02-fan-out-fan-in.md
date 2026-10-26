[← 12.1 Worker Pools](01-worker-pools.md) · [Course Index](../README.md) · [12.3 Pipelines and Cancellation →](03-pipelines.md)

# 12.2 — Fan-Out and Fan-In

## What You'll Learn

- Fan-out: N consumers on ONE channel means each item goes to exactly one of them, not to all
- Fan-in: the canonical generic `merge`, with a `WaitGroup` and exactly one `close`
- Fan-in destroys order; how to get it back with indices or sequence numbers
- Abandoning a merge early leaks every goroutine behind it, and the context fix
- `tee` when you genuinely do want every consumer to see every value

## Coming From Another Language

| Other | Go |
|---|---|
| RxJS `merge(a$, b$)` | `merge(a, b)` over channels — you write it once, ~15 lines |
| Node `EventEmitter` — every listener gets every event | a channel gives each value to ONE receiver; use `tee` to duplicate |
| Python `asyncio.as_completed(tasks)` | fan-in: results in completion order, original order lost |
| Rust `futures::stream::select_all` | `merge`, but you own the close and the cancellation |
| Go broadcast via `close(ch)` (10.4) | the only real one-to-many: a closed channel wakes every receiver |

Fan-out and fan-in are two halves of one idea: fan-out splits a stream across workers,
fan-in collapses many streams back into one. The parts are all from modules 10 and 11 —
channels, `range` until close, `select`, `WaitGroup`, `context`. What is new is the
composition, and the two traps it hides: a channel is a *queue*, not a broadcast, and a
merge you walk away from does not clean itself up.

## Fan-out: each item goes to exactly ONE consumer

Three consumers `range` over the *same* channel. Anyone arriving from an event-emitter
model expects all three to see all nine items. They do not — a channel hands each value
to exactly one receiver.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
)

type claim struct {
  consumer int
  item     int
}

func main() {
  items := make(chan int)

  var mu sync.Mutex
  var claims []claim

  var wg sync.WaitGroup
  for c := 1; c <= 3; c++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      for item := range items {
        mu.Lock()
        claims = append(claims, claim{consumer: c, item: item})
        mu.Unlock()
      }
    }()
  }

  for i := 1; i <= 9; i++ {
    items <- i
  }
  close(items)
  wg.Wait()

  sort.Slice(claims, func(i, j int) bool { return claims[i].item < claims[j].item })
  for _, c := range claims {
    fmt.Printf("item %d -> consumer %d\n", c.item, c.consumer)
  }
  fmt.Println("total claims:", len(claims), "for 9 items")

  perItem := map[int]int{}
  for _, c := range claims {
    perItem[c.item]++
  }
  dupes := 0
  for _, n := range perItem {
    if n != 1 {
      dupes++
    }
  }
  fmt.Println("items delivered more than once:", dupes)
}
```

Output (one run — the `consumer` column varies, see below):

```
item 1 -> consumer 3
item 2 -> consumer 3
item 3 -> consumer 3
item 4 -> consumer 3
item 5 -> consumer 3
item 6 -> consumer 3
item 7 -> consumer 3
item 8 -> consumer 2
item 9 -> consumer 2
total claims: 9 for 9 items
items delivered more than once: 0
```

Which consumer wins each item varies run to run; the last two lines are the stable part.
Here is the `consumer` column from three real runs, flattened to one line each:

```
3 1 2 3 3 3 3 3 3
1 3 1 1 2 3 1 1 1
3 3 3 3 3 3 3 3 3
```

Run three gave every single item to one consumer. That is legal and common: on an
unbuffered channel, handing off to a receiver that is already parked is cheapest, so a
fast producer with fast consumers can starve the others completely. Fan-out gives you a
*bound* on concurrency, never a fair distribution.

## The one real broadcast is close

10.4 showed `close(ch)` waking every receiver. That is the exception that proves the
rule — a closed channel yields forever, so all three consumers see it; a value does not.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
)

func main() {
  start := make(chan struct{})
  work := make(chan int)

  var mu sync.Mutex
  var sawStart []int
  var gotWork []int

  var wg sync.WaitGroup
  for c := 1; c <= 3; c++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      <-start
      mu.Lock()
      sawStart = append(sawStart, c)
      mu.Unlock()
      for n := range work {
        mu.Lock()
        gotWork = append(gotWork, c)
        mu.Unlock()
        _ = n
      }
    }()
  }

  close(start)
  work <- 100
  close(work)
  wg.Wait()

  sort.Ints(sawStart)
  fmt.Println("close(start) woke consumers:", sawStart, "- count", len(sawStart))
  fmt.Println("one send on work was received by", len(gotWork), "consumer")
}
```

Output:

```
close(start) woke consumers: [1 2 3] - count 3
one send on work was received by 1 consumer
```

`close` broadcasts; a send does not. This is exactly why a cancellation channel is
*closed* rather than written to: one `close` releases N goroutines.

## Fan-in: the canonical merge

N inputs, one output. Each input gets a forwarding goroutine, a `WaitGroup` counts them,
and one separate goroutine performs the single `close` after `Wait`. Generic over `T`
with plain channels, so it works for any element type.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
  "time"
)

func merge[T any](chans ...<-chan T) <-chan T {
  out := make(chan T)
  var wg sync.WaitGroup
  wg.Add(len(chans))
  for _, c := range chans {
    go func(c <-chan T) {
      defer wg.Done()
      for v := range c {
        out <- v
      }
    }(c)
  }
  go func() {
    wg.Wait()
    close(out)
  }()
  return out
}

func source(label string, n int, delay time.Duration) <-chan string {
  out := make(chan string)
  go func() {
    defer close(out)
    for i := 1; i <= n; i++ {
      time.Sleep(delay)
      out <- fmt.Sprintf("%s%d", label, i)
    }
  }()
  return out
}

func main() {
  a := source("a", 3, 10*time.Millisecond)
  b := source("b", 3, 15*time.Millisecond)
  c := source("c", 3, 7*time.Millisecond)

  var got []string
  for v := range merge(a, b, c) {
    got = append(got, v)
  }

  fmt.Println("arrival count:", len(got))
  sort.Strings(got)
  fmt.Println("sorted:", got)
}
```

Output:

```
arrival count: 9
sorted: [a1 a2 a3 b1 b2 b3 c1 c2 c3]
```

The shape matters more than the code:

1. `wg.Add(len(chans))` happens **before** any goroutine starts — never inside them.
2. Each forwarder `range`s its own input, so it exits when that input closes.
3. **Exactly one goroutine closes `out`, after `wg.Wait()`.** A forwarder cannot: it does
   not know whether it is the last one.
4. `out` is returned as `<-chan T`, so the caller cannot close it either.

That is the ownership rule from 12.1 — whoever creates a channel closes it — applied to a
channel with N writers. `merge` and `source` are reused for the rest of this lesson; later
blocks show only what changes.

## Fan-in loses order

Print values as they arrive instead of collecting and sorting, and the interleaving shows.

```go
func main() {
  a := source("a", 3, 10*time.Millisecond)
  b := source("b", 3, 15*time.Millisecond)
  c := source("c", 3, 7*time.Millisecond)

  for v := range merge(a, b, c) {
    fmt.Print(v, " ")
  }
  fmt.Println()
}
```

Interleaving varies between runs. Three real runs:

```
c1 a1 b1 c2 a2 c3 a3 b2 b3 
c1 a1 c2 b1 a2 c3 a3 b2 b3 
c1 a1 b1 c2 a2 c3 a3 b2 b3 
```

Sources `a`, `b` and `c` tick at 10ms, 15ms and 7ms, so the broad shape is stable but
`b1` and `c2` trade places. There is no ordering guarantee across merged channels at all:
if output order matters, the order has to travel in the data.

## Restoring order with a sequence number

Wrap each value with the slot it belongs in, then write into that slot. No sorting and no
lock — distinct indices of a slice are distinct memory, so `-race` is clean.

```go
type seq[T any] struct {
  n int
  v T
}

func source(label string, base, n int, delay time.Duration) <-chan seq[string] {
  out := make(chan seq[string])
  go func() {
    defer close(out)
    for i := 0; i < n; i++ {
      time.Sleep(delay)
      out <- seq[string]{n: base + i, v: fmt.Sprintf("%s%d", label, i+1)}
    }
  }()
  return out
}

func main() {
  a := source("a", 0, 3, 10*time.Millisecond)
  b := source("b", 3, 3, 15*time.Millisecond)
  c := source("c", 6, 3, 7*time.Millisecond)

  slots := make([]string, 9)
  filled := 0
  for s := range merge(a, b, c) {
    slots[s.n] = s.v
    filled++
  }
  fmt.Println("filled:", filled)
  fmt.Println("in order:", slots)
}
```

Output:

```
filled: 9
in order: [a1 a2 a3 b1 b2 b3 c1 c2 c3]
```

Byte-identical on every run. This is the practical payoff of fan-in: parallelism *and* a
deterministic result, because order lives in the values rather than in the scheduler.

## Fan-out + fan-in = a parallel map

Put the two halves together and you get the function you actually wanted: run `f` over a
slice with bounded concurrency, results in input order.

```go
package main

import (
  "fmt"
  "strings"
  "sync"
  "sync/atomic"
  "time"
)

type indexed[T any] struct {
  i int
  v T
}

func ParallelMap[T, R any](in []T, workers int, f func(T) R) []R {
  jobs := make(chan indexed[T])
  results := make(chan indexed[R])

  var wg sync.WaitGroup
  for w := 0; w < workers; w++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      for j := range jobs {
        results <- indexed[R]{i: j.i, v: f(j.v)}
      }
    }()
  }

  go func() {
    for i, v := range in {
      jobs <- indexed[T]{i: i, v: v}
    }
    close(jobs)
  }()

  go func() {
    wg.Wait()
    close(results)
  }()

  out := make([]R, len(in))
  for r := range results {
    out[r.i] = r.v
  }
  return out
}

func main() {
  words := []string{"fan", "out", "fan", "in", "merge", "tee", "or-done", "pipeline"}

  var calls atomic.Int64
  start := time.Now()
  lengths := ParallelMap(words, 4, func(w string) string {
    calls.Add(1)
    time.Sleep(20 * time.Millisecond)
    return fmt.Sprintf("%s=%d", strings.ToUpper(w), len(w))
  })
  elapsed := time.Since(start)

  fmt.Println("results in input order:")
  for i, l := range lengths {
    fmt.Printf("  %d %s\n", i, l)
  }
  fmt.Println("f called:", calls.Load(), "times for", len(words), "items")
  fmt.Println("8 items x 20ms over 4 workers took about 40ms:", elapsed.Round(10*time.Millisecond))
}
```

Output:

```
results in input order:
  0 FAN=3
  1 OUT=3
  2 FAN=3
  3 IN=2
  4 MERGE=5
  5 TEE=3
  6 OR-DONE=7
  7 PIPELINE=8
f called: 8 times for 8 items
8 items x 20ms over 4 workers took about 40ms: 40ms
```

Fan-out is the `for w := 0; w < workers; w++` loop over one `jobs` channel; fan-in is
`results` plus the `wg.Wait(); close(results)` goroutine. The index rides along inside
`indexed[T]`, so `out[r.i] = r.v` reassembles input order. Eight items of 20ms over four
workers is two rounds, and 40ms is what it measures.

Idiomatic: for a fixed slice with no streaming requirement, `errgroup` with `SetLimit` and
a pre-sized output slice (12.1) is shorter. Build this when you want the stages as
reusable channel-to-channel pieces — which is what 12.3 does with them.

## Abandoning a merge leaks every goroutine behind it

`break` out of a `range` over a merged channel and nothing tells the producers. They block
forever on a send nobody will ever receive.

```go
func counter(n int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for i := 0; i < n; i++ {
      out <- i
    }
  }()
  return out
}

func main() {
  fmt.Println("before:", runtime.NumGoroutine())

  merged := merge(counter(100), counter(100), counter(100))
  for v := range merged {
    if v == 2 {
      break
    }
  }

  time.Sleep(100 * time.Millisecond)
  runtime.GC()
  fmt.Println("after abandoning the merge:", runtime.NumGoroutine())
}
```

Output:

```
before: 1
after abandoning the merge: 8
```

Seven leaked goroutines from one `break`: three `counter` producers blocked on `out <- i`,
three merge forwarders blocked on `out <- v`, and the closer stuck in `wg.Wait()` forever.
`runtime.GC()` does not help — a goroutine blocked on a channel is not garbage, it is
live. This is the most common leak in channel code; 12.5 covers finding it in a running
service.

## The fix: context through every send

Every send becomes a `select` with `ctx.Done()`, and `orDone` wraps a receive so a stage
can stop mid-stream. Cancel once and everything unwinds.

```go
package main

import (
  "context"
  "fmt"
  "runtime"
  "sync"
  "time"
)

func orDone[T any](ctx context.Context, in <-chan T) <-chan T {
  out := make(chan T)
  go func() {
    defer close(out)
    for {
      select {
      case <-ctx.Done():
        return
      case v, ok := <-in:
        if !ok {
          return
        }
        select {
        case out <- v:
        case <-ctx.Done():
          return
        }
      }
    }
  }()
  return out
}

func merge[T any](ctx context.Context, chans ...<-chan T) <-chan T {
  out := make(chan T)
  var wg sync.WaitGroup
  wg.Add(len(chans))
  for _, c := range chans {
    go func(c <-chan T) {
      defer wg.Done()
      for v := range orDone(ctx, c) {
        select {
        case out <- v:
        case <-ctx.Done():
          return
        }
      }
    }(c)
  }
  go func() {
    wg.Wait()
    close(out)
  }()
  return out
}

func counter(ctx context.Context, n int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for i := 0; i < n; i++ {
      select {
      case out <- i:
      case <-ctx.Done():
        return
      }
    }
  }()
  return out
}

func main() {
  fmt.Println("before:", runtime.NumGoroutine())

  ctx, cancel := context.WithCancel(context.Background())
  merged := merge(ctx, counter(ctx, 100), counter(ctx, 100), counter(ctx, 100))
  for v := range merged {
    if v == 2 {
      break
    }
  }
  cancel()

  time.Sleep(100 * time.Millisecond)
  runtime.GC()
  fmt.Println("after cancel:", runtime.NumGoroutine())
}
```

Output:

```
before: 1
after cancel: 1
```

Back to one goroutine. `orDone` is the reusable piece: it turns "`range` this channel"
into "`range` this channel, but give up if cancelled", with a nested `select` on the send
so it cannot block forever delivering a value nobody wants. Note that `counter` needs the
`select` too — wrapping only the consumer side still leaves the producer stuck.

Rule: any goroutine sending on a channel whose lifetime it does not control needs a
`select` on a cancellation signal.

## tee: when you DO want every consumer to see everything

Duplication is explicit in Go. `tee` fans one input out to N outputs by sending each value
to all of them.

```go
package main

import (
  "fmt"
  "sync"
)

func tee[T any](in <-chan T, n int) []<-chan T {
  outs := make([]chan T, n)
  for i := range outs {
    outs[i] = make(chan T)
  }
  go func() {
    defer func() {
      for _, o := range outs {
        close(o)
      }
    }()
    for v := range in {
      for _, o := range outs {
        o <- v
      }
    }
  }()

  ro := make([]<-chan T, n)
  for i, o := range outs {
    ro[i] = o
  }
  return ro
}

func main() {
  src := make(chan int)
  go func() {
    defer close(src)
    for i := 1; i <= 4; i++ {
      src <- i
    }
  }()

  outs := tee(src, 3)

  var mu sync.Mutex
  seen := map[int][]int{}

  var wg sync.WaitGroup
  for i, o := range outs {
    wg.Add(1)
    go func(i int, o <-chan int) {
      defer wg.Done()
      for v := range o {
        mu.Lock()
        seen[i] = append(seen[i], v)
        mu.Unlock()
      }
    }(i, o)
  }
  wg.Wait()

  for i := 0; i < 3; i++ {
    fmt.Printf("consumer %d saw %v\n", i, seen[i])
  }
}
```

Output:

```
consumer 0 saw [1 2 3 4]
consumer 1 saw [1 2 3 4]
consumer 2 saw [1 2 3 4]
```

Deterministic: every consumer sees every value, in order. The cost is coupling — the loop
sends to each output in turn, so **the slowest consumer throttles all of them**, and one
consumer that stops reading blocks the entire tee. Buffer the outputs to absorb jitter, or
give each output its own `select` on a `done` channel if a consumer may leave early.

## Errors in a fan-in

A merged channel has one element type, so errors travel inside it. A `Result[T]` carrying
both a `Value` and an `Err` preserves partial results: you learn that `ap` is down *and*
still get the rows from `eu` and `us`.

```go
type Result[T any] struct {
  Shard string
  Value T
  Err   error
}

var errShardDown = errors.New("shard unreachable")

func shard(name string, rows int, fail bool) <-chan Result[int] {
  out := make(chan Result[int])
  go func() {
    defer close(out)
    if fail {
      out <- Result[int]{Shard: name, Err: fmt.Errorf("query %s: %w", name, errShardDown)}
      return
    }
    for i := 1; i <= rows; i++ {
      out <- Result[int]{Shard: name, Value: i * 10}
    }
  }()
  return out
}

func main() {
  all := merge(
    shard("eu", 2, false),
    shard("us", 3, false),
    shard("ap", 0, true),
  )

  var rows []string
  var errs []error
  for r := range all {
    if r.Err != nil {
      errs = append(errs, r.Err)
      continue
    }
    rows = append(rows, fmt.Sprintf("%s:%d", r.Shard, r.Value))
  }

  sort.Strings(rows)
  fmt.Println("rows:", rows)
  joined := errors.Join(errs...)
  fmt.Println("errors:", joined)
  fmt.Println("errors.Is(errShardDown):", errors.Is(joined, errShardDown))
  fmt.Printf("got %d rows and %d errors\n", len(rows), len(errs))
}
```

Output:

```
rows: [eu:10 eu:20 us:10 us:20 us:30]
errors: query ap: shard unreachable
errors.Is(errShardDown): true
got 5 rows and 1 errors
```

`errors.Join` (8.5) collects them and `errors.Is` still sees through the wrapping. Use
this when every source matters. Use `errgroup` (11.8) when the first failure should cancel
the rest: a scatter-gather across replicas wants `Result[T]`, a fan-out over required
sub-requests wants `errgroup`.

## Quick Reference

| Helper | Signature | Notes |
|---|---|---|
| fan-out | N goroutines `range`-ing one `<-chan T` | each item to exactly ONE consumer |
| broadcast | `close(done)` | the only real one-to-many |
| `merge` | `merge[T](...<-chan T) <-chan T` | `wg.Add(len)` up front, one `close` after `Wait` |
| `orDone` | `orDone[T](ctx, <-chan T) <-chan T` | makes a `range` cancellable |
| `tee` | `tee[T](<-chan T, n) []<-chan T` | explicit duplication; slowest consumer sets the pace |
| ordered fan-in | `indexed[T]{i, v}` + `out[i] = v` | deterministic, no lock, no sort |
| errors | `Result[T]{Value, Err}` | partial results survive |

| Need | Reach for |
|---|---|
| bound concurrency over a slice, order preserved | fan-out + fan-in parallel map, or `errgroup.SetLimit` |
| combine N long-lived streams | `merge` |
| every consumer sees every value | `tee` |
| stop early without leaking | `context` + `orDone`, `select` on every send |
| first error aborts the rest | `errgroup` (11.8) |
| all errors, partial results kept | `Result[T]` + `errors.Join` |

## Common Mistakes

**Expecting N consumers on one channel to each see every value.** They share the stream;
one value goes to one consumer. Use `tee`, or `close` for a pure signal.

**Closing the merged channel from inside a forwarding goroutine.** No forwarder knows it
is the last, so the second `close` panics with `close of closed channel`. Close once,
after `wg.Wait()`.

**Calling `wg.Add(1)` inside the forwarder goroutine.** `Wait` can return before any
forwarder starts, closing `out` while values are still coming. `Add` before `go`.

**Breaking out of a `range` over a merged channel.** Every producer behind it blocks
forever on its next send. Cancel the context first, or do not break.

**Reading a fan-in's arrival order as the answer.** It is scheduler-dependent. Carry an
index in the value, or sort before printing.

## Exercises

1. Change `merge` to use a buffered `out` of size 100 and re-run the leak example.
   `runtime.NumGoroutine()` drops but does not reach 1 — work out exactly which goroutines
   survive, and why buffering only postpones the problem.
2. Write `func first[T any](ctx context.Context, chans ...<-chan T) (T, bool)` returning
   the first value from whichever channel produces one, then cancelling the rest. Prove
   with `runtime.NumGoroutine()` that nothing is left behind.
3. Rewrite `tee` so a slow consumer cannot throttle the others: give each output its own
   buffered channel and drop values for any consumer whose buffer is full. Report the drop
   count per consumer.
4. Build an ordered merge: given K channels each already sorted ascending, produce one
   globally sorted channel. You need one pending value per input, not a sequence number.
5. Take the parallel map and make `f` return `(R, error)`. Implement both policies — keep
   all errors, and abort on the first — and compare how much code each needs.

---

[← 12.1 Worker Pools](01-worker-pools.md) · [Course Index](../README.md) · [12.3 Pipelines and Cancellation →](03-pipelines.md)
