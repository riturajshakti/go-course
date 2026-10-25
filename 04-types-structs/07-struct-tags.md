[← 4.6 Named Types, Type Definitions, and Aliases](06-named-types.md) · [Course Index](../README.md) · [4.8 Constructors and the Functional Options Pattern →](08-constructors-options.md)

# 4.7 — Struct Tags

## What You'll Learn

- A tag is a raw string literal after a field, with the conventional shape `` `key:"value,opt" key2:"v"` ``
- Tags are inert metadata; only code that calls `reflect` gives them meaning
- A malformed tag fails **silently** — `json: "a"` with a space is simply ignored
- Unexported fields are invisible to every encoder, tag or no tag

## Coming From Another Language

| Other | Go |
|---|---|
| Python `@dataclass(field(metadata={}))` | a backtick string after the field |
| TS decorators (`@Column()`) | tags carry data only; there is no decorator to execute |
| Rust `#[serde(rename = "x")]` | same purpose, but checked by the compiler in Rust and not in Go |
| Java annotations | closest analogue: read reflectively at run time |
| Pydantic `Field(alias=...)` | `json:"alias"`, enforced by `encoding/json` |

The key difference from every one of those: Go's compiler does not validate tag *contents*.
A typo in a tag is a working program with the wrong behaviour.

## Syntax

```go
type User struct {
  ID    int    `json:"id" db:"user_id" validate:"required"`
  Email string `json:"email,omitempty" db:"email" validate:"required,email"`
}
```

The rules: a raw string literal (backticks, so `"` needs no escaping) immediately after the
field's type, holding space-separated `key:"value"` pairs, where the value is conventionally
a comma-separated list whose first element is the name and the rest are options. The
convention is enforced by `reflect.StructTag`'s parser, not the compiler — the tag itself can
legally be any string.

## The json tag

```go
package main

import (
  "encoding/json"
  "fmt"
  "time"
)

type Item struct {
  Name     string    `json:"name"`
  Skipped  string    `json:"-"`
  Note     string    `json:"note,omitempty"`
  Count    int       `json:"count,string"`
  When     time.Time `json:"when,omitzero"`
  Dash     string    `json:"-,"`
  internal string
}

func main() {
  it := Item{Name: "widget", Skipped: "secret", Count: 7, Dash: "literal dash", internal: "hidden"}
  b, err := json.MarshalIndent(it, "", "  ")
  fmt.Println(err)
  fmt.Println(string(b))

  it.When = time.Date(2026, 3, 4, 5, 6, 7, 0, time.UTC)
  it.Note = "visible"
  b2, _ := json.Marshal(it)
  fmt.Println(string(b2))
}
```

Output:

```
<nil>
{
  "name": "widget",
  "count": "7",
  "-": "literal dash"
}
{"name":"widget","note":"visible","count":"7","when":"2026-03-04T05:06:07Z","-":"literal dash"}
```

| Tag | Effect |
|---|---|
| `json:"name"` | rename the key |
| `json:"-"` | omit the field entirely |
| `json:"-,"` | the key is literally `"-"` (note the trailing comma) |
| `json:"note,omitempty"` | omit when the value is false, 0, "", nil, or an empty slice/map |
| `json:"count,string"` | encode a number or bool as a JSON string |
| `json:"when,omitzero"` | omit when the value is its type's zero value (Go 1.24+) |
| no tag | the Go field name is the key, matched case-insensitively on decode |

`omitempty` and `omitzero` differ exactly where it matters: `omitempty` has no idea what an
empty `time.Time` is (a struct is never "empty"), so `When` would always be emitted;
`omitzero` compares against the zero value and drops it. For a `*T`, `omitempty` already
works, which is why pre-1.24 code models optional fields as pointers.

## Tags are just metadata, read via reflect

