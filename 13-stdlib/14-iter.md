[← 13.13 embed: Embedding Files in the Binary](13-embed.md) · [Course Index](../README.md) · [13.15 reflect: The Basics →](15-reflect.md)

# 13.14 — iter: Range-over-Func Iterators

## What You'll Learn

- The three range-over-func signatures and the `iter.Seq` / `iter.Seq2` aliases
- How `break`, `continue`, `return` and `goto` all surface as `yield` returning **false** — and the panic if you ignore it
- `iter.Pull` / `iter.Pull2`, and the merge problem that genuinely requires them
- Composing generic `Filter`/`Map`/`Take`, including over an **infinite** sequence
- A real benchmark: the overhead is ~2.5x a plain loop, and `iter.Pull` is ~70x

## Coming From Another Language

| Other | Go |
|---|---|
| Python generators (`yield`) | a function that **calls** `yield`, rather than being suspended by it |
| JS `[Symbol.iterator]` / generators | a plain function value; no protocol object, no `.next()` |
| Rust `Iterator` trait + adapters | an interface-free func type, and **no** stdlib `map`/`filter` |
| `IEnumerable` / LINQ | compose by writing the adapters yourself (they are five lines) |
| Python's `itertools` | `slices`/`maps`/`strings` provide sources, not combinators |

Go's iterators are **push**-based and inverted relative to most languages: your iterator
function receives a `yield` callback and calls it once per element. The loop body *is*
`yield`. That is why there is no iterator object, no state machine to write, and no
`.next()` — and why a recursive tree walk is naturally expressible, which is the thing
generators are usually needed for.

Stable since Go 1.23.

## The three signatures

A function is rangeable if it takes exactly one argument which is a `yield` function
returning `bool`:

| Signature | Loop form | Alias |
|---|---|---|
| `func(func() bool)` | `for range seq` | — |
| `func(func(V) bool)` | `for v := range seq` | `iter.Seq[V]` |
| `func(func(K, V) bool)` | `for k, v := range seq` | `iter.Seq2[K, V]` |

```go
type (
  Seq[V any]     func(yield func(V) bool)
  Seq2[K, V any] func(yield func(K, V) bool)
)
```

Those are **type aliases for function types**, nothing more — the `iter` package contains
these two declarations plus `Pull` and `Pull2`. You can write the signature out longhand
and it behaves identically; the aliases just document intent.

```go
package main

import (
  "fmt"
  "iter"
  "maps"
  "slices"
  "strings"
)

func Count(n int) iter.Seq[int] {
  return func(yield func(int) bool) {
    for i := range n {
      if !yield(i) {
        return
      }
    }
  }
}

func Values[S ~[]E, E any](s S) iter.Seq[E] {
  return func(yield func(E) bool) {
    for _, v := range s {
      if !yield(v) {
        return
      }
    }
  }
}

func Enumerate[E any](s []E) iter.Seq2[int, E] {
  return func(yield func(int, E) bool) {
    for i, v := range s {
      if !yield(i, v) {
        return
      }
    }
  }
}

func NoArgs() func(func() bool) {
  return func(yield func() bool) {
    for range 3 {
      if !yield() {
        return
      }
    }
  }
}

func main() {
  fmt.Println("--- the three signatures ---")
  n := 0
  for range NoArgs() {
    n++
  }
  fmt.Println("func(func() bool), iterations:", n)

  fmt.Print("iter.Seq[int]:   ")
  for v := range Count(5) {
    fmt.Print(v, " ")
  }
  fmt.Println()

  fmt.Print("iter.Seq2[int,string]: ")
  for i, v := range Enumerate([]string{"a", "b", "c"}) {
    fmt.Printf("%d=%s ", i, v)
  }
  fmt.Println()

  fmt.Println("\n--- the types are just function types ---")
  fmt.Printf("%T\n", Count(3))
  fmt.Printf("%T\n", Enumerate([]string{"a"}))
  fmt.Printf("slices.Values: %T\n", slices.Values([]int{1}))
  fmt.Printf("maps.All:      %T\n", maps.All(map[string]int{"a": 1}))

  fmt.Println("\n--- stdlib Seq-returning functions on 1.27 ---")
  s := []string{"x", "y", "z"}
  for i, v := range slices.All(s) {
    fmt.Printf("slices.All %d=%s  ", i, v)
  }
  fmt.Println()
  for i, v := range slices.Backward(s) {
    fmt.Printf("slices.Backward %d=%s  ", i, v)
  }
  fmt.Println()
  fmt.Println("slices.Collect(slices.Values):", slices.Collect(slices.Values(s)))
  fmt.Println("slices.Sorted:                ", slices.Sorted(Values([]int{3, 1, 2})))
  fmt.Println("slices.AppendSeq:             ", slices.AppendSeq([]int{0}, Count(3)))
  for c := range slices.Chunk([]int{1, 2, 3, 4, 5}, 2) {
    fmt.Print("slices.Chunk ", c, "  ")
  }
  fmt.Println()

  m := map[string]int{"b": 2, "a": 1, "c": 3}
  fmt.Println("slices.Sorted(maps.Keys):     ", slices.Sorted(maps.Keys(m)))
  fmt.Println("slices.Sorted(maps.Values):   ", slices.Sorted(maps.Values(m)))
  m2 := map[string]int{}
  maps.Insert(m2, maps.All(m))
  fmt.Println("maps.Insert + maps.All:       ", len(m2))
  fmt.Println("maps.Collect(maps.All):       ", len(maps.Collect(maps.All(m))))

  fmt.Printf("strings.SplitSeq:  %q\n", slices.Collect(strings.SplitSeq("a,b,c", ",")))
  fmt.Printf("strings.FieldsSeq: %q\n", slices.Collect(strings.FieldsSeq("  foo  bar ")))
  fmt.Printf("strings.Lines:     %q\n", slices.Collect(strings.Lines("a\nb\n")))
}
```

