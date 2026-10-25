[← 10.6 select, default, and Timeouts](../10-concurrency-basics/06-select.md) · [Course Index](../README.md) · [11.2 sync.Mutex and sync.RWMutex →](02-mutex.md)

# 11.1 — sync.WaitGroup

## What You'll Learn

- `Add`/`Done`/`Wait`, the `defer wg.Done()` idiom, and Go 1.25's `wg.Go`
- Why `Add` must happen before the goroutine starts, and what goes wrong when it doesn't
- Writing results into a pre-sized slice by index needs no lock; `append` does
- The `WaitGroup` + channel + `close` fan-in pattern, and why errors need their own channel

## Coming From Another Language

| Other | Go |
|---|---|
| JS `await Promise.all([...])` | `wg.Wait()` — waits, but returns nothing |
| Python `asyncio.gather(*tasks)` | `wg.Wait()` + your own result slice |
| Python `ThreadPoolExecutor` + `as_completed` | no results, no errors; WaitGroup is a counter only |
| Rust: join handles carry a return value | goroutines return nothing; you write to a slice or channel |
| JS `Promise.all` rejects on the first error | `Wait` has no error path at all — see [11.8](08-errgroup.md) |

A `WaitGroup` is a counting semaphore, nothing more. It does not collect results,
it does not propagate errors, and it does not limit concurrency. `Add(n)` raises
the counter, `Done` lowers it by one, and `Wait` blocks until it hits zero. Every
pattern in this lesson is about working around the fact that it carries no data.

## Add, Done, Wait

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var wg sync.WaitGroup
  for i := 1; i <= 3; i++ {
    wg.Add(1)
    go func() {
      defer wg.Done()
      _ = i * i
    }()
  }
  wg.Wait()
  fmt.Println("all three finished")
}
```

Output:

```
all three finished
```

The zero value is ready to use — no `NewWaitGroup`. `defer wg.Done()` is not
optional style: if the goroutine panics or returns early from any of several
branches, a non-deferred `Done` is skipped and `Wait` hangs forever.

## wg.Go — the modern form

Go 1.25 added `wg.Go(f)`, which does the `Add(1)`, starts the goroutine, and
defers the `Done` for you.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
)

func main() {
  var mu sync.Mutex
  var out []string

  var wg sync.WaitGroup
  for _, name := range []string{"alpha", "beta", "gamma"} {
    wg.Go(func() {
      mu.Lock()
      defer mu.Unlock()
      out = append(out, name)
    })
  }
  wg.Wait()

  sort.Strings(out)
  fmt.Println(out)
}
```

Output:

```
[alpha beta gamma]
```

The append order varies run to run, which is why this sorts before printing.

Idiomatic: on Go 1.25+, use `wg.Go`. It makes the `Add`/`Done` mismatch bugs
below structurally impossible. The explicit form stays valid and is what you will
read in all existing code, so you need to know both.

## Add must come before the goroutine starts

Calling `Add` inside the goroutine is the classic bug. `Wait` can observe a
counter of zero before any goroutine has run, and returns immediately.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
)

func main() {
  var wg sync.WaitGroup
  var done atomic.Int64
  for i := 0; i < 1000; i++ {
    go func() {
      wg.Add(1)
      defer wg.Done()
      done.Add(1)
    }()
  }
  wg.Wait()
  fmt.Println("finished:", done.Load(), "of 1000")
}
```

The count is different every run. Three real runs:

```
finished: 989 of 1000
```

```
finished: 992 of 1000
```

```
finished: 981 of 1000
```

`Wait` returned while ~15 goroutines had not yet reached their `Add`. Nothing
panics at runtime — you just silently lose work, and the loss is load-dependent,
so it shows up in production and not in your test.

This one does get caught statically. `go vet` output:

```
# example.com/scratch
main.go:14:13: WaitGroup.Add called from inside new goroutine
```

Run `go vet ./...` in CI and this bug never reaches review. The runtime still
cannot help you, which is why the pattern is worth recognising by eye.

## Passing a WaitGroup by value

A `WaitGroup` must not be copied. Pass `*sync.WaitGroup`, or capture it in a
closure as every example above does.

```go
package main

import (
  "fmt"
  "sync"
)

func work(wg sync.WaitGroup, n int) {
  defer wg.Done()
  _ = n
}

