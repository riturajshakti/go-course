[← 13.8 regexp](08-regexp.md) · [Course Index](../README.md) · [13.10 path/filepath and io/fs →](10-filepath-fs.md)

# 13.9 — math, math/big, and math/rand

## What You'll Learn

- The whole `math` surface: it is `float64`-only, and the `min`/`max` **builtins** are what you want for ints
- Float traps with real output: `0.1+0.2`, `NaN != NaN`, integer wrap vs float `Inf`, precision loss above 2^53
- `math/big` for exact arithmetic, and the real cost of it
- `math/rand/v2` with a **seeded** source for reproducibility, and why tokens need `crypto/rand`

## Coming From Another Language

| Other | Go |
|---|---|
| JS `Math.max(1, 2)` on anything numeric | `math.Max` is `float64` **only**; use the `max` builtin for ints |
| Python ints are arbitrary precision | Go `int64` **wraps silently**; reach for `math/big` |
| Python `//` and `%` floor toward negative | Go `/` and `%` truncate toward zero; `math.Mod` matches Go, not Python |
| JS `1/0` → `Infinity`, same for ints | float `1/0` is `+Inf`, but **integer** `1/0` **panics** |
| `Math.random()` auto-seeded | `rand.IntN` auto-seeded too, but **not** secure; `crypto/rand` for secrets |

Three things to internalize. `math` operates on `float64` and nothing else, so an `int`
argument is a compile error rather than a conversion. Integer overflow is **silent
wraparound** while float overflow saturates to `±Inf`, so a Go integer bug looks like a
plausible negative number. And `math/rand` is a simulation tool, not a security tool.

## The math reference

Every function below takes and returns `float64`.

**Rounding and remainders**

| Func | Signature | Note |
|---|---|---|
| `Floor` | `(x float64) float64` | toward -Inf |
| `Ceil` | `(x float64) float64` | toward +Inf |
| `Round` | `(x float64) float64` | half **away from zero** |
| `RoundToEven` | `(x float64) float64` | half to even - banker's rounding |
| `Trunc` | `(x float64) float64` | toward zero |
| `Abs` | `(x float64) float64` | also clears the sign of `-0` |
| `Mod` | `(x, y float64) float64` | sign of **x**; matches Go's `%` |
| `Remainder` | `(x, y float64) float64` | IEEE 754; quotient rounds to **nearest** |

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  fmt.Println("--- rounding ---")
  fmt.Printf("%-7s %-7s %-7s %-7s %-12s %-7s\n", "x", "Floor", "Ceil", "Trunc", "RoundToEven", "Round")
  for _, x := range []float64{2.5, -2.5, 2.4, 3.5, -0.5} {
    fmt.Printf("%-7.1f %-7.0f %-7.0f %-7.0f %-12.0f %-7.0f\n",
      x, math.Floor(x), math.Ceil(x), math.Trunc(x), math.RoundToEven(x), math.Round(x))
  }

  fmt.Println("\n--- Mod keeps the sign of x; Remainder is IEEE 754 ---")
  fmt.Println("Mod(7,3):", math.Mod(7, 3), " Mod(-7,3):", math.Mod(-7, 3), " Abs(-3.5):", math.Abs(-3.5))
  fmt.Println("Mod(5,3):", math.Mod(5, 3), " Remainder(5,3):", math.Remainder(5, 3))
  fmt.Println("Mod(1.5,1):", math.Mod(1.5, 1), " Remainder(1.5,1):", math.Remainder(1.5, 1))
}
```

Output:

```
--- rounding ---
x       Floor   Ceil    Trunc   RoundToEven  Round  
2.5     2       3       2       2            3      
-2.5    -3      -2      -2      -2           -3     
2.4     2       3       2       2            2      
3.5     3       4       3       4            4      
-0.5    -1      -0      -0      -0           -1     

--- Mod keeps the sign of x; Remainder is IEEE 754 ---
Mod(7,3): 1  Mod(-7,3): -1  Abs(-3.5): 3.5
Mod(5,3): 2  Remainder(5,3): -1
Mod(1.5,1): 0.5  Remainder(1.5,1): -0.5
```

`Round(2.5)` is **3** and `RoundToEven(2.5)` is **2** - half away from zero versus half to
the nearest even integer, so `2.5 -> 2` and `3.5 -> 4`. Banker's rounding is the
accounting and IEEE 754 default because always rounding half upward introduces a
systematic positive bias: round a million prices with `Round` and the total is measurably
too high. Use `RoundToEven` for money and statistics, `Round` for display. `Ceil(-0.5)`
prints `-0`, negative zero, which equals `0` but keeps its sign bit.

`Mod` keeps the sign of the dividend, matching Go's integer `%`. **It does not match
Python's `%`**, which keeps the sign of the divisor: Python's `-7 % 3` is `2`, Go's is
`-1` - a real porting bug. `Remainder` is the IEEE 754 operation and differs whenever the
quotient rounds *up*, hence `Mod(5,3) == 2` but `Remainder(5,3) == -1`.

Idiomatic: `math.Mod` unless you specifically want IEEE 754 semantics.

**Powers, roots, exponentials and logs**

| Func | Signature | Note |
|---|---|---|
| `Pow` | `(x, y float64) float64` | general; slower than `x*x` |
| `Pow10` | `(n int) float64` | `10**n`, table-driven |
| `Sqrt` | `(x float64) float64` | `NaN` for x < 0 |
| `Cbrt` | `(x float64) float64` | handles negatives |
| `Exp` | `(x float64) float64` | e**x |
| `Exp2` | `(x float64) float64` | 2**x |
| `Expm1` | `(x float64) float64` | `e**x - 1`, accurate near 0 |
| `Log` | `(x float64) float64` | natural; `-Inf` at 0, `NaN` below |
| `Log2` / `Log10` | `(x float64) float64` | exact for exact powers |
| `Log1p` | `(x float64) float64` | `log(1+x)`, accurate near 0 |
| `Hypot` | `(p, q float64) float64` | `sqrt(p*p+q*q)` **without** overflow |

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  fmt.Println("Pow(2,10):", math.Pow(2, 10), " Pow(2,0.5):", math.Pow(2, 0.5), " Pow10(6):", math.Pow10(6))
  fmt.Println("Sqrt(2):", math.Sqrt(2), " Sqrt(-1):", math.Sqrt(-1), " Cbrt(-27):", math.Cbrt(-27))
  fmt.Println("Exp(1):", math.Exp(1), " Exp2(10):", math.Exp2(10))
  fmt.Println("Log(E):", math.Log(math.E), " Log2(1024):", math.Log2(1024), " Log10(1000):", math.Log10(1000))
  fmt.Println("Log(0):", math.Log(0), " Log(-1):", math.Log(-1))

  fmt.Println("\n--- Expm1/Log1p vs the naive form, near zero ---")
  fmt.Println("Expm1(1e-10):", math.Expm1(1e-10), " Exp(1e-10)-1:", math.Exp(1e-10)-1)
  fmt.Println("Log1p(1e-15):", math.Log1p(1e-15), " Log(1+1e-15):", math.Log(1+1e-15))

  fmt.Println("\n--- Hypot avoids an overflowing intermediate ---")
  p, q := 1e200, 1e200
  fmt.Println("Hypot(3,4):", math.Hypot(3, 4))
  fmt.Println("Hypot(1e200,1e200):", math.Hypot(p, q), " naive sqrt(p*p+q*q):", math.Sqrt(p*p+q*q))
}
```

