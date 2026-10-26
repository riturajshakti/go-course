[← 14.4 testify, Briefly](04-testify.md) · [Course Index](../README.md) · [14.6 Fuzzing →](06-fuzzing.md)

# 14.5 — Benchmarks

## What You'll Learn

- `for b.Loop()` is the modern form and it fixes two real bugs that `for range b.N` does not
- Reading `iterations ns/op B/op allocs/op` field by field, and `-benchmem`, `-benchtime`, `-cpu`
- `b.ResetTimer` demonstrated by a benchmark that is 290x wrong without it
- Dead code elimination measuring nothing, sub-benchmarks showing complexity, and `benchstat` for real comparison

## Coming From Another Language

| Other | Go |
|---|---|
| `pytest-benchmark`, `timeit` | built in: `func BenchmarkXxx(b *testing.B)` |
| `console.time()` | the framework picks the iteration count for you |
| `criterion` (Rust) | same statistical idea, but you add `benchstat` for the stats |
| jsperf-style manual loops | never write your own timing loop; `b.Loop()` owns the timer |
| JIT warmup concerns | no JIT, but caches and the scheduler still need `-count` to average out |

All measurements below are from darwin/arm64 on an Apple M4 Pro. **Benchmark numbers are
machine-specific and vary run to run** — the five-run spread at the end of this lesson is the
honest picture. Compare ratios on your own hardware, never absolute numbers across machines.

## The shape

A benchmark lives in a `_test.go` file, takes `*testing.B`, and runs its body many times. It does
not run under plain `go test` — you need `-bench`.

```go
package join

import "strings"

func Concat(parts []string) string {
  var s string
  for _, p := range parts {
    s += p
  }
  return s
}

func Builder(parts []string) string {
  var b strings.Builder
  for _, p := range parts {
    b.WriteString(p)
  }
  return b.String()
}

func BuilderGrown(parts []string) string {
  n := 0
  for _, p := range parts {
    n += len(p)
  }
  var b strings.Builder
  b.Grow(n)
  for _, p := range parts {
    b.WriteString(p)
  }
  return b.String()
}
```

```go
package join

import (
  "strconv"
  "testing"
)

func parts(n int) []string {
  p := make([]string, n)
  for i := range p {
    p[i] = strconv.Itoa(i)
  }
  return p
}

func BenchmarkConcatLoop(b *testing.B) {
  p := parts(100)
  for b.Loop() {
    Concat(p)
  }
}

func BenchmarkConcatBN(b *testing.B) {
  p := parts(100)
  for range b.N {
    Concat(p)
  }
}

func BenchmarkBuilder(b *testing.B) {
  p := parts(100)
  for b.Loop() {
    Builder(p)
  }
}

func BenchmarkBuilderGrown(b *testing.B) {
  p := parts(100)
  for b.Loop() {
    BuilderGrown(p)
  }
}
```

