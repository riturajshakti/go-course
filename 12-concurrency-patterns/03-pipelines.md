[← 12.2 Fan-Out and Fan-In](02-fan-out-fan-in.md) · [Course Index](../README.md) · [12.4 Rate Limiting and Semaphores →](04-rate-limiting.md)

# 12.3 — Pipelines and Cancellation

## What You'll Learn

- The stage convention: `func stage(ctx, <-chan In) <-chan Out`, each stage owning and closing its own output
- Cancellation has to reach every stage, and every send needs a `select` — or you leak
- Backpressure comes free from unbuffered channels; what a buffer actually changes
- Errors in a pipeline: a `Result[T]` travelling down the same channel

## Coming From Another Language

| Other | Go |
|---|---|
| Unix `grep x \| sort \| uniq` | the same model: stages, channels instead of pipes, backpressure included |
| Node `readable.pipe(transform).pipe(dest)` | each stage is a function returning a channel; you wire them by nesting calls |
| RxJS `source$.pipe(map(f), filter(p))` | `filter(p)(map(f)(source))`, but each operator is a goroutine you can see |
| Python generators `(f(x) for x in src)` | lazy like a generator, but genuinely concurrent — each stage runs in parallel |
| Rust `iter().map().filter()` | iterator chains are lazy and single-threaded; Go stages are goroutines |

A pipeline is the generator convention from 10.5 applied repeatedly: a stage takes a
receive-only channel, returns a receive-only channel, and runs its own goroutine in
between. Unlike an iterator chain, every stage executes *concurrently* — stage 2 works on
item 1 while stage 1 produces item 2 — which is where the throughput comes from and where
the leaks come from.

## A three-stage pipeline

`generate` produces, `square` transforms, `odd` filters. Each stage is the same five
lines in a different costume.

```go
package main

import "fmt"

func generate(nums ...int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for _, n := range nums {
      out <- n
    }
  }()
  return out
}

func square(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for n := range in {
      out <- n * n
    }
  }()
  return out
}

func odd(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for n := range in {
      if n%2 == 1 {
        out <- n
      }
    }
  }()
  return out
}

func main() {
  for v := range odd(square(generate(1, 2, 3, 4, 5, 6, 7, 8))) {
    fmt.Println(v)
  }
}
```

Output:

```
1
9
25
49
```

Deterministic: a single path through the pipeline preserves order, because every stage
processes one item at a time in receive order. That changes the moment a stage has more
than one worker — see "A bounded stage" below.

## The ownership rule

Every stage above follows one rule, and it is the whole discipline:

> **A stage creates its output channel, is the only sender on it, and closes it with
> `defer close(out)` when its input is exhausted.**

Three consequences:

1. Nobody else may close `out` — hence the `<-chan Out` return type, which makes `close`
   on it a compile error for the caller.
2. `defer close(out)` fires on *every* exit path, including an early `return` for
   cancellation. A stage that returns without closing hangs everything downstream.
3. Closure propagates: stage 1 closing ends stage 2's `range`, which closes stage 2's
   output, and the teardown rolls downhill to `main` on its own.

## Backpressure is the default

An unbuffered channel means a send blocks until a receiver takes the value, so a fast
producer runs exactly as fast as its slowest consumer. Here a producer that would happily
emit instantly is paced by a 40ms consumer.

```go
package main

import (
  "fmt"
  "time"
)

var start time.Time

func elapsed() time.Duration {
  return time.Since(start).Round(10 * time.Millisecond)
}

func produce(n, buf int) <-chan int {
  out := make(chan int, buf)
  go func() {
    defer close(out)
    for i := 1; i <= n; i++ {
      out <- i
      fmt.Printf("  produced %d, producer now at %v\n", i, elapsed())
    }
  }()
  return out
}

func run(buf int) {
  fmt.Printf("buffer %d:\n", buf)
  start = time.Now()
  for range produce(5, buf) {
    time.Sleep(40 * time.Millisecond)
  }
}

func main() {
  run(0)
  run(5)
}
```

Output:

