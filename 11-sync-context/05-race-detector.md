[← 11.4 sync/atomic](04-atomic.md) · [Course Index](../README.md) · [11.6 context: Cancellation and Timeouts →](06-context-cancel.md)

# 11.5 — The Race Detector

## What You'll Learn

- How to read a `WARNING: DATA RACE` report line by line — the two stacks, the two creation stacks, the count
- `-race` only finds races that actually *execute* in that run, so it is exactly as good as your tests
- `concurrent map writes` is a different mechanism from `-race` and fires without it
- The 1.21-vs-1.22 loop-variable change, demonstrated by pinning `go 1.21`

## Coming From Another Language

| Other | Go |
|---|---|
| JS is single-threaded; no data races by construction | real shared memory, real races |
| Python's GIL hides most races behind bytecode granularity | no GIL; `x++` on a shared int races |
| Rust's borrow checker rejects data races at *compile* time | Go catches them at *runtime*, only if they happen |
| ThreadSanitizer via `-fsanitize=thread` in C++ | the same engine, built into the toolchain as `-race` |
| `pytest -p no:randomly`, no equivalent | `go test -race ./...` is the standard pre-merge gate |

Rust's guarantee is static and total; Go's is dynamic and partial. That trade is the
single most important thing to internalize about this tool: a clean `-race` run proves
that *the interleavings you exercised* were race-free, and proves nothing about the ones
you did not. The detector is a near-zero-false-positive instrument — if it reports a
race, there is a race — but its recall depends entirely on your test coverage and on
luck. Which is why the rest of this module exists: `sync.Mutex`, `sync/atomic` and
channels are how you avoid races; `-race` is how you find the ones you missed.

## Running it

```bash
go run -race .
go test -race ./...
go build -race -o myapp .
go install -race ./cmd/...
```

`-race` is a build flag, so it applies to whatever the command builds. It requires cgo
and a supported platform (linux/amd64, linux/arm64, darwin/amd64, darwin/arm64,
windows/amd64, freebsd/amd64, netbsd/amd64, linux/ppc64le, linux/s390x).

