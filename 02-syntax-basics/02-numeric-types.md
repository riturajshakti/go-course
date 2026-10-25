[← 2.1 Variables, Zero Values, and Declaration Forms](01-variables.md) · [Course Index](../README.md) · [2.3 Booleans, Operators, and Precedence →](03-operators.md)

# 2.2 — Numeric Types, Overflow, and Conversion

## What You'll Learn

- All fifteen numeric types, plus `byte` and `rune` as aliases
- Signed integer overflow **wraps silently** — no panic, no error
- Go never converts between numeric types implicitly, not even `int` to `int64`
- Division by zero is a compile error for constants and a panic for variables

## Coming From Another Language

| Other | Go |
|---|---|
| JS one `number` (float64) | 15 numeric types; `int` is the default |
| JS `1/0 === Infinity` | `1/0` on *constants* is a compile error; on floats it is `+Inf` |
| Python ints are arbitrary precision | `int` is 64-bit and wraps on overflow |
| Python `7/2 == 3.5` | `7/2 == 3`; integer division truncates |
| Rust debug build panics on overflow | Go always wraps, in every build mode |
| Rust `as` casts | `T(x)` conversions, required even between int sizes |

## Every numeric type

```go
package main

import (
  "fmt"
  "math"
  "unsafe"
)

func main() {
  fmt.Println("int size bytes:", unsafe.Sizeof(int(0)))
  fmt.Println("uint size bytes:", unsafe.Sizeof(uint(0)))
  fmt.Println("uintptr size bytes:", unsafe.Sizeof(uintptr(0)))
  fmt.Println("MaxInt:", math.MaxInt)
  fmt.Println("MinInt:", math.MinInt)
  fmt.Println("MaxInt64:", math.MaxInt64)
  fmt.Println("MaxUint64:", uint64(math.MaxUint64))

  fmt.Println()
  fmt.Println("int8 ", math.MinInt8, math.MaxInt8)
  fmt.Println("int16", math.MinInt16, math.MaxInt16)
  fmt.Println("int32", math.MinInt32, math.MaxInt32)
  fmt.Println("int64", math.MinInt64, math.MaxInt64)
  fmt.Println("uint8 ", 0, math.MaxUint8)
  fmt.Println("uint16", 0, math.MaxUint16)
  fmt.Println("uint32", 0, uint32(math.MaxUint32))

  fmt.Println()
  var b byte = 'A'
  var r rune = '世'
  fmt.Printf("byte %T %d %c\n", b, b, b)
  fmt.Printf("rune %T %d %c %U\n", r, r, r, r)

  var f32 float32 = 1.0 / 3.0
  var f64 float64 = 1.0 / 3.0
  fmt.Println(f32, f64)

  var c64 complex64 = 1 + 2i
  c128 := complex(3.0, 4.0)
  fmt.Println(c64, c128, real(c128), imag(c128), cmplxAbs(c128))
}

func cmplxAbs(c complex128) float64 {
  return math.Hypot(real(c), imag(c))
}
```

Output on darwin/arm64 (a 64-bit platform):

```
int size bytes: 8
uint size bytes: 8
uintptr size bytes: 8
MaxInt: 9223372036854775807
MinInt: -9223372036854775808
MaxInt64: 9223372036854775807
MaxUint64: 18446744073709551615

int8  -128 127
int16 -32768 32767
int32 -2147483648 2147483647
int64 -9223372036854775808 9223372036854775807
uint8  0 255
uint16 0 65535
uint32 0 4294967295

byte uint8 65 A
rune int32 19990 世 U+4E16
0.33333334 0.3333333333333333
(1+2i) (3+4i) 3 4 5
```

`int`, `uint`, and `uintptr` are platform-dependent: 64-bit on amd64/arm64, 32-bit on 386/arm. The sized types are fixed everywhere. `byte` is an *alias* for `uint8` and `rune` is an alias for `int32` — note `%T` prints the underlying name, because an alias is not a distinct type (see 4.6).

Idiomatic: use `int` for counts, indices, and loop variables. Reach for a sized type only when a wire format, a file format, or memory pressure demands it.

