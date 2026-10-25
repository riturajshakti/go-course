[← 2.3 Booleans, Operators, and Precedence](03-operators.md) · [Course Index](../README.md) · [2.5 Functions, Multiple Returns, and Named Results →](05-functions.md)

# 2.4 — Constants, Typed vs Untyped, and iota

## What You'll Learn

- Untyped constants have arbitrary precision and adopt a type at the use site
- A typed constant is as rigid as a variable, which is usually the wrong choice
- `iota` plus implicit repetition generates enums and bit flags
- Constants cannot hold a struct, slice, map, or anything a function returned

## Coming From Another Language

| Other | Go |
|---|---|
| JS `const x = []` works | a slice can never be a constant |
| JS `const` prevents rebinding | Go's `const` is a compile-time value, substituted at use |
| Python has no constants | `const` is enforced by the compiler |
| Rust `const X: i32 = 1` — always typed | Go constants are usually *untyped*, which is the whole trick |
| TS `enum` | `iota` plus a named type plus a `String()` method |

## Declaration forms

```go
package main

import "fmt"

const Greeting = "hello"

const (
  A = 1
  B = 2
)

const Typed int32 = 7

const (
  X, Y       = 1, 2
  Name       = "go"
  MaxRetries = 3
)

func main() {
  const local = 10
  const localTyped float64 = 10

  fmt.Printf("%v %T\n", Greeting, Greeting)
  fmt.Printf("%v %T\n", Typed, Typed)
  fmt.Printf("%v %T\n", local, local)
  fmt.Printf("%v %T\n", localTyped, localTyped)
  fmt.Println(A, B, X, Y, Name, MaxRetries)
}
```

Output:

```
hello string
7 int32
10 int
10 float64
1 2 1 2 go 3
```

Note `%T` on `local` prints `int`, not "untyped int". An untyped constant has no type until it is used; passing it to `fmt.Printf` *is* a use, and it gets its **default type** there — `int`, `float64`, `complex128`, `rune`, `string`, or `bool` depending on the literal's kind.

Constants live in packages and in functions, but not inside blocks that need run-time values. They may only be built from other constants and operators.

## Untyped constants have arbitrary precision

This is the part with no equivalent in most languages. A constant is not stored in a machine type during compilation — it is an exact mathematical value.

```go
package main

import "fmt"

const Huge = 1 << 100

const Big = 1 << 62

func main() {
  fmt.Println(Huge >> 98)
  fmt.Printf("%v %T\n", float64(Huge), float64(Huge))
  fmt.Println(Huge / (1 << 90))

  var f float64 = Huge
  fmt.Println(f)

  var i int64 = Big
  fmt.Println(i)

  const n = 3
  var a int8 = n
  var b float32 = n
  var c complex128 = n
  var d uint64 = n
  fmt.Printf("%T %T %T %T\n", a, b, c, d)
}
```

Output:

```
4
1.2676506002282294e+30 float64
1024
1.2676506002282294e+30
4611686018427387904
int8 float32 complex128 uint64
```

`1 << 100` does not fit in any Go integer type, yet `Huge >> 98` is computed exactly and `float64(Huge)` works because the result does fit a `float64`. The same `const n = 3` becomes an `int8`, a `float32`, a `complex128`, and a `uint64` with no conversion written anywhere.

## The overflow is reported at the use site

```go
package main

import "fmt"

const Huge = 1 << 100

func main() {
  var i int64 = Huge
  fmt.Println(i)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:17: cannot use Huge (untyped int constant 1267650600228229401496703205376) as int64 value in variable declaration (overflows)
```

Even printing it fails, because `fmt.Println` takes `...any` and the constant must become its default type `int` first:

```go
package main

import "fmt"

const Huge = 1 << 100

func main() {
  fmt.Println(Huge)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: cannot use Huge (untyped int constant 1267650600228229401496703205376) as int value in argument to fmt.Println (overflows)
```

The declaration itself is always fine. Only a use that needs a concrete type can fail — and the error quotes the exact value, which is how you know it really was carrying 31 digits.

Give it a type and the error moves to the declaration itself:
`const Big int64 = 1 << 100` fails with
`cannot use 1 << 100 (untyped int constant 1267650600228229401496703205376) as int64 value in constant declaration (overflows)`.

## Why untyped is the default you want

A typed constant is as rigid as a variable:

```go
package main

import "fmt"

const Ratio float64 = 1.5

func main() {
  var n int = 10
  fmt.Println(n * Ratio)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:15: invalid operation: n * Ratio (mismatched types int and float64)
```

Drop the type and the same constant works with every numeric type:

```go
package main

import "fmt"

const RatioUntyped = 1.5

const RatioTyped float64 = 1.5

func main() {
  var n int = 10
  var f float64 = 10

  fmt.Println(f * RatioUntyped)
  fmt.Println(f * RatioTyped)
  fmt.Println(float64(n) * RatioTyped)

  const half = 1.0 / 2.0
  var i32 int32 = 4
  fmt.Println(float64(i32) * half)

  const two = 2
  fmt.Println(i32 * two)
  fmt.Println(f * two)
  fmt.Printf("%T %T\n", i32*two, f*two)
}
```

