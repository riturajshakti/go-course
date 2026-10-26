[← 13.6 encoding/json: Marshal and Unmarshal](06-json.md) · [Course Index](../README.md) · [13.8 regexp →](08-regexp.md)

# 13.7 — encoding/json: Streaming, Tags, and Custom Marshalers

## What You'll Learn

- `Encoder`/`Decoder` for streams: `More()`, `Token()`, `DisallowUnknownFields`, NDJSON
- Implementing `json.Marshaler`/`Unmarshaler`, and the **infinite recursion** trap with its type-alias fix
- `encoding.TextMarshaler` and that `json` uses it for **map keys**
- `json.RawMessage` for a polymorphic envelope — the single most useful pattern here

## Coming From Another Language

| Other | Go |
|---|---|
| `JSON.parse` on a whole string | `Decoder` reads from an `io.Reader`, incrementally |
| `JSONStream` / `oboe.js` for huge arrays | `Decoder.Token()` + `Decoder.More()` |
| Pydantic validators | `UnmarshalJSON` on a named type |
| `toJSON()` on a class | `MarshalJSON() ([]byte, error)` |
| TS discriminated unions | `json.RawMessage` payload + a `switch` on the tag |
| `JSON.parse(s, reviver)` | no reviver; implement `UnmarshalJSON` per type |

The structural difference from the previous lesson is that `Decoder` is a *stream* parser:
it can read a 10GB array one element at a time without holding it in memory, and it can
read several JSON values concatenated in one stream. The important design consequence is
that `json.Unmarshal` validates the entire input while `Decoder.Decode` deliberately does
not — it stops at the end of the first value and leaves the rest for you, which is both
what makes NDJSON work and a real source of bugs. The other Go-specific thing is that
custom marshaling is interface satisfaction, not inheritance, so the obvious
implementation of `MarshalJSON` recurses infinitely and the fix is a type alias.

## Encoder and Decoder

| Method | On | Does |
|---|---|---|
| `NewEncoder(w)` | | encoder writing to `w` |
| `Encode(v)` | `*Encoder` | one value **plus a newline** |
| `SetIndent(prefix, indent)` | `*Encoder` | pretty-print subsequent values |
| `SetEscapeHTML(bool)` | `*Encoder` | the only way to disable `<` escaping |
| `NewDecoder(r)` | | decoder reading from `r` |
| `Decode(&v)` | `*Decoder` | the **next** value; `io.EOF` when done |
| `More()` | `*Decoder` | is there another element/value in the stream? |
| `Token()` | `*Decoder` | the next lexical token |
| `DisallowUnknownFields()` | `*Decoder` | error on a key with no matching field |
| `InputOffset()` | `*Decoder` | byte offset just after the last token |

```go
package main

import (
  "bytes"
  "encoding/json"
  "fmt"
  "strings"
)

type ev struct {
  Seq  int    `json:"seq"`
  Name string `json:"name"`
}

func main() {
  fmt.Println("--- Encoder ---")
  var buf bytes.Buffer
  enc := json.NewEncoder(&buf)
  for i := 1; i <= 3; i++ {
    enc.Encode(ev{Seq: i, Name: fmt.Sprintf("e%d", i)})
  }
  fmt.Printf("%q\n", buf.String())

  buf.Reset()
  enc = json.NewEncoder(&buf)
  enc.SetIndent("", "  ")
  enc.Encode(ev{Seq: 1, Name: "indented"})
  fmt.Print(buf.String())

  fmt.Println("--- Decoder on concatenated values with More() ---")
  dec := json.NewDecoder(strings.NewReader(`{"seq":1,"name":"a"}{"seq":2,"name":"b"} {"seq":3,"name":"c"}`))
  for dec.More() {
    var e ev
    if err := dec.Decode(&e); err != nil {
      fmt.Println("  err:", err)
      break
    }
    fmt.Printf("  %+v offset=%d\n", e, dec.InputOffset())
  }

  fmt.Println("--- DisallowUnknownFields ---")
  var e ev
  fmt.Println("  default:", json.NewDecoder(strings.NewReader(`{"seq":1,"nope":true}`)).Decode(&e))
  dec = json.NewDecoder(strings.NewReader(`{"seq":1,"nope":true}`))
  dec.DisallowUnknownFields()
  fmt.Println("  strict: ", dec.Decode(&e))

  fmt.Println("--- Decode does NOT reject trailing content ---")
  dec = json.NewDecoder(strings.NewReader(`{"seq":1} GARBAGE`))
  fmt.Println("  Decode:", dec.Decode(&e))
  fmt.Println("  Unmarshal on the same bytes:", json.Unmarshal([]byte(`{"seq":1} GARBAGE`), &e))
  fmt.Println("  More() after:", dec.More())

}
```

Output:

