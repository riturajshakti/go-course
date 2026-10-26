[← 11.2 sync.Mutex and sync.RWMutex](02-mutex.md) · [Course Index](../README.md) · [11.4 sync/atomic →](04-atomic.md)

# 11.3 — sync.Once, sync.Pool, and sync.Map

## What You'll Learn

- `Once.Do` runs the body exactly once and **blocks** every other caller until it finishes
- A panic inside `Do` still marks the `Once` as done — the body never gets a second chance
- `sync.Pool` reduces garbage, not object lifetime: anything in it can vanish at any GC
- `sync.Map` wins on disjoint-key writes and read-mostly caches, and loses on one hot key

## Coming From Another Language

| Other | Go |
|---|---|
| JS module top-level code runs once on import | `sync.Once` for lazy init; a package `var` for eager |
| Python `functools.lru_cache` on a no-arg function | `sync.OnceValue(f)` |
| Rust `OnceLock` / `LazyLock` | `sync.Once` + a var, or `sync.OnceValue` |
| Rust's `Once` poisons on panic | Go's does *not* poison — it just marks done, silently |
| JS object pools are a manual array | `sync.Pool`, but the GC may empty it at any moment |
| `ConcurrentHashMap` (Java) | `sync.Map`, but untyped (`any` keys and values) |

Three specialised types, each with a sharp edge. `Once` is the only one you will reach
for often. `Pool` is a GC-pressure tool whose contents are allowed to disappear between
any two statements, which rules out every use that needs an object to *stay* pooled.
`sync.Map` is explicitly described by its own documentation as "specialized" — the
default remains a plain map plus a mutex from [11.2](02-mutex.md), and you switch only
after measuring.

## sync.Once: exactly one initialization

`Do` takes a `func()` and runs it on the first call only. The zero value is ready.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
)

type Config struct {
  Region string
}

var (
  once   sync.Once
  cfg    *Config
  initN  atomic.Int64
  callsN atomic.Int64
)

func load() *Config {
  callsN.Add(1)
  once.Do(func() {
    initN.Add(1)
    cfg = &Config{Region: "eu-west-1"}
  })
  return cfg
}

func main() {
  var wg sync.WaitGroup
  for range 100 {
    wg.Go(func() {
      c := load()
      if c == nil || c.Region != "eu-west-1" {
        panic("saw a partially initialized config")
      }
    })
  }
  wg.Wait()
  fmt.Println("load called:", callsN.Load())
  fmt.Println("init ran:   ", initN.Load())
  fmt.Println("region:     ", cfg.Region)
}
```

Output:

```
load called: 100
init ran:    1
region:      eu-west-1
```

Deterministic by construction: 100 goroutines, one initialization. The `panic` guard
never fires because `Do` establishes a happens-before edge — the return from the body
synchronizes before the return from *every* `Do` call, so no goroutine can observe a
half-built `Config`.

## Do blocks until the first call completes

This is the part people miss. The 99 goroutines that did not win the race do not skip
`Do` — they wait for the winner's body to finish.

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

func main() {
  var once sync.Once
  start := time.Now()
  var wg sync.WaitGroup

  for i := range 3 {
    wg.Go(func() {
      once.Do(func() {
        time.Sleep(200 * time.Millisecond)
      })
      fmt.Printf("goroutine %d returned from Do after %v\n", i, time.Since(start).Round(50*time.Millisecond))
    })
    time.Sleep(10 * time.Millisecond)
  }
  wg.Wait()
}
```

The three lines can print in any order; sorted, the output is:

```
goroutine 0 returned from Do after 200ms
goroutine 1 returned from Do after 200ms
goroutine 2 returned from Do after 200ms
```

All three return at 200ms even though goroutines 1 and 2 arrived 10ms and 20ms late.
That is the guarantee you want — nobody proceeds with uninitialized state — but it also
means a slow `Do` body stalls every caller. Opening a database connection inside `Do`
blocks every request that touches it until the dial completes.

## A panic inside Do still marks it done

