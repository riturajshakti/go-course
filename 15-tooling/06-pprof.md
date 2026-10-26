[← 15.5 Cross-Compilation and Static Binaries](05-cross-compilation.md) · [Course Index](../README.md) · [15.7 Delve, Briefly →](07-delve.md)

# 15.6 — pprof Profiling

## What You'll Learn

- `runtime/pprof` in a program, `net/http/pprof` in a service, `-cpuprofile` in a benchmark
- `pprof -top`'s five columns, and why `-cum` answers a different question than `flat`
- The heap profile's four sample types, and that `alloc_space` and `inuse_space` can disagree completely
- A real optimisation, profile-driven: 91x faster, 1000x less garbage

## Coming From Another Language

| Other | Go |
|---|---|
| `node --prof` + a separate processor | `runtime/pprof` is stdlib, output is a protobuf anyone can read |
| `cProfile` / `py-spy` | sampling CPU profiler at 100 Hz, built in, negligible overhead |
| `perf record` needs root and symbols | works unprivileged on your own process |
| heap snapshots in Chrome DevTools | `heap` profile with four distinct sample types |
| `console.time()` everywhere | `-list=Func` gives per-line attribution for free |
| no production profiling without an APM vendor | `import _ "net/http/pprof"` and it is live |

## The rule

Profile first. Go's compiler and GC are good enough that your intuition about
where the time goes is wrong more often than it is right, and the hot spot is
almost never the clever code you remember writing. Everything in this lesson
exists to replace a guess with a measurement.

## runtime/pprof in a program

A program with two distinct problems: `hashLoop` burns CPU, `slowJoin` burns
allocations, and `fillCache` retains memory.

```go
package main

import (
  "crypto/sha256"
  "fmt"
  "io"
  "os"
  "runtime"
  "runtime/pprof"
  "strconv"
)

func fnv(data []byte) uint64 {
  h := uint64(14695981039346656037)
  for _, b := range data {
    h ^= uint64(b)
    h *= 1099511628211
  }
  return h
}

func hashLoop(n int) uint64 {
  var acc uint64
  buf := make([]byte, 0, 32)
  for i := 0; i < n; i++ {
    buf = strconv.AppendInt(buf[:0], int64(i), 10)
    acc ^= fnv(buf)
    acc ^= uint64(sha256.Sum256(buf)[0])
  }
  return acc
}

func slowJoin(n int) string {
  s := ""
  for i := 0; i < n; i++ {
    s += strconv.Itoa(i) + ","
  }
  return s
}

var cache [][]byte

func fillCache(n int) {
  cache = make([][]byte, 0, n)
  for i := 0; i < n; i++ {
    cache = append(cache, make([]byte, 1024))
  }
}

func writeProfile(name string, write func(io.Writer) error) {
  f, err := os.Create(name)
  if err != nil {
    panic(err)
  }
  if err := write(f); err != nil {
    panic(err)
  }
  if err := f.Close(); err != nil {
    panic(err)
  }
}

func main() {
  cpu, err := os.Create("cpu.prof")
  if err != nil {
    panic(err)
  }
  if err := pprof.StartCPUProfile(cpu); err != nil {
    panic(err)
  }

  acc := hashLoop(8000000)
  s := slowJoin(40000)
  fillCache(20000)

  pprof.StopCPUProfile()
  if err := cpu.Close(); err != nil {
    panic(err)
  }

  runtime.GC()
  writeProfile("mem.prof", pprof.WriteHeapProfile)

  fmt.Println("acc:", acc, "len:", len(s), "cache:", len(cache))
}
```

```bash
go build -o prof .
GOMAXPROCS=1 ./prof
```

Output:

```
acc: 16537844561382142899 len: 228890 cache: 20000
```

Four things in that `main` are load-bearing. `StopCPUProfile` must run before
you read the file, and `defer` is a trap here if anything else in `main`
panics. The heap profile needs a `runtime.GC()` first, or `inuse_*` reports
whatever the last GC cycle happened to leave. `cache` is a package-level
variable so that the slices are still reachable when the heap profile is
written — a local would be dead by then and would show zero `inuse_space`,
which is the single most common reason a heap profile looks empty. And
`GOMAXPROCS=1` is only here to keep this lesson's numbers legible; it is not a
profiling requirement.

## pprof -top and its columns

```bash
go tool pprof -top -nodecount=12 prof cpu.prof
```

Output (sample counts vary run to run; the shape does not):