```
--- Encoder ---
"{\"seq\":1,\"name\":\"e1\"}\n{\"seq\":2,\"name\":\"e2\"}\n{\"seq\":3,\"name\":\"e3\"}\n"
{
  "seq": 1,
  "name": "indented"
}
--- Decoder on concatenated values with More() ---
  {Seq:1 Name:a} offset=20
  {Seq:2 Name:b} offset=40
  {Seq:3 Name:c} offset=61
--- DisallowUnknownFields ---
  default: <nil>
  strict:  json: unknown field "nope"
--- Decode does NOT reject trailing content ---
  Decode: <nil>
  Unmarshal on the same bytes: invalid character 'G' after top-level value
  More() after: true
```

`Encode` writes a trailing newline, which is what makes a loop of `Encode` calls produce
valid NDJSON for free — and why `httptest` body assertions need a `strings.TrimSpace`.

`Decode` on concatenated values just works: `{"a":1}{"b":2}` needs no separator, and
whitespace between values is fine. `More()` is the loop condition when you know the stream
is a sequence; `errors.Is(err, io.EOF)` is the one when you don't.

**`Decode` does not reject trailing garbage.** `{"seq":1} GARBAGE` decodes cleanly while
`json.Unmarshal` on the identical bytes errors. That is correct for a stream — the decoder
has not read that far — but it means `Decode(r.Body)` accepts `{"valid":1}<script>` without
complaint. For `Unmarshal`-like strictness, check `dec.More()` afterwards, as in the
generic helper below. `DisallowUnknownFields` turns a client's typo into an error instead
of a silently empty field — the most valuable decoder setting for an API.

### NDJSON, complete

```go
package main

import (
  "bufio"
  "bytes"
  "encoding/json"
  "errors"
  "fmt"
  "io"
  "strings"
)

type logLine struct {
  Level string `json:"level"`
  Msg   string `json:"msg"`
  Code  int    `json:"code,omitzero"`
}

func writeNDJSON(w io.Writer, lines []logLine) error {
  bw := bufio.NewWriter(w)
  enc := json.NewEncoder(bw)
  for _, l := range lines {
    if err := enc.Encode(l); err != nil {
      return err
    }
  }
  return bw.Flush()
}

func readNDJSON(r io.Reader) ([]logLine, error) {
  var out []logLine
  dec := json.NewDecoder(r)
  for {
    var l logLine
    err := dec.Decode(&l)
    if errors.Is(err, io.EOF) {
      return out, nil
    }
    if err != nil {
      return out, fmt.Errorf("record %d: %w", len(out)+1, err)
    }
    out = append(out, l)
  }
}

func main() {
  var buf bytes.Buffer
  writeNDJSON(&buf, []logLine{
    {Level: "info", Msg: "started"},
    {Level: "warn", Msg: "slow query", Code: 1},
    {Level: "error", Msg: "crash", Code: 500},
  })
  fmt.Print(buf.String())

  back, err := readNDJSON(bytes.NewReader(buf.Bytes()))
  fmt.Println("read err:", err, "records:", len(back))

  bad := "{\"level\":\"info\"}\n{\"level\":}\n{\"level\":\"ok\"}\n"
  got, err := readNDJSON(strings.NewReader(bad))
  fmt.Println("records before failure:", len(got))
  fmt.Println("err:", err)
}
```

Output:

```
{"level":"info","msg":"started"}
{"level":"warn","msg":"slow query","code":1}
{"level":"error","msg":"crash","code":500}
read err: <nil> records: 3
records before failure: 1
err: record 2: invalid character '}' looking for beginning of value
```

Neither side buffers the whole file. The writer wraps a `bufio.Writer` (13.3) so each
record is not a syscall, and the reader returns the records it read alongside the error —
what a log pipeline wants, rather than discarding 999,999 good records because the
millionth was malformed. The decoder does not need the newlines at all; NDJSON is a
convention for humans and line-based tools.

## Token() for a huge array

`Decode` on a `[]T` buffers the whole array. `Token()` plus `More()` lets you walk into the
array and decode one element at a time, with constant memory.

