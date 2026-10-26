[← 13.14 iter: Range-over-Func Iterators](14-iter.md) · [Course Index](../README.md) · [14.1 go test and the Basics →](../14-testing/01-go-test.md)

# 13.15 — reflect: The Basics

## What You'll Learn

- The `Type` / `Value` / `Kind` triad, and why **Type is not Kind** (a named type's Kind is its underlying type)
- Reading struct fields and tags, consistent with 4.7
- **Addressability**: you must pass a pointer and call `.Elem()` — with every real panic captured
- `DeepEqual`'s three traps: NaN, nil-vs-empty, and unexported fields
- Three complete realistic uses — struct→map, an `env:` config loader, a `validate:` checker
- The honest cost, benchmarked: index access is cheap, `FieldByName` is ~12x

## Coming From Another Language

| Other | Go |
|---|---|
| Python `getattr(obj, "x")` | `reflect.ValueOf(o).FieldByName("X")` — exported fields only |
| JS `obj[key] = v` on anything | you need a **pointer** and `.Elem()`, or it panics |
| Python/JS decorators and metaclasses | no hooks; you walk the type yourself at runtime |
| Java annotations with runtime retention | struct tags, but they are **plain strings** with no schema |
| Rust has no runtime reflection | Go has it, and the stdlib's `encoding/json` is built on it |
| duck typing everywhere | reflection is the **exception**; the type system is the rule |

Reflection in Go is the escape hatch that `encoding/json`, `database/sql`, template
engines and DI containers are built from. It converts compile-time errors into runtime
panics, so the rule is: **use it at the edges, where data crosses into or out of your
program, and never in a hot path.**

## TypeOf, ValueOf, and Kind

Three concepts. A **`Type`** describes a type. A **`Value`** holds a value plus its Type. A
**`Kind`** is the small enum of structural categories — `Int`, `String`, `Struct`, `Slice`,
`Map`, `Ptr`, `Func`, `Chan`, `Interface`, and so on.

```go
package main

import (
  "fmt"
  "reflect"
  "time"
)

type Celsius float64
type Point struct{ X, Y int }
type Named Point

func main() {
  fmt.Printf("%-22s %-22s %-10s\n", "value", "Type", "Kind")
  for _, v := range []any{
    42, "text", 3.5, true, Celsius(21.5),
    Point{1, 2}, Named{1, 2}, &Point{1, 2},
    []int{1, 2}, [2]int{1, 2}, map[string]int{"a": 1},
    time.Second, time.Time{}, nil, []byte("ab"), struct{ A int }{1},
  } {
    t := reflect.TypeOf(v)
    if t == nil {
      fmt.Printf("%-22v %-22s %-10s\n", v, "<nil> (a nil Type!)", "Invalid")
      continue
    }
    fmt.Printf("%-22v %-22s %-10s\n", trunc(fmt.Sprint(v)), t, t.Kind())
  }
  fmt.Println("chan/func Kinds:", reflect.TypeFor[chan int]().Kind(), reflect.TypeFor[func()]().Kind())

  fmt.Println("\n--- Type vs Kind: the crucial distinction ---")
  t := reflect.TypeFor[Celsius]()
  fmt.Printf("  Celsius: Type=%v Kind=%v Name=%q PkgPath=%q\n", t, t.Kind(), t.Name(), t.PkgPath())
  fmt.Println("  Kind == Float64:", t.Kind() == reflect.Float64,
    " Type == float64:", t == reflect.TypeFor[float64]())
  nt := reflect.TypeFor[Named]()
  fmt.Println("  Named vs Point: same Kind:", nt.Kind() == reflect.TypeFor[Point]().Kind(),
    " same Type:", nt == reflect.TypeFor[Point]())

  fmt.Println("\n--- composite types have no Name; Elem/Key unwrap them ---")
  at := reflect.TypeFor[struct{ A int }]()
  fmt.Printf("  anonymous struct: Name=%q String=%q\n", at.Name(), at.String())
  pt := reflect.TypeFor[*Point]()
  fmt.Println("  *Point: Kind:", pt.Kind(), " Elem():", pt.Elem(), " Elem().Kind():", pt.Elem().Kind())
  mt := reflect.TypeFor[map[string][]int]()
  fmt.Println("  map: Key():", mt.Key(), " Elem():", mt.Elem(), " Elem().Elem():", mt.Elem().Elem())

  fmt.Println("\n--- Type metadata ---")
  for _, tt := range []reflect.Type{
    reflect.TypeFor[int](), reflect.TypeFor[Point](),
    reflect.TypeFor[[]int](), reflect.TypeFor[time.Time](),
  } {
    fmt.Printf("  %-14v Size=%-3d Comparable=%-6v NumMethod=%d\n",
      tt, tt.Size(), tt.Comparable(), tt.NumMethod())
  }
}

func trunc(s string) string {
  if len(s) > 20 {
    return s[:17] + "..."
  }
  return s
}
```

Output:

```
value                  Type                   Kind      
42                     int                    int       
text                   string                 string    
3.5                    float64                float64   
true                   bool                   bool      
21.5                   main.Celsius           float64   
{1 2}                  main.Point             struct    
{1 2}                  main.Named             struct    
&{1 2}                 *main.Point            ptr       
[1 2]                  []int                  slice     
[1 2]                  [2]int                 array     
map[a:1]               map[string]int         map       
1s                     time.Duration          int64     
0001-01-01 00:00:...   time.Time              struct    
<nil>                  <nil> (a nil Type!)    Invalid   
[97 98]                []uint8                slice     
{1}                    struct { A int }       struct    
chan/func Kinds: chan func

--- Type vs Kind: the crucial distinction ---
  Celsius: Type=main.Celsius Kind=float64 Name="Celsius" PkgPath="main"
  Kind == Float64: true  Type == float64: false
  Named vs Point: same Kind: true  same Type: false

--- composite types have no Name; Elem/Key unwrap them ---
  anonymous struct: Name="" String="struct { A int }"
  *Point: Kind: ptr  Elem(): main.Point  Elem().Kind(): struct
  map: Key(): string  Elem(): []int  Elem().Elem(): int

--- Type metadata ---
  int            Size=8   Comparable=true   NumMethod=0
  main.Point     Size=16  Comparable=true   NumMethod=0
  []int          Size=24  Comparable=false  NumMethod=0
  time.Time      Size=24  Comparable=true   NumMethod=43
```

**Type is not Kind, and confusing them is the first real bug.** `Celsius`'s Type is
`main.Celsius` but its Kind is `float64`; `Named`'s Kind is `struct`, the same Kind as
`Point`, while its Type is different. So:

- **Switch on `Kind()`** when you care about structure — "is this an integer I can
  `SetInt` on?", "can I iterate it?". That is what a decoder does, and it is why the env
  loader below handles `Celsius` and `UserID` for free.
- **Compare `Type`** when you care about identity — "is this exactly a `time.Duration`?".
  `time.Duration`'s Kind is `int64`, so a Kind switch would format it as a number; the
  loader below checks `fv.Type() == reflect.TypeFor[time.Duration]()` **before** the Kind
  switch for precisely this reason.

**`reflect.TypeFor[T]()`** (Go 1.22+) is the modern way to get a Type from a type
parameter or a literal type, replacing `reflect.TypeOf(T{})` and the
`reflect.TypeOf((*T)(nil)).Elem()` dance needed for interfaces. 18.2 uses it.

**`reflect.TypeOf(nil)` returns a nil `Type`**, not a Type whose Kind is Invalid — so
`reflect.TypeOf(v).Kind()` on a nil `any` is a nil-pointer panic. Check for nil first.

**Anonymous types have an empty `Name()`** and an empty `PkgPath()`; `String()` is the
only readable form. `Name()` is also empty for `[]int`, `map[k]v`, `*T` and every other
composite type — only *named* types have names.

The full Kind list: `Bool`, `Int`…`Int64`, `Uint`…`Uint64`, `Uintptr`, `Float32`,
`Float64`, `Complex64`, `Complex128`, `Array`, `Chan`, `Func`, `Interface`, `Map`,
`Pointer` (printed `ptr`), `Slice`, `String`, `Struct`, `UnsafePointer`, plus `Invalid`
for the zero `Value`.

### The method reference

| `Type` method | Returns |
|---|---|
| `Name()` / `PkgPath()` / `String()` | the bare name / import path / full name |
| `Kind()` | the structural category |
| `Size()` / `Align()` | bytes |
| `Comparable()` | whether `==` is defined |
| `NumField()` / `Field(i)` / `FieldByName(n)` | struct fields |
| `NumMethod()` / `Method(i)` / `MethodByName(n)` | the exported method set |
| `Elem()` | the pointed-to / element type (`Ptr`, `Slice`, `Array`, `Map`, `Chan`) |
| `Key()` | a map's key type |
| `In(i)` / `Out(i)` / `NumIn()` / `NumOut()` / `IsVariadic()` | func signatures |
| `Implements(i)` / `AssignableTo(t)` / `ConvertibleTo(t)` | relationships |

| `Value` method | Returns |
|---|---|
| `Kind()` / `Type()` | as above |
| `Int()` / `Uint()` / `Float()` / `String()` / `Bool()` / `Bytes()` | the typed value — **panics on the wrong Kind** |
| `Interface()` | the value as `any` — panics on an unexported-derived Value |
| `Len()` / `Cap()` / `Index(i)` | slices, arrays, strings |
| `NumField()` / `Field(i)` / `FieldByName(n)` / `FieldByIndex(ix)` | struct fields |
| `MapKeys()` / `MapIndex(k)` / `MapRange()` | maps |
| `Elem()` | dereference a pointer, or unwrap an interface |
| `IsValid()` / `IsZero()` / `IsNil()` | state |
| `CanAddr()` / `CanSet()` / `CanInterface()` | capability checks |

`String()` is the exception to "panics on the wrong Kind": it returns
`"<main.Point Value>"` rather than panicking, which makes a mistaken `.String()` silently
produce garbage instead of failing loudly.

## Struct fields and tags

```go
package main

import (
  "fmt"
  "reflect"
)

type User struct {
  ID        int      `json:"id" db:"user_id" validate:"required"`
  Email     string   `json:"email,omitempty" validate:"required,email"`
  Age       int      `json:"-"`
  Tags      []string `json:"tags"`
  internal  string
  Untagged  float64
  Malformed string `json: "bad"`
}

func main() {
  t := reflect.TypeFor[User]()
  fmt.Printf("%-10s %-10s %-8s %-18s %-10s %s\n",
    "Name", "Type", "Exported", "json", "db", "Offset")
  for i := range t.NumField() {
    f := t.Field(i)
    fmt.Printf("%-10s %-10v %-8v %-18q %-10q %d\n",
      f.Name, f.Type, f.IsExported(), f.Tag.Get("json"), f.Tag.Get("db"), f.Offset)
  }

  fmt.Println("\n--- Tag.Get vs Tag.Lookup: absent vs empty vs MALFORMED ---")
  for _, name := range []string{"ID", "Untagged", "Age", "Malformed"} {
    f, _ := t.FieldByName(name)
    v, ok := f.Tag.Lookup("json")
    fmt.Printf("  %-10s Get=%-10q Lookup=(%q, %v)  whole tag=%q\n",
      name, f.Tag.Get("json"), v, ok, f.Tag)
  }
  _, ok := t.FieldByName("Nope")
  fmt.Println("  FieldByName(\"Nope\") found:", ok)

  fmt.Println("\n--- reading values ---")
  u := User{ID: 7, Email: "ada@example.com", Age: 36, Tags: []string{"admin", "beta"}}
  v := reflect.ValueOf(u)
  fmt.Println("  Field(0).Int():      ", v.Field(0).Int())
  fmt.Println("  FieldByName(Email):  ", v.FieldByName("Email").String())
  fmt.Println("  Tags Len/Index(0):   ", v.FieldByName("Tags").Len(), v.FieldByName("Tags").Index(0))
  fmt.Println("  Interface() round trip:", v.Interface().(User).ID)
  iv := v.FieldByName("internal")
  fmt.Println("  unexported: IsValid:", iv.IsValid(), " CanInterface:", iv.CanInterface())

  fmt.Println("\n--- IsValid / IsZero / IsNil ---")
  var nilSlice []int
  var nilPtr *User
  for _, x := range []any{0, "", User{}, 7, nilSlice, nilPtr} {
    rv := reflect.ValueOf(x)
    isNil := "n/a (would PANIC)"
    switch rv.Kind() {
    case reflect.Slice, reflect.Map, reflect.Pointer:
      isNil = fmt.Sprint(rv.IsNil())
    }
    fmt.Printf("  %-14v IsZero=%-6v IsNil=%s\n", rv.Type(), rv.IsZero(), isNil)
  }
  zero := reflect.ValueOf(nil)
  fmt.Println("  reflect.ValueOf(nil): IsValid =", zero.IsValid(), " Kind =", zero.Kind())

  fmt.Println("\n--- maps ---")
  mv := reflect.ValueOf(map[string]int{"a": 1, "b": 2})
  fmt.Println("  Len:", mv.Len(), " MapIndex(\"a\"):", mv.MapIndex(reflect.ValueOf("a")).Int())
  fmt.Println("  MapIndex missing IsValid:", mv.MapIndex(reflect.ValueOf("z")).IsValid())
}
```

`go vet` flags the deliberately-broken tag before the program runs:

```
main.go:15:3: struct field tag `json: "bad"` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
```

Output:

```
Name       Type       Exported json               db         Offset
ID         int        true     "id"               "user_id"  0
Email      string     true     "email,omitempty"  ""         8
Age        int        true     "-"                ""         24
Tags       []string   true     "tags"             ""         32
internal   string     false    ""                 ""         56
Untagged   float64    true     ""                 ""         72
Malformed  string     true     ""                 ""         80

--- Tag.Get vs Tag.Lookup: absent vs empty vs MALFORMED ---
  ID         Get="id"       Lookup=("id", true)  whole tag="json:\"id\" db:\"user_id\" validate:\"required\""
  Untagged   Get=""         Lookup=("", false)  whole tag=""
  Age        Get="-"        Lookup=("-", true)  whole tag="json:\"-\""
  Malformed  Get=""         Lookup=("", false)  whole tag="json: \"bad\""
  FieldByName("Nope") found: false

--- reading values ---
  Field(0).Int():       7
  FieldByName(Email):   ada@example.com
  Tags Len/Index(0):    2 admin
  Interface() round trip: 7
  unexported: IsValid: true  CanInterface: false

--- IsValid / IsZero / IsNil ---
  int            IsZero=true   IsNil=n/a (would PANIC)
  string         IsZero=true   IsNil=n/a (would PANIC)
  main.User      IsZero=true   IsNil=n/a (would PANIC)
  int            IsZero=false  IsNil=n/a (would PANIC)
  []int          IsZero=true   IsNil=true
  *main.User     IsZero=true   IsNil=true
  reflect.ValueOf(nil): IsValid = false  Kind = invalid

--- maps ---
  Len: 2  MapIndex("a"): 1
  MapIndex missing IsValid: false
```

**`Tag.Get` returns `""` for both "absent" and "present but empty"; `Tag.Lookup` tells
them apart.** Look at `Malformed`: `Lookup` returned `("", false)` even though the field
clearly has a tag string, because `json: "bad"` (with a space) does not parse as a valid
tag pair, so the whole tag is unparseable and `Get`/`Lookup` see nothing. Tags are plain
strings with a loose convention, and only `go vet` protects you — the same point 4.7
makes.

**`IsZero()` is the general emptiness test** and works on every Kind, including structs;
`IsNil()` **panics** unless the Kind is a pointer, slice, map, chan, func or interface,
which is why the loop above guards it. For "is this field unset?", `IsZero()` is almost
always what you want.

**The zero `reflect.Value` has `IsValid() == false` and Kind `invalid`.** That is what you
get from a missing `FieldByName`, a missing `MapIndex`, or `reflect.ValueOf(nil)`. Calling
most methods on it panics, so `IsValid()` is the mandatory guard after any lookup that
might fail — and note `FieldByName` returns the zero Value rather than an `ok` flag, so the
check is on you.

**An unexported field is readable as a `Value` but `CanInterface()` is false** — calling
`.Interface()` on it panics. `encoding/json` skips unexported fields for exactly this
reason (13.6).

**`f.Offset` and `f.Index`** give the byte offset within the struct and the index path for
`FieldByIndex`, which is how you reach a promoted field of an embedded struct (shown
below).

## Addressability and settability

The thing everyone hits. `reflect.ValueOf(x)` receives a **copy** — mutating it could not
possibly affect `x`, so reflect refuses rather than silently discarding your write.

```go
package main

import (
  "fmt"
  "reflect"
)

type Config struct {
  Name    string
  Port    int
  Tags    []string
  private string
}

func main() {
  c := Config{Name: "api", Port: 8080}

  fmt.Println("--- a non-pointer Value is NOT addressable ---")
  v := reflect.ValueOf(c)
  fmt.Println("  CanAddr:", v.CanAddr(), " CanSet:", v.CanSet(), " Field(0).CanSet:", v.Field(0).CanSet())

  fmt.Println("\n--- pass a POINTER and call Elem() ---")
  pv := reflect.ValueOf(&c)
  ev := pv.Elem()
  fmt.Println("  ptr  CanSet:", pv.CanSet())
  fmt.Println("  Elem CanAddr:", ev.CanAddr(), " CanSet:", ev.CanSet(), " Field(0).CanSet:", ev.Field(0).CanSet())
  ev.FieldByName("Name").SetString("worker")
  ev.FieldByName("Port").SetInt(9090)
  ev.FieldByName("Tags").Set(reflect.ValueOf([]string{"a", "b"}))
  fmt.Printf("  after mutation: %+v\n", c)
  pf := ev.FieldByName("private")
  fmt.Println("  unexported field: CanAddr:", pf.CanAddr(), " CanSet:", pf.CanSet())

  fmt.Println("\n--- slices, arrays and maps differ ---")
  s := []int{1, 2, 3}
  sv := reflect.ValueOf(s)
  fmt.Println("  slice from ValueOf: CanAddr:", sv.CanAddr(), " Index(0).CanSet:", sv.Index(0).CanSet())
  sv.Index(0).SetInt(99)
  fmt.Println("    after Index(0).SetInt(99):", s)
  a := [3]int{1, 2, 3}
  fmt.Println("  array from ValueOf: Index(0).CanSet:", reflect.ValueOf(a).Index(0).CanSet())
  reflect.ValueOf(&a).Elem().Index(0).SetInt(99)
  fmt.Println("    via a pointer:", a)
  m := map[string]int{"a": 1}
  mv := reflect.ValueOf(m)
  mv.SetMapIndex(reflect.ValueOf("b"), reflect.ValueOf(2))
  mv.SetMapIndex(reflect.ValueOf("a"), reflect.Value{})
  fmt.Println("  map SetMapIndex (add b, delete a with the zero Value):", m)

  fmt.Println("\n--- New, MakeSlice, MakeMap, Append, Zero ---")
  np := reflect.New(reflect.TypeFor[Config]())
  np.Elem().FieldByName("Name").SetString("made by reflect")
  fmt.Printf("  reflect.New: %T %+v\n", np.Interface(), np.Interface())
  ms := reflect.MakeSlice(reflect.TypeFor[[]int](), 2, 5)
  fmt.Println("  MakeSlice len/cap:", ms.Len(), ms.Cap())
  fmt.Println("  after Append:", reflect.Append(ms, reflect.ValueOf(7), reflect.ValueOf(8)).Interface())
  mm := reflect.MakeMap(reflect.TypeFor[map[string]int]())
  mm.SetMapIndex(reflect.ValueOf("k"), reflect.ValueOf(1))
  fmt.Println("  MakeMap:", mm.Interface())
  fmt.Printf("  Zero(Config): %+v\n", reflect.Zero(reflect.TypeFor[Config]()).Interface())

  fmt.Println("\n--- Convert vs Assign ---")
  iv := reflect.ValueOf(42)
  f64 := reflect.TypeFor[float64]()
  fmt.Println("  int ConvertibleTo float64:", iv.Type().ConvertibleTo(f64),
    " AssignableTo:", iv.Type().AssignableTo(f64))
  fmt.Println("  Convert to float64:", iv.Convert(f64).Interface())
  fmt.Printf("  Convert int->string: %q (a RUNE, not \"42\")\n",
    iv.Convert(reflect.TypeFor[string]()).Interface())
}
```

Output:

```
--- a non-pointer Value is NOT addressable ---
  CanAddr: false  CanSet: false  Field(0).CanSet: false

--- pass a POINTER and call Elem() ---
  ptr  CanSet: false
  Elem CanAddr: true  CanSet: true  Field(0).CanSet: true
  after mutation: {Name:worker Port:9090 Tags:[a b] private:}
  unexported field: CanAddr: true  CanSet: false

--- slices, arrays and maps differ ---
  slice from ValueOf: CanAddr: false  Index(0).CanSet: true
    after Index(0).SetInt(99): [99 2 3]
  array from ValueOf: Index(0).CanSet: false
    via a pointer: [99 2 3]
  map SetMapIndex (add b, delete a with the zero Value): map[b:2]

--- New, MakeSlice, MakeMap, Append, Zero ---
  reflect.New: *main.Config &{Name:made by reflect Port:0 Tags:[] private:}
  MakeSlice len/cap: 2 5
  after Append: [0 0 7 8]
  MakeMap: map[k:1]
  Zero(Config): {Name: Port:0 Tags:[] private:}

--- Convert vs Assign ---
  int ConvertibleTo float64: true  AssignableTo: false
  Convert to float64: 42
  Convert int->string: "*" (a RUNE, not "42")
```

**The rule: `reflect.ValueOf(&x).Elem()` is the only way in.** Note the pointer Value
itself has `CanSet: false` — it is also a copy — and `.Elem()` is what produces the
addressable struct behind it. Every reflect-based decoder (`json.Unmarshal`, the env loader
below) takes a pointer for this reason, which is why passing a non-pointer to
`json.Unmarshal` is an error (13.6).

**Three different answers for slices, arrays and maps.** A slice's **elements are
settable even from `ValueOf`** (`sv.Index(0).SetInt(99)` mutated `s`) because a slice value
contains a pointer to a shared backing array (7.4) — the element is addressable through it.
An **array's** are not, because an array value is the data. And a **map** is settable via
`SetMapIndex` without any pointer, because the map header points at shared state; passing
the zero `reflect.Value{}` as the value **deletes** the key.

**An unexported field is `CanAddr` but not `CanSet`.** Reflection respects export rules;
there is no legitimate way around it.

**`Convert` is the runtime `T(v)` and follows the same rules** — including that
`int → string` produces the **rune** `"*"` (code point 42), not `"42"`. For the digits you
want `strconv`. Note `AssignableTo` is false for `int → float64` while `ConvertibleTo` is
true: assignability is what `Set` requires, convertibility is what `Convert` requires.

### The panics

```go
package main

import (
  "fmt"
  "reflect"
)

type Config struct{ Name string }

func main() {
  c := Config{Name: "api"}
  v := reflect.ValueOf(c)
  fmt.Println("about to Set a field of a non-pointer Value")
  v.FieldByName("Name").SetString("boom")
}
```

This panics on purpose. Output (reflect's own frames, which contain varying addresses, are
elided):

```
about to Set a field of a non-pointer Value
panic: reflect: reflect.Value.SetString using unaddressable value

goroutine 1 [running]:
...
main.main()
	/tmp/scratch/main.go:14 +0xb0
exit status 2
```

With `private string` as the field and `reflect.ValueOf(&c).Elem()`:

```
about to Set an unexported field
panic: reflect: reflect.Value.SetString using value obtained using unexported field
```

With `Port int` and `SetString("8080")`:

```
about to SetString an int field
panic: reflect: call of reflect.Value.SetString on int Value
```

And on the zero `Value`, `Kind()` is safe but `Interface()` is not:

```go
package main

import (
  "fmt"
  "reflect"
)

func main() {
  var v reflect.Value
  fmt.Println("IsValid:", v.IsValid())
  fmt.Println("calling Kind() on the zero Value is fine:", v.Kind())
  fmt.Println("but Interface() is not:")
  fmt.Println(v.Interface())
}
```

This panics on purpose. Output:

```
IsValid: false
calling Kind() on the zero Value is fine: invalid
but Interface() is not:
panic: reflect: call of reflect.Value.Interface on zero Value
```

Those four messages are worth recognizing on sight. **`using unaddressable value`** means
you forgot the pointer. **`using value obtained using unexported field`** means your
decoder is walking a field it should skip — check `f.IsExported()`. **`on int Value`**
means your Kind switch has a hole. **`on zero Value`** means a lookup failed and you did
not check `IsValid()`. The guards `CanSet()`, `IsExported()`, `Kind()` and `IsValid()`
prevent all four, and every production reflect code path has them.

## Methods, Implements, and DeepEqual

```go
package main

import (
  "fmt"
  "math"
  "reflect"
)

type Greeter struct{ Prefix string }

func (g Greeter) Greet(name string) string { return g.Prefix + ", " + name + "!" }

func (g Greeter) Add(a, b int) (int, error) {
  if a < 0 || b < 0 {
    return 0, fmt.Errorf("negative input: %d, %d", a, b)
  }
  return a + b, nil
}

func (g *Greeter) SetPrefix(p string) { g.Prefix = p }

func (g Greeter) String() string { return "Greeter(" + g.Prefix + ")" }

type Stringer interface{ String() string }

type Inner struct{ A int }
type Outer struct {
  Inner
  B      int
  hidden string
}

func main() {
  g := Greeter{Prefix: "Hello"}
  v := reflect.ValueOf(g)
  t := v.Type()
  pt := reflect.TypeFor[*Greeter]()

  fmt.Println("--- a VALUE's method set excludes pointer methods ---")
  fmt.Println("  Greeter NumMethod:", t.NumMethod(), " *Greeter NumMethod:", pt.NumMethod())
  for i := range t.NumMethod() {
    fmt.Printf("    %-10s %v\n", t.Method(i).Name, t.Method(i).Type)
  }

  fmt.Println("\n--- calling by name ---")
  m := v.MethodByName("Greet")
  fmt.Println("  bound method NumIn:", m.Type().NumIn(), "(the receiver is already bound)")
  fmt.Printf("  Greet(\"Ada\") = %q\n", m.Call([]reflect.Value{reflect.ValueOf("Ada")})[0].String())
  add := v.MethodByName("Add")
  r1 := add.Call([]reflect.Value{reflect.ValueOf(2), reflect.ValueOf(3)})
  fmt.Printf("  Add(2, 3) = %d, err=%v\n", r1[0].Int(), r1[1].Interface())
  r2 := add.Call([]reflect.Value{reflect.ValueOf(-1), reflect.ValueOf(3)})
  fmt.Printf("  Add(-1, 3) = %d, err=%v\n", r2[0].Int(), r2[1].Interface())

  pg := &Greeter{Prefix: "Hi"}
  reflect.ValueOf(pg).MethodByName("SetPrefix").Call([]reflect.Value{reflect.ValueOf("Hey")})
  fmt.Println("  SetPrefix via a pointer:", pg.Prefix)
  fmt.Println("  via the value, IsValid():", v.MethodByName("SetPrefix").IsValid())
  fmt.Println("  a missing method IsValid():", v.MethodByName("Nope").IsValid())

  si := reflect.TypeFor[Stringer]()
  fmt.Println("  Greeter/*Greeter/int implement Stringer:",
    t.Implements(si), pt.Implements(si), reflect.TypeFor[int]().Implements(si))

  fmt.Println("\n--- embedded fields flatten, with Anonymous and an Index path ---")
  ot := reflect.TypeFor[Outer]()
  for i := range ot.NumField() {
    f := ot.Field(i)
    fmt.Printf("  %-8s %-12v Anonymous=%-6v Index=%v\n", f.Name, f.Type, f.Anonymous, f.Index)
  }
  af, _ := ot.FieldByName("A")
  fmt.Println("  promoted \"A\" Index:", af.Index,
    " FieldByIndex:", reflect.ValueOf(Outer{Inner: Inner{A: 1}}).FieldByIndex(af.Index).Int())

  fmt.Println("\n--- DeepEqual and its three traps ---")
  var nilS []int
  var nilM map[string]int
  nan := math.NaN()
  fmt.Println("  equal structs/slices/maps: ", reflect.DeepEqual(Inner{1}, Inner{1}),
    reflect.DeepEqual([]int{1, 2}, []int{1, 2}),
    reflect.DeepEqual(map[string]int{"a": 1}, map[string]int{"a": 1}))
  fmt.Println("  nil vs empty slice:        ", reflect.DeepEqual(nilS, []int{}), "<- TRAP 1 (len is equal)")
  fmt.Println("  nil vs empty map:          ", reflect.DeepEqual(nilM, map[string]int{}))
  fmt.Println("  NaN vs itself:             ", reflect.DeepEqual(nan, nan), "<- TRAP 2")
  fmt.Println("  []float64{NaN} vs itself:  ", reflect.DeepEqual([]float64{nan}, []float64{nan}))
  fmt.Println("  unexported fields:         ", reflect.DeepEqual(
    Outer{hidden: "a"}, Outer{hidden: "b"}), "<- TRAP 3 (it SEES them)")
  fmt.Println("  int vs int64:              ", reflect.DeepEqual(1, int64(1)))
  fmt.Println("  pointers to equal values:  ", reflect.DeepEqual(&Inner{1}, &Inner{1}))
  fmt.Println("  == on a comparable struct: ", Inner{1} == Inner{1})
}
```

Output:

```
--- a VALUE's method set excludes pointer methods ---
  Greeter NumMethod: 3  *Greeter NumMethod: 4
    Add        func(main.Greeter, int, int) (int, error)
    Greet      func(main.Greeter, string) string
    String     func(main.Greeter) string

--- calling by name ---
  bound method NumIn: 1 (the receiver is already bound)
  Greet("Ada") = "Hello, Ada!"
  Add(2, 3) = 5, err=<nil>
  Add(-1, 3) = 0, err=negative input: -1, 3
  SetPrefix via a pointer: Hey
  via the value, IsValid(): false
  a missing method IsValid(): false
  Greeter/*Greeter/int implement Stringer: true true false

--- embedded fields flatten, with Anonymous and an Index path ---
  Inner    main.Inner   Anonymous=true   Index=[0]
  B        int          Anonymous=false  Index=[1]
  hidden   string       Anonymous=false  Index=[2]
  promoted "A" Index: [0 0]  FieldByIndex: 1

--- DeepEqual and its three traps ---
  equal structs/slices/maps:  true true true
  nil vs empty slice:         false <- TRAP 1 (len is equal)
  nil vs empty map:           false
  NaN vs itself:              false <- TRAP 2
  []float64{NaN} vs itself:   false
  unexported fields:          false <- TRAP 3 (it SEES them)
  int vs int64:               false
  pointers to equal values:   true
  == on a comparable struct:  true
```

**Methods are listed alphabetically, and `Method(i).Type` includes the receiver** as the
first parameter — hence `func(main.Greeter, string) string` for a one-argument method. A
`Value.MethodByName(...)` is already bound, so `NumIn()` is 1, not 2.

**The value's method set excludes pointer methods** (3 vs 4), so
`v.MethodByName("SetPrefix")` is the invalid zero Value. Call it through
`reflect.ValueOf(&g)`.

**`Call` takes and returns `[]reflect.Value`**, and a returned `error` arrives as a
`reflect.Value` you `.Interface()` to inspect. Argument count and types must match exactly
or `Call` panics — use `Type().NumIn()`/`In(i)` to check first. There is also `CallSlice`
for variadics.

**Embedded fields are flattened with `Anonymous: true`** and promoted fields get a
multi-element `Index` path (`[0 0]`) for `FieldByIndex` — the same promotion rules that
make JSON embedding flatten (13.6, 4.5).

**`DeepEqual`'s three traps**, all visible above:

1. **`nil` vs empty is `false`** for slices and maps, even though `len` is equal. A handler
   returning `nil` where a test expects `[]` fails, and the message is unhelpful.
2. **`NaN` is never equal to itself**, so any struct or slice containing a `NaN` fails
   against an identical copy (13.9).
3. **It compares unexported fields**, so a struct with a cached field, a `sync.Mutex`, or a
   `time.Time`'s monotonic clock reading can differ when the data you care about does not.

So: prefer `==` for comparable structs, `slices.Equal`/`maps.Equal` for collections (which
treat nil and empty as equal), a hand-written `Equal` method for domain types, and
`DeepEqual` only in tests on data you fully control. 4.3 covers comparability;
`google/go-cmp` is the third-party answer when you need configurable comparison.

## Three realistic uses

### 1. Struct to map, honoring json tags

```go
package main

import (
  "fmt"
  "reflect"
  "slices"
  "strings"
)

type Address struct {
  City string `json:"city"`
  Zip  string `json:"zip,omitempty"`
}

type User struct {
  ID       int      `json:"id"`
  Name     string   `json:"name"`
  Email    string   `json:"email,omitempty"`
  Password string   `json:"-"`
  Tags     []string `json:"tags,omitempty"`
  Address  Address  `json:"address"`
  internal int
  NoTag    bool
}

func StructToMap(v any) (map[string]any, error) {
  rv := reflect.ValueOf(v)
  for rv.Kind() == reflect.Pointer {
    if rv.IsNil() {
      return nil, fmt.Errorf("structToMap: nil pointer")
    }
    rv = rv.Elem()
  }
  if rv.Kind() != reflect.Struct {
    return nil, fmt.Errorf("structToMap: want a struct, got %s", rv.Kind())
  }
  out := map[string]any{}
  rt := rv.Type()
  for i := range rt.NumField() {
    f := rt.Field(i)
    if !f.IsExported() {
      continue
    }
    name, opts, _ := strings.Cut(f.Tag.Get("json"), ",")
    if name == "-" && opts == "" {
      continue
    }
    if name == "" {
      name = f.Name
    }
    fv := rv.Field(i)
    if slices.Contains(strings.Split(opts, ","), "omitempty") && fv.IsZero() {
      continue
    }
    if fv.Kind() == reflect.Struct && fv.Type().Name() != "" && fv.NumField() > 0 {
      nested, err := StructToMap(fv.Interface())
      if err != nil {
        return nil, err
      }
      out[name] = nested
      continue
    }
    out[name] = fv.Interface()
  }
  return out, nil
}

func main() {
  u := User{ID: 7, Name: "Ada", Email: "ada@example.com", Password: "secret",
    Tags: []string{"admin"}, Address: Address{City: "London"}}

  m, err := StructToMap(u)
  fmt.Println("err:", err)
  for _, k := range slices.Sorted(mapKeys(m)) {
    fmt.Printf("  %-9s %#v\n", k, m[k])
  }

  fmt.Println("\n--- omitempty drops the zero fields; json:\"-\" is always dropped ---")
  m2, _ := StructToMap(User{ID: 1, Name: "Bob", Password: "hunter2"})
  for _, k := range slices.Sorted(mapKeys(m2)) {
    fmt.Printf("  %-9s %#v\n", k, m2[k])
  }

  fmt.Println("\n--- a pointer works, and so do the error cases ---")
  m3, err := StructToMap(&User{ID: 2, Name: "Cy"})
  fmt.Println("  pointer: keys =", len(m3), "err =", err)
  _, err = StructToMap(42)
  fmt.Println("  non-struct:", err)
  var nilUser *User
  _, err = StructToMap(nilUser)
  fmt.Println("  nil pointer:", err)
}

func mapKeys(m map[string]any) func(func(string) bool) {
  return func(yield func(string) bool) {
    for k := range m {
      if !yield(k) {
        return
      }
    }
  }
}
```

Output:

```
err: <nil>
  NoTag     false
  address   map[string]interface {}{"city":"London"}
  email     "ada@example.com"
  id        7
  name      "Ada"
  tags      []string{"admin"}

--- omitempty drops the zero fields; json:"-" is always dropped ---
  NoTag     false
  address   map[string]interface {}{"city":""}
  id        1
  name      "Bob"

--- a pointer works, and so do the error cases ---
  pointer: keys = 4 err = <nil>
  non-struct: structToMap: want a struct, got int
  nil pointer: structToMap: nil pointer
```

The three habits every reflect function needs are all here: **unwrap pointers in a loop**
(a `**T` is legal), **check the Kind** and return an error rather than panicking, and
**skip unexported fields** before touching them. `Zip` is absent from the nested map
because of `omitempty`, `Password` is gone because of `json:"-"`, and `NoTag` falls back
to its Go field name.

### 2. An env-var config loader

This is the same design [18.2](../18-production/02-config.md) builds, made standalone and
testable by injecting the lookup function.

```go
package main

import (
  "errors"
  "fmt"
  "reflect"
  "strconv"
  "strings"
  "time"
)

type Config struct {
  Port        int           `env:"PORT" default:"8080"`
  Host        string        `env:"HOST" default:"0.0.0.0"`
  DatabaseURL string        `env:"DATABASE_URL,required"`
  Timeout     time.Duration `env:"TIMEOUT" default:"5s"`
  Debug       bool          `env:"DEBUG" default:"false"`
  Origins     []string      `env:"ORIGINS" default:"http://localhost:3000"`
  Ignored     string
  internal    int
}

func Load(dst any, lookup func(string) (string, bool)) error {
  v := reflect.ValueOf(dst)
  if v.Kind() != reflect.Pointer || v.IsNil() || v.Elem().Kind() != reflect.Struct {
    return errors.New("env: destination must be a non-nil pointer to struct")
  }
  v = v.Elem()
  t := v.Type()
  for i := range t.NumField() {
    f := t.Field(i)
    if !f.IsExported() {
      continue
    }
    tag, ok := f.Tag.Lookup("env")
    if !ok {
      continue
    }
    name, opts, _ := strings.Cut(tag, ",")
    required := opts == "required"

    raw, found := lookup(name)
    if !found {
      if required {
        return fmt.Errorf("env: %s is required", name)
      }
      raw = f.Tag.Get("default")
      if raw == "" {
        continue
      }
    }
    if err := setField(v.Field(i), raw); err != nil {
      return fmt.Errorf("env: %s=%q: %w", name, raw, err)
    }
  }
  return nil
}

func setField(fv reflect.Value, raw string) error {
  if fv.Type() == reflect.TypeFor[time.Duration]() {
    d, err := time.ParseDuration(raw)
    if err != nil {
      return err
    }
    fv.SetInt(int64(d))
    return nil
  }
  switch fv.Kind() {
  case reflect.String:
    fv.SetString(raw)
  case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
    n, err := strconv.ParseInt(raw, 10, fv.Type().Bits())
    if err != nil {
      return err
    }
    fv.SetInt(n)
  case reflect.Bool:
    b, err := strconv.ParseBool(raw)
    if err != nil {
      return err
    }
    fv.SetBool(b)
  case reflect.Slice:
    if fv.Type().Elem().Kind() != reflect.String {
      return fmt.Errorf("unsupported slice element %s", fv.Type().Elem())
    }
    parts := strings.Split(raw, ",")
    for i := range parts {
      parts[i] = strings.TrimSpace(parts[i])
    }
    fv.Set(reflect.ValueOf(parts))
  default:
    return fmt.Errorf("unsupported kind %s", fv.Kind())
  }
  return nil
}

func env(m map[string]string) func(string) (string, bool) {
  return func(k string) (string, bool) { v, ok := m[k]; return v, ok }
}

func main() {
  fmt.Println("--- defaults plus one required var ---")
  var c Config
  err := Load(&c, env(map[string]string{"DATABASE_URL": "postgres://localhost/app"}))
  fmt.Println("  err:", err)
  fmt.Printf("  %+v\n", c)

  fmt.Println("\n--- everything overridden from the environment ---")
  var c2 Config
  err = Load(&c2, env(map[string]string{
    "DATABASE_URL": "postgres://db/prod",
    "PORT":         "9090",
    "HOST":         "127.0.0.1",
    "TIMEOUT":      "30s",
    "DEBUG":        "true",
    "ORIGINS":      "https://a.com, https://b.com",
  }))
  fmt.Println("  err:", err)
  fmt.Printf("  %+v\n", c2)

  fmt.Println("\n--- the error paths ---")
  var c3 Config
  fmt.Println("  missing required:", Load(&c3, env(map[string]string{})))
  fmt.Println("  bad int:         ", Load(&c3, env(map[string]string{
    "DATABASE_URL": "x", "PORT": "not-a-port"})))
  fmt.Println("  bad duration:    ", Load(&c3, env(map[string]string{
    "DATABASE_URL": "x", "TIMEOUT": "30"})))
  fmt.Println("  bad bool:        ", Load(&c3, env(map[string]string{
    "DATABASE_URL": "x", "DEBUG": "yes"})))
  fmt.Println("  not a pointer:   ", Load(Config{}, env(nil)))
  var nilp *Config
  fmt.Println("  nil pointer:     ", Load(nilp, env(nil)))
  fmt.Println("  pointer to int:  ", Load(new(int), env(nil)))
}
```

Output:

```
--- defaults plus one required var ---
  err: <nil>
  {Port:8080 Host:0.0.0.0 DatabaseURL:postgres://localhost/app Timeout:5s Debug:false Origins:[http://localhost:3000] Ignored: internal:0}

--- everything overridden from the environment ---
  err: <nil>
  {Port:9090 Host:127.0.0.1 DatabaseURL:postgres://db/prod Timeout:30s Debug:true Origins:[https://a.com https://b.com] Ignored: internal:0}

--- the error paths ---
  missing required: env: DATABASE_URL is required
  bad int:          env: PORT="not-a-port": strconv.ParseInt: parsing "not-a-port": invalid syntax
  bad duration:     env: TIMEOUT="30": time: missing unit in duration "30"
  bad bool:         env: DEBUG="yes": strconv.ParseBool: parsing "yes": invalid syntax
  not a pointer:    env: destination must be a non-nil pointer to struct
  nil pointer:      env: destination must be a non-nil pointer to struct
  pointer to int:   env: destination must be a non-nil pointer to struct
```

Sixty lines replace a hundred lines of hand-written `os.Getenv` plus `strconv` plus
error-wrapping, and adding a field to `Config` now requires no loader change at all. That
is the trade reflection offers.

Four details that matter. **`fv.Type() == reflect.TypeFor[time.Duration]()` comes before
the Kind switch**, because `Duration`'s Kind is `int64` — the Type-vs-Kind distinction
doing real work. **`fv.Type().Bits()` is passed to `ParseInt`** so an `int8` field
overflows at 127 rather than silently wrapping. **`strconv.ParseBool` accepts only
`1`/`t`/`T`/`TRUE`/`true`/`True` and the false equivalents**, so `yes` is an error — the
same strictness 18.2 documents. And every failure is **wrapped with the variable name and
the raw value**, which is the difference between a usable startup error and "invalid
syntax".

### 3. A tag-driven validator

```go
package main

import (
  "fmt"
  "reflect"
  "strconv"
  "strings"
)

type FieldError struct {
  Field string
  Rule  string
  Msg   string
}

func (e FieldError) Error() string { return e.Field + ": " + e.Msg }

type ValidationErrors []FieldError

func (v ValidationErrors) Error() string {
  parts := make([]string, len(v))
  for i, e := range v {
    parts[i] = e.Error()
  }
  return strings.Join(parts, "; ")
}

type SignupRequest struct {
  Email    string   `json:"email" validate:"required,email"`
  Password string   `json:"password" validate:"required,min=8"`
  Age      int      `json:"age" validate:"min=13,max=120"`
  Nickname string   `json:"nickname"`
  Website  string   `json:"website" validate:"url"`
  Tags     []string `json:"tags" validate:"required"`
}

func Validate(v any) error {
  rv := reflect.ValueOf(v)
  for rv.Kind() == reflect.Pointer {
    rv = rv.Elem()
  }
  if rv.Kind() != reflect.Struct {
    return fmt.Errorf("validate: want a struct, got %s", rv.Kind())
  }
  var errs ValidationErrors
  rt := rv.Type()
  for i := range rt.NumField() {
    f := rt.Field(i)
    if !f.IsExported() {
      continue
    }
    tag := f.Tag.Get("validate")
    if tag == "" {
      continue
    }
    name := f.Name
    if j, _, _ := strings.Cut(f.Tag.Get("json"), ","); j != "" && j != "-" {
      name = j
    }
    fv := rv.Field(i)
    for _, rule := range strings.Split(tag, ",") {
      key, arg, _ := strings.Cut(rule, "=")
      if err := apply(name, key, arg, fv); err != nil {
        errs = append(errs, *err)
      }
    }
  }
  if len(errs) > 0 {
    return errs
  }
  return nil
}

func apply(name, rule, arg string, fv reflect.Value) *FieldError {
  fail := func(msg string) *FieldError { return &FieldError{Field: name, Rule: rule, Msg: msg} }
  switch rule {
  case "required":
    if fv.IsZero() {
      return fail("is required")
    }
  case "email":
    if s := fv.String(); s != "" && !strings.Contains(s, "@") {
      return fail("must be a valid email")
    }
  case "url":
    if s := fv.String(); s != "" && !strings.HasPrefix(s, "http") {
      return fail("must start with http")
    }
  case "min":
    n, _ := strconv.Atoi(arg)
    switch fv.Kind() {
    case reflect.String:
      if fv.Len() > 0 && fv.Len() < n {
        return fail("must be at least " + arg + " characters")
      }
    case reflect.Int, reflect.Int64:
      if fv.Int() != 0 && fv.Int() < int64(n) {
        return fail("must be at least " + arg)
      }
    }
  case "max":
    n, _ := strconv.Atoi(arg)
    if fv.Kind() == reflect.Int && fv.Int() > int64(n) {
      return fail("must be at most " + arg)
    }
  }
  return nil
}

func main() {
  fmt.Println("--- a valid request ---")
  ok := SignupRequest{Email: "ada@example.com", Password: "hunter2password",
    Age: 36, Website: "https://ada.dev", Tags: []string{"beta"}}
  fmt.Println("  err:", Validate(ok))

  fmt.Println("\n--- an empty request: every required rule fires ---")
  if err := Validate(SignupRequest{}); err != nil {
    for _, e := range err.(ValidationErrors) {
      fmt.Printf("  %-9s %-9s %s\n", e.Field, e.Rule, e.Msg)
    }
  }

  fmt.Println("\n--- individual rule failures ---")
  bad := SignupRequest{Email: "not-an-email", Password: "short", Age: 150,
    Website: "ftp://x", Tags: []string{"a"}}
  if err := Validate(bad); err != nil {
    for _, e := range err.(ValidationErrors) {
      fmt.Printf("  %-9s %-9s %s\n", e.Field, e.Rule, e.Msg)
    }
    fmt.Println("\n  Error():", err)
  }

  fmt.Println("\n--- too young, and a non-struct ---")
  fmt.Println(" ", Validate(SignupRequest{Email: "a@b.c", Password: "longenough", Age: 10,
    Website: "http://x", Tags: []string{"a"}}))
  fmt.Println(" ", Validate(42))
}
```

Output:

```
--- a valid request ---
  err: <nil>

--- an empty request: every required rule fires ---
  email     required  is required
  password  required  is required
  tags      required  is required

--- individual rule failures ---
  email     email     must be a valid email
  password  min       must be at least 8 characters
  age       max       must be at most 120
  website   url       must start with http

  Error(): email: must be a valid email; password: must be at least 8 characters; age: must be at most 120; website: must start with http

--- too young, and a non-struct ---
  age: must be at least 13
  validate: want a struct, got int
```

**It collects *all* the errors rather than returning the first**, which is what an API
needs: one 422 response listing every bad field. The `FieldError` carries the **JSON** name
(from the `json` tag), not the Go field name, so the response keys match the request keys —
the detail that makes a validator usable from a browser form.

This is the mechanism behind `go-playground/validator` (16.8), which has fifty rules,
cross-field conditions and i18n messages. Eighty lines gets you the useful 80%, and the
real lesson is that it is *just* a tag read plus a Kind switch.

## The cost

```go
package main

import (
  "reflect"
  "testing"
)

type Rec struct {
  A int
  B string
  C float64
}

var r = Rec{A: 1, B: "x", C: 2.5}
var anyR any = r

func BenchmarkDirectRead(b *testing.B) {
  var sink int
  for b.Loop() {
    sink = r.A
  }
  _ = sink
}

func BenchmarkReflectRead(b *testing.B) {
  var sink int64
  for b.Loop() {
    sink = reflect.ValueOf(anyR).Field(0).Int()
  }
  _ = sink
}

func BenchmarkReflectReadInterface(b *testing.B) {
  var sink any
  for b.Loop() {
    sink = reflect.ValueOf(anyR).Field(1).Interface()
  }
  _ = sink
}

func BenchmarkReflectReadCachedValue(b *testing.B) {
  v := reflect.ValueOf(r)
  var sink int64
  for b.Loop() {
    sink = v.Field(0).Int()
  }
  _ = sink
}

func BenchmarkReflectFieldByName(b *testing.B) {
  v := reflect.ValueOf(r)
  var sink int64
  for b.Loop() {
    sink = v.FieldByName("A").Int()
  }
  _ = sink
}

func BenchmarkDirectWrite(b *testing.B) {
  var x Rec
  for b.Loop() {
    x.A = 42
  }
  _ = x
}

func BenchmarkReflectWrite(b *testing.B) {
  var x Rec
  v := reflect.ValueOf(&x).Elem()
  for b.Loop() {
    v.Field(0).SetInt(42)
  }
  _ = x
}

func BenchmarkTypeOf(b *testing.B) {
  for b.Loop() {
    _ = reflect.TypeOf(anyR)
  }
}

func BenchmarkTagLookup(b *testing.B) {
  t := reflect.TypeFor[Rec]()
  for b.Loop() {
    _, _ = t.Field(0).Tag.Lookup("json")
  }
}
```

`go test -bench=. -benchmem`. Machine-specific (darwin/arm64, Apple M4 Pro).

Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkDirectRead-12                	633229509	         1.612 ns/op	       0 B/op	       0 allocs/op
BenchmarkReflectRead-12               	623518198	         1.882 ns/op	       0 B/op	       0 allocs/op
BenchmarkReflectReadInterface-12      	362590070	         3.323 ns/op	       0 B/op	       0 allocs/op
BenchmarkReflectReadCachedValue-12    	699321075	         1.721 ns/op	       0 B/op	       0 allocs/op
BenchmarkReflectFieldByName-12        	63743044	        19.42 ns/op	       0 B/op	       0 allocs/op
BenchmarkDirectWrite-12               	713721572	         1.636 ns/op	       0 B/op	       0 allocs/op
BenchmarkReflectWrite-12              	511317697	         2.358 ns/op	       0 B/op	       0 allocs/op
BenchmarkTypeOf-12                    	713852302	         1.728 ns/op	       0 B/op	       0 allocs/op
BenchmarkTagLookup-12                 	100000000	        11.17 ns/op	       0 B/op	       0 allocs/op
```

**This is more nuanced than "reflection is slow", so read it carefully.**

**Indexed access is nearly free.** `Field(0).Int()` is 1.88 ns against 1.61 for `r.A`, and
`TypeOf` is 1.73 ns — these are small, allocation-free operations on precomputed type
metadata, and the compiler optimizes a lot of this microbenchmark away.

**`FieldByName` is ~12x** (19.4 vs 1.6 ns) and **`Tag.Lookup` is ~7x** (11.2 ns), because
both do **string work per call** — a linear name search and a tag re-parse. That is where
the cost actually lives, and it is why every production reflect-based library
(`encoding/json`, `sqlx`, validators) **caches a per-`reflect.Type` field plan** on first
use and then works by index. If you write reflection that runs more than occasionally, do
the same: build `map[reflect.Type][]fieldPlan` once and index into it.

**`.Interface()` costs ~2x an `.Int()`** (3.3 vs 1.9 ns) and boxes the value, which for a
non-pointer-shaped type allocates — the loader above calls `.Interface()` once per field,
not per row.

**Writing is ~1.4x reading.** `SetInt` through reflection is cheap; the expensive part was
getting the addressable Value, which you do once.

The honest summary is not "reflect is 100x slower" — it is that **a naive
`FieldByName`-per-call implementation is ~12x, a cached index-based one is close to free,
and both are irrelevant next to the JSON encoding or the database round trip they sit
inside.** Measure before you optimize, and never put uncached reflection in a loop that
runs per request per field.

## The guidance

**Generics replaced many old uses of reflect.** Before Go 1.18, a `Map`, `Filter`, `Keys`,
`Contains` or `Max` over arbitrary types meant reflection with `any` and runtime type
checks. Now they are type-parameterized, compile-time-checked and fast — `slices.Contains`
and `maps.Keys` are the obvious examples. If you are reaching for reflection to write a
generic *algorithm*, you probably want generics instead (module 9). Reflection is for when
the **shape** of the data is unknown at compile time, not just the element type.

**The legitimate uses are all at the edges**, where data crosses a boundary and you cannot
know the type in advance:

- **Encoding/decoding**: JSON, YAML, CSV, protobuf, form bodies.
- **Database mapping**: scanning rows into structs, building INSERTs from fields.
- **Configuration**: the `env:` loader above.
- **Validation**: the `validate:` checker above.
- **Dependency injection** and test fixtures, where types are registered at runtime.
- **Testing helpers**: `DeepEqual`, fuzzers, golden-file comparison.

**What you give up.** Compile-time checking: a renamed field breaks a tag-driven mapping at
runtime, in production, on the one code path nobody tested. Readability: the next reader
cannot see which fields are involved by reading the struct. Refactoring: `gopls` will not
rename a field named in a tag string. Performance, as measured. And errors become panics
unless you guard every step.

**So the rule, stated plainly:** reflection belongs in a small, well-tested package at the
edge of your system, behind a normal typed API. Write a test for the `User` struct going
through your loader, so a renamed field fails in CI rather than at 3am. Never put it in a
hot path. And if a type switch over a handful of known types would do the job, use the type
switch (5.3) — it is checked, readable and fast.

`unsafe` and `//go:linkname` go further still (unchecked memory access and linking to
private symbols); both are out of scope here and out of scope for almost all application
code.