What it does: the compiler instruments every memory access to a potentially-shared
location, and a runtime library (Google's ThreadSanitizer) maintains vector clocks that
record which goroutines have synchronized with which. When two goroutines access the same
address, at least one access is a write, and no happens-before edge connects them, it
prints a report. It understands the real synchronization primitives — channels, mutexes,
`WaitGroup`, `Once`, `sync/atomic` — so correctly synchronized code is silent.

The cost is real:

```
== plain ==
BenchmarkHash-12       	   18470	     62966 ns/op
BenchmarkShuffle-12    	   35593	     33557 ns/op
== -race ==
BenchmarkHash-12       	    5073	    234190 ns/op
BenchmarkShuffle-12    	    4368	    273835 ns/op
```

3.7x and 8.2x slower on this machine (darwin/arm64, Apple M4 Pro) for two
memory-access-heavy loops. Memory overhead is worse than the CPU cost: the runtime keeps
shadow state for every monitored word, typically 5-10x the program's own footprint, and
it tracks only the most recent four accesses per word plus a bounded number of goroutines
(8128). Those limits are why you cannot ship it.

## A real race report, annotated

A counter incremented by two goroutines with no synchronization.

```go
package main

import (
  "fmt"
  "sync"
)

var counter int

func increment(wg *sync.WaitGroup) {
  defer wg.Done()
  for range 1000 {
    counter++
  }
}

func main() {
  var wg sync.WaitGroup
  wg.Add(2)
  go increment(&wg)
  go increment(&wg)
  wg.Wait()
  fmt.Println("counter:", counter)
}
```

`go run -race .` — a real captured run. The hex address, the goroutine numbers and the
`+0x..` offsets differ on every run, and the two goroutines can appear in either order:

```
==================
WARNING: DATA RACE
Read at 0x000104cc6450 by goroutine 8:
  main.increment()
      /tmp/scratch/main.go:13 +0x84
  main.main.gowrap2()
      /tmp/scratch/main.go:21 +0x2c

Previous write at 0x000104cc6450 by goroutine 7:
  main.increment()
      /tmp/scratch/main.go:13 +0x9c
  main.main.gowrap1()
      /tmp/scratch/main.go:20 +0x2c

Goroutine 8 (running) created at:
  main.main()
      /tmp/scratch/main.go:21 +0x108

Goroutine 7 (finished) created at:
  main.main()
      /tmp/scratch/main.go:20 +0xa8
==================
counter: 2000
Found 1 data race(s)
exit status 66
```

Line by line:

| Part | Meaning |
|---|---|
| `==================` | report delimiters; one pair per distinct race |
| `WARNING: DATA RACE` | the header — there are no "possible" races, only confirmed ones |
| `Read at 0x000104cc6450 by goroutine 8` | the *second* of the two conflicting accesses, with its kind and the address |
| `main.increment() /tmp/scratch/main.go:13` | the stack of that access, innermost frame first — line 13 is `counter++` |
| `main.main.gowrap2() .../main.go:21` | the compiler-generated wrapper for `go increment(&wg)` on line 21 |
| `Previous write at 0x000104cc6450 by goroutine 7` | the *first* access; `Previous` always means the earlier one |
| `Goroutine 8 (running) created at:` | where that goroutine was spawned, and its state at report time |
| `Goroutine 7 (finished) created at:` | same for the other; `(finished)` means it had already returned |
| `counter: 2000` | the program's own output — it keeps running after a report |
| `Found 1 data race(s)` | printed at exit, counting the reports |
| `exit status 66` | the race exit code |

Three details that matter when you are debugging for real.

**The address is the key to grouping.** Both accesses are at `0x000104cc6450`, which is
how you know they really are the same variable and not two coincidental lines. When a
report shows two *different* lines of your code, the shared address is what tells you
they touch one object.

**The creation stacks are usually where the fix goes.** The access stacks tell you *what*
touched the memory; the "created at" stacks tell you *which two concurrent paths* reached
it, which is the information you need to decide whether to add a lock, restructure
ownership, or serialize through a channel.

**`counter: 2000` is a trap.** The program printed the right answer *and* has a race.
A racy counter very often produces the correct total on a fast machine; the race is a
property of the code, not of the output. Never use "the result looked right" as evidence.

Note what is *not* here: Go's race runtime does not print a `SUMMARY:` line (the C/C++
ThreadSanitizer does). The closing summary is `Found N data race(s)`.

## The same race under `go test -race`

Same bug, reached from a test. Package `counter`:

```go
package counter

var counter int

func Increment(n int) {
  for range n {
    counter++
  }
}

func Value() int { return counter }
```

```go
package counter

import (
  "sync"
  "testing"
)

func TestConcurrentIncrement(t *testing.T) {
  var wg sync.WaitGroup
  for range 2 {
    wg.Go(func() { Increment(1000) })
  }
  wg.Wait()
  if got := Value(); got != 2000 {
    t.Errorf("got %d, want 2000", got)
  }
}
```

`go test -race .` — the goroutine-creation stacks are longer here because they run under
the test harness; goroutine 8's is identical to goroutine 9's and is elided:

```
==================
WARNING: DATA RACE
Read at 0x0001044441e8 by goroutine 9:
  example.com/scratch.Increment()
      /tmp/scratch/counter.go:7 +0x34
  example.com/scratch.TestConcurrentIncrement.func1()
      /tmp/scratch/counter_test.go:11 +0x18

Previous write at 0x0001044441e8 by goroutine 8:
  example.com/scratch.Increment()
      /tmp/scratch/counter.go:7 +0x4c
  example.com/scratch.TestConcurrentIncrement.func1()
      /tmp/scratch/counter_test.go:11 +0x18

Goroutine 9 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c
  testing.tRunner()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:2193 +0x164
  testing.(*T).Run.gowrap1()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:2258 +0x34

Goroutine 8 (finished) created at:
  [the same three frames]
==================
--- FAIL: TestConcurrentIncrement (0.00s)
    testing.go:1865: race detected during execution of test
FAIL
FAIL	example.com/scratch	0.504s
FAIL
```

