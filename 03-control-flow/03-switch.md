[← 3.2 Every for Loop Form](02-for-loops.md) · [Course Index](../README.md) · [3.4 goto, Labels, break, and continue →](04-labels-goto.md)

# 3.3 — Every switch Form

## What You'll Learn

- Cases do **not** fall through; `break` is implicit and almost never written
- `switch { case cond: }` with no subject is the idiomatic `if/else if` chain
- Cases take multiple comma-separated values, constants are checked for duplicates, and `default` may appear anywhere
- `break` inside a `switch` leaves the `switch`, not the enclosing loop

## Coming From Another Language

| Other | Go |
|---|---|
| JS/C fall through unless you `break` | each case breaks; `fallthrough` is opt-in |
| JS `case 1: case 2:` for grouping | `case 1, 2:` |
| JS `switch` on anything | expression switch requires comparable, same-typed cases |
| Python `match`/`case` with patterns | no destructuring; a type switch is the closest thing |
| Rust `match` is exhaustive | Go's is not; add `default` yourself |
| JS/C `switch(true)` hack | `switch { case cond: }` is first-class syntax |

## The expression switch

```go
package main

import "fmt"

func name(n int) string {
  switch n {
  case 1:
    return "one"
  case 2:
    return "two"
  case 3:
    return "three"
  default:
    return "many"
  }
}

func main() {
  for _, n := range []int{1, 2, 3, 9} {
    fmt.Println(n, name(n))
  }
}
```

Output:

```
1 one
2 two
3 three
9 many
```

No `break`. There is no implicit fall through, so a case body ends at the next
`case`. This is the single biggest difference from C and JavaScript and it means a
missing `break` can never be a bug in Go.

## Multiple values per case

```go
package main

import "fmt"

func main() {
  for _, c := range []rune{'a', 'e', 'z', '7', ' '} {
    switch c {
    case 'a', 'e', 'i', 'o', 'u':
      fmt.Printf("%q vowel\n", c)
    case '0', '1', '2', '3', '4', '5', '6', '7', '8', '9':
      fmt.Printf("%q digit\n", c)
    default:
      fmt.Printf("%q other\n", c)
    }
  }
}
```

Output:

```
'a' vowel
'e' vowel
'z' other
'7' digit
' ' other
```

## switch with no subject

Omit the expression and each case is a `bool`. This replaces the `if/else if`
chain and is extremely common in real Go.

```go
package main

import "fmt"

func grade(score int) string {
  switch {
  case score >= 90:
    return "A"
  case score >= 80:
    return "B"
  case score >= 70:
    return "C"
  default:
    return "F"
  }
}

func main() {
  for _, s := range []int{95, 83, 71, 20} {
    fmt.Println(s, grade(s))
  }
}
```

Output:

```
95 A
83 B
71 C
20 F
```

Cases are evaluated top to bottom and the first true one wins, exactly like
`else if`. `switch { ... }` is shorthand for `switch true { ... }`, and both
compile to the same thing:

```go
package main

import "fmt"

func main() {
  switch true {
  case 1 > 2:
    fmt.Println("no")
  case 2 > 1:
    fmt.Println("switch true is the same as bare switch")
  }
}
```

Output:

```
switch true is the same as bare switch
```

Idiomatic: write the bare `switch {`. Nobody writes `switch true`.

## switch with an init statement

