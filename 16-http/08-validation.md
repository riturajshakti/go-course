[← 16.7 chi: Routing, Groups, and Middleware](07-chi.md) · [Course Index](../README.md) · [16.9 HTTP Errors and Problem Responses →](09-error-responses.md)

# 16.8 — Request Validation

## What You'll Learn

- Hand-written validation with a `Validate() error` method returning a field-error map — zero dependencies, fully idiomatic, and often enough
- `go-playground/validator/v10`: the tag vocabulary, `dive`, cross-field rules, custom validators
- `RegisterTagNameFunc` so error messages carry **JSON** field names, not Go field names — commonly missed and badly wrong without it
- Turning `validator.ValidationErrors` into a clean field-keyed 422 body

## Coming From Another Language

| Other | Go |
|---|---|
| Pydantic / FastAPI: validation is the type annotation | struct tags are *data*; a library reads them at runtime via reflection |
| `zod` / `joi`: a schema object separate from the type | tags live on the struct, so the type and its rules cannot drift apart |
| Pydantic reports `loc: ["body","address","zip"]` | you build the path from `FieldError.Namespace()` yourself |
| class-validator decorators | `validate:"required,min=2"` |
| Rust `validator` crate derive | nearly identical design, minus compile-time checking |
| Errors raised as exceptions | returned as `error`; `errors.As` to a `validator.ValidationErrors` |

Nothing in Go validates anything for you. The decoder (16.5) checks *shape*; whether
`age: -4` is acceptable is a question only your domain can answer, and it is always an
explicit step.

## Hand-written validation

Start here, because it needs no dependency, reads like ordinary Go, gives you exactly the
messages you want, and is what the standard library's own style would suggest. The pattern:
a `Validate() error` method returning a type that *is* a field-error map.

```go
package main

import (
  "encoding/json"
  "fmt"
  "net/mail"
  "sort"
  "strings"
)

type FieldErrors map[string]string

func (fe FieldErrors) Error() string {
  keys := make([]string, 0, len(fe))
  for k := range fe {
    keys = append(keys, k)
  }
  sort.Strings(keys)
  parts := make([]string, 0, len(keys))
  for _, k := range keys {
    parts = append(parts, k+": "+fe[k])
  }
  return "validation failed: " + strings.Join(parts, "; ")
}

func (fe FieldErrors) Add(field, msg string) {
  if _, exists := fe[field]; !exists {
    fe[field] = msg
  }
}

type CreateUser struct {
  Name     string   `json:"name"`
  Email    string   `json:"email"`
  Age      int      `json:"age"`
  Role     string   `json:"role"`
  Password string   `json:"password"`
  Confirm  string   `json:"confirm"`
  Tags     []string `json:"tags"`
}

func (u CreateUser) Validate() error {
  fe := FieldErrors{}

  if strings.TrimSpace(u.Name) == "" {
    fe.Add("name", "is required")
  } else if n := len([]rune(u.Name)); n < 2 || n > 50 {
    fe.Add("name", "must be between 2 and 50 characters")
  }

  if u.Email == "" {
    fe.Add("email", "is required")
  } else if _, err := mail.ParseAddress(u.Email); err != nil {
    fe.Add("email", "must be a valid email address")
  }

  if u.Age < 18 || u.Age > 120 {
    fe.Add("age", "must be between 18 and 120")
  }

  switch u.Role {
  case "admin", "editor", "viewer":
  case "":
    fe.Add("role", "is required")
  default:
    fe.Add("role", "must be one of: admin, editor, viewer")
  }

  if len(u.Password) < 10 {
    fe.Add("password", "must be at least 10 characters")
  }
  if u.Password != u.Confirm {
    fe.Add("confirm", "must match password")
  }

  for i, t := range u.Tags {
    if strings.TrimSpace(t) == "" {
      fe.Add(fmt.Sprintf("tags[%d]", i), "must not be empty")
    }
  }

  if len(fe) > 0 {
    return fe
  }
  return nil
}

func main() {
  cases := []CreateUser{
    {Name: "Ada Lovelace", Email: "ada@example.com", Age: 36, Role: "admin",
      Password: "correct-horse", Confirm: "correct-horse", Tags: []string{"math"}},
    {},
    {Name: "A", Email: "not-an-email", Age: 7, Role: "wizard",
      Password: "short", Confirm: "mismatch", Tags: []string{"ok", "  "}},
  }
  for i, c := range cases {
    err := c.Validate()
    fmt.Printf("=== case %d\n", i)
    if err == nil {
      fmt.Println("valid")
      continue
    }
    fmt.Println("Error():", err)
    var fe FieldErrors
    if ok := asFieldErrors(err, &fe); ok {
      out, _ := json.MarshalIndent(map[string]any{
        "error":  "validation failed",
        "fields": fe,
      }, "", "  ")
      fmt.Println(string(out))
    }
  }
}

func asFieldErrors(err error, dst *FieldErrors) bool {
  fe, ok := err.(FieldErrors)
  if ok {
    *dst = fe
  }
  return ok
}
```