Output:

```
15
15
15
2
8
20
int32 float64
```

The last two lines are the payoff: `two` is the same constant in both expressions and becomes `int32` in one and `float64` in the other. This is how `math.Pi` can be multiplied by a `float32` without a conversion, and how `time.Second` interoperates — though `time.Second` is in fact a *typed* constant, which is why `n * time.Second` with an `int` variable `n` fails and `n * time.Duration(1)` or `time.Duration(n) * time.Second` is needed.

Idiomatic: declare constants without a type unless you specifically want the type to be enforced at every use — which is exactly what you want for an enum.

## iota

`iota` is the index of the current ConstSpec within a `const` block, starting at 0. Combined with implicit repetition — omitting the `=` expression repeats the previous one — it generates sequences.

```go
package main

import "fmt"

const (
  Sunday = iota
  Monday
  Tuesday
  Wednesday
)

const (
  _ = iota
  January
  February
)

const (
  _  = iota
  KB = 1 << (10 * iota)
  MB
  GB
  TB
  PB
)

const (
  FlagA = 1 << iota
  FlagB
  FlagC
  FlagD
)

const (
  StatusOK = iota + 200
  StatusCreated
  StatusAccepted
)

const (
  First = iota
  Second
)

const (
  Alpha = iota
  Beta
)

const (
  Even = iota * 2
  _
  Four
  Six
)

func main() {
  fmt.Println(Sunday, Monday, Tuesday, Wednesday)
  fmt.Println(January, February)
  fmt.Println(KB, MB, GB, TB, PB)
  fmt.Printf("%04b %04b %04b %04b\n", FlagA, FlagB, FlagC, FlagD)
  fmt.Println(StatusOK, StatusCreated, StatusAccepted)
  fmt.Println(First, Second, Alpha, Beta)
  fmt.Println(Even, Four, Six)
}
```

Output:

```
0 1 2 3
1 2
1024 1048576 1073741824 1099511627776 1125899906842624
0001 0010 0100 1000
200 201 202
0 1 0 1
0 4 6
```

Reading each block:

- **Basic.** `Sunday` is 0. Each later line repeats `= iota`, and `iota` has advanced.
- **Skip with `_`.** Burning index 0 makes `January` equal 1.
- **Size units.** `KB` is at `iota == 1`, so `1 << 10`. The repeated expression re-evaluates `iota` each line, giving `1<<20`, `1<<30`, and so on.
- **Bit flags.** `1 << iota` gives one distinct bit per line.
- **Offset.** `iota + 200` starts the sequence at 200.
- **Reset.** `iota` restarts at 0 in every `const` block, so `Alpha` is 0 again.
- **Gaps.** A bare `_` consumes an index, so `Four` lands at `iota == 2`.

The important subtlety: `iota` counts **lines in the block**, not the number of constants assigned, and it increments even on lines that only repeat the previous expression or assign to `_`.

## Typed constant enums with a String method

```go
package main

import "fmt"

type Level int

const (
  Debug Level = iota
  Info
  Warn
  Error
)

func (l Level) String() string {
  switch l {
  case Debug:
    return "DEBUG"
  case Info:
    return "INFO"
  case Warn:
    return "WARN"
  case Error:
    return "ERROR"
  }
  return fmt.Sprintf("Level(%d)", int(l))
}

func main() {
  fmt.Println(Debug, Info, Warn, Error)
  fmt.Println(Level(9))
  fmt.Printf("%v %d %s\n", Warn, Warn, Warn)

  var l Level = 2
  fmt.Println(l == Warn, l < Error)

  levels := []Level{Error, Debug}
  fmt.Println(levels)
}
```

Output:

```
DEBUG INFO WARN ERROR
Level(9)
WARN 2 WARN
true true
[ERROR DEBUG]
```

Here the type is deliberate. `Debug Level = iota` types the whole block, and the rigidity becomes a feature:

```go
package main

import "fmt"

type Level int

const Info Level = 1

func main() {
  n := 1
  fmt.Println(n == Info)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:11:20: invalid operation: n == Info (mismatched types int and Level)
```

A plain `int` cannot be compared to a `Level`, so you cannot pass a bare number where a level is expected. Note `%d` still works because `Level`'s underlying type is `int`, and `fmt` reaches through to it.

Idiomatic: write the `String()` method by hand for small enums, or generate it with `stringer` (`//go:generate stringer -type=Level`) for large ones. See 15.8. This is also why `Level(9)` must return something sensible rather than panicking — `fmt` calls `String()` on values you never declared.

## What a constant cannot be

```go
package main

const Primes = []int{2, 3, 5}

func main() {
  _ = Primes
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:3:16: []int{…} (value of type []int) is not constant
```

```go
package main

type Point struct{ X, Y int }

const Origin = Point{0, 0}

func main() {
  _ = Origin
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:5:16: Point{…} (value of struct type Point) is not constant
```

A map gets the matching message, `map[string]int{…} (value of type map[string]int)
is not constant`. Constants are limited to the *basic* types: booleans, strings, runes, integers, floats, and complex numbers. There is no way to make an immutable slice, map, or struct in Go. The convention is a package-level `var` with a comment or a function that returns a fresh copy.