Go's `Once` does not poison and does not retry. The body ran, so it is finished,
successful or not.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var once sync.Once
  attempts := 0

  func() {
    defer func() { fmt.Println("recovered:", recover()) }()
    once.Do(func() {
      attempts++
      panic("config file missing")
    })
  }()

  once.Do(func() {
    attempts++
    fmt.Println("second Do body ran")
  })

  fmt.Println("attempts:", attempts)
}
```

Output:

```
recovered: config file missing
attempts: 1
```

The second `Do` is a no-op: `second Do body ran` never prints, and `attempts` stays at 1.
Every later caller sails straight through into whatever broken state the panic left
behind. If initialization can fail, store the error and check it:

```go
var (
  once    sync.Once
  conn    *Conn
  connErr error
)

func Get() (*Conn, error) {
  once.Do(func() { conn, connErr = dial() })
  return conn, connErr
}
```

That returns the same error forever, which is usually what you want for a fatal config
problem. If you need retries, `Once` is the wrong primitive — use a mutex and a `done`
flag you can reset.

## OnceFunc, OnceValue, OnceValues

Go 1.21 added three wrappers that remove the package-level variable plumbing. All three
exist in 1.27.

```go
package main

import (
  "errors"
  "fmt"
  "sync"
  "sync/atomic"
)

func main() {
  var runs atomic.Int64

  greet := sync.OnceFunc(func() {
    runs.Add(1)
    fmt.Println("greet body ran")
  })
  greet()
  greet()
  greet()

  port := sync.OnceValue(func() int {
    runs.Add(1)
    return 8080
  })
  fmt.Println("port:", port(), port(), port())

  parse := sync.OnceValues(func() (string, error) {
    runs.Add(1)
    return "", errors.New("bad DSN")
  })
  v1, e1 := parse()
  v2, e2 := parse()
  fmt.Printf("parse: %q %v / %q %v\n", v1, e1, v2, e2)
  fmt.Println("same error value returned twice:", errors.Is(e1, e2))

  fmt.Println("total bodies run:", runs.Load())
}
```

Output:

```
greet body ran
port: 8080 8080 8080
parse: "" bad DSN / "" bad DSN
same error value returned twice: true
total bodies run: 3
```

`OnceValues` is the direct answer to the fallible-init problem above: the `(T, error)`
pair is captured once and handed back to every caller, same error identity included.
These wrappers *do* re-panic the original value on every subsequent call, unlike bare
`Once.Do` — a meaningful difference if you rely on panics propagating.

Idiomatic: `sync.OnceValue` / `sync.OnceValues` for new code. Bare `sync.Once` when the
init has to assign several unrelated package variables, or when the `Once` lives inside
a struct.

## Lazy singleton vs a package-level var

```go
var defaultRegistry = newRegistry()

var (
  registryOnce sync.Once
  registry     *Registry
)

func DefaultRegistry() *Registry {
  registryOnce.Do(func() { registry = newRegistry() })
  return registry
}
```

Use the plain `var` when the initializer is cheap, cannot fail, and has no ordering
dependency on other packages; use `Once` when it is expensive, may fail, touches the
network or filesystem, or must not run in programs that never call it (an `init`-time
dial makes every test that imports your package slow).

## sync.Pool: Get, Put, New

`Get` returns something from the pool, or calls `New` if the pool is empty. There is no
size limit, no `Len`, and no way to inspect the contents.

```go
package main

import (
  "fmt"
  "runtime"
  "sync"
)