Output:

```
--- the three signatures ---
func(func() bool), iterations: 3
iter.Seq[int]:   0 1 2 3 4 
iter.Seq2[int,string]: 0=a 1=b 2=c 

--- the types are just function types ---
iter.Seq[int]
iter.Seq2[int,string]
slices.Values: iter.Seq[int]
maps.All:      iter.Seq2[string,int]

--- stdlib Seq-returning functions on 1.27 ---
slices.All 0=x  slices.All 1=y  slices.All 2=z  
slices.Backward 2=z  slices.Backward 1=y  slices.Backward 0=x  
slices.Collect(slices.Values): [x y z]
slices.Sorted:                 [1 2 3]
slices.AppendSeq:              [0 0 1 2]
slices.Chunk [1 2]  slices.Chunk [3 4]  slices.Chunk [5]  
slices.Sorted(maps.Keys):      [a b c]
slices.Sorted(maps.Values):    [1 2 3]
maps.Insert + maps.All:        3
maps.Collect(maps.All):        3
strings.SplitSeq:  ["a" "b" "c"]
strings.FieldsSeq: ["foo" "bar"]
strings.Lines:     ["a\n" "b\n"]
```

The sources the stdlib ships, verified by compiling the above on go1.27.1:

| Package | Returning a `Seq` | Consuming a `Seq` |
|---|---|---|
| `slices` | `All` (Seq2), `Values`, `Backward` (Seq2), `Chunk` | `Collect`, `Sorted`, `SortedFunc`, `SortedStableFunc`, `AppendSeq` |
| `maps` | `All` (Seq2), `Keys`, `Values` | `Collect`, `Insert` |
| `strings` | `SplitSeq`, `SplitAfterSeq`, `FieldsSeq`, `FieldsFuncSeq`, `Lines` | — |
| `bytes` | the same five `…Seq`/`Lines` forms | — |

`maps.Keys` returning an iterator rather than a slice is the one that trips people coming
from `golang.org/x/exp/maps` — `len()` on it does not compile (7.9). The canonical idiom
for stable map iteration is `slices.Sorted(maps.Keys(m))`.

## How the loop desugars

In one paragraph, because the mental model is all you need: `for v := range seq { body }`
compiles to `seq(func(v T) bool { body; return true })`. The loop body becomes the `yield`
function, and it returns `true` to ask for the next element. Any exit from the body —
`break`, `return`, `goto`, a labelled break, or a panic — makes `yield` return **false**
instead, and your iterator **must** then stop calling it and return. The compiler
rewrites `break`/`continue`/`return` inside the body into the right control flow plus the
right boolean, which is why they all work as if the loop were an ordinary `for`.