```go
package main

import (
  "fmt"
  "reflect"
)

type User struct {
  ID    int    `json:"id" db:"user_id" validate:"required"`
  Email string `json:"email,omitempty" db:"email" validate:"required,email"`
  Name  string `json:"name"`
  temp  string
}

func main() {
  t := reflect.TypeOf(User{})
  fmt.Println(t.Name(), t.NumField())

  for i := 0; i < t.NumField(); i++ {
    f := t.Field(i)
    db, ok := f.Tag.Lookup("db")
    fmt.Printf("%-6s exported=%-5t json=%-18q db=%-9q found=%t validate=%q\n",
      f.Name, f.IsExported(), f.Tag.Get("json"), db, ok, f.Tag.Get("validate"))
  }

  f, _ := t.FieldByName("Email")
  fmt.Println(string(f.Tag))
}
```

Output:

```
User 4
ID     exported=true  json="id"               db="user_id" found=true validate="required"
Email  exported=true  json="email,omitempty"  db="email"   found=true validate="required,email"
Name   exported=true  json="name"             db=""        found=false validate=""
temp   exported=false json=""                 db=""        found=false validate=""
```

```
json:"email,omitempty" db:"email" validate:"required,email"
```

`Get` returns `""` both for a missing key and for a key explicitly set to `""`. `Lookup`
returns a second `bool` that distinguishes them — use it whenever an absent tag and an empty
tag should behave differently. A field can carry any number of keys; different libraries
read different ones and ignore the rest.

## Malformed tags fail silently

This is the gotcha worth the whole lesson.

```go
package main

import (
  "encoding/json"
  "fmt"
  "reflect"
)

type Broken struct {
  A string `json: "a"`
  B string `json "b"`
  C string `Json:"c"`
  D string `json:"d"`
}

func main() {
  b, _ := json.Marshal(Broken{A: "1", B: "2", C: "3", D: "4"})
  fmt.Println(string(b))

  t := reflect.TypeOf(Broken{})
  for i := 0; i < t.NumField(); i++ {
    f := t.Field(i)
    v, ok := f.Tag.Lookup("json")
    fmt.Printf("%s raw=%q json=%q ok=%t\n", f.Name, string(f.Tag), v, ok)
  }
}
```

Output:

```
{"A":"1","B":"2","C":"3","d":"4"}
A raw="json: \"a\"" json="" ok=false
B raw="json \"b\"" json="" ok=false
C raw="Json:\"c\"" json="" ok=false
D raw="json:\"d\"" json="d" ok=true
```

Three different mistakes, three fields silently exported under their Go names. The program
compiles, runs, and produces wrong JSON — exactly the kind of bug that reaches production
and then an API consumer.

`go vet` catches two of the three:

```bash
go vet .
```

Output:

```
main.go:10:2: struct field tag `json: "a"` not compatible with reflect.StructTag.Get: bad syntax for struct tag value
main.go:11:2: struct field tag `json "b"` not compatible with reflect.StructTag.Get: bad syntax for struct tag pair
```

`Json:"c"` is *syntactically* valid — it is a well-formed tag for a key named `Json`, which
nothing reads. Vet has no way to know that was meant to be `json`, so it stays silent. Tag
keys are case-sensitive; mind the lowercase.

The same analyzer also reports two fields claiming one JSON name, as
`struct field B repeats json tag "x" also at main.go:10`. Run `go vet ./...` in CI (15.3) and
the detectable classes never ship.

## Unexported fields are invisible

```go
package main

import (
  "encoding/json"
  "fmt"
)

type Row struct {
  ID     int    `json:"id"`
  Token  string `json:"token"`
  secret string
}

func main() {
  var r Row
  err := json.Unmarshal([]byte(`{"id":1,"token":"t","secret":"s"}`), &r)
  fmt.Println(err)
  fmt.Printf("%+v\n", r)

  out, _ := json.Marshal(Row{ID: 1, Token: "t", secret: "s"})
  fmt.Println(string(out))

  var r2 Row
  json.Unmarshal([]byte(`{"ID":9,"TOKEN":"caseless"}`), &r2)
  fmt.Printf("%+v\n", r2)
}
```

