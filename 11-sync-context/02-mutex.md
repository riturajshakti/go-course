[← 11.1 sync.WaitGroup](01-waitgroup.md) · [Course Index](../README.md) · [11.3 sync.Once, sync.Pool, and sync.Map →](03-once-pool-map.md)

# 11.2 — sync.Mutex and sync.RWMutex

## What You'll Learn

- The zero value is a ready, unlocked mutex; `defer mu.Unlock()` is the default shape
- Go mutexes are **not reentrant** and have no owner — re-locking self-deadlocks, and any goroutine may unlock
- `RWMutex` helps when read sections are *long*, and costs you when they are short
- Lock ordering, and why holding a lock across a channel send or I/O is the classic deadlock

## Coming From Another Language

| Other | Go |
|---|---|
| Rust `Mutex<T>` wraps the data | Go's mutex guards nothing by convention — discipline is yours |
| Rust `lock().unwrap()` returns a guard | `Lock()` returns nothing; you `defer Unlock()` |
| Python `threading.RLock` (reentrant) | no reentrant mutex exists; re-locking deadlocks |
| Python `with lock:` | `mu.Lock(); defer mu.Unlock()` |
| JS has no mutex (single-threaded event loop) | real shared-memory locking, with real data races |
| Java `synchronized` is per-object and reentrant | neither: no owner tracking, no recursion |

Go's mutex is deliberately minimal: two bits of state, no owner field, no recursion
count. That buys speed (an uncontended `Lock`/`Unlock` pair is under 2ns) and costs
safety — the compiler does not know which fields a mutex protects, so nothing stops
you reading `c.n` without the lock. Rust's `Mutex<T>` makes the association a type;
in Go it is a comment-free convention: the mutex sits immediately above the fields it
guards, and every access goes through a method that takes it.

## Lock, Unlock, and the counter race

Without a lock, `counter++` is read-modify-write on shared memory and loses updates.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  counter := 0
  var wg sync.WaitGroup
  for range 1000 {
    wg.Go(func() { counter++ })
  }
  wg.Wait()
  fmt.Println("counter:", counter)
}
```

This program is racy on purpose. The result is below 1000 and differs every run; three
real runs:

```
counter: 977
```

```
counter: 962
```

```
counter: 972
```

One mutex fixes it, and the result becomes exactly 1000 every time.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var mu sync.Mutex
  counter := 0

  var wg sync.WaitGroup
  for range 1000 {
    wg.Go(func() {
      mu.Lock()
      counter++
      mu.Unlock()
    })
  }
  wg.Wait()
  fmt.Println("counter:", counter)
}
```

Output:

```
counter: 1000
```

`var mu sync.Mutex` is fully usable — there is no constructor, and a mutex in a freshly
allocated struct is unlocked. For a counter specifically, [11.4](04-atomic.md) is faster;
a mutex earns its keep as soon as you guard more than one word.

## Protecting struct state

The idiomatic unit is a struct with the mutex as its first field and methods that take
it. Note `Snapshot` returns a *copy*: handing the caller the internal map would let them
read it after the lock is released.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
)

type Registry struct {
  mu    sync.Mutex
  hosts map[string]int
}

func NewRegistry() *Registry {
  return &Registry{hosts: make(map[string]int)}
}

func (r *Registry) Hit(host string) {
  r.mu.Lock()
  defer r.mu.Unlock()
  r.hosts[host]++
}

func (r *Registry) Snapshot() map[string]int {
  r.mu.Lock()
  defer r.mu.Unlock()
  out := make(map[string]int, len(r.hosts))
  for k, v := range r.hosts {
    out[k] = v
  }
  return out
}

func main() {
  r := NewRegistry()
  var wg sync.WaitGroup
  for range 500 {
    wg.Go(func() {
      r.Hit("api")
      r.Hit("web")
    })
  }
  wg.Wait()

  snap := r.Snapshot()
  keys := make([]string, 0, len(snap))
  for k := range snap {
    keys = append(keys, k)
  }
  sort.Strings(keys)
  for _, k := range keys {
    fmt.Printf("%s=%d\n", k, snap[k])
  }
}
```

Output:

```
api=500
web=500
```

Deterministic by construction — 500 goroutines each hit both keys exactly once — and
clean under `go run -race .`. The map keys are sorted before printing because map
iteration order is random.

Methods are on `*Registry`, not `Registry`. A value receiver would copy the mutex on
every call, so each call would lock its own private copy and protect nothing.

## Named field vs embedded

Embedding `sync.Mutex` promotes `Lock` and `Unlock` into your type's method set, which
makes them part of your public API.

```go
package main

