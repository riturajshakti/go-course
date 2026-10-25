[← 2.2 Numeric Types, Overflow, and Conversion](02-numeric-types.md) · [Course Index](../README.md) · [2.4 Constants, Typed vs Untyped, and iota →](04-constants-iota.md)

# 2.3 — Booleans, Operators, and Precedence

## What You'll Learn

- `x++` is a statement, not an expression, and there is no ternary
- `&^` (AND NOT) is a Go-specific bit-clear operator
- `==` works on structs and arrays but is a compile error on slices, maps, and funcs
- Go has five precedence levels for binary operators, where C has fifteen

## Coming From Another Language

| Other | Go |
|---|---|
| JS `a ? b : c` | no ternary; use a 3-line `if` or a helper function |
| JS `y = x++` | `x++` is a statement; `y = x++` does not compile |
| JS `a && b` returns an operand | Go's `&&` returns a `bool` only; both sides must be `bool` |
| JS truthiness (`if (arr.length)`) | only `bool` is allowed in a condition |
| Python `a < b < c` chaining | not allowed; write `a < b && b < c` |
| Python `~x` | `^x` is Go's bitwise NOT |
| Rust `a & !b` | `a &^ b` |

## Only bool is a condition

```go
package main

import "fmt"

func main() {
  n := 0
  s := ""
  var p *int

  fmt.Println(n == 0, s == "", p == nil)
  fmt.Println(!(n == 0))

  t, f := true, false
  fmt.Println(t && f, t || f, t != f, t == f)
}
```

Output:

```
true true true
false
false true true false
```

There is no truthiness. `if n` on an `int` is a compile error, which is why Go code is full of explicit `!= 0`, `!= ""`, and `!= nil`. `!=` on two bools is XOR; `==` is XNOR.

## && and || short-circuit

Proven with a side-effecting function, not asserted.

```go
package main

import "fmt"

func loud(name string, v bool) bool {
  fmt.Println("  evaluated", name)
  return v
}

func main() {
  fmt.Println("false && loud:")
  fmt.Println(loud("A", false) && loud("B", true))

  fmt.Println("true || loud:")
  fmt.Println(loud("C", true) || loud("D", false))

  fmt.Println("true && loud:")
  fmt.Println(loud("E", true) && loud("F", false))

  fmt.Println()
  var m map[string]int
  fmt.Println(m != nil && m["k"] > 0)
}
```

Output:

```
false && loud:
  evaluated A
false
true || loud:
  evaluated C
true
true && loud:
  evaluated E
  evaluated F
false

false
```

`B` and `D` are never evaluated. This is a guarantee, not an optimization, which is what makes the nil-guard idiom `p != nil && p.Field > 0` safe. Note the last line: a nil map read is actually legal, so that guard is belt-and-braces — but the same shape with a nil *pointer* is load-bearing.

## Bitwise operators, including &^

```go
package main

import "fmt"

func main() {
  const a, b = 0b1100, 0b1010
  fmt.Printf("a      = %04b\n", a)
  fmt.Printf("b      = %04b\n", b)
  fmt.Printf("a & b  = %04b\n", a&b)
  fmt.Printf("a | b  = %04b\n", a|b)
  fmt.Printf("a ^ b  = %04b\n", a^b)
  fmt.Printf("a &^ b = %04b\n", a&^b)
  fmt.Printf("^a     = %b (%d)\n", ^a, ^a)

  fmt.Println()
  fmt.Printf("1<<0=%d 1<<10=%d 1<<20=%d\n", 1<<0, 1<<10, 1<<20)
  fmt.Printf("256>>4 = %d\n", 256>>4)

  var si int8 = -8
  var ui uint8 = 248
  fmt.Printf("int8 -8  >>1 = %d (%08b)\n", si>>1, uint8(si>>1))
  fmt.Printf("uint8 248>>1 = %d (%08b)\n", ui>>1, ui>>1)

  var x uint8 = 0b1000_0001
  fmt.Printf("uint8 shift out: %08b << 1 = %08b\n", x, x<<1)

  n := 5
  fmt.Println("shift by more than width:", uint8(1)<<uint(n*2))
}
```