Output:

```
Pow(2,10): 1024  Pow(2,0.5): 1.4142135623730951  Pow10(6): 1e+06
Sqrt(2): 1.4142135623730951  Sqrt(-1): NaN  Cbrt(-27): -3
Exp(1): 2.718281828459045  Exp2(10): 1024
Log(E): 1  Log2(1024): 10  Log10(1000): 3
Log(0): -Inf  Log(-1): NaN

--- Expm1/Log1p vs the naive form, near zero ---
Expm1(1e-10): 1.00000000005e-10  Exp(1e-10)-1: 1.000000082740371e-10
Log1p(1e-15): 9.999999999999995e-16  Log(1+1e-15): 1.110223024625156e-15

--- Hypot avoids an overflowing intermediate ---
Hypot(3,4): 5
Hypot(1e200,1e200): 1.414213562373095e+200  naive sqrt(p*p+q*q): +Inf
```

`Expm1(1e-10)` gives `1.00000000005e-10` while `Exp(1e-10)-1` gives
`1.000000082740371e-10` - the naive form lost eight significant digits to catastrophic
cancellation, because subtracting 1 from a value that close to 1 discards everything the
mantissa held. `Log1p` is the same fix for `log(1+x)` and is off in the **first** digit
above. Use them whenever the argument is small: compound interest, rates, log-odds.

`Hypot` avoids an overflowing intermediate: `1e200 * 1e200` is `1e400`, past
`MaxFloat64`, so the naive expression returns `+Inf` while `Hypot` is correct.

Nothing in `math` panics or errors. `Sqrt(-1)` and `Log(-1)` give `NaN`, `Log(0)` gives
`-Inf`, and that value flows silently onward until it reaches your output.

**Trigonometry** - all in **radians**. `Sin`/`Cos`/`Tan`, `Asin`/`Acos` (`NaN` outside
[-1, 1]), `Atan` (result in (-Pi/2, Pi/2)), `Atan2(y, x)` (result in [-Pi, Pi], **correct
quadrant**), `Sinh`/`Cosh`/`Tanh`, and `Sincos(x)` for both at once. There is no degree
conversion in the package; write `d * math.Pi / 180` yourself.

```go
package main

import (
  "fmt"
  "math"
)

func rad2deg(r float64) float64 { return r * 180 / math.Pi }

func main() {
  thirty := 30.0
  fmt.Println("Sin(30 deg):", math.Sin(thirty*math.Pi/180))
  fmt.Println("Sin(Pi):    ", math.Sin(math.Pi))
  fmt.Println("Tan(Pi/2):  ", math.Tan(math.Pi/2))
  fmt.Println("Asin(2):    ", math.Asin(2))
  fmt.Println("Atan2(-1, -1) deg:     ", rad2deg(math.Atan2(-1, -1)))
  fmt.Println("Atan(-1/1) deg (wrong):", rad2deg(math.Atan(1/-1.0)))
}
```

Output:

```
Sin(30 deg): 0.49999999999999994
Sin(Pi):     1.2246467991473515e-16
Tan(Pi/2):   1.6331239353195392e+16
Asin(2):     NaN
Atan2(-1, -1) deg:      -135
Atan(-1/1) deg (wrong): -45
```

`Sin(30 deg)` is `0.49999999999999994`, not `0.5`, and `Sin(Pi)` is `1.2246e-16`, not `0`
- `math.Pi` is itself only the nearest `float64` to Pi, so you are taking the sine of a
number very slightly off Pi. `Tan(Pi/2)` should be infinite and returns `1.63e16`. Never
compare a trig result to an exact value. (`thirty` is a variable, not the literal `30`:
as a constant expression the compiler folds it exactly and prints `0.5`, hiding the very
effect being shown - see the float-gotchas section.)

**Use `Atan2(y, x)`, not `Atan(y/x)`.** `Atan2(-1, -1)` gives the correct
third-quadrant -135 degrees, while `Atan(1/-1)` gives -45 because dividing first destroyed
which argument was negative. `Atan2` also handles `x == 0` without dividing by zero.

**Comparison: the builtins versus math.Max/Min**

`min` and `max` are **language builtins** since Go 1.21: generic over any ordered type,
variadic, and the right default. `math.Max`/`math.Min` are `float64`-only and remain for
compatibility.

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  fmt.Println("min(3,1,2) ints:", min(3, 1, 2), " max(3,1,2):", max(3, 1, 2), " min(5,2,8,1):", min(5, 2, 8, 1))
  fmt.Println(`min("b","a") strings:`, min("b", "a"))
  fmt.Println("max(2.5,1.5) floats:", max(2.5, 1.5), " math.Max(2.5,1.5):", math.Max(2.5, 1.5))

  nan := math.NaN()
  fmt.Println("\n--- they agree on NaN ---")
  fmt.Println("math.Max(1,NaN):", math.Max(1, nan), " max(1.0,NaN):", max(1.0, nan))
  fmt.Println("math.Min(1,NaN):", math.Min(1, nan), " min(1.0,NaN):", min(1.0, nan))

}
```

Output:

```
min(3,1,2) ints: 1  max(3,1,2): 3  min(5,2,8,1): 1
min("b","a") strings: a
max(2.5,1.5) floats: 2.5  math.Max(2.5,1.5): 2.5

--- they agree on NaN ---
math.Max(1,NaN): NaN  max(1.0,NaN): NaN
math.Min(1,NaN): NaN  min(1.0,NaN): NaN
```

The builtins and `math.Max`/`math.Min` agree on the awkward cases: NaN propagates through
both (unlike Rust's `f64::max`, which ignores it), and both give `+0` for
`max(+0, -0)` and `-0` for `min(+0, -0)`. So there is no behavioural reason to prefer
either for floats, and every reason to prefer the builtin for everything else.

Passing ints to `math.Max` is a compile error, not an implicit conversion:

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  a, b := 3, 7
  fmt.Println(math.Max(a, b))
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:24: cannot use a (variable of type int) as float64 value in argument to math.Max
./main.go:10:27: cannot use b (variable of type int) as float64 value in argument to math.Max
```

