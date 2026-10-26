[← 16.4 Request Parsing: Path, Query, Headers, Body](04-request-parsing.md) · [Course Index](../README.md) · [16.6 Middleware from Scratch →](06-middleware.md)

# 16.5 — JSON Requests and Responses

## What You'll Learn

- One reusable `decodeJSON` helper that classifies **every** decode failure into a clean client message and the right status — the most valuable 40 lines in this module
- Why `json.Marshal` + `Write` beats `json.NewEncoder(w).Encode` for an API: an encode error after `WriteHeader` is **unfixable**, and you can prove it
- `DisallowUnknownFields`, `MaxBytesReader`, and `Content-Type` validation as the three gates every JSON endpoint needs
- A complete CRUD API over an in-memory store, exercised end to end

## Coming From Another Language

| Other | Go |
|---|---|
| `express.json()` middleware parses for you | you decode explicitly, per handler, into a typed struct |
| `JSON.parse` throws one `SyntaxError` | **five distinct** error types, each needing a different message |
| FastAPI/Pydantic returns a field-level 422 for free | you build that yourself (here, and 16.8) |
| `res.json(obj)` | `json.Marshal` then `w.Write`, or a `writeJSON` helper |
| Extra JSON keys are silently ignored | ignored by default; `DisallowUnknownFields()` makes them a 400 |
| `req.body` is `{}` for an empty body | an empty body is `io.EOF` from the decoder |

## Decoder vs ReadAll+Unmarshal

Two ways to read a JSON body:

```go
var in CreateItem
err := json.NewDecoder(r.Body).Decode(&in)
```

```go
body, err := io.ReadAll(r.Body)
if err == nil {
  err = json.Unmarshal(body, &in)
}
```

The decoder is preferred for a request body: it **streams**, so it never holds the whole
body plus the parsed value in memory at once, it needs no intermediate `[]byte`, and it is
one call instead of two error checks. `ReadAll`+`Unmarshal` earns its place in exactly two
cases: when you need the raw bytes for a signature check (webhooks) or for logging, and
when you want to decode the same body twice into different types.

Neither one bounds the body. That is on you.

## Every way a decode can fail

Before writing the helper, here is the full taxonomy, captured for real. The handler prints
the error, its dynamic type, and the result of every classification check.

```go
package main

import (
  "encoding/json"
  "errors"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

type CreateItem struct {
  Name  string  `json:"name"`
  Price float64 `json:"price"`
  Qty   int     `json:"qty"`
}

func rawDecode(w http.ResponseWriter, r *http.Request) {
  var in CreateItem
  dec := json.NewDecoder(r.Body)
  dec.DisallowUnknownFields()
  err := dec.Decode(&in)
  if err == nil {
    fmt.Fprintf(w, "OK %+v\n", in)
    return
  }
  var se *json.SyntaxError
  var ute *json.UnmarshalTypeError
  var mbe *http.MaxBytesError
  fmt.Fprintf(w, "err      = %v\n", err)
  fmt.Fprintf(w, "type     = %T\n", err)
  fmt.Fprintf(w, "SyntaxError      = %v", errors.As(err, &se))
  if errors.As(err, &se) {
    fmt.Fprintf(w, " offset=%d", se.Offset)
  }
  fmt.Fprintln(w)
  fmt.Fprintf(w, "UnmarshalTypeErr = %v", errors.As(err, &ute))
  if errors.As(err, &ute) {
    fmt.Fprintf(w, " field=%q value=%q type=%v offset=%d", ute.Field, ute.Value, ute.Type, ute.Offset)
  }
  fmt.Fprintln(w)
  fmt.Fprintf(w, "io.EOF           = %v\n", errors.Is(err, io.EOF))
  fmt.Fprintf(w, "io.ErrUnexpectedEOF = %v\n", errors.Is(err, io.ErrUnexpectedEOF))
  fmt.Fprintf(w, "MaxBytesError    = %v\n", errors.As(err, &mbe))
}

func main() {
  srv := httptest.NewServer(http.MaxBytesHandler(http.HandlerFunc(rawDecode), 80))
  defer srv.Close()

  cases := []struct{ name, body string }{
    {"valid", `{"name":"widget","price":9.99,"qty":3}`},
    {"syntax error", `{"name":"widget",}`},
    {"wrong type", `{"name":"widget","qty":"three"}`},
    {"wrong type top", `["not","an","object"]`},
    {"unknown field", `{"name":"w","colour":"red"}`},
    {"empty body", ``},
    {"truncated", `{"name":"wid`},
    {"too large", `{"name":"` + strings.Repeat("A", 200) + `"}`},
    {"trailing data", `{"name":"a"}{"name":"b"}`},
    {"null", `null`},
  }
  for _, c := range cases {
    resp, err := http.Post(srv.URL, "application/json", strings.NewReader(c.body))
    if err != nil {
      fmt.Printf("=== %s: client err %v\n", c.name, err)
      continue
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("=== %s\n%s\n", c.name, b)
  }
}
```

Output:

```
=== valid
OK {Name:widget Price:9.99 Qty:3}