The important difference from `go run -race`: the test framework turns the race into a
**test failure** (`race detected during execution of test`), so a racy package fails CI
even if every assertion passed. The assertion here *did* pass — `Value()` returned 2000.

Because `wg.Go` spawns the goroutine from inside `sync`, the innermost creation frame is
`sync.(*WaitGroup).Go()` rather than your code. Your frame is the next one up. This is
worth knowing before you go looking for a bug inside the standard library.

## It only finds races that actually happen

This is the caveat that decides how much the tool is worth to you.

```go
package main

import (
  "fmt"
  "os"
  "sync"
)

var shared int

func mightRace(parallel bool) {
  if !parallel {
    shared++
    return
  }
  var wg sync.WaitGroup
  for range 2 {
    wg.Go(func() { shared++ })
  }
  wg.Wait()
}

func main() {
  parallel := len(os.Args) > 1 && os.Args[1] == "parallel"
  mightRace(parallel)
  fmt.Println("shared:", shared, "parallel:", parallel)
}
```

`go run -race .` — the sequential path:

```
shared: 1 parallel: false
```

Clean. No report, exit status 0. The racy code is *compiled and instrumented*, it was
simply never executed concurrently. Now `go run -race . parallel`:

```
==================
WARNING: DATA RACE
Read at 0x00010467e450 by goroutine 7:
  main.mightRace.func1()
      /tmp/scratch/main.go:18 +0x28

Previous write at 0x00010467e450 by goroutine 8:
  main.mightRace.func1()
      /tmp/scratch/main.go:18 +0x40

Goroutine 7 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c
  main.main()
      /tmp/scratch/main.go:25 +0x328

Goroutine 8 (finished) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c
  main.main()
      /tmp/scratch/main.go:25 +0x328
==================
shared: 2 parallel: true
Found 1 data race(s)
exit status 66
```

Same binary, same instrumentation, one argument different. The first run is a
false *negative* and the detector has no way to know it. Two practical consequences:

- **A clean `-race` run does not prove the absence of races.** It is evidence about the
  paths you ran, nothing more. Treat "we run `-race` in CI" as a smoke detector, not a
  proof.
- **Coverage of concurrent paths is what buys you detection.** A test that calls your
  API from one goroutine finds nothing. Tests worth running under `-race` spawn several
  goroutines, use `t.Parallel()`, and run the real concurrent entry points. `-count=10`
  and `-cpu=1,4,12` shake out more interleavings.

Even when the racy code *does* run concurrently, detection is not guaranteed: the runtime
remembers only the last few accesses per word, so with enough intervening traffic a
conflicting pair can be forgotten before the second access arrives.

## Pattern 1: the unsynchronized counter

Already shown above. The fix is `atomic.Int64` ([11.4](04-atomic.md)) or a mutex:

```go
var counter atomic.Int64
var wg sync.WaitGroup
for range 2 {
  wg.Go(func() {
    for range 1000 {
      counter.Add(1)
    }
  })
}
wg.Wait()
```

## Pattern 2: append to a shared slice

The most common real-world race in Go. `append` reads the length, the capacity and the
backing array pointer, then writes them — three words, no atomicity.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var results []int
  var wg sync.WaitGroup
  for i := range 8 {
    wg.Go(func() {
      results = append(results, i*i)
    })
  }
  wg.Wait()
  fmt.Println("len:", len(results))
}
```

`go run -race .` reports two distinct races here (one on the slice header, one on the
array the `growslice` path touches). Truncated to the first report; the second has the
same shape with `runtime.growslice` in the stack:

```
==================
WARNING: DATA RACE
Write at 0x00c000010018 by goroutine 7:
  main.main.func1()
      /tmp/scratch/main.go:13 +0x98

Previous read at 0x00c000010018 by goroutine 9:
  main.main.func1()
      /tmp/scratch/main.go:13 +0x30