```
File: prof
Type: cpu
Time: 2026-10-26 13:39:07 IST
Duration: 776.52ms, Total samples = 560ms (72.12%)
Showing nodes accounting for 520ms, 92.86% of 560ms total
Showing top 12 nodes out of 32
      flat  flat%   sum%        cum   cum%
     190ms 33.93% 33.93%      190ms 33.93%  runtime.kevent
     140ms 25.00% 58.93%      140ms 25.00%  crypto/internal/fips140/sha256.blockSHA2
      40ms  7.14% 66.07%       40ms  7.14%  internal/strconv.formatBase10
      40ms  7.14% 73.21%       40ms  7.14%  runtime.memmove
      20ms  3.57% 76.79%      210ms 37.50%  crypto/internal/fips140/sha256.(*Digest).Write
      20ms  3.57% 80.36%       20ms  3.57%  internal/byteorder.BEPutUint64 (inline)
      20ms  3.57% 83.93%       20ms  3.57%  main.fnv
      10ms  1.79% 85.71%      260ms 46.43%  crypto/internal/fips140/sha256.(*Digest).Sum
      10ms  1.79% 87.50%      250ms 44.64%  crypto/internal/fips140/sha256.(*Digest).checkSum
      10ms  1.79% 89.29%       10ms  1.79%  crypto/internal/fips140/sha256.New (inline)
      10ms  1.79% 91.07%      150ms 26.79%  crypto/internal/fips140/sha256.block
      10ms  1.79% 92.86%      280ms 50.00%  crypto/sha256.Sum256
```

The five columns, which are the whole interface:

| Column | Means |
|---|---|
| `flat` | time spent **in this function's own code**, excluding callees |
| `flat%` | that, as a fraction of the total samples |
| `sum%` | running total of `flat%` down the list — "the top N rows are this much of the profile" |
| `cum` | time in this function **and everything it called** |
| `cum%` | that, as a fraction of total |

Read the header too. `Duration: 776.52ms, Total samples = 560ms (72.12%)` —
the program ran for 776ms and the profiler collected 560ms of samples, 72% of
wall time, because with `GOMAXPROCS=1` the single thread was sometimes
blocked rather than running. On a multi-core run that percentage goes **above**
100% (a 4-core saturated program yields ~400%), which is normal and often
confusing the first time.

`flat` and `cum` answer different questions. `blockSHA2` has `flat` 140ms and
`cum` 140ms — it is a leaf, all its own work. `crypto/sha256.Sum256` has `flat`
10ms and `cum` 280ms — it does almost nothing itself but is responsible for
half the profile. **Optimise by `cum`, micro-optimise by `flat`.** Sort by
`cum` to find out *what to stop doing*; sort by `flat` to find out *what to
make faster*.

```bash
go tool pprof -top -cum -nodecount=8 prof cpu.prof
```

Output:

```
Showing top 8 nodes out of 32
      flat  flat%   sum%        cum   cum%
      10ms  1.79%  1.79%      360ms 64.29%  main.hashLoop
         0     0%  1.79%      360ms 64.29%  main.main
         0     0%  1.79%      360ms 64.29%  runtime.main
      10ms  1.79%  3.57%      280ms 50.00%  crypto/sha256.Sum256
      10ms  1.79%  5.36%      260ms 46.43%  crypto/internal/fips140/sha256.(*Digest).Sum
      10ms  1.79%  7.14%      250ms 44.64%  crypto/internal/fips140/sha256.(*Digest).checkSum
      20ms  3.57% 10.71%      210ms 37.50%  crypto/internal/fips140/sha256.(*Digest).Write
         0     0% 10.71%      200ms 35.71%  runtime.systemstack
```

Now the story is legible top-down: `main.hashLoop` is 64% of the program, and
50% of it is `sha256.Sum256`. That is actionable; the `flat` view's top row
was not.

One honest note on that `flat` view: `runtime.kevent` at 33.93% is the
scheduler's kqueue wait on darwin, inflated here because the total sample count
is small and this is a virtualised host. It is profiler and scheduler
overhead, not your program. Learn to recognise the runtime noise floor —
`kevent`, `usleep`, `madvise`, `pthread_cond_wait`, `pthread_kill` — and read
past it. The `-cum` view and `-list` both route around it automatically, which
is one more reason to start there.

## pprof -list: per-line attribution

This is the command that makes profiling worth doing:

```bash
go tool pprof -list 'main.hashLoop' prof cpu.prof
```

Output (the source path is this scratch directory; yours will differ):

```
Total: 560ms
ROUTINE ======================== main.hashLoop in /tmp/scratch/main.go
      10ms      360ms (flat, cum) 64.29% of Total
         .          .     22:func hashLoop(n int) uint64 {
         .          .     23:	var acc uint64
         .          .     24:	buf := make([]byte, 0, 32)
      10ms       10ms     25:	for i := 0; i < n; i++ {
         .       50ms     26:		buf = strconv.AppendInt(buf[:0], int64(i), 10)
         .       20ms     27:		acc ^= fnv(buf)
         .      280ms     28:		acc ^= uint64(sha256.Sum256(buf)[0])
         .          .     29:	}
         .          .     30:	return acc
         .          .     31:}
```