func main() {
  var wg sync.WaitGroup
  wg.Add(1)
  go work(wg, 1)
  wg.Wait()
  fmt.Println("done")
}
```

`go vet` catches this. Output:

```
# example.com/scratch
main.go:8:14: work passes lock by value: sync.WaitGroup contains sync.noCopy
main.go:16:11: call of work copies lock value: sync.WaitGroup contains sync.noCopy
```

Run it anyway and the copy's `Done` decrements a counter nobody is waiting on.
This program deadlocks on purpose. Output:

```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [sync.WaitGroup.Wait]:
sync.runtime_SemacquireWaitGroup(0x1000e8200?, 0xd8?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/sema.go:114 +0x38
sync.(*WaitGroup).Wait(0x6d8bd92f0100)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:206 +0xa4
main.main()
	/tmp/scratch/main.go:17 +0x84
exit status 2
```

The goroutine dump is truncated here to the one goroutine that matters; the real
output lists all of them. Note the runtime names the blocked state
`[sync.WaitGroup.Wait]`, which tells you exactly what to look for.

## Negative counter panics

More `Done` calls than `Add` is a panic, not a silent wrap-around.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var wg sync.WaitGroup
  wg.Add(1)
  wg.Done()
  wg.Done()
  fmt.Println("unreachable")
}
```

This program panics on purpose. Output:

```
panic: sync: negative WaitGroup counter

goroutine 1 [running]:
sync.(*WaitGroup).Add(0x7b6af7696020, 0xffffffffffffffff)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:118 +0x264
sync.(*WaitGroup).Done(...)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:156
main.main()
	/tmp/scratch/main.go:12 +0x58
exit status 2
```

`Done` is just `Add(-1)` — hence the `0xffffffffffffffff` delta in the trace.

## Wait on zero, and reuse after Wait

`Wait` with a zero counter returns immediately, and a `WaitGroup` is reusable
once `Wait` has returned.

```go
package main

import (
  "fmt"
  "sync"
  "sync/atomic"
)

func main() {
  var wg sync.WaitGroup
  wg.Wait()
  fmt.Println("Wait on a zero counter returned immediately")

  var n atomic.Int64
  for round := 1; round <= 3; round++ {
    for i := 0; i < 5; i++ {
      wg.Go(func() { n.Add(1) })
    }
    wg.Wait()
    fmt.Println("round", round, "total", n.Load())
  }
}
```

Output:

```
Wait on a zero counter returned immediately
round 1 total 5
round 2 total 10
round 3 total 15
```

The one rule: do not call `Add` concurrently with a `Wait` that may already have
dropped to zero. Finish the round, then start the next.

## Collecting results by index — no lock needed

Pre-size the slice and give each goroutine exactly one slot. This is the single
most useful WaitGroup pattern.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  words := []string{"go", "is", "fun", "and", "fast"}
  lengths := make([]int, len(words))

  var wg sync.WaitGroup
  for i, w := range words {
    wg.Add(1)
    go func() {
      defer wg.Done()
      lengths[i] = len(w)
    }()
  }
  wg.Wait()
  fmt.Println(lengths)
}
```

Output:

```
[2 2 3 3 4]
```

This is deterministic and race-free, and `go run -race .` agrees. The reason is
precise: a data race requires two goroutines touching the *same memory*. Distinct
slice elements are distinct memory, the backing array is allocated before any
goroutine starts and never reallocated (no `append`), and `wg.Wait()` establishes
the happens-before edge that makes every write visible to the final `Println`.
Writing `lengths[i]` from goroutine `i` only is not sharing — it is partitioning.

Note this relies on Go 1.22+ per-iteration loop variables, so `i` and `w` are
fresh each iteration. Before 1.22 every goroutine saw the last values. See
[11.5](05-race-detector.md) for that bug under the race detector.

## Appending needs a mutex

`append` is three operations on shared state — read the header, maybe grow, write
the length — so concurrent appends corrupt each other.

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var wg sync.WaitGroup
  var got []int
  for i := 0; i < 100; i++ {
    wg.Go(func() { got = append(got, i) })
  }
  wg.Wait()
  fmt.Println("len:", len(got))
}
```

This program is racy on purpose. The length is wrong and varies; one real run:

```
len: 82
```

Under `go run -race .` the detector names it. Truncated to the first race of four:

```
==================
WARNING: DATA RACE
Write at 0x00c0001be000 by goroutine 7:
  main.main.func1()
      /tmp/scratch/main.go:12 +0x94

Previous read at 0x00c0001be000 by goroutine 11:
  main.main.func1()
      /tmp/scratch/main.go:12 +0x30

Goroutine 7 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c

Goroutine 11 (running) created at:
  sync.(*WaitGroup).Go()
      /opt/homebrew/Cellar/go/1.27.1/libexec/src/sync/waitgroup.go:238 +0x6c
==================
len: 65
Found 4 data race(s)
exit status 66
```

Addresses and goroutine numbers differ every run. The fix is one lock:

```go
package main

import (
  "fmt"
  "sync"
)

func main() {
  var mu sync.Mutex
  var wg sync.WaitGroup
  var got []int
  for i := 0; i < 100; i++ {
    wg.Go(func() {
      mu.Lock()
      defer mu.Unlock()
      got = append(got, i)
    })
  }
  wg.Wait()
  fmt.Println("len:", len(got))
}
```

Output:

```
len: 100
```

