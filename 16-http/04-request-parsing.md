[← 16.3 Handlers, HandlerFunc, and the Interface](03-handlers.md) · [Course Index](../README.md) · [16.5 JSON Requests and Responses →](05-json-api.md)

# 16.4 — Request Parsing: Path, Query, Headers, Body

## What You'll Learn

- `url.Values` for queries, and why a missing param and an empty param are both `""` — `Has` is the only way to tell them apart
- The header canonicalization rule: `Get` is case-insensitive, the underlying **map key is not**
- `r.Body` can be read exactly **once**, and the second read returns empty with a **nil error**
- `http.MaxBytesReader` and the typed `*http.MaxBytesError` — the one security measure every body-reading handler needs
- Multipart uploads end to end, including the memory-vs-disk threshold

## Coming From Another Language

| Other | Go |
|---|---|
| Express `req.query.tag` is a string *or* an array | `url.Values` is always `map[string][]string`; `Get` takes the first |
| Express `req.body` pre-parsed by `body-parser` | nothing is parsed until you ask; `r.Body` is a raw `io.ReadCloser` |
| Express `req.headers['x-api-key']` (lowercase keys) | `r.Header.Get` is case-insensitive, but the map key is `X-Api-Key` |
| Express `req.body` readable repeatedly | `r.Body` is a **stream**; read it twice and the second is empty |
| `express.json({limit:'1mb'})` by default | **no limit by default** — add `http.MaxBytesReader` yourself |
| `req.ip` with `trust proxy` | `r.RemoteAddr` is the real peer; `X-Forwarded-For` is unverified input |

## Path values

Covered in 16.2, recapped because it is step one of parsing. `r.PathValue(name)` returns the
wildcard's decoded value, or `""` if the pattern has no such wildcard.

```go
mux.HandleFunc("GET /users/{userID}/posts/{postID}", func(w http.ResponseWriter, r *http.Request) {
  userID := r.PathValue("userID")
  postID := r.PathValue("postID")
  // ...
})
```

`r.URL.Path` is the raw (cleaned, decoded) path if you need the whole thing. Remember from
16.2 that a path value can contain `/` (via `%2F`) and spaces, so validate before using one
as a filename or a key.

## Query parameters: url.Values

`r.URL.Query()` parses `RawQuery` into a `url.Values`, which is `map[string][]string` with
convenience methods. It **re-parses on every call**, so grab it once.

```go
package main

import (
  "fmt"
  "net/http/httptest"
  "net/url"
  "strconv"
)

func main() {
  req := httptest.NewRequest("GET",
    "/search?q=go+http&tag=a&tag=b&tag=&page=3&limit=abc&empty=&flag", nil)
  q := req.URL.Query()

  fmt.Printf("type: %T\n", q)
  fmt.Printf("q        = %q\n", q.Get("q"))
  fmt.Printf("tag      = %q  (Get returns only the first)\n", q.Get("tag"))
  fmt.Printf("tag all  = %q\n", q["tag"])
  fmt.Printf("missing  = %q  (absent and empty are both \"\")\n", q.Get("missing"))
  fmt.Printf("Has(empty)   = %v\n", q.Has("empty"))
  fmt.Printf("Has(missing) = %v\n", q.Has("missing"))
  fmt.Printf("Has(flag)    = %v  Get(flag)=%q\n", q.Has("flag"), q.Get("flag"))

  page, err := strconv.Atoi(q.Get("page"))
  fmt.Printf("page  -> %d, %v\n", page, err)
  limit, err := strconv.Atoi(q.Get("limit"))
  fmt.Printf("limit -> %d, %v\n", limit, err)
  none, err := strconv.Atoi(q.Get("missing"))
  fmt.Printf("missing -> %d, %v\n", none, err)

  q.Set("page", "4")
  q.Add("tag", "c")
  q.Del("limit")
  fmt.Printf("Encode() = %s\n", q.Encode())

  built := url.Values{}
  built.Set("name", "a b&c=d")
  built.Add("ids", "1")
  built.Add("ids", "2")
  fmt.Printf("built    = %s\n", built.Encode())

  fmt.Printf("RawQuery = %q\n", req.URL.RawQuery)
  fmt.Printf("URL.Path = %q  RequestURI = %q\n", req.URL.Path, req.RequestURI)
}
```

Output:

```
type: url.Values
q        = "go http"
tag      = "a"  (Get returns only the first)
tag all  = ["a" "b" ""]
missing  = ""  (absent and empty are both "")
Has(empty)   = true
Has(missing) = false
Has(flag)    = true  Get(flag)=""
page  -> 3, <nil>
limit -> 0, strconv.Atoi: parsing "abc": invalid syntax
missing -> 0, strconv.Atoi: parsing "": invalid syntax
Encode() = empty=&flag=&page=4&q=go+http&tag=a&tag=b&tag=&tag=c
built    = ids=1&ids=2&name=a+b%26c%3Dd
RawQuery = "q=go+http&tag=a&tag=b&tag=&page=3&limit=abc&empty=&flag"
URL.Path = "/search"  RequestURI = "/search?q=go+http&tag=a&tag=b&tag=&page=3&limit=abc&empty=&flag"
```

**`Get` collapses the absent and the empty case to `""`.** `Has` is the only way to
distinguish `?empty=` from no `empty` at all. That matters for a PATCH endpoint where
"field omitted" and "field cleared" mean different things
([13.6](../13-stdlib/06-json.md) covers the same problem for JSON).

**A bare `?flag` parses as `flag=""`**, so `Has("flag")` is `true` and `Get("flag")` is
`""`. Boolean query params therefore test with `Has`, not with truthiness.

**`+` decodes to a space in a query string**, which is why `q=go+http` gave `"go http"`.
In a *path*, `+` is literal — only `%20` is a space there. `Encode()` re-encodes with `+`
and sorts keys alphabetically, which makes it stable and therefore usable as a cache key.

**Parsing typed params needs explicit error handling.** Both `"abc"` and `""` give
`strconv.Atoi: ... invalid syntax` ([6.6](../06-strings/06-strconv.md)), and they need
different responses: `"abc"` is a 400, missing is usually a default. A small helper is worth
writing:

```go
func intParam(q url.Values, name string, def int) (int, error) {
  if !q.Has(name) || q.Get(name) == "" {
    return def, nil
  }
  n, err := strconv.Atoi(q.Get(name))
  if err != nil {
    return 0, fmt.Errorf("%s must be an integer", name)
  }
  return n, nil
}
```

Idiomatic: call `r.URL.Query()` once into a local, use `Has` for presence, and map every
parse failure to a 400 with the parameter name in the message (16.9).

## Headers, and the canonicalization rule

`http.Header` is `map[string][]string`. Every key stored through `Set`/`Add` and every key
read through `Get`/`Values` goes through `textproto.CanonicalMIMEHeaderKey` first — but a
**direct map access does not**.

```go
package main

import (
  "fmt"
  "mime"
  "net/http"
  "net/http/httptest"
  "sort"
  "strings"
)

func main() {
  req := httptest.NewRequest("POST", "/x", strings.NewReader(""))
  req.Header.Set("content-type", "application/json; charset=utf-8")
  req.Header.Add("x-custom-id", "one")
  req.Header.Add("X-CUSTOM-ID", "two")
  req.Header.Set("authorization", "Bearer abc123")

  fmt.Printf("Get(\"Content-Type\")  = %q\n", req.Header.Get("Content-Type"))
  fmt.Printf("Get(\"content-type\")  = %q\n", req.Header.Get("content-type"))
  fmt.Printf("Get(\"CONTENT-TYPE\")  = %q\n", req.Header.Get("CONTENT-TYPE"))
  fmt.Printf("Values(\"X-Custom-Id\")= %q\n", req.Header.Values("X-Custom-Id"))

  keys := make([]string, 0, len(req.Header))
  for k := range req.Header {
    keys = append(keys, k)
  }
  sort.Strings(keys)
  fmt.Printf("actual map keys      = %q\n", keys)

  fmt.Printf("direct map lookup req.Header[\"content-type\"] = %q (nil!)\n",
    req.Header["content-type"])
  fmt.Printf("direct map lookup req.Header[\"Content-Type\"] = %q\n",
    req.Header["Content-Type"])
  fmt.Printf("CanonicalHeaderKey(\"x-custom-id\") = %q\n",
    http.CanonicalHeaderKey("x-custom-id"))
  fmt.Printf("CanonicalHeaderKey(\"X-API-KEY\")   = %q\n",
    http.CanonicalHeaderKey("X-API-KEY"))

  mt, params, err := mime.ParseMediaType(req.Header.Get("Content-Type"))
  fmt.Printf("ParseMediaType -> %q %v %v\n", mt, params, err)
  _, _, err = mime.ParseMediaType("application/json; charset")
  fmt.Printf("ParseMediaType bad -> err=%v\n", err)
  mt2, p2, _ := mime.ParseMediaType("multipart/form-data; boundary=xyz")
  fmt.Printf("ParseMediaType mp  -> %q %v\n", mt2, p2)
}
```