Line 28 is 280ms of the function's 360ms. No guessing, no instrumentation, no
`time.Now()` sprinkled through the loop. The argument is a **regexp** matched
against the fully-qualified function name, so `-list 'hashLoop'` works and
`-list 'main\.'` lists every function in your package. In a test binary the
qualifier is the import path, not `main`: `-list 'example.com/scratch.slowJoin'`.

## The heap profile's four sample types

One file, four different questions:

```bash
go tool pprof -alloc_space  -top -nodecount=5 prof mem.prof
go tool pprof -alloc_objects -top -nodecount=5 prof mem.prof
go tool pprof -inuse_space  -top -nodecount=5 prof mem.prof
go tool pprof -inuse_objects -top -nodecount=5 prof mem.prof
```

Output:

```
Type: alloc_space
Showing nodes accounting for 4.23GB, 99.46% of 4.25GB total
Dropped 13 nodes (cum <= 0.02GB)
      flat  flat%   sum%        cum   cum%
    4.23GB 99.46% 99.46%     4.23GB 99.46%  main.slowJoin
         0     0% 99.46%     4.25GB   100%  main.main
         0     0% 99.46%     4.25GB   100%  runtime.main
```

```
Type: alloc_objects
Showing nodes accounting for 61127, 100% of 61142 total
Dropped 12 nodes (cum <= 305)
      flat  flat%   sum%        cum   cum%
     40115 65.61% 65.61%      40115 65.61%  main.slowJoin
     21012 34.37%   100%      21012 34.37%  main.fillCache (inline)
         0     0%   100%      61132   100%  main.main
         0     0%   100%      61132   100%  runtime.main
```

```
Type: inuse_space
Showing nodes accounting for 20.52MB, 100% of 20.52MB total
      flat  flat%   sum%        cum   cum%
   20.52MB   100%   100%    20.52MB   100%  main.fillCache (inline)
         0     0%   100%    20.52MB   100%  main.main
         0     0%   100%    20.52MB   100%  runtime.main
```

```
Type: inuse_objects
Showing nodes accounting for 21012, 100% of 21012 total
      flat  flat%   sum%        cum   cum%
     21012   100%   100%      21012   100%  main.fillCache (inline)
         0     0%   100%      21012   100%  main.main
         0     0%   100%      21012   100%  runtime.main
```

The two views **completely disagree about who the culprit is**, and both are
right:

| Sample type | Measures | Use it to find |
|---|---|---|
| `alloc_space` | total bytes **ever** allocated | GC pressure — the thing making your program slow |
| `alloc_objects` | total allocation **count** | allocation-rate problems; many small allocations cost more than the bytes suggest |
| `inuse_space` | bytes **live at profile time** | memory leaks and footprint — the thing making your pod get OOM-killed |
| `inuse_objects` | live object **count** | leaks of many small objects; also a GC-scan-cost proxy |

`slowJoin` allocated **4.23 GB** and retains none of it — a pure garbage
machine, invisible to `inuse_space`. `fillCache` allocated 20 MB and retains
all of it — invisible to `alloc_space` at this scale. A slow program with flat
memory is an `alloc_space` problem; a program whose RSS climbs forever is an
`inuse_space` problem. Pick the sample type that matches the symptom, or you
will profile the wrong function.

`inuse_space` is the default when you pass no flag. Two more knobs:
`runtime.MemProfileRate` controls the sampling rate (default: one sample per
512 KB allocated; set it to 1 at startup for exact profiles and a large
slowdown), and `-sample_index=alloc_space` is the long-hand for the flags
above.

## go test -cpuprofile -memprofile -benchmem

For anything you can write a benchmark for (14.5), this is the fastest route to
a profile — no code changes at all:

```go
package main

import "testing"

func BenchmarkSlowJoin(b *testing.B) {
  for b.Loop() {
    _ = slowJoin(5000)
  }
}

func BenchmarkFastJoin(b *testing.B) {
  for b.Loop() {
    _ = fastJoin(5000)
  }
}
```