`go test -count=1 -run '^$' -bench '.' -benchmem .`. The `-run '^$'` matches no tests, so only
benchmarks run. Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkConcatLoop-12      	  638506	      1854 ns/op	    9744 B/op	      99 allocs/op
BenchmarkConcatBN-12        	  646676	      1863 ns/op	    9744 B/op	      99 allocs/op
BenchmarkBuilder-12         	 4207819	       281.0 ns/op	     504 B/op	       6 allocs/op
BenchmarkBuilderGrown-12    	 3716773	       327.6 ns/op	     192 B/op	       1 allocs/op
PASS
ok  	example.com/scratch	5.309s
```

## b.Loop() vs b.N

Both forms appear above and report the same number for `Concat` (1854 vs 1863 ns/op, inside the
noise). `b.Loop()` is newer and is now the idiomatic choice:

```go
for b.Loop() {
  Work()
}
```

versus the classic:

```go
for range b.N {
  Work()
}
```

What `b.Loop()` does that `b.N` does not:

- **It keeps loop values alive.** The compiler is told not to optimize away arguments, results and
  variables inside the loop braces. This is the fix for dead code elimination, demonstrated below,
  and it is the big one.
- **It handles the timer itself.** The first call resets the timer, so setup before the loop is
  excluded without `b.ResetTimer()`. When it returns false it stops the timer, so cleanup after the
  loop is excluded too.
- **It runs the benchmark function once per measurement.** A `b.N` benchmark is *re-invoked* with
  growing `N` (1, 100, 10000, ...) until it takes long enough, so its setup runs several times.
  `b.Loop()` calls your function once and iterates internally.

That third point has a practical consequence: with `b.N`, expensive setup before the loop is paid
on every retry. With `b.Loop()` it is paid once.

Constraints: the condition must be written **exactly** `b.Loop()`, and you must not combine it with
a `b.N` loop in the same benchmark. After the loop, `b.N` holds the total iteration count, so you
can still compute your own averages.

**Idiomatic:** `for b.Loop()`. Use `for range b.N` only when you must support a toolchain older
than Go 1.24.

## Reading the output

```
BenchmarkBuilder-12         	 4207819	       281.0 ns/op	     504 B/op	       6 allocs/op
```

| Field | Value | Meaning |
|---|---|---|
| `BenchmarkBuilder` | — | the function name |
| `-12` | 12 | `GOMAXPROCS` the benchmark ran with |
| `4207819` | iterations | how many times the body ran; chosen to fill `-benchtime` |
| `281.0 ns/op` | nanoseconds | wall time per iteration — **the number you compare** |
| `504 B/op` | bytes | heap bytes allocated per iteration (`-benchmem`) |
| `6 allocs/op` | count | number of heap allocations per iteration (`-benchmem`) |

And the header:

| Line | Meaning |
|---|---|
| `goos: darwin` / `goarch: arm64` | the target platform |
| `pkg: example.com/scratch` | the package benchmarked |
| `cpu: Apple M4 Pro` | the detected CPU — why cross-machine comparison is meaningless |

The iteration count is **not** a result. It is an artifact of how long each iteration took; a higher
count simply means a faster body. Comparing iteration counts between benchmarks tells you nothing
that `ns/op` does not.

Read the real numbers above as the lesson they are: `strings.Builder` is **6.6x faster** than `+=`
(281 vs 1854 ns/op) and allocates 6 times instead of 99. Adding `Grow` cuts allocations to exactly
1 and bytes from 504 to 192 — but `ns/op` went *up* slightly (327.6 vs 281.0), because computing
the total length means a second pass over the slice. That is a genuine trade-off you would never
see without `-benchmem`, and a good reminder that fewer allocations does not automatically mean
faster.

`allocs/op` is often more actionable than `ns/op`: it is far more stable between runs, and in a
service under GC pressure it predicts behaviour better than a micro-timing.

## -benchtime and -count

| Flag | Effect |
|---|---|
| `-bench '.'` | run all benchmarks (`-bench` is required; default runs none) |
| `-bench 'Builder$'` | only matching benchmarks, regexp, split on `/` for sub-benchmarks |
| `-benchmem` | add `B/op` and `allocs/op` |
| `-benchtime 3s` | target duration per benchmark (default `1s`) |
| `-benchtime 100x` | exactly 100 iterations instead of a duration |
| `-count=5` | repeat each benchmark 5 times, for `benchstat` |
| `-cpu 1,4,12` | run once per `GOMAXPROCS` value |
| `-run '^$'` | skip the tests (benchmarks run after tests by default) |
| `-list 'Benchmark.*'` | list benchmark names without running them |

`-benchtime 3s` on the same benchmark:

```
BenchmarkBuilder-12    	10823756	       348.7 ns/op
```

`-benchtime 100x`:

```
BenchmarkBuilder-12    	     100	       719.6 ns/op
```

**719.6 ns/op from 100 iterations versus ~300 ns/op from four million.** The small count is
dominated by cold caches and branch predictors; it is not a better measurement, it is a worse one.
Use `Nx` for reproducing a specific scenario or keeping a slow benchmark bounded, never to go
faster.

## Noise, and comparing properly

Five runs of one unchanged benchmark, `go test -count=5 -run '^$' -bench 'BenchmarkBuilder$' .`:

```
BenchmarkBuilder-12    	 3820080	       304.3 ns/op
BenchmarkBuilder-12    	 4123486	       309.9 ns/op
BenchmarkBuilder-12    	 3596108	       309.5 ns/op
BenchmarkBuilder-12    	 4049274	       302.6 ns/op
BenchmarkBuilder-12    	 4089546	       298.6 ns/op
```

298.6 to 309.9 ns/op — about 4% spread with nothing changed. So a single-run "improvement" under
5% is indistinguishable from noise, and eyeballing two numbers is not a comparison.

`benchstat` does the statistics. Install it:

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

Run each variant with `-count=6` into its own file, with the **same benchmark name** in both (rename
the function, or use a sub-benchmark name, so benchstat can pair them), then compare. Here
`BenchmarkConcatLoop` and `BenchmarkBuilder` were both relabelled `BenchmarkJoin` into
`old.txt` and `new.txt`:

```bash
go test -count=6 -run '^$' -bench 'Concat|Builder' -benchmem . > all.txt
benchstat old.txt new.txt
```

Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
        │ /tmp/m14-old.txt │          /tmp/m14-new.txt          │
        │      sec/op      │   sec/op     vs base               │
Join-12       2009.0n ± 8%   301.6n ± 7%  -84.99% (p=0.002 n=6)

        │ /tmp/m14-old.txt │         /tmp/m14-new.txt          │
        │       B/op       │    B/op     vs base               │
Join-12        9744.0 ± 0%   504.0 ± 0%  -94.83% (p=0.002 n=6)

        │ /tmp/m14-old.txt │         /tmp/m14-new.txt          │
        │    allocs/op     │ allocs/op   vs base               │
Join-12        99.000 ± 0%   6.000 ± 0%  -93.94% (p=0.002 n=6)
```