=== syntax error
err      = invalid character '}' looking for beginning of object key string
type     = *json.SyntaxError
SyntaxError      = true offset=18
UnmarshalTypeErr = false
io.EOF           = false
io.ErrUnexpectedEOF = false
MaxBytesError    = false

=== wrong type
err      = json: cannot unmarshal string into Go struct field CreateItem.qty of type int
type     = *json.UnmarshalTypeError
SyntaxError      = false
UnmarshalTypeErr = true field="qty" value="string" type=int offset=30
io.EOF           = false
io.ErrUnexpectedEOF = false
MaxBytesError    = false

=== wrong type top
err      = json: cannot unmarshal array into Go value of type main.CreateItem
type     = *json.UnmarshalTypeError
SyntaxError      = false
UnmarshalTypeErr = true field="" value="array" type=main.CreateItem offset=1
io.EOF           = false
io.ErrUnexpectedEOF = false
MaxBytesError    = false

=== unknown field
err      = json: unknown field "colour"
type     = *errors.errorString
SyntaxError      = false
UnmarshalTypeErr = false
io.EOF           = false
io.ErrUnexpectedEOF = false
MaxBytesError    = false

=== empty body
err      = EOF
type     = *errors.errorString
SyntaxError      = false
UnmarshalTypeErr = false
io.EOF           = true
io.ErrUnexpectedEOF = false
MaxBytesError    = false

=== truncated
err      = unexpected EOF
type     = *errors.errorString
SyntaxError      = false
UnmarshalTypeErr = false
io.EOF           = false
io.ErrUnexpectedEOF = true
MaxBytesError    = false

=== too large
err      = http: request body too large
type     = *http.MaxBytesError
SyntaxError      = false
UnmarshalTypeErr = false
io.EOF           = false
io.ErrUnexpectedEOF = false
MaxBytesError    = true

=== trailing data
OK {Name:a Price:0 Qty:0}