```bash
go test -bench=Join -benchmem -cpuprofile=cpu.prof -memprofile=mem.prof .
```

Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkSlowJoin-12    	     237	   4961498 ns/op	60599036 B/op	    9905 allocs/op
BenchmarkFastJoin-12    	   22234	     54290 ns/op	   59840 B/op	    4901 allocs/op
PASS
ok  	example.com/scratch	3.017s
```

`-benchmem` adds the last two columns and costs nothing — leave it on
permanently. One caveat: `-cpuprofile` with multiple benchmarks writes a
*combined* profile, so run one benchmark at a time
(`-bench=^BenchmarkSlowJoin$`) when you intend to analyse it. `go test` also
leaves the test binary behind when profiling, which is what lets `pprof`
resolve symbols.

## -base: comparing two profiles

Profile each version separately, then subtract:

```bash
go test -bench=SlowJoin -memprofile=slow-mem.prof .
go test -bench=FastJoin -memprofile=fast-mem.prof .
go tool pprof -alloc_space -base=fast-mem.prof -top -nodecount=5 slow-mem.prof
```

Output:

```
Dropped 41 nodes (cum <= 75.15MB)
Showing top 5 nodes out of 12
      flat  flat%   sum%        cum   cum%
13726.54MB 91.33% 91.33% 13733.54MB 91.38%  example.com/scratch.slowJoin
 -897.15MB  5.97% 85.36%  -897.15MB  5.97%  internal/bytealg.MakeNoZero
 -387.51MB  2.58% 82.79%  -387.51MB  2.58%  internal/strconv.FormatInt
         0     0% 82.79% -1291.66MB  8.59%  example.com/scratch.BenchmarkFastJoin
         0     0% 82.79% 13733.54MB 91.38%  example.com/scratch.BenchmarkSlowJoin
```

Negative numbers are the baseline's cost that the measured profile does not
have. This is how you verify an optimisation actually moved the allocation you
targeted rather than relocating it. `-diff_base` is the variant that also
normalises for differing total sample counts — use it when the two runs did
different amounts of work.

## The payoff: optimising a real function, profile-guided

`slowJoin` is 4.96 ms/op, 60 MB/op, 9905 allocs/op. The `-list` view says why:

```bash
go tool pprof -list 'example.com/scratch.slowJoin' slow-mem.prof
```

The `s += ...` line carries all of it. Each `+=` allocates a **new** string
containing the whole accumulated result, so building an `n`-element string
costs O(n²) bytes copied. 5000 iterations averaging ~12 KB of accumulated
string means ~60 MB of garbage per call, and the benchmark's 237 iterations
produced the 13.4 GB above.

The fix is `strings.Builder` (6.4) with a pre-sized buffer, so the backing array
grows a handful of times instead of 5000:

```go
package main

import (
  "strconv"
  "strings"
)

func fastJoin(n int) string {
  var b strings.Builder
  b.Grow(n * 7)
  for i := 0; i < n; i++ {
    b.WriteString(strconv.Itoa(i))
    b.WriteByte(',')
  }
  return b.String()
}
```

| Version | ns/op | B/op | allocs/op |
|---|---|---|---|
| `slowJoin` | 4,961,498 | 60,599,036 | 9,905 |
| `fastJoin` | 54,290 | 59,840 | 4,901 |
| change | **91x faster** | **1013x less** | 2x fewer |

And the profile after:

```bash
go tool pprof -alloc_space -top -nodecount=4 fast-mem.prof
```

Output:

```
Showing top 4 nodes out of 10
      flat  flat%   sum%        cum   cum%
  897.15MB 69.16% 69.16%   897.15MB 69.16%  internal/bytealg.MakeNoZero
  394.51MB 30.41% 99.57%  1291.66MB 99.57%  internal/strconv.FormatInt
         0     0% 99.57%  1291.66MB 99.57%  example.com/scratch.BenchmarkFastJoin
         0     0% 99.57%  1291.66MB 99.57%  example.com/scratch.fastJoin
```

`slowJoin` is gone from the profile entirely. What remains is `MakeNoZero` (the
Builder's buffer growth) and `FormatInt` (the 4901 remaining allocations — one
`strconv.Itoa` string per iteration). The next step, if 54 µs still mattered,
would be `strconv.AppendInt` into a reused `[]byte` to eliminate those too.
That is the profiling loop: measure, fix the top entry, measure again, stop
when the top entry is something you cannot cheaply remove.

## net/http/pprof on a live server

One blank import registers eleven handlers on `http.DefaultServeMux`:

```go
package main

import (
  "crypto/sha256"
  "log"
  "net/http"
  _ "net/http/pprof"
  "strconv"
)

func work(n int) uint64 {
  var acc uint64
  for i := 0; i < n; i++ {
    acc ^= uint64(sha256.Sum256([]byte(strconv.Itoa(i)))[0])
  }
  return acc
}