## Quick Reference

| Need | Call |
|---|---|
| The Type of a value / of a type | `reflect.TypeOf(v)` / `reflect.TypeFor[T]()` |
| The Value of a value | `reflect.ValueOf(v)` |
| Structural category | `t.Kind()` / `v.Kind()` |
| **Mutate something** | `reflect.ValueOf(&x).Elem()` — pointer + `Elem()` |
| Is it safe to set? | `v.CanSet()` (and `f.IsExported()`) |
| Iterate struct fields | `for i := range t.NumField() { t.Field(i) }` |
| Read a tag | `f.Tag.Get("json")` / `f.Tag.Lookup("json")` for `(v, ok)` |
| A field by name | `v.FieldByName("X")` — check `IsValid()` |
| A promoted field | `v.FieldByIndex(f.Index)` |
| Dereference / unwrap | `v.Elem()` |
| Was a lookup successful? | `v.IsValid()` |
| Is it empty? | `v.IsZero()` (safe for all Kinds) |
| Allocate a `*T` | `reflect.New(t)` |
| Build a slice / map | `reflect.MakeSlice(t, len, cap)` / `reflect.MakeMap(t)` |
| The zero value of a type | `reflect.Zero(t)` |
| Runtime type conversion | `v.Convert(t)` after `t.ConvertibleTo` |
| Call a method | `v.MethodByName("M").Call([]reflect.Value{...})` |
| Does it satisfy an interface? | `t.Implements(reflect.TypeFor[I]())` |
| Deep comparison (tests) | `reflect.DeepEqual(a, b)` — see the traps |