Goroutine 7 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c

Goroutine 9 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c
==================
```

Besides the race, the result is *wrong*: elements get lost when two goroutines write the
same index of the same backing array (see [7.4](../07-slices-maps/04-shared-backing.md)
for why).

Two fixes. Preallocate and write by index, which needs no synchronization at all because
each goroutine owns a distinct element:

```go
results := make([]int, 8)
var wg sync.WaitGroup
for i := range 8 {
  wg.Go(func() { results[i] = i * i })
}
wg.Wait()
```

Or append under a mutex, when the count is not known up front. Prefer the first: it is
faster, deterministic in order, and lock-free.

## Pattern 3: map read + write

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  m := map[string]int{"a": 1}
  var wg sync.WaitGroup
  wg.Go(func() {
    for i := range 10000 {
      m[fmt.Sprint("k", i)] = i
    }
  })
  wg.Go(func() {
    for range 10000 {
      _ = m["a"]
    }
  })
  wg.Wait()
  fmt.Println("done")
}
```

`go run -race .`:

```
==================
WARNING: DATA RACE
Write at 0x00c0001a20f0 by goroutine 7:
  runtime.mapaccess2_faststr()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/internal/runtime/maps/runtime_faststr.go:120 +0x2bc
  main.main.func1()
      /tmp/scratch/main.go:13 +0xa4

Previous read at 0x00c0001a20f0 by goroutine 8:
  runtime.mapaccess1_faststr()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/internal/runtime/maps/runtime_faststr.go:114 +0x4c
  runtime.mapaccess1_faststr()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/internal/runtime/maps/runtime_faststr.go:115 +0x1c

Goroutine 7 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c

Goroutine 8 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c
```

Note the top frames are inside `internal/runtime/maps` — the race is on the map's
internal bookkeeping, not on a variable you named. **Your** frame is the second one
(`main.go:13`). This is the normal shape for map, slice and interface races: skip the
`runtime.*` frames and read the first `main.*` or package-level frame.

Fix: a mutex around both operations, `sync.RWMutex` if reads dominate
([11.2](02-mutex.md)), or `sync.Map` for the two patterns it is good at
([11.3](03-once-pool-map.md)).

## `-race` versus `concurrent map writes`

These are different mechanisms and it is worth knowing which one you are looking at. The
map implementation has its own cheap, always-on consistency check, independent of
`-race`.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  m := map[int]int{}
  var wg sync.WaitGroup
  for g := range 4 {
    wg.Go(func() {
      for i := range 10000 {
        m[g*10000+i] = i
      }
    })
  }
  wg.Wait()
  fmt.Println("len:", len(m))
}
```

Run **without** `-race` — `go run .`. This crashes on purpose; the goroutine dump is
truncated to the first goroutine (the real output lists all of them):

```
fatal error: concurrent map writes

goroutine 20 [running]:
internal/runtime/maps.fatal({0x102c6313b?, 0x0?})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/panic.go:1195 +0x20
main.main.func1()
	/tmp/scratch/main.go:14 +0x44
sync.(*WaitGroup).Go.func1()
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:258 +0x48
created by sync.(*WaitGroup).Go in goroutine 1
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x78
exit status 2
```

The differences:

| | `fatal error: concurrent map writes` | `WARNING: DATA RACE` |
|---|---|---|
| Needs `-race` | no — always on | yes |
| Mechanism | a flag the map sets while writing | happens-before tracking over all memory |
| Covers | maps only | every shared memory access |
| On detection | **kills the process** immediately | prints a report, program continues |
| Recoverable | no — `fatal error`, not a panic; `recover` cannot catch it | n/a |
| Exit status | 2 | 66 |
| Reliability | best-effort; often misses | confirmed, no false positives |

So in production without `-race` you may still get `concurrent map writes` (or
`concurrent map read and map write`, or `concurrent map iteration and map write`) — a
hard crash with no recovery path. That is a feature: a corrupted map is worse than a dead
process. It is also best-effort: the same program can run fine for hours.

Under `-race`, the detector usually reports the race *before* the map's own check trips,
because it is watching every access rather than only the write window.

## Pattern 4: lazy init without Once

```go
package main

