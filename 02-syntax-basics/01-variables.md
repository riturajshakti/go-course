[← 1.6 How Go Thinks: A Mental Model](../01-getting-started/06-mental-model.md) · [Course Index](../README.md) · [2.2 Numeric Types, Overflow, and Conversion →](02-numeric-types.md)

# 2.1 — Variables, Zero Values, and Declaration Forms

## What You'll Learn

- The six declaration forms and which one is idiomatic where
- Every type's zero value, and why Go has no `undefined`
- An unused local variable is a compile error, not a warning
- Shadowing is legal, silent, and the source of the `err` bug you will hit

## Coming From Another Language

| Other | Go |
|---|---|
| JS `let x;` → `undefined` | `var x int` → `0`; there is no `undefined` |
| Python `x = None` as "not set" | the zero value is the "not set" state; `nil` only for pointer-like kinds |
| Rust `let x: i32;` then use → error | `var x int` is immediately usable at `0` |
| TS unused local → lint warning | unused local → **compile error** |
| JS `const` prevents rebinding | Go's `const` is compile-time only; see 2.4 |

## Every declaration form

```go
package main

import "fmt"

func main() {
  var a int
  var b = 42
  var c int = 7
  d := "short"

  var (
    e float64
    f        = true
    g string = "grouped"
  )

  var h, i int = 1, 2
  j, k := 3.5, "mixed"

  fmt.Println(a, b, c, d)
  fmt.Println(e, f, g)
  fmt.Println(h, i, j, k)
}
```

Output:

```
0 42 7 short
0 true grouped
1 2 3.5 mixed
```

- `var a int` — zero value, type explicit.
- `var b = 42` — type inferred from the value (`int`).
- `var c int = 7` — both stated; redundant unless you want a non-default type.
- `d := "short"` — the short form, inside functions only.
- `var (...)` — a grouped block, the only way to declare several package-level vars together.

Idiomatic: `:=` inside functions; `var x T` when you want the zero value; `var` blocks at package level.

## := works only inside a function

```go
package main

import "fmt"

x := 10

func main() {
  fmt.Println(x)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:5:1: syntax error: non-declaration statement outside function body
```

Package level needs `var x = 10`. The parser is not even looking for an expression there.

## An unused local variable is an error

```go
package main

import "fmt"

func main() {
  count := 10
  fmt.Println("hello")
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:3: declared and not used: count
```

This applies to locals only — unused package-level vars, unused struct fields, and unused function parameters are all fine. Unused *imports* are also an error. During debugging, `_ = count` silences it.

## := needs at least one new variable

```go
package main

import "fmt"

func main() {
  x := 1
  x := 2
  fmt.Println(x)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:5: no new variables on left side of :=
```

But a `:=` with a *mix* of new and existing names is legal, and only assigns to the existing one:

```go
package main

import "fmt"

func divmod(a, b int) (int, int) {
  return a / b, a % b
}

func main() {
  a, b := 1, 2
  a, b = b, a
  fmt.Println(a, b)

  x, y, z := 1, 2, 3
  x, y, z = z, x, y
  fmt.Println(x, y, z)

  q, _ := divmod(17, 5)
  _, r := divmod(17, 5)
  fmt.Println(q, r)

  var n int
  n, m := 10, 20
  fmt.Println(n, m)
}
```

Output:

```
2 1
3 1 2
3 2
10 20
```

Multiple assignment evaluates the whole right-hand side first, so `a, b = b, a` swaps with no temporary and `x, y, z = z, x, y` rotates. The last pair is the mixed case: `m` is new, `n` already existed, so the `:=` is allowed and `n` is merely assigned.

## var redeclared in the same block

```go
package main

import "fmt"

func main() {
  var x int
  var x string
  fmt.Println(x)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:7: x redeclared in this block
	./main.go:6:7: other declaration of x
```

## The blank identifier is write-only

`_` discards a value. It is not a variable: you cannot read it back.

```go
package main

import "fmt"

func main() {
  _ = 1
  fmt.Println(_)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:15: cannot use _ as value or type
```