| Trap | Reality |
|---|---|
| `reflect.ValueOf(x)` then `Set…` | **panic**: `using unaddressable value` — pass `&x` and `.Elem()` |
| Setting an unexported field | **panic**: `using value obtained using unexported field` |
| `SetString` on an `int` field | **panic**: `call of reflect.Value.SetString on int Value` |
| A method on the zero Value | **panic**: `on zero Value` — check `IsValid()` first |
| `reflect.TypeOf(nil).Kind()` | nil-pointer panic; `TypeOf(nil)` is a **nil Type** |
| `v.IsNil()` on a struct or int | panics; guard on Kind, or use `IsZero()` |
| Switching on `Kind()` for `time.Duration` | its Kind is `int64`; compare the **Type** |
| `Tag.Get` returning `""` | means absent **or** empty **or** malformed; use `Lookup` |
| A space in a tag (`json: "x"`) | the whole tag is unparseable; `go vet` catches it |
| `v.String()` on a non-string | returns `"<main.T Value>"` instead of panicking |
| `DeepEqual(nil slice, empty slice)` | **false**; use `slices.Equal` |
| `DeepEqual` on anything with a `NaN` | **false**, even against itself |
| `DeepEqual` on a struct with a mutex or cache field | compares those too |
| `FieldByName` in a loop | ~12x an indexed access; cache a plan per `reflect.Type` |
| A pointer method via a value's `MethodByName` | invalid; reflect `&x` |

