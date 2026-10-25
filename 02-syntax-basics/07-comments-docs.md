[← 2.6 Variadic Functions and Closures](06-variadic-closures.md) · [Course Index](../README.md) · [3.1 if, else, and the if-with-statement Form →](../03-control-flow/01-if.md)

# 2.7 — Comments, Doc Comments, and go doc

## What You'll Learn

- Block comments do not nest, and the first `*/` ends them
- A doc comment is just a comment touching a declaration — no special syntax
- `gofmt` reformats doc comments and understands links, lists, and headings
- `//go:` directives are comments to the parser but instructions to the toolchain

## Coming From Another Language

| Other | Go |
|---|---|
| JSDoc `/** @param {string} s */` | prose only; the signature already carries the types |
| Python docstrings — inside the function | Go doc comments go *above* the declaration |
| Rust `///` and `//!` | plain `//`, position is what makes it a doc comment |
| Rust `#[doc]` / `cargo doc` | `go doc` on the command line, pkg.go.dev on the web |
| Sphinx/reST markup | a small fixed syntax: blank-line paragraphs, indented code, `[Link]` |

## The two comment forms

```go
package main

import "fmt"

func main() {
  // a line comment

  /* a block comment */

  /*
     a multi-line
     block comment
  */

  fmt.Println("x") // trailing
  fmt.Println( /* inline */ "y")
}
```

Output:

```
x
y
```

A line comment runs to the end of the line. A block comment containing a newline acts as a newline for the semicolon-insertion rules, which is why `/* */` on one line can sit mid-expression but a multi-line one cannot.

Idiomatic: `//` everywhere, including for multi-line comments. `/* */` appears in real Go almost only in generated code and in license headers.

## Block comments do not nest

```go
package main

import "fmt"

func main() {
  /* outer
  /* inner */
  fmt.Println("after")
  */
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:4: syntax error: unexpected /, expected expression
```

The comment ended at the first `*/`, so `fmt.Println("after")` became live code and the dangling `*/` is a syntax error. This is why you cannot comment out a region that already contains a block comment. Editors handle it by inserting `//` on every line, and so should you.

## A doc comment is a comment with no blank line before the declaration

```go
package main

import "fmt"

// Area returns the area of a rectangle.
func Area(w, h float64) float64 { return w * h }

// Shape is a 2D figure.
type Shape struct {
  // Width is the horizontal extent.
  Width float64

  // Height is the vertical extent.
  Height float64
}

// MaxSides is the largest polygon this package handles.
const MaxSides = 12

// Registry holds every known shape by name.
var Registry = map[string]Shape{}

// This is NOT a doc comment, because of the blank line below it.

func NotDocumented() {}

func main() {
  fmt.Println(Area(2, 3), MaxSides, len(Registry))
}
```

Output:

```
6 12 0
```

There is no `@param`, no `@returns`, and no type annotations — the signature is already machine-readable. The convention is that the comment is a complete sentence starting with the identifier's name: `// Area returns ...`, not `// Returns the area`. That way the generated docs read correctly when the name is stripped or when the line appears in a one-line summary.

A blank line between the comment and the declaration severs the association. Struct fields, constants, variables, methods, and interface methods all take doc comments the same way.

## Package doc and the doc.go convention

The comment above `package X` is the package doc. Exactly one file in a package should have it; when it grows past a few lines, the convention is to move it alone into a `doc.go`.

```go
// Package temperature converts between Celsius, Fahrenheit, and Kelvin.
//
// All conversions are exact within float64 precision. Values below
// [AbsoluteZeroC] are rejected with [ErrBelowAbsoluteZero].
//
// # Usage
//
// Convert a reading and format it:
//
//	c := temperature.Celsius(21.5)
//	fmt.Println(c.Fahrenheit())
//
// # Scales
//
// The supported scales are:
//   - Celsius, the package's canonical type
//   - Fahrenheit
//   - Kelvin
//
// See [the Kelvin definition] for the SI basis.
//
// [the Kelvin definition]: https://en.wikipedia.org/wiki/Kelvin
package temperature
```