Output:

```
Get("Content-Type")  = "application/json; charset=utf-8"
Get("content-type")  = "application/json; charset=utf-8"
Get("CONTENT-TYPE")  = "application/json; charset=utf-8"
Values("X-Custom-Id")= ["one" "two"]
actual map keys      = ["Authorization" "Content-Type" "X-Custom-Id"]
direct map lookup req.Header["content-type"] = [] (nil!)
direct map lookup req.Header["Content-Type"] = ["application/json; charset=utf-8"]
CanonicalHeaderKey("x-custom-id") = "X-Custom-Id"
CanonicalHeaderKey("X-API-KEY")   = "X-Api-Key"
ParseMediaType -> "application/json" map[charset:utf-8] <nil>
ParseMediaType bad -> err=mime: invalid media parameter
ParseMediaType mp  -> "multipart/form-data" map[boundary:xyz]
```

**`x-custom-id` and `X-CUSTOM-ID` landed in the same slot.** `Add` canonicalized both to
`X-Custom-Id`, so `Values` returns both values in insertion order.

**`req.Header["content-type"]` is nil.** This is the trap. Index the map directly and you
must spell the key in canonical form — and `Header` is a plain map, so there is no error, just
a nil slice that reads as absent. Never index `r.Header` directly; always use `Get`/`Values`.

**`X-API-KEY` canonicalizes to `X-Api-Key`, not `X-API-Key`.** Canonical form is
`Title-Case-Per-Hyphen-Segment`, which mangles acronyms. If you log header keys or compare
them against a list, canonicalize both sides with `http.CanonicalHeaderKey`.

**`mime.ParseMediaType` is how you read `Content-Type`.** Never `==` against
`"application/json"` — a real request sends `application/json; charset=utf-8`, which would
fail the comparison. `ParseMediaType` splits the type from its parameters and validates the
syntax, returning `mime: invalid media parameter` for malformed input (which is a 415).
For multipart it hands you the `boundary`.

Request headers worth knowing:

| Header | Read with | Note |
|---|---|---|
| `Content-Type` | `mime.ParseMediaType` | validate it; mismatch is **415**, not 400 |
| `Content-Length` | `r.ContentLength` | `-1` if unknown (chunked); do not trust it as a size limit |
| `Authorization` | `r.Header.Get` + `strings.CutPrefix(v, "Bearer ")` | or `r.BasicAuth()` |
| `Accept` | `r.Header.Get` | content negotiation; no stdlib parser |
| `Accept-Encoding` | `r.Header.Get` | check before gzipping (16.6) |
| `If-None-Match` / `If-Modified-Since` | `http.ServeContent` handles them | caching |
| `X-Forwarded-For` / `X-Real-IP` | `r.Header.Get` | **client-supplied**; see below |
| `Host` | `r.Host`, **not** `r.Header.Get("Host")` | Go moves it off the header map |
| `Cookie` | `r.Cookie` / `r.Cookies` | never parse it yourself |

## The body is a stream you can read once

`r.Body` is an `io.ReadCloser`. Reading it consumes it, and the second read returns zero
bytes with a **nil error** — there is no "already consumed" signal.

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

func doubleRead(w http.ResponseWriter, r *http.Request) {
  first, err1 := io.ReadAll(r.Body)
  second, err2 := io.ReadAll(r.Body)
  fmt.Fprintf(w, "first=%q err=%v\nsecond=%q err=%v\n", first, err1, second, err2)
}

func rewindable(w http.ResponseWriter, r *http.Request) {
  raw, _ := io.ReadAll(r.Body)
  r.Body = io.NopCloser(bytes.NewReader(raw))
  a, _ := io.ReadAll(r.Body)
  r.Body = io.NopCloser(bytes.NewReader(raw))
  b, _ := io.ReadAll(r.Body)
  fmt.Fprintf(w, "a=%q\nb=%q\n", a, b)
}

