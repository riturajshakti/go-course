[← 3.6 panic and recover](../03-control-flow/06-panic-recover.md) · [Course Index](../README.md) · [4.2 Defining and Instantiating Structs →](02-structs.md)

# 4.1 — Pointers and the Absence of Pointer Arithmetic

## What You'll Learn

- `&` takes an address, `*` declares a pointer type and dereferences one — and that is the whole syntax
- Go has pointers but no pointer *arithmetic*, which is what makes them memory-safe
- `p.Field` works on a pointer with no `->`; Go dereferences for you
- Returning `&localVar` is safe: escape analysis moves the variable to the heap

## Coming From Another Language

| Other | Go |
|---|---|
| C `p++`, `p + n`, `*(p + 3)` | not expressible; `p + 1` is a compile error |
| Rust `&x` / `&mut x`, lifetimes, borrow checker | `&x` only, no lifetimes, GC instead of ownership |
| Rust `Box<T>` for heap | `&T{}` — the compiler decides stack vs heap |
| C `p->field` | `p.Field` — same thing, no second operator |
| JS/Python: every object is already a reference | you choose: `T` copies, `*T` shares |

Go's pointers are C's pointers with arithmetic removed. That one subtraction is why
Go can be garbage-collected and memory-safe: a `*int` always points at a whole
`int` or is `nil`, never one byte into the middle of something. You keep explicit
control over copying versus sharing, without the ability to construct an invalid
address.

## Taking and following an address

```go
package main

import "fmt"

func main() {
  x := 42
  p := &x
  fmt.Println(x, *p)
  fmt.Printf("%T %T\n", x, p)

  *p = 99
  fmt.Println(x)

  var q *int
  fmt.Println(q, q == nil)

  r := new(int)
  fmt.Println(*r, r != nil)

  pp := &p
  fmt.Println(**pp)
}
```

Output:

```
42 42
int *int
99
<nil> true
0 true
99
```