Idiomatic: the `min`/`max` builtins, always. Reach for `math.Max` only when you need the
`float64` signature as a function value.

**Special values and bit-level access**

| Func | Note |
|---|---|
| `Inf(sign int)` | `+Inf` for sign >= 0, else `-Inf` |
| `IsInf(f, sign)` | `sign == 0` matches **either** infinity |
| `NaN()` / `IsNaN(f)` | `IsNaN` is the **only** correct test |
| `Signbit(x)` | true for `-0` and negatives; **false for NaN** |
| `Copysign(f, sign)` | magnitude of `f`, sign of `sign` |
| `Nextafter(x, y)` | next representable float64 from x toward y |
| `Float64bits` / `Float64frombits` | the IEEE 754 bit pattern as `uint64` |

Constants: `Pi`, `E`, `Phi`, `Sqrt2`, `Ln2`, and the limits `MaxInt`/`MinInt`,
`MaxInt8`..`MaxInt64`, `MaxUint`..`MaxUint64`, `MaxFloat32`/`MaxFloat64`,
`SmallestNonzeroFloat32`/`SmallestNonzeroFloat64`. They are untyped, so
`math.MaxUint64` needs a `uint64` context or the compiler rejects it.

```go
package main

import (
  "fmt"
  "math"
)

func main() {
  inf, nan := math.Inf(1), math.NaN()
  negZero, posZero := math.Copysign(0, -1), 0.0

  fmt.Println("IsInf(inf,1):", math.IsInf(inf, 1), " IsInf(inf,-1):", math.IsInf(inf, -1),
    " IsInf(inf,0):", math.IsInf(inf, 0), " IsNaN(inf):", math.IsNaN(inf))

  fmt.Println("\n--- NaN poisons every comparison ---")
  fmt.Println("nan == nan:", nan == nan, " nan != nan:", nan != nan)
  fmt.Println("nan < 1:", nan < 1, " nan > 1:", nan > 1, " nan >= nan:", nan >= nan)
  fmt.Println("inf+1:", inf+1, " inf-inf:", inf-inf, " inf*0:", inf*0, " 1/inf:", 1/inf)

  fmt.Println("\n--- signed zero, Copysign, Signbit ---")
  fmt.Println("Signbit(-0.0):", math.Signbit(negZero), " negZero == 0:", negZero == 0)
  fmt.Println("1/negZero:", 1/negZero, " 1/posZero:", 1/posZero)
  fmt.Println("Copysign(3,-1):", math.Copysign(3, -1), " Signbit(nan):", math.Signbit(nan))

  fmt.Println("\n--- Nextafter quantifies float resolution ---")
  fmt.Printf("Nextafter(1, 2) = %.20g\n", math.Nextafter(1, 2))
  fmt.Println("gap above 1.0:", math.Nextafter(1, 2)-1, " gap above 1e16:", math.Nextafter(1e16, 2e16)-1e16)

  fmt.Println("\n--- bit patterns and constants ---")
  fmt.Printf("bits 1.0: %#016x   -0.0: %#016x\n", math.Float64bits(1.0), math.Float64bits(negZero))
  fmt.Printf("bits NaN: %#016x   +Inf: %#016x\n", math.Float64bits(nan), math.Float64bits(inf))
  fmt.Println("Float64frombits(0x3ff0000000000000):", math.Float64frombits(0x3ff0000000000000))
  fmt.Printf("Pi: %.17f  E: %.17f\n", math.Pi, math.E)
  fmt.Println("MaxInt64:", int64(math.MaxInt64), " MaxUint64:", uint64(math.MaxUint64))
  fmt.Println("MaxFloat64:", math.MaxFloat64, " SmallestNonzeroFloat64:", math.SmallestNonzeroFloat64)
}
```

Output:

```
IsInf(inf,1): true  IsInf(inf,-1): false  IsInf(inf,0): true  IsNaN(inf): false

--- NaN poisons every comparison ---
nan == nan: false  nan != nan: true
nan < 1: false  nan > 1: false  nan >= nan: false
inf+1: +Inf  inf-inf: NaN  inf*0: NaN  1/inf: 0

--- signed zero, Copysign, Signbit ---
Signbit(-0.0): true  negZero == 0: true
1/negZero: -Inf  1/posZero: +Inf
Copysign(3,-1): -3  Signbit(nan): false

--- Nextafter quantifies float resolution ---
Nextafter(1, 2) = 1.000000000000000222
gap above 1.0: 2.220446049250313e-16  gap above 1e16: 2

--- bit patterns and constants ---
bits 1.0: 0x3ff0000000000000   -0.0: 0x8000000000000000
bits NaN: 0x7ff8000000000001   +Inf: 0x7ff0000000000000
Float64frombits(0x3ff0000000000000): 1
Pi: 3.14159265358979312  E: 2.71828182845904509
MaxInt64: 9223372036854775807  MaxUint64: 18446744073709551615
MaxFloat64: 1.7976931348623157e+308  SmallestNonzeroFloat64: 5e-324
```

**`nan != nan` is `true`.** Every ordering comparison against NaN is false, `nan >= nan`
included. The consequences are broad: a `NaN` in a slice makes `slices.Sort` produce
garbage, a `NaN` map key can never be looked up again, and a struct holding a `NaN` fails
`==` against an identical copy of itself (4.3). `math.IsNaN(f)` is the test. Note
`Signbit(NaN)` is false - NaN has no meaningful sign, so do not branch on it.

**Negative zero equals zero but divides differently.** `negZero == 0` is `true`, yet
`1/negZero` is `-Inf` while `1/posZero` is `+Inf`. A zero arriving from an underflowing
negative computation carries its sign, so a division can flip your result's sign with no
visible cause.

**`Nextafter` quantifies float resolution.** The gap above `1.0` is 2.2e-16; above `1e16`
it is **2**, so there are no odd integers in `float64` up there at all - the 2^53 limit
seen from the other direction.

## Float gotchas, with real output

This section is the practical core of the lesson. Every line below was captured, not
recalled.

