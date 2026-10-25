[← 3.1 if, else, and the if-with-statement Form](01-if.md) · [Course Index](../README.md) · [3.3 Every switch Form →](03-switch.md)

# 3.2 — Every for Loop Form

## What You'll Learn

- `for` is the only loop keyword; there is no `while`, no `do`, no `foreach`
- All nine range forms: over int, slice, array, string, map, channel, and function iterator
- `range` copies the element, so assigning to the value variable changes nothing
- Loop variables are per-iteration since Go 1.22, which silently fixed a decade of closure bugs

## Coming From Another Language

| Other | Go |
|---|---|
| JS `while (c) {}` | `for c {}` |
| JS `for (;;) {}` | `for {}` |
| Python `for i in range(n)` | `for i := range n` (Go 1.22+) |
| JS `for (const v of xs)` | `for _, v := range xs` |
| JS `for (const k in obj)` | `for k := range m` — but order is randomized |
| Python `for k, v in d.items()` | `for k, v := range m` |
| Python `for ch in s` — characters | `for _, r := range s` — runes, with byte offsets |
| Rust `for x in it` | `for x := range it` where `it` is `func(func(T) bool)` |
| JS `do {} while (c)` | no equivalent; `for { ...; if !c { break } }` |

## The three non-range forms

```go
package main

import "fmt"

func main() {
  n := 0
  for {
    n++
    if n == 3 {
      break
    }
  }
  fmt.Println("bare:", n)

  m := 0
  for m < 3 {
    m++
  }
  fmt.Println("cond:", m)

  sum := 0
  for i := 0; i < 5; i++ {
    sum += i
  }
  fmt.Println("three-clause:", sum)

  for i, j := 0, 10; i < j; i, j = i+2, j-3 {
    fmt.Println("two vars:", i, j)
  }
}
```

Output:

```
bare: 3
cond: 3
three-clause: 10
two vars: 0 10
two vars: 2 7
```

Any of the three clauses may be empty. Write `for ; cond ; {` with both init and
post empty and `gofmt` rewrites it to `for cond {` — identical loops, and the
short form is the only one you will see in real code.

## Range over an int

Go 1.22 added `range` over an integer. It yields `0 .. n-1`.

```go
package main

import "fmt"

func main() {
  for i := range 4 {
    fmt.Print(i, " ")
  }
  fmt.Println()

  for range 3 {
    fmt.Print("x")
  }
  fmt.Println()

  n := -1
  for i := range n {
    fmt.Println("never", i)
  }
  fmt.Println("negative range: zero iterations")

  var u uint8 = 3
  for i := range u {
    fmt.Printf("%d(%T) ", i, i)
  }
  fmt.Println()
}
```

Output:

```
0 1 2 3 
xxx
negative range: zero iterations
0(uint8) 1(uint8) 2(uint8) 
```

A negative bound runs zero times rather than panicking, and the loop variable
takes the type of the operand — ranging a `uint8` gives a `uint8` index.
Idiomatic: `for range n` when you need a count and not the index.

## Range over a slice — all four shapes

```go
package main

import "fmt"

func main() {
  xs := []string{"a", "b", "c"}

  for i, v := range xs {
    fmt.Println(i, v)
  }
  for i := range xs {
    fmt.Println("index only:", i)
  }
  var nilSlice []int
  for i, v := range nilSlice {
    fmt.Println("unreachable", i, v)
  }
  fmt.Println("nil slice ranges zero times")
}
```

Output:

```
0 a
1 b
2 c
index only: 0
index only: 1
index only: 2
nil slice ranges zero times
```

Dropping the second variable gives the index; dropping both gives a counted loop.
Ranging a `nil` slice is legal and does nothing, so no nil check is needed.

## range copies the element

This is the gotcha that bites everyone once. `v` is a fresh copy; writing to it
is writing to a local.

```go
package main

import "fmt"

func main() {
  xs := []int{1, 2, 3}
  for _, v := range xs {
    v *= 10
  }
  fmt.Println("after modifying v:", xs)

  for i := range xs {
    xs[i] *= 10
  }
  fmt.Println("after modifying xs[i]:", xs)
}
```

Output:

```
after modifying v: [1 2 3]
after modifying xs[i]: [10 20 30]
```

The copy costs time too: ranging a `[]SomeBigStruct` by value copies the whole
struct each iteration. Idiomatic: `for i := range xs` and index when the element
is large or you need to mutate it.

## range evaluates the range expression once

For a slice this means the header is snapshotted — writes to the elements are
still visible, because the header points at the same array. For an *array*, the
whole array is copied, so writes are invisible.

```go
package main

import "fmt"

func main() {
  arr := [3]int{1, 2, 3}
  for i, v := range arr {
    if i == 0 {
      arr[1] = 99
      arr[2] = 99
    }
    fmt.Println(i, v, arr)
  }

  sl := []int{1, 2, 3}
  for i, v := range sl {
    if i == 0 {
      sl[1] = 99
      sl[2] = 99
    }
    fmt.Println(i, v, sl)
  }
}
```