Output:

```
a      = 1100
b      = 1010
a & b  = 1000
a | b  = 1110
a ^ b  = 0110
a &^ b = 0100
^a     = -1101 (-13)

1<<0=1 1<<10=1024 1<<20=1048576
256>>4 = 16
int8 -8  >>1 = -4 (11111100)
uint8 248>>1 = 124 (01111100)
uint8 shift out: 10000001 << 1 = 00000010
shift by more than width: 0
```

`&^` is AND NOT, also called bit clear: `a &^ b` keeps the bits of `a` that are not set in `b`. In C you write `a & ~b`. It exists because clearing flags is common enough to deserve one token, and it reads better in `flags &^= Verbose`.

`^` is both the binary XOR and the unary NOT. `^a` on an untyped constant gives `-13` because constants are arbitrary-precision signed integers, where `^x == -x-1`.

Shift behaviour: `>>` on a **signed** type is an arithmetic shift that preserves the sign bit (`-8 >> 1 == -4`, filling with ones). On an **unsigned** type it fills with zeros. Bits shifted off the top are simply discarded, and shifting by more than the type's width gives 0 rather than being undefined as in C.

## x++ is a statement

```go
package main

import "fmt"

func main() {
  x := 1
  y := x++
  fmt.Println(y)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:9: syntax error: unexpected ++ at end of statement
```

And there is no prefix form:

```go
package main

import "fmt"

func main() {
  x := 1
  ++x
  fmt.Println(x)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:3: syntax error: unexpected ++, expected }
```

This removes a whole family of questions (`i++ + ++i`) from the language. Increment, then use the variable on the next line.

## There is no ternary

```go
package main

import "fmt"

func main() {
  n := 5
  s := n > 3 ? "big" : "small"
  fmt.Println(s)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:7:14: invalid character U+003F '?'
./main.go:7:16: syntax error: unexpected literal "big" at end of statement
```

The `?` is not even a token in Go. The two alternatives:

```go
package main

import "fmt"

func label(n int) string {
  if n > 3 {
    return "big"
  }
  return "small"
}

func main() {
  n := 5

  s := "small"
  if n > 3 {
    s = "big"
  }
  fmt.Println(s)
  fmt.Println(label(n), label(1))
}
```

Output:

```
big
big small
```

Idiomatic: declare with the default, then `if` to override. For a value you need in several places, a small named function reads better than either.

Since Go 1.18 you can also write a generic `func ifElse[T any](c bool, a, b T) T`, but both arms are evaluated eagerly, which is a different thing from a ternary. Most codebases do not add it.

## Assignment operators

```go
package main

import "fmt"

func main() {
  x := 7
  x++
  fmt.Println(x)
  x--
  fmt.Println(x)

  x += 10
  x *= 2
  x -= 4
  x /= 2
  x %= 7
  fmt.Println(x)

  var bits uint8 = 0b0011
  bits |= 0b1000
  bits &= 0b1110
  bits ^= 0b0100
  bits &^= 0b0010
  bits <<= 1
  fmt.Printf("%08b\n", bits)
}
```

Output:

```
8
7
1
00011000
```

Every binary arithmetic and bitwise operator has an `op=` form, `&^=` included. There is no `&&=` or `||=`, and no `??=`.

## Comparison: what == works on

```go
package main

import (
  "fmt"
  "slices"
)

type Point struct{ X, Y int }

type Bag struct{ Items []int }

func main() {
  fmt.Println(Point{1, 2} == Point{1, 2})
  fmt.Println(Point{1, 2} == Point{1, 3})

  a := [3]int{1, 2, 3}
  b := [3]int{1, 2, 3}
  fmt.Println(a == b)

  s1 := []int{1, 2, 3}
  s2 := []int{1, 2, 3}
  fmt.Println(slices.Equal(s1, s2))
  fmt.Println(s1 == nil, []int(nil) == nil)

  fmt.Println()
  fmt.Println("apple" < "banana", "Zoo" < "apple", "abc" == "abc")
  fmt.Println("a" < "ab", "" < "a")

  fmt.Println()
  var i1 any = 5
  var i2 any = 5
  var i3 any = int64(5)
  fmt.Println(i1 == i2, i1 == i3)

  var i4 any = Bag{}
  fmt.Println(i4 != nil)
}
```