```go
package main

import (
  "encoding/json"
  "fmt"
  "strings"
)

type item struct {
  ID   int    `json:"id"`
  Name string `json:"name"`
}

func streamItems(r *strings.Reader) error {
  dec := json.NewDecoder(r)

  t, err := dec.Token()
  if err != nil {
    return err
  }
  if d, ok := t.(json.Delim); !ok || d != '{' {
    return fmt.Errorf("expected object start, got %v", t)
  }

  for dec.More() {
    keyTok, err := dec.Token()
    if err != nil {
      return err
    }
    key, _ := keyTok.(string)

    if key != "items" {
      var skip any
      if err := dec.Decode(&skip); err != nil {
        return err
      }
      fmt.Printf("  key %q = %v (decoded whole)\n", key, skip)
      continue
    }

    open, err := dec.Token()
    if err != nil {
      return err
    }
    fmt.Printf("  key %q opens with %v\n", key, open)

    count := 0
    for dec.More() {
      var it item
      if err := dec.Decode(&it); err != nil {
        return err
      }
      count++
      fmt.Printf("    item %d: %+v (offset %d)\n", count, it, dec.InputOffset())
    }
    if _, err := dec.Token(); err != nil {
      return err
    }
    fmt.Println("    streamed", count, "items without buffering the array")
  }

  _, err = dec.Token()
  return err
}

func main() {
  fmt.Println("--- Token() token types ---")
  dec := json.NewDecoder(strings.NewReader(`{"a":[1,"x",true,null]}`))
  for {
    t, err := dec.Token()
    if err != nil {
      break
    }
    fmt.Printf("  %-10T %v\n", t, t)
  }

  fmt.Println("\n--- streaming a huge array element by element ---")
  var sb strings.Builder
  sb.WriteString(`{"total":3,"items":[`)
  for i := 1; i <= 3; i++ {
    if i > 1 {
      sb.WriteString(",")
    }
    fmt.Fprintf(&sb, `{"id":%d,"name":"item-%d"}`, i, i)
  }
  sb.WriteString(`]}`)

  if err := streamItems(strings.NewReader(sb.String())); err != nil {
    fmt.Println("  error:", err)
  }
}
```

Output:

```
--- Token() token types ---
  json.Delim {
  string     a
  json.Delim [
  float64    1
  string     x
  bool       true
  <nil>      <nil>
  json.Delim ]
  json.Delim }

--- streaming a huge array element by element ---
  key "total" = 3 (decoded whole)
  key "items" opens with [
    item 1: {ID:1 Name:item-1} (offset 44)
    item 2: {ID:2 Name:item-2} (offset 69)
    item 3: {ID:3 Name:item-3} (offset 94)
    streamed 3 items without buffering the array
```

`Token()` returns exactly five things: `json.Delim` (a `rune` — one of `{ } [ ]`), `string`,
`float64`, `bool`, or `nil`. Object keys arrive as plain `string` tokens, indistinguishable
from string values except by position, so you track depth yourself — which is why mixing
`Token()` and `Decode()` as above beats tokenizing everything.

The key move is `Decode(&it)` **inside** a `Token()`-driven loop: `Decode` consumes exactly
one array element and leaves the decoder at the next comma, so the two APIs interleave
cleanly. `InputOffset()` advancing by 25 bytes per element confirms nothing is buffered.

Idiomatic: `Decode` for a value you can hold in memory; `Token()` + `More()` + per-element
`Decode` for an array that you cannot.

## Custom marshaling, and the recursion trap

```go
type Marshaler interface {
  MarshalJSON() ([]byte, error)
}

type Unmarshaler interface {
  UnmarshalJSON([]byte) error
}
```

The obvious implementation is an infinite loop:

```go
package main

import (
  "encoding/json"
  "fmt"
)

type Money struct {
  Cents int    `json:"cents"`
  Cur   string `json:"cur"`
}

func (m Money) MarshalJSON() ([]byte, error) {
  return json.Marshal(m)
}

func main() {
  b, err := json.Marshal(Money{Cents: 1250, Cur: "USD"})
  fmt.Println(string(b), err)
}
```

This crashes on purpose. Output (truncated; the real trace is thousands of frames):

```
runtime: goroutine stack exceeds 1000000000-byte limit
runtime: sp=0x7d17717603a0 stack=[0x7d1771760000, 0x7d1791760000]
fatal error: stack overflow

runtime stack:
runtime.throw({0x1022ad7a1?, 0x1021dc354?})
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/panic.go:1243 +0x38 fp=0x16ddf6da0 sp=0x16ddf6d70 pc=0x10221c828
runtime.newstack()
```

`json.Marshal(m)` sees that `Money` has a `MarshalJSON` method and calls it, which calls
`json.Marshal(m)`, forever. It is not a compile error and not a `panic` you can `recover` —
a stack overflow is a `fatal error` that kills the process. The same trap exists in
`UnmarshalJSON`.

The fix is a **type alias** (strictly, a new defined type with the same underlying type)
that does *not* carry the method:

```go
package main

import (
  "encoding/json"
  "fmt"
  "slices"
  "strings"
  "time"
)

type Money struct {
  Cents int    `json:"cents"`
  Cur   string `json:"cur"`
}

func (m Money) MarshalJSON() ([]byte, error) {
  type alias Money
  return json.Marshal(struct {
    alias
    Display string `json:"display"`
  }{alias(m), fmt.Sprintf("%d.%02d %s", m.Cents/100, m.Cents%100, m.Cur)})
}

type Date struct {
  time.Time
}

const dateLayout = "2006-01-02"

func (d Date) MarshalJSON() ([]byte, error) {
  return json.Marshal(d.Format(dateLayout))
}

func (d *Date) UnmarshalJSON(b []byte) error {
  var s string
  if err := json.Unmarshal(b, &s); err != nil {
    return err
  }
  t, err := time.ParseInLocation(dateLayout, s, time.UTC)
  if err != nil {
    return fmt.Errorf("date %q: want layout %q: %w", s, dateLayout, err)
  }
  d.Time = t
  return nil
}

type Status int

const (
  StatusUnknown Status = iota
  StatusActive
  StatusSuspended
)

var statusNames = map[Status]string{
  StatusActive:    "active",
  StatusSuspended: "suspended",
}

func (s Status) MarshalJSON() ([]byte, error) {
  name, ok := statusNames[s]
  if !ok {
    return nil, fmt.Errorf("invalid Status %d", int(s))
  }
  return json.Marshal(name)
}

func (s *Status) UnmarshalJSON(b []byte) error {
  var raw string
  if err := json.Unmarshal(b, &raw); err != nil {
    return fmt.Errorf("status must be a string: %w", err)
  }
  for k, v := range statusNames {
    if v == raw {
      *s = k
      return nil
    }
  }
  valid := make([]string, 0, len(statusNames))
  for _, v := range statusNames {
    valid = append(valid, v)
  }
  slices.Sort(valid)
  return fmt.Errorf("invalid status %q (want one of: %s)", raw, strings.Join(valid, ", "))
}

type account struct {
  Name   string `json:"name"`
  Since  Date   `json:"since"`
  Status Status `json:"status"`
  Fee    Money  `json:"fee"`
}

func main() {
  b, err := json.Marshal(Money{Cents: 1250, Cur: "USD"})
  fmt.Println("alias fix, no recursion:", string(b), err)

  a := account{
    Name:   "acme",
    Since:  Date{time.Date(2026, 3, 7, 0, 0, 0, 0, time.UTC)},
    Status: StatusActive,
    Fee:    Money{Cents: 999, Cur: "EUR"},
  }
  b, _ = json.Marshal(a)
  fmt.Println("marshalled:", string(b))

  var back account
  err = json.Unmarshal([]byte(`{"name":"acme","since":"2026-03-07","status":"suspended","fee":{"cents":1,"cur":"GBP"}}`), &back)
  fmt.Printf("unmarshalled: err=%v status=%d since=%s\n",
    err, back.Status, back.Since.Format(dateLayout))

  fmt.Println("\n--- validation errors ---")
  for _, in := range []string{
    `{"status":"deleted"}`,
    `{"status":1}`,
    `{"since":"07/03/2026"}`,
  } {
    var acc account
    fmt.Printf("  %-24s %v\n", in, json.Unmarshal([]byte(in), &acc))
  }

  _, err = json.Marshal(account{Status: Status(99)})
  fmt.Println("\ninvalid enum on marshal:", err)
}
```

Output:

```
alias fix, no recursion: {"cents":1250,"cur":"USD","display":"12.50 USD"} <nil>
marshalled: {"name":"acme","since":"2026-03-07","status":"active","fee":{"cents":999,"cur":"EUR","display":"9.99 EUR"}}
unmarshalled: err=<nil> status=2 since=2026-03-07

--- validation errors ---
  {"status":"deleted"}     invalid status "deleted" (want one of: active, suspended)
  {"status":1}             status must be a string: json: cannot unmarshal number into Go value of type string
  {"since":"07/03/2026"}   date "07/03/2026": want layout "2006-01-02": parsing time "07/03/2026" as "2006-01-02": cannot parse "07/03/2026" as "2006"

invalid enum on marshal: json: error calling MarshalJSON for type *main.Status: invalid Status 99
```

`type alias Money` inside the method body creates a defined type with `Money`'s fields and
**none** of its methods, so `json.Marshal` on it uses the normal reflection path. Embedding
it in an anonymous struct flattens its fields (13.6) and lets you add `display` alongside —
the canonical "add a computed field" idiom.

The receiver types matter and are easy to get wrong. `MarshalJSON` on a **value** receiver
works for both `Money` and `*Money`. `UnmarshalJSON` must be on a **pointer** receiver,
since it mutates — with a value receiver it compiles, is silently never called, and your
custom parsing simply does not happen.

`Status` is the enum-as-string pattern, validating in both directions: unmarshalling
rejects an unknown name *with the valid set in the message*, and marshalling an
out-of-range value returns an error that `json.Marshal` wraps as
`json: error calling MarshalJSON for type *main.Status`. That wrapper is
`*json.MarshalerError`, whose `Unwrap` gives you your own error back.

Note how much better the `Date` error is than a bare `time.Parse` failure: it names the
value, the expected layout, and keeps the underlying error for `errors.Is`. That is what
makes a 400 response useful.

## TextMarshaler, and map keys

