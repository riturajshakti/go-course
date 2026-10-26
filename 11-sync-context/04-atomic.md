[← 11.3 sync.Once, sync.Pool, and sync.Map](03-once-pool-map.md) · [Course Index](../README.md) · [11.5 The Race Detector →](05-race-detector.md)

# 11.4 — sync/atomic

## What You'll Learn

- The typed API (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`) is the idiomatic one; the bare functions are legacy
- `CompareAndSwap` in a retry loop gives you lock-free read-modify-write over arbitrary logic
- `atomic.Pointer[T]` is the right tool for hot-swapping an immutable config struct
- Atomics protect *one* word; two fields that must change together still need a mutex

## Coming From Another Language

| Other | Go |
|---|---|
| JS has no shared-memory atomics outside `Atomics` on a `SharedArrayBuffer` | `sync/atomic` on any goroutine-shared word |
| Python's GIL makes `x += 1` look atomic (it is not, under `threading`) | no GIL; `x++` on a shared `int` is a data race, full stop |
| Rust `AtomicI64` with an explicit `Ordering` | no ordering parameter — every Go atomic is sequentially consistent |
| Rust `ArcSwap` / `RwLock<Arc<T>>` | `atomic.Pointer[T]` |
| Java `AtomicInteger` | `atomic.Int32` / `atomic.Int64` |

Go gives you no memory-ordering knob. There is no `Relaxed`, no `Acquire`, no
`Release` — every operation in `sync/atomic` is sequentially consistent, which removes
the entire category of bug where a correct-looking Rust or C++ atomic is too weak for
the invariant it guards. The cost is that you cannot buy speed by relaxing ordering, and
the benefit is that an atomic in Go either synchronizes or it does not compile. The other
Go-specific thing: the useful API is a set of *types* with methods, not a set of
functions taking pointers, and the types exist specifically to make a 32-bit alignment
hazard unrepresentable.

## The typed API

The zero value of every atomic type is ready to use, and the types must not be copied
after first use (`go vet` catches copies via `copylocks`).

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
)

func main() {
  var hits atomic.Int64
  var flag atomic.Bool
  var id atomic.Uint64

  var wg sync.WaitGroup
  for range 1000 {
    wg.Go(func() {
      hits.Add(1)
      id.Add(2)
      flag.Store(true)
    })
  }
  wg.Wait()

  fmt.Println("hits:", hits.Load())
  fmt.Println("id:  ", id.Load())
  fmt.Println("flag:", flag.Load())

  fmt.Println("Swap:", hits.Swap(0), "now", hits.Load())
  hits.Store(42)
  fmt.Println("CompareAndSwap 42->43:", hits.CompareAndSwap(42, 43), hits.Load())
  fmt.Println("CompareAndSwap 42->99:", hits.CompareAndSwap(42, 99), hits.Load())
  fmt.Println("Add(-3):", hits.Add(-3))

  var zero atomic.Int32
  fmt.Println("zero value is usable:", zero.Load(), zero.Add(7))
}
```

Output:

```
hits: 1000
id:   2000
flag: true
Swap: 1000 now 0
CompareAndSwap 42->43: true 43
CompareAndSwap 42->99: false 43
Add(-3): 40
zero value is usable: 0 7
```

Deterministic by construction: 1000 goroutines each adding 1 always lands on 1000, which
is exactly the property an unsynchronized `hits++` does not have (11.5 shows that
failing). Note `Add` returns the *new* value, `Swap` the *old* one, and `CompareAndSwap`
a `bool`.

The full typed set: `atomic.Int32`, `atomic.Int64`, `atomic.Uint32`, `atomic.Uint64`,
`atomic.Uintptr`, `atomic.Bool`, `atomic.Pointer[T]`, and `atomic.Value`. `Bool` has no
`Add`; `Pointer[T]` and `Value` have no `Add` either.

## The older function API

Every typed method has an older free-function equivalent that takes a pointer to a plain
variable.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
)