func main() {
  created := 0
  pool := sync.Pool{
    New: func() any {
      created++
      return new([]byte)
    },
  }

  a := pool.Get()
  fmt.Println("after first Get, New calls:", created)
  pool.Put(a)

  b := pool.Get()
  fmt.Println("recycled the same object:", a == b, "New calls:", created)
  pool.Put(b)

  runtime.GC()
  c := pool.Get()
  fmt.Println("after runtime.GC, got the old object:", c == b, "New calls:", created)

  pool.Put(c)
  runtime.GC()
  runtime.GC()
  d := pool.Get()
  fmt.Println("after two GCs, got the old object:", d == c, "New calls:", created)

  fmt.Println("Get on a pool with no New:", new(sync.Pool).Get())
}
```

Output:

```
after first Get, New calls: 1
recycled the same object: true New calls: 1
after runtime.GC, got the old object: false New calls: 2
after two GCs, got the old object: false New calls: 3
Get on a pool with no New: <nil>
```

Three facts in one run. `Get` after a matching `Put` really did hand back the identical
pointer. A single `runtime.GC()` was enough to empty the pool — on this toolchain the
collector clears pooled objects on *any* GC, so do not count on the "survives one cycle"
folklore. And with no `New` set, `Get` returns a nil `any`, which means a type assertion
like `pool.Get().(*bytes.Buffer)` panics. Always set `New`.

Because eviction is invisible and unbounded, `sync.Pool` is **not** an object cache and
not a connection pool: a pooled database connection would be silently dropped mid-GC
without being closed. For connections use a real pool with lifetime management
(`database/sql`'s, or pgx's — [17.5](../17-database/05-pooling.md)).

## You must reset a pooled object

`Get` may hand you an object with another request's data still in it. Nothing resets it
for you.

```go
package main

import (
  "bytes"
  "fmt"
  "sync"
)

var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func renderBroken(name string) string {
  b := bufPool.Get().(*bytes.Buffer)
  defer bufPool.Put(b)
  fmt.Fprintf(b, "hello %s", name)
  return b.String()
}

func render(name string) string {
  b := bufPool.Get().(*bytes.Buffer)
  b.Reset()
  defer bufPool.Put(b)
  fmt.Fprintf(b, "hello %s", name)
  return b.String()
}

func main() {
  fmt.Println(renderBroken("ada"))
  fmt.Println(renderBroken("bob"))
  fmt.Println(renderBroken("cyd"))

  fmt.Println("--- with Reset ---")
  fmt.Println(render("ada"))
  fmt.Println(render("bob"))
  fmt.Println(render("cyd"))
}
```

Output:

```
hello ada
hello adahello bob
hello adahello bobhello cyd
--- with Reset ---
hello ada
hello bob
hello cyd
```

In a web handler that bug leaks one user's response into the next user's. Reset
immediately after `Get`, not before `Put` — resetting before `Put` looks equivalent but
breaks the moment any code path returns without reaching the `Put`.

## The benchmark: does pooling pay?

A 20-line report rendered into a `bytes.Buffer`, with and without the pool.

```go
package main

import (
  "bytes"
  "fmt"
  "sync"
  "testing"
)

var pool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func BenchmarkNoPool(b *testing.B) {
  b.ReportAllocs()
  for b.Loop() {
    buf := new(bytes.Buffer)
    for i := range 20 {
      fmt.Fprintf(buf, "row %d\n", i)
    }
    _ = buf.Len()
  }
}

func BenchmarkPool(b *testing.B) {
  b.ReportAllocs()
  for b.Loop() {
    buf := pool.Get().(*bytes.Buffer)
    buf.Reset()
    for i := range 20 {
      fmt.Fprintf(buf, "row %d\n", i)
    }
    _ = buf.Len()
    pool.Put(buf)
  }
}
```

`go test -bench . -benchmem` on this machine (darwin/arm64, Apple M4 Pro, 12 threads) —
absolute numbers are machine-specific, the allocation column is the point:

```
BenchmarkNoPool-12    	 1838144	       628.4 ns/op	     496 B/op	       4 allocs/op
BenchmarkPool-12      	 2226216	       538.0 ns/op	       0 B/op	       0 allocs/op
```

**4 allocs/op to 0**, and 496 bytes of garbage per operation to zero. The time saving is
only 14% — the real win is the garbage the collector never has to scan. That is also the
test for whether a pool is worth it: if a profile shows GC time, pool the big short-lived
buffers; if it does not, a pool is complexity for nothing. The standard library uses one
inside `fmt` for exactly this reason.

## sync.Map: every method

The zero value is ready. Keys and values are `any`.

```go
package main

import (
  "fmt"
  "slices"
  "sync"
)