That is the output to put in a pull request: `-84.99%` with `p=0.002` over `n=6` samples, and a
`± 8%` confidence interval that tells you the measurement was stable enough to trust. If benchstat
prints `~` instead of a percentage, the difference was not statistically significant — which is
exactly the conclusion you want before you merge a "optimization" that adds complexity.

## b.ResetTimer, StopTimer, StartTimer

A benchmark with expensive setup inside the measured region reports the setup, not the work. Three
versions of the same lookup, with a two-million-element slice to build:

```go
package s

import "testing"

func BenchmarkLookupNoReset(b *testing.B) {
  data := BuildSorted(2_000_000)
  for range b.N {
    Lookup(data, 1999998)
  }
}

func BenchmarkLookupReset(b *testing.B) {
  data := BuildSorted(2_000_000)
  b.ResetTimer()
  for range b.N {
    Lookup(data, 1999998)
  }
}

func BenchmarkLookupLoop(b *testing.B) {
  data := BuildSorted(2_000_000)
  for b.Loop() {
    Lookup(data, 1999998)
  }
}
```

`go test -count=1 -run '^$' -bench '.' -benchtime 200x .` — fixed at 200 iterations so the setup
cost cannot be amortized away. Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkLookupNoReset-12    	     200	      3746 ns/op
BenchmarkLookupReset-12      	     200	        12.91 ns/op
BenchmarkLookupLoop-12       	     200	        11.04 ns/op
```

**3746 ns/op versus 12.91 ns/op — the unreset benchmark is 290x wrong.** It is reporting a slice of
the setup cost spread over the iterations, and it looks like a plausible number, which is what makes
this bug dangerous. `BenchmarkLookupLoop` needs no `ResetTimer` at all: `b.Loop()` reset the timer
on its first call. That is the single best argument for the new form.

| Call | Use |
|---|---|
| `b.ResetTimer()` | zero the clock and allocation counters after setup |
| `b.StopTimer()` | pause for per-iteration setup inside the loop |
| `b.StartTimer()` | resume after it |
| `b.ReportAllocs()` | force `B/op`/`allocs/op` for this benchmark without `-benchmem` |
| `b.ReportMetric(v, unit)` | add a custom column |
| `b.SetBytes(n)` | report a `MB/s` throughput column |
| `b.Elapsed()` | the measured duration so far |

Stop/Start inside the loop when each iteration needs fresh state:

```go
func BenchmarkInsert(b *testing.B) {
  for b.Loop() {
    b.StopTimer()
    tree := buildTree(1000)
    b.StartTimer()
    tree.Insert(42)
  }
}
```

They are not free — each pair costs real time — so for a cheap operation the overhead can dominate.
Better, where possible: build N fixtures up front, or make the operation idempotent so no reset is
needed.

## b.ReportMetric and b.SetBytes

```go
func BenchmarkHashReportMetric(b *testing.B) {
  data := make([]byte, 4096)
  b.SetBytes(int64(len(data)))
  n := 0
  for b.Loop() {
    Hash(data)
    n++
  }
  b.ReportMetric(float64(n)/b.Elapsed().Seconds(), "hash/s")
  b.ReportMetric(float64(len(data)), "bytes/hash")
}
```

Output:

```
BenchmarkHashReportMetric-12    	  983524	      1227 ns/op	3338.64 MB/s	      4096 bytes/hash	    815097 hash/s	       0 B/op	       0 allocs/op
```

`SetBytes` produced the `3338.64 MB/s` column; the two `ReportMetric` calls added their own.
Custom metrics are the way to report cache hit rates, rows processed, or compression ratios —
`benchstat` picks them up and compares them like any other column. Passing `"ns/op"` as the unit
replaces the default column.

## b.RunParallel and -cpu

`b.RunParallel` runs the body on `GOMAXPROCS` goroutines, which is how you measure contention
rather than single-threaded speed.

```go
func BenchmarkCounterSerial(b *testing.B) {
  var ctr Counter
  for b.Loop() {
    ctr.Inc()
  }
}