func main() {
  var hits int64
  var wg sync.WaitGroup
  for range 1000 {
    wg.Go(func() { atomic.AddInt64(&hits, 1) })
  }
  wg.Wait()

  fmt.Println("LoadInt64:          ", atomic.LoadInt64(&hits))
  atomic.StoreInt64(&hits, 42)
  fmt.Println("after StoreInt64:   ", atomic.LoadInt64(&hits))
  fmt.Println("SwapInt64:          ", atomic.SwapInt64(&hits, 7))
  fmt.Println("CompareAndSwapInt64:", atomic.CompareAndSwapInt64(&hits, 7, 8))
  fmt.Println("CompareAndSwapInt64:", atomic.CompareAndSwapInt64(&hits, 7, 9))
  fmt.Println("final:              ", atomic.LoadInt64(&hits))

  var n int32
  atomic.AddInt32(&n, 5)
  var u uint64
  atomic.AddUint64(&u, 3)
  fmt.Println("int32:", atomic.LoadInt32(&n), "uint64:", atomic.LoadUint64(&u))
}
```

Output:

```
LoadInt64:           1000
after StoreInt64:    42
SwapInt64:           42
CompareAndSwapInt64: true
CompareAndSwapInt64: false
final:               8
int32: 5 uint64: 3
```

Idiomatic: use the typed API for all new code. The functions exist for pre-1.19 code and
for the rare case where you must do atomic operations on a field whose layout is fixed by
something else (a C struct, a memory-mapped region).

The reason the types were added: on 32-bit platforms (`GOARCH=386`, `arm`) a 64-bit
atomic operation requires the address to be 8-byte aligned, and the compiler only
guarantees 4-byte alignment for an `int64` that is not the first field of a struct — so
`atomic.AddInt64(&s.counter, 1)` can panic with "unaligned 64-bit atomic operation" on
those platforms while working fine on arm64. `atomic.Int64` embeds an alignment-forcing
field, so the hazard cannot occur.

## CompareAndSwap in a retry loop

`Add` only does addition. For any other read-modify-write — a maximum, a saturating
counter, a bitmask update — you load, compute, and CAS, retrying if someone else won.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
)

var retries atomic.Int64

func casIncrement(v *atomic.Int64) {
  for {
    old := v.Load()
    if v.CompareAndSwap(old, old+1) {
      return
    }
    retries.Add(1)
  }
}

func casMax(v *atomic.Int64, n int64) {
  for {
    old := v.Load()
    if old >= n {
      return
    }
    if v.CompareAndSwap(old, n) {
      return
    }
  }
}

func main() {
  var counter atomic.Int64
  var high atomic.Int64

  var wg sync.WaitGroup
  for i := range 500 {
    wg.Go(func() {
      casIncrement(&counter)
      casMax(&high, int64(i))
    })
  }
  wg.Wait()

  fmt.Println("counter:", counter.Load())
  fmt.Println("high water mark:", high.Load())
  fmt.Println("CAS retries happened:", retries.Load() > 0)
}
```

Output:

```
counter: 500
high water mark: 499
CAS retries happened: true
```

Both results are deterministic: 500 increments land on 500, and the high-water mark of
`0..499` is 499 regardless of the order the goroutines ran in. The retry count itself is
not printed because it varies per run — all that matters is that it is nonzero, proving
real contention was handled correctly.

The shape is always the same:

```go
for {
  old := v.Load()
  new := f(old)
  if v.CompareAndSwap(old, new) {
    return
  }
}
```

This is lock-free, not wait-free: a goroutine can in principle retry forever. In practice
under normal contention the loop runs once or twice. Note `casMax` has an early `return`
when no update is needed — a CAS loop that always writes is slower under contention than
one that checks first.

## atomic.Pointer[T]: lock-free config hot-swap

The textbook use. The config struct is immutable once published; a reload builds a whole
new one and swaps the pointer. Readers get a consistent snapshot with no lock at all.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
)

type Config struct {
  Endpoint string
  Retries  int
  Timeout  int
}

type Client struct {
  cfg atomic.Pointer[Config]
}

func NewClient(c *Config) *Client {
  cl := &Client{}
  cl.cfg.Store(c)
  return cl
}

func (c *Client) Config() *Config { return c.cfg.Load() }

func (c *Client) Reload(n *Config) *Config { return c.cfg.Swap(n) }