Package docs start with `Package <name>` by convention. For a `package main`, document the *command*: `// Command foo does X.`

## The gofmt doc comment syntax (Go 1.19+)

Doc comments have a small, fixed syntax that `gofmt` parses and normalizes. There are five constructs and no more:

| Construct | Written as |
|---|---|
| paragraph | separated by a `//` line with nothing after it |
| heading | `// # Heading` |
| list item | `//   - item`, indented with a space before the `-` |
| code block | a line indented with a **tab** after the `//` |
| doc link | `[Name]`, `[Type.Method]`, `[pkg.Name]` |
| URL link | `[text]` plus a `// [text]: https://...` definition at the end |

`gofmt` rewrites comments that are close but not conforming. Given this:

```go
package fmtdoc

// Process handles input.
// Steps:
// * validate
// * transform
// Example:
//	Process("x")
//Tight comment with no space.
func Process(s string) string { return s }
```

`gofmt -d` reports:

```
--- bad.go.orig
+++ bad.go
@@ -5,6 +5,8 @@
 // * validate
 // * transform
 // Example:
+//
 //	Process("x")
-//Tight comment with no space.
+//
+// Tight comment with no space.
 func Process(s string) string { return s }
```

It inserted the blank `//` line a code block requires, and it added the missing space after `//`. Note what it did *not* do: `* validate` is left alone, because `*` is not a list marker in Go doc comments — only `-` is. Those two lines will render as ordinary prose. Run `gofmt` and read the diff; it is the fastest way to learn the syntax.

## Reading the docs with go doc

Given the `temperature` package above with documented declarations, `go doc ./temperature` prints the package doc plus a one-line index:

```
package temperature // import "example.com/docdemo/temperature"

Package temperature converts between Celsius, Fahrenheit, and Kelvin.

All conversions are exact within float64 precision. Values below AbsoluteZeroC
are rejected with ErrBelowAbsoluteZero.

# example.com/scratch

Convert a reading and format it:

    c := temperature.Celsius(21.5)
    fmt.Println(c.Fahrenheit())

# example.com/scratch

The supported scales are:
  - Celsius, the package's canonical type
  - Fahrenheit
  - Kelvin

See the Kelvin definition for the SI basis.

[the Kelvin definition]: https://en.wikipedia.org/wiki/Kelvin

var ErrBelowAbsoluteZero = errors.New("temperature below absolute zero")
func CToF(c float64) float64
func Validate(c Celsius) error
type Celsius float64
    const AbsoluteZeroC Celsius = -273.15
    func FromFahrenheit(f Fahrenheit) Celsius
type Fahrenheit float64
type Kelvin float64
```

Note the grouping: `AbsoluteZeroC` and `FromFahrenheit` are listed *under* `Celsius`, because a constant of a type and a function returning it belong to that type. This is automatic, and it is why a constructor named `NewX` or `FromY` returning `X` shows up in the right place without any annotation.

The `[the Kelvin definition]` link renders as plain text in the terminal and as a hyperlink on pkg.go.dev.

`go doc <pkg> <Symbol>` prints one declaration with its full comment:

```
package temperature // import "example.com/docdemo/temperature"

func Validate(c Celsius) error
    Validate reports whether c is physically possible.

    It returns ErrBelowAbsoluteZero when c is colder than AbsoluteZeroC.
```

A method takes `Type.Method`:

```
package temperature // import "example.com/docdemo/temperature"

func (c Celsius) Fahrenheit() Fahrenheit
    Fahrenheit converts c to the Fahrenheit scale.
```

A type prints its own doc plus everything attached to it:

```
package temperature // import "example.com/docdemo/temperature"

type Celsius float64
    Celsius is a temperature in degrees Celsius.

const AbsoluteZeroC Celsius = -273.15
func FromFahrenheit(f Fahrenheit) Celsius
func (c Celsius) Fahrenheit() Fahrenheit
func (c Celsius) Kelvin() Kelvin
```

`go doc -all` prints every exported declaration with its full comment, grouped into `CONSTANTS`, `VARIABLES`, `FUNCTIONS`, and `TYPES` sections. The `FUNCTIONS` section of the same package:

```
FUNCTIONS

func CToF(c float64) float64
    CToF converts Celsius to Fahrenheit.

    Deprecated: use Celsius.Fahrenheit instead. CToF will be removed in v2.

func Validate(c Celsius) error
    Validate reports whether c is physically possible.

    It returns ErrBelowAbsoluteZero when c is colder than AbsoluteZeroC.
```

It works on the stdlib and on any module in your build:

```
package strings // import "strings"

func Contains(s, substr string) bool
    Contains reports whether substr is within s.
```

| Command | Shows |
|---|---|
| `go doc` | the package in the current directory |
| `go doc ./temperature` | that package's doc and index |
| `go doc -all ./temperature` | every exported declaration with full comments |
| `go doc ./temperature Celsius` | one type plus its constants, constructors, methods |
| `go doc ./temperature Celsius.Fahrenheit` | one method |
| `go doc strings.Contains` | a stdlib symbol |
| `go doc -u ./temperature` | unexported declarations too |
| `go doc -src ./temperature Validate` | the source of the declaration |
| `go doc -cmd` | a `main` package's exported symbols |

## Unexported declarations are hidden

```
doc: no symbol round2 in package example.com/docdemo/temperature
```

That is `go doc ./temperature round2` on an unexported function. Add `-u` and it appears:

```
package temperature // import "example.com/docdemo/temperature"

func round2(f float64) float64
```

So unexported identifiers still deserve comments — for the next maintainer, and because `go doc -u` is how you read an unfamiliar internal package.

## The Deprecated marker

A paragraph beginning exactly `Deprecated: ` is a recognized convention. `gopls`, `staticcheck`, and pkg.go.dev all key off it.

```go
// CToF converts Celsius to Fahrenheit.
//
// Deprecated: use [Celsius.Fahrenheit] instead. CToF will be removed in v2.
func CToF(c float64) float64 {
  return c*9/5 + 32
}
```

It must be its own paragraph (a blank `//` line before it), it is case-sensitive, and the convention is to say what to use instead. Note from the `-all` output above that the symbol is **not** hidden from the docs — deprecation is advisory.

## Example functions document by executing

A function named `ExampleX` in a `_test.go` file with an `// Output:` comment is both documentation and a test.

```go
package temperature_test

import (
  "fmt"

  "example.com/docdemo/temperature"
)

func ExampleCelsius_Fahrenheit() {
  c := temperature.Celsius(100)
  fmt.Println(c.Fahrenheit())
  // Output: 212
}

func ExampleValidate() {
  fmt.Println(temperature.Validate(-300))
  // Output: temperature below absolute zero
}
```

`go test ./temperature` runs them and compares stdout against the `// Output:` text:

```
ok  	example.com/docdemo/temperature	0.491s
```

Change the expected output to `999` and it fails like any other test:

```
--- FAIL: ExampleCelsius_Fahrenheit (0.00s)
got:
212
want:
999
FAIL
FAIL	example.com/docdemo/temperature	0.767s
FAIL
```

The naming is what binds an example to a declaration: `Example`, `ExampleValidate`, `ExampleCelsius`, `ExampleCelsius_Fahrenheit` (type and method separated by `_`), and `ExampleValidate_negative` for a second example with a lowercase suffix. These are the only doc comments that cannot rot. Full coverage in 14.7.

## Directive comments are not comments semantically

`//go:` directives have no space after `//` and are instructions to the toolchain, not prose.