=== null
OK {Name: Price:0 Qty:0}
```

The full taxonomy, with the detail that matters for each:

| Input | Error | Recognize with | Useful detail |
|---|---|---|---|
| `{"a":1,}` | `*json.SyntaxError` | `errors.As` | `.Offset` — character position |
| `{"qty":"three"}` | `*json.UnmarshalTypeError` | `errors.As` | `.Field`, `.Type`, `.Value`, `.Offset` |
| `["a","b"]` into a struct | `*json.UnmarshalTypeError` | `errors.As` | `.Field` is **empty** — the *whole body* is wrong |
| `{"colour":"red"}` | plain `*errors.errorString` | string prefix match | the field name is only in the message |
| `` (empty) | `io.EOF` | `errors.Is(err, io.EOF)` | — |
| `{"name":"wid` | `io.ErrUnexpectedEOF` | `errors.Is` | — |
| 200 bytes, limit 80 | `*http.MaxBytesError` | `errors.As` | `.Limit` |

Three traps hide in the last three rows of the output.

**`DisallowUnknownFields` gives an untyped error.** There is no `json.UnknownFieldError`, so
the only way to extract `"colour"` is to match the `"json: unknown field "` prefix. Ugly, but
it is a documented, stable message, and the alternative is a useless generic 400.

**`{"name":"a"}{"name":"b"}` decoded successfully.** `Decode` reads exactly **one** JSON
value and stops. Trailing garbage is not an error unless you check for it — a second
`Decode` must return `io.EOF`.

**`null` decoded successfully too**, leaving the struct at its zero value. A JSON `null`
body is a valid JSON value, so `Decode` is happy and you get `{Name: Price:0 Qty:0}` with
no error at all. Your validation layer catches it (16.8), not your decoder.

## One decode helper that handles all of it

Putting the taxonomy to work. This helper validates `Content-Type`, bounds the body,
rejects unknown fields, rejects trailing data, and returns a `*BadRequest` carrying both a
status and a message that is safe to show a client.

```go
package main

import (
  "encoding/json"
  "errors"
  "fmt"
  "io"
  "mime"
  "net/http"
  "net/http/httptest"
  "strings"
)

type BadRequest struct {
  Status  int
  Message string
}

func (e *BadRequest) Error() string { return e.Message }

func decodeJSON(w http.ResponseWriter, r *http.Request, dst any, maxBytes int64) error {
  ct := r.Header.Get("Content-Type")
  if ct != "" {
    mt, _, err := mime.ParseMediaType(ct)
    if err != nil {
      return &BadRequest{http.StatusUnsupportedMediaType, "malformed Content-Type header"}
    }
    if mt != "application/json" {
      return &BadRequest{http.StatusUnsupportedMediaType,
        fmt.Sprintf("Content-Type must be application/json, got %q", mt)}
    }
  }

  r.Body = http.MaxBytesReader(w, r.Body, maxBytes)
  dec := json.NewDecoder(r.Body)
  dec.DisallowUnknownFields()

  if err := dec.Decode(dst); err != nil {
    var syntaxErr *json.SyntaxError
    var typeErr *json.UnmarshalTypeError
    var maxErr *http.MaxBytesError
    var invalidErr *json.InvalidUnmarshalError

    switch {
    case errors.As(err, &syntaxErr):
      return &BadRequest{http.StatusBadRequest,
        fmt.Sprintf("body contains malformed JSON at character %d", syntaxErr.Offset)}

    case errors.Is(err, io.ErrUnexpectedEOF):
      return &BadRequest{http.StatusBadRequest, "body contains truncated JSON"}

    case errors.As(err, &typeErr):
      if typeErr.Field != "" {
        return &BadRequest{http.StatusBadRequest,
          fmt.Sprintf("field %q must be of type %s, got %s",
            typeErr.Field, typeErr.Type, typeErr.Value)}
      }
      return &BadRequest{http.StatusBadRequest,
        fmt.Sprintf("body must be a JSON object, got %s", typeErr.Value)}

    case errors.Is(err, io.EOF):
      return &BadRequest{http.StatusBadRequest, "body must not be empty"}

    case strings.HasPrefix(err.Error(), "json: unknown field "):
      name := strings.TrimPrefix(err.Error(), "json: unknown field ")
      return &BadRequest{http.StatusBadRequest,
        fmt.Sprintf("body contains unknown field %s", name)}

    case errors.As(err, &maxErr):
      return &BadRequest{http.StatusRequestEntityTooLarge,
        fmt.Sprintf("body must not exceed %d bytes", maxErr.Limit)}

    case errors.As(err, &invalidErr):
      panic(err)
    }
    return err
  }

  if err := dec.Decode(&struct{}{}); !errors.Is(err, io.EOF) {
    return &BadRequest{http.StatusBadRequest, "body must contain exactly one JSON object"}
  }
  return nil
}

type CreateItem struct {
  Name  string  `json:"name"`
  Price float64 `json:"price"`
  Qty   int     `json:"qty"`
}

func writeJSON(w http.ResponseWriter, status int, v any) error {
  buf, err := json.Marshal(v)
  if err != nil {
    return err
  }
  w.Header().Set("Content-Type", "application/json; charset=utf-8")
  w.Header().Set("Content-Length", fmt.Sprint(len(buf)))
  w.WriteHeader(status)
  _, err = w.Write(buf)
  return err
}

func handler(w http.ResponseWriter, r *http.Request) {
  var in CreateItem
  if err := decodeJSON(w, r, &in, 100); err != nil {
    var br *BadRequest
    if errors.As(err, &br) {
      writeJSON(w, br.Status, map[string]string{"error": br.Message})
      return
    }
    writeJSON(w, 500, map[string]string{"error": "internal error"})
    return
  }
  writeJSON(w, http.StatusCreated, map[string]any{"created": in})
}

func main() {
  srv := httptest.NewServer(http.HandlerFunc(handler))
  defer srv.Close()

  cases := []struct{ name, ct, body string }{
    {"valid", "application/json", `{"name":"widget","price":9.99,"qty":3}`},
    {"syntax error", "application/json", `{"name":"widget",}`},
    {"wrong field type", "application/json", `{"name":"widget","qty":"three"}`},
    {"not an object", "application/json", `["a","b"]`},
    {"unknown field", "application/json", `{"name":"w","colour":"red"}`},
    {"empty body", "application/json", ``},
    {"truncated", "application/json", `{"name":"wid`},
    {"too large", "application/json", `{"name":"` + strings.Repeat("A", 200) + `"}`},
    {"two objects", "application/json", `{"name":"a"}{"name":"b"}`},
    {"wrong content-type", "text/plain", `{"name":"a"}`},
    {"malformed content-type", "application/json; charset", `{"name":"a"}`},
  }
  for _, c := range cases {
    req, _ := http.NewRequest("POST", srv.URL, strings.NewReader(c.body))
    req.Header.Set("Content-Type", c.ct)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      fmt.Printf("%-24s client err %v\n", c.name, err)
      continue
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-24s %-32s %s\n", c.name, resp.Status, strings.TrimSpace(string(b)))
  }
}
```

Output:

```
valid                    201 Created                      {"created":{"name":"widget","price":9.99,"qty":3}}
syntax error             400 Bad Request                  {"error":"body contains malformed JSON at character 18"}
wrong field type         400 Bad Request                  {"error":"field \"qty\" must be of type int, got string"}
not an object            400 Bad Request                  {"error":"body must be a JSON object, got array"}
unknown field            400 Bad Request                  {"error":"body contains unknown field \"colour\""}
empty body               400 Bad Request                  {"error":"body must not be empty"}
truncated                400 Bad Request                  {"error":"body contains truncated JSON"}
too large                413 Request Entity Too Large     {"error":"body must not exceed 100 bytes"}
two objects              400 Bad Request                  {"error":"body must contain exactly one JSON object"}
wrong content-type       415 Unsupported Media Type       {"error":"Content-Type must be application/json, got \"text/plain\""}
malformed content-type   415 Unsupported Media Type       {"error":"malformed Content-Type header"}
```

Eleven inputs, eleven messages a client can act on, three distinct status codes. Compare
that with the one-liner it replaces:

```go
if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
  http.Error(w, "bad request", 400)
}
```

Details worth defending:

**The order of the switch cases matters.** `errors.Is(err, io.ErrUnexpectedEOF)` must come
before the `io.EOF` check, because truncated and empty are different messages. And the
`*json.SyntaxError` check goes first because it is the most specific.

**`*json.InvalidUnmarshalError` panics deliberately.** It means `dst` was not a non-nil
pointer — a *programmer* error, so a 400 would hide a bug in your own handler.

**`maxErr.Limit` is in the message.** Telling the client the actual limit is far more useful
than "too large", and it is not sensitive information.

**The second `Decode` into `&struct{}{}` is the trailing-data check**, which must return
`io.EOF` for the body to be exactly one value. And the empty-`Content-Type` case is allowed,
since some clients omit it on a POST — being strict only when the header is *present* is the
common compromise.

## Encoding a response: Marshal beats Encode

`json.NewEncoder(w).Encode(v)` writes directly to the `ResponseWriter`. That sounds
efficient and is the wrong default for an API, because of what happens when encoding fails
*after* the status is already on the wire (16.1).

```go
package main