Output:

```
=== case 0
valid
=== case 1
Error(): validation failed: age: must be between 18 and 120; email: is required; name: is required; password: must be at least 10 characters; role: is required
{
  "error": "validation failed",
  "fields": {
    "age": "must be between 18 and 120",
    "email": "is required",
    "name": "is required",
    "password": "must be at least 10 characters",
    "role": "is required"
  }
}
=== case 2
Error(): validation failed: age: must be between 18 and 120; confirm: must match password; email: must be a valid email address; name: must be between 2 and 50 characters; password: must be at least 10 characters; role: must be one of: admin, editor, viewer; tags[1]: must not be empty
{
  "error": "validation failed",
  "fields": {
    "age": "must be between 18 and 120",
    "confirm": "must match password",
    "email": "must be a valid email address",
    "name": "must be between 2 and 50 characters",
    "password": "must be at least 10 characters",
    "role": "must be one of: admin, editor, viewer",
    "tags[1]": "must not be empty"
  }
}
```

What this gets right, and why it is not a fallback option:

**`FieldErrors` is a `map[string]string` that implements `error`.** It satisfies `error` via
a method on a named map type ([4.6](../04-types-structs/06-named-types.md)) and marshals
straight to a JSON object with no conversion step.

**It collects *all* failures.** Returning on the first one makes users fix a form field at a
time. `Add` keeps the first message per field so the earlier, more specific check wins.

**The keys are JSON names.** `"age"`, not `"Age"`. The client sees the names it sent.

**`len([]rune(u.Name))` not `len(u.Name)`.** Byte length is wrong for a length rule —
`"Zoë"` is 4 bytes and 3 characters ([6.2](../06-strings/02-utf8-iteration.md)).

**`net/mail.ParseAddress` instead of a regexp.** Email regexps are always wrong; the
standard library has a real RFC 5322 parser. (It accepts `"Ada <a@b>"` form too, so trim or
reject display names if you need a bare address.)

**Cross-field rules are trivial.** `u.Password != u.Confirm` is just an `if` — no special
mechanism needed, which is where tag-based libraries have to get clever.

The cost: about 45 lines per request type, and it grows. That is the trade-off the next
section buys out of.

## go-playground/validator/v10

```bash
go get github.com/go-playground/validator/v10@v10.30.5
```

Real output (the module and its transitive dependencies):

```
go: added github.com/gabriel-vasile/mimetype v1.4.15
go: added github.com/go-playground/locales v0.14.1
go: added github.com/go-playground/universal-translator v0.18.1
go: added github.com/go-playground/validator/v10 v10.30.5
go: added github.com/leodido/go-urn v1.5.0
go: added golang.org/x/crypto v0.57.0
go: added golang.org/x/sys v0.48.0
go: added golang.org/x/text v0.42.0
```

```bash
go list -m github.com/go-playground/validator/v10
```

```
github.com/go-playground/validator/v10 v10.30.5
```

Everything below ran against **v10.30.5** on Go 1.27.1. Note it pulls in seven
dependencies — unlike chi, this is not a zero-dependency choice, which is a legitimate
reason some teams stay with the hand-written approach.

### The tag vocabulary, and FieldError's details

One struct exercising the tags worth knowing, then every detail `FieldError` exposes.