import (
  "fmt"
  "sync"
)

type Registry struct{ Name string }

var registry *Registry

func Get() *Registry {
  if registry == nil {
    registry = &Registry{Name: "default"}
  }
  return registry
}

func main() {
  var wg sync.WaitGroup
  for range 4 {
    wg.Go(func() {
      for range 1000 {
        _ = Get().Name
      }
    })
  }
  wg.Wait()
  fmt.Println("name:", Get().Name)
}
```

`go run -race .`:

```
==================
WARNING: DATA RACE
Write at 0x00010097f840 by goroutine 10:
  main.Get()
      /tmp/scratch/main.go:14 +0xbc
  main.main.func1()
      /tmp/scratch/main.go:24 +0x28

Previous read at 0x00010097f840 by goroutine 7:
  main.Get()
      /tmp/scratch/main.go:13 +0x60
  main.main.func1()
      /tmp/scratch/main.go:24 +0x28

Goroutine 10 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c

Goroutine 7 (finished) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c
==================
```

Read line 13 (`if registry == nil`) against write line 14 (`registry = ...`) — the
check-then-act is the whole bug. Beyond the race, a second goroutine can observe a
non-nil pointer to a struct whose fields are not yet visible, and return a half-built
object. Fix with `sync.Once` ([11.3](03-once-pool-map.md)):

```go
var (
  regOnce sync.Once
  reg     *Registry
)

func Get() *Registry {
  regOnce.Do(func() { reg = &Registry{Name: "default"} })
  return reg
}
```

## Pattern 5: two goroutines writing one struct field

A subtlety worth getting right: writing **different** fields of a struct from different
goroutines is *not* a race — distinct fields are distinct memory. The race appears when
two methods happen to touch the same field.

```go
package main

import (
  "fmt"
  "sync"
)

type Stats struct {
  Requests int
  Errors   int
}

func (s *Stats) recordOK() { s.Requests++ }

func (s *Stats) recordErr() {
  s.Requests++
  s.Errors++
}

func main() {
  var s Stats
  var wg sync.WaitGroup
  wg.Go(func() {
    for range 5000 {
      s.recordOK()
    }
  })
  wg.Go(func() {
    for range 5000 {
      s.recordErr()
    }
  })
  wg.Wait()
  fmt.Println("requests:", s.Requests, "errors:", s.Errors)
}
```

`recordOK` and `recordErr` both write `s.Requests`. `go run -race .` reports two races
(read-vs-write and write-vs-write on the same address); the first:

```
==================
WARNING: DATA RACE
Read at 0x00c00019e030 by goroutine 8:
  main.(*Stats).recordErr()
      /tmp/scratch/main.go:16 +0x48
  main.main.func2()
      /tmp/scratch/main.go:30 +0x44

Previous write at 0x00c00019e030 by goroutine 7:
  main.(*Stats).recordOK()
      /tmp/scratch/main.go:13 +0x50
  main.main.func1()
      /tmp/scratch/main.go:25 +0x3c

Goroutine 8 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c

Goroutine 7 (finished) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c
==================
```

This is the report shape you most want to be able to read: two *different* functions,
one shared address. The fix puts the mutex in the struct and takes it in every method —
and because `Stats` has an invariant across two fields (`Errors <= Requests`), a mutex is
required rather than two atomics ([11.4](04-atomic.md)):

```go
type Stats struct {
  mu       sync.Mutex
  Requests int
  Errors   int
}

func (s *Stats) recordOK() {
  s.mu.Lock()
  defer s.mu.Unlock()
  s.Requests++
}