Output:

```
<nil>
{ID:1 Token:t secret:}
{"id":1,"token":"t"}
{ID:9 Token:caseless secret:}
```

`secret` is skipped in both directions, with no error — the incoming `"secret":"s"` is
dropped. Reflection cannot set an unexported field, so adding `json:"secret"` to it changes
nothing. Note also that decoding is case-insensitive: `"ID"` and `"TOKEN"` matched tags
spelled `id` and `token`, which is a deliberate leniency in `encoding/json` and a frequent
surprise.

## Tags on embedded fields

```go
package main

import (
  "encoding/json"
  "fmt"
)

type Meta struct {
  Version int `json:"version"`
}

type Plain struct {
  Meta
  Name string `json:"name"`
}

type Renamed struct {
  Meta `json:"meta"`
  Name string `json:"name"`
}

type Hidden struct {
  Meta `json:"-"`
  Name string `json:"name"`
}

func main() {
  m := Meta{Version: 3}
  for _, v := range []any{Plain{m, "p"}, Renamed{m, "r"}, Hidden{m, "h"}} {
    b, _ := json.Marshal(v)
    fmt.Printf("%T %s\n", v, b)
  }
}
```

Output:

```
main.Plain {"version":3,"name":"p"}
main.Renamed {"meta":{"version":3},"name":"r"}
main.Hidden {"name":"h"}
```

An untagged embedded struct is flattened. Tag it with a name and it nests; tag it `-` and it
disappears. The embedded field's own name (`Meta`) is what the tag applies to, which is why
a tag on an anonymous field is legal at all.

## Writing a tag reader

A complete tag-driven column-list builder — about forty lines, and the core of every Go
micro-ORM.

```go
package main

import (
  "fmt"
  "reflect"
  "strings"
)

func columns(v any) []string {
  t := reflect.TypeOf(v)
  if t.Kind() == reflect.Pointer {
    t = t.Elem()
  }
  var cols []string
  for i := 0; i < t.NumField(); i++ {
    f := t.Field(i)
    if !f.IsExported() {
      continue
    }
    tag, ok := f.Tag.Lookup("db")
    if !ok {
      cols = append(cols, strings.ToLower(f.Name))
      continue
    }
    name, opts, _ := strings.Cut(tag, ",")
    if name == "-" {
      continue
    }
    if name == "" {
      name = strings.ToLower(f.Name)
    }
    if strings.Contains(opts, "readonly") {
      continue
    }
    cols = append(cols, name)
  }
  return cols
}

type Account struct {
  ID        int    `db:"account_id"`
  Email     string `db:"email"`
  CreatedAt string `db:"created_at,readonly"`
  Computed  string `db:"-"`
  Fallback  string
  internal  int
}

func main() {
  cols := columns(Account{})
  fmt.Println(cols)
  fmt.Printf("INSERT INTO accounts (%s) VALUES (...)\n", strings.Join(cols, ", "))
  fmt.Println(columns(&Account{}))
}
```

Output:

```
[account_id email fallback]
INSERT INTO accounts (account_id, email, fallback) VALUES (...)
[account_id email fallback]
```

The shape to copy: handle the pointer case with `t.Elem()`, skip unexported fields with
`IsExported`, split name from options with `strings.Cut`, treat `-` as "skip", and fall back
to a derived name when the tag is absent. Real code caches the result per `reflect.Type`,
because this walk is far too slow to run per row. 13.15 covers `reflect` properly.

## Common tag keys