import (
  "fmt"
  "sync"
)

func guarded(l sync.Locker, label string, f func()) {
  l.Lock()
  defer l.Unlock()
  fmt.Println("holding", label)
  f()
}

type Session struct {
  sync.Mutex
  id int
}

func main() {
  var mu sync.Mutex
  var rw sync.RWMutex

  guarded(&mu, "Mutex", func() {})
  guarded(&rw, "RWMutex (write)", func() {})
  guarded(rw.RLocker(), "RWMutex (read via RLocker)", func() {})

  s := &Session{id: 7}
  s.Lock()
  s.id++
  s.Unlock()
  fmt.Println("embedded mutex exposes Lock/Unlock on Session:", s.id)

  var _ sync.Locker = s
  fmt.Println("*Session satisfies sync.Locker, like it or not")
}
```

Output:

```
holding Mutex
holding RWMutex (write)
holding RWMutex (read via RLocker)
embedded mutex exposes Lock/Unlock on Session: 8
*Session satisfies sync.Locker, like it or not
```

Idiomatic: a named `mu` field. Embedding invites callers to lock your type from outside,
which means your invariants are no longer yours to enforce — and you can never change
the locking strategy without breaking them. Embed only when external locking *is* the
contract, which is rare.

The example also shows `sync.Locker`, the one-method-pair interface `{ Lock(); Unlock() }`.
Both `*Mutex` and `*RWMutex` satisfy it, and `rw.RLocker()` adapts the read side so
`RLock`/`RUnlock` can be passed where a `Locker` is wanted.

## Copying a struct that contains a mutex

A `sync.Mutex` must never be copied: the copy carries the original's lock state, and
the two then protect different memory. `go vet`'s copylocks check catches it.

```go
package main

import (
  "fmt"
  "sync"
)

type Counter struct {
  mu sync.Mutex
  n  int
}

func (c *Counter) Inc() {
  c.mu.Lock()
  defer c.mu.Unlock()
  c.n++
}

func describe(c Counter) string {
  return fmt.Sprintf("n=%d", c.n)
}

func main() {
  var c Counter
  c.Inc()
  fmt.Println(describe(c))
}
```

`go vet` output:

```
# example.com/scratch
main.go:19:17: describe passes lock by value: example.com/scratch.Counter contains sync.Mutex
main.go:26:24: call of describe copies lock value: example.com/scratch.Counter contains sync.Mutex
```

It compiles and prints `n=1` anyway — the diagnostic is the whole warning you get. Take
`*Counter`, or if you genuinely need a snapshot value, define a separate lock-free
struct for it.

## Unlocking an unlocked mutex is fatal

Not a panic you can recover from — a runtime fatal error.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var mu sync.Mutex
  mu.Lock()
  mu.Unlock()
  mu.Unlock()
  fmt.Println("unreachable")
}
```

This program dies on purpose. Output (goroutine dump truncated to the one frame that
matters):

```
fatal error: sync: unlock of unlocked mutex

goroutine 1 [running]:
internal/sync.fatal({0x1046e82f4?, 0x415148e721e0?})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/panic.go:1205 +0x20
internal/sync.(*Mutex).unlockSlow(0x415148f88020, 0xffffffff)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/internal/sync/mutex.go:204 +0x38
internal/sync.(*Mutex).Unlock(...)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/internal/sync/mutex.go:198
sync.(*Mutex).Unlock(...)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/mutex.go:65
main.main()
	/tmp/scratch/main.go:12 +0xfc
exit status 2
```

`fatal error`, not `panic` — `recover` cannot catch it, and the process always dies. This
is also why a mutex has no owner: *any* goroutine may unlock one locked by another. That
is occasionally useful (a handoff) and mostly a footgun.

## A Mutex is not reentrant

Calling a locked method from another locked method of the same object deadlocks the
goroutine against itself. This is the single most common mutex bug in Go.

```go
package main

import (
  "fmt"
  "sync"
)

type Account struct {
  mu      sync.Mutex
  balance int
}

func (a *Account) Balance() int {
  a.mu.Lock()
  defer a.mu.Unlock()
  return a.balance
}

func (a *Account) Deposit(n int) {
  a.mu.Lock()
  defer a.mu.Unlock()
  fmt.Println("balance before deposit:", a.Balance())
  a.balance += n
}

func main() {
  var a Account
  a.Deposit(10)
  fmt.Println("unreachable")
}
```