func (s *Stats) recordErr() {
  s.mu.Lock()
  defer s.mu.Unlock()
  s.Requests++
  s.Errors++
}
```

## All five fixes, verified clean

All five fixes above were also run together in one program under `go run -race .`: no
report, exit status 0, and every value deterministic — the counter exactly 2000, the
indexed slice `[0 1 4 9 16 25 36 49]` in order without sorting, the map at 1001 entries,
and `Requests: 10000 Errors: 5000`.

## The pre-1.22 loop variable, made racy on purpose

Before Go 1.22, `for i := 0; i < n; i++` created **one** `i` for the whole loop, so a
goroutine closing over it read a variable the loop body kept writing — a genuine data
race, not merely a surprising value. Go 1.22 gives each iteration its own variable, and
the behaviour is selected by the `go` line in `go.mod`, which makes this the cleanest
possible demonstration: the same source file, two modules.

`go.mod`:

```
module example.com/scratch

go 1.21
```

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var wg sync.WaitGroup
  seen := make([]int, 0, 5)
  var mu sync.Mutex
  for i := 0; i < 5; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      mu.Lock()
      seen = append(seen, i)
      mu.Unlock()
    }()
  }
  wg.Wait()
  fmt.Println(len(seen))
}
```

Note the `append` is correctly locked — the only unsynchronized thing left is `i` itself.
`go run -race .` with `go 1.21`:

```
==================
WARNING: DATA RACE
Read at 0x00c00011c040 by goroutine 7:
  main.main.func1()
      /tmp/scratch/main.go:17 +0xc8

Previous write at 0x00c00011c040 by main goroutine:
  main.main()
      /tmp/scratch/main.go:12 +0x100

Goroutine 7 (running) created at:
  main.main()
      /tmp/scratch/main.go:14 +0xe8
==================
5
Found 1 data race(s)
exit status 66
```

Read it: the write is at line 12, which is the `for` statement itself (`i++`), by the
**main goroutine** — the only report in this lesson where one side is `main` rather than
a spawned goroutine. There is a single "created at" stack because the other party is
main, which was never created by anything.

Change nothing but the `go.mod` line to `go 1.22` (or later) and the same file is clean:

```
5
```

Which is why "upgrade the `go` directive" is a real bug fix and not just housekeeping,
and why a module still pinned below 1.22 should be read with this in mind.

## GORACE options and exit status

The `GORACE` environment variable configures the runtime, as `option=value`
space-separated pairs. The two worth remembering: `halt_on_error=1` stops the process at
the first report (the default is to keep going and collect more), and `log_path=<prefix>`
writes reports to `<prefix>.<pid>` instead of stderr. Others include `exitcode=N` (the
default 66), `strip_path_prefix`, `history_size` (more interleaving history, more
memory), and `atexit_sleep_ms`.

Verified exit statuses on this toolchain:

| Command | Exit status |
|---|---|
| racy program under `-race` | 66 |
| racy program under `-race` with `GORACE=exitcode=7` | 7 |
| clean program under `-race` | 0 |
| `concurrent map writes` fatal error (no `-race`) | 2 |
| `go test -race` on a racy package | 1 (`FAIL`) |

66 is deliberately not 1, so a wrapper script can distinguish "the program failed" from
"the program raced".

## In CI, and why you cannot ship it

The standard gate is one line:

```bash
go test -race ./...
```

Add `-count=1` to defeat the test cache, and consider a nightly `-count=10
-cpu=1,4,8,12` run to shake out interleavings a single pass misses. Wiring this into
GitHub Actions is [18.9](../18-production/09-ci.md).

Why `-race` stays out of production builds:

- **Speed**: 3.7-8.2x slower on the loops measured above; 5-10x is the usual range.
- **Memory**: 5-10x the program's footprint in shadow state. A service that fits in 2GB
  will not fit in 2GB.
- **Hard limits**: 8128 goroutines tracked, and only the last four accesses per word, so
  a large production workload degrades the detector's own accuracy.
- **Requires cgo** and only supports a subset of platforms, so a `CGO_ENABLED=0` static
  binary cannot have it.
- **It changes timing**, which can mask the very race you are chasing.

The one legitimate exception is a dedicated canary instance taking a small share of real
traffic, specifically to find races that your tests do not reach. Budget the capacity
before you try it.