```go
package main

import (
  "fmt"
  "math"
)

func nearlyEqual(a, b, epsilon float64) bool {
  if a == b {
    return true
  }
  scale := math.Max(math.Abs(a), math.Abs(b))
  return math.Abs(a-b)/scale < epsilon
}

func main() {
  fmt.Println("--- 0.1 + 0.2 is not 0.3 (at RUNTIME) ---")
  a, b := 0.1, 0.2
  sum := a + b
  fmt.Println("0.1 + 0.2 == 0.3:", sum == 0.3)
  fmt.Printf("0.1 + 0.2 = %.20f\n0.3       = %.20f\n", sum, 0.3)
  fmt.Println("difference:", sum-0.3, " nearlyEqual:", nearlyEqual(sum, 0.3, 1e-9))

  const c = 0.1 + 0.2
  fmt.Println("but as a CONSTANT expression, 0.1+0.2 == 0.3:", c == 0.3)

  var acc float64
  for range 10 {
    acc += 0.1
  }
  fmt.Printf("ten runtime adds of 0.1 = %.20f  == 1.0: %v\n", acc, acc == 1.0)

  price, hundred := 0.07, 100.0
  fmt.Printf("0.07 * 100 = %.20f -> int() gives %d\n", price*hundred, int(price*hundred))

  fmt.Println("\n--- float division by zero gives Inf; INTEGER division panics ---")
  zeroF := 0.0
  fmt.Println("1.0/0.0:", 1.0/zeroF, " -1.0/0.0:", -1.0/zeroF, " 0.0/0.0:", zeroF/zeroF)

  fmt.Println("\n--- integer overflow WRAPS silently; floats saturate to Inf ---")
  var i8 int8 = 127
  i8++
  var u8 uint8 = 0
  u8--
  maxI64 := int64(math.MaxInt64)
  fmt.Println("int8 127+1:", i8, " uint8 0-1:", u8, " MaxInt64+1:", maxI64+1)
  maxF, tiny := math.MaxFloat64, math.SmallestNonzeroFloat64
  fmt.Println("MaxFloat64*2:", maxF*2, " SmallestNonzero/2:", tiny/2)

  fmt.Println("\n--- int -> float64 loses precision above 2^53 ---")
  const twoTo53 = 1 << 53
  fmt.Println("2^53:", int64(twoTo53), " float64(2^53+1) ->", int64(float64(twoTo53+1)))
  fmt.Println("float64(2^53) == float64(2^53+1):", float64(twoTo53) == float64(twoTo53+1))
  id := int64(9007199254740993)
  fmt.Println("ID", id, "round-tripped through float64 ->", int64(float64(id)),
    " survived:", int64(float64(id)) == id)

}
```

Output:

```
--- 0.1 + 0.2 is not 0.3 (at RUNTIME) ---
0.1 + 0.2 == 0.3: false
0.1 + 0.2 = 0.30000000000000004441
0.3       = 0.29999999999999998890
difference: 5.551115123125783e-17  nearlyEqual: true
but as a CONSTANT expression, 0.1+0.2 == 0.3: true
ten runtime adds of 0.1 = 0.99999999999999988898  == 1.0: false
0.07 * 100 = 7.00000000000000088818 -> int() gives 7

--- float division by zero gives Inf; INTEGER division panics ---
1.0/0.0: +Inf  -1.0/0.0: -Inf  0.0/0.0: NaN

--- integer overflow WRAPS silently; floats saturate to Inf ---
int8 127+1: -128  uint8 0-1: 255  MaxInt64+1: -9223372036854775808
MaxFloat64*2: +Inf  SmallestNonzero/2: 0

--- int -> float64 loses precision above 2^53 ---
2^53: 9007199254740992  float64(2^53+1) -> 9007199254740992
float64(2^53) == float64(2^53+1): true
ID 9007199254740993 round-tripped through float64 -> 9007199254740992  survived: false
```

Read this one carefully, because several lines are Go-specific.

**Go constants are arbitrary-precision.** `const c = 0.1 + 0.2` compares **equal** to
`0.3` while the runtime `a + b` does not. Untyped constant expressions are evaluated
exactly by the compiler (at least 256 bits of mantissa) and only rounded to `float64` when
assigned. So `0.1 + 0.2 == 0.3` written as a literal expression is `true`, and the same
arithmetic on variables is `false`. This makes the classic demo *fail to demonstrate*
unless you force runtime evaluation, which is exactly why the example above assigns to
`a` and `b` first. It also means constant-folded initializers in your code are more
accurate than the equivalent runtime computation.

**Integer division by zero panics.** Floats give you `+Inf`/`-Inf` and `NaN`, but
`1 / intZero` is a runtime panic, `panic: runtime error: integer divide by zero` (2.2).
The compiler rejects a literal `1/0` outright, so the divisor has to be a variable to
reach runtime at all.

**Integer overflow wraps, silently.** `MaxInt64 + 1` is `MinInt64`. There is no panic, no
flag, no error — just a large positive number becoming a large negative one. Go made this
choice for speed, and it means every `int` accumulator over untrusted input is a potential
bug. Floats instead saturate to `+Inf`, which at least propagates visibly.

**Above 2^53, `float64` cannot hold consecutive integers.** `float64(2^53)` and
`float64(2^53+1)` are the *same value*. An `int64` ID of `9007199254740993` does not
survive a round trip through `float64` — which is the same corruption 13.6 showed for
`map[string]any` JSON decoding, here at the language level. Any time an ID passes through
a `float64`, a JavaScript number, or an untyped JSON number, it is at risk.

**`int(f)` truncates toward zero and overflow is implementation-defined.** `int(2.9)` is
`2`, `int(-2.9)` is `-2` (use `math.Round` first for `-3`), and `int64(1e19)` gave
`MaxInt64` on darwin/arm64 - the spec leaves it undefined when the value does not fit, so
another architecture may differ. Range-check before converting.

**Money.** `0.07 * 100` is `7.0000000000000009`, so `int(...)` happens to give 7 here, but
`0.29 * 100` and friends do not always cooperate. Store money as an integer number of
minor units (cents) or as `big.Rat`; never as `float64`.

**Comparing floats.** Use a relative epsilon, as `nearlyEqual` does above: an absolute
`math.Abs(a-b) < 1e-9` is wrong for both very large values (where 1e-9 is below the
representable gap, so it degenerates to `==`) and very small ones (where everything
compares equal).

## math/big: exact arithmetic

`math/big` provides `Int` (arbitrary-precision integers), `Rat` (exact rationals) and
`Float` (arbitrary but fixed precision). All three are **mutable** and use a
destination-first convention: `z.Add(x, y)` sets `z` to `x+y` and returns `z`. That avoids
an allocation per operation and is why the API looks unusual.

