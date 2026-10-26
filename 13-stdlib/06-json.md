[← 13.5 time: Timers, Tickers, and After](05-timers.md) · [Course Index](../README.md) · [13.7 encoding/json: Streaming, Tags, and Custom Marshalers →](07-json-advanced.md)

# 13.6 — encoding/json: Marshal and Unmarshal

## What You'll Learn

- The full Go↔JSON type mapping, including `[]byte` → **base64** and nil-vs-empty slices
- Every `json` struct tag option: rename, `-`, `omitempty`, `omitzero`, `string`
- Field matching on `Unmarshal` is **case-insensitive**, and unknown fields are ignored
- The `float64`-for-all-numbers trap in `map[string]any`, with real precision loss
- Pointers for the absent/null/zero three-state problem — the most useful pattern here

## Coming From Another Language

| Other | Go |
|---|---|
| `JSON.parse` giving you a plain object | you declare a struct, or get `map[string]any` |
| `JSON.stringify` keeping insertion order | struct fields in declaration order; **map keys sorted** |
| Python `json.loads` → `dict` with ints | all numbers become `float64` unless typed |
| JS numbers are all doubles anyway | Go gives you exact `int64` *if* you type the field |
| Pydantic / `zod` rejecting unknown keys | unknown keys silently ignored by default |
| `bytes` needing manual base64 | `[]byte` is **automatically** base64 |

Go's JSON is reflection-driven and struct-first: you declare the shape and the decoder
fills it, which means you get exact `int64`s and compile-time field names, but also that
anything not in your struct vanishes without complaint. Two behaviours surprise almost
everyone: `[]byte` marshals as a base64 **string**, not an array of numbers; and
`Unmarshal` matches field names case-insensitively, so `{"USERNAME":...}` populates
`UserName`. The third thing to internalize is that Go's zero values collide with JSON's
`null` and "absent" — a `0` in an `int` field could mean the client sent 0, sent null, or
sent nothing, and the fix is a pointer.

## The type mapping

| Go | JSON | Note |
|---|---|---|
| `bool` | `true` / `false` | |
| `int`, `int64`, `float64`, … | number | exact; `NaN`/`Inf` **error** |
| `string` | string | invalid UTF-8 becomes `U+FFFD` |
| `[]byte`, `[N]byte` | **base64 string** | catches everyone |
| `[]T` non-nil | array | |
| `[]T` nil | **`null`** | not `[]` |
| `[]T` empty non-nil | `[]` | |
| `map[string]T` non-nil | object, **keys sorted** | |
| `map[string]T` nil | **`null`** | |
| `map[K]T` with integer/float K | object with stringified keys | |
| `map[bool]T` | **error** | |
| `struct` | object | exported fields only |
| `*T` non-nil | the pointee | |
| `*T` nil | `null` | |
| `time.Time` | RFC 3339 string | via its `MarshalJSON` |
| `time.Duration` | **number of nanoseconds** | no JSON methods |
| `json.RawMessage` | verbatim | |
| `chan`, `func`, `complex` | **error** | |

```go
package main

import (
  "encoding/json"
  "fmt"
  "time"
)

type inner struct {
  K string `json:"k"`
}

type withSkips struct {
  Unexported string `json:"-"`
  hidden     int
  Shown      string
}

func main() {
  vals := []any{
    nil,
    true,
    42,
    int64(1 << 62),
    "text",
    []byte("bytes!"),
    []int{1, 2, 3},
    []int{},
    []int(nil),
    map[string]int{"b": 2, "a": 1},
    map[string]int(nil),
    inner{K: "v"},
    (*inner)(nil),
    time.Date(2026, 3, 7, 15, 4, 5, 0, time.UTC),
    time.Duration(90 * time.Second),
    map[int]string{1: "a", 2: "b"},
    withSkips{},
  }
  for _, v := range vals {
    b, err := json.Marshal(v)
    if err != nil {
      fmt.Printf("%-22T ERROR: %v\n", v, err)
      continue
    }
    fmt.Printf("%-22T %s\n", v, b)
  }

  fmt.Println("\n--- things that cannot marshal ---")
  for _, v := range []any{
    make(chan int),
    func() {},
    complex(1, 2),
    map[bool]string{true: "t"},
  } {
    _, err := json.Marshal(v)
    fmt.Printf("%-20T %v\n", v, err)
  }

  _, err := json.Marshal(map[string]float64{"x": nanVal()})
  fmt.Println("NaN:", err)
}

func nanVal() float64 {
  z := 0.0
  return z / z
}
```

Output:

```
<nil>                  null
bool                   true
int                    42
int64                  4611686018427387904
string                 "text"
[]uint8                "Ynl0ZXMh"
[]int                  [1,2,3]
[]int                  []
[]int                  null
map[string]int         {"a":1,"b":2}
map[string]int         null
main.inner             {"k":"v"}
*main.inner            null
time.Time              "2026-03-07T15:04:05Z"
time.Duration          90000000000
map[int]string         {"1":"a","2":"b"}
main.withSkips         {"Shown":""}

--- things that cannot marshal ---
chan int             json: unsupported type: chan int
func()               json: unsupported type: func()
complex128           json: unsupported type: complex128
map[bool]string      json: unsupported value: jsontext: object member name must be a string after offset 2
NaN: json: unsupported value: NaN
```

The lines to read carefully.

`[]int(nil)` is **`null`** while `[]int{}` is `[]`. A JSON consumer that does
`data.items.length` crashes on the first and not the second, and the difference is purely
whether you wrote `var s []int` or `s := []int{}`. Same for maps. If your API must always
emit `[]`, initialize the slice.

`int64(1 << 62)` marshals exactly — Go has no 2^53 limit on the *encoding* side.

`map[int]string` works, with keys stringified and sorted as strings; `map[bool]string`
fails. The 1.27 error mentions `jsontext`, the new internal tokenizer.

`withSkips` emits only `{"Shown":""}` — the `json:"-"` field and the unexported `hidden`
are both dropped, the latter with no way to opt in.

### []byte is base64

```go
package main

import (
  "encoding/base64"
  "encoding/json"
  "fmt"
)

func main() {
  b, _ := json.Marshal([]byte("bytes!"))
  fmt.Println("marshal []byte(\"bytes!\"):", string(b))
  fmt.Println("base64 of \"bytes!\":     ", `"`+base64.StdEncoding.EncodeToString([]byte("bytes!"))+`"`)

  var back []byte
  json.Unmarshal(b, &back)
  fmt.Printf("round-trips: %q\n", back)

  type named []byte
  b, _ = json.Marshal(named("also base64"))
  fmt.Println("a named []byte type:", string(b))

  b, _ = json.Marshal([]int8{1, 2, 3})
  fmt.Println("[]int8 is an array:", string(b))
}
```

Output:

```
marshal []byte("bytes!"): "Ynl0ZXMh"
base64 of "bytes!":      "Ynl0ZXMh"
round-trips: "bytes!"
a named []byte type: "YWxzbyBiYXNlNjQ="
[]int8 is an array: [1,2,3]
```

`[]byte` is special-cased to base64 (standard encoding, with padding), and it round-trips
correctly — which is exactly why it is invisible until some other language's client reads
your API and gets a string where it expected numbers. A named type whose underlying type
is `[]byte` behaves the same. `[]int8` is **not** special-cased, so it emits an array;
`[]uint8` is `[]byte` and is.

Idiomatic: use `[]int` for numbers, or a named type with a `MarshalJSON` for hex (13.7).

## Struct tags

```
`json:"name,option,option"`
```

| Tag | Effect |
|---|---|
| `json:"name"` | use `name` as the JSON key |
| `json:"-"` | never marshal or unmarshal this field |
| `json:"-,"` | the key is literally `"-"` (note the trailing comma) |
| `json:",omitempty"` | omit when the value is "empty" (see below) |
| `json:",omitzero"` | omit when the value equals its zero value (Go 1.24+) |
| `json:",string"` | encode a number or bool **as a JSON string** |
| no tag | the Go field name, verbatim |

`omitempty` omits: `false`, `0`, `""`, a nil pointer, a nil interface, and **any** slice,
map, array or string of length zero. It does **not** omit a zero struct, a zero
`time.Time`, or a non-empty-but-zero array. `omitzero` omits whenever the value is the
type's zero value, which does cover structs.

```go
package main

import (
  "encoding/json"
  "fmt"
  "time"
)

type tags struct {
  Renamed  string `json:"renamed"`
  Skipped  string `json:"-"`
  Literal  string `json:"-,"`
  Default  string
  lower    string
  AsString int  `json:"as_string,string"`
  Ptr      *int `json:"ptr,omitempty"`
}

type omit struct {
  S     string          `json:"s,omitempty"`
  I     int             `json:"i,omitempty"`
  Sl    []int           `json:"sl,omitempty"`
  M     map[string]int  `json:"m,omitempty"`
  P     *int            `json:"p,omitempty"`
  St    struct{ X int } `json:"st,omitempty"`
  T     time.Time       `json:"t,omitempty"`
  A     [1]int          `json:"a,omitempty"`
}

type omitzero struct {
  S  string          `json:"s,omitzero"`
  I  int             `json:"i,omitzero"`
  St struct{ X int } `json:"st,omitzero"`
  T  time.Time       `json:"t,omitzero"`
}

func main() {
  t := tags{Renamed: "r", Skipped: "gone", Literal: "kept", Default: "d", lower: "x", AsString: 7}
  b, _ := json.MarshalIndent(t, "", "  ")
  fmt.Println(string(b))

  fmt.Println("\n--- omitempty: what it omits ---")
  b, _ = json.Marshal(omit{})
  fmt.Println("all zero:", string(b))
  n := 0
  b, _ = json.Marshal(omit{P: &n, Sl: []int{}, M: map[string]int{}})
  fmt.Println("ptr-to-zero, empty slice/map:", string(b))

  fmt.Println("\n--- omitzero (Go 1.24+) ---")
  b, err := json.Marshal(omitzero{})
  fmt.Println("all zero:", string(b), "err:", err)
  b, _ = json.Marshal(omitzero{S: "x", I: 1})
  fmt.Println("some set:", string(b))

  fmt.Println("\n--- unexported fields are skipped silently ---")
  b, _ = json.Marshal(struct {
    Shown  string
    hidden string
  }{Shown: "a", hidden: "b"})
  fmt.Println(string(b))
  var into struct {
    Shown  string
    hidden string
  }
  err = json.Unmarshal([]byte(`{"Shown":"x","hidden":"y"}`), &into)
  fmt.Printf("unmarshal: %+v err=%v\n", into, err)
}
```