## Overflow wraps, silently

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  var i8 int8 = 127
  i8++
  fmt.Println("int8 127+1 =", i8)

  var u8 uint8 = 0
  u8--
  fmt.Println("uint8 0-1 =", u8)

  var i32 int32 = math.MaxInt32
  fmt.Println("int32 max*2 =", i32*2)

  var u16 uint16 = 60000
  fmt.Println("uint16 60000+10000 =", u16+10000)

  n := math.MaxInt
  fmt.Println("int max+1 =", n+1)
}
```

Output:

```
int8 127+1 = -128
uint8 0-1 = 255
int32 max*2 = -2
uint16 60000+10000 = 4464
int max+1 = -9223372036854775808
```

This is defined behaviour, not undefined: Go specifies two's-complement wraparound for signed and unsigned alike. Unlike Rust, there is no debug-mode check. If you need detection, compare before the operation or use `math/bits` (`bits.Add64` returns a carry).

The `u8--` case matters most in practice: an unsigned loop counter decremented past zero becomes a huge number, and `for i := uint(len(s)-1); i >= 0; i--` never terminates.

## Integer division truncates; % follows the dividend's sign

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  pinf := math.Inf(1)
  ninf := math.Inf(-1)
  nan := math.NaN()

  fmt.Println(pinf, ninf, nan)
  fmt.Println("nan == nan:", nan == nan)
  fmt.Println("IsNaN:", math.IsNaN(nan), "IsInf:", math.IsInf(pinf, 1))

  var zero float64
  fmt.Println("1.0/0.0 =", 1.0/zero)
  fmt.Println("-1.0/0.0 =", -1.0/zero)
  fmt.Println("0.0/0.0 =", zero/zero)

  fmt.Println()
  fmt.Println("7/2 =", 7/2)
  fmt.Println("-7/2 =", -7/2)
  fmt.Println("7%2 =", 7%2)
  fmt.Println("-7%2 =", -7%2)
  fmt.Println("7%-2 =", 7%-2)
  fmt.Println("-7%-2 =", -7%-2)
}
```

Output:

```
+Inf -Inf NaN
nan == nan: false
IsNaN: true IsInf: true
1.0/0.0 = +Inf
-1.0/0.0 = -Inf
0.0/0.0 = NaN

7/2 = 3
-7/2 = -3
7%2 = 1
-7%2 = -1
7%-2 = 1
-7%-2 = -1
```

Integer division truncates toward zero, so `-7/2` is `-3`, not `-4`. The remainder takes the sign of the *dividend*, matching C and Rust but **not** Python (where `-7 % 2` is `1`). So `n % 2 == 1` is false for negative odd `n` — test `n % 2 != 0` instead.

`NaN != NaN` is the IEEE-754 rule. It means a `float64` NaN used as a map key can never be found again, and `slices.Contains` will never match it.

## Two kinds of division by zero

A constant zero divisor is caught at compile time.

```go
package main

import "fmt"

func main() {
  fmt.Println(10 / 0)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:20: invalid operation: division by zero
```

A variable zero divisor panics at run time.

```go
package main

import "fmt"

func main() {
  n := 10
  d := 0
  fmt.Println(n / d)
}
```

This program panics on purpose. Output:

```
panic: runtime error: integer divide by zero

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:8 +0x1c
exit status 2
```

Float division by zero does neither — it yields `±Inf` or `NaN`, as shown above.

## Constant arithmetic is exact; variable arithmetic is not

```go
package main

import "fmt"

func main() {
  fmt.Println("constants: 0.1+0.2 == 0.3 ->", 0.1+0.2 == 0.3)

  a, b, c := 0.1, 0.2, 0.3
  fmt.Println("variables: a+b == c   ->", a+b == c)
  fmt.Printf("%.20f\n%.20f\n", a+b, c)
}
```

Output:

```
constants: 0.1+0.2 == 0.3 -> true
variables: a+b == c   -> false
0.30000000000000004441
0.29999999999999998890
```

The first line is `true` because untyped constants are evaluated at arbitrary precision at compile time — the comparison never touches a `float64`. The second is the familiar floating-point result. Never compare floats with `==`; compare `math.Abs(a-b) < epsilon`. This is the first hint of a bigger topic: see 2.4.

## Conversion is always explicit

```go
package main

import "fmt"

func main() {
  var i int = 10
  var f float64 = 2.5
  fmt.Println(i * f)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: i * f (mismatched types int and float64)
```

Same-size types are no exception. `int` and `int32` are both 64-bit and 32-bit respectively here, but even `int` and `int64` — identical in layout on this platform — do not mix:

```go
package main

import "fmt"

func main() {
  var a int = 1
  var b int32 = 2
  fmt.Println(a + b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: a + b (mismatched types int and int32)
```

And assignment is just as strict:

```go
package main

import "fmt"

func main() {
  var a int = 1
  var b int32 = a
  fmt.Println(b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:17: cannot use a (variable of type int) as int32 value in variable declaration
```

Note how different this is from the constant case. `var b int32 = 1` is fine, because `1` is an untyped constant that adopts `int32`. It is the *variable* `a` that is rigid.

## What conversion does to the value

```go
package main

import "fmt"

func main() {
  var i int = 10
  var f float64 = 2.5
  fmt.Println(float64(i) * f)
  fmt.Println(i * int(f))

  var big int = 300
  fmt.Println("int 300 -> int8:", int8(big))
  fmt.Println("int 300 -> uint8:", uint8(big))

  neg := -1
  fmt.Println("int -1 -> uint8:", uint8(neg))
  fmt.Println("int -1 -> uint64:", uint64(neg))

  fmt.Println()
  a, b, c, d := 2.9, -2.9, 2.1, -2.1
  fmt.Println(int(a), int(b), int(c), int(d))

  var big64 int64 = 1 << 40
  fmt.Println("int64 1<<40 -> int32:", int32(big64))

  var huge int64 = 1<<62 + 1
  fmt.Println("round-trips through float64:", int64(float64(huge)) == huge)
  fmt.Println(huge, int64(float64(huge)))
}
```

Output:

```
25
20
int 300 -> int8: 44
int 300 -> uint8: 44
int -1 -> uint8: 255
int -1 -> uint64: 18446744073709551615

2 -2 2 -2
int64 1<<40 -> int32: 0
round-trips through float64: false
4611686018427387905 4611686018427387904
```

Narrowing an integer keeps the low bits and discards the rest: `300` is `0b100101100`, and the low 8 bits are `0b00101100` = 44. `-1` becomes all-ones, so `uint8(-1)` is 255. `int32(1<<40)` is 0 because the only set bit was above bit 31. `int(f)` **truncates toward zero** — never rounds — so `-2.9` becomes `-2`, not `-3`. For rounding use `math.Round`, `math.Floor`, or `math.Ceil`, each of which returns a `float64` you then convert.

`int64` has 63 bits of magnitude but `float64` has only 53 bits of mantissa, so the round trip loses the low bits.

## A constant float cannot be converted to int if it has a fraction

```go
package main

import "fmt"

func main() {
  fmt.Println(int(2.9))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:6:19: cannot convert 2.9 (untyped float constant) to type int
```

The truncation rule applies to values, not constants. The compiler refuses to silently drop precision from a literal you wrote. `int(2.0)` is fine.

## Out-of-range float to int is saturating here, but do not rely on it

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  f := 1e19
  fmt.Println("1e19 -> int64:", int64(f))
  fmt.Println("NaN -> int64:", int64(math.NaN()))
  fmt.Println("-1e19 -> int64:", int64(-f))
}
```

Output:

```
1e19 -> int64: 9223372036854775807
NaN -> int64: 0
-1e19 -> int64: -9223372036854775808
```

The Go spec says the behaviour is *implementation-specific* when the float does not fit the integer type. On darwin/arm64 the hardware saturates and NaN becomes 0; other architectures may differ. Range-check before converting.

## Parsing from strings

Conversion is not parsing. `int("42")` is a compile error; `strconv` is the tool.

```go
package main

import (
  "fmt"
  "strconv"
)

func main() {
  n, err := strconv.Atoi("123")
  fmt.Println(n, err)
  _, err = strconv.Atoi("12x")
  fmt.Println(err)
  g, err := strconv.ParseFloat("3.25", 64)
  fmt.Println(g, err)
  u, err := strconv.ParseUint("255", 10, 8)
  fmt.Println(u, err)
  _, err = strconv.ParseInt("300", 10, 8)
  fmt.Println(err)
}
```

Output:

```
123 <nil>
strconv.Atoi: parsing "12x": invalid syntax
3.25 <nil>
255 <nil>
strconv.ParseInt: parsing "300": value out of range
```

The `bitSize` argument makes `strconv` do the range check you would otherwise forget. Full coverage in 6.6.

## Numeric literal forms

```go
package main

import "fmt"