import (
  "encoding/json"
  "fmt"
  "io"
  "log"
  "net/http"
  "net/http/httptest"
  "os"
)

type Unserializable struct {
  Name string `json:"name"`
  Fn   func() `json:"fn"`
}

func withEncoder(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "application/json")
  w.WriteHeader(http.StatusOK)
  err := json.NewEncoder(w).Encode(Unserializable{Name: "x"})
  fmt.Println("server: Encode err =", err, "-- but status 200 was already sent")
}

func withMarshal(w http.ResponseWriter, r *http.Request) {
  buf, err := json.Marshal(Unserializable{Name: "x"})
  if err != nil {
    fmt.Println("server: Marshal err =", err, "-- status not yet sent, can send 500")
    http.Error(w, "internal error", http.StatusInternalServerError)
    return
  }
  w.Header().Set("Content-Type", "application/json")
  w.Write(buf)
}

func encodeNewline(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "application/json")
  json.NewEncoder(w).Encode(map[string]int{"a": 1})
}

func marshalNoNewline(w http.ResponseWriter, r *http.Request) {
  buf, _ := json.Marshal(map[string]int{"a": 1})
  w.Header().Set("Content-Type", "application/json")
  w.Write(buf)
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/encoder", withEncoder)
  mux.HandleFunc("/marshal", withMarshal)
  mux.HandleFunc("/nl", encodeNewline)
  mux.HandleFunc("/nonl", marshalNoNewline)

  srv := httptest.NewServer(mux)
  srv.Config.ErrorLog = log.New(os.Stdout, "SERVERLOG: ", 0)
  defer srv.Close()

  for _, p := range []string{"/encoder", "/marshal", "/nl", "/nonl"} {
    resp, err := http.Get(srv.URL + p)
    if err != nil {
      fmt.Printf("%-10s client err: %v\n", p, err)
      continue
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-10s %-28s cl=%-3s body=%q\n", p, resp.Status,
      resp.Header.Get("Content-Length"), b)
  }
}
```

Output:

```
server: Encode err = json: unsupported type: func() -- but status 200 was already sent
/encoder   200 OK                       cl=0   body=""
server: Marshal err = json: unsupported type: func() -- status not yet sent, can send 500
/marshal   500 Internal Server Error    cl=15  body="internal error\n"
/nl        200 OK                       cl=8   body="{\"a\":1}\n"
/nonl      200 OK                       cl=7   body="{\"a\":1}"
```

`/encoder` sent the client **`200 OK` with an empty body**. The encode failed, the handler
*knows* it failed, and it is powerless: the status line went out with `WriteHeader`, so
there is no way to say 500. A client sees a successful response containing nothing — which
is far worse than an error, because it will try to parse it.

`/marshal` failed into a buffer first, so the status had not been chosen yet, and the
handler sent a clean 500.

The last two lines show the other difference: `Encode` **appends a newline**
(`cl=8` vs `cl=7`). It is harmless JSON whitespace, but it breaks byte-exact test assertions,
which is a real annoyance in 16.11.

Idiomatic for an API: marshal to a buffer, then set headers, then `WriteHeader`, then
`Write`. Use `Encode` only when streaming is the point — a large array (below) or
newline-delimited JSON — accepting that an error mid-stream truncates the response. The
`writeJSON` helper from the previous example is the shape to standardize on:

```go
func writeJSON(w http.ResponseWriter, status int, v any) error {
  buf, err := json.Marshal(v)
  if err != nil {
    return err
  }
  w.Header().Set("Content-Type", "application/json; charset=utf-8")
  w.Header().Set("Content-Length", fmt.Sprint(len(buf)))
  w.WriteHeader(status)
  _, err = w.Write(buf)
  return err
}
```

Setting `Content-Type` explicitly is mandatory — Go's sniffing guesses `text/plain` for JSON
(16.1). Setting `Content-Length` is optional (Go computes it for small buffered responses)
but makes it explicit and lets clients show progress.

## Status codes for an API

| Operation | Status | Body | Extra header |
|---|---|---|---|
| `GET` collection / one | 200 | the array (or envelope) / the object | |
| `POST` create | **201** | the created object, with its ID | `Location: /items/{id}` |
| `POST` accepted for async work | 202 | a job handle | |
| `PUT`/`PATCH` update | 200, or 204 with no body | the updated object | |
| `DELETE` | **204** | **empty** | |
| `DELETE`, already gone | 204 or 404 | pick one and document it | |

201 with a `Location` header and the created object is worth the small extra effort: the
client learns the server-assigned ID without a second request. 204 means *no body* — Go
will not send one even if you try (16.1).

## Optional fields: omitempty vs a pointer

Three ways a field can be absent from a response, and they are not interchangeable
([13.6](../13-stdlib/06-json.md)):

```go
type Item struct {
  Tags []string `json:"tags,omitempty"`  // key disappears when empty
  Note *string  `json:"note"`            // key present, value null when nil
  Qty  int      `json:"qty"`             // always present, 0 when unset
}
```

- **`omitempty`** drops the key for the zero value — right for optional collections, but it
  cannot distinguish `0` from "unset", so `Qty int` with `omitempty` hides a legitimate zero.
- **a pointer** gives three states — absent, explicit `null`, present — which is what a PATCH
  endpoint needs, at the cost of an allocation and a nil check at every use.
- **neither** always emits the key. Simplest, and best when the zero value is meaningful.

Response envelope, one line each: a bare array/object is what most clients expect; an envelope
(`{"data":[...],"meta":{...}}`) gives pagination somewhere to live. Pick one and be consistent
— mixing them is the actual mistake.

## Streaming a large JSON array

When the collection does not fit comfortably in memory, write the array brackets yourself
and `Encode` one element at a time.

```go
func streamArray(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "application/json; charset=utf-8")
  w.WriteHeader(http.StatusOK)

  enc := json.NewEncoder(w)
  io.WriteString(w, `{"items":[`)
  for i := 1; i <= 4; i++ {
    if i > 1 {
      io.WriteString(w, ",")
    }
    if err := enc.Encode(Row{ID: i, Name: fmt.Sprintf("row-%d", i)}); err != nil {
      return
    }
  }
  io.WriteString(w, `],"count":4}`)
}
```

Output:

```
200 OK cl="121" te=[]
raw body:
{"items":[{"id":1,"name":"row-1"}
,{"id":2,"name":"row-2"}
,{"id":3,"name":"row-3"}
,{"id":4,"name":"row-4"}
],"count":4}
re-parsed: err=<nil> count=4 items=[{1 row-1} {2 row-2} {3 row-3} {4 row-4}]
```

The newlines are `Encode`'s doing, and they are legal JSON whitespace — the re-parse
succeeds. The honest trade-off: you committed to a 200 before the first row, so a database
error on row 5000 leaves the client with truncated JSON and no way to know it was an error.
Prefer a bounded page for paginated endpoints; stream only for a genuine export, and consider
newline-delimited JSON (`application/x-ndjson`), which recovers cleanly from truncation.

## A complete CRUD API

Everything so far, as one service: a mutex-guarded in-memory store, a handler struct
(16.3), `writeJSON`, proper status codes, and the stdlib router (16.2). This is the API
16.11 writes tests for.

```go
package main