```go
package main

import (
  "fmt"
  "iter"
)

func Chatty(n int) iter.Seq[int] {
  return func(yield func(int) bool) {
    defer fmt.Println("    [iterator: deferred cleanup ran]")
    for i := range n {
      fmt.Printf("    [iterator: yielding %d]\n", i)
      if !yield(i) {
        fmt.Printf("    [iterator: yield(%d) returned false, stopping]\n", i)
        return
      }
    }
    fmt.Println("    [iterator: ran to completion]")
  }
}

func find() string {
  for v := range Chatty(5) {
    if v == 1 {
      return fmt.Sprintf("returned at %d", v)
    }
  }
  return "not found"
}

func main() {
  fmt.Println("=== break ===")
  for v := range Chatty(5) {
    if v == 2 {
      break
    }
  }

  fmt.Println("\n=== continue: yield keeps returning true ===")
  for v := range Chatty(3) {
    if v == 1 {
      continue
    }
    fmt.Println("  body saw", v)
  }

  fmt.Println("\n=== return from the enclosing function ===")
  fmt.Println("  result:", find())

  fmt.Println("\n=== goto out of the loop ===")
  for v := range Chatty(5) {
    if v == 1 {
      goto done
    }
  }
done:
  fmt.Println("  jumped to done")
}
```

Output:

```
=== break ===
    [iterator: yielding 0]
    [iterator: yielding 1]
    [iterator: yielding 2]
    [iterator: yield(2) returned false, stopping]
    [iterator: deferred cleanup ran]

=== continue: yield keeps returning true ===
    [iterator: yielding 0]
  body saw 0
    [iterator: yielding 1]
    [iterator: yielding 2]
  body saw 2
    [iterator: ran to completion]
    [iterator: deferred cleanup ran]

=== return from the enclosing function ===
    [iterator: yielding 0]
    [iterator: yielding 1]
    [iterator: yield(1) returned false, stopping]
    [iterator: deferred cleanup ran]
  result: returned at 1

=== goto out of the loop ===
    [iterator: yielding 0]
    [iterator: yielding 1]
    [iterator: yield(1) returned false, stopping]
    [iterator: deferred cleanup ran]
  jumped to done
```

Read the traces.

**`break` and `return` are indistinguishable from the iterator's side** — both simply make
`yield` return false. The iterator cannot tell *why* the consumer stopped, which is the
right abstraction: it only needs to clean up.

**`continue` does nothing special.** `yield` returns `true` as usual; the `continue` just
skips the rest of the body. The `3`-element trace shows all three yields happening.

**`defer` inside the iterator runs on every exit path** — completion, `break`, `return`,
`goto`. That is where cleanup belongs: a file to close, a transaction to roll back, a
connection to return to the pool. It is the whole reason an iterator can own a resource
safely.

**A labelled `break` out of nested range-over-funcs propagates outward**: the inner
`yield` returns false, then the outer one, and both deferred cleanups run innermost-first.
`goto` behaves identically to `break` from the iterator's side, as the trace shows.

### If you ignore yield's return value

```go
package main

import (
  "fmt"
  "iter"
)

func Broken(n int) iter.Seq[int] {
  return func(yield func(int) bool) {
    for i := range n {
      yield(i)
    }
  }
}

func main() {
  fmt.Println("breaking out of an iterator that ignores yield's return value:")
  for v := range Broken(5) {
    fmt.Println("  body saw", v)
    if v == 1 {
      break
    }
  }
  fmt.Println("unreachable")
}
```

This panics on purpose. Output:

```
breaking out of an iterator that ignores yield's return value:
  body saw 0
  body saw 1
panic: runtime error: range function continued iteration after function for loop body returned false

goroutine 1 [running]:
main.main-range1(...)
	/tmp/scratch/main.go:18
main.Broken.func1(...)
	/tmp/scratch/main.go:11
main.main()
	/tmp/scratch/main.go:18 +0x124
exit status 2
```

The runtime detects it and panics with a precise message rather than silently running the
body again after a `break`. So the `if !yield(v) { return }` shape is not a style
convention — omitting it is a bug the runtime will find, though only when a consumer
actually breaks. Test your iterators with a `break`.