The length is now right, but the *order* is still arbitrary. If you need order,
use the index pattern.

## WaitGroup + channel + close: fan-in

An unbuffered channel plus a closer goroutine lets you consume results as they
arrive instead of waiting for all of them.

```go
package main

import (
  "fmt"
  "sort"
  "sync"
)

func main() {
  results := make(chan int)

  var wg sync.WaitGroup
  for i := 1; i <= 5; i++ {
    wg.Go(func() { results <- i * i })
  }

  go func() {
    wg.Wait()
    close(results)
  }()

  var got []int
  for v := range results {
    got = append(got, v)
  }
  sort.Ints(got)
  fmt.Println(got)
}
```

Output:

```
[1 4 9 16 25]
```

The arrival order varies, hence the sort. The `wg.Wait()` must live in its own
goroutine: call it in `main` before the `range` and you deadlock, because the
senders block on an unbuffered channel nobody is reading yet.

## Errors need their own channel

`Wait` returns nothing, so an error inside a goroutine has nowhere to go. The
index pattern works for errors too, combined with `errors.Join`.

```go
package main

import (
  "errors"
  "fmt"
  "sync"
)

func parse(s string) error {
  if s == "" {
    return errors.New("empty input")
  }
  return nil
}

func main() {
  inputs := []string{"a", "", "c", ""}
  errs := make([]error, len(inputs))

  var wg sync.WaitGroup
  for i, s := range inputs {
    wg.Go(func() { errs[i] = parse(s) })
  }
  wg.Wait()

  fmt.Println(errors.Join(errs...))
}
```

Output:

```
empty input
empty input
```

`errors.Join` drops the nil entries and joins the rest with newlines. When you
also want the first error to *cancel* the remaining work, stop hand-rolling this
and use [errgroup](08-errgroup.md).

## Quick Reference

| Call | Effect |
|---|---|
| `var wg sync.WaitGroup` | zero value, ready |
| `wg.Add(n)` | counter += n; must precede the goroutine |
| `wg.Done()` | counter -= 1; identical to `Add(-1)` |
| `wg.Wait()` | blocks until counter == 0; returns at once if already 0 |
| `wg.Go(f)` | Go 1.25+: `Add(1)`, `go f()`, `defer Done()` |

| Need | Use |
|---|---|
| just wait for N goroutines | `WaitGroup` |
| ordered results | pre-sized slice, write by index |
| unordered results | mutex + `append`, or a channel |
| stream results as they finish | channel + `go func(){ wg.Wait(); close(ch) }()` |
| first error cancels the rest | [`errgroup.WithContext`](08-errgroup.md) |
| bound concurrency | `errgroup.SetLimit` or a semaphore channel |

| Failure | Symptom |
|---|---|
| `Add` inside the goroutine | `Wait` returns early, silently |
| missing `Done` on a branch | `fatal error: all goroutines are asleep - deadlock!` |
| extra `Done` | `panic: sync: negative WaitGroup counter` |
| passed by value | `go vet` copylocks, then a deadlock |

## Common Mistakes

**Calling `wg.Add(1)` inside the goroutine.** `Wait` can see zero before any
goroutine runs; call `Add` in the loop body, or use `wg.Go`.

**Writing `wg.Done()` without `defer`.** An early return or panic skips it and
`Wait` blocks forever; always `defer wg.Done()` on the goroutine's first line.

**Passing `sync.WaitGroup` to a function by value.** The callee decrements a copy;
take `*sync.WaitGroup` and run `go vet` to catch it.

**Appending to a shared slice from several goroutines.** `append` races on the
header and loses elements; write by index, or hold a mutex.

**Calling `wg.Wait()` in `main` before draining the result channel.** The senders
block on a channel nobody reads; put `Wait` + `close` in their own goroutine.

## Exercises

1. Rewrite the Add-inside-the-goroutine program using `wg.Go` and show that it
   always reports 1000. Then explain, in terms of the counter, why the broken
   version loses a different number each run.
2. Write `func sizes(urls []string) []int` that fetches N URLs concurrently
   against an `httptest` server and returns body lengths in input order. Use no
   mutex. Verify with `go test -race`.
3. Take the mutex+`append` version and make the result deterministic without
   sorting and without an index — only by changing when goroutines are started.
   Explain what you gave up.
4. Build a `WaitGroup`-based fan-in that stops early when a consumer has seen 3
   results. Show that the remaining senders leak, then fix it with a `done`
   channel in a `select`.
5. Implement `func RunAll(fns []func() error) error` returning the joined errors,
   using only `sync` and `errors`. Then benchmark it against the `errgroup`
   version from [11.8](08-errgroup.md) and compare the allocation counts.

---

[← 10.6 select, default, and Timeouts](../10-concurrency-basics/06-select.md) · [Course Index](../README.md) · [11.2 sync.Mutex and sync.RWMutex →](02-mutex.md)