import (
  "encoding/json"
  "errors"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
  "sync"
)

type Item struct {
  ID    int      `json:"id"`
  Name  string   `json:"name"`
  Price float64  `json:"price"`
  Tags  []string `json:"tags,omitempty"`
  Note  *string  `json:"note"`
}

type Store struct {
  mu     sync.Mutex
  items  map[int]Item
  nextID int
}

func NewStore() *Store {
  return &Store{items: map[int]Item{}, nextID: 1}
}

var ErrNotFound = errors.New("item not found")

func (s *Store) Create(it Item) Item {
  s.mu.Lock()
  defer s.mu.Unlock()
  it.ID = s.nextID
  s.nextID++
  s.items[it.ID] = it
  return it
}

func (s *Store) Get(id int) (Item, error) {
  s.mu.Lock()
  defer s.mu.Unlock()
  it, ok := s.items[id]
  if !ok {
    return Item{}, ErrNotFound
  }
  return it, nil
}

func (s *Store) List() []Item {
  s.mu.Lock()
  defer s.mu.Unlock()
  out := make([]Item, 0, len(s.items))
  for id := 1; id < s.nextID; id++ {
    if it, ok := s.items[id]; ok {
      out = append(out, it)
    }
  }
  return out
}

func (s *Store) Update(id int, it Item) (Item, error) {
  s.mu.Lock()
  defer s.mu.Unlock()
  if _, ok := s.items[id]; !ok {
    return Item{}, ErrNotFound
  }
  it.ID = id
  s.items[id] = it
  return it, nil
}