```go
package main

import (
  "fmt"
  "reflect"
  "strings"

  "github.com/go-playground/validator/v10"
)

type Address struct {
  Street string `json:"street" validate:"required"`
  Zip    string `json:"zip"    validate:"required,len=5,numeric"`
}

type SignUp struct {
  Name     string            `json:"name"      validate:"required,min=2,max=50"`
  Email    string            `json:"email"     validate:"required,email"`
  Age      int               `json:"age"       validate:"required,gte=18,lte=120"`
  Role     string            `json:"role"      validate:"required,oneof=admin editor viewer"`
  Website  string            `json:"website"   validate:"omitempty,url"`
  Phone    string            `json:"phone"     validate:"omitempty,e164"`
  UserID   string            `json:"user_id"   validate:"omitempty,uuid"`
  Handle   string            `json:"handle"    validate:"required,alphanum,min=3"`
  StartsAt string            `json:"starts_at" validate:"omitempty,datetime=2006-01-02"`
  Tags     []string          `json:"tags"      validate:"omitempty,max=3,dive,required,min=2"`
  Meta     map[string]string `json:"meta"      validate:"omitempty,dive,keys,alphanum,endkeys,required"`
  Address  Address           `json:"address"   validate:"required"`
  Coupon   string            `json:"coupon"    validate:"omitempty,coupon"`
  Password string            `json:"password"  validate:"required,min=10"`
  Confirm  string            `json:"confirm"   validate:"required"`
  Referrer string            `json:"referrer"  validate:"required_if=Role admin"`
  Nickname string            `json:"nickname"  validate:"excluded_with=Handle"`
}

func validCoupon(fl validator.FieldLevel) bool {
  return strings.HasPrefix(fl.Field().String(), "GO-")
}

func signUpStructLevel(sl validator.StructLevel) {
  s := sl.Current().Interface().(SignUp)
  if s.Password != "" && s.Password != s.Confirm {
    sl.ReportError(s.Confirm, "Confirm", "Confirm", "eqfield", "Password")
  }
}

func newValidator() *validator.Validate {
  v := validator.New(validator.WithRequiredStructEnabled())
  v.RegisterTagNameFunc(func(fld reflect.StructField) string {
    name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
    if name == "-" || name == "" {
      return fld.Name
    }
    return name
  })
  v.RegisterValidation("coupon", validCoupon)
  v.RegisterStructValidation(signUpStructLevel, SignUp{})
  return v
}

func main() {
  v := newValidator()

  good := SignUp{
    Name: "Ada", Email: "ada@example.com", Age: 36, Role: "editor",
    Handle: "ada99", Password: "correct-horse", Confirm: "correct-horse",
    Address: Address{Street: "1 Main", Zip: "12345"},
    Tags:    []string{"math", "cs"},
  }
  fmt.Printf("=== valid struct -> err=%v\n", v.Struct(good))

  bad := SignUp{
    Name: "A", Email: "nope", Age: 7, Role: "wizard",
    Website: "htp:/bad", Phone: "12345", UserID: "not-a-uuid",
    Handle: "a!", StartsAt: "01/02/2024",
    Tags:   []string{"ok", "x", "yy", "zz"},
    Meta:   map[string]string{"bad key!": ""},
    Coupon: "XX-1", Password: "short", Confirm: "different",
    Nickname: "nick",
  }
  err := v.Struct(bad)
  fmt.Printf("\n=== invalid struct\nerr type: %T\n", err)

  var ve validator.ValidationErrors
  if ok := asVE(err, &ve); ok {
    fmt.Printf("%d field errors\n\n", len(ve))
    fmt.Printf("%-16s %-14s %-24s %-10s %-22s %v\n",
      "Field()", "Tag()", "Namespace()", "Param()", "StructNamespace()", "Value()")
    for _, fe := range ve {
      fmt.Printf("%-16s %-14s %-24s %-10s %-22s %v\n",
        fe.Field(), fe.Tag(), fe.Namespace(), fe.Param(), fe.StructNamespace(), fe.Value())
    }
  }
}

func asVE(err error, dst *validator.ValidationErrors) bool {
  ve, ok := err.(validator.ValidationErrors)
  if ok {
    *dst = ve
  }
  return ok
}
```

Output:

```
=== valid struct -> err=<nil>

=== invalid struct
err type: validator.ValidationErrors
17 field errors

Field()          Tag()          Namespace()              Param()    StructNamespace()      Value()
name             min            SignUp.name              2          SignUp.Name            A
email            email          SignUp.email                        SignUp.Email           nope
age              gte            SignUp.age               18         SignUp.Age             7
role             oneof          SignUp.role              admin editor viewer SignUp.Role            wizard
website          url            SignUp.website                      SignUp.Website         htp:/bad
phone            e164           SignUp.phone                        SignUp.Phone           12345
user_id          uuid           SignUp.user_id                      SignUp.UserID          not-a-uuid
handle           alphanum       SignUp.handle                       SignUp.Handle          a!
starts_at        datetime       SignUp.starts_at         2006-01-02 SignUp.StartsAt        01/02/2024
tags             max            SignUp.tags              3          SignUp.Tags            [ok x yy zz]
meta[bad key!]   alphanum       SignUp.meta[bad key!]               SignUp.Meta[bad key!]  bad key!
meta[bad key!]   required       SignUp.meta[bad key!]               SignUp.Meta[bad key!]  
address          required       SignUp.address                      SignUp.Address         { }
coupon           coupon         SignUp.coupon                       SignUp.Coupon          XX-1
password         min            SignUp.password          10         SignUp.Password        short
nickname         excluded_with  SignUp.nickname          Handle     SignUp.Nickname        nick
Confirm          eqfield        SignUp.Confirm           Password   SignUp.Confirm         different
```

Everything you need to build a good error body is in those columns.

| `FieldError` method | Gives | Example above |
|---|---|---|
| `Field()` | the field name, as `RegisterTagNameFunc` renders it | `user_id` |
| `StructField()` | the Go field name | `UserID` |
| `Tag()` | which rule failed | `uuid` |
| `Param()` | the rule's argument | `18`, `2006-01-02`, `admin editor viewer` |
| `Value()` | the value that failed | `not-a-uuid` |
| `Namespace()` | the full path with tag names | `SignUp.address.street` for nested |
| `StructNamespace()` | the full path with Go names | `SignUp.Address.Street` |
| `Kind()` | the `reflect.Kind` — lets one message say "characters" vs "items" | `reflect.String` |
| `Type()` | the `reflect.Type` | `string` |
| `ActualTag()` | the tag as matched, for `omitempty` chains | |

Three things in that output to understand:

**`RegisterTagNameFunc` is doing the single most important job.** Without it `Field()`
returns `UserID` and `Namespace()` returns `SignUp.UserID`, so your 422 body names Go
identifiers the client has never seen. With it you get `user_id` — the exact key they sent.
It is four lines, it is easy to forget, and forgetting it produces an API that is confusing
in a way nobody reports as a bug.

**`meta[bad key!]` produced two errors.** `dive,keys,alphanum,endkeys,required` validates the
map's keys (`alphanum`) and then its values (`required`), and both failed for that entry.
The namespace includes the offending key, so you can point at it precisely.

**Only one `tags` error appeared**, for `max=3`. validator stops at the first failing rule
*per field*, so `dive,min=2` never ran on `"x"`. Fix the length and re-submit, and then you
see the element error — which is worth telling clients in your docs.

**`Value()` is in the error.** Never echo it back blindly: for `password` that would put the
rejected password in your response body and, worse, your logs.

The tags, grouped:

| Group | Tags |
|---|---|
| presence | `required`, `omitempty`, `omitnil`, `isdefault` |
| conditional presence | `required_if`, `required_unless`, `required_with`, `required_with_all`, `required_without`, `required_without_all` |
| exclusion | `excluded_if`, `excluded_unless`, `excluded_with`, `excluded_without` |
| size (string length / number value / slice len) | `len`, `min`, `max`, `gt`, `gte`, `lt`, `lte`, `eq`, `ne` |
| cross-field | `eqfield`, `nefield`, `gtfield`, `ltefield`, `eqcsfield` (cross-struct) |
| set membership | `oneof`, `oneofci` |
| strings | `alpha`, `alphanum`, `alphanumunicode`, `ascii`, `lowercase`, `uppercase`, `contains`, `startswith`, `endswith` |
| formats | `email`, `url`, `uri`, `uuid`, `uuid4`, `json`, `jwt`, `base64`, `hexcolor`, `e164`, `ip`, `ipv4`, `cidr`, `mac`, `hostname`, `fqdn`, `semver`, `countrycode` |
| numbers | `numeric`, `number`, `latitude`, `longitude` |
| time | `datetime=<layout>`, `timezone` |
| containers | `dive`, `keys`/`endkeys`, `unique` |
| database-ish | `postgres`, `mongodb`, `ulid` |