Output:

```
0 1 [1 99 99]
1 2 [1 99 99]
2 3 [1 99 99]
0 1 [1 99 99]
1 99 [1 99 99]
2 99 [1 99 99]
```

Same code, different result, purely because one is `[3]int` and the other is
`[]int`. The length is also fixed at loop entry, so appending inside a `range`
over a slice does not extend the loop.

## Range over a map

The key order is deliberately randomized on every run — the runtime picks a random
starting bucket. Do not rely on it.

```go
package main

import "fmt"

func main() {
  m := map[string]int{"a": 1, "b": 2, "c": 3, "d": 4, "e": 5, "f": 6, "g": 7, "h": 8}
  for k, v := range m {
    fmt.Print(k, "=", v, " ")
  }
  fmt.Println()
}
```

Three consecutive runs of the first loop on the same binary:

```
a=1 b=2 c=3 d=4 e=5 f=6 g=7 h=8 
f=6 g=7 h=8 a=1 b=2 c=3 d=4 e=5 
g=7 h=8 a=1 b=2 c=3 d=4 e=5 f=6 
```

The order above is not stable and will differ for you. When you need
determinism, sort the keys:

```go
package main

import (
  "fmt"
  "maps"
  "slices"
)

func main() {
  m := map[string]int{"c": 3, "a": 1, "b": 2}
  for _, k := range slices.Sorted(maps.Keys(m)) {
    fmt.Print(k, "=", m[k], " ")
  }
  fmt.Println()
}
```

Output:

```
a=1 b=2 c=3 
```

Deleting a key mid-range is safe and that key will not be produced. Whether a key
added mid-range is produced is unspecified.

## Range over a string yields byte offsets and runes

The index is a *byte* offset, not a character count, and the value is a `rune`
decoded from UTF-8. Multi-byte characters make the index jump.

```go
package main

import "fmt"

func main() {
  s := "héllo, 世界"
  for i, r := range s {
    fmt.Printf("%d %q %d\n", i, r, r)
  }
  fmt.Println("len:", len(s))
  fmt.Printf("s[1] = %x (a byte, not a rune)\n", s[1])
}
```

Output:

```
0 'h' 104
1 'é' 233
3 'l' 108
4 'l' 108
5 'o' 111
6 ',' 44
7 ' ' 32
8 '世' 19990
11 '界' 30028
len: 14
s[1] = c3 (a byte, not a rune)
```

Nine runes, fourteen bytes, and offsets 2, 9, 10, 12 and 13 never appear as range
indexes. Indexing a string directly gives you a `byte`, so `s[1]` is the first
half of the UTF-8 encoding of `é`.

Invalid UTF-8 does not panic — each bad byte yields `U+FFFD` and advances one:

```go
package main

import "fmt"

func main() {
  bad := "a\xffb"
  for i, r := range bad {
    fmt.Printf("%d %q %U\n", i, r, r)
  }
}
```

Output:

```
0 'a' U+0061
1 '�' U+FFFD
2 'b' U+0062
```

## Range over a channel

The loop receives until the channel is closed, then exits. Full treatment in
[10.4 Closing, range over a Channel, and the comma-ok Receive](../10-concurrency-basics/04-closing-range.md).

```go
package main

import "fmt"

func main() {
  ch := make(chan int, 3)
  go func() {
    for i := range 3 {
      ch <- i * i
    }
    close(ch)
  }()
  for v := range ch {
    fmt.Println(v)
  }
  fmt.Println("channel closed, loop exited")
}
```

Output:

```
0
1
4
channel closed, loop exited
```

A channel range yields one value, never a pair. If nobody closes the channel, the
loop blocks forever — the most common cause of a goroutine leak.

`time.Tick` returns a channel, so `for range time.Tick(d)` is a valid periodic
loop. It leaks the ticker by design, so it belongs only in a program that runs
until exit; otherwise use `time.NewTicker` and `defer t.Stop()`.

## Range over a function iterator

Go 1.23 lets you range over a function whose single parameter is a `yield`
callback. Returning `false` from `yield` means the consumer stopped, and the
iterator must return. Detailed in
[13.14 iter: Range-over-Func Iterators](../13-stdlib/14-iter.md).

```go
package main

import "fmt"

func countdown(from int) func(func(int) bool) {
  return func(yield func(int) bool) {
    for i := from; i > 0; i-- {
      if !yield(i) {
        return
      }
    }
  }
}

func main() {
  for v := range countdown(4) {
    fmt.Print(v, " ")
  }
  fmt.Println()

  for v := range countdown(10) {
    if v == 7 {
      break
    }
    fmt.Print(v, " ")
  }
  fmt.Println()
}
```

Output:

```
4 3 2 1 
10 9 8 
```

The three accepted signatures are `func(func() bool)`, `func(func(V) bool)`, and
`func(func(K, V) bool)` — zero, one, or two loop variables. `iter.Seq[V]` and
`iter.Seq2[K, V]` are named versions of the last two.