Output:

```
true
false
true
true
false true

true true true
true true

true false
true
```

Structs and arrays compare field-by-field and element-by-element — no `.equals` needed. String comparison is lexicographic by **byte**, which is why `"Zoo" < "apple"` is true: uppercase letters are lower in ASCII. That is byte order, not alphabetical order and not locale-aware collation.

Two interface values are equal when their dynamic *types* and dynamic *values* are both equal, so `any(5) != any(int64(5))` even though both print as `5`. See 5.2 for the nil-interface subtleties.

## Comparison: what == rejects

```go
package main

import "fmt"

func main() {
  a := []int{1, 2}
  b := []int{1, 2}
  fmt.Println(a == b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: a == b (slice can only be compared to nil)
```

```go
package main

import "fmt"

func main() {
  a := map[string]int{"x": 1}
  b := map[string]int{"x": 1}
  fmt.Println(a == b)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: a == b (map can only be compared to nil)
```

```go
package main

import "fmt"

func main() {
  f := func() {}
  g := func() {}
  fmt.Println(f == g)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:15: invalid operation: f == g (func can only be compared to nil)
```

The ban is contagious: a struct containing any of those three is itself uncomparable.

```go
package main

import "fmt"

type Bag struct {
  Items []int
}

func main() {
  fmt.Println(Bag{} == Bag{})
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:15: invalid operation: Bag{} == Bag{} (struct containing []int cannot be compared)
```

This also means such a struct cannot be a map key. Use `slices.Equal`, `maps.Equal`, or `reflect.DeepEqual` for value comparison.

## Uncomparable types inside an interface panic at run time

The compile-time check only works when the static type is known. Hide a slice in an `any` and the check moves to run time.

```go
package main

import "fmt"

func main() {
  var a any = []int{1, 2}
  var b any = []int{1, 2}
  fmt.Println(a == b)
}
```

This program panics on purpose. Output:

```
panic: runtime error: comparing uncomparable type []int

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:8 +0x58
exit status 2
```

This is a real production hazard: a `map[any]T` keyed by caller-supplied values panics the moment someone passes a slice.

## Precedence: five levels

```go
package main

import "fmt"

func main() {
  p, q, r := true, false, false

  fmt.Println(2 + 3*4)
  fmt.Println(1 | 2&3)
  fmt.Println(1<<2 + 3)
  fmt.Println((1 << 2) + 3)
  fmt.Println(1 << (2 + 3))

  fmt.Println()
  fmt.Println(1+2 == 3)
  fmt.Println(p || q && r)
  fmt.Println((p || q) && r)

  fmt.Println()
  fmt.Println(4&6 | 1)
  fmt.Println(1 ^ 2&3)

  x := 10
  fmt.Println(-x*2, ^x&0xFF, !(x > 5))
}
```

Output:

```
14
3
7
7
32

true
true
false

5
3
-20 245 false
```

| Level | Operators |
|---|---|
| 5 (highest) | `*` `/` `%` `<<` `>>` `&` `&^` |
| 4 | `+` `-` `\|` `^` |
| 3 | `==` `!=` `<` `<=` `>` `>=` |
| 2 | `&&` |
| 1 (lowest) | `\|\|` |

Unary operators (`-` `+` `!` `^` `*` `&` `<-`) bind tighter than any binary operator.

The surprise for C and JS programmers is that the shifts and `&` sit at the **top** level with `*`, not far below `+`. So `1<<2 + 3` is `(1<<2) + 3 == 7`, and `4&6 | 1` is `(4&6) | 1 == 5` — both of which need parentheses in C to mean what they already mean in Go. `1 | 2&3` is `1 | (2&3) == 1|2 == 3`.