This program deadlocks on purpose. Note that not even the `balance before deposit:` text
appears — `a.Balance()` is evaluated before `Println` can run. Output (truncated):

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.Mutex.Lock]:
internal/sync.runtime_SemacquireMutex(0x104bf9798?, 0xc0?, 0x104b4062e?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/sema.go:95 +0x28
internal/sync.(*Mutex).lockSlow(0x6babe4490020)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/internal/sync/mutex.go:149 +0x174
main.(*Account).Balance(0x6babe4490020)
	/tmp/scratch/main.go:14 +0x78
main.(*Account).Deposit(0x6babe4490020, 0xa)
	/tmp/scratch/main.go:22 +0x9c
main.main()
	/tmp/scratch/main.go:28 +0x34
exit status 2
```

`goroutine 1 [sync.Mutex.Lock]` plus two of your own frames in the stack is the
signature: a locked method calling another locked method.

The fix is the standard layering — an unexported method that assumes the lock is already
held, called by both:

```go
func (a *Account) balanceLocked() int {
  return a.balance
}

func (a *Account) Balance() int {
  a.mu.Lock()
  defer a.mu.Unlock()
  return a.balanceLocked()
}

func (a *Account) Deposit(n int) {
  a.mu.Lock()
  defer a.mu.Unlock()
  a.balance = a.balanceLocked() + n
}
```

The `Locked` suffix is the conventional marker for "caller must hold the mutex".

## TryLock and TryRLock

`TryLock` acquires or returns false immediately; it never blocks.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var mu sync.Mutex

  fmt.Println("TryLock on a free mutex:", mu.TryLock())
  fmt.Println("TryLock while held:     ", mu.TryLock())
  mu.Unlock()
  fmt.Println("TryLock after Unlock:   ", mu.TryLock())
  mu.Unlock()

  var rw sync.RWMutex
  rw.RLock()
  fmt.Println("TryLock on read-held:   ", rw.TryLock())
  fmt.Println("TryRLock on read-held:  ", rw.TryRLock())
  rw.RUnlock()
  rw.RUnlock()

  var wg sync.WaitGroup
  skipped := 0
  var smu sync.Mutex
  mu.Lock()
  for range 100 {
    wg.Go(func() {
      if !mu.TryLock() {
        smu.Lock()
        skipped++
        smu.Unlock()
        return
      }
      mu.Unlock()
    })
  }
  wg.Wait()
  mu.Unlock()
  fmt.Println("goroutines that skipped a held lock:", skipped)
}
```

Output:

```
TryLock on a free mutex: true
TryLock while held:      false
TryLock after Unlock:    true
TryLock on read-held:    false
TryRLock on read-held:   true
goroutines that skipped a held lock: 100
```

The count is exactly 100 because `main` holds the lock for the whole loop, so every
goroutine must fail. `TryLock` on a read-held `RWMutex` fails (a writer must wait for
readers) while `TryRLock` succeeds (readers share).

The standard library's own documentation notes that uses of `TryLock` are rare, and that
needing it often means the design is wrong. Legitimate cases: skipping optional work
rather than queueing for it, and metrics or debug endpoints that must never block the
hot path.

## RWMutex: many readers or one writer

`RLock` is shared, `Lock` is exclusive. This program proves the sharing by counting peak
simultaneous readers.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
  "time"
)

type Cache struct {
  mu sync.RWMutex
  m  map[string]string
}

func (c *Cache) Get(k string) (string, bool) {
  c.mu.RLock()
  defer c.mu.RUnlock()
  v, ok := c.m[k]
  return v, ok
}

func (c *Cache) Set(k, v string) {
  c.mu.Lock()
  defer c.mu.Unlock()
  c.m[k] = v
}