Note the `main-range1` frame in the traceback: that is the compiler-generated function
holding your loop body, and seeing it in a stack trace is how you recognize a
range-over-func in production panics.

## iter.Pull: push to pull

Sometimes you need to advance **two** sequences independently, and a push iterator cannot
do that — you are inside its callback, not driving it. `iter.Pull` inverts control by
running the iterator in a coroutine:

```go
func Pull[V any](seq Seq[V]) (next func() (V, bool), stop func())
func Pull2[K, V any](seq Seq2[K, V]) (next func() (K, V, bool), stop func())
```

The canonical case is merging two sorted sequences.

```go
package main

import (
  "cmp"
  "fmt"
  "iter"
  "slices"
)

func Merge[E cmp.Ordered](a, b iter.Seq[E]) iter.Seq[E] {
  return func(yield func(E) bool) {
    nextA, stopA := iter.Pull(a)
    defer stopA()
    nextB, stopB := iter.Pull(b)
    defer stopB()

    va, okA := nextA()
    vb, okB := nextB()
    for okA && okB {
      if va <= vb {
        if !yield(va) {
          return
        }
        va, okA = nextA()
      } else {
        if !yield(vb) {
          return
        }
        vb, okB = nextB()
      }
    }
    for okA {
      if !yield(va) {
        return
      }
      va, okA = nextA()
    }
    for okB {
      if !yield(vb) {
        return
      }
      vb, okB = nextB()
    }
  }
}

func evens() iter.Seq[int] {
  return func(yield func(int) bool) {
    for i := 0; ; i += 2 {
      if !yield(i) {
        return
      }
    }
  }
}

func odds() iter.Seq[int] {
  return func(yield func(int) bool) {
    for i := 1; ; i += 2 {
      if !yield(i) {
        return
      }
    }
  }
}

func main() {
  fmt.Println("--- iter.Pull turns a push iterator into next/stop ---")
  next, stop := iter.Pull(slices.Values([]int{10, 20, 30}))
  for {
    v, ok := next()
    fmt.Printf("  next() = %d, %v\n", v, ok)
    if !ok {
      break
    }
  }
  fmt.Println("  calling next() after exhaustion:")
  v, ok := next()
  fmt.Printf("  next() = %d, %v\n", v, ok)
  stop()

  fmt.Println("\n--- stop() early, then next() reports done ---")
  next2, stop2 := iter.Pull(slices.Values([]int{1, 2, 3}))
  a, _ := next2()
  fmt.Println("  first value:", a)
  stop2()
  b, ok2 := next2()
  fmt.Printf("  after stop(): %d, %v\n", b, ok2)

  fmt.Println("\n--- iter.Pull2 for a Seq2 ---")
  n3, s3 := iter.Pull2(slices.All([]string{"x", "y"}))
  defer s3()
  for {
    i, val, ok := n3()
    if !ok {
      fmt.Println("  done")
      break
    }
    fmt.Printf("  %d=%s\n", i, val)
  }
  fmt.Println("\n--- merging two sorted sequences: what Pull is FOR ---")
  x := slices.Values([]int{1, 4, 7, 9})
  y := slices.Values([]int{2, 3, 8})
  fmt.Println("  merged:", slices.Collect(Merge(x, y)))

  fmt.Println("  uneven lengths:", slices.Collect(Merge(
    slices.Values([]int{1, 2, 3, 4, 5}), slices.Values([]int{10}))))
  fmt.Println("  one empty:     ", slices.Collect(Merge(
    slices.Values([]int{1, 2}), slices.Values([]int{}))))
  fmt.Println("  strings:       ", slices.Collect(Merge(
    slices.Values([]string{"ant", "cat"}), slices.Values([]string{"bee", "dog"}))))

  fmt.Println("\n--- and it is lazy: break after 8 of an infinite merge ---")
  count := 0
  for v := range Merge(evens(), odds()) {
    fmt.Print(v, " ")
    count++
    if count == 8 {
      break
    }
  }
  fmt.Println()
}
```

Output:

```
--- iter.Pull turns a push iterator into next/stop ---
  next() = 10, true
  next() = 20, true
  next() = 30, true
  next() = 0, false
  calling next() after exhaustion:
  next() = 0, false

--- stop() early, then next() reports done ---
  first value: 1
  after stop(): 0, false

--- iter.Pull2 for a Seq2 ---
  0=x
  1=y
  done

--- merging two sorted sequences: what Pull is FOR ---
  merged: [1 2 3 4 7 8 9]
  uneven lengths: [1 2 3 4 5 10]
  one empty:      [1 2]
  strings:        [ant bee cat dog]

--- and it is lazy: break after 8 of an infinite merge ---
0 1 2 3 4 5 6 7
```

**`Merge` could not be written without `Pull`.** It needs to look at the current head of
both sequences and advance only the smaller one, which requires driving each sequence on
demand. With two push iterators you would have to materialize one into a slice first,
losing the laziness — and the last section proves the laziness is real: merging two
**infinite** sequences and breaking after 8 values terminates immediately.

**`stop()` must always be called, and `defer` is how.** It releases the coroutine; leaking
it leaks a goroutine-like resource and the iterator's `defer`red cleanup never runs. Note
`stop()` is idempotent and safe after exhaustion, so `defer stop()` is unconditionally
correct.

**After `stop()` or exhaustion, `next()` returns the zero value and `false`** forever — no
panic, as both of the first two sections show. That makes the `v, ok := next()` loop shape
safe without extra guards.