`validator.New(validator.WithRequiredStructEnabled())` is the option you want in v10.
Without it, `required` on a *nested struct field* does nothing — the `address required`
line above only appears because the option is set.

### Custom validators and struct-level rules

Two extension points, both in the example above:

**`RegisterValidation(tag, fn)`** adds a tag. The function takes a
`validator.FieldLevel`, from which `Field()` gives a `reflect.Value`. `validCoupon` is the
whole shape:

```go
func validCoupon(fl validator.FieldLevel) bool {
  return strings.HasPrefix(fl.Field().String(), "GO-")
}
v.RegisterValidation("coupon", validCoupon)
```

Use it for genuinely reusable, single-field rules — a product SKU format, an internal ID
scheme. Do **not** use it for anything that touches a database or network: validation runs
synchronously under reflection, per field, with no context.

**`RegisterStructValidation(fn, Type{})`** adds rules that need several fields at once:

```go
func signUpStructLevel(sl validator.StructLevel) {
  s := sl.Current().Interface().(SignUp)
  if s.Password != "" && s.Password != s.Confirm {
    sl.ReportError(s.Confirm, "confirm", "Confirm", "eqfield", "password")
  }
}
```

`ReportError(field, fieldName, structFieldName, tag, param)` synthesizes a `FieldError` that
flows through your message mapper like any other. Note from the captured table that the
tag-name function does **not** apply here — you pass the name you want as the second
argument, so pass the JSON name (`"confirm"`) directly.

The `eqfield` tag could have done password confirmation on its own
(`Confirm string \`validate:"eqfield=Password"\``). Struct-level validation earns its place
when the rule is not expressible as one field compared to one other: "at least one of email
or phone", "end date after start date only when recurring is true".

### A handler doing decode → validate → respond

The full pipeline, with a message translator that turns tags into sentences, and nested and
conditional rules exercised for real.

```go
func message(fe validator.FieldError) string {
  switch fe.Tag() {
  case "required":
    return "is required"
  case "required_if":
    return "is required when " + strings.ToLower(strings.ReplaceAll(fe.Param(), " ", " is "))
  case "email":
    return "must be a valid email address"
  case "url":
    return "must be a valid URL"
  case "numeric":
    return "must contain only digits"
  case "datetime":
    return "must be a date formatted as " + fe.Param()
  case "oneof":
    return "must be one of: " + strings.Join(strings.Fields(fe.Param()), ", ")
  case "min":
    return "must be at least " + fe.Param() + unit(fe)
  case "max":
    return "must be at most " + fe.Param() + unit(fe)
  case "gte":
    return "must be " + fe.Param() + " or greater"
  case "lte":
    return "must be " + fe.Param() + " or less"
  case "eqfield":
    return "must match " + fe.Param()
  case "coupon":
    return "must be a coupon code starting with GO-"
  }
  return "is invalid"
}

func unit(fe validator.FieldError) string {
  switch fe.Kind() {
  case reflect.String:
    return " characters"
  case reflect.Slice, reflect.Array, reflect.Map:
    return " items"
  }
  return ""
}

func fieldErrors(err error) map[string]string {
  var ve validator.ValidationErrors
  if !errors.As(err, &ve) {
    return nil
  }
  out := make(map[string]string, len(ve))
  for _, fe := range ve {
    key := strings.SplitN(fe.Namespace(), ".", 2)[1]
    if _, dup := out[key]; !dup {
      out[key] = message(fe)
    }
  }
  return out
}

func handler(w http.ResponseWriter, r *http.Request) {
  var in SignUp
  r.Body = http.MaxBytesReader(w, r.Body, 1<<16)
  dec := json.NewDecoder(r.Body)
  dec.DisallowUnknownFields()
  if err := dec.Decode(&in); err != nil {
    writeJSON(w, 400, map[string]any{"error": "invalid JSON body"})
    return
  }
  if err := v.Struct(in); err != nil {
    writeJSON(w, 422, map[string]any{
      "error":  "validation failed",
      "fields": fieldErrors(err),
    })
    return
  }
  writeJSON(w, 201, map[string]any{"created": in.Email})
}
```

Driven through `httptest` with four bodies (field maps printed key-sorted, since Go map
iteration order is random):

Output:

```
=== valid -> 201 Created
    {"created":"ada@example.com"}
=== everything wrong -> 422 Unprocessable Entity
    address.street         is required
    address.zip            must contain only digits
    age                    must be 18 or greater
    confirm                must match password
    coupon                 must be a coupon code starting with GO-
    email                  must be a valid email address
    name                   must be at least 2 characters
    password               must be at least 10 characters
    role                   must be one of: admin, editor, viewer
    tags                   must be at most 3 items
    website                must be a valid URL
=== admin needs referrer -> 422 Unprocessable Entity
    referrer               is required when role is admin
=== empty object -> 422 Unprocessable Entity
    address                is required
    age                    is required
    confirm                is required
    email                  is required
    name                   is required
    password               is required
    role                   is required
```

This is the output a client can act on. Four details:

**`address.street` and `address.zip` are dotted paths**, from
`strings.SplitN(fe.Namespace(), ".", 2)[1]` — dropping the leading struct name leaves a path
that mirrors the JSON the client sent. Nested validation came free from
`WithRequiredStructEnabled` plus tags on `Address`.

**`required_if=Role admin` fired only for the admin body.** The "admin needs referrer" case
is otherwise identical to the valid one. Note the rule's param uses the **Go** field name
(`Role`), not the JSON name, even with `RegisterTagNameFunc` installed — a real
inconsistency to remember when writing conditional tags.

**The empty object reported seven `required` errors and stopped.** validator short-circuits
per field, so `email` reports `required` rather than also `email`. That is the behaviour you
want: one message per field.

**`unit(fe)` uses `fe.Kind()`** so `min=2` reads "at least 2 characters" for a string and
"at most 3 items" for a slice. One mapper, correct for both.

### Validating query and path params too

Validation is not only for bodies. Build a struct from the query and run the same validator:

```go
type ListQuery struct {
  Page  int    `validate:"gte=1"`
  Limit int    `validate:"gte=1,lte=100"`
  Sort  string `validate:"omitempty,oneof=name created"`
}
```

Output:

```
{Page:1 Limit:20 Sort:name} -> map[]
{Page:0 Limit:500 Sort:bogus} -> map[Limit:must be 100 or less Page:must be 1 or greater Sort:must be one of: name, created]
```

Parse with `strconv` first (16.4) — a non-numeric `limit` is a 400 for *unparseable* input,
distinct from the 422 for an out-of-range one. For single values there is also
`v.Var(value, "gte=1,lte=100")`, which skips the struct entirely.

Path parameters deserve the same treatment: `chi.URLParam(r, "id")` or `r.PathValue("id")`
is attacker-controlled and can contain a slash (16.2).

### The three-state optional field

A `PATCH` endpoint has to distinguish "field omitted", "field set to null", and "field set
to a value". `omitempty` cannot, because it collapses the zero value into "absent":

```go
type PatchItem struct {
  Name  *string  `json:"name"  validate:"omitnil,min=2"`
  Price *float64 `json:"price" validate:"omitnil,gte=0"`
}
```

With a pointer, `nil` means "not provided" and a non-nil pointer to the zero value means
"explicitly set to zero". `omitnil` (v10) skips the remaining rules when the pointer is nil;
plain `omitempty` would also skip a non-nil pointer to `""`, which is not the same thing.
See [13.6](../13-stdlib/06-json.md) for the JSON side and 16.5 for the response side.

## Where validation belongs in the layering

Validation of *request shape and syntax* is a **transport** concern and belongs in the
handler package, next to the request struct it validates. The request struct is part of your
HTTP API, so its tags — `required`, `email`, `max=50` — describe the API contract, and
changing them is an API change. Your domain layer must not import a validation library or
know about `json` tags; it enforces *invariants* in its constructors
(`NewOrder` returning an error for a negative total), which is a different and
non-negotiable check that survives even if the HTTP layer is replaced by a gRPC one or a CLI.
The practical rule: the transport layer rejects what is *malformed* with a 400/422 and clean
field messages; the domain layer rejects what is *impossible* with a typed error (16.9), and
you keep both because the first is for the user's benefit and the second is for the code's.

## Hand-written vs validator

| | Hand-written | validator/v10 |
|---|---|---|
| Dependencies | none | 7 modules |
| Lines per request type | ~40 | ~1 tag line per field |
| Message control | total | needs a mapper you write |
| Discoverability | plain Go, steppable in a debugger | reflection; typos in tags **panic at runtime** |
| Cross-field rules | an `if` | `eqfield` or `RegisterStructValidation` |
| Nested structs / slices | explicit loops | `dive` |
| Compile-time safety | full | none — a renamed field silently breaks `required_if=Role` |
| Performance | trivial | reflection, but cached; irrelevant at HTTP scale |