func main() {
  c := &Cache{m: map[string]string{"k": "v0"}}

  var concurrent atomic.Int64
  var peak atomic.Int64
  var wg sync.WaitGroup
  for range 8 {
    wg.Go(func() {
      c.mu.RLock()
      n := concurrent.Add(1)
      for {
        p := peak.Load()
        if n <= p || peak.CompareAndSwap(p, n) {
          break
        }
      }
      time.Sleep(20 * time.Millisecond)
      concurrent.Add(-1)
      c.mu.RUnlock()
    })
  }
  wg.Wait()
  fmt.Println("peak simultaneous readers:", peak.Load())

  c.Set("k", "v1")
  v, ok := c.Get("k")
  fmt.Println("get:", v, ok)
  _, ok = c.Get("missing")
  fmt.Println("missing:", ok)
}
```

Output:

```
peak simultaneous readers: 8
get: v1 true
missing: false
```

All eight hold the read lock at once, so the peak is 8 — the 20ms sleep is long enough
that no reader can finish before the others arrive. Swap `RLock` for `Lock` and the peak
drops to 1.

Two rules the API will not enforce. `RLock` does not permit writes: the "read" is a
promise you make. And `RLock` is not upgradable — calling `Lock` while holding `RLock`
deadlocks exactly like the reentrancy case above. Release, re-acquire for write, and
re-check, because the state may have changed in between.

## When RWMutex helps, and when it is slower

`RWMutex` carries more bookkeeping than `Mutex`, so it only pays off when read sections
are long enough for the parallelism to outweigh that overhead. Real numbers from
`go test -bench . -benchmem` on this machine (darwin/arm64, Apple M4 Pro, 12 threads) —
absolute values are machine-specific, the ratios are the point.

First, a short read: one map lookup, 12 goroutines contending.

```
BenchmarkMutexReadHeavy-12            	11558122	        95.79 ns/op	       0 B/op	       0 allocs/op
BenchmarkRWMutexReadHeavy-12          	 8874817	       129.4 ns/op	       0 B/op	       0 allocs/op
```

`RWMutex` is **35% slower** here, despite the workload being 100% reads. The critical
section is a few nanoseconds of real work; the reader-count bookkeeping costs more than
the concurrency saves.

Now the same contention with a read that actually takes a while — summing a 2000-element
slice:

```
BenchmarkMutexLongRead-12      	 1757121	       683.8 ns/op	       0 B/op	       0 allocs/op
BenchmarkRWMutexLongRead-12    	 9355609	       127.5 ns/op	       0 B/op	       0 allocs/op
```

**5.4x faster.** This is the case `RWMutex` exists for: concurrent readers doing enough
work each that running them in parallel is a real win.

Write-heavy, and uncontended:

```
BenchmarkMutexWriteHeavy-12           	11071348	       108.4 ns/op	       0 B/op	       0 allocs/op
BenchmarkRWMutexWriteHeavy-12         	18901965	        64.08 ns/op	       0 B/op	       0 allocs/op
BenchmarkMutexUncontended-12          	720458391	         1.669 ns/op	       0 B/op	       0 allocs/op
BenchmarkRWMutexUncontendedRead-12    	382259394	         3.153 ns/op	       0 B/op	       0 allocs/op
```

Uncontended, `RLock`/`RUnlock` costs about 1.9x a `Lock`/`Unlock` pair — 3.2ns against
1.7ns. (The write-heavy pair runs different contention dynamics under `RunParallel` and
is not a like-for-like comparison; treat it as evidence that "RWMutex is slower for
writes" is too simple a story, not as a reason to prefer it.)

Idiomatic: start with `Mutex`. Move to `RWMutex` only when a profile shows read
contention on a non-trivial critical section. Reaching for `RWMutex` because the
workload "sounds read-heavy" is how you make code slower and harder to reason about.

## Never hold a lock across a channel send or I/O

This is the rule that catches real services. `Publish` holds the lock while sending on
an unbuffered channel; the subscriber needs the same lock to unsubscribe.

```go
package main

import (
  "fmt"
  "sync"
)

type Broker struct {
  mu   sync.Mutex
  subs []chan int
}

func (b *Broker) Publish(v int) {
  b.mu.Lock()
  defer b.mu.Unlock()
  for _, ch := range b.subs {
    ch <- v
  }
}

func (b *Broker) Unsubscribe(ch chan int) {
  b.mu.Lock()
  defer b.mu.Unlock()
  for i, c := range b.subs {
    if c == ch {
      b.subs = append(b.subs[:i], b.subs[i+1:]...)
      return
    }
  }
}

func main() {
  sub := make(chan int)
  b := &Broker{subs: []chan int{sub}}

  go func() {
    <-sub
    b.Unsubscribe(sub)
  }()

  b.Publish(1)
  b.Publish(2)
  fmt.Println("unreachable")
}
```

This program deadlocks on purpose. Output (truncated to the two blocked goroutines):

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.(*Broker).Publish(0x2afdb38d8080?, 0x2)
	/tmp/scratch/main.go:17 +0xbc
main.main()
	/tmp/scratch/main.go:42 +0x100

goroutine 6 [sync.Mutex.Lock]:
internal/sync.runtime_SemacquireMutex(0x2afdb38621e0?, 0x4?, 0x2afdb38c2f68?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/sema.go:95 +0x28
internal/sync.(*Mutex).lockSlow(0x2afdb38d8080)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/internal/sync/mutex.go:149 +0x174
main.(*Broker).Unsubscribe(0x2afdb38d8080, 0x2afdb38e4070)
	/tmp/scratch/main.go:22 +0x78
main.main.func1()
	/tmp/scratch/main.go:38 +0x38
```