```
buffer 0:
  produced 1, producer now at 0s
  produced 2, producer now at 40ms
  produced 3, producer now at 80ms
  produced 4, producer now at 120ms
  produced 5, producer now at 160ms
buffer 5:
  produced 1, producer now at 0s
  produced 2, producer now at 0s
  produced 3, producer now at 0s
  produced 4, producer now at 0s
  produced 5, producer now at 0s
```

Byte-identical on every run. Unbuffered, the producer is dragged along at the consumer's
pace — one value per 40ms, finishing at 160ms. Buffered, it dumps all five values and is
done at `0s`. Both pipelines still take ~200ms end to end, because the consumer is the
bottleneck either way.

**A buffer does not make a pipeline faster.** It decouples the stages so a bursty producer
does not stall on every item, and it hides the fact that you produce faster than you
consume — which is the one piece of information backpressure was giving you for free.
Buffer a stage when its input is bursty and the average rate is sustainable. If the
average rate is not sustainable, a buffer only changes where you run out of memory.

## Taking one value and walking away leaks the whole pipeline

This is the pipeline version of the abandoned merge from 12.2, and it is subtler because
nothing looks wrong.

```go
func generate(n int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for i := 1; i <= n; i++ {
      out <- i
    }
  }()
  return out
}

func square(in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for n := range in {
      out <- n * n
    }
  }()
  return out
}

func main() {
  fmt.Println("goroutines before:", runtime.NumGoroutine())

  vals := square(generate(1000))
  fmt.Println("first value:", <-vals)

  time.Sleep(100 * time.Millisecond)
  runtime.GC()
  fmt.Println("goroutines after taking 1 of 1000:", runtime.NumGoroutine())
}
```

Output:

```
goroutines before: 1
first value: 1
goroutines after taking 1 of 1000: 3
```

Both stages leak. `square` is blocked on `out <- n * n` with no reader, and `generate` is
blocked on `out <- i` because `square` is not receiving. Neither will ever reach its
`defer close(out)`. This is what "every stage must drain or upstream blocks" means: a
downstream stage that stops reading freezes everything above it.

## The fix: context in, select on every send

Every stage takes a `ctx` and wraps each send in a `select`. Nothing else changes.

```go
package main

import (
  "context"
  "fmt"
  "runtime"
  "time"
)

func generate(ctx context.Context, n int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for i := 1; i <= n; i++ {
      select {
      case out <- i:
      case <-ctx.Done():
        return
      }
    }
  }()
  return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
  out := make(chan int)
  go func() {
    defer close(out)
    for n := range in {
      select {
      case out <- n * n:
      case <-ctx.Done():
        return
      }
    }
  }()
  return out
}

func main() {
  fmt.Println("goroutines before:", runtime.NumGoroutine())

  ctx, cancel := context.WithCancel(context.Background())
  vals := square(ctx, generate(ctx, 1000))
  fmt.Println("first value:", <-vals)
  cancel()

  time.Sleep(100 * time.Millisecond)
  runtime.GC()
  fmt.Println("goroutines after cancel:", runtime.NumGoroutine())
}
```

Output:

```
goroutines before: 1
first value: 1
goroutines after cancel: 1
```

Two rules together make this work, and both are required:

- **Every send is a `select` with `ctx.Done()`.** A bare `out <- v` can block forever.
- **The consumer calls `cancel()` when it stops early.** `defer cancel()` is the habit;
  without it the stages are *able* to exit but never told to.

A receive usually does not need the `select`: a stage blocked in `range in` is released
when its upstream closes `in`, which cancellation causes. The `select` on receive matters
when the input may never close, which is what `orDone` handles.

## or-done

When you consume a channel you did not create and cannot be sure will close, wrap it.
`orDone` is from 12.2; it is worth restating as the pipeline primitive it is:

```go
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
```

With it, a consumer writes `for v := range orDone(ctx, in)` and gets a loop that ends on
either input closure *or* cancellation, instead of the nested `select` at every call site.

## A bounded stage

When one stage is much slower than the rest, give that stage a worker pool (12.1) while
the others stay single-goroutine. Order is no longer preserved inside that stage, so carry
a sequence number (12.2) and reassemble.