`_` is also how you satisfy must-use rules: `_, err := f()`, `var _ io.Writer = (*myType)(nil)` for a compile-time interface check, and `import _ "net/http/pprof"` for an import-for-side-effects.

## Zero values for every kind

There is no uninitialized memory in Go. Every declaration without a value gets the type's zero value, recursively.

```go
package main

import "fmt"

type Inner struct {
  N int
  S string
}

type Outer struct {
  I   Inner
  P   *Inner
  Arr [2]int
  Sl  []int
  M   map[string]int
}

func main() {
  var (
    i   int
    f   float64
    c   complex128
    s   string
    b   bool
    r   rune
    by  byte
    p   *int
    sl  []int
    m   map[string]int
    ch  chan int
    fn  func()
    any interface{}
    arr [3]int
    st  Outer
  )

  fmt.Printf("int      %d\n", i)
  fmt.Printf("float64  %v\n", f)
  fmt.Printf("complex  %v\n", c)
  fmt.Printf("string   %q\n", s)
  fmt.Printf("bool     %v\n", b)
  fmt.Printf("rune     %d\n", r)
  fmt.Printf("byte     %d\n", by)
  fmt.Printf("*int     %v\n", p)
  fmt.Printf("[]int    %v %v %d\n", sl, sl == nil, len(sl))
  fmt.Printf("map      %v %v %d\n", m, m == nil, len(m))
  fmt.Printf("chan     %v %v\n", ch, ch == nil)
  fmt.Printf("func     %v\n", fn == nil)
  fmt.Printf("any      %v %v\n", any, any == nil)
  fmt.Printf("[3]int   %v\n", arr)
  fmt.Printf("struct   %+v\n", st)
}
```

Output:

```
int      0
float64  0
complex  (0+0i)
string   ""
bool     false
rune     0
byte     0
*int     <nil>
[]int    [] true 0
map      map[] true 0
chan     <nil> true
func     true
any      <nil> true
[3]int   [0 0 0]
struct   {I:{N:0 S:} P:<nil> Arr:[0 0] Sl:[] M:map[]}
```

| Kind | Zero value |
|---|---|
| all numeric types | `0` (`0+0i` for complex) |
| `string` | `""` — not nil; there is no nil string |
| `bool` | `false` |
| pointer, slice, map, channel, func, interface | `nil` |
| array | an array of its element's zero values |
| struct | a struct of its fields' zero values, recursively |