## Common Mistakes

**Calling `Set` on a `reflect.ValueOf(x)`.** You have a copy; pass `&x` and call `.Elem()`,
and check `CanSet()` before writing.

**Switching on `Kind()` when you need the Type.** `time.Duration`, `Celsius` and `UserID`
all have primitive Kinds — compare `v.Type()` against `reflect.TypeFor[T]()` for exact
matches, before any Kind switch.

**Not checking `IsValid()` after `FieldByName` or `MapIndex`.** A miss returns the zero
`Value`, and the next method call panics.

**Not skipping unexported fields.** `f.IsExported()` before touching a field, or you get
the unexported-field panic on the first struct that has one.

**Using `DeepEqual` as a general equality.** A `nil` slice, a `NaN` or an unexported cache
field all give surprising answers; prefer `==`, `slices.Equal`/`maps.Equal`, or a custom
`Equal`.

**Calling `FieldByName` or `Tag.Get` per row.** Build a field plan per `reflect.Type` once
and index by position.

**Reaching for reflect to write a generic algorithm.** Generics (module 9) give you the
same thing with compile-time checks and no runtime cost.

## Exercises

1. Write `func Scan(dst any, row map[string]any) error` that fills a struct from a map
   using `db:` tags, reporting an error for a missing required column and a type mismatch.
   Table-test every failure mode, including a non-pointer destination.
2. Extend the env loader to support nested structs with a prefix (`DB_HOST` from
   `Database.Host`), `*int` for optional values, and `uint`/`float64` fields. Keep the
   error messages naming the full variable path.
3. Add a per-`reflect.Type` field-plan cache to the validator and benchmark it against the
   uncached version over 100,000 structs. Report the speedup you measure.
4. Write `func Diff(a, b any) []string` that reports which exported fields differ between
   two values of the same struct type, recursing into nested structs. Explain why this is
   better than `DeepEqual` for a test failure message.
5. Implement `func CallByName(v any, method string, args ...any) ([]any, error)` that
   validates the argument count and types with `Type().NumIn()`/`In(i)` and returns an
   error instead of panicking. Test it against a value method, a pointer method, a
   variadic method and a missing method.
6. Take one of the three examples and rewrite it without reflection (code generation, or
   an explicit `Fields()` method per type). Compare the line counts, the benchmark numbers,
   and what breaks when a field is renamed.

---

[← 13.14 iter: Range-over-Func Iterators](14-iter.md) · [Course Index](../README.md) · [14.1 go test and the Basics →](../14-testing/01-go-test.md)