func main() {
  var m sync.Map

  m.Store("a", 1)
  m.Store("b", 2)

  v, ok := m.Load("a")
  fmt.Println("Load a:", v, ok)
  v, ok = m.Load("zz")
  fmt.Println("Load zz:", v, ok)

  actual, loaded := m.LoadOrStore("b", 99)
  fmt.Println("LoadOrStore b:", actual, loaded)
  actual, loaded = m.LoadOrStore("c", 3)
  fmt.Println("LoadOrStore c:", actual, loaded)

  prev, loaded := m.Swap("c", 33)
  fmt.Println("Swap c:", prev, loaded)
  prev, loaded = m.Swap("d", 4)
  fmt.Println("Swap d:", prev, loaded)

  fmt.Println("CompareAndSwap d 4->44:", m.CompareAndSwap("d", 4, 44))
  fmt.Println("CompareAndSwap d 4->44 again:", m.CompareAndSwap("d", 4, 44))

  v, loaded = m.LoadAndDelete("a")
  fmt.Println("LoadAndDelete a:", v, loaded)
  v, loaded = m.LoadAndDelete("a")
  fmt.Println("LoadAndDelete a again:", v, loaded)

  fmt.Println("CompareAndDelete b 2:", m.CompareAndDelete("b", 2))
  m.Delete("c")

  keys := []string{}
  m.Range(func(k, v any) bool {
    keys = append(keys, fmt.Sprintf("%v=%v", k, v))
    return true
  })
  slices.Sort(keys)
  fmt.Println("remaining:", keys)

  m.Clear()
  n := 0
  m.Range(func(k, v any) bool { n++; return true })
  fmt.Println("after Clear, entries:", n)
}
```

Output:

```
Load a: 1 true
Load zz: <nil> false
LoadOrStore b: 2 true
LoadOrStore c: 3 false
Swap c: 3 true
Swap d: <nil> false
CompareAndSwap d 4->44: true
CompareAndSwap d 4->44 again: false
LoadAndDelete a: 1 true
LoadAndDelete a again: <nil> false
CompareAndDelete b 2: true
remaining: [d=44]
after Clear, entries: 0
```

`Clear` (Go 1.23) and `CompareAndDelete` (1.20) both exist in 1.27. Note the missing-key
result: `nil` and `false`, so you always need the comma-ok form — unlike a plain map
there is no typed zero value to fall back on. `LoadOrStore`'s second return is `loaded`,
true when the key was already present, which makes it the atomic "insert if absent"
you would otherwise write with a mutex and a double check.

## Range is a snapshot, and may miss concurrent writes

`Range` stops when the callback returns false. It does not take a global lock, so it is
explicitly allowed to miss entries stored while it is running.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var m sync.Map
  for i := range 100 {
    m.Store(i, i*i)
  }

  seen := 0
  m.Range(func(k, v any) bool {
    seen++
    return seen < 10
  })
  fmt.Println("stopped after:", seen)

  total := 0
  m.Range(func(k, v any) bool {
    total++
    return true
  })
  fmt.Println("full Range saw:", total)

  var wg sync.WaitGroup
  var n sync.Map
  n.Store("seed", 0)
  wg.Go(func() {
    for i := range 1000 {
      n.Store(fmt.Sprint("k", i), i)
    }
  })
  counted := 0
  n.Range(func(k, v any) bool { counted++; return true })
  wg.Wait()

  after := 0
  n.Range(func(k, v any) bool { after++; return true })
  fmt.Println("Range during concurrent writes saw at least 1:", counted >= 1)
  fmt.Println("Range after Wait saw:", after)
}
```

Output:

```
stopped after: 10
full Range saw: 100
Range during concurrent writes saw at least 1: true
Range after Wait saw: 1001
```

The concurrent-`Range` count is deliberately not printed: it lands anywhere from 1 to
1001 depending on scheduling, which is exactly the documented contract. The final count
is a stable 1001 because the writer has finished. Clean under `go run -race .`.

Two consequences. There is no `Len`: counting means a full `Range`, which is O(n) and
still only approximate under concurrent writes. And `Range` is the only way to enumerate,
so any code that needs a consistent view of all entries wants a mutex-guarded plain map
instead.

## When sync.Map wins, and when it loses

The documentation names two cases: write-once-read-many, and goroutines working on
**disjoint** sets of keys. Here are all three shapes measured against an
`RWMutex`+`map[int]int` wrapper, `b.RunParallel` on 12 threads.