The practical consequence: a nil slice is usable. `len`, `cap`, `range`, and `append` all work on it. A nil map is readable (returns the value's zero) but writing to it panics. A nil `string` does not exist — that is `""`.

## The zero value is meant to be useful

Many stdlib types are designed so their zero value is a working instance, which is why you see declarations with no constructor.

```go
package main

import (
  "bytes"
  "fmt"
  "strings"
  "sync"
)

func main() {
  var buf bytes.Buffer
  buf.WriteString("usable")
  fmt.Println(buf.String())

  var sb strings.Builder
  sb.WriteString("also usable")
  fmt.Println(sb.String())

  var mu sync.Mutex
  mu.Lock()
  mu.Unlock()
  fmt.Println("mutex locked and unlocked")

  var wg sync.WaitGroup
  wg.Add(1)
  go func() { defer wg.Done() }()
  wg.Wait()
  fmt.Println("waitgroup done")
}
```

Output:

```
usable
also usable
mutex locked and unlocked
waitgroup done
```

## Type mismatch at declaration

```go
package main

import "fmt"

func main() {
  var s string = 42
  fmt.Println(s)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:18: cannot use 42 (untyped int constant) as string value in variable declaration
```

## Scope: every block is a scope

```go
package main

import "fmt"

func main() {
  x := "outer"
  fmt.Println(x)
  {
    x := "block"
    fmt.Println(x)
  }
  fmt.Println(x)

  for i := 0; i < 2; i++ {
    x := fmt.Sprintf("loop-%d", i)
    fmt.Println(x)
  }
  fmt.Println(x)

  if x := 99; x > 10 {
    fmt.Println("if-scoped x:", x)
  }
  fmt.Println(x)
}
```

Output:

```
outer
block
outer
loop-0
loop-1
outer
if-scoped x: 99
outer
```

A bare `{ }` is a block. `if`, `for`, and `switch` each open one, and the init statement of `if x := 99;` declares into that block — the variable is dead after the closing brace. See 3.1 for why that form is idiomatic.

## Shadowing: the bug Go will not warn you about

`:=` in a nested block creates a *new* variable even when the outer one has the same name. With `err`, that silently throws away the error.

```go
package main

import (
  "errors"
  "fmt"
)

func find(k string) (string, error) {
  if k == "" {
    return "", errors.New("empty key")
  }
  return "v:" + k, nil
}

func main() {
  var err error

  if v, err := find("a"); err == nil {
    fmt.Println("inner ok:", v)
  }
  fmt.Println("outer err:", err)

  v, err := find("")
  fmt.Println(v, err)
}
```

Output:

```
inner ok: v:a
outer err: <nil>
 empty key
```

The `err` assigned inside the `if` is a different variable from the outer `err`, which stays `nil` forever. The compiler is happy because the inner `err` *is* used (in the condition). This is the single most common Go bug of this kind, and it compiles clean.

`go vet` does not ship a shadow analyzer by default — it lives in `golang.org/x/tools/go/analysis/passes/shadow` and must be installed and wired in as a `-vettool`. Most teams rely on `golangci-lint` with the `govet` shadow check enabled instead (see 15.3). Until then, the defence is habit: never introduce a second `err` in a nested scope you expect to inspect from outside.

## Quick Reference

| Form | Where | Notes |
|---|---|---|
| `var x T` | package or function | zero value |
| `var x = v` | package or function | type inferred |
| `var x T = v` | package or function | explicit both; use for a non-default type |
| `x := v` | function only | the short form |
| `var ( ... )` | package or function | grouped |
| `var x, y T = a, b` | package or function | multiple with one type |
| `x, y := a, b` | function only | multiple, each inferred |
| `x, y = y, x` | function only | simultaneous assignment |
| `_ = v` | function only | discard |

| Rule | Consequence |
|---|---|
| Unused local variable | compile error |
| Unused import | compile error |
| Unused package-level var, field, or param | legal |
| `:=` with no new name on the left | compile error |
| `:=` with at least one new name | legal; existing names are assigned |
| `:=` in a nested block | new variable; shadows silently |

## Common Mistakes

**Using `:=` for `err` inside an `if` or `for` and reading `err` after the block.** The inner `err` is a different variable; declare `err` once with `var` and use plain `=` inside.

**Reaching for `:=` at package level.** It is a statement, not a declaration; use `var x = v`.

**Expecting an unused variable to be a warning.** It stops the build; delete it or assign it to `_`.

**Writing to a nil map because the zero value looked usable.** Reads work and return the zero value, but writes panic; build it with `make` or a literal.

**Writing `var s string = ""` or `var n int = 0`.** The zero value is already that; write `var s string`.

## Exercises

1. Write one program that declares the same `int` variable six different ways (six names), prints all six, and passes `gofmt -l` with no output.
2. Declare one variable of each of the fifteen kinds in the zero-value table without using a literal, and print whether each `== nil` where that comparison is legal. Explain in a comment-free way (prose in your notes) why `string` and `int` are not in the nil-comparable set.
3. Write a function `load(path string) ([]byte, error)` and a `main` that calls it inside an `if` with `:=`, then prints the outer `err`. Observe the bug. Fix it two ways: with `var err error` plus `=`, and by moving the call out of the `if`.
4. Take a struct with a `[]int` field, a `map[string]int` field, and a `*struct` field. Declare it with `var`, then `range` the slice, read a missing map key, and read `len` of both — all without initializing anything. Find the one operation that panics.
5. Chain four nested blocks that each shadow `n` with a different value and print `n` on the way in and on the way out of each. Predict the eight lines before running it.

---

[← 1.6 How Go Thinks: A Mental Model](../01-getting-started/06-mental-model.md) · [Course Index](../README.md) · [2.2 Numeric Types, Overflow, and Conversion →](02-numeric-types.md)