Note `1+2 == 3` parses as `(1+2) == 3`: arithmetic beats comparison at every level, so `a+1 > b` needs no parentheses.

Idiomatic: parenthesize anyway when mixing `&`, `|`, and `+` in one expression. `gofmt` will even help by tightening the spacing around the higher-precedence operator — look at how it formatted `1<<2 + 3` and `^x&0xFF` above. That spacing is a precedence hint from the formatter.

## No chained comparison, no operator overloading

`a < b < c` parses as `(a < b) < c`, which is `bool < int` and a type error — write
`a < b && b < c`. Go also has no operator overloading: you cannot make `+` work on
your `Money` type, which is why `math/big` uses `Add` and `Mul` methods.

## Quick Reference

| Operator | Applies to | Note |
|---|---|---|
| `+` | numbers, strings | string concatenation |
| `-` `*` `/` | numbers | `/` truncates on integers |
| `%` | integers only | sign follows the dividend |
| `&` `\|` `^` | integers | AND, OR, XOR |
| `&^` | integers | AND NOT (bit clear); Go-specific |
| `<<` `>>` | integers | `>>` is arithmetic on signed types |
| `^x` (unary) | integers | bitwise NOT |
| `==` `!=` | comparable types | structs and arrays yes; slices/maps/funcs no |
| `<` `<=` `>` `>=` | ordered types | numbers and strings only |
| `&&` `\|\|` | `bool` only | short-circuit |
| `!x` | `bool` only | |
| `x++` `x--` | statements | not expressions; no prefix form |
| `op=` | all binary arithmetic/bitwise | including `&^=` |
| `&x` `*p` | pointers | see 4.1 |
| `<-ch` | channels | see 10.2 |

| Comparable? | Type |
|---|---|
| yes | all numerics, `string`, `bool`, pointers, channels, interfaces, arrays of comparables, structs of comparables |
| only against `nil` | slice, map, func |
| no | struct or array containing a slice, map, or func |
| panics at run time | an uncomparable value stored in an interface |

| Instead of | Use |
|---|---|
| `a ? b : c` | `if`, or a helper function |
| `y = x++` | `x++` then use `x` |
| `s1 == s2` for slices | `slices.Equal` |
| `m1 == m2` for maps | `maps.Equal` |
| `a & ~b` | `a &^ b` |
| `a < b < c` | `a < b && b < c` |

## Common Mistakes

**Writing `y := x++`.** `x++` is a statement with no value; increment on its own line.

**Comparing slices or maps with `==`.** It does not compile; use `slices.Equal` or `maps.Equal`.

**Keying a `map[any]T` with caller-supplied values.** A slice or map value panics at run time; constrain the key type instead.

**Assuming `1<<2 + 3` means `1 << (2+3)` as it would in C.** Shifts bind tighter than `+` in Go; add parentheses when it matters.

**Testing truthiness with `if n` or `if s`.** Only `bool` is a valid condition; write `n != 0` or `s != ""`.

## Exercises

1. Write a `Flags uint8` type with four `1 << iota` constants and methods `Set`, `Clear`, `Toggle`, and `Has`, implemented with `|`, `&^`, `^`, and `&`. Print the bits with `%08b` after each call.
2. Demonstrate short-circuiting three more ways: guarding a nil pointer dereference, guarding a slice index, and guarding a division by zero. Show that removing the guard panics in each case.
3. Build a program that prints `x >> 1` for `x` from `-8` to `8` as both `int8` and `uint8` (converted). Explain the two different fill patterns in one sentence.
4. Write a struct that is comparable and one that is not, differing by a single field. Put both in a `map[T]bool` and record which one fails and with what message.
5. For each of the five precedence levels, write one expression whose value changes when you add parentheses, and one where the parentheses are redundant. Run `gofmt` and note which spacing it chooses.

---

[← 2.2 Numeric Types, Overflow, and Conversion](02-numeric-types.md) · [Course Index](../README.md) · [2.4 Constants, Typed vs Untyped, and iota →](04-constants-iota.md)