func main() {
  c := NewClient(&Config{Endpoint: "a.example", Retries: 3, Timeout: 5})

  var wg sync.WaitGroup
  var consistent atomic.Int64
  var reads atomic.Int64

  for range 200 {
    wg.Go(func() {
      for range 500 {
        cfg := c.Config()
        reads.Add(1)
        if cfg.Endpoint == "a.example" && cfg.Retries == 3 && cfg.Timeout == 5 {
          consistent.Add(1)
        } else if cfg.Endpoint == "b.example" && cfg.Retries == 9 && cfg.Timeout == 30 {
          consistent.Add(1)
        }
      }
    })
  }
  wg.Go(func() {
    for i := range 100 {
      if i%2 == 0 {
        c.Reload(&Config{Endpoint: "b.example", Retries: 9, Timeout: 30})
      } else {
        c.Reload(&Config{Endpoint: "a.example", Retries: 3, Timeout: 5})
      }
    }
  })
  wg.Wait()

  fmt.Println("reads:                ", reads.Load())
  fmt.Println("fully consistent:     ", consistent.Load())
  fmt.Println("torn reads:           ", reads.Load()-consistent.Load())
  old := c.Reload(&Config{Endpoint: "c.example", Retries: 1, Timeout: 1})
  fmt.Println("Swap returned old endpoint:", old.Endpoint)
  fmt.Println("current endpoint:          ", c.Config().Endpoint)
  fmt.Println("CompareAndSwap stale ptr:  ", c.cfg.CompareAndSwap(old, &Config{Endpoint: "d.example"}))
}
```

Output:

```
reads:                 100000
fully consistent:      100000
torn reads:            0
Swap returned old endpoint: a.example
current endpoint:           c.example
CompareAndSwap stale ptr:   false
```

100,000 reads interleaved with 100 reloads and not one torn read. The reason is that a
reader never sees a half-updated `Config`: it loads a pointer to a struct that was fully
built before `Store` published it. Compare this with mutating `cfg.Endpoint` and
`cfg.Retries` in place, where a reader can land between the two writes.

Requirements for this to be correct: nothing ever mutates a `*Config` after it is
published, and `Config` contains no pointers to mutable shared state. Violate either and
you are back to needing a lock.

The final `CompareAndSwap` returns false because `old` is no longer the current pointer —
that is the ABA-safe way to do "update the config only if nobody else changed it since I
read it".

## atomic.Value and its inconsistent-type panic

`atomic.Value` predates generics. It stores an `any`, and after the first `Store` it
refuses any value of a different dynamic type.

```go
package main

import (
  "fmt"
  "sync/atomic"
)

func main() {
  var v atomic.Value
  fmt.Println("empty Load:", v.Load())

  v.Store("eu-west-1")
  fmt.Println("Load:", v.Load().(string))

  old := v.Swap("us-east-1")
  fmt.Println("Swap returned:", old, "now:", v.Load())

  fmt.Println("CompareAndSwap:", v.CompareAndSwap("us-east-1", "ap-south-1"))
  fmt.Println("now:", v.Load())

  v.Store(42)
  fmt.Println("never reached")
}
```

This program panics on purpose. Output:

```
empty Load: <nil>
Load: eu-west-1
Swap returned: eu-west-1 now: us-east-1
CompareAndSwap: true
now: ap-south-1
panic: sync/atomic: store of inconsistently typed value into Value

goroutine 1 [running]:
sync/atomic.(*Value).Store(0x4cde0979c020, {0x10295c868, 0x1028a5558})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/atomic/value.go:78 +0xd0
main.main()
	/tmp/scratch/main.go:21 +0x308
exit status 2
```

The two hex addresses differ on every run. Three traps in one type: the empty `Load`
returns a nil `any` rather than a typed zero, every read needs a type assertion, and the
type mismatch is a runtime panic the compiler cannot see. `Store(nil)` panics too.

Idiomatic: `atomic.Pointer[T]` for anything pointer-shaped. Reach for `atomic.Value` only
when you must store a non-pointer `any` whose type is genuinely fixed — and even then a
one-field struct behind `atomic.Pointer` is usually cleaner.

## Atomic is not enough for two fields

An atomic guarantee covers one word. Two atomic writes are two separate events, and a
reader can land between them.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
)

type Account struct {
  checking atomic.Int64
  savings  atomic.Int64
}

func (a *Account) transfer(n int64) {
  a.checking.Add(-n)
  a.savings.Add(n)
}

func (a *Account) total() int64 {
  return a.checking.Load() + a.savings.Load()
}

type LockedAccount struct {
  mu       sync.Mutex
  checking int64
  savings  int64
}

func (a *LockedAccount) transfer(n int64) {
  a.mu.Lock()
  defer a.mu.Unlock()
  a.checking -= n
  a.savings += n
}

func (a *LockedAccount) total() int64 {
  a.mu.Lock()
  defer a.mu.Unlock()
  return a.checking + a.savings
}

func main() {
  a := &Account{}
  a.checking.Store(1000)

  var wg sync.WaitGroup
  var bad, samples atomic.Int64
  for range 4 {
    wg.Go(func() {
      for range 200000 {
        a.transfer(1)
        a.transfer(-1)
      }
    })
  }
  wg.Go(func() {
    for range 200000 {
      samples.Add(1)
      if a.total() != 1000 {
        bad.Add(1)
      }
    }
  })
  wg.Wait()
  fmt.Println("atomic: samples", samples.Load(), "inconsistent totals:", bad.Load() > 0)
  fmt.Println("atomic: final total at rest:", a.total())

  b := &LockedAccount{checking: 1000}
  var wg2 sync.WaitGroup
  var bad2, samples2 atomic.Int64
  for range 4 {
    wg2.Go(func() {
      for range 200000 {
        b.transfer(1)
        b.transfer(-1)
      }
    })
  }
  wg2.Go(func() {
    for range 200000 {
      samples2.Add(1)
      if b.total() != 1000 {
        bad2.Add(1)
      }
    }
  })
  wg2.Wait()
  fmt.Println("mutex:  samples", samples2.Load(), "inconsistent totals:", bad2.Load() > 0)
}
```