func main() {
  http.HandleFunc("/busy", func(w http.ResponseWriter, r *http.Request) {
    if _, err := w.Write([]byte(strconv.FormatUint(work(300000), 10))); err != nil {
      log.Println(err)
    }
  })
  log.Fatal(http.ListenAndServe("localhost:6060", nil))
}
```

```bash
go run . &
curl -s http://localhost:6060/debug/pprof/
```

Output (the HTML index, trimmed to its table):

```
Types of profiles available:
<table>
<thead><td>Count</td><td>Profile</td></thead>
<tr><td>24</td><td><a href='allocs?debug=1'>allocs</a></td></tr>
<tr><td>0</td><td><a href='block?debug=1'>block</a></td></tr>
<tr><td>0</td><td><a href='cmdline?debug=1'>cmdline</a></td></tr>
<tr><td>3</td><td><a href='goroutine?debug=1'>goroutine</a></td></tr>
<tr><td>0</td><td><a href='goroutineleak?debug=1'>goroutineleak</a></td></tr>
<tr><td>24</td><td><a href='heap?debug=1'>heap</a></td></tr>
<tr><td>0</td><td><a href='mutex?debug=1'>mutex</a></td></tr>
<tr><td>0</td><td><a href='profile?debug=1'>profile</a></td></tr>
<tr><td>0</td><td><a href='symbol?debug=1'>symbol</a></td></tr>
<tr><td>14</td><td><a href='threadcreate?debug=1'>threadcreate</a></td></tr>
<tr><td>0</td><td><a href='trace?debug=1'>trace</a></td></tr>
</table>
```

`goroutineleak` is new in Go 1.27 and reports goroutines the runtime has
determined can never make progress — 12.5's leak hunt gets considerably easier.

Now point `pprof` straight at the URL while the server is under load:

```bash
# in another shell: a loop hitting /busy continuously
go tool pprof -top -nodecount=10 -seconds=5 http://localhost:6060/debug/pprof/profile
```

Output:

```
Fetching profile over HTTP from http://localhost:6060/debug/pprof/profile?seconds=5
Please wait... (5s)
Saved profile in /Users/raj/pprof/pprof.srv.samples.cpu.001.pb.gz
Duration: 5.13s, Total samples = 10880ms (212.25%)
Showing nodes accounting for 9770ms, 89.80% of 10880ms total
Dropped 82 nodes (cum <= 54.40ms)
Showing top 10 nodes out of 74
      flat  flat%   sum%        cum   cum%
    2800ms 25.74% 25.74%     2800ms 25.74%  runtime.madvise
    2780ms 25.55% 51.29%     2780ms 25.55%  runtime.usleep
    1190ms 10.94% 62.22%     1190ms 10.94%  runtime.pthread_kill
     870ms  8.00% 70.22%      870ms  8.00%  crypto/internal/fips140/sha256.blockSHA2
     760ms  6.99% 77.21%      760ms  6.99%  syscall.rawsyscalln
     370ms  3.40% 80.61%     1760ms 16.18%  crypto/internal/fips140/sha256.(*Digest).checkSum
     310ms  2.85% 83.46%     1440ms 13.24%  crypto/internal/fips140/sha256.(*Digest).Write
     270ms  2.48% 85.94%      270ms  2.48%  runtime.procyieldAsm
     230ms  2.11% 88.05%     2050ms 18.84%  crypto/internal/fips140/sha256.(*Digest).Sum
     190ms  1.75% 89.80%      190ms  1.75%  runtime.memmove
```

`212.25%` of wall time — the handler was running on more than two cores. `pprof`
downloaded the profile, fetched the symbol table from `/debug/pprof/symbol`,
and cached the result under `~/pprof/`. The `-cum` view of the same profile
finds the handler:

```bash
go tool pprof -top -cum -nodecount=10 srv ~/pprof/pprof.srv.samples.cpu.002.pb.gz
```

Output:

```
Showing top 10 nodes out of 74
      flat  flat%   sum%        cum   cum%
         0     0%     0%      4.18s 38.42%  runtime.systemstack
         0     0%     0%      3.62s 33.27%  net/http.(*conn).serve
         0     0%     0%      3.02s 27.76%  runtime.lock (inline)
         0     0%     0%      3.02s 27.76%  runtime.lock2
         0     0%     0%      3.02s 27.76%  runtime.lockWithRank (inline)
     0.01s 0.092% 0.092%      2.88s 26.47%  runtime.(*mheap).alloc.func1
         0     0% 0.092%      2.88s 26.47%  runtime.(*mheap).allocSpan
         0     0% 0.092%      2.86s 26.29%  main.main.func1
     0.01s 0.092%  0.18%      2.86s 26.29%  main.work
         0     0%  0.18%      2.86s 26.29%  net/http.(*ServeMux).ServeHTTP
