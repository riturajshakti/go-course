[← 3.3 Every switch Form](03-switch.md) · [Course Index](../README.md) · [3.5 defer: Order, Arguments, and Gotchas →](05-defer.md)

# 3.4 — goto, Labels, break, and continue

## What You'll Learn

- `break label` and `continue label` target an outer loop, replacing the boolean-flag dance
- A `switch` or `select` swallows a plain `break`, which is the main reason labels exist
- `goto` may not jump forward over a variable declaration, nor into a block — both are compile errors
- An unused label is a compile error, and labels are function-scoped

## Coming From Another Language

| Other | Go |
|---|---|
| JS `outer: for (...) { break outer; }` | `break outer` — same syntax, label before the `for` |
| Python — no labelled break | `break label` |
| Rust `'outer: loop { break 'outer; }` | no tick; `outer:` then `break outer` |
| C `goto cleanup;` for error paths | legal, but `defer` makes it unnecessary |
| C `goto` can skip initializers | compile error in Go |

## Plain break only reaches the innermost loop

The flag dance below is what you write without labels. It is correct and ugly.

```go
package main

import "fmt"

func main() {
  grid := [][]int{{1, 2, 3}, {4, 5, 6}, {7, 8, 9}}
  found := false
  for _, row := range grid {
    for _, v := range row {
      if v == 5 {
        found = true
        break
      }
    }
    if found {
      break
    }
  }
  fmt.Println("flag dance:", found)

outer:
  for i, row := range grid {
    for j, v := range row {
      if v == 5 {
        fmt.Println("labeled break at", i, j)
        break outer
      }
    }
  }
  fmt.Println("done")
}
```

Output:

```
flag dance: true
labeled break at 1 1
done
```

The label goes on its own line immediately before the statement it names, at one
tab less indentation than the loop body — `gofmt` enforces that outdent.

## continue label

`continue label` skips to the next iteration of the labelled loop, running its
post statement.

```go
package main

import "fmt"

func main() {
rows:
  for i, row := range [][]int{{1, 2, 3}, {4, -1, 6}, {7, 8, 9}} {
    for _, v := range row {
      if v < 0 {
        fmt.Println("skipping row", i)
        continue rows
      }
    }
    fmt.Println("row", i, "clean:", row)
  }
}
```

Output:

```
row 0 clean: [1 2 3]
skipping row 1
row 2 clean: [7 8 9]
```

Without the label there is no way to abandon the current row from inside the
inner loop and keep going with the next one.

## The switch-inside-a-loop problem

This is the real reason labels earn their keep. A plain `break` inside a `switch`
exits the `switch`, so the loop keeps running.

```go
package main

import "fmt"

func main() {
  for i := range 4 {
    switch {
    case i == 2:
      fmt.Println("want to stop the loop here")
      break
    }
    fmt.Println("still looping, i =", i)
  }

  fmt.Println("--- fixed with a label ---")
loop:
  for i := range 4 {
    switch {
    case i == 2:
      fmt.Println("stopping at", i)
      break loop
    }
    fmt.Println("i =", i)
  }
}
```

Output:

```
still looping, i = 0
still looping, i = 1
want to stop the loop here
still looping, i = 2
still looping, i = 3
--- fixed with a label ---
i = 0
i = 1
stopping at 2
```

Note the first loop's third line: the `break` ran, the `switch` ended, and
execution fell straight into the `fmt.Println` after it. No warning, no vet
complaint. If you have a `switch` in a loop and want to leave the loop, you need
a label or a `return`.

## Labelling a switch or select

A label can name any statement, not just a loop. Labelling the `switch` lets code
inside a nested loop jump out of the whole `switch`:

```go
package main

import "fmt"

func main() {
sw:
  switch 2 {
  case 2:
    for i := range 5 {
      if i == 3 {
        break sw
      }
      fmt.Println("i =", i)
    }
    fmt.Println("not printed")
  }
  fmt.Println("after switch")
}
```

Output:

```
i = 0
i = 1
i = 2
after switch
```

`select` has the same problem as `switch` — a plain `break` only leaves the
`select`. This is the standard shape for an event loop, covered further in
[10.6 select, default, and Timeouts](../10-concurrency-basics/06-select.md).

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  tick := time.Tick(5 * time.Millisecond)
  done := time.After(18 * time.Millisecond)
  n := 0
poll:
  for {
    select {
    case <-tick:
      n++
      fmt.Println("tick", n)
    case <-done:
      fmt.Println("done")
      break poll
    }
  }
  fmt.Println("exited the for, not just the select")
}
```

Output:

```
tick 1
tick 2
tick 3
done
exited the for, not just the select
```

The tick count depends on timer scheduling; on a loaded machine you may see two
or four.

## Labels are scoped to the function

Two functions may use the same label name with no conflict. Labels live in their
own namespace, so a label may also share a name with a variable.

```go
package main

import "fmt"