```go
package main

import (
  "context"
  "fmt"
  "sync"
  "sync/atomic"
  "time"
)

type item struct {
  seq int
  val string
}

func gen(ctx context.Context, n int) <-chan item {
  out := make(chan item)
  go func() {
    defer close(out)
    for i := 0; i < n; i++ {
      select {
      case out <- item{seq: i, val: fmt.Sprintf("req%d", i)}:
      case <-ctx.Done():
        return
      }
    }
  }()
  return out
}

func slowStage(ctx context.Context, in <-chan item, workers int, inFlight, maxSeen *atomic.Int64) <-chan item {
  out := make(chan item)
  var wg sync.WaitGroup
  wg.Add(workers)
  for w := 0; w < workers; w++ {
    go func() {
      defer wg.Done()
      for it := range in {
        cur := inFlight.Add(1)
        for {
          old := maxSeen.Load()
          if cur <= old || maxSeen.CompareAndSwap(old, cur) {
            break
          }
        }
        time.Sleep(20 * time.Millisecond)
        inFlight.Add(-1)

        it.val = it.val + ":done"
        select {
        case out <- it:
        case <-ctx.Done():
          return
        }
      }
    }()
  }
  go func() {
    wg.Wait()
    close(out)
  }()
  return out
}

func main() {
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel()

  const n, workers = 8, 4
  var inFlight, maxSeen atomic.Int64

  start := time.Now()
  slots := make([]string, n)
  for it := range slowStage(ctx, gen(ctx, n), workers, &inFlight, &maxSeen) {
    slots[it.seq] = it.val
  }
  elapsed := time.Since(start)

  fmt.Println("in order:", slots)
  fmt.Println("max concurrent in the slow stage:", maxSeen.Load(), "<= workers", workers)
  fmt.Printf("%d items x 20ms over %d workers: %v\n", n, workers, elapsed.Round(10*time.Millisecond))
}
```

Output:

```
in order: [req0:done req1:done req2:done req3:done req4:done req5:done req6:done req7:done]
max concurrent in the slow stage: 4 <= workers 4
8 items x 20ms over 4 workers: 40ms
```

The atomic high-water mark proves the bound: never more than 4 of the 8 items in flight.
Note the stage still owns its output channel — the `wg.Wait(); close(out)` goroutine is
the fan-in from 12.2, because this stage has N senders instead of one. That is the only
structural change a bounded stage makes.

## Errors in a pipeline

A stage returns a channel, not an `(T, error)` pair, so there is nowhere to return an
error *to*. Two options: a separate error channel, or a `Result[T]` carrying both down the
same channel. Use `Result[T]`.

```go
type Result[T any] struct {
  Val T
  Err error
}
```

Idiomatic: `Result[T]`. A separate error channel forces every consumer into a `select`
over two channels, decouples an error from the item that caused it, and gives you a second
channel whose closure you must also coordinate. With `Result[T]` an error travels in
sequence with its item, a stage that cannot handle an incoming error passes it through
untouched, and there is exactly one channel to close.

The pass-through is the part worth noticing: a failed item still flows to the end of the
pipeline, so the final consumer sees every input accounted for as either a value or an
error.

## A realistic pipeline: read, parse, validate, aggregate

Four stages over in-memory CSV. `readLines` strips the header, `parse` turns text into a
`sale`, `validate` rejects non-positive amounts, and `main` aggregates. Errors from any
stage ride the same channel to the end.