func bounded(w http.ResponseWriter, r *http.Request) {
  r.Body = http.MaxBytesReader(w, r.Body, 20)
  data, err := io.ReadAll(r.Body)
  fmt.Fprintf(w, "read %d bytes, err=%v\n", len(data), err)
  if err != nil {
    fmt.Fprintf(w, "err type: %T\n", err)
  }
}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("POST /double", doubleRead)
  mux.HandleFunc("POST /rewind", rewindable)
  mux.HandleFunc("POST /bounded", bounded)

  srv := httptest.NewServer(mux)
  defer srv.Close()

  post := func(path, body string) {
    resp, err := http.Post(srv.URL+path, "text/plain", strings.NewReader(body))
    if err != nil {
      fmt.Printf("%s -> client err %v\n", path, err)
      return
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("=== %s (%d byte body) %s\n%s", path, len(body), resp.Status, b)
  }

  post("/double", "hello body")
  post("/rewind", "hello body")
  post("/bounded", "short")
  post("/bounded", strings.Repeat("x", 100))
}
```

Output:

```
=== /double (10 byte body) 200 OK
first="hello body" err=<nil>
second="" err=<nil>
=== /rewind (10 byte body) 200 OK
a="hello body"
b="hello body"
=== /bounded (5 byte body) 200 OK
read 5 bytes, err=<nil>
=== /bounded (100 byte body) 200 OK
read 20 bytes, err=http: request body too large
err type: *http.MaxBytesError
```

**`second=""` with `err=<nil>`** is the whole problem. If a middleware reads the body for
logging and does not restore it, your handler decodes an empty body and gets "unexpected end
of JSON input" — with nothing pointing at the middleware. The `/rewind` handler shows the
only workaround: buffer the bytes once, then hand out fresh readers with
`io.NopCloser(bytes.NewReader(raw))`. Do that only when you genuinely need two passes,
because it defeats streaming and holds the whole body in memory.

**You do not have to close `r.Body`.** The server closes it for you after the handler
returns. (Closing it yourself is harmless but pointless.) The rule is the exact opposite for
a *response* body on the client side, where you must always close — 16.10.

**`http.MaxBytesReader(w, r.Body, n)`** is non-negotiable for any handler that reads a body.
Without it, `io.ReadAll(r.Body)` will happily allocate however many gigabytes a client sends:
a one-line denial of service. Note the behaviour precisely:

- It reads up to `n` bytes successfully, *then* errors — so `len(data)` is `20`, not `0`.
- The error is `*http.MaxBytesError`, so you can use `errors.As` to recognize it and return
  **413 Request Entity Too Large** rather than a generic 400 (16.5, 16.9).
- It takes `w` so it can tell the server not to attempt a keep-alive read of the rest.

`http.MaxBytesHandler(h, n)` wraps the whole thing as middleware if you prefer it global:

```go
mux.Handle("POST /items", http.MaxBytesHandler(itemHandler, 1<<20))
```

## Form bodies, and the query/body precedence

`r.ParseForm` populates two maps, and the difference between them is the detail people get
wrong.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

func formHandler(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "before ParseForm: r.Form=%v r.PostForm=%v\n", r.Form, r.PostForm)
  if err := r.ParseForm(); err != nil {
    fmt.Fprintf(w, "ParseForm err: %v\n", err)
    return
  }
  fmt.Fprintf(w, "r.Form     = %v\n", r.Form)
  fmt.Fprintf(w, "r.PostForm = %v\n", r.PostForm)
  fmt.Fprintf(w, "FormValue(\"name\")     = %q  (body wins over query)\n", r.FormValue("name"))
  fmt.Fprintf(w, "PostFormValue(\"name\") = %q  (body only)\n", r.PostFormValue("name"))
  fmt.Fprintf(w, "FormValue(\"only_q\")   = %q\n", r.FormValue("only_q"))
  fmt.Fprintf(w, "PostFormValue(\"only_q\")= %q  (query invisible here)\n", r.PostFormValue("only_q"))
  fmt.Fprintf(w, "r.Form[\"name\"]        = %q  (query appended after body)\n", r.Form["name"])
}

func main() {
  srv := httptest.NewServer(http.HandlerFunc(formHandler))
  defer srv.Close()

  resp, err := http.Post(srv.URL+"/submit?name=from-query&only_q=qv",
    "application/x-www-form-urlencoded",
    strings.NewReader("name=from-body&extra=eb"))
  if err != nil {
    panic(err)
  }
  b, _ := io.ReadAll(resp.Body)
  resp.Body.Close()
  fmt.Print(string(b))
}
```