```

`main.work`, reached through `ServeMux.ServeHTTP` from `(*conn).serve`, at
26.29% cum. Also visible: `runtime.lock2` and `mheap.allocSpan` at 27.76% —
heap-allocation lock contention, because `work` allocates a fresh `[]byte`
per iteration. That is the next thing to fix, and the profile found it without
anyone suspecting it.

The URL form works for every endpoint:

```bash
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof http://localhost:6060/debug/pprof/allocs
go tool pprof -seconds=30 http://localhost:6060/debug/pprof/profile
curl -s "http://localhost:6060/debug/pprof/goroutine?debug=1" | head -20
curl -o trace.out "http://localhost:6060/debug/pprof/trace?seconds=5"
```

`?debug=1` gives a human-readable text form straight in the terminal, which is
how you read a goroutine dump without `pprof` at all:

```
goroutine profile: total 3
1 @ 0x1049d3efc 0x104a13494 0x104baadb4 ...
#	0x104baadb3	runtime/pprof.writeRuntimeProfile+0xb3	.../pprof.go:848
#	0x104baaa5f	runtime/pprof.writeGoroutine+0x4f	.../pprof.go:781
#	0x104ba82c3	runtime/pprof.(*Profile).WriteTo+0x143	.../pprof.go:405
#	0x104bbca43	net/http/pprof.handler.ServeHTTP+0x433	.../pprof.go:272
...
```

The leading number is how many goroutines share that identical stack — which
is the whole technique for finding a leak: 40,000 goroutines parked on the same
channel receive is one line, not 40,000.

## Goroutine, block, and mutex profiles

The CPU profile cannot see time spent *waiting*. These three can. Unlike CPU
and heap, block and mutex profiling is **off by default** and must be enabled:

```go
package main

import (
  "fmt"
  "os"
  "runtime"
  "runtime/pprof"
  "sync"
  "time"
)

func contend(workers, iters int) {
  var mu sync.Mutex
  shared := 0
  var wg sync.WaitGroup
  for w := 0; w < workers; w++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      for i := 0; i < iters; i++ {
        mu.Lock()
        shared++
        time.Sleep(time.Microsecond)
        mu.Unlock()
      }
    }()
  }
  wg.Wait()
  fmt.Println("shared:", shared)
}

func writeProfile(name, which string) {
  f, err := os.Create(name)
  if err != nil {
    panic(err)
  }
  if err := pprof.Lookup(which).WriteTo(f, 0); err != nil {
    panic(err)
  }
  if err := f.Close(); err != nil {
    panic(err)
  }
}

func main() {
  runtime.SetBlockProfileRate(1)
  runtime.SetMutexProfileFraction(1)

  contend(8, 400)

  writeProfile("block.prof", "block")
  writeProfile("mutex.prof", "mutex")
  writeProfile("goroutine.prof", "goroutine")
}
```

```bash
go build -o cont . && ./cont
```

Output:

```
shared: 3200
```

```bash
go tool pprof -top -nodecount=6 cont block.prof
```

Output (delays vary run to run):

```
Type: delay
Showing nodes accounting for 98.91ms, 100% of 98.91ms total
      flat  flat%   sum%        cum   cum%
   78.27ms 79.12% 79.12%    78.27ms 79.12%  sync.(*Mutex).Lock (inline)
   20.65ms 20.88%   100%    20.65ms 20.88%  sync.(*WaitGroup).Wait
         0     0%   100%    20.65ms 20.88%  main.contend
         0     0%   100%    78.27ms 79.12%  main.contend.func1
         0     0%   100%    20.65ms 20.88%  main.main
         0     0%   100%    20.65ms 20.88%  runtime.main
```

`Type: delay`, not `cpu`: this is **wall time spent blocked**, 78ms of it
waiting on `mu.Lock()` inside the worker goroutines. A CPU profile of the same
program would show almost nothing, because waiting burns no CPU. The
`WaitGroup.Wait` entry is `main` legitimately waiting for the workers — read
block profiles with that in mind, since every correct `Wait`, channel receive,
and `select` appears here too.

The mutex profile attributes the delay to the **holder** rather than the waiter:

```bash
go tool pprof -top -nodecount=6 cont mutex.prof
```

Output:

```
Type: delay
Showing nodes accounting for 79.83ms, 100% of 79.83ms total
Dropped 6 nodes (cum <= 0.40ms)
      flat  flat%   sum%        cum   cum%
   79.83ms   100%   100%    79.83ms   100%  sync.(*Mutex).Unlock (inline)
         0     0%   100%    79.83ms   100%  main.contend.func1