```go
package main

import (
  "bufio"
  "context"
  "fmt"
  "sort"
  "strconv"
  "strings"
)

const csv = `txn,region,amount
1,eu,120.50
2,us,80.00
3,eu,bogus
4,ap,45.25
5,us,-10.00
6,eu,200.00
7,ap,15.75
8,us,99.99
`

type Result[T any] struct {
  Val T
  Err error
}

type sale struct {
  txn    int
  region string
  amount float64
}

func readLines(ctx context.Context, text string) <-chan Result[string] {
  out := make(chan Result[string])
  go func() {
    defer close(out)
    sc := bufio.NewScanner(strings.NewReader(text))
    first := true
    for sc.Scan() {
      if first {
        first = false
        continue
      }
      select {
      case out <- Result[string]{Val: sc.Text()}:
      case <-ctx.Done():
        return
      }
    }
    if err := sc.Err(); err != nil {
      select {
      case out <- Result[string]{Err: err}:
      case <-ctx.Done():
      }
    }
  }()
  return out
}

func parse(ctx context.Context, in <-chan Result[string]) <-chan Result[sale] {
  out := make(chan Result[sale])
  go func() {
    defer close(out)
    for r := range in {
      var res Result[sale]
      switch {
      case r.Err != nil:
        res = Result[sale]{Err: r.Err}
      default:
        res = parseLine(r.Val)
      }
      select {
      case out <- res:
      case <-ctx.Done():
        return
      }
    }
  }()
  return out
}

func parseLine(line string) Result[sale] {
  f := strings.Split(line, ",")
  if len(f) != 3 {
    return Result[sale]{Err: fmt.Errorf("line %q: want 3 fields, got %d", line, len(f))}
  }
  txn, err := strconv.Atoi(f[0])
  if err != nil {
    return Result[sale]{Err: fmt.Errorf("line %q: txn: %w", line, err)}
  }
  amt, err := strconv.ParseFloat(f[2], 64)
  if err != nil {
    return Result[sale]{Err: fmt.Errorf("txn %d: amount: %w", txn, err)}
  }
  return Result[sale]{Val: sale{txn: txn, region: f[1], amount: amt}}
}

func validate(ctx context.Context, in <-chan Result[sale]) <-chan Result[sale] {
  out := make(chan Result[sale])
  go func() {
    defer close(out)
    for r := range in {
      if r.Err == nil && r.Val.amount <= 0 {
        r = Result[sale]{Err: fmt.Errorf("txn %d: amount %.2f not positive", r.Val.txn, r.Val.amount)}
      }
      select {
      case out <- r:
      case <-ctx.Done():
        return
      }
    }
  }()
  return out
}

func main() {
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel()

  totals := map[string]float64{}
  counts := map[string]int{}
  var errs []string

  for r := range validate(ctx, parse(ctx, readLines(ctx, csv))) {
    if r.Err != nil {
      errs = append(errs, r.Err.Error())
      continue
    }
    totals[r.Val.region] += r.Val.amount
    counts[r.Val.region]++
  }

  regions := make([]string, 0, len(totals))
  for k := range totals {
    regions = append(regions, k)
  }
  sort.Strings(regions)

  fmt.Println("aggregates:")
  for _, r := range regions {
    fmt.Printf("  %-3s %d sales, total %7.2f\n", r, counts[r], totals[r])
  }
  fmt.Println("rejected:")
  for _, e := range errs {
    fmt.Println("  " + e)
  }
  fmt.Printf("accepted %d, rejected %d\n", counts["eu"]+counts["us"]+counts["ap"], len(errs))
}
```

Output:

```
aggregates:
  ap  2 sales, total   61.00
  eu  2 sales, total  320.50
  us  2 sales, total  179.99
rejected:
  txn 3: amount: strconv.ParseFloat: parsing "bogus": invalid syntax
  txn 5: amount -10.00 not positive
accepted 6, rejected 2
```

Byte-identical on every run. The regions are sorted before printing because map iteration
order is random (7.6), not because the pipeline is nondeterministic — eight inputs produce
eight outputs in input order.

Note how `parse` handles an incoming error: it rewraps it and forwards it, never inspects
it. And `validate` only checks `r.Err == nil` before applying its own rule. Each stage
knows one job plus "pass errors through".

## Cancelling the realistic pipeline

Same stages, different consumer: stop after three accepted rows. The `break` is safe only
because `cancel()` precedes it and every stage selects on `ctx.Done()`.

```go
func main() {
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel()

  var accepted int
  for r := range validate(ctx, parse(ctx, readLines(ctx, csv))) {
    if r.Err != nil {
      continue
    }
    accepted++
    fmt.Printf("accepted txn %d (%s %.2f)\n", r.Val.txn, r.Val.region, r.Val.amount)
    if accepted == 3 {
      fmt.Println("budget reached, cancelling")
      cancel()
      break
    }
  }

  time.Sleep(100 * time.Millisecond)
  runtime.GC()
  fmt.Println("goroutines after cancel+break:", runtime.NumGoroutine())
  fmt.Println("ctx err:", ctx.Err())
}
```