`[chan send]` on one goroutine and `[sync.Mutex.Lock]` on another is the diagnostic
pattern. The fix: take a snapshot under the lock, release it, then do the blocking work.

```go
func (b *Broker) snapshot() []chan int {
  b.mu.Lock()
  defer b.mu.Unlock()
  return append([]chan int(nil), b.subs...)
}

func (b *Broker) Publish(v int) {
  for _, ch := range b.snapshot() {
    ch <- v
  }
}
```

With those two methods replacing `Publish`, the program runs to completion: the
subscriber receives `1`, unsubscribes, and the second publish reaches nobody.
`append([]chan int(nil), b.subs...)` copies the slice so later mutation of `b.subs`
cannot affect the iteration.

The same rule covers HTTP calls, database queries, file writes and `log` calls: anything
that can block for an unbounded time must happen with the lock released. A lock held
across a 2-second network call serialises every other caller behind that 2 seconds.

## Lock ordering and the two-mutex deadlock

Two goroutines taking the same two locks in opposite orders is the AB-BA deadlock.

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

type Vault struct {
  mu   sync.Mutex
  name string
  n    int
}

func transfer(from, to *Vault, n int) {
  from.mu.Lock()
  defer from.mu.Unlock()
  time.Sleep(10 * time.Millisecond)
  to.mu.Lock()
  defer to.mu.Unlock()
  from.n -= n
  to.n += n
}

func main() {
  a := &Vault{name: "a", n: 100}
  b := &Vault{name: "b", n: 100}

  var wg sync.WaitGroup
  wg.Go(func() { transfer(a, b, 10) })
  wg.Go(func() { transfer(b, a, 10) })
  wg.Wait()
  fmt.Println("unreachable")
}
```

This program deadlocks on purpose; the `Sleep` only makes it reliable, the bug is there
without it. Output (truncated to the first blocked worker):

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
sync.runtime_SemacquireWaitGroup(0x10310a920?, 0xe8?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/sema.go:114 +0x38
main.main()
	/tmp/scratch/main.go:32 +0x148

goroutine 19 [sync.Mutex.Lock]:
internal/sync.runtime_SemacquireMutex(0x0?, 0x0?, 0x0?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/sema.go:95 +0x28
internal/sync.(*Mutex).lockSlow(0x2eafd9598080)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/internal/sync/mutex.go:149 +0x174
main.transfer(0x2eafd9598060, 0x2eafd9598080, 0xa)
	/tmp/scratch/main.go:19 +0xfc
main.main.func1()
	/tmp/scratch/main.go:30 +0x24
exit status 2
```

Both workers are stuck on line 19 — the *second* `Lock` — each holding what the other
wants. The cure is a total order on locks, here by a stable id:

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

type Vault struct {
  mu sync.Mutex
  id int
  n  int
}

func transfer(from, to *Vault, n int) {
  first, second := from, to
  if first.id > second.id {
    first, second = second, first
  }
  first.mu.Lock()
  defer first.mu.Unlock()
  time.Sleep(time.Millisecond)
  second.mu.Lock()
  defer second.mu.Unlock()

  from.n -= n
  to.n += n
}