| Key | Read by | Typical value |
|---|---|---|
| `json` | `encoding/json` | `json:"id,omitempty"` |
| `xml` | `encoding/xml` | `xml:"id,attr"` |
| `yaml` | `gopkg.in/yaml.v3` | `yaml:"id"` |
| `toml` | `github.com/BurntSushi/toml` | `toml:"id"` |
| `db` | `sqlx`, `pgx` row scanners, many micro-ORMs | `db:"user_id"` |
| `gorm` | `gorm.io/gorm` | `gorm:"primaryKey;size:64"` (semicolons, not commas) |
| `validate` | `go-playground/validator` | `validate:"required,email,max=255"` |
| `form` | form decoders, `chi` helpers | `form:"page"` |
| `env` | `caarlos0/env`, `kelseyhightower/envconfig` | `env:"PORT" envDefault:"8080"` |
| `mapstructure` | `spf13/viper` | `mapstructure:"log_level"` |
| `bson` | `go.mongodb.org/mongo-driver` | `bson:"_id,omitempty"` |

A key you invent is as valid as any of these; it just needs code to read it. Note `gorm`'s
semicolon separator — the comma convention is a convention, not a rule.

Forward pointers: 13.6 and 13.7 for `encoding/json` in depth, 13.15 for `reflect`, 16.8 for
request validation with `validate` tags, 18.2 for `env`-tagged configuration.

## Quick Reference

| Syntax | Meaning |
|---|---|
| `` `json:"name"` `` | one key, one value |
| `` `json:"name,omitempty" db:"col"` `` | two keys, space-separated |
| `` `json:",omitempty"` `` | keep the Go field name, add the option |
| `` `json:"-"` `` | skip the field |
| `` `json:"-,"` `` | the key is the literal `-` |

| reflect API | Returns |
|---|---|
| `reflect.TypeOf(v)` | the `reflect.Type` |
| `t.NumField()` | field count (panics unless `t` is a struct) |
| `t.Field(i)` | a `reflect.StructField` |
| `t.FieldByName("X")` | `(StructField, bool)` |
| `f.Tag` | the whole tag as a `reflect.StructTag` |
| `f.Tag.Get("k")` | the value, or `""` if absent or malformed |
| `f.Tag.Lookup("k")` | `(value, found)` — distinguishes absent from empty |
| `f.IsExported()` | whether an encoder can see it |
| `f.Anonymous` | whether the field is embedded |

## Common Mistakes

**Putting a space after the colon: `` `json: "id"` ``.** The tag is silently ignored; run
`go vet` and it tells you exactly this.

**Capitalising the key: `` `JSON:"id"` ``.** Keys are case-sensitive and vet cannot detect
it; the field falls back to its Go name.

**Tagging an unexported field.** Encoders cannot read or write it at all; export the field
or write a custom marshaler.

**Using `omitempty` on a struct or `time.Time` field.** A struct is never "empty"; use
`omitzero` (Go 1.24+) or a `*T`.

**Forgetting that decoding matches names case-insensitively.** `{"ID": 1}` fills a field
tagged `json:"id"`, so a case-only distinction between two keys is not reliable.

## Exercises

1. Define a `Product` struct with `json`, `db`, and `validate` tags on every field. Marshal
   it, then print every tag with `reflect` and confirm each key parses.
2. Introduce all three malformed-tag forms from this lesson into one struct, run `go vet`,
   and record which ones it reports and which it misses.
3. Write `func tagMap(v any) map[string]string` returning Go field name to JSON key,
   resolving `-`, `,omitempty`, and the empty-name case correctly. Test it against a struct
   with an embedded field.
4. Compare `omitempty` and `omitzero` on fields of type `time.Time`, `*int`, `[]string`, and
   `struct{}`. Build a table of which are omitted under each.
5. Extend the `columns` reader to also emit `$1, $2, ...` placeholders and a
   `[]any` of values via `reflect.Value`, producing a complete parameterised INSERT.

---

[← 4.6 Named Types, Type Definitions, and Aliases](06-named-types.md) · [Course Index](../README.md) · [4.8 Constructors and the Functional Options Pattern →](08-constructors-options.md)