## Quick Reference

| Command | Effect |
|---|---|
| `go run -race .` | run with the detector; report to stderr, exit 66 if any |
| `go test -race ./...` | the standard CI gate; a race fails the test |
| `go build -race -o app .` | a race-instrumented binary |
| `go test -race -count=10 ./...` | ten passes, more interleavings |
| `go test -race -cpu=1,4,12 ./...` | vary `GOMAXPROCS` to vary scheduling |

| `GORACE` option | Effect |
|---|---|
| `halt_on_error=1` | exit at the first report |
| `log_path=/tmp/race` | write to `/tmp/race.<pid>` instead of stderr |
| `exitcode=N` | exit status on detection (default 66) |
| `history_size=N` | more per-goroutine access history (0-7, default 1), more memory |
| `strip_path_prefix=/src/` | shorten paths in reports |
| `atexit_sleep_ms=N` | delay at exit so in-flight goroutines get reported |

| Report line | What it tells you |
|---|---|
| `WARNING: DATA RACE` | a confirmed race, not a heuristic |
| `Read/Write at 0x...` | the access kind and the shared address |
| `Previous read/write at 0x...` | the earlier conflicting access at the same address |
| `(running)` / `(finished)` | the goroutine's state when the report was printed |
| `Goroutine N created at:` | where each party was spawned — usually where the fix goes |
| `Found N data race(s)` | the count, printed at exit |

| Racy pattern | Fix |
|---|---|
| `counter++` from several goroutines | [`atomic.Int64`](04-atomic.md) or a mutex |
| `append` to a shared slice | preallocate and index, or append under a mutex |
| map read + write | [mutex](02-mutex.md) or [`sync.Map`](03-once-pool-map.md) |
| `if x == nil { x = ... }` | [`sync.Once`](03-once-pool-map.md) |
| two methods writing one field | a mutex *inside* the struct |
| closing over a pre-1.22 loop variable | raise the `go` directive to 1.22+ |

## Common Mistakes

**Treating a clean `-race` run as proof there are no races.** It only covers the
interleavings that executed; write tests that actually run your concurrent entry points
in parallel.

**Believing a correct result means there is no race.** The racy counter in this lesson
printed exactly 2000 while racing; correctness of output is not evidence.

**Reading the top stack frame when it is inside `runtime`.** For map, slice and
interface races the first frames are runtime internals — the first `main.*` or
package-level frame is your bug.

**Shipping a `-race` binary to production.** 5-10x slower, 5-10x the memory, and it
tracks at most 8128 goroutines; use a dedicated canary if you need it in a real
environment.

**Expecting `recover` to handle `concurrent map writes`.** It is a `fatal error`, not a
panic, and nothing can catch it — the process dies.

**Running `-race` only in CI, never locally.** The feedback loop is the point; `go test
-race ./...` on the package you are editing costs seconds.

## Exercises

1. Write a racy LRU cache (map plus a slice for recency, no locks) and a test that is
   clean under `-race` with one goroutine. Then add the minimum concurrency to the test
   that makes the detector fire, and report exactly what you had to change.
2. Take the pre-1.22 loop-variable module in this lesson, keep `go 1.21`, and fix the
   race three ways: `i := i` inside the loop, passing `i` as an argument, and raising the
   `go` directive. Verify all three under `-race`.
3. Run a racy program with `GORACE="halt_on_error=1 log_path=/tmp/r"` and with
   `history_size=7`. Describe what changed in the reports and in the process's peak RSS.
4. Build the same service twice, with and without `-race`, and measure p99 latency and
   RSS under a fixed load. Produce the number you would quote to justify not shipping it.
5. Construct a program with a genuine data race that `-race` *misses* when run normally
   but catches under `-count=20 -cpu=1,12`. Explain which mechanism was responsible for
   the miss.

---

[← 11.4 sync/atomic](04-atomic.md) · [Course Index](../README.md) · [11.6 context: Cancellation and Timeouts →](06-context-cancel.md)