func BenchmarkCounterParallel(b *testing.B) {
  var ctr Counter
  b.RunParallel(func(pb *testing.PB) {
    for pb.Next() {
      ctr.Inc()
    }
  })
}
```

`Counter` is a mutex-guarded int. Output:

```
BenchmarkCounterSerial-12       	491587286	         2.248 ns/op	       0 B/op	       0 allocs/op
BenchmarkCounterParallel-12     	10623687	       107.8 ns/op	       0 B/op	       0 allocs/op
```

48x slower in parallel — that is lock contention, and it is invisible to the serial benchmark. The
`*testing.PB` is a per-goroutine iteration source; `pb.Next()` returns false when the shared budget
is exhausted. Note you do **not** use `b.Loop()` or `b.N` inside `RunParallel`.

`-cpu` sweeps `GOMAXPROCS`, which turns one benchmark into a scalability curve.
`go test -count=1 -run '^$' -bench 'Counter' -cpu 1,4,12 .`. Output:

```
BenchmarkCounterSerial         	498567138	         2.276 ns/op
BenchmarkCounterSerial-4       	535184674	         2.210 ns/op
BenchmarkCounterSerial-12      	551064549	         2.202 ns/op
BenchmarkCounterParallel       	568512912	         2.103 ns/op
BenchmarkCounterParallel-4     	17421834	        63.36 ns/op
BenchmarkCounterParallel-12    	11295087	       109.6 ns/op
```

Read the parallel row: 2.1 ns at one CPU, 63.4 at four, 109.6 at twelve. Adding cores made it
**worse**, which is the signature of a contended lock — the fix is sharding, `sync/atomic`, or
per-goroutine state ([11.2](../11-sync-context/02-mutex.md),
[11.4](../11-sync-context/04-atomic.md)). The serial benchmark is flat, as expected, and note the
first row has no `-N` suffix because `GOMAXPROCS=1`.

## Sub-benchmarks over input sizes

`b.Run` works like `t.Run` and is how you show complexity rather than assert it.

```go
package f

import (
  "fmt"
  "testing"
)