Same shape as `if`: one simple statement, a semicolon, then the optional subject.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  switch h := time.Date(2026, 10, 25, 14, 0, 0, 0, time.UTC).Hour(); {
  case h < 12:
    fmt.Println("morning", h)
  case h < 18:
    fmt.Println("afternoon", h)
  default:
    fmt.Println("evening", h)
  }

  switch n := 3 * 3; n {
  case 9:
    fmt.Println("nine", n)
  default:
    fmt.Println("other", n)
  }
}
```

Output:

```
afternoon 14
nine 9
```

Note the first one: `switch h := ...; {` — init statement, no subject. The
trailing semicolon before `{` is required to mark the missing expression.

## fallthrough

`fallthrough` must be the last statement in a case body. It transfers to the next
case body **without evaluating that case's condition**.

```go
package main

import "fmt"

func main() {
  switch 2 {
  case 1:
    fmt.Println("one")
  case 2:
    fmt.Println("two")
  case 3:
    fmt.Println("three")
  }
  fmt.Println("--- with fallthrough ---")
  switch 2 {
  case 1:
    fmt.Println("one")
  case 2:
    fmt.Println("two")
    fallthrough
  case 3:
    fmt.Println("three")
    fallthrough
  case 99:
    fmt.Println("ninety-nine (condition never checked)")
  default:
    fmt.Println("default reached by fallthrough")
  }
}
```

Output:

```
two
--- with fallthrough ---
two
three
ninety-nine (condition never checked)
```

The `default` was not reached: `case 99` has no `fallthrough`, so the chain stops
there. `99 != 2` was never tested. A `fallthrough` in the last clause has nowhere
to go:

```go
package main

import "fmt"

func main() {
  switch 1 {
  case 1:
    fmt.Println("one")
  default:
    fmt.Println("last clause")
    fallthrough
  }
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:11:5: cannot fallthrough final case in switch
```

Idiomatic: you will go years without writing `fallthrough`. Group values with
`case a, b:` instead; reach for `fallthrough` only for genuinely cumulative
logic, such as a lexer accumulating character classes.

## default can go anywhere

Position does not affect behaviour — `default` is tried only after every other
case has failed.

```go
package main

import "fmt"

func main() {
  switch 1 {
  default:
    fmt.Println("default in the middle")
  case 1:
    fmt.Println("one")
  case 2:
    fmt.Println("two")
  }

  switch 7 {
  case 1:
    fmt.Println("one")
  default:
    fmt.Println("default in the middle, 7 landed here")
  case 2:
    fmt.Println("two")
  }

  switch 5 {
  case 5:
  case 6:
    fmt.Println("six")
  }
  fmt.Println("empty case body does nothing and does NOT fall through")
}
```

Output:

```
one
default in the middle, 7 landed here
empty case body does nothing and does NOT fall through
```

An empty case body is the way to say "this value is handled by doing nothing" —
in C you would need a bare `break`. Idiomatic: put `default` last anyway, for
readers.

## break in a switch breaks the switch

An explicit `break` exits the `switch` early. Inside a loop, that is almost never
what the author meant.

```go
package main

import "fmt"

func main() {
  for i := range 4 {
    switch {
    case i == 2:
      fmt.Println("breaking the switch, not the loop")
      break
    default:
      fmt.Println("i =", i)
    }
  }
  fmt.Println("loop ran all four times")
}
```

Output:

```
i = 0
i = 1
breaking the switch, not the loop
i = 3
loop ran all four times
```

The loop kept going. To exit the loop you must label it — see
[3.4 goto, Labels, break, and continue](04-labels-goto.md). A `break` that is the
last statement of a case is pure noise; delete it.

A `break` is useful inside a case to skip the rest of a long body:

```go
package main

import "fmt"

func main() {
  n := 5
  switch {
  case n > 3:
    fmt.Println("big")
    if n > 4 {
      break
    }
    fmt.Println("only when n is exactly 4")
  }
  fmt.Println("after")
}
```

Output:

```
big
after
```

## Duplicate constant cases are a compile error

The compiler checks for duplicates among *constant* cases.

```go
package main

import "fmt"

func main() {
  switch 1 {
  case 1:
    fmt.Println("a")
  case 1:
    fmt.Println("b")
  }
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:8: duplicate case 1 (constant of type int) in expression switch
	./main.go:7:8: previous case
```

Strings too:

```go
package main

import "fmt"

func main() {
  s := "go"
  switch s {
  case "go":
    fmt.Println("gopher")
  case "go":
    fmt.Println("dup")
  }
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:8: duplicate case "go" (constant of type string) in expression switch
	./main.go:8:8: previous case
```

Variables are *not* checked, because their values are not known at compile time —
the first matching case simply wins at runtime:

```go
package main

import "fmt"

func main() {
  n := 1
  m := 1
  switch 1 {
  case n:
    fmt.Println("n")
  case m:
    fmt.Println("m")
  }
}
```

Output:

```
n
```

## The type switch, briefly

`switch v := x.(type)` dispatches on the dynamic type of an interface value. Full
treatment in [5.3 Type Assertions and Type Switches](../05-interfaces/03-assertions-type-switches.md).

```go
package main

import "fmt"

func describe(v any) string {
  switch x := v.(type) {
  case nil:
    return "nil"
  case int:
    return fmt.Sprintf("int %d", x+1)
  case string:
    return fmt.Sprintf("string of len %d", len(x))
  case []int, []string:
    return fmt.Sprintf("a slice: %v", x)
  case error:
    return "error: " + x.Error()
  default:
    return fmt.Sprintf("unknown %T", x)
  }
}

func main() {
  fmt.Println(describe(nil))
  fmt.Println(describe(7))
  fmt.Println(describe("hey"))
  fmt.Println(describe([]int{1, 2}))
  fmt.Println(describe(fmt.Errorf("boom")))
  fmt.Println(describe(3.5))
}
```

Output:

```
nil
int 8
string of len 3
a slice: [1 2]
error: boom
unknown float64
```

In a single-type case, `x` has that concrete type (`x+1` works because `x` is an
`int`). In the multi-type `case []int, []string:` it keeps the interface type, so
only interface operations are available. `case nil` matches a nil interface
value. `fallthrough` is not allowed at all:

```
./main.go:10:3: cannot fallthrough in type switch
```

## Quick Reference

| Form | Notes |
|---|---|
| `switch x { case a: }` | expression switch; `x` compared with `==` |
| `switch { case cond: }` | no subject; cases are `bool`. The `if/else if` replacement |
| `switch true { case cond: }` | identical to the above; never written |
| `switch init; x { }` | one simple statement before the subject |
| `switch init; { }` | init statement with no subject — note the `;` |
| `case a, b, c:` | matches any of the listed values |
| `case a:` with empty body | matches and does nothing; no fall through |
| `default:` | any position; tried last regardless |
| `fallthrough` | last statement in a case; skips the next case's condition |
| `break` | exits the `switch`, **not** an enclosing loop |
| `break Label` | exits the labelled statement — see 3.4 |
| `switch v := x.(type) { }` | type switch; see 5.3 |

| Rule | Behaviour |
|---|---|
| implicit break | always, at the end of every case body |
| duplicate constant cases | compile error, with the previous case's line |
| duplicate variable cases | allowed; first match wins |
| no case matches, no default | the whole `switch` does nothing |
| `fallthrough` in the final clause | compile error |
| `fallthrough` in a type switch | compile error |
| case value type | must be assignable to the subject's type |

## Common Mistakes

**Writing `break` at the end of every case out of C habit.** It is implicit;
`gofmt` will not remove the redundant keyword but reviewers will.

**Using `break` in a `switch` inside a loop to exit the loop.** It leaves only the
`switch`; label the loop and write `break label`.

**Expecting a `switch` to be exhaustive like a Rust `match`.** It is not; add a
`default` that panics or returns an error for unhandled values.

**Reaching for `fallthrough` to match two values.** Use `case a, b:` — the
`fallthrough` version also skips the next case's condition, which is rarely what
you want.

**Assuming `switch` on a struct or interface compares deeply.** It uses `==`, so
non-comparable types (slices, maps, funcs) are a compile error and interface
comparison can panic at runtime.

## Exercises

1. Rewrite a four-branch `if/else if` chain as a bare `switch` and confirm the
   behaviour is identical for every boundary value.
2. Write `func httpClass(code int) string` returning `"info"`, `"ok"`,
   `"redirect"`, `"client error"`, `"server error"`, using a bare `switch` with
   range comparisons. Then rewrite it with `case 200, 201, 204:`-style value
   lists and note which version you would review more happily.
3. Build a `switch` whose cases are two `const` values that happen to be equal
   (`const a = 2; const b = 1 + 1`). Record whether the duplicate check fires, and
   then repeat with `var`.
4. Write a loop containing a `switch` where one case must exit the loop. Solve it
   three ways: a boolean flag, a labelled `break`, and extracting the loop into a
   function that `return`s. Pick a favourite and say why.
5. Write a `fallthrough` chain of four cases where each case appends to a
   `[]string`, and verify that a mid-chain case whose condition is false still
   runs its body.

---

[← 3.2 Every for Loop Form](02-for-loops.md) · [Course Index](../README.md) · [3.4 goto, Labels, break, and continue →](04-labels-goto.md)