**`Pull` is expensive.** See the benchmark below: ~70x a plain loop, because each `next()`
switches between coroutines. Use it when you genuinely need independent advancement
(merging, zipping, lookahead, a parser's token stream) and never as a general way to
consume a sequence.

## Composing iterators

The stdlib deliberately ships **no** `Map`/`Filter`/`Reduce` over `iter.Seq` — the
proposal was deferred because the right API is still being debated and the functions are
trivial to write. [9.5](../09-generics/05-when-not-to.md) makes the general argument
against a generic-combinator layer in Go; here are the five-line versions.

```go
package main

import (
  "fmt"
  "iter"
  "slices"
  "strings"
)

func Filter[E any](seq iter.Seq[E], keep func(E) bool) iter.Seq[E] {
  return func(yield func(E) bool) {
    for v := range seq {
      if keep(v) && !yield(v) {
        return
      }
    }
  }
}

func Map[A, B any](seq iter.Seq[A], f func(A) B) iter.Seq[B] {
  return func(yield func(B) bool) {
    for v := range seq {
      if !yield(f(v)) {
        return
      }
    }
  }
}

func Take[E any](seq iter.Seq[E], n int) iter.Seq[E] {
  return func(yield func(E) bool) {
    i := 0
    for v := range seq {
      if i >= n || !yield(v) {
        return
      }
      i++
    }
  }
}

func Naturals() iter.Seq[int] {
  return func(yield func(int) bool) {
    for i := 1; ; i++ {
      if !yield(i) {
        return
      }
    }
  }
}

type Node struct {
  Val         int
  Left, Right *Node
}

func (n *Node) InOrder() iter.Seq[int] {
  return func(yield func(int) bool) {
    n.walk(yield)
  }
}

func (n *Node) walk(yield func(int) bool) bool {
  if n == nil {
    return true
  }
  return n.Left.walk(yield) && yield(n.Val) && n.Right.walk(yield)
}

func main() {
  fmt.Println("--- composing generic iterator adapters ---")
  nums := slices.Values([]int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10})
  evens := Filter(nums, func(n int) bool { return n%2 == 0 })
  fmt.Println("Filter even:          ", slices.Collect(evens))

  labels := Map(slices.Values([]int{1, 2, 3}), func(n int) string {
    return strings.Repeat("*", n)
  })
  fmt.Printf("Map to strings:        %q\n", slices.Collect(labels))

  pipeline := Map(
    Filter(slices.Values([]int{1, 2, 3, 4, 5, 6}), func(n int) bool { return n%2 == 1 }),
    func(n int) int { return n * n },
  )
  fmt.Println("odd then squared:     ", slices.Collect(pipeline))

  fmt.Println("\n--- laziness: Take over an INFINITE sequence ---")
  fmt.Println("first 5 naturals:     ", slices.Collect(Take(Naturals(), 5)))
  bigSquares := Take(Filter(Map(Naturals(), func(n int) int { return n * n }),
    func(n int) bool { return n%3 == 0 }), 4)
  fmt.Println("first 4 squares /3:   ", slices.Collect(bigSquares))
  fmt.Println("Take(seq, 0):         ", slices.Collect(Take(Naturals(), 0)))

  fmt.Println("\n--- iterating a tree recursively ---")
  root := &Node{Val: 5,
    Left:  &Node{Val: 3, Left: &Node{Val: 1}, Right: &Node{Val: 4}},
    Right: &Node{Val: 8, Left: &Node{Val: 7}, Right: &Node{Val: 9}},
  }
  fmt.Println("in-order:             ", slices.Collect(root.InOrder()))
  fmt.Print("break at the first > 4: ")
  for v := range root.InOrder() {
    if v > 4 {
      fmt.Println(v)
      break
    }
  }
  var empty *Node
  fmt.Println("nil tree:             ", slices.Collect(empty.InOrder()))
}
```

Output:

```
--- composing generic iterator adapters ---
Filter even:           [2 4 6 8 10]
Map to strings:        ["*" "**" "***"]
odd then squared:      [1 9 25]

--- laziness: Take over an INFINITE sequence ---
first 5 naturals:      [1 2 3 4 5]
first 4 squares /3:    [9 36 81 144]
Take(seq, 0):          []

--- iterating a tree recursively ---
in-order:              [1 3 4 5 7 8 9]
break at the first > 4: 5
nil tree:              []
```

**`Take(Naturals(), 5)` over an infinite generator terminates.** That is the property a
slice cannot have: `Naturals()` never ends, and `Take` stops asking after five. Composing
`Map` then `Filter` then `Take` over it evaluates exactly as many naturals as needed to
find four squares divisible by 3 — the whole pipeline is demand-driven with no
intermediate slice at any stage.

**The tree iterator is the best argument for the feature.** `walk` recurses and returns
`false` upward as soon as `yield` says stop, so `n.Left.walk(yield) && yield(n.Val) &&
n.Right.walk(yield)` is the entire in-order traversal with early termination — short-circuit
`&&` does the propagation. Writing that as an explicit stack-based `Iterator` struct with
a `Next()` method is thirty lines and a well-known interview question; here it is three.
And it works on a `nil` receiver, so the empty tree needs no special case.

Note each adapter adds a layer of `yield` indirection, which the benchmark quantifies.

## The error story

An iterator cannot return an error — the signature has no room for one. The convention is
`iter.Seq2[V, error]`, yielding a value-and-error pair per step, which lets the consumer
decide whether to skip or stop.

```go
package main

import (
  "bufio"
  "fmt"
  "iter"
  "strconv"
  "strings"
)

func ParseInts(r *strings.Reader) iter.Seq2[int, error] {
  return func(yield func(int, error) bool) {
    sc := bufio.NewScanner(r)
    for sc.Scan() {
      line := strings.TrimSpace(sc.Text())
      if line == "" {
        continue
      }
      n, err := strconv.Atoi(line)
      if !yield(n, err) {
        return
      }
    }
    if err := sc.Err(); err != nil {
      yield(0, err)
    }
  }
}

func main() {
  fmt.Println("--- Seq2[V, error]: the convention for a fallible iterator ---")
  input := "1\n2\nthree\n4\n"
  var sum int
  for n, err := range ParseInts(strings.NewReader(input)) {
    if err != nil {
      fmt.Println("  error:", err)
      continue
    }
    fmt.Println("  parsed", n)
    sum += n
  }
  fmt.Println("  sum of the valid ones:", sum)

  fmt.Println("\n--- stopping at the first error ---")
  var firstErr error
  var got []int
  for n, err := range ParseInts(strings.NewReader(input)) {
    if err != nil {
      firstErr = err
      break
    }
    got = append(got, n)
  }
  fmt.Println("  values before the error:", got)
  fmt.Println("  firstErr:", firstErr)
}
```

Output:

```
--- Seq2[V, error]: the convention for a fallible iterator ---
  parsed 1
  parsed 2
  error: strconv.Atoi: parsing "three": invalid syntax
  parsed 4
  sum of the valid ones: 7

--- stopping at the first error ---
  values before the error: [1 2]
  firstErr: strconv.Atoi: parsing "three": invalid syntax
```

**The consumer chooses the policy**, which is the advantage over a callback that returns
an error: `continue` to skip bad rows, `break` to stop at the first. The cost is that
every loop body must check `err`, and the compiler will not remind you — forgetting the
check gives you a zero value silently.

The alternative shape, for when the error is terminal rather than per-item, is a struct
that accumulates it:

```go
type Reader struct {
  err error
}

func (r *Reader) All() iter.Seq[Row] { /* stops and sets r.err */ }
func (r *Reader) Err() error         { return r.err }
```

That is `bufio.Scanner`'s design (13.3) — iterate, then check `Err()` once afterwards. Use
it when one failure ends the sequence; use `Seq2[V, error]` when individual items can fail
independently.

## Performance

```go
package main

import (
  "iter"
  "slices"
  "testing"
)

var data = makeData(1000)

func makeData(n int) []int {
  s := make([]int, n)
  for i := range s {
    s[i] = i
  }
  return s
}

func BenchmarkPlainLoop(b *testing.B) {
  for b.Loop() {
    sum := 0
    for _, v := range data {
      sum += v
    }
    _ = sum
  }
}

func BenchmarkRangeOverFunc(b *testing.B) {
  for b.Loop() {
    sum := 0
    for v := range slices.Values(data) {
      sum += v
    }
    _ = sum
  }
}

func BenchmarkCallback(b *testing.B) {
  for b.Loop() {
    sum := 0
    forEach(data, func(v int) { sum += v })
    _ = sum
  }
}

func forEach[E any](s []E, f func(E)) {
  for _, v := range s {
    f(v)
  }
}

func BenchmarkPull(b *testing.B) {
  for b.Loop() {
    sum := 0
    next, stop := iter.Pull(slices.Values(data))
    for {
      v, ok := next()
      if !ok {
        break
      }
      sum += v
    }
    stop()
    _ = sum
  }
}

func BenchmarkCollect(b *testing.B) {
  for b.Loop() {
    _ = slices.Collect(slices.Values(data))
  }
}
```

`go test -bench=. -benchmem`, summing a 1000-element slice. Machine-specific
(darwin/arm64, Apple M4 Pro); the ratios are the point.

Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkPlainLoop-12         	 2828970	       452.5 ns/op	       0 B/op	       0 allocs/op
BenchmarkRangeOverFunc-12     	 1000000	      1137 ns/op	       0 B/op	       0 allocs/op
BenchmarkCallback-12          	 1751739	       685.6 ns/op	       0 B/op	       0 allocs/op
BenchmarkPull-12              	   37252	     32123 ns/op	     256 B/op	       7 allocs/op
BenchmarkCollect-12           	  566978	      1969 ns/op	   25208 B/op	      12 allocs/op
```

Honest reading:

**Range-over-func costs ~2.5x a plain slice loop** (1137 vs 452 ns for 1000 elements —
about 0.7 ns per element of overhead). That is a real cost but a small absolute one, and
it **allocates nothing**: the closure and the loop-body function are both stack-allocated
here.

**A plain callback is cheaper than range-over-func** (686 vs 1137 ns), because the compiler
can inline `forEach`'s loop while the range-over-func machinery adds the `false`-propagation
bookkeeping. So you do not use iterators for speed.

**`iter.Pull` is ~70x a plain loop** (32 µs) with 7 allocations. Each `next()` is a
coroutine switch. This is the number to remember: `Pull` is a correctness tool for
problems that need independent advancement, not a convenience.

**Each composed adapter adds another `yield` indirection.** Measured separately, one
`Filter` layer lands in the same 0.9–1.1 µs band — cheap individually, but a ten-stage
pipeline is not free.

**`slices.Collect` allocates 25 KB** for a 1000-element `[]int`, as any slice-building
would — it grows by doubling, so `slices.AppendSeq` into a pre-sized slice is cheaper when
you know the length.

## When an iterator earns its place

**Use one when:**

- The source is **not** a slice: a tree, a graph, a linked list, a database cursor, a
  paginated API, a file being scanned.
- The sequence is **infinite** or unbounded, or expensive per element.
- You want to avoid **materializing** a large intermediate — streaming a million rows
  through a filter without ever holding them all.
- The iteration owns a **resource** that `defer` should release.
- You are exposing iteration over a type whose internals you do not want to leak.

**Use a slice when:**

- You already have one. `for i, v := range s` is clearer and faster than
  `for i, v := range slices.All(s)`, which exists for generic code, not for you.
- The caller needs random access, `len`, sorting, or more than one pass.
- The data is small and bounded.

The deciding question is whether the consumer should control *when* the next element is
produced. If yes, an iterator. If the whole collection already exists in memory, a slice.

## Quick Reference

| Need | Write |
|---|---|
| A key-value iterator | `func(yield func(K, V) bool)`, aliased `iter.Seq2[K, V]` |
| The mandatory shape | `for … { if !yield(v) { return } }` |
| Cleanup on any exit | `defer` inside the iterator function |
| A slice's values / index-value | `slices.Values(s)` / `slices.All(s)` |
| Map keys, sorted | `slices.Sorted(maps.Keys(m))` |
| Seq → slice | `slices.Collect(seq)`, `slices.Sorted(seq)`, `slices.AppendSeq(dst, seq)` |
| Seq2 → map | `maps.Collect(seq2)` or `maps.Insert(m, seq2)` |
| Drive a sequence manually | `next, stop := iter.Pull(seq)`; **`defer stop()`** |
| Two sequences in step | `iter.Pull` on both (merge, zip, diff) |
| Map / Filter / Take | write them; five lines each, none in the stdlib |
| A fallible iterator | `iter.Seq2[V, error]`, or a `.Err()` method |

| Trap | Reality |
|---|---|
| `yield(v)` without checking its result | **panic**: "range function continued iteration after …" |
| Forgetting `defer stop()` after `Pull` | leaks the coroutine; the iterator's `defer` never runs |
| Using `Pull` to consume a sequence | ~70x slower than ranging it |
| `slices.All(s)` over a plain slice | pointless; `range s` is clearer and faster |
| Ignoring `err` in a `Seq2[V, error]` loop | silently gives you zero values |
| Expecting an iterator to be faster | it is ~2.5x a plain loop; it buys laziness, not speed |

## Common Mistakes

**Calling `yield` without checking its return value.** The runtime panics as soon as a
consumer `break`s; always `if !yield(v) { return }`.

**Forgetting `defer stop()` after `iter.Pull`.** The coroutine and the iterator's deferred
cleanup both leak.

**Reaching for `iter.Pull` to consume a sequence.** Range over it instead; `Pull` is for
independent advancement and costs ~70x.

**Wrapping a slice you already have in `slices.All`.** It adds overhead and noise;
`range s` already yields index and value.

**Returning an `iter.Seq` from something that can fail.** Use `iter.Seq2[V, error]` or a
separate `Err()` method, or the failure has nowhere to go.

**Assuming an iterator can be ranged twice.** A `Seq` over a slice can; one over a scanner
or a network cursor cannot — document which yours is.

## Exercises

1. Write `Zip[A, B any](a iter.Seq[A], b iter.Seq[B]) iter.Seq2[A, B]` stopping at the
   shorter sequence, using `iter.Pull`. Test it with equal, unequal and empty inputs, and
   prove with an infinite sequence that it terminates.
2. Implement `Lines(r io.Reader) iter.Seq2[string, error]` over a `bufio.Scanner`,
   including the `sc.Err()` case, and test it against a reader that fails mid-stream.
3. Add `Reduce`, `Any`, `All`, `Count` and `Last` over `iter.Seq[E]`, then explain for each
   whether it can work on an infinite sequence.
4. Give a binary tree `PreOrder`, `PostOrder` and `LevelOrder` iterators. Which one cannot
   be written recursively with the `&&` trick, and why?
5. Write an iterator over a paginated API (fake the pages) that fetches the next page only
   when the consumer asks, and prove with a request counter that breaking after the first
   page issues exactly one fetch.
6. Benchmark a five-stage `Map`/`Filter` pipeline over 100,000 elements against the
   equivalent single `for` loop, and decide at what pipeline depth the iterator version
   stops being worth it.

---

[← 13.13 embed: Embedding Files in the Binary](13-embed.md) · [Course Index](../README.md) · [13.15 reflect: The Basics →](15-reflect.md)