| Type | Use when |
|---|---|
| `big.Int` | integers beyond 64 bits: crypto, factorials, large counters |
| `big.Rat` | exact fractions: money, unit conversion, anything that must sum to exactly 1 |
| `big.Float` | high-precision decimals with a chosen mantissa size |

```go
package main

import (
  "fmt"
  "math/big"
)

func factorialBig(n int64) *big.Int {
  r := big.NewInt(1)
  for i := int64(2); i <= n; i++ {
    r.Mul(r, big.NewInt(i))
  }
  return r
}

func factorialInt64(n int64) int64 {
  r := int64(1)
  for i := int64(2); i <= n; i++ {
    r *= i
  }
  return r
}

func main() {
  fmt.Println("--- big.Int: arbitrary precision ---")
  p := new(big.Int).Exp(big.NewInt(2), big.NewInt(1000), nil)
  fmt.Println("2^1000 has", len(p.String()), "digits, BitLen", p.BitLen())
  fmt.Println("2^1000 mod 1e9 =", new(big.Int).Mod(p, big.NewInt(1e9)))

  fmt.Println("\n--- 30! overflows int64 silently; big.Int does not ---")
  for _, n := range []int64{25, 30, 50} {
    fmt.Printf("%d!  int64: %-22d big: %s\n", n, factorialInt64(n), factorialBig(n))
  }

  fmt.Println("\n--- SetString returns a bool, not an error ---")
  n, ok := new(big.Int).SetString("123456789012345678901234567890", 10)
  fmt.Println("decimal:", n, "ok:", ok)
  h, ok := new(big.Int).SetString("ff", 16)
  fmt.Println("hex ff:", h, "ok:", ok)
  auto, ok := new(big.Int).SetString("0xdeadbeef", 0)
  fmt.Println("base 0 auto-detect:", auto, "ok:", ok)
  bad, ok := new(big.Int).SetString("12x34", 10)
  fmt.Println("invalid input:", bad, "ok:", ok)
  ignored, _ := new(big.Int).SetString("not a number", 10)
  fmt.Println("ignoring ok leaves you holding:", ignored)

  fmt.Println("\n--- destination-first arithmetic; Div/Mod vs Quo/Rem ---")
  x, y := big.NewInt(100), big.NewInt(7)
  fmt.Println("Add:", new(big.Int).Add(x, y), " Sub:", new(big.Int).Sub(x, y),
    " Mul:", new(big.Int).Mul(x, y))
  q, r := new(big.Int), new(big.Int)
  q.QuoRem(x, y, r)
  fmt.Println("QuoRem(100, 7):", q, r)
  neg := big.NewInt(-100)
  fmt.Println("Div(-100,7):", new(big.Int).Div(neg, y), " Quo(-100,7):", new(big.Int).Quo(neg, y))
  fmt.Println("Mod(-100,7):", new(big.Int).Mod(neg, y), " Rem(-100,7):", new(big.Int).Rem(neg, y))

  fmt.Println("\n--- Cmp, not ==; and Int64 does not report overflow ---")
  fmt.Println("Cmp(100,7):", x.Cmp(y), " Cmp(7,100):", y.Cmp(x), " Cmp(100,100):", x.Cmp(big.NewInt(100)))
  fmt.Println("Sign(-5):", neg.Sign())
  fmt.Println("42 IsInt64:", big.NewInt(42).IsInt64(), big.NewInt(42).Int64())
  fmt.Println("2^1000 IsInt64:", p.IsInt64(), " Int64() returns:", p.Int64())
}
```

Output:

```
--- big.Int: arbitrary precision ---
2^1000 has 302 digits, BitLen 1001
2^1000 mod 1e9 = 668069376

--- 30! overflows int64 silently; big.Int does not ---
25!  int64: 7034535277573963776    big: 15511210043330985984000000
30!  int64: -8764578968847253504   big: 265252859812191058636308480000000
50!  int64: -3258495067890909184   big: 30414093201713378043612608166064768844377641568960512000000000000

--- SetString returns a bool, not an error ---
decimal: 123456789012345678901234567890 ok: true
hex ff: 255 ok: true
base 0 auto-detect: 3735928559 ok: true
invalid input: <nil> ok: false
ignoring ok leaves you holding: <nil>

--- destination-first arithmetic; Div/Mod vs Quo/Rem ---
Add: 107  Sub: 93  Mul: 700
QuoRem(100, 7): 14 2
Div(-100,7): -15  Quo(-100,7): -14
Mod(-100,7): 5  Rem(-100,7): -2

--- Cmp, not ==; and Int64 does not report overflow ---
Cmp(100,7): 1  Cmp(7,100): -1  Cmp(100,100): 0
Sign(-5): -1
42 IsInt64: true 42
2^1000 IsInt64: false  Int64() returns: 0
```

**`int64` 30! is negative.** `factorialInt64(30)` returned `-8764578968847253504`. No
panic, no warning - a plausible-looking number that is simply wrong. This is the best
argument for `big.Int`: the failure is invisible.

**`SetString` returns a `bool`, not an `error`.** On failure the receiver is left nil and
`ok` is `false`, and `fmt` prints a nil `*big.Int` as `<nil>`. Write
`n, _ := new(big.Int).SetString(userInput, 10)` and skip the check, and you hold a nil
pointer that panics on the first method call needing a value. Always check `ok`.

**`Div`/`Mod` are Euclidean; `Quo`/`Rem` truncate.** `-100 / 7` is `-15` with `Div` and
`-14` with `Quo`. `Quo`/`Rem` match Go's own `/` and `%` on `int`; `Div`/`Mod` give a
non-negative remainder, which is what modular arithmetic and crypto want. Picking the
wrong pair is a quiet off-by-one on negative inputs.

**Never compare with `==`.** `big.Int` is a struct containing a slice, so `==` does not
compile for the values and compares *pointers* for the usual `*big.Int` form. Use
`Cmp(y) == 0`.

**`Int64()` on an out-of-range value returns garbage** - `0` for 2^1000 here - rather than
failing. Guard with `IsInt64()` first.