Output:

```
accepted txn 1 (eu 120.50)
accepted txn 2 (us 80.00)
accepted txn 4 (ap 45.25)
budget reached, cancelling
goroutines after cancel+break: 1
ctx err: context canceled
```

Three accepted rows out of eight inputs, and the pipeline tears itself down to a single
goroutine. txn 3 is missing because it failed to parse and txn 3's error was skipped by
`continue`. The `defer cancel()` at the top is still the safety net: if the loop had ended
normally, `cancel()` would fire on return and release any stage still parked on a send.

## Draining on early exit

`cancel()` is the right tool when you abandon a pipeline. Draining — `for range in {}` to
consume the rest — is the alternative, and it is only correct when the remaining input is
bounded and cheap. Over `generate(1000)` it is a waste; over an infinite source it never
returns. Cancel the context, do not drain.

The exception is a buffered output channel you own: if you cancel while values sit in the
buffer they are simply dropped, which is fine for a pipeline and wrong if each value holds
a resource that needs closing. In that case drain and release.

## Quick Reference

| Piece | Form |
|---|---|
| stage signature | `func stage(ctx context.Context, in <-chan In) <-chan Out` |
| stage body | `out := make(chan Out)` · `go func(){ defer close(out); ... }()` · `return out` |
| ownership | the stage creates, is sole sender on, and closes its own output |
| every send | `select { case out <- v: case <-ctx.Done(): return }` |
| wiring | `c := stage3(ctx, stage2(ctx, stage1(ctx, src)))` |
| cancellable consume | `for v := range orDone(ctx, in)` |
| bounded stage | N workers on `in` + `go func(){ wg.Wait(); close(out) }()` |
| errors | `Result[T]{Val, Err}` down the same channel |
| early exit | `cancel()` then `break` — never `break` alone |

| Channel | Effect |
|---|---|
| unbuffered | full backpressure; producer paced by consumer; leak shows up immediately |
| small buffer | absorbs burstiness; backpressure still applies once full |
| large buffer | hides a rate mismatch until memory runs out |

## Common Mistakes

**A stage that `return`s on cancellation without closing its output.** Downstream blocks
forever on a receive. Always `defer close(out)` as the first statement in the goroutine.

**A bare `out <- v` in a stage.** Works until a consumer stops reading, then leaks the
whole upstream. Every send in a cancellable pipeline is a `select`.

**Breaking out of the consumer loop without calling `cancel()`.** The stages are prepared
to exit but nothing tells them to. `defer cancel()` plus an explicit `cancel()` before an
early `break`.

**Closing the input channel from inside a stage.** The stage does not own it; the second
`close` panics. Only ever close what you created.

**Adding buffers to make a slow pipeline faster.** Buffers decouple, they do not
accelerate. Find the slow stage and give it a worker pool instead.

## Exercises

1. Add a fifth stage to the realistic pipeline that converts every amount to a second
   currency, and give it a 3-worker pool. Keep the final aggregate byte-identical by
   carrying a sequence number through.
2. Remove the `select` from `validate`'s send, leaving the others intact. Re-run the
   cancellation example and use `runtime.NumGoroutine()` to work out how many goroutines
   leak and which ones.
3. Write `func batch[T any](ctx context.Context, in <-chan T, n int, timeout time.Duration) <-chan []T`
   that emits slices of up to `n` items, flushing early if `timeout` passes. You need a
   `time.Timer` in the `select`; make sure a partial final batch is still emitted.
4. Instrument each stage of the realistic pipeline with a counter of how long it spent
   blocked on a send versus doing work. Which stage is the bottleneck, and does a buffer
   on its output move the bottleneck or just hide it?
5. Build a pipeline whose source is infinite (`for i := 0; ; i++`) and a consumer that
   stops at 10 items. Prove with `runtime.NumGoroutine()` that it leaks nothing, then
   prove that removing `defer cancel()` makes it leak.

---

[← 12.2 Fan-Out and Fan-In](02-fan-out-fan-in.md) · [Course Index](../README.md) · [12.4 Rate Limiting and Semaphores →](04-rate-limiting.md)