Nor can a constant come from a function call, even a pure one —
`const Upper = strings.ToUpper("go")` gives
`strings.ToUpper("go") (value of type string) is not constant`. Nor from a variable:

```go
package main

var n = 10

const Doubled = n * 2

func main() {
  _ = Doubled
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:5:17: n * 2 (value of type int) is not constant
```

## Which expressions are constant

`len` and `cap` of a constant string, `unsafe.Sizeof`, `real`, `imag`, and `complex` on constants, plus all the arithmetic operators, are evaluated at compile time.

```go
package main

import (
  "fmt"
  "unsafe"
)

const Name = "gopher"

const NameLen = len(Name)

const Computed = 60 * 60 * 24 * 365

const Mask = 1<<16 - 1

const Sized = unsafe.Sizeof(int64(0))

func main() {
  var arr [NameLen]byte
  fmt.Println(NameLen, len(arr))
  fmt.Println(Computed, Mask, Sized)

  s := "gopher"
  fmt.Println(len(s))

  const emptyLen = len("")
  fmt.Println(emptyLen)

  fmt.Printf("%T %T %T\n", NameLen, Mask, Sized)
}
```

Output:

```
6 6
31536000 65535 8
6
0
int int uintptr
```

`NameLen` works as an array length, which is itself proof that it is constant — array lengths must be. `len` of a *variable* string is not constant:

```go
package main

var s = "gopher"

const L = len(s)

func main() {
  _ = L
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:5:11: len(s) (value of type int) is not constant
```

And this is the error you get for a non-constant array length, which is how Go tells you it has no variable-length arrays (that is what slices are for):

```go
package main

import "fmt"

func main() {
  n := 5
  var arr [n]int
  fmt.Println(arr)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:3: declared and not used: n
./main.go:7:12: invalid array length n
```

## Quick Reference

| Form | Meaning |
|---|---|
| `const X = v` | untyped constant |
| `const X T = v` | typed constant; rigid like a variable |
| `const ( A = 1 \n B = 2 )` | a block |
| `const ( A, B = 1, 2 )` | multiple in one spec |
| `const ( A = iota \n B \n C )` | implicit repetition; `A`=0, `B`=1, `C`=2 |
| `const ( _ = iota \n A )` | skip an index; `A`=1 |
| `const ( A = 1 << iota \n B )` | bit flags |
| `const ( A = iota + N \n B )` | offset start |
| `const ( A T = iota \n B )` | typed enum; `T` applies to the whole block |

| Default type of an untyped constant | Literal kind |
|---|---|
| `int` | `42`, `0x1f`, `1<<10` |
| `rune` (`int32`) | `'A'` |
| `float64` | `1.5`, `1e3` |
| `complex128` | `2i` |
| `string` | `"x"` |
| `bool` | `true` |

| Allowed in a constant | Not allowed |
|---|---|
| bool, string, rune, int, float, complex | slice, map, struct, array, pointer, interface, chan, func |
| arithmetic, bitwise, comparison, shifts | any function call except the listed builtins |
| `len`/`cap` of a constant string | `len` of a variable |
| `unsafe.Sizeof`, `real`, `imag`, `complex`, `min`, `max` | anything reading a variable |
| other constants | |

## Common Mistakes

**Typing a constant out of habit (`const Timeout int64 = 30`).** It then refuses to mix with other numeric types; leave it untyped unless you want the enforcement.

**Expecting `const xs = []int{1,2}` to work.** Slices can never be constants; use a package-level `var`, and remember callers can mutate it.

**Assuming `iota` counts declared constants.** It counts lines in the block, including bare `_` lines, and it resets to 0 in every new block.

**Forgetting the default case in an enum's `String()`.** `fmt` will call it on out-of-range values and print an empty string; return `Type(N)` instead.

**Mixing an `int` variable with a typed constant like `time.Second`.** Convert explicitly: `time.Duration(n) * time.Second`.

## Exercises

1. Write a `ByteSize float64` type with `KB` through `EB` generated by one `1 << (10 * iota)` block, and a `String()` method that prints the largest fitting unit with two decimals. Check `ByteSize(1500)`, `ByteSize(1<<40)`, and `ByteSize(0)`.
2. Declare `const MaxUint64Plus1 = 1 << 64`. Show three uses that compile and three that do not, and for each failure record the exact compiler message.
3. Build a `Weekday` enum with a `String()`, a `Parse(string) (Weekday, error)`, and a `Valid() bool`. Make the invalid case impossible to construct from outside the package without a conversion.
4. Write two `const` blocks that produce the same values, one with `iota` and one with explicit numbers. Then insert a new constant in the middle of each and describe which one you would rather maintain.
5. Find a constant expression using only `len`, `unsafe.Sizeof`, and arithmetic that is valid as an array length and evaluates to the number of bytes in four `int64` values. Prove it is constant by using it as an array length.

---

[← 2.3 Booleans, Operators, and Precedence](03-operators.md) · [Course Index](../README.md) · [2.5 Functions, Multiple Returns, and Named Results →](05-functions.md)
