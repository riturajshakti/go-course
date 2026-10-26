[← 16.10 The net/http Client](10-http-client.md) · [Course Index](../README.md) · [16.12 Context Propagation Through Handlers →](12-context-propagation.md)

# 16.11 — Testing Handlers with httptest

## What You'll Learn

- `httptest.NewRequest` + `NewRecorder` for the fast path (no network), `NewServer`/`NewTLSServer` for when you need a real round trip
- Assert on **decoded structs**, never on JSON strings — key order makes string comparison a flake waiting to happen
- `SetPathValue` and chi's route context for testing a handler without its router
- The **double-server** pattern: test a handler that calls an external API by pointing it at a second `httptest` server
- Injecting a fake clock and ID generator so responses are byte-exact reproducible

## Coming From Another Language

| Other | Go |
|---|---|
| `supertest(app).get('/x').expect(200)` | `h.ServeHTTP(rec, req)` then check `rec.Code` — no framework needed |
| Jest `nock` to intercept outbound HTTP | a second `httptest.Server`, or a fake `RoundTripper` (16.10) |
| `pytest` fixtures with `TestClient` | a helper returning `(http.Handler, *Store)` and `t.Cleanup` |
| `expect(res.body).toEqual({...})` deep-compares | `json.Unmarshal` into a struct, then compare fields |
| Mock libraries for the HTTP layer | `httptest` *is* the real server; nothing is mocked |

`net/http/httptest` is 400 lines of standard library and replaces an entire category of
testing dependency. Everything in this module's output was produced with it.

## NewRequest and NewRecorder: the fast path

```go
func TestNewRequestVsRecorder(t *testing.T) {
  req := httptest.NewRequest("POST", "/items/42?x=1", nil)
  t.Logf("Method=%q URL.Path=%q RawQuery=%q", req.Method, req.URL.Path, req.URL.RawQuery)
  t.Logf("Host=%q RemoteAddr=%q Proto=%q", req.Host, req.RemoteAddr, req.Proto)
  t.Logf("RequestURI=%q ContentLength=%d", req.RequestURI, req.ContentLength)

  full := httptest.NewRequest("GET", "http://api.example.com/v1/x", nil)
  t.Logf("absolute URL -> Host=%q URL.Host=%q", full.Host, full.URL.Host)

  rec := httptest.NewRecorder()
  t.Logf("fresh recorder: Code=%d Body.Len=%d Flushed=%v HeaderLen=%d",
    rec.Code, rec.Body.Len(), rec.Flushed, len(rec.Header()))

  h := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("X-Custom", "yes")
    w.WriteHeader(http.StatusCreated)
    fmt.Fprint(w, `{"ok":true}`)
  })
  h.ServeHTTP(rec, req)
  t.Logf("after handler: Code=%d Body=%q X-Custom=%q",
    rec.Code, rec.Body.String(), rec.Header().Get("X-Custom"))

  res := rec.Result()
  defer res.Body.Close()
  t.Logf("Result(): Status=%q StatusCode=%d ContentLength=%d",
    res.Status, res.StatusCode, res.ContentLength)
}
```

Real `go test -v` output:

```
=== RUN   TestNewRequestVsRecorder
    basics_test.go:12: Method="POST" URL.Path="/items/42" RawQuery="x=1"
    basics_test.go:13: Host="example.com" RemoteAddr="192.0.2.1:1234" Proto="HTTP/1.1"
    basics_test.go:14: RequestURI="/items/42?x=1" ContentLength=0
    basics_test.go:17: absolute URL -> Host="api.example.com" URL.Host="api.example.com"
    basics_test.go:20: fresh recorder: Code=200 Body.Len=0 Flushed=false HeaderLen=0
    basics_test.go:29: after handler: Code=201 Body="{\"ok\":true}" X-Custom="yes"
    basics_test.go:34: Result(): Status="201 Created" StatusCode=201 ContentLength=-1
--- PASS: TestNewRequestVsRecorder (0.00s)
```

**`httptest.NewRequest` vs `http.NewRequest`:** the test version fills in the *server-side*
fields — `Host: "example.com"`, `RemoteAddr: "192.0.2.1:1234"` (a documentation-reserved
address), `Proto: "HTTP/1.1"`, and `RequestURI`. `http.NewRequest` builds a **client** request,
where `RequestURI` must be empty and `RemoteAddr` is meaningless; passing one to a handler
produces confusing failures. `httptest.NewRequest` also **panics** on a bad URL instead of
returning an error, which is what you want in a test — no `if err != nil` noise.

**`rec.Code` starts at 200.** A fresh recorder reports 200 before anything runs, which mirrors
the implicit-200 rule (16.1) but means `rec.Code == 200` does **not** prove your handler
wrote a status. Assert on the body too, or check `len(rec.Header())`.