`encoding/json` checks for `encoding.TextMarshaler`/`TextUnmarshaler` as a fallback, and
it is the **only** way to use a struct as a map key.

```go
package main

import (
  "encoding/json"
  "fmt"
  "net/netip"
  "strings"
)

type Region struct {
  Country string
  City    string
}

func (r Region) MarshalText() ([]byte, error) {
  if r.Country == "" {
    return nil, fmt.Errorf("Region: empty country")
  }
  return []byte(r.Country + "/" + r.City), nil
}

func (r *Region) UnmarshalText(b []byte) error {
  country, city, ok := strings.Cut(string(b), "/")
  if !ok {
    return fmt.Errorf("Region %q: want COUNTRY/CITY", b)
  }
  r.Country, r.City = country, city
  return nil
}

type both struct{}

func (both) MarshalText() ([]byte, error) { return []byte("from-text"), nil }
func (both) MarshalJSON() ([]byte, error) { return []byte(`"from-json"`), nil }

func main() {
  fmt.Println("--- TextMarshaler used for MAP KEYS ---")
  m := map[Region]int{
    {Country: "DE", City: "Berlin"}: 10,
    {Country: "FR", City: "Paris"}:  20,
    {Country: "AT", City: "Vienna"}: 5,
  }
  b, err := json.Marshal(m)
  fmt.Println(" ", string(b), err)

  var back map[Region]int
  err = json.Unmarshal(b, &back)
  fmt.Println("  unmarshal err:", err, "len:", len(back))
  fmt.Println("  DE/Berlin:", back[Region{Country: "DE", City: "Berlin"}])

  b, _ = json.Marshal(Region{Country: "IT", City: "Rome"})
  fmt.Println("  as a VALUE:", string(b))

  fmt.Println("\n--- errors from TextMarshaler/TextUnmarshaler ---")
  _, err = json.Marshal(map[Region]int{{}: 1})
  fmt.Println("  marshal:", err)
  err = json.Unmarshal([]byte(`{"bad-key":1}`), &back)
  fmt.Println("  unmarshal:", err)

  addr := netip.MustParseAddr("192.0.2.1")
  b, _ = json.Marshal(map[netip.Addr]string{addr: "web"})
  fmt.Println("  netip.Addr as a key:", string(b))

  b, _ = json.Marshal(both{})
  fmt.Println("\n  MarshalJSON wins over MarshalText:", string(b))
}
```

Output:

```
--- TextMarshaler used for MAP KEYS ---
  {"AT/Vienna":5,"DE/Berlin":10,"FR/Paris":20} <nil>
  unmarshal err: <nil> len: 3
  DE/Berlin: 10
  as a VALUE: "IT/Rome"

--- errors from TextMarshaler/TextUnmarshaler ---
  marshal: json: error calling MarshalText for type *main.Region: Region: empty country
  unmarshal: Region "bad-key": want COUNTRY/CITY
  netip.Addr as a key: {"192.0.2.1":"web"}

  MarshalJSON wins over MarshalText: "from-json"
```

`map[Region]int` marshals and **round-trips**, which is impossible without
`TextMarshaler` — a struct key is otherwise rejected outright. The keys come out sorted by
their text form, so the output is deterministic.

The precedence is `MarshalJSON` > `MarshalText` > reflection. Implementing `TextMarshaler`
is the cheaper choice for any type whose JSON form is a single string, because you get map
keys, `fmt` and `flag` compatibility for free; implement `MarshalJSON` only when the JSON
form is not a string. `net/netip.Addr`, `time.Time`, `net.IP` and `big.Int` all implement
it, which is why `map[netip.Addr]string` works out of the box.

## RawMessage: the polymorphic envelope

The most useful pattern in this lesson. A message with a `type` discriminator and a payload
whose shape depends on it cannot be expressed as one struct — but it can be decoded in two
passes, with `json.RawMessage` deferring the second.