func BenchmarkContains(b *testing.B) {
  for _, n := range []int{10, 1000, 100000} {
    data := Data(n)
    target := n - 1
    b.Run(fmt.Sprintf("linear/n=%d", n), func(b *testing.B) {
      for b.Loop() {
        ContainsLinear(data, target)
      }
    })
    b.Run(fmt.Sprintf("binary/n=%d", n), func(b *testing.B) {
      for b.Loop() {
        ContainsBinary(data, target)
      }
    })
  }
}
```

`go test -count=1 -run '^$' -bench 'BenchmarkContains' .`. Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkContains/linear/n=10-12         	370905310	         2.953 ns/op
BenchmarkContains/binary/n=10-12         	549256166	         2.192 ns/op
BenchmarkContains/linear/n=1000-12       	 5176810	       234.2 ns/op
BenchmarkContains/binary/n=1000-12       	264697537	         4.357 ns/op
BenchmarkContains/linear/n=100000-12     	   53179	     22498 ns/op
BenchmarkContains/binary/n=100000-12     	165473865	         7.284 ns/op
PASS
ok  	example.com/scratch	7.563s
```

The complexity is right there in the numbers. Linear: 2.95 → 234.2 → 22498, each ~100x for 100x the
input — O(n). Binary: 2.19 → 4.36 → 7.28, roughly +2 ns per 100x — O(log n). And the crossover
matters: at n=10 the linear scan is only 35% slower, so for small slices the simpler code is fine.
Target one with `-bench 'BenchmarkContains/binary'`.

## The dead code elimination trap

The most common benchmarking mistake: the compiler notices the result is unused, deletes the call,
and you benchmark an empty loop.

```go
package h

func Square(x int) int { return x * x }
```

```go
package h

import "testing"

func BenchmarkSquareDiscarded(b *testing.B) {
  for range b.N {
    Square(42)
  }
}

var Sink int

func BenchmarkSquareSink(b *testing.B) {
  var local int
  for range b.N {
    local = Square(42)
  }
  Sink = local
}

func BenchmarkSquareLoop(b *testing.B) {
  for b.Loop() {
    Square(42)
  }
}
```