**`rec.Result()` builds an `*http.Response`** for code that wants one (a client-side parser, a
shared assertion helper). Note `ContentLength: -1` — the recorder does not compute one. Close
its body, as `go vet`'s lostcancel-adjacent checks and good hygiene both suggest.

| `*httptest.ResponseRecorder` | Holds |
|---|---|
| `Code int` | the status; **200 before the handler runs** |
| `Body *bytes.Buffer` | everything written; `.String()` / `.Bytes()` |
| `HeaderMap` | deprecated — use `Header()` |
| `Header() http.Header` | the response headers |
| `Flushed bool` | whether `Flush()` was called |
| `Result() *http.Response` | a real response value |
| `WriteHeader`, `Write`, `Flush` | it implements `http.ResponseWriter` and `http.Flusher` |

The recorder does **not** implement `http.Hijacker`, and `http.ResponseController`'s deadline
methods return `"feature not supported"` against it (16.3) — the one real difference from a
live server.

## Path values: SetPathValue vs going through the router

A handler reading `r.PathValue("id")` gets `""` when called directly, because nothing routed
the request.

```go
func TestSetPathValue(t *testing.T) {
  h := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "id=%q", r.PathValue("id"))
  })

  bare := httptest.NewRequest("GET", "/items/42", nil)
  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, bare)
  t.Logf("without SetPathValue: %s", rec.Body.String())

  withVal := httptest.NewRequest("GET", "/items/42", nil)
  withVal.SetPathValue("id", "42")
  rec = httptest.NewRecorder()
  h.ServeHTTP(rec, withVal)
  t.Logf("with SetPathValue:    %s", rec.Body.String())

  mux := http.NewServeMux()
  mux.Handle("GET /items/{id}", h)
  rec = httptest.NewRecorder()
  mux.ServeHTTP(rec, httptest.NewRequest("GET", "/items/42", nil))
  t.Logf("through the mux:      %s", rec.Body.String())
}
```

Output:

```
=== RUN   TestSetPathValue
    basics_test.go:46: without SetPathValue: id=""
    basics_test.go:52: with SetPathValue:    id="42"
    basics_test.go:58: through the mux:      id="42"
--- PASS: TestSetPathValue (0.00s)
```

`(*http.Request).SetPathValue` (Go 1.22+, confirmed working on 1.27.1) is for testing a
handler in isolation. The third form — serving through the real mux — is usually better,
because it tests your route patterns too and cannot drift from them. Use `SetPathValue` when
the handler is the unit you want and the router is tested elsewhere.

For chi, the equivalent is building a route context by hand:

```go
rctx := chi.NewRouteContext()
rctx.URLParams.Add("userID", "7")
rctx.URLParams.Add("postID", "9")
withCtx := bare.WithContext(context.WithValue(bare.Context(), chi.RouteCtxKey, rctx))
```

Output:

```
=== RUN   TestChiRouteContext
    extra_test.go:68: without a route context: userID="" postID=""
    extra_test.go:76: with a route context:    userID="7" postID="9"
--- PASS: TestChiRouteContext (0.00s)
```

## NewServer, NewUnstartedServer, NewTLSServer

Use a real server when the thing under test needs a network: an HTTP **client** (16.10), a
streaming handler, TLS behaviour, or an end-to-end smoke test.

```go
func TestServerVariants(t *testing.T) {
  h := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "tls=%v proto=%s", r.TLS != nil, r.Proto)
  })

  plain := httptest.NewServer(h)
  defer plain.Close()
  resp, err := plain.Client().Get(plain.URL)
  // ... read and log

  tlsSrv := httptest.NewTLSServer(h)
  defer tlsSrv.Close()
  resp, err = tlsSrv.Client().Get(tlsSrv.URL)
  // ... read and log

  _, err = http.Get(tlsSrv.URL)
  t.Logf("default client against the TLS server errors: %v", err != nil)

  un := httptest.NewUnstartedServer(h)
  un.Config.ReadHeaderTimeout = 1
  un.Start()
  defer un.Close()
}
```

Output (the stray `log` line is the TLS server reporting the rejected handshake from the
default client — expected, and proof the assertion is real):

```
=== RUN   TestServerVariants
    basics_test.go:75: NewServer:    scheme=http body="tls=false proto=HTTP/1.1"
    basics_test.go:85: NewTLSServer: scheme=https body="tls=true proto=HTTP/1.1" (server.Client() trusts the cert)
    basics_test.go:89: default client against the TLS server errors: true
    basics_test.go:95: NewUnstartedServer: configured before Start, URL=http
2026/10/26 13:57:57 http: TLS handshake error from 127.0.0.1:61597: remote error: tls: bad certificate
--- PASS: TestServerVariants (0.02s)
```