```go
package main

import (
  "fmt"
  "math"
  "math/big"
)

func main() {
  tenth := big.NewRat(1, 10)
  rsum := new(big.Rat)
  for range 10 {
    rsum.Add(rsum, tenth)
  }
  var fsum float64
  for range 10 {
    fsum += 0.1
  }
  fmt.Println("ten times 1/10:", rsum, "== 1:", rsum.Cmp(big.NewRat(1, 1)) == 0)
  fmt.Println("ten times 0.1: ", fsum, "== 1:", fsum == 1.0)

  third := big.NewRat(1, 3)
  fmt.Println("1/3:", third, " to 10 places:", third.FloatString(10),
    " Num/Denom:", third.Num(), third.Denom())
  money, ok := new(big.Rat).SetString("19.99")
  fmt.Println("19.99 exactly:", money, "ok:", ok, "=", money.FloatString(2))

  fmt.Println("\n--- big.Float: arbitrary but FIXED precision ---")
  fmt.Println("default prec of big.NewFloat:", big.NewFloat(1).Prec())
  oneThird := new(big.Float).SetPrec(200).Quo(big.NewFloat(1), big.NewFloat(3))
  lowPrec := new(big.Float).SetPrec(10).Quo(big.NewFloat(1), big.NewFloat(3))
  fmt.Println("1/3 at 200 bits:", oneThird.Text('f', 50))
  fmt.Println("1/3 at 10 bits: ", lowPrec.Text('f', 20))
  _, acc := oneThird.Float64()
  fmt.Println("Float64() rounding direction:", acc)
  fmt.Println("sqrt(2) at 200 bits:", new(big.Float).SetPrec(200).Sqrt(big.NewFloat(2)).Text('f', 50))
  fmt.Println("math.Sqrt(2):       ", math.Sqrt(2))
}
```

Output:

```
ten times 1/10: 1/1 == 1: true
ten times 0.1:  0.9999999999999999 == 1: false
1/3: 1/3  to 10 places: 0.3333333333  Num/Denom: 1 3
19.99 exactly: 1999/100 ok: true = 19.99

--- big.Float: arbitrary but FIXED precision ---
default prec of big.NewFloat: 53
1/3 at 200 bits: 0.33333333333333333333333333333333333333333333333333
1/3 at 10 bits:  0.33349609375000000000
Float64() rounding direction: Below
sqrt(2) at 200 bits: 1.41421356237309504880168872420969807856967187537695
math.Sqrt(2):        1.4142135623730951
```

**`big.Rat` is the only one of the three genuinely exact for decimals.** Ten additions of
`1/10` give exactly `1`; ten additions of `0.1` as `float64` give `0.9999999999999999`.
`SetString("19.99")` stores `1999/100` and `FloatString(2)` renders it back exactly. For
money, `big.Rat` or integer cents; `big.Float` still rounds.

**`big.Float` has a precision you must set.** `big.NewFloat` defaults to 53 bits - exactly
`float64`, so it buys you nothing until you call `SetPrec`. At 10 bits, `1/3` is
`0.33349609375`, visibly wrong. At 200 bits, `sqrt(2)` is correct to 50 decimal places.
`Float64()` also returns an `Accuracy` (`Below`, `Above`, `Exact`) telling you which way
it rounded.

### The cost

```go
package main

import (
  "math/big"
  "testing"
)

func BenchmarkInt64Mul(b *testing.B) {
  x, y := int64(1234567), int64(7654321)
  var sink int64
  for b.Loop() {
    sink = x * y
  }
  _ = sink
}

func BenchmarkBigIntMul(b *testing.B) {
  x, y := big.NewInt(1234567), big.NewInt(7654321)
  z := new(big.Int)
  for b.Loop() {
    z.Mul(x, y)
  }
}

func BenchmarkFloat64Add(b *testing.B) {
  x, y := 1.5, 2.5
  var sink float64
  for b.Loop() {
    sink = x + y
  }
  _ = sink
}

func BenchmarkBigRatAdd(b *testing.B) {
  x, y := big.NewRat(3, 2), big.NewRat(5, 2)
  z := new(big.Rat)
  for b.Loop() {
    z.Add(x, y)
  }
}
```

`go test -bench=. -benchmem`. These numbers are machine-specific (darwin/arm64, Apple M4
Pro) and vary a few percent between runs; the ratios are what to read.

Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkInt64Mul-12      	654466918	         1.600 ns/op	       0 B/op	       0 allocs/op
BenchmarkBigIntMul-12     	190751656	         6.289 ns/op	       0 B/op	       0 allocs/op
BenchmarkFloat64Add-12    	763251762	         1.573 ns/op	       0 B/op	       0 allocs/op
BenchmarkBigRatAdd-12     	15616002	        75.80 ns/op	     120 B/op	       5 allocs/op
```

`big.Int` multiply is about 4x a native `int64` multiply when you reuse the destination
(allocating a fresh `new(big.Int)` per operation roughly doubles that again). `big.Rat`
addition is **~48x** a `float64` add and allocates five times, because every operation
normalizes the fraction with a GCD.

That is the real trade-off: `big` is not catastrophically slow for occasional use, and
reusing destinations matters more than the type choice. But 48x with five allocations per
add rules `big.Rat` out of a hot loop, so compute in integer minor units where you can and
reserve `big.Rat` for the places exactness is the requirement.

## math/rand/v2

Go 1.22 added `math/rand/v2`, and it is the version to use. It drops the global `Seed`,
renames `Intn` to `IntN` (and adds `Int32N`/`Int64N`/`UintN`), adds the generic `rand.N`,
and replaces the old source with `PCG` and `ChaCha8`.

| Call | Returns |
|---|---|
| `IntN(n)` / `Int32N(n)` / `Int64N(n)` / `UintN(n)` | uniform in `[0, n)`; **panics if n ≤ 0** |
| `N(n)` | generic over any integer type, uniform in `[0, n)` |
| `Int()` / `Int64()` / `Uint64()` | full-width |
| `Float64()` / `Float32()` | uniform in `[0, 1)` |
| `NormFloat64()` | normal, mean 0, stddev 1 |
| `ExpFloat64()` | exponential, rate 1 |
| `Perm(n)` | a random permutation of `[0, n)` |
| `Shuffle(n, swap)` | shuffles via your swap function |
| `New(src)` | a `*Rand` over an explicit source |
| `NewPCG(s1, s2)` | fast seeded source |
| `NewChaCha8(seed [32]byte)` | slower, resistant to prediction |

**A seeded source is fully reproducible**, which is what you want in tests and
simulations:

```go
package main

import (
  "fmt"
  "math/rand/v2"
)