Output:

```
atomic: samples 200000 inconsistent totals: true
atomic: final total at rest: 1000
mutex:  samples 200000 inconsistent totals: false
```

The atomic version has no data race — `go run -race` reports nothing — and yet
`checking + savings` is observably wrong mid-flight, because `transfer` is two atomic
operations with a gap between them. The mutex version holds the lock across both writes
and across the read, so the invariant `total == 1000` is never observable as broken.

This is the dividing line. Atomics give you *race freedom*; a mutex gives you
*atomicity of a critical section*. If your invariant spans more than one word, you need
the second thing. (`inconsistent totals: true` is reproducible here but is fundamentally
a race against the scheduler; the point is only that it is *possible*, which is enough to
disqualify the design.)

## The three-way benchmark

An unsynchronized counter (wrong), an atomic counter, and a mutex counter, all under
`b.RunParallel` on 12 threads.

```go
package main

import (
  "sync"
  "sync/atomic"
  "testing"
)

var plain int64
var at atomic.Int64
var mu sync.Mutex
var guarded int64

func BenchmarkUnsynchronized(b *testing.B) {
  b.RunParallel(func(pb *testing.PB) {
    for pb.Next() {
      plain++
    }
  })
}

func BenchmarkAtomic(b *testing.B) {
  b.RunParallel(func(pb *testing.PB) {
    for pb.Next() {
      at.Add(1)
    }
  })
}

func BenchmarkMutex(b *testing.B) {
  b.RunParallel(func(pb *testing.PB) {
    for pb.Next() {
      mu.Lock()
      guarded++
      mu.Unlock()
    }
  })
}
```