Output:

```
before ParseForm: r.Form=map[] r.PostForm=map[]
r.Form     = map[extra:[eb] name:[from-body from-query] only_q:[qv]]
r.PostForm = map[extra:[eb] name:[from-body]]
FormValue("name")     = "from-body"  (body wins over query)
PostFormValue("name") = "from-body"  (body only)
FormValue("only_q")   = "qv"
PostFormValue("only_q")= ""  (query invisible here)
r.Form["name"]        = ["from-body" "from-query"]  (query appended after body)
```

| | Contains | Body wins? |
|---|---|---|
| `r.PostForm` | body only | n/a |
| `r.Form` | body **then** query, body values first | yes, for `Get`/`FormValue` |
| `r.FormValue(k)` | first value from `r.Form` | yes |
| `r.PostFormValue(k)` | first value from `r.PostForm` | body only |

`r.Form` and `r.PostForm` are both empty until `ParseForm` runs. `FormValue` and
`PostFormValue` call it implicitly and **swallow the error**, which is why the explicit
`r.ParseForm()` call is better practice: a malformed body or an oversized form is an error
you want to see and turn into a 400.

`ParseForm` only parses the body when `Content-Type` is
`application/x-www-form-urlencoded`. For any other type — including JSON — it parses the
query only and leaves the body untouched, which is exactly what you want.

Security note: because `r.Form` merges query and body, an endpoint that reads `FormValue`
can be driven by query parameters even when you intended a body-only API. For a JSON API
this never comes up; for a form endpoint handling state changes, prefer `PostFormValue`.

## Multipart file uploads

`r.ParseMultipartForm(maxMemory)` parses the whole multipart body: text fields into
`r.MultipartForm.Value`, files into `r.MultipartForm.File`. Files up to `maxMemory` bytes
in total stay in RAM; the rest spill to temporary files on disk.

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "mime/multipart"
  "net/http"
  "net/http/httptest"
  "os"
  "path/filepath"
  "sort"
)

func upload(dest string) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {
    if err := r.ParseMultipartForm(1 << 10); err != nil {
      http.Error(w, "bad multipart: "+err.Error(), http.StatusBadRequest)
      return
    }
    defer r.MultipartForm.RemoveAll()

    fmt.Fprintf(w, "text fields: %v\n", r.MultipartForm.Value)
    names := make([]string, 0, len(r.MultipartForm.File))
    for k := range r.MultipartForm.File {
      names = append(names, k)
    }
    sort.Strings(names)
    fmt.Fprintf(w, "file fields: %v\n", names)

    file, hdr, err := r.FormFile("document")
    if err != nil {
      http.Error(w, err.Error(), http.StatusBadRequest)
      return
    }
    defer file.Close()

    fmt.Fprintf(w, "FileHeader: Filename=%q Size=%d Content-Type=%q\n",
      hdr.Filename, hdr.Size, hdr.Header.Get("Content-Type"))

    out, err := os.Create(filepath.Join(dest, filepath.Base(hdr.Filename)))
    if err != nil {
      http.Error(w, err.Error(), 500)
      return
    }
    defer out.Close()
    n, err := io.Copy(out, file)
    fmt.Fprintf(w, "wrote %d bytes to disk, err=%v\n", n, err)
    fmt.Fprintf(w, "FormValue(\"title\") = %q\n", r.FormValue("title"))
  }
}

func buildBody(size int) (string, io.Reader) {
  var buf bytes.Buffer
  mw := multipart.NewWriter(&buf)
  mw.WriteField("title", "quarterly report")
  fw, _ := mw.CreateFormFile("document", "report.txt")
  fw.Write(bytes.Repeat([]byte("D"), size))
  mw.Close()
  return mw.FormDataContentType(), &buf
}