**`srv.Client()` is the important method.** It returns a client preconfigured for that
server — and for a TLS server, one whose `RootCAs` trusts the server's self-signed
certificate. The default client **cannot** talk to `NewTLSServer`, as the output proves: you
would otherwise need `InsecureSkipVerify: true`, which is a habit you do not want in your
codebase.

**`srv.URL` has a kernel-assigned port** and so differs every run. Never hardcode it; always
read it from the server. Pass it to the thing under test as a `baseURL` field (16.10).

**`NewUnstartedServer` + `Start()`** is how you configure `srv.Config` (the `*http.Server`) or
`srv.Listener` before accepting. That is where you set `ErrorLog` to capture server logs — as
every panic example in this module does — or `StartTLS()` for a custom TLS config.

**`defer srv.Close()`** is mandatory, or the listener and its goroutines outlive the test.
`t.Cleanup(srv.Close)` is the nicer form, especially in a helper
([14.3](../14-testing/03-fixtures-helpers.md)).

| Constructor | Gives |
|---|---|
| `httptest.NewServer(h)` | started HTTP server on 127.0.0.1, random port |
| `httptest.NewTLSServer(h)` | started HTTPS server with a self-signed cert |
| `httptest.NewUnstartedServer(h)` | not listening yet; configure then `Start()`/`StartTLS()` |
| `srv.URL` | `http://127.0.0.1:PORT` — varies every run |
| `srv.Client()` | a client that trusts this server |
| `srv.Config` | the `*http.Server` — set `ErrorLog`, timeouts |
| `srv.Listener` | the `net.Listener` |
| `srv.CloseClientConnections()` | force-drop open connections (for testing a client's error path) |

## Assert on structs, not JSON strings

```go
func TestJSONComparisonNotStringComparison(t *testing.T) {
  h, _ := newTestAPI(t)
  rec := httptest.NewRecorder()
  req := httptest.NewRequest(http.MethodPost, "/items",
    strings.NewReader(`{"price":9.99,"name":"widget"}`))
  h.ServeHTTP(rec, req)

  raw := rec.Body.String()
  t.Logf("raw body: %s", raw)
  if raw == `{"name":"widget","id":1,"price":9.99,"note":null}` {
    t.Log("string comparison happened to match")
  } else {
    t.Logf("string comparison FAILS even though the JSON is correct: key order differs")
  }

  var got Item
  decodeInto(t, rec, &got)
  want := Item{ID: 1, Name: "widget", Price: 9.99}
  if got.ID != want.ID || got.Name != want.Name || got.Price != want.Price {
    t.Errorf("decoded = %+v, want %+v", got, want)
  }
  t.Log("struct comparison is order-independent and passes")
}
```

Output:

```
=== RUN   TestJSONComparisonNotStringComparison
    api_test.go:168: raw body: {"id":1,"name":"widget","price":9.99,"note":null}
    api_test.go:172: string comparison FAILS even though the JSON is correct: key order differs
    api_test.go:181: struct comparison is order-independent and passes
--- PASS: TestJSONComparisonNotStringComparison (0.00s)
```

Four reasons string comparison is the wrong tool, all demonstrable: key order follows
**struct field order**, so reordering fields in your type breaks tests that have nothing to do
with the change; `json.NewEncoder` appends a `\n` and `json.Marshal` does not (16.5), so the
same data has two valid encodings; whitespace and float formatting (`11` vs `11.0`) vary; and
a mismatch gives you a wall of unreadable diff instead of a named field.

Unmarshal into the type you expect and compare fields. For deeply nested values,
`github.com/google/go-cmp`'s `cmp.Diff(want, got)` gives a readable field-by-field diff and
is the one testing dependency worth adding ([14.4](../14-testing/04-testify.md) covers the
assertion-library question).

Unmarshal into `map[string]any` instead when you want to assert that a key is **absent** —
that is how you test `omitempty` behaviour, which a struct cannot distinguish from a zero
value.

## A complete table-driven test file

The CRUD API from 16.5, tested properly. Note the helper returning both the handler and the
store, so a test can seed state without going through HTTP.

```go
package main

import (
  "encoding/json"
  "net/http"
  "net/http/httptest"
  "strings"
  "testing"
)

func newTestAPI(t *testing.T) (http.Handler, *Store) {
  t.Helper()
  store := NewStore()
  return NewRouter(store), store
}

func decodeInto(t *testing.T, rec *httptest.ResponseRecorder, dst any) {
  t.Helper()
  if err := json.Unmarshal(rec.Body.Bytes(), dst); err != nil {
    t.Fatalf("response body is not valid JSON: %v (body=%q)", err, rec.Body.String())
  }
}

func TestCreateItem(t *testing.T) {
  tests := []struct {
    name       string
    body       string
    wantStatus int
    wantErrMsg string
    wantItem   *Item
    wantHeader map[string]string
  }{
    {
      name:       "valid full payload",
      body:       `{"name":"widget","price":9.99,"tags":["a","b"],"note":null}`,
      wantStatus: http.StatusCreated,
      wantItem:   &Item{ID: 1, Name: "widget", Price: 9.99, Tags: []string{"a", "b"}},
      wantHeader: map[string]string{
        "Location":     "/items/1",
        "Content-Type": "application/json; charset=utf-8",
      },
    },
    {
      name:       "minimal payload",
      body:       `{"name":"gadget"}`,
      wantStatus: http.StatusCreated,
      wantItem:   &Item{ID: 1, Name: "gadget"},
    },
    {
      name:       "blank name is unprocessable",
      body:       `{"name":"   "}`,
      wantStatus: http.StatusUnprocessableEntity,
      wantErrMsg: "name is required",
    },
    {
      name:       "unknown field is rejected",
      body:       `{"name":"x","colour":"red"}`,
      wantStatus: http.StatusBadRequest,
      wantErrMsg: "invalid JSON body",
    },
    {
      name:       "malformed JSON",
      body:       `{"name":`,
      wantStatus: http.StatusBadRequest,
      wantErrMsg: "invalid JSON body",
    },
    {
      name:       "empty body",
      body:       ``,
      wantStatus: http.StatusBadRequest,
      wantErrMsg: "invalid JSON body",
    },
  }

  for _, tc := range tests {
    t.Run(tc.name, func(t *testing.T) {
      h, _ := newTestAPI(t)
      req := httptest.NewRequest(http.MethodPost, "/items", strings.NewReader(tc.body))
      req.Header.Set("Content-Type", "application/json")
      rec := httptest.NewRecorder()

      h.ServeHTTP(rec, req)

      if rec.Code != tc.wantStatus {
        t.Errorf("status = %d, want %d (body=%q)", rec.Code, tc.wantStatus, rec.Body.String())
      }
      for k, want := range tc.wantHeader {
        if got := rec.Header().Get(k); got != want {
          t.Errorf("header %s = %q, want %q", k, got, want)
        }
      }
      if tc.wantItem != nil {
        var got Item
        decodeInto(t, rec, &got)
        if got.ID != tc.wantItem.ID || got.Name != tc.wantItem.Name ||
          got.Price != tc.wantItem.Price || len(got.Tags) != len(tc.wantItem.Tags) {
          t.Errorf("item = %+v, want %+v", got, *tc.wantItem)
        }
      }
      if tc.wantErrMsg != "" {
        var got struct{ Error string }
        decodeInto(t, rec, &got)
        if got.Error != tc.wantErrMsg {
          t.Errorf("error = %q, want %q", got.Error, tc.wantErrMsg)
        }
      }
    })
  }
}

func TestGetItem(t *testing.T) {
  h, store := newTestAPI(t)
  store.Create(Item{Name: "seeded", Price: 5})

  tests := []struct {
    name       string
    path       string
    wantStatus int
  }{
    {"existing", "/items/1", http.StatusOK},
    {"missing", "/items/99", http.StatusNotFound},
    {"non-numeric id", "/items/abc", http.StatusBadRequest},
    {"zero id", "/items/0", http.StatusBadRequest},
    {"no id", "/items/", http.StatusNotFound},
  }
  for _, tc := range tests {
    t.Run(tc.name, func(t *testing.T) {
      rec := httptest.NewRecorder()
      h.ServeHTTP(rec, httptest.NewRequest(http.MethodGet, tc.path, nil))
      if rec.Code != tc.wantStatus {
        t.Errorf("GET %s = %d, want %d (body=%q)", tc.path, rec.Code, tc.wantStatus, rec.Body.String())
      }
    })
  }
}

func TestMethodNotAllowed(t *testing.T) {
  h, _ := newTestAPI(t)
  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, httptest.NewRequest(http.MethodPatch, "/items/1", nil))
  if rec.Code != http.StatusMethodNotAllowed {
    t.Errorf("status = %d, want 405", rec.Code)
  }
  if allow := rec.Header().Get("Allow"); allow == "" {
    t.Error("405 response must include an Allow header")
  } else {
    t.Logf("Allow = %q", allow)
  }
}

func TestEmptyListIsArrayNotNull(t *testing.T) {
  h, _ := newTestAPI(t)
  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, httptest.NewRequest(http.MethodGet, "/items", nil))
  if got := rec.Body.String(); got != "[]" {
    t.Errorf("empty list body = %q, want %q", got, "[]")
  }
}

func TestFullLifecycleOverRealServer(t *testing.T) {
  srv := httptest.NewServer(NewRouter(NewStore()))
  t.Cleanup(srv.Close)
  client := srv.Client()

  resp, err := client.Post(srv.URL+"/items", "application/json",
    strings.NewReader(`{"name":"lifecycle","price":1}`))
  if err != nil {
    t.Fatal(err)
  }
  var created Item
  json.NewDecoder(resp.Body).Decode(&created)
  resp.Body.Close()
  if resp.StatusCode != http.StatusCreated {
    t.Fatalf("create = %d, want 201", resp.StatusCode)
  }
  t.Logf("created id=%d via %s", created.ID, resp.Header.Get("Location"))

  resp, err = client.Get(srv.URL + "/items/1")
  if err != nil {
    t.Fatal(err)
  }
  resp.Body.Close()
  t.Logf("get -> %d", resp.StatusCode)

  req, _ := http.NewRequest(http.MethodDelete, srv.URL+"/items/1", nil)
  resp, err = client.Do(req)
  if err != nil {
    t.Fatal(err)
  }
  resp.Body.Close()
  if resp.StatusCode != http.StatusNoContent {
    t.Errorf("delete = %d, want 204", resp.StatusCode)
  }
  t.Logf("delete -> %d", resp.StatusCode)

  resp, err = client.Get(srv.URL + "/items/1")
  if err != nil {
    t.Fatal(err)
  }
  resp.Body.Close()
  if resp.StatusCode != http.StatusNotFound {
    t.Errorf("get after delete = %d, want 404", resp.StatusCode)
  }
  t.Logf("get after delete -> %d", resp.StatusCode)
}
```

Real `go test -race -v` output:

```
=== RUN   TestCreateItem
=== RUN   TestCreateItem/valid_full_payload
=== RUN   TestCreateItem/minimal_payload
=== RUN   TestCreateItem/blank_name_is_unprocessable
=== RUN   TestCreateItem/unknown_field_is_rejected
=== RUN   TestCreateItem/malformed_JSON
=== RUN   TestCreateItem/empty_body
--- PASS: TestCreateItem (0.00s)
    --- PASS: TestCreateItem/valid_full_payload (0.00s)
    --- PASS: TestCreateItem/minimal_payload (0.00s)
    --- PASS: TestCreateItem/blank_name_is_unprocessable (0.00s)
    --- PASS: TestCreateItem/unknown_field_is_rejected (0.00s)
    --- PASS: TestCreateItem/malformed_JSON (0.00s)
    --- PASS: TestCreateItem/empty_body (0.00s)
=== RUN   TestGetItem
=== RUN   TestGetItem/existing
=== RUN   TestGetItem/missing
=== RUN   TestGetItem/non-numeric_id
=== RUN   TestGetItem/zero_id
=== RUN   TestGetItem/no_id
--- PASS: TestGetItem (0.00s)
    --- PASS: TestGetItem/existing (0.00s)
    --- PASS: TestGetItem/missing (0.00s)
    --- PASS: TestGetItem/non-numeric_id (0.00s)
    --- PASS: TestGetItem/zero_id (0.00s)
    --- PASS: TestGetItem/no_id (0.00s)
=== RUN   TestMethodNotAllowed
    api_test.go:147: Allow = "DELETE, GET, HEAD, PUT"
--- PASS: TestMethodNotAllowed (0.00s)
=== RUN   TestEmptyListIsArrayNotNull
--- PASS: TestEmptyListIsArrayNotNull (0.00s)
=== RUN   TestFullLifecycleOverRealServer
    api_test.go:200: created id=1 via /items/1
    api_test.go:207: get -> 200
    api_test.go:218: delete -> 204
    api_test.go:228: get after delete -> 404
--- PASS: TestFullLifecycleOverRealServer (0.00s)
PASS
ok  	example.com/scratch	1.291s
```

Five things the structure gets right.

**`newTestAPI` returns the handler *and* the store.** The 404 test seeds via
`store.Create(...)` rather than a POST, so a `GET` test does not fail because `POST` is
broken. Fresh state per subtest is why `newTestAPI` is called inside `t.Run`, not outside.

**Subtests are independent** ([14.2](../14-testing/02-table-driven.md)), so one failure does
not mask the rest, and `-run 'TestCreateItem/empty_body'` targets one case.

**`t.Errorf` for assertions, `t.Fatalf` for setup.** A bad status is worth continuing past so
you also learn about the body; an undecodable body makes every later assertion meaningless.

**The failure messages include the body.** `status = 500, want 201 (body="...")` tells you what
went wrong without a second run.

**`TestMethodNotAllowed` logged `Allow = "DELETE, GET, HEAD, PUT"`** — the stdlib mux computed
it (16.2), including the implicit `HEAD`. That is a real behaviour worth asserting, and it is
exactly what chi's custom 405 handler would *not* give you (16.7).

The whole suite is clean under `-race`, which matters: `httptest.NewServer` runs handlers on
real goroutines, so a shared-state bug in a handler surfaces here.

## Testing middleware in isolation

A middleware is a function from handler to handler, so the stub handler is both the assertion
target and the proof that `next` ran.

```go
func TestMiddlewareInIsolation(t *testing.T) {
  var called bool
  stub := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    called = true
    w.WriteHeader(http.StatusTeapot)
  })
  h := RequireHeader("X-Api-Key")(stub)

  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, httptest.NewRequest("GET", "/", nil))
  if called {
    t.Error("next handler should NOT have been called")
  }
  t.Logf("no header: status=%d called=%v body=%q", rec.Code, called, rec.Body.String())

  called = false
  req := httptest.NewRequest("GET", "/", nil)
  req.Header.Set("X-Api-Key", "k")
  rec = httptest.NewRecorder()
  h.ServeHTTP(rec, req)
  if !called {
    t.Error("next handler SHOULD have been called")
  }
  t.Logf("with header: status=%d called=%v", rec.Code, called)
}
```

Output:

```
=== RUN   TestMiddlewareInIsolation
    extra_test.go:45: no header: status=400 called=false body="missing X-Api-Key\n"
    extra_test.go:55: with header: status=418 called=true
--- PASS: TestMiddlewareInIsolation (0.00s)
```

`418` is a deliberately absurd status: it can only have come from the stub, so it proves the
middleware passed through rather than writing its own 200. Asserting `called == false` for the
short-circuit case is the half people forget — a middleware that both rejects *and* calls
`next` writes two responses and logs `superfluous response.WriteHeader` (16.1).

## Deterministic responses: inject the clock and the ID generator

A response containing a timestamp or a generated ID cannot be asserted byte-exactly unless
both come from something you control ([14.8](../14-testing/08-test-doubles.md)).

```go
type Clock interface{ Now() time.Time }
type fixedClock struct{ t time.Time }

func (c fixedClock) Now() time.Time { return c.t }

func receiptHandler(clock Clock, nextID func() string) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    buf, _ := json.Marshal(Receipt{ID: nextID(), CreatedAt: clock.Now()})
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    w.Write(buf)
  })
}

func TestDeterministicResponse(t *testing.T) {
  clock := fixedClock{t: time.Date(2024, 3, 1, 12, 0, 0, 0, time.UTC)}
  n := 0
  ids := func() string { n++; return fmt.Sprintf("rcpt-%04d", n) }

  h := receiptHandler(clock, ids)
  for range 2 {
    rec := httptest.NewRecorder()
    h.ServeHTTP(rec, httptest.NewRequest("POST", "/receipts", nil))
    t.Logf("body = %s", rec.Body.String())
  }

  rec := httptest.NewRecorder()
  h.ServeHTTP(rec, httptest.NewRequest("POST", "/receipts", nil))
  want := `{"id":"rcpt-0003","created_at":"2024-03-01T12:00:00Z"}`
  if got := rec.Body.String(); got != want {
    t.Errorf("body = %s, want %s", got, want)
  } else {
    t.Log("byte-exact assertion possible because both clock and IDs are injected")
  }
}
```

Output:

```
=== RUN   TestDeterministicResponse
    extra_test.go:109: body = {"id":"rcpt-0001","created_at":"2024-03-01T12:00:00Z"}
    extra_test.go:109: body = {"id":"rcpt-0002","created_at":"2024-03-01T12:00:00Z"}
    extra_test.go:118: byte-exact assertion possible because both clock and IDs are injected
--- PASS: TestDeterministicResponse (0.00s)
```

The handler's dependencies are a `Clock` interface and a `func() string`. Production passes
`realClock{}` and a UUID generator; the test passes a frozen time and a counter. This is the
same technique used for the slog time attribute (`ReplaceAttr`) and the request-ID generator
throughout this module — and it is why every "Output:" block in these twelve lessons is
reproducible.

The counterexample: a handler calling `time.Now()` and `uuid.New()` directly is untestable
beyond "it returned 201", and no amount of test cleverness fixes it. The fix is always in the
handler's constructor.

## Testing a file upload

Build the multipart body with `multipart.Writer` — the same code as a real client (16.10) —
and use `mw.FormDataContentType()` for the header, which carries the generated boundary.

```go
func TestFileUpload(t *testing.T) {
  var buf bytes.Buffer
  mw := multipart.NewWriter(&buf)
  if err := mw.WriteField("title", "my avatar"); err != nil {
    t.Fatal(err)
  }
  fw, err := mw.CreateFormFile("avatar", "me.png")
  if err != nil {
    t.Fatal(err)
  }
  fw.Write([]byte("\x89PNG\r\n\x1a\nfake-png-bytes"))
  if err := mw.Close(); err != nil {
    t.Fatal(err)
  }

  req := httptest.NewRequest("POST", "/upload", &buf)
  req.Header.Set("Content-Type", mw.FormDataContentType())
  rec := httptest.NewRecorder()
  http.HandlerFunc(uploadHandler).ServeHTTP(rec, req)

  if rec.Code != http.StatusOK {
    t.Fatalf("status = %d, body = %q", rec.Code, rec.Body.String())
  }
  t.Logf("upload accepted: %s", rec.Body.String())
}
```

Output:

```
=== RUN   TestFileUpload
    extra_test.go:164: upload accepted: name="me.png" size=22 title="my avatar" sha=16
--- PASS: TestFileUpload (0.00s)
```

`mw.Close()` is **mandatory** — it writes the closing boundary, and without it
`ParseMultipartForm` fails with `unexpected EOF`. It is the single most common mistake in
upload tests. Write the text fields *before* the file parts, as a browser does, if your handler
reads them in order with `MultipartReader`.

## The double-server pattern

To test a handler that calls an external API, stand up a second `httptest.Server` as the
upstream and point your client's `baseURL` at it. No mocking library, and the real HTTP stack
is exercised on both sides.

```go
func TestDoubleServerPattern(t *testing.T) {
  upstream := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    t.Logf("  upstream received: %s %s", r.Method, r.URL.RequestURI())
    switch r.URL.Query().Get("city") {
    case "lisbon":
      fmt.Fprint(w, `{"temp":29.5}`)
    case "oslo":
      fmt.Fprint(w, `{"temp":4}`)
    default:
      w.WriteHeader(http.StatusNotFound)
    }
  }))
  t.Cleanup(upstream.Close)

  wc := &WeatherClient{baseURL: upstream.URL, http: upstream.Client()}
  ours := httptest.NewServer(forecastHandler(wc))
  t.Cleanup(ours.Close)

  for _, city := range []string{"lisbon", "oslo", "atlantis"} {
    resp, err := ours.Client().Get(ours.URL + "/forecast?city=" + city)
    if err != nil {
      t.Fatal(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    t.Logf("%-9s -> %s %s", city, resp.Status, b)
  }
}
```

Output:

```
=== RUN   TestDoubleServerPattern
    extra_test.go:211:   upstream received: GET /temp?city=lisbon
    extra_test.go:234: lisbon    -> 200 OK {"verdict":"hot","temp":29.5}
    extra_test.go:211:   upstream received: GET /temp?city=oslo
    extra_test.go:234: oslo      -> 200 OK {"verdict":"mild","temp":4}
    extra_test.go:211:   upstream received: GET /temp?city=atlantis
    extra_test.go:234: atlantis  -> 502 Bad Gateway upstream unavailable
--- PASS: TestDoubleServerPattern (0.00s)
```

The upstream handler **asserts on what it received** (`t.Logf` here; `t.Errorf` in a real
test) and controls what it returns, so you can drive your handler's error paths — a 404, a 500,
a malformed body, a wrong content type — without touching your own code. That is testing a
third-party integration you cannot otherwise reach.

The alternative is a fake `http.RoundTripper` returning canned responses (16.10), which is
faster and needs no network, but does not exercise real serialization or the real transport.
Use the double server for integration confidence and a fake RoundTripper for unit speed.

## Testing timeouts and cancellation

```go
func TestUpstreamTimeoutPropagates(t *testing.T) {
  upstream := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    select {
    case <-time.After(2 * time.Second):
      fmt.Fprint(w, `{"temp":1}`)
    case <-r.Context().Done():
      t.Logf("  upstream saw cancellation: %v", r.Context().Err())
    }
  }))
  t.Cleanup(upstream.Close)

  wc := &WeatherClient{baseURL: upstream.URL,
    http: &http.Client{Timeout: 100 * time.Millisecond}}
  ours := httptest.NewServer(forecastHandler(wc))
  t.Cleanup(ours.Close)

  start := time.Now()
  resp, err := ours.Client().Get(ours.URL + "/forecast?city=lisbon")
  // ... read, log, assert 502
}
```

Output:

```
=== RUN   TestUpstreamTimeoutPropagates
    extra_test.go:244:   upstream saw cancellation: context canceled
    extra_test.go:261: our service -> 502 Bad Gateway "upstream unavailable\n" (returned fast: true)
--- PASS: TestUpstreamTimeoutPropagates (0.10s)
```

The test took 0.10s against a handler that would have taken 2s. Note the upstream's own
context reported `context canceled` — the client's timeout closed the connection, and the
server noticed. That propagation is 16.12's subject.

Keep timeout tests in the tens of milliseconds and assert on **behaviour** (the status, that
it returned fast) rather than on an exact duration, which is the classic flaky assertion.

## What httptest does not cover

One line: real TLS certificate chains and client-cert auth, HTTP proxies, HTTP/2-specific
behaviour (`httptest` is HTTP/1.1 unless you configure `NextProtos` and `EnableHTTP2`), real
network latency and packet loss, and anything about your load balancer or ingress. Those need
an integration environment.

## Quick Reference

| Call | Purpose |
|---|---|
| `httptest.NewRequest(m, target, body)` | a **server-side** request; panics on a bad URL |
| `req.SetPathValue(k, v)` | set a wildcard value without a router (1.22+) |
| `httptest.NewRecorder()` | an in-memory `ResponseWriter`; `Code` starts at 200 |
| `rec.Code` / `rec.Body` / `rec.Header()` / `rec.Flushed` | the assertions |
| `rec.Result()` | an `*http.Response` for client-side code |
| `httptest.NewServer(h)` / `NewTLSServer(h)` | a real listener; `defer srv.Close()` |
| `httptest.NewUnstartedServer(h)` + `Start()` | configure `srv.Config` first |
| `srv.URL` | random port — never hardcode |
| `srv.Client()` | trusts the TLS server's self-signed cert |
| `srv.Config.ErrorLog` | capture the server's own log lines |
| `srv.CloseClientConnections()` | force a client-side error |

| Pattern | Use |
|---|---|
| recorder, no server | the default — fast, no ports, no goroutines |
| `NewServer` | testing a **client**, streaming, end-to-end smoke |
| `NewTLSServer` + `srv.Client()` | TLS paths without `InsecureSkipVerify` |
| double server | a handler that calls an upstream |
| fake `RoundTripper` | a client, with no network at all |
| injected clock + ID gen | byte-exact response assertions |
| `t.Cleanup(srv.Close)` | in helpers, better than `defer` |

## Common Mistakes

**Using `http.NewRequest` for a handler test.** It builds a *client* request with no `Host` or
`RemoteAddr` and a `RequestURI` the server rejects; use `httptest.NewRequest`.

**Treating `rec.Code == 200` as proof the handler ran.** A fresh recorder already reports 200,
so assert on the body or headers too.

**Comparing response bodies as strings.** Key order follows struct field order and `Encode`
adds a newline, so a harmless refactor breaks the test; unmarshal and compare structs.

**Forgetting `defer srv.Close()`.** The listener and its goroutines outlive the test and leak
across the package run; prefer `t.Cleanup(srv.Close)`.

**Using the default client against `httptest.NewTLSServer`.** It cannot verify the self-signed
cert; use `srv.Client()` rather than `InsecureSkipVerify`.

**Omitting `mw.Close()` in a multipart test.** The closing boundary is missing, so the handler
fails with `unexpected EOF` and the test looks like a handler bug.

**Testing a handler that calls `time.Now()` directly.** You cannot assert the response, only
its status; inject a clock.

**Asserting an exact duration in a timeout test.** It is flaky on a loaded CI box; assert the
status and a generous upper bound instead.

## Exercises

1. Add table-driven tests for `PUT /items/{id}` and `DELETE /items/{id}` covering 200, 204,
   400, 404 and 405, with the `Allow` header asserted on the 405. Run with
   `-race -count=5` and confirm no flakes.
2. Write a test that asserts `tags` is **absent** from the JSON when empty but `note` is
   present as `null`, by unmarshaling into `map[string]any`. Explain why a struct cannot
   express this assertion.
3. Build a fake `http.RoundTripper` returning canned responses from a map, and write the same
   client tests twice — once with it, once with a double `httptest` server. Compare the
   runtimes with `-count=20` and state what each one catches that the other misses.
4. Test an SSE handler by reading `srv.URL` with a `bufio.Scanner` and asserting on the first
   three events, then cancel the request context and assert the handler observed it. Keep the
   whole test under 200ms.
5. Write a test helper `assertJSON(t, rec, want any)` that unmarshals both sides into
   `map[string]any` and reports the first differing key path. Use it to replace the field-by-field
   comparisons in this lesson's test file, then compare its failure output against `cmp.Diff`'s.

---

[← 16.10 The net/http Client](10-http-client.md) · [Course Index](../README.md) · [16.12 Context Propagation Through Handlers →](12-context-propagation.md)