func main() {
  fmt.Println("--- a SEEDED source is fully deterministic ---")
  for run := 1; run <= 2; run++ {
    r := rand.New(rand.NewPCG(1, 2))
    fmt.Printf("run %d, seed (1,2):", run)
    for range 6 {
      fmt.Printf(" %3d", r.IntN(100))
    }
    fmt.Println()
  }
  r := rand.New(rand.NewPCG(42, 42))
  fmt.Print("seed (42,42):     ")
  for range 6 {
    fmt.Printf(" %3d", r.IntN(100))
  }
  fmt.Println()

  fmt.Println("\n--- the v2 API, all from one seeded source ---")
  g := rand.New(rand.NewPCG(1, 2))
  fmt.Println("IntN(10):", g.IntN(10), " Int32N(1000):", g.Int32N(1000),
    " Int64N(1<<40):", g.Int64N(1<<40), " UintN(10):", g.UintN(10))
  fmt.Println("Int():", g.Int(), " Uint64():", g.Uint64())
  fmt.Println("Float64():", g.Float64(), " Float32():", g.Float32())
  fmt.Println("NormFloat64():", g.NormFloat64(), " ExpFloat64():", g.ExpFloat64())
  fmt.Println("Perm(8):", g.Perm(8))

  deck := []string{"A", "K", "Q", "J", "10", "9", "8", "7"}
  g.Shuffle(len(deck), func(i, j int) { deck[i], deck[j] = deck[j], deck[i] })
  fmt.Println("Shuffle:", deck)

  fmt.Println("\n--- rand.N is generic over any integer type ---")
  type ms int32
  fmt.Println("rand.N(5) in [0,5):", rand.N(5) < 5)
  fmt.Printf("rand.N on a named type: %T\n", rand.N(ms(100)))

  fmt.Println("\n--- ChaCha8: slower, resistant to prediction ---")
  var seed [32]byte
  for i := range seed {
    seed[i] = byte(i)
  }
  c := rand.New(rand.NewChaCha8(seed))
  fmt.Print("fixed 32-byte seed:")
  for range 6 {
    fmt.Printf(" %3d", c.IntN(100))
  }
  fmt.Println()

  fmt.Println("\n--- top-level funcs are auto-seeded, so these vary run to run ---")
  a, b := rand.IntN(1000), rand.IntN(1000)
  fmt.Println("both in range:", a >= 0 && a < 1000, b >= 0 && b < 1000)
}
```

Every seeded line below is byte-identical on every run; only the last section varies,
which is why it prints a range check instead of the values.

Output:

```
--- a SEEDED source is fully deterministic ---
run 1, seed (1,2):  76  61  78  79  23   4
run 2, seed (1,2):  76  61  78  79  23   4
seed (42,42):       61  37  63  51  95  26

--- the v2 API, all from one seeded source ---
IntN(10): 7  Int32N(1000): 616  Int64N(1<<40): 22300033160  UintN(10): 7
Int(): 4321634407747778896  Uint64(): 760102831717374652
Float64(): 0.8192750264193612  Float32(): 0.042272687
NormFloat64(): -0.11707238034168332  ExpFloat64(): 1.2334925132631804
Perm(8): [4 1 6 2 0 5 3 7]
Shuffle: [9 7 10 J 8 K A Q]

--- rand.N is generic over any integer type ---
rand.N(5) in [0,5): true
rand.N on a named type: main.ms

--- ChaCha8: slower, resistant to prediction ---
fixed 32-byte seed:  67  60  15  47  20   9

--- top-level funcs are auto-seeded, so these vary run to run ---
both in range: true true
```

`run 1` and `run 2` print the identical six numbers, and they print those same six on
every execution. That is the property to use: inject a `*rand.Rand` built from
`rand.NewPCG(seed, seed)` into anything you want to test, and the simulation is replayable
from its seed.

Idiomatic: top-level `rand.IntN` for jitter, backoff and sampling where you do not care;
an explicit `rand.New(rand.NewPCG(...))` whenever reproducibility matters. A `*rand.Rand`
is **not** safe for concurrent use, while the top-level functions are.

### Seeding: what changed

The top-level functions have been **automatically seeded** since Go 1.20, so there is
nothing to do for ordinary use. In `math/rand/v2` the global `Seed` function was removed
entirely. In v1 it still exists, is deprecated, and — this is the part that surprises
people — **has been a no-op since Go 1.24**:

```go
package main

import (
  "fmt"
  mrand "math/rand"
)

func three(r func(int) int) []int { return []int{r(100), r(100), r(100)} }

func main() {
  mrand.Seed(1)
  a := three(mrand.Intn)
  mrand.Seed(1)
  b := three(mrand.Intn)
  fmt.Println("Seed(1) then 3 draws:", a)
  fmt.Println("Seed(1) again:       ", b)
  fmt.Println("same sequence:", a[0] == b[0] && a[1] == b[1] && a[2] == b[2])

  fmt.Println("\nNew(NewSource(1)):", three(mrand.New(mrand.NewSource(1)).Intn))
  fmt.Println("again:            ", three(mrand.New(mrand.NewSource(1)).Intn))
}
```

The first two lines vary run to run; the captured run was:

Output:

```
Seed(1) then 3 draws: [52 40 33]
Seed(1) again:        [70 49 80]
same sequence: false

New(NewSource(1)): [81 87 47]
again:             [81 87 47]
```

`rand.Seed(1)` followed by the same calls produced a **different** sequence, and the
values differ on every run. The only reproducible lines are the ones from an explicit
`New(NewSource(1))`, which print `[81 87 47]` every time. So v1 code that called
`rand.Seed(n)` to get a repeatable sequence has silently stopped working — it compiles,
it does not warn, and the determinism is gone. The fix is a local generator, and in v2
that is the only option the API offers.

### math/rand is NOT cryptographically secure

Both `math/rand` and `math/rand/v2` say so in their package docs. The output is
predictable from a small amount of observed output, so using it for anything an attacker
would like to guess is a real vulnerability:

**Never** use `math/rand` for session tokens, password-reset links, API keys, CSRF tokens,
nonces, salts, OAuth state, invite codes, or anything else that must be unguessable. Use
`crypto/rand`.

```go
package main

import (
  crand "crypto/rand"
  "encoding/base64"
  "fmt"
  "math/big"
  "strings"
)

func token(nBytes int) string {
  b := make([]byte, nBytes)
  if _, err := crand.Read(b); err != nil {
    panic(err)
  }
  return base64.RawURLEncoding.EncodeToString(b)
}

func main() {
  b := make([]byte, 16)
  n, err := crand.Read(b)
  fmt.Println("Read into a 16-byte slice: n =", n, "err =", err)

  t := token(32)
  fmt.Println("32-byte token length:", len(t), "chars; 1000 draws all distinct:", distinct(1000))

  s := crand.Text()
  const alpha = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567"
  fmt.Println("crand.Text() length:", len(s), "chars; RFC 4648 base32 only:",
    strings.IndexFunc(s, func(r rune) bool { return !strings.ContainsRune(alpha, r) }) == -1)
  fmt.Println("two Text() calls differ:", crand.Text() != crand.Text())

  max := big.NewInt(1_000_000)
  v, err := crand.Int(crand.Reader, max)
  fmt.Println("crand.Int err:", err, " in [0, 1e6):", v.Sign() >= 0 && v.Cmp(max) < 0)
  fmt.Println("(Int PANICS on max <= 0 rather than returning an error)")
  fmt.Println("\nNo value above is reproducible, which is the entire point.")
}