```
BenchmarkRWMutexDisjointWrite-12    	 3743252	       356.4 ns/op	      69 B/op	       0 allocs/op
BenchmarkSyncMapDisjointWrite-12    	30144188	        47.18 ns/op	      87 B/op	       3 allocs/op
BenchmarkRWMutexReadMostly-12       	 9808879	       122.4 ns/op	       0 B/op	       0 allocs/op
BenchmarkSyncMapReadMostly-12       	1000000000	         1.142 ns/op	       0 B/op	       0 allocs/op
BenchmarkRWMutexSingleHotKey-12     	24357836	        50.36 ns/op	       0 B/op	       0 allocs/op
BenchmarkSyncMapSingleHotKey-12     	14533812	        76.67 ns/op	      28 B/op	       0 allocs/op
```

Read the three pairs:

- **Disjoint keys, append-mostly: 7.6x faster.** Every `RWMutex` write serialises on one
  lock; `sync.Map` shards so the writers barely interact.
- **Read-mostly over 1000 stable keys: 107x faster** — 1.1ns against 122ns. Reads hit a
  lock-free read-only map with no atomic write at all, so they scale perfectly across
  cores while `RLock` still has to touch a shared reader counter.
- **One hot key, 50/50 read/write: 1.5x *slower*.** Every write has to promote the entry
  out of the read-only map, so `sync.Map` pays its bookkeeping and gets no sharding
  benefit. It also allocates 28 B/op where the mutex version allocates nothing.

Idiomatic: start with `map` + `sync.Mutex`. Reach for `sync.Map` only for a cache that
mostly grows and is mostly read, or for genuinely partitioned keys — and confirm with a
benchmark on your own access pattern, because the hot-key case is a real regression.

## You lose type safety

`sync.Map` predates generics and was never updated, so every `Load` returns `any` and
every mistake is a runtime panic rather than a compile error. A thin generic wrapper
fixes that:

```go
package main

import (
  "fmt"
  "slices"
  "sync"
)

type TypedMap[K comparable, V any] struct {
  m sync.Map
}

func (t *TypedMap[K, V]) Store(k K, v V) { t.m.Store(k, v) }

func (t *TypedMap[K, V]) Load(k K) (V, bool) {
  v, ok := t.m.Load(k)
  if !ok {
    var zero V
    return zero, false
  }
  return v.(V), true
}

func (t *TypedMap[K, V]) LoadOrStore(k K, v V) (V, bool) {
  actual, loaded := t.m.LoadOrStore(k, v)
  return actual.(V), loaded
}

func (t *TypedMap[K, V]) Delete(k K) { t.m.Delete(k) }

func (t *TypedMap[K, V]) Range(f func(K, V) bool) {
  t.m.Range(func(k, v any) bool { return f(k.(K), v.(V)) })
}

func main() {
  var sessions TypedMap[string, int]
  sessions.Store("alice", 1)
  sessions.Store("bob", 2)

  n, ok := sessions.Load("alice")
  fmt.Println("alice:", n, ok)
  n, ok = sessions.Load("nobody")
  fmt.Println("nobody:", n, ok)

  out := []string{}
  sessions.Range(func(k string, v int) bool {
    out = append(out, fmt.Sprintf("%s=%d", k, v))
    return true
  })
  slices.Sort(out)
  fmt.Println(out)

  var raw sync.Map
  raw.Store("count", 1)
  v, _ := raw.Load("count")
  _, isString := v.(string)
  fmt.Println("untyped Load gives any; assertion to string succeeded:", isString)
}
```

Output:

```
alice: 1 true
nobody: 0 false
[alice=1 bob=2]
untyped Load gives any; assertion to string succeeded: false
```

The wrapper restores the typed zero value on a miss (`nobody: 0 false`) and moves the
assertions into one audited place. Note `TypedMap` embeds a `sync.Map` by value, so like
`sync.Map` itself it must not be copied after first use — always pass `*TypedMap`.

## Quick Reference

| `sync.Once` | Signature | Note |
|---|---|---|
| `o.Do(f)` | `func(f func())` | runs `f` once; blocks other callers until it returns |
| `sync.OnceFunc(f)` | `func(func()) func()` | wrapper; re-panics the original value on later calls |
| `sync.OnceValue(f)` | `func(func() T) func() T` | caches one result |
| `sync.OnceValues(f)` | `func(func() (T1, T2)) func() (T1, T2)` | caches a `(value, error)` pair |