```

`Unlock` is where the time is charged, because that is the moment the runtime
knows how long others were kept waiting. So the two profiles answer different
questions: block says *who waited*, mutex says *whose critical section caused
the waiting*. For the lock you need to shrink, read the mutex profile.

| Profile | Enable with | Reports |
|---|---|---|
| `cpu` | `StartCPUProfile` / `?seconds=N` | on-CPU samples at 100 Hz |
| `heap` / `allocs` | always on | the four sample types above |
| `goroutine` | always on | every goroutine's current stack |
| `goroutineleak` | always on (Go 1.27+) | goroutines that can never progress |
| `block` | `runtime.SetBlockProfileRate(n)` | time blocked on sync primitives |
| `mutex` | `runtime.SetMutexProfileFraction(n)` | contention, charged to the holder |
| `threadcreate` | always on | stacks that created OS threads |

`SetBlockProfileRate(1)` samples every blocking event, which is fine for a
debugging session and too expensive for production; `SetBlockProfileRate(10000)`
(sample events longer than 10 µs) is a reasonable always-on setting.
`SetMutexProfileFraction(n)` samples 1/n contention events; `100` is a common
production value, `0` disables it.

## The execution tracer

`go tool trace` is a different instrument: not a sampling profiler but a
complete event log of every goroutine transition, GC cycle, syscall, and
scheduler decision, with microsecond timestamps. It is what you reach for when
the question is *why is my program not using all the cores* or *why did that
request take 200 ms when the CPU profile says it did 2 ms of work* — questions a
profile structurally cannot answer because the answer is about time *not*
spent running.

```bash
go test -bench=FastJoin -trace=tr.out .
go tool trace tr.out
```

```bash
# or from a live server
curl -o trace.out "http://localhost:6060/debug/pprof/trace?seconds=5"
go tool trace trace.out
```

The trace file is real (554,832 bytes from that benchmark run), but
`go tool trace` starts a local web server and opens a browser to render the
timeline. **Not executed here**, since this environment has no browser — so no
output block follows. What you get is a goroutine-analysis page, a
network/syscall/scheduler-latency breakdown, and the per-goroutine timeline
view that makes a serialised "parallel" pipeline instantly obvious.
`runtime/trace.WithRegion` and `trace.Log` let you annotate your own spans into
it (12.3's pipelines are the natural use).

## Profiling in production

The endpoints are safe to have, dangerous to expose. `/debug/pprof/` leaks your
full source structure, every goroutine stack, command-line flags, and
environment-shaped data; `?seconds=300` is a trivial resource-exhaustion
vector; and `profile` adds measurable overhead while it runs.

Never put them on `http.DefaultServeMux` behind your public listener. The
blank-import convenience is precisely the trap, because it registers on the
default mux — which is also what your public server uses if you passed `nil` to
`ListenAndServe`. Mount them on a separate, bound-to-localhost admin listener
instead:

```go
func startAdminServer(addr string) *http.Server {
  mux := http.NewServeMux()
  mux.HandleFunc("/debug/pprof/", pprof.Index)
  mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
  mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
  mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
  mux.HandleFunc("/debug/pprof/trace", pprof.Trace)

  srv := &http.Server{Addr: addr, Handler: mux}
  go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
      log.Println("admin server:", err)
    }
  }()
  return srv
}
```

(That is `net/http/pprof`'s `pprof`, not `runtime/pprof`.) Bind it to
`127.0.0.1:6060` and reach it with `kubectl port-forward`, or put it behind the
same auth middleware as your other admin routes. 18.6 covers the admin-listener
pattern alongside health checks, and 18.7 covers the metrics endpoint that
belongs beside it.

The complementary approach is continuous profiling: a sidecar scrapes
`/debug/pprof/profile?seconds=10` every few minutes and ships it to Pyroscope,
Datadog, or Google Cloud Profiler, giving you a profile from *when the incident
happened* rather than from when you remembered to look. Overhead is a few
percent of one core. Worth it for any service you are on call for.

## Quick Reference

| In-program API | Does |
|---|---|
| `pprof.StartCPUProfile(w)` / `StopCPUProfile()` | CPU profile to a writer |
| `pprof.WriteHeapProfile(w)` | heap profile (call `runtime.GC()` first) |
| `pprof.Lookup("block").WriteTo(w, 0)` | any named profile |
| `runtime.SetBlockProfileRate(n)` | enable block profiling |
| `runtime.SetMutexProfileFraction(n)` | enable mutex profiling |
| `runtime.MemProfileRate = 1` | exact heap profiling (slow) |
| `import _ "net/http/pprof"` | register endpoints on the default mux |
| `pprof.Do(ctx, labels, f)` | attach labels, visible via `-tagfocus` |

| `go test` flag | Does |
|---|---|
| `-cpuprofile=f` | CPU profile from the benchmark |
| `-memprofile=f` | heap profile |
| `-blockprofile=f` | block profile |
| `-mutexprofile=f` | mutex profile |
| `-trace=f` | execution trace |
| `-benchmem` | B/op and allocs/op columns |
| `-benchtime=10s` / `-benchtime=100x` | longer or fixed-count runs |

| `go tool pprof` | Does |
|---|---|
| `-top` | the table, sorted by `flat` |
| `-top -cum` | sorted by cumulative |
| `-nodecount=N` | limit rows |
| `-list=Regexp` | per-line source attribution |
| `-peek=Regexp` | callers and callees of one function |
| `-tree` | call tree text |
| `-png -output=f.png` | graph image (needs graphviz) |
| `-svg`, `-pdf` | other graph formats (needs graphviz) |
| `-http=:8080` | interactive web UI (needs graphviz) |
| `-base=other.prof` | subtract a baseline |
| `-diff_base=other.prof` | subtract, normalised for total |
| `-alloc_space`, `-inuse_space`, … | pick a heap sample type |
| `-focus=Re`, `-ignore=Re`, `-hide=Re` | filter the node set |
| `BIN PROFILE` | symbolise with a local binary |
| `http://host/debug/pprof/X` | fetch and symbolise over HTTP |