## Loop variables are per-iteration

Since Go 1.22 each iteration gets a fresh variable, so a closure captures that
iteration's value. This is the single most user-visible change Go has made to
loop semantics.

```go
package main

import "fmt"

func main() {
  var fs []func()
  for i := range 3 {
    fs = append(fs, func() { fmt.Print(i, " ") })
  }
  for _, f := range fs {
    f()
  }
  fmt.Println()
}
```

Output:

```
0 1 2 
```

Before 1.22 that printed `3 3 3`. The semantics are selected by the `go`
directive in `go.mod`, not by the toolchain version. The same source compiled with
`go 1.21` in `go.mod` prints:

```
3 3 3 
```

and with `go 1.22` or later:

```
0 1 2 
```

One variable shared by all three closures, versus three separate variables. If
you maintain a module still declaring `go 1.21` or lower, the old trap is live;
`go vet`'s `loopclosure` check catches the common cases.

## break, continue, and nesting

```go
package main

import "fmt"

func main() {
  for i := range 4 {
    if i == 1 {
      continue
    }
    if i == 3 {
      break
    }
    fmt.Println("i =", i)
  }

  for i := range 3 {
    for j := range 3 {
      if j == 1 {
        break
      }
      fmt.Println(i, j)
    }
  }
}
```

Output:

```
i = 0
i = 2
0 0
1 0
2 0
```

`break` and `continue` apply to the innermost enclosing `for` only. To escape an
outer loop you need a label — that is [3.4 goto, Labels, break, and continue](04-labels-goto.md).
`continue` in a three-clause loop still runs the post statement, so
`for i := 0; i < n; i++ { ...; continue }` does not spin forever.

## Quick Reference

| Form | Yields | Notes |
|---|---|---|
| `for { }` | — | infinite; exit with `break`/`return` |
| `for cond { }` | — | the `while` loop |
| `for init; cond; post { }` | — | any clause may be empty |
| `for i := range n` | `0..n-1` | Go 1.22+; `n` is any integer type |
| `for range n` | — | counted loop, index discarded |
| `for i, v := range slice` | index, copy of element | |
| `for i := range slice` | index | no element copy |
| `for range slice` | — | `len(slice)` iterations |
| `for i, v := range array` | index, element of a **copy** | array is copied on entry |
| `for k, v := range map` | key, value | order randomized every run |
| `for k := range map` | key | |
| `for i, r := range string` | byte offset, `rune` | invalid bytes yield `U+FFFD` |
| `for v := range ch` | received value | ends at `close`; one variable only |
| `for v := range fn` | yielded value(s) | Go 1.23+; `fn` is `func(func(V) bool)` |

| Statement | Effect |
|---|---|
| `break` | leave the innermost `for`, `switch`, or `select` |
| `continue` | next iteration of the innermost `for`; post statement still runs |
| `break L` / `continue L` | target the labelled statement — see 3.4 |
| `goto` | see 3.4 |

| Nil / empty behaviour | Result |
|---|---|
| `range` over nil slice | 0 iterations |
| `range` over nil map | 0 iterations |
| `range` over negative int | 0 iterations |
| `range` over nil channel | blocks forever |

## Common Mistakes

**Assigning to the range value variable to mutate the collection.** `v` is a
copy; index with `xs[i] = ...` instead.

**Ranging an array and expecting to see mid-loop writes.** The array is copied at
loop entry; range a slice (`arr[:]`) if you need the live view.

**Treating the string range index as a character position.** It is a byte offset
and it skips; use `utf8.RuneCountInString` for a count.

**Depending on map iteration order, even in a test.** It is randomized per run;
sort the keys with `slices.Sorted(maps.Keys(m))`.

**Using `break` inside a `switch` inside a loop to exit the loop.** It only
leaves the `switch`; label the loop and use `break label`.

## Exercises

1. Write the same "sum 1..100" loop in five forms: bare `for` with a counter,
   `for cond`, three-clause, `for i := range 101`, and a range over a slice built
   with `make`. Confirm all five print 5050.
2. Range over `"naïve 日本"` and print, for each iteration, the byte offset, the
   rune, and `utf8.RuneLen(r)`. Verify the offsets sum to `len(s)`.
3. Range over a map while deleting a key you have already seen, and then while
   adding one. Run each 20 times and describe what the spec does and does not
   guarantee about what you observed.
4. Write a function iterator `func evens(limit int) func(func(int) bool)` and
   range over it with and without a `break`. Add a print inside the iterator to
   prove the `break` causes `yield` to return `false`.
5. In a scratch module, set `go 1.21` in `go.mod` and launch three goroutines
   from a loop that each print the loop variable. Record the output, bump to
   `go 1.22`, and record it again. Then find the `go vet` message that would have
   warned you.
---

[← 3.1 if, else, and the if-with-statement Form](01-if.md) · [Course Index](../README.md) · [3.3 Every switch Form →](03-switch.md)