func main() {
  fmt.Println(42, 0b1010, 0o755, 0755, 0xFF, 0xff)
  fmt.Println(1_000_000, 0xDE_AD_BE_EF)
  fmt.Println(1.5, .5, 1., 1e3, 1.5e-3, 0x1p4)
  fmt.Println(3i, 1+2i)
  fmt.Println('A', 'A'+1, '\n', '世')
}
```

Output:

```
42 10 493 493 255 255
1000000 3735928559
1.5 0.5 1 1000 0.0015 16
(0+3i) (1+2i)
65 66 10 19990
```

`_` is a digit separator anywhere between digits. `0o755` is the modern octal form; bare `0755` still works. `0x1p4` is a hexadecimal float (1 × 2⁴). A rune literal in single quotes is an untyped integer constant, which is why `'A'+1` is 66.

## Quick Reference

| Type | Bits | Range |
|---|---|---|
| `int8` | 8 | -128 … 127 |
| `int16` | 16 | -32768 … 32767 |
| `int32` / `rune` | 32 | -2147483648 … 2147483647 |
| `int64` | 64 | ±9.22e18 |
| `int` | 64 here, platform-dependent | `math.MinInt` … `math.MaxInt` |
| `uint8` / `byte` | 8 | 0 … 255 |
| `uint16` | 16 | 0 … 65535 |
| `uint32` | 32 | 0 … 4294967295 |
| `uint64` | 64 | 0 … 1.84e19 |
| `uint` | 64 here, platform-dependent | 0 … `math.MaxUint` |
| `uintptr` | pointer-sized | for `unsafe` only |
| `float32` | 32 | ~7 significant digits |
| `float64` | 64 | ~15 significant digits |
| `complex64` | 64 | two `float32` |
| `complex128` | 128 | two `float64` |

| Operation | Behaviour |
|---|---|
| signed/unsigned overflow | wraps, two's complement, no panic |
| `a / b` on integers | truncates toward zero |
| `a % b` | sign follows `a` |
| `x / 0` constant | compile error |
| `x / 0` variable integer | runtime panic |
| `x / 0.0` float | `+Inf`, `-Inf`, or `NaN` |
| `NaN == NaN` | `false` |
| `intN(largerInt)` | keeps low bits |
| `int(float)` | truncates toward zero |
| `int(float)` out of range | implementation-specific |
| `T(x)` between any two numeric types | always allowed, never implicit |

| Helper | Use |
|---|---|
| `math.MaxInt`, `math.MinInt`, `math.MaxUint8`, … | limits as untyped constants |
| `math.Inf(1)`, `math.NaN()` | construct specials |
| `math.IsNaN`, `math.IsInf` | test specials |
| `math.Round`, `Floor`, `Ceil`, `Trunc`, `Abs` | float rounding, all return `float64` |
| `math.Hypot` | `sqrt(x²+y²)` without overflow |
| `math/bits.Add64`, `Mul64` | overflow-aware integer math |
| `strconv.Atoi`, `ParseInt`, `ParseUint`, `ParseFloat` | string to number |

## Common Mistakes

**Assuming overflow will be caught.** It wraps silently in every build; check bounds yourself or use `math/bits`.

**Using an unsigned loop counter that counts down to zero.** `i >= 0` is always true for `uint`, so the loop never ends; use `int`.

**Testing `n % 2 == 1` for oddness.** It is `-1` for negative odd numbers; use `n % 2 != 0`.

**Comparing `float64` values with `==`.** Only constants compare exactly; compare `math.Abs(a-b) < epsilon` for variables.

**Writing `int64(x)` on a `float64` without a range check.** Out-of-range results are implementation-specific, not an error.

## Exercises

1. Write a `func addChecked(a, b int) (int, bool)` that returns the sum and whether it overflowed, using only comparisons. Verify it against `math.MaxInt` and `math.MinInt`. Then reimplement it with `math/bits.Add64` and confirm they agree.
2. Print `uint8(n)` for `n` from `-5` to `260` and describe the pattern in one sentence. Do the same for `int8(n)`.
3. Build a table that, for each of `float32` and `float64`, finds the smallest positive integer that cannot be represented exactly (hint: look for `n` where `float(n) == float(n+1)`).
4. Write a `func div(a, b int) (int, error)` that returns an error instead of panicking on zero. Then write the version that recovers from the panic instead, and say which you would ship.
5. Parse the strings `"42"`, `"-1"`, `"3.0"`, `"0x1f"`, `" 7"`, and `"9223372036854775808"` with both `strconv.Atoi` and `strconv.ParseInt(s, 0, 64)`. Explain every difference in the results.

---

[← 2.1 Variables, Zero Values, and Declaration Forms](01-variables.md) · [Course Index](../README.md) · [2.3 Booleans, Operators, and Precedence →](03-operators.md)