`go test -bench . -benchmem` on this machine (darwin/arm64, Apple M4 Pro, 12 threads) —
absolute numbers are machine-specific, the ratios are the point:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkUnsynchronized-12    	1000000000	         0.2082 ns/op	       0 B/op	       0 allocs/op
BenchmarkAtomic-12            	20746320	        59.36 ns/op	       0 B/op	       0 allocs/op
BenchmarkMutex-12             	14300929	        84.03 ns/op	       0 B/op	       0 allocs/op
```

Read it carefully. The unsynchronized version is 285x faster than the atomic one and
**produces the wrong answer** — it is in the table only to show what you are paying for
correctness, not as an option. Between the two correct versions, atomic is about 1.4x
faster than the mutex, which is a much smaller gap than people expect: both are
hammering one cache line across 12 cores, and cache-line ping-pong dominates. That 1.4x
is the entire prize for giving up the ability to guard an invariant.

Idiomatic: use a mutex by default. Switch to an atomic when the state really is a single
counter or flag or pointer, and the profile says that line matters.

## Sequential consistency, in one paragraph

Go's memory model states that the atomic operations in `sync/atomic` are sequentially
consistent: all goroutines observe them in one single total order, consistent with
program order within each goroutine. There is no ordering parameter to get wrong, and an
atomic `Store` that is observed by an atomic `Load` establishes a happens-before edge, so
everything the storing goroutine did beforehand is visible to the loading goroutine
afterwards. That is precisely why the `atomic.Pointer` config example is safe: building
the `Config` happens-before the `Store`, and the `Store` happens-before every `Load` that
sees it, so the fields are fully visible. The formal rules — and why "it worked when I
tested it" is not evidence — are in
[12.6](../12-concurrency-patterns/06-memory-model.md).

## Quick Reference

| Typed value | Methods |
|---|---|
| `atomic.Int32` / `Int64` | `Load`, `Store`, `Swap`, `CompareAndSwap`, `Add`, `And`, `Or` |
| `atomic.Uint32` / `Uint64` / `Uintptr` | same as above |
| `atomic.Bool` | `Load`, `Store`, `Swap`, `CompareAndSwap` (no `Add`) |
| `atomic.Pointer[T]` | `Load`, `Store`, `Swap`, `CompareAndSwap` |
| `atomic.Value` | `Load`, `Store`, `Swap`, `CompareAndSwap` — panics on a type change |

| Method | Returns | Note |
|---|---|---|
| `v.Load()` | `T` | the current value |
| `v.Store(x)` | — | unconditional write |
| `v.Swap(x)` | `T` (old) | write and read the previous value |
| `v.CompareAndSwap(old, new)` | `bool` | writes only if the value is still `old` |
| `v.Add(delta)` | `T` (new) | integer types only; negative delta subtracts |
| `v.And(mask)` / `v.Or(mask)` | `T` (old) | bitwise, Go 1.23+ |

| Legacy function | Typed equivalent |
|---|---|
| `atomic.AddInt64(&x, 1)` | `v.Add(1)` |
| `atomic.LoadInt64(&x)` | `v.Load()` |
| `atomic.StoreInt64(&x, n)` | `v.Store(n)` |
| `atomic.SwapInt64(&x, n)` | `v.Swap(n)` |
| `atomic.CompareAndSwapInt64(&x, o, n)` | `v.CompareAndSwap(o, n)` |
| `atomic.LoadPointer` / `StorePointer` (`unsafe.Pointer`) | `atomic.Pointer[T]` |

| Need | Use |
|---|---|
| a shared counter or ID generator | `atomic.Int64` |
| a one-shot "closed" / "ready" flag | `atomic.Bool` |
| hot-swap an immutable config or routing table | `atomic.Pointer[T]` |
| a max, a bitmask, a saturating counter | `CompareAndSwap` in a loop |
| two or more fields that must change together | [`sync.Mutex`](02-mutex.md) |
| exactly-once initialization | [`sync.Once`](03-once-pool-map.md) |

## Common Mistakes

**Mixing an atomic access and a plain access to the same variable.** Every read and
write of that variable must go through `sync/atomic`; one plain `x++` anywhere reinstates
the data race.

**Copying a struct that contains an atomic field.** The copy has an independent value and
silently stops synchronizing; pass a pointer, and let `go vet`'s `copylocks` check catch
you.

**Using two atomic operations where the invariant spans both.** `a.Add(-n)` followed by
`b.Add(n)` is race-free but not atomic as a pair; hold a mutex across both.

**Reaching for `atomic.Value` instead of `atomic.Pointer[T]`.** `Value` loses type
safety and panics at runtime if a second `Store` has a different dynamic type.

**Assuming an atomic counter is free.** It is roughly 60ns under 12-way contention on
one cache line — only 1.4x faster than a mutex — so the win comes from reducing shared
writes, not from the instruction.

**Writing `atomic.AddInt64(&s.count, 1)` on a struct field.** That is the 32-bit
alignment panic waiting to happen; use `atomic.Int64` as the field type instead.

## Exercises

1. Build a lock-free `RateLimiter` with `atomic.Int64` holding packed
   `(tokens, lastRefillUnixMilli)` in one word, refilling with a CAS loop. Prove under
   `-race` that 1000 goroutines never hand out more than the configured burst.
2. Write a `Counters` type with `atomic.Pointer[map[string]int64]` doing copy-on-write
   updates via a CAS loop. Benchmark it against `map` + `RWMutex` at 99% reads and at
   50% reads, and explain the crossover.
3. Take the broken two-field `Account` from this lesson and fix it *without* a mutex by
   packing both balances into a single `atomic.Uint64` (32 bits each). State in two
   sentences what this approach costs you.
4. Implement `atomic.Bool` yourself on top of `atomic.Int32`, including
   `CompareAndSwap`, and show with a test that your version and the stdlib version agree
   under 8 concurrent writers.
5. Measure how the atomic counter benchmark scales: run it with `-cpu 1,2,4,8,12` and
   plot ns/op against thread count. Explain why the curve bends the way it does, then
   fix it with per-CPU sharded counters summed on read.

---

[← 11.3 sync.Once, sync.Pool, and sync.Map](03-once-pool-map.md) · [Course Index](../README.md) · [11.5 The Race Detector →](05-race-detector.md)