`go test -count=1 -run '^$' -bench '.' -benchtime 2s .`. Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkSquareDiscarded-12    	1000000000	         0.2324 ns/op
BenchmarkSquareSink-12         	1000000000	         0.2232 ns/op
BenchmarkSquareLoop-12         	1000000000	         1.564 ns/op
```

Two tells that a benchmark measured nothing: **`ns/op` below about 1** on anything that is not a
trivial field read, and an iteration count pinned at exactly `1000000000` (the hard cap). A
multiply cannot take 0.23 ns; the loop was emptied.

The interesting part is the second row. The classic advice is "assign to a package-level sink", and
here **it did not work** — 0.2232 ns/op, identical to the discarded version. `Square(42)` inlines
to a compile-time constant, so there is nothing left to compute no matter where you store it. A
sink inside the loop (`Sink = Square(42)`) measured 0.2328 ns/op, the same.

`for b.Loop()` reports **1.564 ns/op**, 6.7x higher, because it is the only one of the three that
keeps the loop body's values alive by construction. That is the fix. If you are stuck on `b.N`,
vary the input so the result cannot be folded (`Square(i)` over the loop index) *and* consume it,
but prefer `b.Loop()`.

A corollary for interpreting any benchmark: if a change makes something look impossibly fast,
suspect elimination before celebrating.

## What a micro-benchmark cannot tell you

A benchmark measures one function with warm caches, no contention, and a perfectly predictable
access pattern. Real workloads have none of that. Specific ways micro-benchmarks mislead:

- **GC is attributed elsewhere.** A function that allocates heavily may benchmark fine while making
  everything *else* slower; `allocs/op` is the signal, not `ns/op`.
- **Cache behaviour is unrealistic.** Looping over the same 4KB buffer four million times is a
  100% cache hit rate you will never see in production.
- **Branch prediction is trained** by uniform input; mixed real input is slower.
- **Nothing contends.** Only `RunParallel` with `-cpu` exposes contention.
- **The input is one size.** Sub-benchmarks over sizes are the minimum defence.

So use benchmarks to compare two implementations of the same thing, and use a profiler on the real
workload to find out what to optimize in the first place ([15.6](../15-tooling/06-pprof.md)).
Optimizing a function that accounts for 0.2% of your latency is a measurable, verifiable waste of
time.

## Quick Reference

| Form | Meaning |
|---|---|
| `func BenchmarkXxx(b *testing.B)` | a benchmark |
| `for b.Loop() { ... }` | **idiomatic**: keeps values alive, manages the timer |
| `for range b.N { ... }` | the classic form; needs `ResetTimer` and a sink |
| `b.Run(name, fn)` | sub-benchmark, named `Parent/name` |
| `b.RunParallel(func(pb *testing.PB){ for pb.Next() {...} })` | contention measurement |
| `b.ResetTimer()` / `StopTimer()` / `StartTimer()` | timer control |
| `b.ReportAllocs()` | allocation columns for this benchmark |
| `b.ReportMetric(v, "unit")` | a custom column |
| `b.SetBytes(n)` | adds a `MB/s` column |
| `b.Elapsed()` | measured duration so far |

| Command | Effect |
|---|---|
| `go test -bench . -benchmem ./...` | everything, with allocations |
| `go test -run '^$' -bench 'X$'` | one benchmark, no tests |
| `go test -bench . -benchtime 3s` | longer, steadier measurement |
| `go test -bench . -benchtime 500x` | fixed iteration count |
| `go test -bench . -count=6 > new.txt` | samples for benchstat |
| `benchstat old.txt new.txt` | statistical comparison with p-values |
| `go test -bench . -cpu 1,4,12` | a scalability curve |
| `go test -bench . -cpuprofile cpu.out` | profile the benchmark ([15.6](../15-tooling/06-pprof.md)) |

| Output column | Meaning |
|---|---|
| `-12` suffix | `GOMAXPROCS` |
| iterations | how many times the body ran; not a result |
| `ns/op` | time per operation — the comparison number |
| `B/op` | heap bytes per operation |
| `allocs/op` | heap allocations per operation; the most stable signal |
| `MB/s` | from `b.SetBytes` |

## Common Mistakes

**Omitting `b.ResetTimer()` in a `b.N` benchmark with expensive setup.** The example above was 290x
wrong and the wrong number looked entirely plausible; `for b.Loop()` removes the trap.

**Benchmarking a call whose result is discarded.** The compiler deletes it — suspect any
sub-nanosecond `ns/op` or an iteration count of exactly `1000000000`.

**Trusting a package-level sink to defeat elimination.** It did not here; `for b.Loop()` is the
reliable fix.

**Comparing two single runs.** An unchanged benchmark varied 4% across five runs; use `-count=6`
and `benchstat`, and believe the `p` value.

**Using `-benchtime 100x` to make benchmarks fast.** 100 iterations measured 719.6 ns/op against
~300 for four million — cold caches, not a result.

**Quoting `ns/op` across machines.** The `cpu:` header line exists because the number is
meaningless without it; compare ratios.

**Optimizing what a benchmark says is slow rather than what a profiler says is hot.** A
micro-benchmark cannot tell you whether the function matters.

## Exercises

1. Benchmark `Concat` vs `Builder` vs `BuilderGrown` at 10, 100 and 10000 parts with sub-benchmarks.
   Find the size at which `Grow` starts winning on `ns/op`, not just allocations.
2. Write a benchmark with the setup inside the timed region, capture the wrong number, then fix it
   with `ResetTimer` and again with `b.Loop()`. Report all three.
3. Construct a function whose benchmark is eliminated. Verify it with the sub-nanosecond tell, then
   defeat the elimination three ways: `b.Loop()`, varying the input, and a sink. Report which
   actually worked.
4. Run one benchmark with `-count=10` and compute the min, max and spread yourself. Then use
   `benchstat` on two variants and state the smallest improvement you could detect with `p<0.05`.
5. Take a `sync.Mutex`-guarded map and benchmark it with `RunParallel` at `-cpu 1,2,4,8,12`. Replace
   it with sharded locks and produce a benchstat comparison at `-cpu 12`.
6. Add `b.ReportMetric` to a benchmark to report "entries processed per second", and confirm
   `benchstat` compares the custom column.
7. Benchmark a function, then profile the same function with `-cpuprofile` and `go tool pprof`.
   Report one thing the profile told you that the benchmark did not.

---

[← 14.4 testify, Briefly](04-testify.md) · [Course Index](../README.md) · [14.6 Fuzzing →](06-fuzzing.md)