func (s *Store) Delete(id int) error {
  s.mu.Lock()
  defer s.mu.Unlock()
  if _, ok := s.items[id]; !ok {
    return ErrNotFound
  }
  delete(s.items, id)
  return nil
}

func writeJSON(w http.ResponseWriter, status int, v any) {
  buf, err := json.Marshal(v)
  if err != nil {
    http.Error(w, `{"error":"encoding failure"}`, http.StatusInternalServerError)
    return
  }
  w.Header().Set("Content-Type", "application/json; charset=utf-8")
  w.Header().Set("Content-Length", fmt.Sprint(len(buf)))
  w.WriteHeader(status)
  w.Write(buf)
}

func writeErr(w http.ResponseWriter, status int, msg string) {
  writeJSON(w, status, map[string]string{"error": msg})
}

type API struct{ store *Store }

func (a *API) list(w http.ResponseWriter, r *http.Request) {
  writeJSON(w, http.StatusOK, a.store.List())
}

func (a *API) create(w http.ResponseWriter, r *http.Request) {
  var in Item
  r.Body = http.MaxBytesReader(w, r.Body, 1<<16)
  dec := json.NewDecoder(r.Body)
  dec.DisallowUnknownFields()
  if err := dec.Decode(&in); err != nil {
    writeErr(w, http.StatusBadRequest, "invalid JSON body")
    return
  }
  if strings.TrimSpace(in.Name) == "" {
    writeErr(w, http.StatusUnprocessableEntity, "name is required")
    return
  }
  created := a.store.Create(in)
  w.Header().Set("Location", fmt.Sprintf("/items/%d", created.ID))
  writeJSON(w, http.StatusCreated, created)
}

func (a *API) get(w http.ResponseWriter, r *http.Request) {
  id, err := itemID(r)
  if err != nil {
    writeErr(w, http.StatusBadRequest, "id must be a positive integer")
    return
  }
  it, err := a.store.Get(id)
  if errors.Is(err, ErrNotFound) {
    writeErr(w, http.StatusNotFound, "item not found")
    return
  }
  writeJSON(w, http.StatusOK, it)
}

func (a *API) update(w http.ResponseWriter, r *http.Request) {
  id, err := itemID(r)
  if err != nil {
    writeErr(w, http.StatusBadRequest, "id must be a positive integer")
    return
  }
  var in Item
  if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
    writeErr(w, http.StatusBadRequest, "invalid JSON body")
    return
  }
  updated, err := a.store.Update(id, in)
  if errors.Is(err, ErrNotFound) {
    writeErr(w, http.StatusNotFound, "item not found")
    return
  }
  writeJSON(w, http.StatusOK, updated)
}