The zero value of every pointer type is `nil`, and `nil` prints as `<nil>`. `new(int)`
allocates a zeroed `int` and hands back its address. `**pp` is a pointer to a pointer —
legal, occasionally needed (a function that must reassign the caller's pointer), rarely
what you want.

## No pointer arithmetic

```go
package main

import "fmt"

func main() {
  x := 10
  p := &x
  p = p + 1
  fmt.Println(p)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:8:7: invalid operation: p + 1 (mismatched types *int and untyped int)
```

There is no `p++`, no `p[2]`, no casting an `int` to a pointer. To walk a contiguous
region you use a slice, which carries its own bounds. The `unsafe` package (`unsafe.Pointer`,
`uintptr`) can defeat all of this and is reserved for `runtime`, cgo, and the two or three
libraries that genuinely need it.

## Dereferencing nil panics

```go
package main

import "fmt"

type Point struct {
  X, Y int
}

func main() {
  var p *Point
  fmt.Println(p)
  fmt.Println(p.X)
}
```

This program panics on purpose. Output:

```
<nil>
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x0 pc=0x104b8b57c]

goroutine 1 [running]:
main.main()
	/tmp/scratch/main.go:12 +0x4c
exit status 2
```

Printing a nil pointer is fine. Reading a field through it is not. The `pc=` address
varies per build; the message and line number do not.

## p.Field needs no arrow

Go has no `->`. The selector auto-dereferences as many levels as needed.

```go
package main

import "fmt"

type Point struct {
  X, Y int
}

func main() {
  p := &Point{X: 1, Y: 2}
  fmt.Println(p.X, (*p).X)
  p.X = 10
  fmt.Println(*p)

  q := new(Point)
  fmt.Println(*q)
  fmt.Printf("%T %T\n", p, q)

  arr := [4]int{1, 2, 3, 4}
  ap := &arr[2]
  *ap = 30
  fmt.Println(arr)

  s := []int{1, 2, 3}
  sp := &s[0]
  *sp = 100
  fmt.Println(s)

  fmt.Println(&arr[1] == &arr[1], p == q)
}
```

Output:

```
1 1
{10 2}
{0 0}
*main.Point *main.Point
[1 2 30 4]
[100 2 3]
true false
```

Idiomatic: write `p.X`, never `(*p).X`. You can take the address of an array element or a
slice element, but not of a map element — maps move their storage, so map entries are not
addressable (see 4.2).

## new(T) vs &T{}

Both allocate and return `*T`.

| Form | Zero value? | Set fields inline? |
|---|---|---|
| `new(T)` | yes | no |
| `&T{}` | yes | yes |
| `&T{A: 1}` | remaining fields zeroed | yes |

Idiomatic: `&T{}`. `new` is mostly used for non-struct types, as in `new(int)`, and even
there `v := 0; p := &v` is just as common.

## Pointer equality compares addresses

Two pointers are `==` when they hold the same address, regardless of what they point at.

```go
package main

import "fmt"

type Point struct {
  X, Y int
}

func main() {
  a := &Point{1, 2}
  b := &Point{1, 2}
  fmt.Println(a == b, *a == *b)

  c := a
  fmt.Println(a == c)
}
```

Output:

```
false true
true
```

`a == b` is false — different allocations. `*a == *b` compares the structs field-wise and
is true. More on struct comparison in 4.3.

## Passing a value vs passing a pointer

Go is pass-by-value, always. Passing `*T` copies the pointer, not the thing it points at.

```go
package main

import "fmt"

type Counter struct {
  N int
}

func byValue(c Counter) {
  c.N++
}

func byPointer(c *Counter) {
  c.N++
}

func main() {
  c := Counter{N: 0}
  byValue(c)
  fmt.Println(c.N)
  byPointer(&c)
  fmt.Println(c.N)

  n := 5
  swapTest(&n)
  fmt.Println(n)
}

func swapTest(p *int) {
  *p *= 2
}
```

Output:

```
0
1
10
```

`byValue` incremented its own copy and threw it away. There is no out-parameter, no
`ref` keyword: if a function must change your value, it takes a pointer.

## Returning a pointer to a local is safe

This is the one place a C programmer flinches. In Go it is correct and common.

```go
package main

import "fmt"

type Config struct {
  Name string
  Port int
}

func newConfig(name string) *Config {
  c := Config{Name: name, Port: 8080}
  return &c
}

func main() {
  a := newConfig("alpha")
  b := newConfig("beta")
  a.Port = 9090
  fmt.Println(*a, *b)
  fmt.Println(a == b)
}
```

Output:

```
{alpha 9090} {beta 8080}
false
```

The compiler performs *escape analysis*: when a local's address outlives the function, the
variable is allocated on the heap instead of the stack, and the garbage collector frees it
when the last reference goes away. There is no dangling pointer to create — Go simply does
not give you a way to express one. Ask the compiler what it decided:

```bash
go build -gcflags=-m .
```

Output:

```
# example.com/scratch
./main.go:10:6: can inline newConfig
./main.go:16:16: inlining call to newConfig
./main.go:17:16: inlining call to newConfig
./main.go:19:13: inlining call to fmt.Println
./main.go:20:13: inlining call to fmt.Println
./main.go:10:16: leaking param: name
./main.go:11:2: moved to heap: c
./main.go:19:13: ... argument does not escape
./main.go:19:14: *a escapes to heap
./main.go:19:18: *b escapes to heap
./main.go:20:13: ... argument does not escape
./main.go:20:16: a == b escapes to heap
```

`moved to heap: c` is the decision. The flip side: `&T{}` is not automatically a heap
allocation. A pointer that never escapes its function stays on the stack and costs nothing,
which is why "pointers are slower because they allocate" is wrong in Go often enough to be
worth measuring rather than assuming.

## When to use a pointer at all

| Reason | Example |
|---|---|
| The callee must mutate the value | `func (c *Counter) Inc()` |
| The struct is large and copied often | a 200-byte config passed down a call chain |
| `nil` must mean "absent" | `*time.Time` for an optional timestamp |
| The value has identity, not just contents | a `*sql.DB`, an open file, a connection |
| The type contains a `sync.Mutex` or similar | copying it copies the lock, which is a bug |

Otherwise prefer a value. Small structs (a `Point`, a `time.Time`) are cheaper to copy than
to chase through a pointer, and values cannot be nil.

## Quick Reference

| Syntax | Meaning |
|---|---|
| `&x` | address of `x` (`x` must be addressable) |
| `*T` | the type "pointer to `T`" |
| `*p` | the value `p` points at |
| `new(T)` | allocate a zeroed `T`, return `*T` |
| `&T{...}` | allocate a `T` with these fields, return `*T` |
| `p.F` | field `F` of `*p`; no `->` |
| `p == q` | same address? |
| `p == nil` | is the pointer unset |
| `**T` | pointer to pointer |
| `unsafe.Pointer`, `uintptr` | the escape hatch; not for application code |

| Addressable | Not addressable |
|---|---|
| variables, struct fields of addressable values | map elements |
| array and slice elements | function and method return values |
| `*p` for any pointer `p` | constants, literals (except composite literals) |

## Common Mistakes

**Reaching for `*T` everywhere out of habit from C or Java.** Go copies cheaply; use a
value unless you need mutation, nil-as-absent, large-struct avoidance, or identity.

**Returning `&local` and worrying it dangles.** Escape analysis heap-allocates it; this is
idiomatic Go, and `go build -gcflags=-m` will show you the decision.

**Expecting `p + 1` or `p[1]` to walk memory.** Use a slice, which carries bounds; pointer
arithmetic does not exist outside `unsafe`.

**Taking `&m[k]` on a map.** Map elements are not addressable; store `*T` as the value or
read-modify-write the whole element.

**Assuming a nil-check on the outer pointer is enough.** Each pointer in a chain can be nil
independently, so `a.B.C` can still panic after `a != nil`.

## Exercises

1. Write `func incr(p *int)` and `func incrAll(ps []*int)`. Call the second on a slice built
   from a loop variable's address and confirm whether all entries alias the same `int` under
   Go 1.22+ loop-variable semantics.
2. Implement `func swap(a, b *string)` and then `func swapPtrs(a, b **string)`. Show, with
   printed values, exactly what each one changes for the caller.
3. Build a linked list with `type node struct { v int; next *node }`, write
   `func reverse(head *node) *node`, and verify the original head now points at the tail.
4. Write a function that returns `*int` for a local and a second that returns `int`. Run
   `go build -gcflags=-m` on both and quote the line that differs.
5. Make a struct with an optional field modelled as `*string` and one modelled as
   `string` plus a `bool`. Marshal both to JSON (peek ahead to 13.6) and argue which reads
   better at the call site.

---

[← 3.6 panic and recover](../03-control-flow/06-panic-recover.md) · [Course Index](../README.md) · [4.2 Defining and Instantiating Structs →](02-structs.md)