| `sync.Pool` | Signature | Note |
|---|---|---|
| `p.Get()` | `func() any` | pooled object, or `New()`, or `nil` if `New` is unset |
| `p.Put(x)` | `func(x any)` | returns an object; no size limit, no error |
| `p.New` | `func() any` field | always set it, or `Get` can return nil |

| `sync.Map` | Signature | Note |
|---|---|---|
| `m.Store(k, v)` | `func(key, value any)` | overwrites |
| `m.Load(k)` | `(value any, ok bool)` | `nil, false` when absent |
| `m.LoadOrStore(k, v)` | `(actual any, loaded bool)` | atomic insert-if-absent |
| `m.LoadAndDelete(k)` | `(value any, loaded bool)` | atomic take |
| `m.Delete(k)` | `func(key any)` | no-op if absent |
| `m.Swap(k, v)` | `(previous any, loaded bool)` | store and return the old value |
| `m.CompareAndSwap(k, old, new)` | `bool` | both values must be comparable |
| `m.CompareAndDelete(k, old)` | `bool` | Go 1.20+ |
| `m.Range(f)` | `func(f func(k, v any) bool)` | stop on false; may miss concurrent writes |
| `m.Clear()` | `func()` | Go 1.23+ |
| — | — | **no `Len`** |

| Need | Use |
|---|---|
| one-time init, may fail | `sync.OnceValues` |
| one-time init of several package vars | `sync.Once` |
| cheap, infallible init | a package-level `var` |
| cut GC pressure from big short-lived buffers | `sync.Pool` |
| keep objects alive and bounded (connections) | a real pool, not `sync.Pool` |
| read-mostly cache that only grows | `sync.Map` |
| disjoint keys per goroutine | `sync.Map` |
| anything else, including one hot key | `map` + [`sync.Mutex`](02-mutex.md) |

## Common Mistakes

**Assuming a panic inside `Once.Do` lets the next caller retry.** It does not — the
`Once` is permanently done; capture the error with `sync.OnceValues` and return it to
every caller.

**Using `sync.Pool` as a connection pool or cache.** Any GC may discard the contents
without closing or notifying anything; use a pool that manages lifetime.

**Forgetting to reset a pooled object right after `Get`.** You inherit the previous
user's bytes, which in an HTTP handler leaks one response into another.

**Calling `pool.Get().(*T)` with no `New` function set.** `Get` returns a nil `any` and
the assertion panics; always set `New`.

**Counting `sync.Map` entries in a loop as if it were a `Len`.** `Range` is O(n) and
approximate under concurrent writes; if you need an exact size, use a guarded plain map.

**Choosing `sync.Map` because the code is concurrent.** It is measurably slower on a
contended single key; benchmark against `map` + mutex before switching.

## Exercises

1. Build a `Lazy[T any]` type with a `Get() (T, error)` method over `sync.OnceValues`,
   then prove with `go test -race` that 500 concurrent `Get`s run the initializer exactly
   once and all receive the identical error value when it fails.
2. Take the `renderBroken` pool bug and write a test that catches it without reading the
   source: call it twice with different names and assert on the second result.
3. Benchmark the buffer pool with a 100-byte payload and a 1MB payload. Find where
   pooling stops paying for itself, and explain the `B/op` column in each case.
4. Implement a `Counters` type that tracks per-key hit counts, once with
   `sync.Map` + `atomic.Int64` values and once with `map[string]int` + `sync.Mutex`.
   Benchmark both with 12 goroutines hitting 10,000 distinct keys, then with all 12
   hitting one key, and report which wins where.
5. Write a `Range` that collects a `sync.Map` into a sorted `[]struct{K string; V int}`
   while another goroutine stores new keys. Run it under `-race`, confirm it is clean,
   and then explain in two sentences why a clean race report does not mean the snapshot
   is complete.

---

[← 11.2 sync.Mutex and sync.RWMutex](02-mutex.md) · [Course Index](../README.md) · [11.4 sync/atomic →](04-atomic.md)