func (a *API) remove(w http.ResponseWriter, r *http.Request) {
  id, err := itemID(r)
  if err != nil {
    writeErr(w, http.StatusBadRequest, "id must be a positive integer")
    return
  }
  if err := a.store.Delete(id); errors.Is(err, ErrNotFound) {
    writeErr(w, http.StatusNotFound, "item not found")
    return
  }
  w.WriteHeader(http.StatusNoContent)
}

func itemID(r *http.Request) (int, error) {
  var id int
  _, err := fmt.Sscanf(r.PathValue("id"), "%d", &id)
  if err != nil || id < 1 {
    return 0, errors.New("bad id")
  }
  return id, nil
}

func NewRouter(s *Store) http.Handler {
  a := &API{store: s}
  mux := http.NewServeMux()
  mux.HandleFunc("GET /items", a.list)
  mux.HandleFunc("POST /items", a.create)
  mux.HandleFunc("GET /items/{id}", a.get)
  mux.HandleFunc("PUT /items/{id}", a.update)
  mux.HandleFunc("DELETE /items/{id}", a.remove)
  return mux
}

func main() {
  srv := httptest.NewServer(NewRouter(NewStore()))
  defer srv.Close()

  do := func(method, path, body string) {
    var rdr io.Reader
    if body != "" {
      rdr = strings.NewReader(body)
    }
    req, _ := http.NewRequest(method, srv.URL+path, rdr)
    if body != "" {
      req.Header.Set("Content-Type", "application/json")
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      fmt.Printf("%-6s %-12s client err %v\n", method, path, err)
      return
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    loc := ""
    if l := resp.Header.Get("Location"); l != "" {
      loc = " Location=" + l
    }
    fmt.Printf("%-6s %-12s %-28s%s\n", method, path, resp.Status, loc)
    if len(b) > 0 {
      fmt.Printf("       %s\n", strings.TrimSpace(string(b)))
    }
  }

  do("GET", "/items", "")
  do("POST", "/items", `{"name":"widget","price":9.99,"tags":["a","b"],"note":null}`)
  do("POST", "/items", `{"name":"gadget","price":19.5}`)
  do("POST", "/items", `{"name":"  "}`)
  do("POST", "/items", `{"name":"x","bogus":1}`)
  do("GET", "/items", "")
  do("GET", "/items/1", "")
  do("GET", "/items/99", "")
  do("GET", "/items/abc", "")
  do("PUT", "/items/1", `{"name":"widget v2","price":11.0,"note":"restocked"}`)
  do("DELETE", "/items/2", "")
  do("DELETE", "/items/2", "")
  do("GET", "/items", "")
  do("PATCH", "/items/1", "")
}
```

Output (clean under `go run -race .`):

```
GET    /items       200 OK                      
       []
POST   /items       201 Created                  Location=/items/1
       {"id":1,"name":"widget","price":9.99,"tags":["a","b"],"note":null}
POST   /items       201 Created                  Location=/items/2
       {"id":2,"name":"gadget","price":19.5,"note":null}
POST   /items       422 Unprocessable Entity    
       {"error":"name is required"}
POST   /items       400 Bad Request             
       {"error":"invalid JSON body"}
GET    /items       200 OK                      
       [{"id":1,"name":"widget","price":9.99,"tags":["a","b"],"note":null},{"id":2,"name":"gadget","price":19.5,"note":null}]
GET    /items/1     200 OK                      
       {"id":1,"name":"widget","price":9.99,"tags":["a","b"],"note":null}
GET    /items/99    404 Not Found               
       {"error":"item not found"}
GET    /items/abc   400 Bad Request             
       {"error":"id must be a positive integer"}
PUT    /items/1     200 OK                      
       {"id":1,"name":"widget v2","price":11,"note":"restocked"}
DELETE /items/2     204 No Content              
DELETE /items/2     404 Not Found               
       {"error":"item not found"}
GET    /items       200 OK                      
       [{"id":1,"name":"widget v2","price":11,"note":"restocked"}]
PATCH  /items/1     405 Method Not Allowed      
       Method Not Allowed
```

Read the output for the JSON details this lesson has been building toward:

**`[]`, not `null`.** `List` returns `make([]Item, 0, n)`, so an empty collection marshals
to `[]`. A nil slice marshals to `null`, which breaks every client that does
`data.forEach`. Always initialize a slice you are going to marshal.

**`tags` vanished on item 2 but `note` did not.** `omitempty` dropped the empty `Tags`;
`*string` with no `omitempty` emitted `"note":null`. Exactly the three-state distinction
from above, visible in one response.

**`"price":11`, not `11.0`.** `float64` marshals in the shortest round-trip form. Monetary
values belong in integer cents or a decimal type, not `float64`.

**The 405 came free** from the stdlib mux (16.2) — nobody wrote a `PATCH` handler, and
nobody wrote the 405 either. Note its body is plain text, not JSON, because it is the mux's
built-in response: making *every* error JSON needs a custom 405 handler, which is 16.7 and
16.9.

**422 for a semantically invalid field, 400 for unparseable JSON.** That split is the
convention this course follows; 16.9 settles the debate.

## Quick Reference

| Decode | Call | Note |
|---|---|---|
| stream-decode | `json.NewDecoder(r.Body).Decode(&v)` | preferred for request bodies |
| reject extra keys | `dec.DisallowUnknownFields()` | error is untyped; match the message prefix |
| numbers as strings | `dec.UseNumber()` | avoids float64 precision loss for big ints |
| trailing data check | second `Decode` must give `io.EOF` | |
| bound the body | `http.MaxBytesReader(w, r.Body, n)` | **before** the decoder |

| Error | Check | Status |
|---|---|---|
| `*json.SyntaxError` | `errors.As` | 400 |
| `*json.UnmarshalTypeError` | `errors.As` | 400 (`.Field` empty ⇒ whole body wrong) |
| `io.EOF` | `errors.Is` | 400 "body must not be empty" |
| `io.ErrUnexpectedEOF` | `errors.Is` | 400 "truncated" |
| unknown field | `strings.HasPrefix(err.Error(), "json: unknown field ")` | 400 |
| `*http.MaxBytesError` | `errors.As` | **413** |
| `*json.InvalidUnmarshalError` | `errors.As` | **panic** — your bug, not theirs |

| Encode | Call | When |
|---|---|---|
| buffer then write | `json.Marshal` + `w.Write` | **default for APIs** — an error is still fixable |
| stream | `json.NewEncoder(w).Encode` | large arrays, NDJSON; appends `\n` |
| indent | `json.MarshalIndent(v, "", "  ")` | debug endpoints only |

## Common Mistakes

**Using `json.NewEncoder(w).Encode` after `WriteHeader`.** An encode failure then yields a
200 with an empty body and no way to correct it; marshal to a buffer first.

**Returning one generic 400 for every decode failure.** Clients cannot fix what they cannot
see; classify with the `errors.As`/`errors.Is` switch and name the offending field.

**Decoding without `MaxBytesReader`.** A single request can exhaust your memory, and the
limit must be applied *before* the decoder reads anything.

**Marshaling a nil slice.** It becomes `null` rather than `[]`, which breaks clients that
iterate; build it with `make([]T, 0, n)`.

**Forgetting `Content-Type: application/json`.** Go's sniffing labels JSON as `text/plain`,
so strict clients reject a perfectly good response.

**Using `omitempty` on a meaningful zero.** `Qty int` with `omitempty` hides a legitimate
`0`, and `Active bool` hides `false`; use a pointer or drop the tag.

**Trusting `Decode` to reject a `null` body.** `null` decodes with no error into a
zero-value struct, so the check belongs in validation (16.8), not in the decoder.

## Exercises

1. Extend `decodeJSON` to also report the **line and column** of a syntax error, by
   buffering the body and converting `syntaxErr.Offset` into a position. Verify against a
   multi-line body with the error on line 3.
2. Add `UseNumber()` to the decoder and show, with a real captured value, what happens to
   the ID `12345678901234567890` with and without it. Explain which clients this breaks.
3. Write `writeJSON` and `writeError` helpers, then deliberately pass a value containing a
   channel. Show that the 500 path works, and then make the helper log the internal error
   while still returning a safe message (preview of 16.9).
4. Convert the CRUD API's `PUT` into a true `PATCH` using `*string` and `*float64` fields,
   so that an omitted field leaves the value alone and an explicit `null` clears it. Prove
   all three states with three real requests.
5. Add `GET /items?page=&limit=` with an envelope
   `{"data":[...],"meta":{"page":1,"limit":20,"total":57}}`, returning 400 for a
   non-integer `limit` and clamping it to 100. Exercise it through `httptest` and show the
   real responses for page 1, the last page, and a page past the end.

---

[← 16.4 Request Parsing: Path, Query, Headers, Body](04-request-parsing.md) · [Course Index](../README.md) · [16.6 Middleware from Scratch →](06-middleware.md)