func distinct(n int) bool {
  seen := make(map[string]bool, n)
  for range n {
    t := token(32)
    if seen[t] {
      return false
    }
    seen[t] = true
  }
  return true
}
```

The token values themselves differ on every run, so the program asserts properties rather
than printing them.

Output:

```
Read into a 16-byte slice: n = 16 err = <nil>
32-byte token length: 43 chars; 1000 draws all distinct: true
crand.Text() length: 26 chars; RFC 4648 base32 only: true
two Text() calls differ: true
crand.Int err: <nil>  in [0, 1e6): true
(Int PANICS on max <= 0 rather than returning an error)

No value above is reproducible, which is the entire point.
```

**`crypto/rand.Text()` (Go 1.24+) is the answer for tokens.** It returns a 26-character
RFC 4648 base32 string with at least 128 bits of entropy — no byte slice, no encoder, no
length decision to get wrong. For a token, prefer it over hand-rolling `Read` + base64.

**`crypto/rand.Read` either fills the buffer or fails.** It never returns a short read, so
`n` is always `len(b)`. An error from it means the OS entropy source is broken, which is
unrecoverable — panic or exit, do not fall back to `math/rand`.

**`crypto/rand.Int` gives an unbiased value in a range.** The naive
`int(binary.BigEndian.Uint64(b) % n)` is *biased* when `n` does not divide 2^64 evenly;
`crand.Int` rejects and retries to avoid that. Note it **panics** rather than erroring on
`max <= 0`, with `panic: crypto/rand: argument to Int is <= 0` - so validate the bound
before calling it.

Idiomatic: `crypto/rand.Text()` for tokens and secrets, `crypto/rand.Int` for a secure
number in a range, `math/rand/v2` for everything that is not a secret.

## Quick Reference

| Need | Call |
|---|---|
| Round half away from zero | `math.Round(x)` |
| Round half to even (money, stats) | `math.RoundToEven(x)` |
| Min/max of ints, strings, anything | the **`min`/`max` builtins** |
| Test for NaN | `math.IsNaN(x)` — never `x == NaN` |
| `sqrt(a²+b²)` without overflow | `math.Hypot(a, b)` |
| Accurate `e**x - 1` / `log(1+x)` | `math.Expm1(x)` / `math.Log1p(x)` |
| Correct-quadrant angle | `math.Atan2(y, x)` |
| Integers beyond 64 bits | `new(big.Int).SetString(s, 10)` |
| Compare two `big.Int` | `x.Cmp(y) == 0` — **not** `==` |
| Exact decimal arithmetic | `big.Rat`, or integer minor units |
| Random int in `[0, n)` | `rand.IntN(n)` (`math/rand/v2`) |
| Reproducible randomness | `rand.New(rand.NewPCG(s1, s2))` |
| A secure token | `crypto/rand.Text()` |
| Secure number in a range | `crypto/rand.Int(crand.Reader, max)` |

| Trap | Reality |
|---|---|
| `math.Max(i, j)` on ints | compile error; `math` is `float64`-only |
| `0.1 + 0.2 == 0.3` on variables | `false`; as a `const` expression it is `true` |
| `nan == nan` | `false`; use `math.IsNaN` |
| `1 / intZero` | **panics**; `1.0 / 0.0` is `+Inf` |
| `MaxInt64 + 1` | wraps to `MinInt64`, silently |
| `float64(int64Above2Pow53)` | loses the low bits |
| `big.Int` `SetString` failure | returns `ok == false` and a nil receiver |
| `bigA == bigB` | compares pointers, not values; use `Cmp` |
| `big.NewFloat` precision | 53 bits — same as `float64` until you `SetPrec` |
| `rand.Seed(1)` for a repeatable run | a **no-op** since Go 1.24; use `New(NewSource(n))` |
| `math/rand` for a session token | predictable; use `crypto/rand` |

## Common Mistakes

**Using `math.Max`/`math.Min` for integers.** They take `float64` only; the generic
`min`/`max` builtins cover every ordered type and are variadic.

**Comparing floats with `==`.** Use a relative epsilon, and never compare a trig or
`Pow` result to an exact value.

**Storing money in a `float64`.** `0.1` is not representable, so sums drift; use integer
cents or `big.Rat`.

**Relying on `rand.Seed(n)` for a reproducible sequence.** It has been a no-op since Go
1.24; build a local generator with `rand.New(rand.NewPCG(s1, s2))`.

**Using `math/rand` for tokens, IDs or secrets.** The output is predictable; use
`crypto/rand.Text()` or `crypto/rand.Read`.

**Ignoring the `bool` from `big.Int.SetString`.** A failed parse leaves you holding a nil
`*big.Int` that panics on first use.

## Exercises

1. Write `func Round(x float64, places int) float64` using `math.Round` and `math.Pow10`,
   then table-test it against `0.145`, `2.675`, `-2.5` and `1e20`. Explain which results
   are surprising and why `math.Pow10` makes it worse rather than better.
2. Implement `func SafeAdd(a, b int64) (int64, error)` that detects signed overflow
   *before* it happens, and prove with a table that it rejects `MaxInt64 + 1` and
   `MinInt64 - 1` while accepting everything adjacent.
3. Compute the 200th Fibonacci number with `int64`, `float64` and `big.Int`, print all
   three, and explain exactly how each wrong answer became wrong.
4. Write a seeded dice simulator that takes a seed and rolls 10,000 times, and a test that
   asserts the exact same histogram for a given seed. Then show that the top-level
   `rand.IntN` cannot pass that test.
5. Implement `func ParseMoney(s string) (*big.Rat, error)` accepting `"19.99"`, `"-0.01"`
   and `"1,234.50"`, rejecting `"19.999"` and `"abc"`, then sum a slice of them and show
   the total is exact where `float64` would drift. Then write `func Bucket(id string, n
   int) int` that assigns a stable A/B bucket from a hash of the ID, show it is
   deterministic across runs where `rand.IntN(n)` is not, and explain why `crypto/rand`
   is the wrong tool for it.

---

[← 13.8 regexp](08-regexp.md) · [Course Index](../README.md) · [13.10 path/filepath and io/fs →](10-filepath-fs.md)