`-png` needs Graphviz, which is **not installed here**:

```bash
go tool pprof -png -output=cpu.png prof cpu.prof
```

Output:

```
failed to execute dot. Is Graphviz installed? Error: exec: "dot": executable file not found in $PATH
```

`brew install graphviz` / `apt install graphviz` fixes it, and then `-http=:8080`
gives you the flame graph, call graph, and source views in a browser. Every
analysis in this lesson was done in text mode, which is deliberate: `-top`,
`-list`, and `-base` answer most questions faster than a flame graph does, and
they work over ssh.

## Common Mistakes

**Optimising before profiling.** The hot spot is nearly always somewhere you
did not expect; `-list` takes thirty seconds and is never wrong.

**Reading `flat` when you needed `cum`.** `flat` finds the leaf to micro-optimise;
`cum` finds the call you should stop making. Start with `-cum`.

**Writing a heap profile without `runtime.GC()` first.** `inuse_*` then reflects
whatever the last GC cycle left behind, which is noise.

**Expecting `inuse_space` to show a function that churns garbage.** It retains
nothing, so it is invisible there — that is `alloc_space`. Match the sample type
to the symptom.

**Taking a local variable's allocations as a leak because `inuse_space` is
empty.** The reverse mistake: the object was already unreachable when you
profiled. Keep it reachable, or profile at the right moment.

**Blank-importing `net/http/pprof` in a service with a public default mux.**
You have just published every goroutine stack and a CPU-exhaustion endpoint to
the internet; use a separate localhost listener.

**Leaving `SetBlockProfileRate(1)` on in production.** Every blocking event is
sampled; use 10000, or enable it only while investigating.

**Trusting one short profile.** 500 ms of samples on a virtualised host is
mostly scheduler noise (see the `runtime.kevent` row above). Profile for 30
seconds under representative load.

## Exercises

1. Reproduce the `slowJoin` result: write the O(n²) version, benchmark it with
   `-benchmem`, find the line with `-list`, fix it, and report all three
   numbers before and after. Then use `-base` to prove the allocation moved.
2. Write a program where `alloc_space` and `inuse_space` name different
   functions, as the one in this lesson does. Predict both before you run
   `pprof`.
3. Add `net/http/pprof` to a server, generate sustained load, and capture a
   30-second CPU profile over HTTP. Find your handler in the `-cum` view and
   note what fraction of the profile is runtime overhead versus your code.
4. Write a program with a genuinely contended mutex. Capture both a block and a
   mutex profile, and explain why the top entry differs between them. Then
   shrink the critical section and show the delay drop.
5. Install Graphviz and open the same CPU profile with `-http=:8080`. Compare
   how long the flame graph takes to answer "where is the time" versus
   `-top -cum`.
6. Take an execution trace of a worker pool from 12.1 and use the goroutine
   analysis view to determine whether the workers are actually running in
   parallel. Then add `trace.WithRegion` annotations around each stage.
7. Leave `SetBlockProfileRate(1)` on and benchmark a channel-heavy program with
   and without it. Quantify the overhead and pick a production-safe rate.

---

[← 15.5 Cross-Compilation and Static Binaries](05-cross-compilation.md) · [Course Index](../README.md) · [15.7 Delve, Briefly →](07-delve.md)