func main() {
  dest, _ := os.MkdirTemp("", "uploads")
  defer os.RemoveAll(dest)

  srv := httptest.NewServer(upload(dest))
  defer srv.Close()

  for _, size := range []int{64, 4096} {
    ct, body := buildBody(size)
    resp, err := http.Post(srv.URL+"/upload", ct, body)
    if err != nil {
      panic(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("=== upload of %d bytes (maxMemory=1024) -> %s\n%s", size, resp.Status, b)
  }
  entries, _ := os.ReadDir(dest)
  for _, e := range entries {
    info, _ := e.Info()
    fmt.Printf("on disk: %s (%d bytes)\n", e.Name(), info.Size())
  }
}
```

Output:

```
=== upload of 64 bytes (maxMemory=1024) -> 200 OK
text fields: map[title:[quarterly report]]
file fields: [document]
FileHeader: Filename="report.txt" Size=64 Content-Type="application/octet-stream"
wrote 64 bytes to disk, err=<nil>
FormValue("title") = "quarterly report"
=== upload of 4096 bytes (maxMemory=1024) -> 200 OK
text fields: map[title:[quarterly report]]
file fields: [document]
FileHeader: Filename="report.txt" Size=4096 Content-Type="application/octet-stream"
wrote 4096 bytes to disk, err=<nil>
FormValue("title") = "quarterly report"
```

Plus, after both uploads:

```
on disk: report.txt (4096 bytes)
```

The two uploads straddle `maxMemory = 1024`, and the handler code is identical for both —
the 64-byte file was buffered in RAM and the 4096-byte one was written to a temp file, and
`multipart.File` is an `io.Reader` either way. That abstraction is the whole point.

Five things to get right:

**`defer r.MultipartForm.RemoveAll()`.** Spilled temp files are *not* cleaned up
automatically on every path. Forget this and a busy upload endpoint fills `/tmp`.

**`hdr.Filename` is attacker-controlled.** It can be `../../etc/passwd` or
`C:\Windows\x`. `filepath.Base` is the minimum; generating your own name from a UUID and
keeping the original only as metadata is better.

**`hdr.Header.Get("Content-Type")` is attacker-controlled too.** Here it is
`application/octet-stream` because `CreateFormFile` sets that by default. Never trust it to
decide how to process a file; sniff the real bytes with `http.DetectContentType`.

**`maxMemory` is not a total size limit.** It only decides where bytes land. Combine it with
`http.MaxBytesReader` on `r.Body` *before* calling `ParseMultipartForm` to cap the request.

**`r.FormFile(name)` is the shortcut** when you want one file: it calls
`ParseMultipartForm(32 << 20)` for you if needed and returns the first file for that field.
Use `r.MultipartForm.File[name]` (a `[]*multipart.FileHeader`) for a multi-file field.

For truly large uploads, skip `ParseMultipartForm` entirely and stream with
`r.MultipartReader()`, which gives you one part at a time and never buffers.

## RemoteAddr, X-Forwarded-For, cookies, and the rest of *http.Request

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

func meta(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "Method      = %q\n", r.Method)
  fmt.Fprintf(w, "URL.Path    = %q\n", r.URL.Path)
  fmt.Fprintf(w, "URL.RawQuery= %q\n", r.URL.RawQuery)
  fmt.Fprintf(w, "RequestURI  = %q\n", r.RequestURI)
  fmt.Fprintf(w, "Proto       = %q (Major=%d Minor=%d)\n", r.Proto, r.ProtoMajor, r.ProtoMinor)
  fmt.Fprintf(w, "Host        = %q\n", r.Host)
  fmt.Fprintf(w, "URL.Host    = %q (empty on the server side)\n", r.URL.Host)
  fmt.Fprintf(w, "ContentLength = %d\n", r.ContentLength)
  fmt.Fprintf(w, "TLS         = %v (nil means plain HTTP)\n", r.TLS == nil)
  fmt.Fprintf(w, "RemoteAddr  = %q (host:port, the real peer)\n", strings.Split(r.RemoteAddr, ":")[0]+":<port>")
  fmt.Fprintf(w, "X-Forwarded-For = %q (CLIENT-SUPPLIED, do not trust)\n",
    r.Header.Get("X-Forwarded-For"))

  c, err := r.Cookie("session")
  fmt.Fprintf(w, "Cookie(session) = %v, err=%v\n", c, err)
  _, err = r.Cookie("nope")
  fmt.Fprintf(w, "Cookie(nope) err = %v (errors.Is ErrNoCookie: %v)\n", err, err == http.ErrNoCookie)
  for _, ck := range r.Cookies() {
    fmt.Fprintf(w, "  cookie %s=%s\n", ck.Name, ck.Value)
  }

  http.SetCookie(w, &http.Cookie{
    Name:     "session",
    Value:    "new-token",
    Path:     "/",
    MaxAge:   3600,
    HttpOnly: true,
    Secure:   true,
    SameSite: http.SameSiteLaxMode,
  })
  fmt.Fprintf(w, "Set-Cookie sent = %q\n", w.Header().Get("Set-Cookie"))
}

func main() {
  srv := httptest.NewServer(http.HandlerFunc(meta))
  defer srv.Close()

  req, _ := http.NewRequest("PUT", srv.URL+"/a/b?x=1", strings.NewReader("body!"))
  req.Header.Set("X-Forwarded-For", "203.0.113.9, 70.41.3.18")
  req.AddCookie(&http.Cookie{Name: "session", Value: "old-token"})
  req.AddCookie(&http.Cookie{Name: "theme", Value: "dark"})
  resp, err := http.DefaultClient.Do(req)
  if err != nil {
    panic(err)
  }
  b, _ := io.ReadAll(resp.Body)
  resp.Body.Close()
  fmt.Print(string(b))
}
```

Output (the `Host` port is kernel-assigned and differs every run; `RemoteAddr`'s port is
masked in the handler for the same reason):

```
Method      = "PUT"
URL.Path    = "/a/b"
URL.RawQuery= "x=1"
RequestURI  = "/a/b?x=1"
Proto       = "HTTP/1.1" (Major=1 Minor=1)
Host        = "127.0.0.1:56740"
URL.Host    = "" (empty on the server side)
ContentLength = 5
TLS         = true (nil means plain HTTP)
RemoteAddr  = "127.0.0.1:<port>" (host:port, the real peer)
X-Forwarded-For = "203.0.113.9, 70.41.3.18" (CLIENT-SUPPLIED, do not trust)
Cookie(session) = session=old-token, err=<nil>
Cookie(nope) err = http: named cookie not present (errors.Is ErrNoCookie: true)
  cookie session=old-token
  cookie theme=dark
Set-Cookie sent = "session=new-token; Path=/; Max-Age=3600; HttpOnly; Secure; SameSite=Lax"
```

**`r.URL.Host` is empty on the server.** Server-side `r.URL` holds only the path and query —
the authority lives in `r.Host`. On the *client* side `r.URL` is a full absolute URL. Same
type, different population rules.

**`X-Forwarded-For` is just a header a client sent.** Nothing validated it. Anyone can
`curl -H 'X-Forwarded-For: 1.2.3.4'` and your rate limiter, audit log, or geo-block now
believes them. The only safe rule: trust it **only** if the request came from a proxy you
control, and then read the correct position in the list (your proxy appends, so the
rightmost entries are the trusted ones). If there is no proxy, use `r.RemoteAddr` and ignore
the header entirely. Split `r.RemoteAddr` with `net.SplitHostPort`, not `strings.Split` —
IPv6 addresses contain colons.

**`r.Cookie` returns `http.ErrNoCookie`**, a sentinel, so test it with
`errors.Is(err, http.ErrNoCookie)` ([8.3](../08-errors/03-is-as-unwrap.md)).

**The `http.Cookie` flags are security-critical.** `HttpOnly` blocks JavaScript access
(XSS mitigation); `Secure` sends it over HTTPS only; `SameSite=Lax` is the sane CSRF
default (`Strict` breaks inbound links, `None` requires `Secure`). The serialized form in
the output shows exactly which attributes made it onto the wire. Note `MaxAge: 0` means
"session cookie, no expiry attribute" and `MaxAge: -1` means "delete now" — a real
three-state field.

## Quick Reference

| Path | Call | Note |
|---|---|---|
| wildcard value | `r.PathValue("id")` | `""` if absent; decoded, may contain `/` |
| full path | `r.URL.Path` | cleaned and decoded |
| raw path | `r.URL.RawPath`, `r.RequestURI` | pre-decoding; `RequestURI` includes the query |

| Query | Call | Note |
|---|---|---|
| parse | `r.URL.Query()` | re-parses each call — store it |
| first value | `v.Get(k)` | `""` for both absent and empty |
| all values | `v["k"]` or the map directly | `[]string` |
| presence | `v.Has(k)` | the only way to tell absent from empty |
| mutate | `v.Set/Add/Del` | then `v.Encode()` (sorted, stable) |

| Header | Call | Note |
|---|---|---|
| read | `r.Header.Get(k)` | case-insensitive; first value |
| read all | `r.Header.Values(k)` | case-insensitive |
| write | `h.Set(k,v)` / `h.Add(k,v)` | canonicalizes the key |
| canonical form | `http.CanonicalHeaderKey(k)` | `X-API-KEY` → `X-Api-Key` |
| content type | `mime.ParseMediaType(v)` | never compare with `==` |
| basic auth | `r.BasicAuth() (user, pass string, ok bool)` | |

| Body | Call | Note |
|---|---|---|
| read all | `io.ReadAll(r.Body)` | once only; bound it first |
| bound | `http.MaxBytesReader(w, r.Body, n)` | error is `*http.MaxBytesError` → 413 |
| bound as middleware | `http.MaxBytesHandler(h, n)` | |
| close | not your job | the server closes it |
| form | `r.ParseForm()` then `r.PostForm` | urlencoded bodies only |
| multipart | `r.ParseMultipartForm(maxMemory)` | `defer r.MultipartForm.RemoveAll()` |
| one file | `r.FormFile(name)` | returns `(multipart.File, *multipart.FileHeader, error)` |
| streaming multipart | `r.MultipartReader()` | no buffering at all |

| Request field | Holds |
|---|---|
| `r.Method` | `"GET"`, `"POST"`, … |
| `r.Host` | the authority; **not** in `r.Header` |
| `r.Proto` / `ProtoMajor` / `ProtoMinor` | `"HTTP/1.1"`, `1`, `1` |
| `r.ContentLength` | `-1` when unknown |
| `r.TLS` | `*tls.ConnectionState`, `nil` on plain HTTP |
| `r.RemoteAddr` | `"host:port"` of the real peer; split with `net.SplitHostPort` |
| `r.RequestURI` | the raw request-target, path + query |
| `r.Context()` | the request context (16.12) |

## Common Mistakes

**Indexing `r.Header` directly with a lowercase key.** The map key is canonical
(`X-Api-Key`), so you silently get a nil slice; use `Get` or `Values`.

**Comparing `Content-Type` with `==`.** Real requests include `; charset=utf-8`, so the
comparison fails on valid input; parse it with `mime.ParseMediaType`.

**Reading `r.Body` twice.** The second read returns empty with a nil error, so the bug
surfaces as a confusing decode failure far from the cause; buffer the bytes if you really
need two passes.

**Reading a body without `http.MaxBytesReader`.** There is no default limit, so a single
request can exhaust your memory; cap it and map `*http.MaxBytesError` to 413.

**Trusting `X-Forwarded-For` or `hdr.Filename`.** Both are plain client input — one forges
the client IP, the other is a path traversal; validate against a proxy allowlist and run the
filename through `filepath.Base`.

**Forgetting `r.MultipartForm.RemoveAll()`.** Uploads over `maxMemory` leave temp files
behind and fill the disk; defer the cleanup right after a successful parse.

**Using `FormValue` and expecting body-only semantics.** It merges the query in, so a
query parameter can override your form field; use `PostFormValue` for body-only reads.

## Exercises

1. Write a `parseListOptions(r *http.Request) (ListOptions, error)` that reads `page`
   (default 1), `limit` (default 20, max 100), `sort` (one of `created`, `name`), and a
   repeated `tag` param. Return a distinct error message per failure and verify each with a
   table-driven test over `httptest.NewRequest`.
2. Write a middleware that logs the request body and still lets the handler read it. Prove
   with a test that the handler sees the full body, then delete the restore step and capture
   the exact error the handler's JSON decode produces.
3. Build a handler with a 1 KiB `MaxBytesReader`. Return 413 with a JSON body when
   `errors.As` matches `*http.MaxBytesError` and 400 otherwise. Drive it with a 500-byte
   body, a 2 KiB body, and a malformed body, and show all three real responses.
4. Write an upload handler that accepts only `image/png` by sniffing the first 512 bytes
   with `http.DetectContentType` (ignoring the declared `Content-Type`), caps the request at
   5 MiB, and stores files under a generated name. Test it with a real PNG header and with a
   text file renamed `evil.png`.
5. Write `clientIP(r *http.Request, trustedProxies []netip.Prefix) netip.Addr` that uses
   `X-Forwarded-For` only when `r.RemoteAddr` is inside a trusted prefix, and returns the
   rightmost untrusted entry. Test with a spoofed header from an untrusted peer and from a
   trusted one.

---

[← 16.3 Handlers, HandlerFunc, and the Interface](03-handlers.md) · [Course Index](../README.md) · [16.5 JSON Requests and Responses →](05-json-api.md)