Output:

```
{
  "renamed": "r",
  "-": "kept",
  "Default": "d",
  "as_string": "7"
}

--- omitempty: what it omits ---
all zero: {"st":{"X":0},"t":"0001-01-01T00:00:00Z","a":[0]}
ptr-to-zero, empty slice/map: {"p":0,"st":{"X":0},"t":"0001-01-01T00:00:00Z","a":[0]}

--- omitzero (Go 1.24+) ---
all zero: {} err: <nil>
some set: {"s":"x","i":1}

--- unexported fields are skipped silently ---
{"Shown":"a"}
unmarshal: {Shown:x hidden:} err=<nil>
```

The `omitempty` output is the whole lesson: with every field at its zero value, `st`, `t`
and `a` all **still appear**. A zero `time.Time` serializing as
`"0001-01-01T00:00:00Z"` in your API response is the single most common symptom.
`omitzero` (confirmed present on go1.27.1) omits all four and is what `omitempty` should
have been.

`json:",string"` wrapped `7` as `"7"` — that is for APIs (and JavaScript clients) that
need large integers as strings to avoid float truncation.

`json:"-,"` with the trailing comma means the key is literally `-`; `json:"-"` without it
means skip. One character apart.

Unexported fields are invisible in both directions: `hidden` was neither written nor read,
and `Unmarshal` returned no error for the `"hidden"` key in the input.

## Unmarshal behaviours that surprise

```go
package main

import (
  "encoding/json"
  "fmt"
)

type user struct {
  UserName string `json:"userName"`
  ID       int    `json:"id"`
}

type base struct {
  ID   int    `json:"id"`
  Kind string `json:"kind"`
}

type flat struct {
  base
  Name string `json:"name"`
}

type nested struct {
  Base base   `json:"base"`
  Name string `json:"name"`
}

type shadow struct {
  base
  ID string `json:"id"`
}

func main() {
  fmt.Println("--- field matching is CASE-INSENSITIVE ---")
  for _, in := range []string{
    `{"userName":"exact","id":1}`,
    `{"username":"lower","id":2}`,
    `{"USERNAME":"upper","ID":3}`,
    `{"UserName":"pascal","id":4}`,
    `{"user_name":"snake","id":5}`,
  } {
    var u user
    err := json.Unmarshal([]byte(in), &u)
    fmt.Printf("  %-30s -> %+v err=%v\n", in, u, err)
  }

  fmt.Println("\n--- unknown fields are ignored by default ---")
  var u user
  err := json.Unmarshal([]byte(`{"userName":"a","id":1,"nope":true,"extra":{"deep":1}}`), &u)
  fmt.Printf("  %+v err=%v\n", u, err)

  fmt.Println("\n--- embedded structs FLATTEN ---")
  b, _ := json.Marshal(flat{base: base{ID: 1, Kind: "k"}, Name: "n"})
  fmt.Println("  embedded:", string(b))
  b, _ = json.Marshal(nested{Base: base{ID: 1, Kind: "k"}, Name: "n"})
  fmt.Println("  named field:", string(b))

  var f flat
  json.Unmarshal([]byte(`{"id":9,"kind":"z","name":"m"}`), &f)
  fmt.Printf("  unmarshal flat: %+v\n", f)

  fmt.Println("\n--- a shallower field shadows the embedded one ---")
  b, _ = json.Marshal(shadow{base: base{ID: 1, Kind: "k"}, ID: "str"})
  fmt.Println("  shadowed:", string(b))

  fmt.Println("\n--- duplicate keys: last wins ---")
  json.Unmarshal([]byte(`{"id":1,"id":2,"id":3}`), &u)
  fmt.Printf("  %+v\n", u)
}
```