func main() {
  a := &Vault{id: 1, n: 100}
  b := &Vault{id: 2, n: 100}

  var wg sync.WaitGroup
  for range 50 {
    wg.Go(func() { transfer(a, b, 10) })
    wg.Go(func() { transfer(b, a, 10) })
  }
  wg.Wait()
  fmt.Println("a:", a.n, "b:", b.n, "total:", a.n+b.n)
}
```

Output:

```
a: 100 b: 100 total: 200
```

100 transfers in both directions complete, and the total is conserved. Every path now
takes the lower id first, so no cycle can form. In a real system the order is often by
primary key, memory address, or simply a documented module order — what matters is that
it is global and written down. Go has no deadlock detector for this; the runtime's
`fatal error` only fires when *every* goroutine is blocked, so a deadlock between two
goroutines in a live server hangs those requests silently. More on finding these:
[12.5](../12-concurrency-patterns/05-deadlocks-leaks.md).

Lock granularity, in one line: one mutex for the whole struct is correct and simple;
split into several only when a profile shows contention, and then accept that you have
just created a lock-ordering problem.

## Quick Reference

| Method | Receiver | Behaviour |
|---|---|---|
| `mu.Lock()` | `*Mutex`, `*RWMutex` | blocks until exclusive |
| `mu.Unlock()` | `*Mutex`, `*RWMutex` | releases; fatal error if not locked |
| `mu.TryLock()` | `*Mutex`, `*RWMutex` | `bool`, never blocks |
| `rw.RLock()` | `*RWMutex` | shared read; many may hold |
| `rw.RUnlock()` | `*RWMutex` | releases one read hold |
| `rw.TryRLock()` | `*RWMutex` | `bool`, never blocks |
| `rw.RLocker()` | `*RWMutex` | a `sync.Locker` whose Lock is `RLock` |

| Property | Go mutex |
|---|---|
| zero value usable | yes, unlocked |
| reentrant | **no** — self-deadlock |
| owner tracked | no — any goroutine can unlock |
| copyable | **no** — `go vet` copylocks |
| fair | no strict FIFO; starvation mode after 1ms waiting |
| recoverable misuse | no — `fatal error`, not a panic |

| Failure | Symptom |
|---|---|
| no lock at all | wrong results, found by `-race` |
| extra `Unlock` | `fatal error: sync: unlock of unlocked mutex` |
| re-lock in same goroutine | `goroutine N [sync.Mutex.Lock]` + your frames twice |
| opposite lock orders | all workers blocked on their second `Lock` |
| lock held across a send | one goroutine `[chan send]`, another `[sync.Mutex.Lock]` |
| struct copied | `go vet`: passes/copies lock by value |

| Need | Use |
|---|---|
| guard several fields together | `sync.Mutex` |
| long concurrent reads, profiled contention | `sync.RWMutex` |
| a single integer or pointer | [`sync/atomic`](04-atomic.md) |
| one-time initialization | [`sync.Once`](03-once-pool-map.md) |
| hand data between goroutines | a channel ([10.2](../10-concurrency-basics/02-channels.md)) |

## Common Mistakes

**Calling a locked method from another locked method.** Go mutexes are not reentrant and
this self-deadlocks; extract an unexported `fooLocked` helper that assumes the lock.

**Using a value receiver on a method that locks.** Each call locks its own copy of the
mutex and protects nothing; always use a pointer receiver.

**Holding the lock across a channel send, HTTP call, or query.** Any blocking operation
under a lock serialises every other caller and can deadlock; snapshot the state, release,
then block.

**Returning the internal map or slice from a getter.** The caller reads it after you
unlock, which is a race the detector will find; return a copy.

**Choosing `RWMutex` because the workload sounds read-heavy.** It is measurably slower for
short critical sections; start with `Mutex` and switch only on profile evidence.

**Taking two locks in whatever order the arguments arrive in.** That is the AB-BA
deadlock; impose a total order, such as by id or address.

## Exercises

1. Build a `SafeSet[T comparable]` with `Add`, `Has`, `Delete`, `Len` and `Items() []T`
   methods over a `map[T]struct{}` and one mutex. Prove with `go test -race` that 1000
   concurrent `Add`s of 100 distinct values always give `Len() == 100`.
2. Take the reentrancy deadlock and fix it with the `balanceLocked` pattern. Then add a
   `Transfer(to *Account, n int)` method and show that two concurrent opposite transfers
   deadlock before you impose a lock order.
3. Instrument a `Mutex` by wrapping it in a type that counts acquisitions and total wait
   time using `time.Now()` around `Lock`. Report the numbers for 8 goroutines contending
   on a 1ms critical section, and explain why `sync.Locker` makes the wrapper a drop-in.
4. Reproduce the short-read vs long-read `RWMutex` benchmarks on your own machine, then
   find the critical-section length at which `RWMutex` overtakes `Mutex` by bisecting on
   the slice size.
5. Write a `Memoize` that caches expensive results under an `RWMutex` with the
   check-RLock / compute / check-again-under-Lock pattern. Show that two concurrent
   lookups of the same uncached key may both compute, then fix it so only one does
   without holding a lock during the computation.

---

[← 11.1 sync.WaitGroup](01-waitgroup.md) · [Course Index](../README.md) · [11.3 sync.Once, sync.Pool, and sync.Map →](03-once-pool-map.md)