Idiomatic: hand-written for a handful of endpoints or when messages matter a lot;
validator/v10 once you have a dozen request types and the boilerplate dominates. Both are
normal in production Go. What is *not* normal is mixing them per-endpoint.

A real hazard with tags: a misspelled tag (`requird`) panics at the first `v.Struct` call
with `Undefined validation function`, which means a route nobody tested can crash in
production. Mitigate it by calling `v.Struct` on a zero value of every request type in a
startup check or a test.

## Quick Reference

| Call | Purpose |
|---|---|
| `validator.New(validator.WithRequiredStructEnabled())` | build a validator; **always** pass the option |
| `v.Struct(s) error` | validate; returns `validator.ValidationErrors` |
| `v.StructPartial(s, "A","B")` / `StructExcept` | validate a subset |
| `v.Var(val, "gte=1,lte=100")` | validate one value, no struct |
| `v.VarWithValue(a, b, "eqfield")` | compare two values |
| `v.RegisterValidation(tag, fn)` | add a single-field tag |
| `v.RegisterStructValidation(fn, Type{})` | add cross-field rules |
| `v.RegisterTagNameFunc(fn)` | **use JSON names in errors** |
| `v.RegisterAlias(alias, tags)` | name a common tag combination |
| `v.StructCtx(ctx, s)` | pass a context to custom validators |

| Error handling | Note |
|---|---|
| `errors.As(err, &validator.ValidationErrors{})` | the normal case |
| `*validator.InvalidValidationError` | you passed a non-struct — **your bug**, panic |
| `fe.Namespace()` | `Struct.field.subfield`; drop the first segment for a client path |
| one error per field | validator stops at the first failing rule per field |

## Common Mistakes

**Forgetting `RegisterTagNameFunc`.** Your 422 body names Go fields (`UserID`) that the
client never sent; four lines fix every message at once.

**Omitting `WithRequiredStructEnabled()`.** `required` on a nested struct field silently does
nothing, so an entirely missing `address` passes validation.

**Returning on the first validation error.** Users fix one field per round trip; collect them
all and return a field map.

**Using `omitempty` for an optional PATCH field.** It cannot distinguish "absent" from
"set to zero", so a deliberate `0` or `""` is ignored; use a pointer with `omitnil`.

**Echoing `fe.Value()` into the response or the log.** For a password or a token that leaks
the secret; name the field and the rule, never the value.

**Misspelling a tag.** It panics at the first use with `Undefined validation function`, so an
untested route crashes in production; validate a zero value of every request type at startup.

**Using byte length for a character limit.** `len(s)` counts bytes, so `max=50` rejects a
50-character name containing any non-ASCII; validator's `max` on a string counts runes, but
your hand-written check must use `len([]rune(s))`.

**Validating in the domain layer with HTTP tags.** It couples your business logic to your
transport; keep request structs and their tags in the handler package.

## Exercises

1. Write hand-written `Validate()` for a `CreateOrder` with a line-item slice, requiring at
   least one item, a positive quantity per item, and a currency from a fixed set. Return
   indexed keys like `items[2].quantity` and verify the full field map for a body with three
   different failures.
2. Convert the same struct to validator tags with `dive`, and show that both produce the same
   field keys. Then delete `RegisterTagNameFunc` and show exactly how the output changes.
3. Register a custom `notprofane` validator backed by a word list, plus a struct-level rule
   requiring `end_date` after `start_date` only when `recurring` is true. Prove all four
   combinations of `recurring` and date ordering.
4. Write a startup check that calls `v.Struct` on a zero value of every request type in your
   package and fails loudly. Introduce a typo'd tag and capture the real panic message.
5. Build a `ParseAndValidate[T any](w, r) (T, error)` generic helper
   ([9.1](../09-generics/01-type-parameters.md)) that decodes with the 16.5 classifier,
   validates, and returns one error type carrying a status plus a field map. Use it for three
   different request types and show the real 400 and 422 bodies.

---

[← 16.7 chi: Routing, Groups, and Middleware](07-chi.md) · [Course Index](../README.md) · [16.9 HTTP Errors and Problem Responses →](09-error-responses.md)