Output:

```
--- field matching is CASE-INSENSITIVE ---
  {"userName":"exact","id":1}    -> {UserName:exact ID:1} err=<nil>
  {"username":"lower","id":2}    -> {UserName:lower ID:2} err=<nil>
  {"USERNAME":"upper","ID":3}    -> {UserName:upper ID:3} err=<nil>
  {"UserName":"pascal","id":4}   -> {UserName:pascal ID:4} err=<nil>
  {"user_name":"snake","id":5}   -> {UserName: ID:5} err=<nil>

--- unknown fields are ignored by default ---
  {UserName:a ID:1} err=<nil>

--- embedded structs FLATTEN ---
  embedded: {"id":1,"kind":"k","name":"n"}
  named field: {"base":{"id":1,"kind":"k"},"name":"n"}
  unmarshal flat: {base:{ID:9 Kind:z} Name:m}

--- a shallower field shadows the embedded one ---
  shadowed: {"kind":"k","id":"str"}

--- duplicate keys: last wins ---
  {UserName:a ID:3}
```

Four things.

**Case-insensitive matching.** `username`, `USERNAME` and `UserName` all populate the
field tagged `userName`. An exact match wins if one exists, but otherwise any
case-variant matches. `user_name` does **not** — only case differs, not punctuation. This
means you cannot have two fields differing only in case, and it means a client's typo in
capitalization silently works, which hides bugs in both directions.

**Unknown fields are ignored.** A client sending `{"nmae":"typo"}` gets a 200 and an empty
name. `Decoder.DisallowUnknownFields` is the fix, covered in 13.7.

**Embedded structs flatten.** `flat` emits `id`, `kind` and `name` at the top level, with
no `base` wrapper — that is Go's promotion rules (4.5) applied to JSON. To nest, give the
field a name. This is the mechanism behind "add common fields to every response".

**A shallower field shadows a promoted one.** `shadow` has both `base.ID int` and
`ID string` tagged `id`; the outer one wins and `base.ID` is dropped entirely — silently,
with no error. Two embedded structs with the same tag at the *same* depth cancel out and
both disappear.

Duplicate keys are legal JSON and the last value wins; there is no error. See 13.7 if you
need to reject them.

## The float64 trap in map[string]any

Decoding into `any` loses all type information: JSON numbers become `float64`, always.

```go
package main

import (
  "bytes"
  "encoding/json"
  "fmt"
)

func main() {
  const doc = `{"id":9007199254740993,"big":12345678901234567890,"n":42,"f":1.5,"s":"x","b":true,"nil":null,"arr":[1,2],"obj":{"k":1}}`

  fmt.Println("--- map[string]any: every number is a float64 ---")
  var m map[string]any
  json.Unmarshal([]byte(doc), &m)
  for _, k := range []string{"id", "big", "n", "f", "s", "b", "nil", "arr", "obj"} {
    fmt.Printf("  %-5s %-12T %v\n", k, m[k], m[k])
  }

  fmt.Println("\n--- the precision loss is real ---")
  var one map[string]any
  json.Unmarshal([]byte(`{"id":9007199254740993}`), &one)
  f := one["id"].(float64)
  fmt.Println("  input:         9007199254740993")
  fmt.Printf("  as float64:    %.0f\n", f)
  fmt.Printf("  back to int64: %d\n", int64(f))
  fmt.Println("  equal to input:", int64(f) == 9007199254740993)
  out, _ := json.Marshal(one)
  fmt.Println("  re-marshalled:", string(out))

  fmt.Println("\n--- into a typed struct it is exact ---")
  var typed struct {
    ID int64 `json:"id"`
  }
  json.Unmarshal([]byte(`{"id":9007199254740993}`), &typed)
  fmt.Println("  int64 field:", typed.ID, typed.ID == 9007199254740993)

  fmt.Println("\n--- json.Number keeps the literal ---")
  dec := json.NewDecoder(bytes.NewReader([]byte(doc)))
  dec.UseNumber()
  var nm map[string]any
  dec.Decode(&nm)
  fmt.Printf("  id type: %T value: %v\n", nm["id"], nm["id"])
  num := nm["id"].(json.Number)
  i, err := num.Int64()
  fmt.Println("  Int64():", i, err, i == 9007199254740993)
  fl, _ := num.Float64()
  fmt.Printf("  Float64(): %.0f\n", fl)
  fmt.Println("  String():", num.String())

  bignum := nm["big"].(json.Number)
  i, err = bignum.Int64()
  fmt.Println("  overflowing Int64():", i, err)

  out, _ = json.Marshal(nm)
  fmt.Println("  re-marshalled with UseNumber:", string(out))

  fmt.Println("\n--- json.Number in a struct field ---")
  var sn struct {
    ID json.Number `json:"id"`
  }
  json.Unmarshal([]byte(`{"id":9007199254740993}`), &sn)
  fmt.Println("  no UseNumber needed:", sn.ID)
}
```