func a() {
same:
  for range 1 {
    break same
  }
  fmt.Println("a ok")
}

func b() {
same:
  for range 1 {
    break same
  }
  fmt.Println("b ok")
}

func main() {
  a()
  b()
}
```

Output:

```
a ok
b ok
```

## An unused label is a compile error

Like unused variables and imports, an unused label will not build.

```go
package main

import "fmt"

func main() {
unused:
  for i := range 2 {
    fmt.Println(i)
  }
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:1: label unused defined and not used
```

## goto

`goto label` jumps to a label in the same function. The retry loop is the pattern
you will see in old C-influenced code:

```go
package main

import "fmt"

func main() {
  n := 0
retry:
  n++
  fmt.Println("attempt", n)
  if n < 3 {
    goto retry
  }
  fmt.Println("succeeded after", n)
}
```

Output:

```
attempt 1
attempt 2
attempt 3
succeeded after 3
```

That is just a `for` loop written badly. Idiomatic: write the `for`. In
production Go, `goto` appears almost exclusively in machine-generated code and in
a handful of hot stdlib loops; you will not need it.

## goto cannot jump over a variable declaration

A forward `goto` that skips a `:=` would leave that variable in scope but
uninitialised, so the compiler refuses.

```go
package main

import "fmt"

func main() {
  goto done
  x := 5
  fmt.Println(x)
done:
  fmt.Println("done")
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:8: goto done jumps over declaration of x at ./main.go:7:5
```

The restriction is about *scope*, not about source order — a backward jump over a
declaration is fine, because the declaration is re-executed on the way through:

```go
package main

import "fmt"

func main() {
  i := 0
top:
  v := i * 2
  fmt.Println(v)
  i++
  if i < 3 {
    goto top
  }
}
```

Output:

```
0
2
4
```

## goto cannot jump into a block

The target must be in the same block as the `goto`, or in an enclosing one.

```go
package main

import "fmt"

func main() {
  goto inside
  for i := range 3 {
  inside:
    fmt.Println(i)
  }
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:8: goto inside jumps into block starting at ./main.go:7:20
```

Jumping *out* of a block is allowed, which is the one place `goto` is still
occasionally defensible: a single shared cleanup tail. In Go that job belongs to
`defer`, which is [3.5 defer: Order, Arguments, and Gotchas](05-defer.md).

## Quick Reference

| Statement | Target | Effect |
|---|---|---|
| `break` | innermost `for`/`switch`/`select` | leave it |
| `break L` | the statement labelled `L` | leave it; `L` must be a `for`, `switch`, or `select` |
| `continue` | innermost `for` | next iteration; post statement runs |
| `continue L` | the `for` labelled `L` | next iteration of that loop; `L` must be a `for` |
| `goto L` | any label in the same function | jump; restrictions below |
| `return` | the function | also runs all pending `defer`s |

| `goto` restriction | Error text |
|---|---|
| jumps forward over a `:=` | `goto L jumps over declaration of x at ...` |
| jumps into a block | `goto L jumps into block starting at ...` |
| target is a label in another function | `label L not defined` |

| Label rule | Detail |
|---|---|
| placement | its own line, immediately before the statement, outdented one tab |
| scope | the whole enclosing function body |
| namespace | separate from variables; a label may share a name with one |
| unused | compile error: `label L defined and not used` |
| duplicate in one function | compile error |
| `continue L` where `L` is a `switch` | compile error: `invalid continue label L` |

## Common Mistakes

**Using a plain `break` inside a `switch` or `select` in a loop.** It leaves only
the `switch`/`select`; label the loop and write `break label`.

**Putting the label on the same line as the loop.** It belongs on the preceding
line; `gofmt` also outdents it one tab relative to the loop body.

**Adding a label "for later" and not using it.** That is a compile error, not a
warning.

**Writing a `goto` retry loop.** Use `for` — it reads better and cannot trip the
jump-over-declaration rule.

**Reaching for `goto cleanup` out of C habit.** Use `defer`, which runs on panic
as well as on every return path.

## Exercises

1. Write a 3x3x3 triple-nested loop that finds the first triple summing to 12 and
   exits all three loops. Do it with labels, then with an extracted function and
   a `return`, and compare readability.
2. Build a loop containing a `switch` with three cases, where one case must
   `continue` the loop and another must `break` out of it. Verify both with
   printed output.
3. Deliberately trigger all three `goto` errors — jumping over a declaration,
   jumping into a block, and naming a label in another function. Record the exact
   messages.
4. Convert the `goto retry` example into a `for` loop with a max-attempts bound,
   then into a function returning `(result, error)` on exhaustion.
5. Write a `select` inside a `for` where one channel signals shutdown. Use a
   labelled `break` and then rewrite it using a `return` from an extracted
   `func run() error`. Which version is easier to test?

---

[← 3.3 Every switch Form](03-switch.md) · [Course Index](../README.md) · [3.5 defer: Order, Arguments, and Gotchas →](05-defer.md)