```go
//go:build !windows

//go:generate echo generating

// Package main shows directive comments.
package main

import "fmt"

func main() {
  fmt.Println("unix only")
}
```

`go run .` prints:

```
unix only
```

`go generate ./...` prints:

```
generating
```

And `go doc .` prints only the prose:

```
Package main shows directive comments.
```

The directives are excluded from the doc comment even when adjacent. If you put one *inside* the doc comment, `gofmt` moves it out with a blank line:

```go
// Package main shows what happens when a directive sits inside the doc comment.
//go:generate echo generating
package main
```

`gofmt -d` reports:

```
@@ -1,4 +1,5 @@
 // Package main shows what happens when a directive sits inside the doc comment.
+//
 //go:generate echo generating
 package main
```

Common directives: `//go:build` for conditional compilation (15.4), `//go:generate` for code generation (15.8), `//go:embed` for embedding files (13.13), and `//go:noinline` and friends for the compiler. The rule is no space: `// go:build` is an ordinary comment and silently does nothing — a classic way to lose an afternoon.

## Quick Reference

| Syntax | Meaning |
|---|---|
| `// text` | line comment |
| `/* text */` | block comment; does not nest |
| `// X does ...` above `func X` | doc comment |
| blank line between comment and decl | not a doc comment |
| `// Package x ...` above `package x` | package doc |
| `doc.go` | the file convention for a long package doc |
| `//` alone | paragraph break inside a doc comment |
| `// # Heading` | a doc heading |
| `//   - item` | a list item |
| `//\ttext` (tab) | a code block |
| `[Name]`, `[T.Method]`, `[pkg.Name]` | doc link |
| `// [text]: https://...` | URL link definition |
| `// Deprecated: ...` | its own paragraph; tooling recognizes it |
| `//go:directive` | toolchain instruction; **no space** after `//` |

| Convention | Example |
|---|---|
| start with the identifier | `// Area returns the area ...` |
| complete sentences, ending in a period | |
| package doc starts with `Package` | `// Package temperature converts ...` |
| command doc starts with `Command` | `// Command serve runs the API.` |
| bool-returning function | `// Valid reports whether ...` |
| error variable | `// ErrX is returned when ...` |

## Common Mistakes

**Trying to comment out a region that contains a `/* */` comment.** The inner `*/` closes the outer one; prefix every line with `//` instead.

**Leaving a blank line between the doc comment and the declaration.** The association is lost and the comment disappears from `go doc`.

**Writing `// go:generate` with a space.** It becomes an ordinary comment and does nothing; directives take no space.

**Writing `// Returns the area of the rectangle.`** Start with the identifier so the summary line reads correctly: `// Area returns ...`.

**Restating the types in the comment.** The signature already has them; explain the contract, the units, the error cases, and the zero value instead.

## Exercises

1. Write a two-file package with `doc.go` holding a package doc that uses all five doc constructs: a heading, a list, an indented code block, a `[Symbol]` link, and a URL link. Verify each renders correctly in `go doc` and that `gofmt -l` is silent.
2. Deliberately break your doc comments five ways (blank line before the declaration, `*` bullets, no blank line before a code block, a space in a directive, a `Deprecated:` that is not its own paragraph). Run `gofmt -d` and `go doc` after each and record which tool catches which.
3. Document an exported type with four methods, a constructor, and a sentinel error. Run `go doc` on the package and confirm the grouping puts the constructor and the error where you expect.
4. Write an `ExampleX` for a function that prints map contents, make it fail because of iteration order, and fix it with `// Unordered output:`. Explain why that directive exists.
5. Take any unfamiliar stdlib package and learn it using only `go doc`, `go doc -all`, and `go doc -src`, without opening a browser. Write down the three commands that were most useful.

---

[← 2.6 Variadic Functions and Closures](06-variadic-closures.md) · [Course Index](../README.md) · [3.1 if, else, and the if-with-statement Form →](../03-control-flow/01-if.md)