Output:

```
--- map[string]any: every number is a float64 ---
  id    float64      9.007199254740992e+15
  big   float64      1.2345678901234567e+19
  n     float64      42
  f     float64      1.5
  s     string       x
  b     bool         true
  nil   <nil>        <nil>
  arr   []interface {} [1 2]
  obj   map[string]interface {} map[k:1]

--- the precision loss is real ---
  input:         9007199254740993
  as float64:    9007199254740992
  back to int64: 9007199254740992
  equal to input: false
  re-marshalled: {"id":9007199254740992}

--- into a typed struct it is exact ---
  int64 field: 9007199254740993 true

--- json.Number keeps the literal ---
  id type: json.Number value: 9007199254740993
  Int64(): 9007199254740993 <nil> true
  Float64(): 9007199254740992
  String(): 9007199254740993
  overflowing Int64(): 9223372036854775807 strconv.ParseInt: parsing "12345678901234567890": value out of range
  re-marshalled with UseNumber: {"arr":[1,2],"b":true,"big":12345678901234567890,"f":1.5,"id":9007199254740993,"n":42,"nil":null,"obj":{"k":1},"s":"x"}

--- json.Number in a struct field ---
  no UseNumber needed: 9007199254740993
```

This is a genuine data-corruption bug, not a curiosity. An ID of `9007199254740993` read
into `map[string]any` and written back out becomes `9007199254740992` — a *different
record*. Snowflake IDs, Twitter IDs, database bigints and nanosecond timestamps are all in
this range.

The decoded `any` values are exactly four types plus nil: `float64`, `string`, `bool`,
`[]any`, `map[string]any`. A type switch over an unknown document only ever needs those
five cases.

Three fixes, in order of preference. Declare a struct with an `int64` field and the
decoding is exact. If the shape is genuinely unknown, use `json.Number` — either via
`Decoder.UseNumber()` for an `any` tree or as the field type directly, where no decoder
configuration is needed. `json.Number` is a `string` under the hood that keeps the original
literal, so `Int64()` is exact, `Float64()` still truncates, and `String()` gives you the
bytes as sent. Note it also round-trips `12345678901234567890` — a value no Go integer
type can hold — while reporting a clean `strconv` error if you ask for `Int64()`.

Idiomatic: a struct. Reach for `map[string]any` only for passthrough, and then prefer
`json.RawMessage` (13.7), which does not decode at all.

## Pointers for absent / null / zero

The practically most valuable pattern in this lesson. A `PATCH` endpoint must distinguish
"set quota to 0" from "don't touch quota", and a plain `int` cannot.

```go
package main

import (
  "encoding/json"
  "fmt"
)

type patchPlain struct {
  Name  string `json:"name"`
  Quota int    `json:"quota"`
}

type patchPtr struct {
  Name  *string `json:"name"`
  Quota *int    `json:"quota"`
}

func describe(p patchPtr) string {
  if p.Quota == nil {
    return "absent (leave unchanged)"
  }
  return fmt.Sprintf("present = %d", *p.Quota)
}

func main() {
  docs := []string{
    `{"name":"a","quota":10}`,
    `{"name":"a","quota":0}`,
    `{"name":"a"}`,
    `{"name":"a","quota":null}`,
  }

  fmt.Println("--- plain fields cannot tell the three cases apart ---")
  for _, d := range docs {
    var p patchPlain
    json.Unmarshal([]byte(d), &p)
    fmt.Printf("  %-26s -> Quota=%d\n", d, p.Quota)
  }

  fmt.Println("\n--- pointers distinguish absent / null / zero ---")
  for _, d := range docs {
    var p patchPtr
    json.Unmarshal([]byte(d), &p)
    fmt.Printf("  %-26s -> %s\n", d, describe(p))
  }

  fmt.Println("\n--- marshalling pointers back ---")
  q := 0
  n := "x"
  b, _ := json.Marshal(patchPtr{Name: &n, Quota: &q})
  fmt.Println("  set to zero:", string(b))
  b, _ = json.Marshal(patchPtr{Name: &n})
  fmt.Println("  nil quota:  ", string(b))
  type withOmit struct {
    Quota *int `json:"quota,omitempty"`
  }
  b, _ = json.Marshal(withOmit{})
  fmt.Println("  nil + omitempty:", string(b))
  b, _ = json.Marshal(withOmit{Quota: &q})
  fmt.Println("  ptr-to-zero + omitempty:", string(b))
}
```

Output:

```
--- plain fields cannot tell the three cases apart ---
  {"name":"a","quota":10}    -> Quota=10
  {"name":"a","quota":0}     -> Quota=0
  {"name":"a"}               -> Quota=0
  {"name":"a","quota":null}  -> Quota=0

--- pointers distinguish absent / null / zero ---
  {"name":"a","quota":10}    -> present = 10
  {"name":"a","quota":0}     -> present = 0
  {"name":"a"}               -> absent (leave unchanged)
  {"name":"a","quota":null}  -> absent (leave unchanged)

--- marshalling pointers back ---
  set to zero: {"name":"x","quota":0}
  nil quota:   {"name":"x","quota":null}
  nil + omitempty: {}
  ptr-to-zero + omitempty: {"quota":0}
```

With a plain `int`, the last three inputs all give `Quota=0` and there is no way to tell
which of them it was. With `*int`, "explicitly 0" and "not mentioned" are finally different values, which
is what a partial-update handler needs.

Be honest about the limit: `null` and absent both leave the pointer nil, so a pointer gives
you **two** of the three states, not three. Distinguishing an explicit `null` ("clear this
field") from an omitted key needs `json.RawMessage` or a custom type with an
`UnmarshalJSON` that records that it was called — see 13.7.

On the way out, `*int` nil emits `null`, and `omitempty` on a pointer does what you
actually want: nil is omitted, pointer-to-zero is emitted as `0`. That combination —
`*T` with `omitempty` — is the standard shape for an optional request field.

## Errors

| Type | Raised when |
|---|---|
| `*json.InvalidUnmarshalError` | the target is not a non-nil pointer |
| `*json.UnmarshalTypeError` | a JSON value does not fit the Go type |
| `*json.SyntaxError` | the input is not valid JSON; has an `Offset` |
| `*json.UnsupportedTypeError` | marshalling a `chan`, `func` or `complex` |
| `*json.UnsupportedValueError` | marshalling `NaN`, `±Inf`, or a cycle |
| `*json.MarshalerError` | a `MarshalJSON` implementation returned an error |

```go
package main

import (
  "encoding/json"
  "errors"
  "fmt"
)

type rec struct {
  Name  string `json:"name"`
  Count int    `json:"count"`
  Tags  []string
}

func main() {
  fmt.Println("--- Unmarshal into a non-pointer ---")
  var r rec
  target := any(r)
  err := json.Unmarshal([]byte(`{"name":"a"}`), target)
  fmt.Println("  err:", err)
  fmt.Printf("  type: %T\n", err)
  var ive *json.InvalidUnmarshalError
  if errors.As(err, &ive) {
    fmt.Printf("  Type field: %v\n", ive.Type)
  }
  err = json.Unmarshal([]byte(`{}`), nil)
  fmt.Println("  nil target:", err)
  var p *rec
  err = json.Unmarshal([]byte(`{}`), p)
  fmt.Println("  nil pointer:", err)

  fmt.Println("\n--- UnmarshalTypeError ---")
  for _, in := range []string{
    `{"count":"not a number"}`,
    `{"name":123}`,
    `{"Tags":"not an array"}`,
    `{"count":1.5}`,
    `[1,2,3]`,
  } {
    var r rec
    err := json.Unmarshal([]byte(in), &r)
    fmt.Printf("  %-26s %v\n", in, err)
    var ute *json.UnmarshalTypeError
    if errors.As(err, &ute) {
      fmt.Printf("      Value=%q Type=%v Struct=%q Field=%q Offset=%d\n",
        ute.Value, ute.Type, ute.Struct, ute.Field, ute.Offset)
    }
  }

  fmt.Println("\n--- SyntaxError ---")
  for _, in := range []string{
    `{"name":}`,
    `{"name":"a",}`,
    `{`,
    ``,
    `{"a":1} trailing`,
  } {
    var r rec
    err := json.Unmarshal([]byte(in), &r)
    var se *json.SyntaxError
    if errors.As(err, &se) {
      fmt.Printf("  %-18q Offset=%-3d %v\n", in, se.Offset, err)
    } else {
      fmt.Printf("  %-18q (not a SyntaxError) %v\n", in, err)
    }
  }

  fmt.Println("\n--- Valid ---")
  for _, in := range []string{`{"a":1}`, `"str"`, `null`, `{`, ``, `{"a":1} x`} {
    fmt.Printf("  %-12q Valid=%v\n", in, json.Valid([]byte(in)))
  }

  fmt.Println("\n--- unsupported types and values ---")
  _, err = json.Marshal(make(chan int))
  fmt.Println("  chan:", err)
  var ute2 *json.UnsupportedTypeError
  fmt.Println("  errors.As UnsupportedTypeError:", errors.As(err, &ute2))
  _, err = json.Marshal(map[string]any{"x": func() {}})
  fmt.Println("  func in a map:", err)
}
```

`go vet` flags the deliberately-wrong call on line 26 with
`call of Unmarshal passes non-pointer as second argument` — it sees through the `any(r)`
indirection, which is itself worth knowing: vet catches this class of bug before you run
the program. The rest of the output:

```
--- Unmarshal into a non-pointer ---
  err: json: Unmarshal(non-pointer main.rec)
  type: *json.InvalidUnmarshalError
  Type field: main.rec
  nil target: json: Unmarshal(nil)
  nil pointer: json: Unmarshal(nil *main.rec)

--- UnmarshalTypeError ---
  {"count":"not a number"}   json: cannot unmarshal string into Go struct field rec.count of type int
      Value="string" Type=int Struct="rec" Field="count" Offset=23
  {"name":123}               json: cannot unmarshal number into Go struct field rec.name of type string
      Value="number" Type=string Struct="rec" Field="name" Offset=11
  {"Tags":"not an array"}    json: cannot unmarshal string into Go struct field rec.Tags of type []string
      Value="string" Type=[]string Struct="rec" Field="Tags" Offset=22
  {"count":1.5}              json: cannot unmarshal number 1.5 into Go struct field rec.count of type int
      Value="number 1.5" Type=int Struct="rec" Field="count" Offset=12
  [1,2,3]                    json: cannot unmarshal array into Go value of type main.rec
      Value="array" Type=main.rec Struct="" Field="" Offset=1

--- SyntaxError ---
  "{\"name\":}"      Offset=9   invalid character '}' looking for beginning of value
  "{\"name\":\"a\",}" Offset=13  invalid character '}' looking for beginning of object key string
  "{"                Offset=1   unexpected end of JSON input
  ""                 Offset=0   unexpected end of JSON input
  "{\"a\":1} trailing" Offset=9   invalid character 't' after top-level value

--- Valid ---
  "{\"a\":1}"  Valid=true
  "\"str\""    Valid=true
  "null"       Valid=true
  "{"          Valid=false
  ""           Valid=false
  "{\"a\":1} x" Valid=false
```

followed by:

```
--- unsupported types and values ---
  chan: json: unsupported type: chan int
  errors.As UnsupportedTypeError: true
  func in a map: json: unsupported type: func()
```

`*json.UnmarshalTypeError` is the one to handle in an API: `Field` and `Struct` let you
produce `{"error":"count must be an integer"}` instead of leaking Go type names. Note
`{"count":1.5}` is a type error too — JSON has one number type, so a fractional value into
an `int` fails with the value included in `Value`.

`*json.SyntaxError.Offset` is the byte position, which is what you need to point at the
problem in a request body. `json.Valid` is the cheap pre-check — it tokenizes without
allocating a tree — and it correctly rejects trailing garbage, which bare `Unmarshal` also
does but `Decoder.Decode` does not (13.7).

`*json.UnmarshalTypeError` does not stop decoding — other fields may already be
populated — so treat a non-nil error as "the whole value is untrustworthy".

## HTML escaping and key ordering

```go
package main

import (
  "bytes"
  "encoding/json"
  "fmt"
)

type ordered struct {
  Zebra string `json:"zebra"`
  Apple string `json:"apple"`
  Mango string `json:"mango"`
}

func main() {
  fmt.Println("--- HTML escaping is ON by default ---")
  v := map[string]string{"html": `<a href="x">a & b</a>`, "u": "  "}
  b, _ := json.Marshal(v)
  fmt.Println("  Marshal:", string(b))

  var buf bytes.Buffer
  enc := json.NewEncoder(&buf)
  enc.Encode(v)
  fmt.Print("  Encoder default: ", buf.String())

  buf.Reset()
  enc = json.NewEncoder(&buf)
  enc.SetEscapeHTML(false)
  enc.Encode(v)
  fmt.Print("  SetEscapeHTML(false): ", buf.String())

  fmt.Println("\n--- key ordering ---")
  b, _ = json.Marshal(ordered{Zebra: "z", Apple: "a", Mango: "m"})
  fmt.Println("  struct (field order):", string(b))
  b, _ = json.Marshal(map[string]int{"zebra": 1, "apple": 2, "mango": 3})
  fmt.Println("  map (sorted):        ", string(b))

  fmt.Println("\n--- Encoder adds a newline, Marshal does not ---")
  b, _ = json.Marshal(map[string]int{"a": 1})
  fmt.Printf("  Marshal:  %q\n", b)
  buf.Reset()
  json.NewEncoder(&buf).Encode(map[string]int{"a": 1})
  fmt.Printf("  Encode:   %q\n", buf.String())
}
```

Output:

```
--- HTML escaping is ON by default ---
  Marshal: {"html":"<a href=\"x\">a & b</a>","u":"  "}
  Encoder default: {"html":"<a href=\"x\">a & b</a>","u":"  "}
  SetEscapeHTML(false): {"html":"<a href=\"x\">a & b</a>","u":"  "}

--- key ordering ---
  struct (field order): {"zebra":"z","apple":"a","mango":"m"}
  map (sorted):         {"apple":2,"mango":3,"zebra":1}

--- Encoder adds a newline, Marshal does not ---
  Marshal:  "{\"a\":1}"
  Encode:   "{\"a\":1}\n"
```

`<`, `>` and `&` become `<`, `>` and `&` so the output is safe to embed in a
`<script>` tag. It is still valid JSON and any decoder reads it back identically — but it
is ugly in logs and breaks byte-for-byte comparisons against fixtures. `json.Marshal` has
**no** way to turn it off; only `Encoder.SetEscapeHTML(false)` does, which is a reason to
use an `Encoder` even when writing to a buffer. `U+2028`/`U+2029` stay escaped either way,
because they are line terminators in JavaScript.

Struct keys come out in **field declaration order**; map keys come out **sorted**. So
marshalling a map is deterministic (unlike iterating one — 7.6), and reordering your struct
fields changes your API's output byte-for-byte.

`Encoder.Encode` appends a `\n`; `Marshal` does not. That trailing newline is why
`httptest` body comparisons so often fail by one character.

## Quick Reference

| Need | Call |
|---|---|
| Struct → JSON | `json.Marshal(v)` |
| Pretty JSON | `json.MarshalIndent(v, "", "  ")` |
| JSON → struct | `json.Unmarshal(data, &v)` |
| Check syntax only | `json.Valid(data)` |
| Reindent existing JSON | `json.Indent(&buf, data, "", "  ")` |
| Minify | `json.Compact(&buf, data)` |
| Rename a field | `json:"snake_name"` |
| Never serialize a field | `json:"-"` |
| Omit zero values properly | `json:",omitzero"` |
| Big integers as strings | `json:",string"` |
| Exact large numbers | an `int64` field, or `json.Number` |
| Optional request field | `*T` with `json:",omitempty"` |
| Disable HTML escaping | `enc := json.NewEncoder(w); enc.SetEscapeHTML(false)` |
| Field-level error detail | `var e *json.UnmarshalTypeError; errors.As(err, &e)` |

| Trap | Reality |
|---|---|
| `[]byte` in a response | base64 string, not an array |
| `var s []T` in a response | marshals as `null`, not `[]` |
| `omitempty` on a struct or `time.Time` | never omits; use `omitzero` |
| `{"NMAE":...}` typo | case-insensitive match may accept it |
| Unknown fields in a request | silently ignored |
| An embedded struct | flattens into the parent object |
| Two same-tag fields at one depth | both vanish, no error |
| `map[string]any` with an int64 ID | silently truncated to `float64` |
| A plain `int` for a PATCH field | cannot distinguish 0 from absent |
| `*T` for absent vs null | gives two states, not three |
| Comparing `Encode` output to a fixture | it has a trailing `\n` |
| Expecting `<` in marshalled output | it is `<` unless you use an `Encoder` |

## Common Mistakes

**Returning a nil slice from an API handler.** It marshals as `null` and breaks clients
that iterate; initialize it with `make([]T, 0)`.

**Tagging a struct or `time.Time` field `omitempty`.** It has no effect; use `omitzero`
(Go 1.24+) or a pointer.

**Decoding IDs into `map[string]any`.** Anything above 2^53 is silently corrupted; declare
an `int64` field or use `json.Number`.

**Using a plain `int`/`string` for a partial-update payload.** You cannot tell "set to
zero" from "not provided"; use `*int`.

**Assuming unknown fields are an error.** They are ignored by default — enable
`Decoder.DisallowUnknownFields` (13.7) if a typo should be rejected.

## Exercises

1. Write `func strictUnmarshal(data []byte, v any) error` that rejects unknown fields,
   duplicate keys and trailing content, returning an error naming the offending key.
   Table-test each failure mode.
2. Define a `type Optional[T any] struct { Value T; Set bool }` with `UnmarshalJSON` that
   distinguishes absent, `null` and a value — all three. Prove it with a table of the
   four inputs from the pointer example.
3. Benchmark decoding a 10,000-element array into `[]struct{...}` versus
   `[]map[string]any`, with `-benchmem`, and explain the allocation difference.
4. Write `func fieldErrors(err error) map[string]string` that walks a decode error with
   `errors.As` and turns `*json.UnmarshalTypeError` and `*json.SyntaxError` into a
   client-safe message keyed by field name.
5. Build a round-trip fuzz test: marshal a struct with every type from the mapping table,
   unmarshal it, and assert deep equality. Explain which fields cannot round-trip and why.

---

[← 13.5 time: Timers, Tickers, and After](05-timers.md) · [Course Index](../README.md) · [13.7 encoding/json: Streaming, Tags, and Custom Marshalers →](07-json-advanced.md)