```go
package main

import (
  "encoding/json"
  "fmt"
)

type Envelope struct {
  Type    string          `json:"type"`
  ID      string          `json:"id"`
  Payload json.RawMessage `json:"payload"`
}

type UserCreated struct {
  Email string `json:"email"`
  Plan  string `json:"plan"`
}

type OrderPlaced struct {
  OrderID string  `json:"orderId"`
  Total   float64 `json:"total"`
}

type Event interface{ isEvent() }

func (*UserCreated) isEvent() {}
func (*OrderPlaced) isEvent() {}

func decodeEvent(data []byte) (string, Event, error) {
  var env Envelope
  if err := json.Unmarshal(data, &env); err != nil {
    return "", nil, fmt.Errorf("envelope: %w", err)
  }

  var ev Event
  switch env.Type {
  case "user.created":
    ev = &UserCreated{}
  case "order.placed":
    ev = &OrderPlaced{}
  default:
    return env.ID, nil, fmt.Errorf("unknown event type %q", env.Type)
  }
  if err := json.Unmarshal(env.Payload, ev); err != nil {
    return env.ID, nil, fmt.Errorf("payload for %s: %w", env.Type, err)
  }
  return env.ID, ev, nil
}

func main() {
  docs := []string{
    `{"type":"user.created","id":"e1","payload":{"email":"a@b.c","plan":"pro"}}`,
    `{"type":"order.placed","id":"e2","payload":{"orderId":"o9","total":42.5}}`,
    `{"type":"nope","id":"e3","payload":{}}`,
    `{"type":"order.placed","id":"e4","payload":{"total":"not a number"}}`,
  }
  for _, d := range docs {
    id, ev, err := decodeEvent([]byte(d))
    if err != nil {
      fmt.Printf("  id=%-3s ERROR: %v\n", id, err)
      continue
    }
    fmt.Printf("  id=%-3s %-18T %+v\n", id, ev, ev)
  }

  fmt.Println("\n--- RawMessage passes JSON through untouched ---")
  raw := []byte(`{"type":"user.created","id":"e1","payload":{"b":2,"a":1,"nested":{"keep":"order"}}}`)
  var env Envelope
  json.Unmarshal(raw, &env)
  fmt.Printf("  payload bytes verbatim: %s\n", env.Payload)
  out, _ := json.Marshal(env)
  fmt.Printf("  re-marshalled:          %s\n", out)

  out, _ = json.Marshal(Envelope{Type: "t", ID: "i"})
  fmt.Println("\n  unset RawMessage:", string(out))

  fmt.Println("\n--- invalid JSON in a RawMessage fails on marshal ---")
  _, err := json.Marshal(Envelope{Type: "t", Payload: json.RawMessage(`{bad`)})
  fmt.Println("  err:", err)
}
```

Output:

```
  id=e1  *main.UserCreated  &{Email:a@b.c Plan:pro}
  id=e2  *main.OrderPlaced  &{OrderID:o9 Total:42.5}
  id=e3  ERROR: unknown event type "nope"
  id=e4  ERROR: payload for order.placed: json: cannot unmarshal string into Go struct field OrderPlaced.total of type float64

--- RawMessage passes JSON through untouched ---
  payload bytes verbatim: {"b":2,"a":1,"nested":{"keep":"order"}}
  re-marshalled:          {"type":"user.created","id":"e1","payload":{"b":2,"a":1,"nested":{"keep":"order"}}}

  unset RawMessage: {"type":"t","id":"i","payload":null}

--- invalid JSON in a RawMessage fails on marshal ---
  err: json: error calling MarshalJSON for type *jsontext.Value: invalid character 'b' looking for beginning of object key string
```

`json.RawMessage` is `[]byte` with `MarshalJSON`/`UnmarshalJSON` that copy verbatim. Two
visible consequences: the payload's original key order `{"b":2,"a":1}` survives a
decode-then-re-encode, which no `map[string]any` round trip would; and malformed bytes in
a `RawMessage` are caught on *marshal*, not on assignment.

The error messages justify the two passes: `payload for order.placed: ... field
OrderPlaced.total` names both the discriminator and the exact failing field, which a
single-pass `map[string]any` decode could not.

An unset `RawMessage` is nil and marshals as `null`; adding `omitempty` drops it instead,
because a `RawMessage` is a slice and `omitempty` understands empty slices.

Use this whenever a payload is opaque to the current layer: an API gateway forwarding
bodies, a webhook dispatcher, a job queue, a config file whose plugin sections each have
their own schema.

### omitempty on a nested struct, and interface fields

```go
package main

import (
  "bytes"
  "encoding/json"
  "fmt"
  "io"
)

type addr struct {
  Street string `json:"street,omitempty"`
  City   string `json:"city,omitempty"`
}

type withValue struct {
  Name string `json:"name"`
  Addr addr   `json:"addr,omitempty"`
}

type withPtr struct {
  Name string `json:"name"`
  Addr *addr  `json:"addr,omitempty"`
}

func decodeStrict[T any](r io.Reader) (T, error) {
  var v T
  dec := json.NewDecoder(r)
  dec.DisallowUnknownFields()
  if err := dec.Decode(&v); err != nil {
    return v, err
  }
  if dec.More() {
    return v, fmt.Errorf("unexpected trailing JSON")
  }
  return v, nil
}

func main() {
  b, _ := json.Marshal(withValue{Name: "a"})
  fmt.Println("value field:  ", string(b))
  b, _ = json.Marshal(withPtr{Name: "a"})
  fmt.Println("pointer field:", string(b))
  b, _ = json.Marshal(withPtr{Name: "a", Addr: &addr{City: "Berlin"}})
  fmt.Println("pointer set:  ", string(b))

  fmt.Println("\n--- a generic strict decoder ---")
  s, err := decodeStrict[withValue](bytes.NewReader([]byte(`{"name":"ok"}`)))
  fmt.Printf("  ok:       %+v err=%v\n", s, err)
  _, err = decodeStrict[withValue](bytes.NewReader([]byte(`{"name":"x","nope":1}`)))
  fmt.Println("  unknown: ", err)
  _, err = decodeStrict[withValue](bytes.NewReader([]byte(`{"name":"x"} {"name":"y"}`)))
  fmt.Println("  trailing:", err)

  fmt.Println("\n--- interface fields ---")
  var h struct {
    V any `json:"v"`
  }
  json.Unmarshal([]byte(`{"v":{"a":1}}`), &h)
  fmt.Printf("  any becomes %T: %v\n", h.V, h.V)

  var ih struct {
    V fmt.Stringer `json:"v"`
  }
  err = json.Unmarshal([]byte(`{"v":{"a":1}}`), &ih)
  fmt.Println("  non-empty interface:", err)
}
```

Output:

```
value field:   {"name":"a","addr":{}}
pointer field: {"name":"a"}
pointer set:   {"name":"a","addr":{"city":"Berlin"}}

--- a generic strict decoder ---
  ok:       {Name:ok Addr:{Street: City:}} err=<nil>
  unknown:  json: unknown field "nope"
  trailing: unexpected trailing JSON

--- interface fields ---
  any becomes map[string]interface {}: map[a:1]
  non-empty interface: json: cannot unmarshal object into Go struct field .v of type fmt.Stringer
```

`omitempty` on a nested **struct** field emits `"addr":{}` — it does not recurse to decide
emptiness, and a struct is never empty to it. The fixes are `*addr` (nil is omitted) or
`omitzero`. The pointer is usually better: it also makes "no address" and "an address with
all fields blank" distinguishable on the way in.

`decodeStrict[T]` is the helper worth copying into every service: generic over the target,
rejecting unknown fields *and* trailing content, which together give `Unmarshal`'s
strictness with `Decoder`'s streaming. Type inference means callers write
`decodeStrict[CreateUserRequest](r.Body)` with no reflection and no `any`.

An `any` field decodes to `map[string]any` (the 13.6 float64 trap applies). A non-empty
interface field fails with a clear error — the decoder cannot invent a concrete type and
there is no registry to teach it one. Use `json.RawMessage` and decode in a second pass.

## Performance: Decoder vs Unmarshal

```go
package main

import (
  "bytes"
  "encoding/json"
  "fmt"
  "strings"
  "testing"
)

type item struct {
  ID   int    `json:"id"`
  Name string `json:"name"`
}

var doc = func() []byte {
  var sb strings.Builder
  sb.WriteString("[")
  for i := 0; i < 10000; i++ {
    if i > 0 {
      sb.WriteString(",")
    }
    fmt.Fprintf(&sb, `{"id":%d,"name":"item-%d"}`, i, i)
  }
  sb.WriteString("]")
  return []byte(sb.String())
}()

func BenchmarkUnmarshalWhole(b *testing.B) {
  b.SetBytes(int64(len(doc)))
  for b.Loop() {
    var out []item
    if err := json.Unmarshal(doc, &out); err != nil {
      b.Fatal(err)
    }
  }
}

func BenchmarkDecoderStreaming(b *testing.B) {
  b.SetBytes(int64(len(doc)))
  for b.Loop() {
    dec := json.NewDecoder(bytes.NewReader(doc))
    dec.Token()
    n := 0
    for dec.More() {
      var it item
      if err := dec.Decode(&it); err != nil {
        b.Fatal(err)
      }
      n++
    }
    dec.Token()
    if n != 10000 {
      b.Fatal(n)
    }
  }
}
```

10,000 elements, about 300KB of JSON. `go test -bench . -benchmem`, machine-specific
(darwin/arm64, Apple M4 Pro). Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkUnmarshalWhole-12      	     796	   1262007 ns/op	 243.88 MB/s	 1129573 B/op	   10019 allocs/op
BenchmarkDecoderStreaming-12    	     642	   1870861 ns/op	 164.51 MB/s	  400946 B/op	   20012 allocs/op
```

The ranking is the opposite of the folklore. `json.Unmarshal` on bytes you already have is
the **fastest**. A third variant, `json.NewDecoder(bytes.NewReader(doc)).Decode(&out)` on
the same bytes, measured 1585347 ns/op and 2180027 B/op here — about 25% slower than
`Unmarshal` and allocating roughly twice as much, because it buffers while reading.

Streaming is the slowest per byte — about 1.5x `Unmarshal` — but allocates **400KB instead
of 2.2MB**, a 5x reduction, because the decoded elements are discarded rather than
accumulated into a slice.

So the rule is about memory, not speed: if the document already fits comfortably in a
`[]byte`, use `json.Unmarshal`. Use `Decoder` when the input is a stream you do not want
to buffer (an HTTP body, a multi-gigabyte file), or when you need `DisallowUnknownFields`
or per-element processing. Choosing `Decoder` over `Unmarshal` as a blanket "it's
streaming so it's faster" is simply wrong.

## encoding/json/v2

Go 1.27 ships `encoding/json/v2` and `encoding/json/jsontext` as importable packages — `v2`
for semantic marshaling with an options-based API (`MarshalWrite`, `UnmarshalRead`,
per-call options) and `jsontext` for the syntactic layer. Both compile and run on this
toolchain with no `GOEXPERIMENT` flag: `import "encoding/json/v2"` then
`json.Marshal(map[string]int{"a": 1})` prints `{"a":1}`, verified here.

The v1 `encoding/json` in this toolchain is itself reimplemented on top of them, which is
why 13.6's `map[bool]string` error mentions `jsontext` and why `json.RawMessage`'s `%T`
prints as `jsontext.Value`. Keep using `encoding/json` for now; read the v2 API before you
design a new serialization layer.

## Quick Reference

| Need | Call |
|---|---|
| Write JSON to a stream | `json.NewEncoder(w).Encode(v)` |
| Read JSON from a stream | `json.NewDecoder(r).Decode(&v)` |
| Reject unknown fields | `dec.DisallowUnknownFields()` |
| Reject trailing content too | check `dec.More()` after `Decode` |
| Read concatenated values | `for dec.More() { dec.Decode(&v) }` |
| Stream a huge array | `dec.Token()` then per-element `Decode` |
| Custom JSON for a type | `MarshalJSON` (value) / `UnmarshalJSON` (**pointer**) |
| Avoid recursion in `MarshalJSON` | `type alias T` inside the method |
| A string-valued type | `MarshalText`/`UnmarshalText` instead |
| A struct as a map key | `TextMarshaler` + `TextUnmarshaler` |
| Defer decoding a payload | `json.RawMessage` + a `switch` |
| Decode into an interface | `RawMessage`, then decode per type |
| Fastest whole-document decode | `json.Unmarshal` |
| Lowest-memory decode | `Decoder` + per-element `Decode` |

| Trap | Reality |
|---|---|
| `MarshalJSON` calling `json.Marshal(m)` | `fatal error: stack overflow` |
| `UnmarshalJSON` on a value receiver | compiles, never called |
| `Decode` on a request body | accepts trailing garbage; `Unmarshal` does not |
| `omitempty` on a nested struct | emits `{}`; use `*T` or `omitzero` |
| Decoding into a non-empty interface | always an error |
| Choosing `Decoder` "for speed" | it is slower than `Unmarshal`, but uses less memory |
| Invalid bytes in a `RawMessage` | the error surfaces at marshal time |

## Common Mistakes

**Calling `json.Marshal` on your own type inside its `MarshalJSON`.** That is infinite
recursion and a fatal stack overflow; define `type alias T` in the method and marshal that.

**Declaring `UnmarshalJSON` with a value receiver.** It satisfies nothing the decoder looks
for, so it is never called and your validation silently does not run.

**Using `Decoder.Decode` on an HTTP body without checking `More()`.** Trailing bytes after
the JSON value are accepted; add the check or use `json.Unmarshal` on the read body.

**Decoding a big array with `Decode(&slice)` to "stream" it.** That buffers the whole
slice; the element-at-a-time `Token()`+`Decode` loop is what bounds memory.

**Reaching for `map[string]any` for a polymorphic payload.** `json.RawMessage` keeps the
bytes exact, gives better errors, and costs nothing.

## Exercises

1. Write `type NullableString struct { Value string; Valid, Set bool }` whose
   `UnmarshalJSON` distinguishes absent, `null` and a string — the three states 13.6's
   pointer could not. Table-test all four inputs and round-trip with `MarshalJSON`.
2. Implement `TransformStream(r io.Reader, w io.Writer, f func(json.RawMessage) (json.RawMessage, error)) error`
   rewriting each element of a top-level array without buffering more than one. Verify
   constant memory with `-benchmem`.
3. Build an event dispatcher: `Register[T Event](name string)` plus
   `Decode(data []byte) (Event, error)` over a `map[string]func() Event`, replacing the
   hand-written switch with equally specific errors.
4. Write `UnmarshalJSON` for a type accepting **either** a string or an array of strings,
   deciding on the first byte. Table-test `"a"`, `["a","b"]`, `[]`, `null` and `5`.
5. Reproduce the recursion crash, then write a `go vet`-style check with `go/ast` that
   flags any `MarshalJSON` method whose body calls `json.Marshal` with the receiver.

---

[← 13.6 encoding/json: Marshal and Unmarshal](06-json.md) · [Course Index](../README.md) · [13.8 regexp →](08-regexp.md)
