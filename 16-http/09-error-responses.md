[← 16.8 Request Validation](08-validation.md) · [Course Index](../README.md) · [16.10 The net/http Client →](10-http-client.md)

# 16.9 — HTTP Errors and Problem Responses

## What You'll Learn

- The handler-returns-error pattern: a custom handler type so handlers can `return err` instead of writing a response — the single most valuable idea in this lesson
- One `classify(err) APIError` function mapping domain errors to status codes with `errors.Is`/`errors.As`
- Internal error text must **never** reach the client — proven with a real database password that stays in the log
- RFC 9457 `application/problem+json`, the pragmatic `{"error":{...}}` alternative, and a status-code selection table

## Coming From Another Language

| Other | Go |
|---|---|
| Express `next(err)` reaching an error-handling middleware | no error channel; a handler type that **returns** `error` gives you one |
| `throw new NotFoundError()` caught by a framework | `return fmt.Errorf("...: %w", ErrNotFound)`, mapped at the boundary |
| `@app.exception_handler(HTTPException)` | one adapter's `ServeHTTP`, written once |
| Frameworks that leak stack traces in dev mode | you never leak; the request ID is the bridge |
| `err.statusCode` set on the error object | status lives in the **transport** layer, not on the domain error |

This lesson builds directly on [8.6](../08-errors/06-strategy.md), which established
sentinel errors, typed errors, and `%w` wrapping. The question here is narrower: given an
error from three layers down, what does the client see?

## The problem with writing errors inline

Every handler in 16.5 looked like this:

```go
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
```

Three problems, which get worse with every handler you add. The status/message mapping is
**duplicated** in every handler, so the shapes drift. Forgetting a `return` after
`writeErr` writes two responses. And there is nowhere to put the cross-cutting work —
logging, request-ID correlation, metrics — because the error is consumed where it is found.

## The handler-returns-error pattern

Define a handler type that returns `error` and give it a `ServeHTTP` method. That method is
the **one place** in the service where an error becomes an HTTP response.

```go
type Handler func(http.ResponseWriter, *http.Request) error

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  err := h(w, r)
  if err == nil {
    return
  }
  // classify, log once, write one response
}
```

That is the same trick as `http.HandlerFunc` (16.3): a named function type with a method,
so it satisfies `http.Handler` and plugs into any mux or middleware chain unchanged. The
handler above becomes:

```go
func (a *API) get(w http.ResponseWriter, r *http.Request) error {
  id, err := itemID(r)
  if err != nil {
    return fmt.Errorf("parsing id %q: %w", r.PathValue("id"), ErrBadInput)
  }
  it, err := a.store.Get(r.Context(), id)
  if err != nil {
    return err
  }
  return writeJSON(w, http.StatusOK, it)
}
```

Every error path is now `return err`, which means a missing `return` is a **compile error**
rather than a double-written response, and `go vet` catches an ignored error.

## Mapping domain errors to status codes

The classifier is one function, and it is the only place that knows both your domain and
HTTP. Build it on the sentinels and typed errors from 8.6.

```go
var (
  ErrNotFound     = errors.New("resource not found")
  ErrConflict     = errors.New("resource already exists")
  ErrUnauthorized = errors.New("not authenticated")
  ErrForbidden    = errors.New("not permitted")
)

type ValidationError struct{ Fields map[string]string }
type RateLimitError struct{ RetryAfterSeconds int }

type APIError struct {
  Status  int
  Code    string
  Message string
  Fields  map[string]string
  Retry   int
}

func classify(err error) APIError {
  var ve *ValidationError
  var rl *RateLimitError
  switch {
  case errors.Is(err, ErrNotFound):
    return APIError{http.StatusNotFound, "not_found", "The requested resource does not exist.", nil, 0}
  case errors.Is(err, ErrConflict):
    return APIError{http.StatusConflict, "conflict", "The resource already exists.", nil, 0}
  case errors.Is(err, ErrUnauthorized):
    return APIError{http.StatusUnauthorized, "unauthorized", "Authentication is required.", nil, 0}
  case errors.Is(err, ErrForbidden):
    return APIError{http.StatusForbidden, "forbidden", "You do not have permission to do that.", nil, 0}
  case errors.As(err, &ve):
    return APIError{http.StatusUnprocessableEntity, "validation_failed", "One or more fields are invalid.", ve.Fields, 0}
  case errors.As(err, &rl):
    return APIError{http.StatusTooManyRequests, "rate_limited", "Too many requests.", nil, rl.Retry}
  }
  return APIError{http.StatusInternalServerError, "internal_error", "An unexpected error occurred.", nil, 0}
}
```

Three design rules in that function.

**`errors.Is` for sentinels, `errors.As` for typed errors.** `errors.Is` sees through any
depth of `%w` wrapping, so a repository returning
`fmt.Errorf("loading user %d: %w", id, ErrNotFound)` still classifies as 404 even after a
service layer wraps it again ([8.3](../08-errors/03-is-as-unwrap.md)).

**The default is 500.** An error you did not anticipate is a bug, and a bug is a 500. Never
default to 400 — that silently reclassifies your outages as client errors and your alerts
go quiet.

**The `Message` is a fixed, safe string.** It is written by you for a client's eyes. It is
never `err.Error()`.

Where this function lives matters: the HTTP package, not the domain package. Your domain
exports `ErrNotFound`; it must not know that 404 exists. That is what keeps the domain
reusable from a CLI or a gRPC server.

## The complete thing, with internal detail staying internal

Everything together: the handler adapter, the classifier, RFC 9457 problem responses, a
request ID correlating the log with the client's response, and a handler that fails with a
database connection string in the error.

```go
package main

import (
  "context"
  "encoding/json"
  "errors"
  "fmt"
  "io"
  "log/slog"
  "net/http"
  "net/http/httptest"
  "os"
  "sort"
  "strings"
  "sync/atomic"
)

var (
  ErrNotFound     = errors.New("resource not found")
  ErrConflict     = errors.New("resource already exists")
  ErrUnauthorized = errors.New("not authenticated")
  ErrForbidden    = errors.New("not permitted")
)

type ValidationError struct{ Fields map[string]string }

func (e *ValidationError) Error() string {
  keys := make([]string, 0, len(e.Fields))
  for k := range e.Fields {
    keys = append(keys, k)
  }
  sort.Strings(keys)
  return "validation failed on: " + strings.Join(keys, ", ")
}

type RateLimitError struct{ RetryAfterSeconds int }

func (e *RateLimitError) Error() string {
  return fmt.Sprintf("rate limit exceeded, retry in %ds", e.RetryAfterSeconds)
}

type APIError struct {
  Status  int
  Code    string
  Message string
  Fields  map[string]string
  Retry   int
}

func classify(err error) APIError {
  var ve *ValidationError
  var rl *RateLimitError
  switch {
  case errors.Is(err, ErrNotFound):
    return APIError{http.StatusNotFound, "not_found", "The requested resource does not exist.", nil, 0}
  case errors.Is(err, ErrConflict):
    return APIError{http.StatusConflict, "conflict", "The resource already exists.", nil, 0}
  case errors.Is(err, ErrUnauthorized):
    return APIError{http.StatusUnauthorized, "unauthorized", "Authentication is required.", nil, 0}
  case errors.Is(err, ErrForbidden):
    return APIError{http.StatusForbidden, "forbidden", "You do not have permission to do that.", nil, 0}
  case errors.As(err, &ve):
    return APIError{http.StatusUnprocessableEntity, "validation_failed", "One or more fields are invalid.", ve.Fields, 0}
  case errors.As(err, &rl):
    return APIError{http.StatusTooManyRequests, "rate_limited", "Too many requests.", nil, rl.RetryAfterSeconds}
  }
  return APIError{http.StatusInternalServerError, "internal_error", "An unexpected error occurred.", nil, 0}
}

var log = slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
  ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
    if a.Key == slog.TimeKey {
      return slog.Attr{}
    }
    return a
  },
}))

type Problem struct {
  Type     string            `json:"type"`
  Title    string            `json:"title"`
  Status   int               `json:"status"`
  Detail   string            `json:"detail,omitempty"`
  Instance string            `json:"instance,omitempty"`
  TraceID  string            `json:"trace_id,omitempty"`
  Errors   map[string]string `json:"errors,omitempty"`
}

type Handler func(http.ResponseWriter, *http.Request) error

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  err := h(w, r)
  if err == nil {
    return
  }
  api := classify(err)
  reqID := RequestIDFrom(r.Context())

  if api.Status >= 500 {
    log.Error("request failed", "request_id", reqID, "method", r.Method,
      "path", r.URL.Path, "status", api.Status, "internal_error", err.Error())
  } else {
    log.Warn("request rejected", "request_id", reqID, "method", r.Method,
      "path", r.URL.Path, "status", api.Status, "internal_error", err.Error())
  }

  p := Problem{
    Type:     "https://api.example.com/problems/" + api.Code,
    Title:    http.StatusText(api.Status),
    Status:   api.Status,
    Detail:   api.Message,
    Instance: r.URL.Path,
    TraceID:  reqID,
    Errors:   api.Fields,
  }
  buf, _ := json.Marshal(p)
  w.Header().Set("Content-Type", "application/problem+json")
  if api.Retry > 0 {
    w.Header().Set("Retry-After", fmt.Sprint(api.Retry))
  }
  w.WriteHeader(api.Status)
  w.Write(buf)
}

type reqIDKey struct{}

func RequestIDFrom(ctx context.Context) string {
  id, _ := ctx.Value(reqIDKey{}).(string)
  return id
}

func contextWithID(r *http.Request, id string) context.Context {
  return context.WithValue(r.Context(), reqIDKey{}, id)
}

func RequestID(gen func() string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      id := gen()
      w.Header().Set("X-Request-Id", id)
      next.ServeHTTP(w, r.WithContext(contextWithID(r, id)))
    })
  }
}

func main() {
  var n atomic.Int64
  gen := func() string { return fmt.Sprintf("req-%03d", n.Add(1)) }

  mux := http.NewServeMux()
  mux.Handle("GET /ok", Handler(func(w http.ResponseWriter, r *http.Request) error {
    w.Header().Set("Content-Type", "application/json")
    io.WriteString(w, `{"ok":true}`)
    return nil
  }))
  mux.Handle("GET /missing", Handler(func(w http.ResponseWriter, r *http.Request) error {
    return fmt.Errorf("loading user 42 from table users: %w", ErrNotFound)
  }))
  mux.Handle("POST /dup", Handler(func(w http.ResponseWriter, r *http.Request) error {
    return fmt.Errorf("insert: duplicate key value violates unique constraint \"users_email_key\": %w", ErrConflict)
  }))
  mux.Handle("GET /secret", Handler(func(w http.ResponseWriter, r *http.Request) error {
    return ErrUnauthorized
  }))
  mux.Handle("GET /admin", Handler(func(w http.ResponseWriter, r *http.Request) error {
    return fmt.Errorf("user 42 lacks role admin: %w", ErrForbidden)
  }))
  mux.Handle("POST /create", Handler(func(w http.ResponseWriter, r *http.Request) error {
    return &ValidationError{Fields: map[string]string{
      "email": "must be a valid email address",
      "age":   "must be 18 or greater",
    }}
  }))
  mux.Handle("GET /limited", Handler(func(w http.ResponseWriter, r *http.Request) error {
    return &RateLimitError{RetryAfterSeconds: 30}
  }))
  mux.Handle("GET /broken", Handler(func(w http.ResponseWriter, r *http.Request) error {
    return fmt.Errorf("query postgres://app:hunter2@db.internal:5432/prod: connection refused")
  }))

  srv := httptest.NewServer(RequestID(gen)(mux))
  defer srv.Close()

  type rq struct{ m, p string }
  for _, q := range []rq{{"GET", "/ok"}, {"GET", "/missing"}, {"POST", "/dup"},
    {"GET", "/secret"}, {"GET", "/admin"}, {"POST", "/create"},
    {"GET", "/limited"}, {"GET", "/broken"}} {
    req, _ := http.NewRequest(q.m, srv.URL+q.p, nil)
    resp, _ := http.DefaultClient.Do(req)
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("CLIENT %-6s %-10s %-28s ct=%-28q retry=%q\n", q.m, q.p, resp.Status,
      resp.Header.Get("Content-Type"), resp.Header.Get("Retry-After"))
    fmt.Printf("   %s\n", b)
  }
}
```

The ID generator is injected so the output is reproducible; `slog`'s time attribute is
stripped for the same reason. `level=WARN`/`level=ERROR` lines are the **server log**;
`CLIENT` lines are what the client received.

Output:

```
CLIENT GET    /ok        200 OK                       ct="application/json"           retry=""
   {"ok":true}
level=WARN msg="request rejected" request_id=req-002 method=GET path=/missing status=404 internal_error="loading user 42 from table users: resource not found"
CLIENT GET    /missing   404 Not Found                ct="application/problem+json"   retry=""
   {"type":"https://api.example.com/problems/not_found","title":"Not Found","status":404,"detail":"The requested resource does not exist.","instance":"/missing","trace_id":"req-002"}
level=WARN msg="request rejected" request_id=req-003 method=POST path=/dup status=409 internal_error="insert: duplicate key value violates unique constraint \"users_email_key\": resource already exists"
CLIENT POST   /dup       409 Conflict                 ct="application/problem+json"   retry=""
   {"type":"https://api.example.com/problems/conflict","title":"Conflict","status":409,"detail":"The resource already exists.","instance":"/dup","trace_id":"req-003"}
level=WARN msg="request rejected" request_id=req-004 method=GET path=/secret status=401 internal_error="not authenticated"
CLIENT GET    /secret    401 Unauthorized             ct="application/problem+json"   retry=""
   {"type":"https://api.example.com/problems/unauthorized","title":"Unauthorized","status":401,"detail":"Authentication is required.","instance":"/secret","trace_id":"req-004"}
level=WARN msg="request rejected" request_id=req-005 method=GET path=/admin status=403 internal_error="user 42 lacks role admin: not permitted"
CLIENT GET    /admin     403 Forbidden                ct="application/problem+json"   retry=""
   {"type":"https://api.example.com/problems/forbidden","title":"Forbidden","status":403,"detail":"You do not have permission to do that.","instance":"/admin","trace_id":"req-005"}
level=WARN msg="request rejected" request_id=req-006 method=POST path=/create status=422 internal_error="validation failed on: age, email"
CLIENT POST   /create    422 Unprocessable Entity     ct="application/problem+json"   retry=""
   {"type":"https://api.example.com/problems/validation_failed","title":"Unprocessable Entity","status":422,"detail":"One or more fields are invalid.","instance":"/create","trace_id":"req-006","errors":{"age":"must be 18 or greater","email":"must be a valid email address"}}
level=WARN msg="request rejected" request_id=req-007 method=GET path=/limited status=429 internal_error="rate limit exceeded, retry in 30s"
CLIENT GET    /limited   429 Too Many Requests        ct="application/problem+json"   retry="30"
   {"type":"https://api.example.com/problems/rate_limited","title":"Too Many Requests","status":429,"detail":"Too many requests.","instance":"/limited","trace_id":"req-007"}
level=ERROR msg="request failed" request_id=req-008 method=GET path=/broken status=500 internal_error="query postgres://app:hunter2@db.internal:5432/prod: connection refused"
CLIENT GET    /broken    500 Internal Server Error    ct="application/problem+json"   retry=""
   {"type":"https://api.example.com/problems/internal_error","title":"Internal Server Error","status":500,"detail":"An unexpected error occurred.","instance":"/broken","trace_id":"req-008"}
```

### Internal text never reaches the client

Look at the last pair of lines, which is the whole point of the lesson.

The server log says:

```
internal_error="query postgres://app:hunter2@db.internal:5432/prod: connection refused"
```

The client gets:

```json
{"type":"https://api.example.com/problems/internal_error","title":"Internal Server Error","status":500,"detail":"An unexpected error occurred.","instance":"/broken","trace_id":"req-008"}
```

The database **password**, the internal **hostname**, the database **name**, and the fact
that it is Postgres at all — none of it crossed the boundary. What crossed was
`trace_id: "req-008"`, and the log line carries `request_id=req-008`. A user pastes
`req-008` into a support ticket and you `grep` one line in your logs. That is the entire
mechanism, and it is worth more than any amount of detail in the response body.

The same pattern protects you in the 409 case: the log names the violated constraint
(`users_email_key` — which reveals your schema), the client sees "The resource already
exists."

Idiomatic: the client gets a **code** (machine-readable), a **safe message**
(human-readable), and a **trace ID** (correlatable). Never `err.Error()`.

### Logged once, at the boundary

Every error above produced exactly **one** log line, written in `ServeHTTP`. The repository
did not log, the service did not log, the handler did not log — they wrapped with `%w` and
returned.

That is the rule, and the reason is operational: layered logging produces three or four lines
per failure, at three different severities, with no shared identifier, and your incident
review becomes archaeology. The wrapping chain already carries the context that each layer
would have logged — `"loading user 42 from table users: resource not found"` tells you the
operation, the ID, and the cause in one string.

Note the **severity split**: 5xx is `ERROR` (your bug, page someone), 4xx is `WARN` (the
client's problem, useful for spotting a broken integration but not an outage). Alerting on a
4xx rate is a decision, not a default; alerting on 5xx always is.

## RFC 9457 problem+json, and the pragmatic alternative

The output above is RFC 9457 (which obsoleted RFC 7807). Five standard members:

| Member | Meaning |
|---|---|
| `type` | a URI identifying the problem *kind* — the stable, machine-readable key |
| `title` | a short human summary of the `type`, same for every occurrence |
| `status` | the HTTP status, duplicated in the body |
| `detail` | a human explanation of *this* occurrence |
| `instance` | a URI identifying this occurrence |

Plus any extension members you like — above, `trace_id` and `errors`. The content type must
be `application/problem+json`.

The argument for it: it is a standard, so generic clients and gateways can render it, and
`type` gives you a documented URL per error kind that clients can branch on without parsing
prose.

The argument against: `type` URIs are bureaucratic for an internal API, `title` duplicates
`http.StatusText`, and `status` duplicates the status line. Many teams ship this instead:

```json
{"error": {"code": "validation_failed", "message": "One or more fields are invalid.",
           "fields": {"email": "must be a valid email address"},
           "trace_id": "req-006"}}
```

Both are fine. What matters is that **every** error from your API has the same shape, so a
client writes one error handler. The failure mode is an API that returns problem+json from
handlers, `{"error":"..."}` from middleware, and Go's plain-text `404 page not found` from
the router — which is exactly why 16.7 spends time on custom `NotFound`/`MethodNotAllowed`
handlers.

## Status code selection

| Status | Use when | Note |
|---|---|---|
| **400** Bad Request | the request is unparseable — malformed JSON, bad syntax, unknown field | the client cannot fix it by changing values |
| **401** Unauthorized | no or invalid credentials | misnamed; it means *unauthenticated*. Send `WWW-Authenticate` |
| **403** Forbidden | authenticated, but not permitted | do not use for "not authenticated" |
| **404** Not Found | the resource does not exist | also for a resource the caller may not *know* exists, instead of 403 |
| **405** Method Not Allowed | the path exists, the method does not | **must** include `Allow` |
| **409** Conflict | the request conflicts with current state — duplicate key, stale version | the classic for optimistic concurrency |
| **410** Gone | deliberately and permanently removed | rare; 404 is fine |
| **413** Content Too Large | body exceeded the limit | from `*http.MaxBytesError` (16.4) |
| **415** Unsupported Media Type | wrong or unparseable `Content-Type` | not 400 |
| **422** Unprocessable Content | syntactically valid, semantically invalid — a field failed validation | with a `fields` map (16.8) |
| **429** Too Many Requests | rate limited | **send `Retry-After`** |
| **500** Internal Server Error | your bug | the default for anything unclassified |
| **501** Not Implemented | the method is not supported at all | |
| **502** Bad Gateway | an upstream returned garbage | |
| **503** Service Unavailable | overloaded, shutting down, dependency down | send `Retry-After`; what a health check failure returns |
| **504** Gateway Timeout | an upstream timed out | chi's `middleware.Timeout` (16.7) |

**400 vs 422, settled in one line:** 400 when you could not *parse* the request, 422 when you
parsed it fine and a value is unacceptable — and if your team prefers 400 for both, that is
also defensible, so pick one, write it down, and stop re-litigating it.

**`Retry-After`** takes either seconds (`Retry-After: 30`) or an HTTP date. Send it on 429
and on 503 — the output above shows `retry="30"` on the 429 only. Without it, clients
retry immediately and make your overload worse.

## Non-JSON clients

A browser hitting your API directly, or a `curl` without `Accept`, should not get JSON it
cannot read. Negotiate on `Accept`:

```go
func (h Handler) writeError(w http.ResponseWriter, r *http.Request, api APIError, traceID string) {
  if !strings.Contains(r.Header.Get("Accept"), "json") &&
    strings.Contains(r.Header.Get("Accept"), "text/html") {
    w.Header().Set("Content-Type", "text/plain; charset=utf-8")
    w.WriteHeader(api.Status)
    fmt.Fprintf(w, "%d %s\n\n%s\n\nTrace ID: %s\n",
      api.Status, http.StatusText(api.Status), api.Message, traceID)
    return
  }
  // problem+json as above
}
```

Keep the trace ID in **both** forms — it is the only thing a user can report. For a pure
API, defaulting everything to JSON regardless of `Accept` is a legitimate simplification;
just do not send an empty body, which is what chi's `middleware.Recoverer` does (16.7).

## Quick Reference

| Piece | Shape |
|---|---|
| handler type | `type Handler func(http.ResponseWriter, *http.Request) error` |
| adapter | `func (h Handler) ServeHTTP(w, r)` — classify, log once, write once |
| register it | `mux.Handle("GET /x", Handler(a.get))` — it is an `http.Handler` |
| classifier | `func classify(error) APIError` — in the transport package |
| sentinels | `errors.Is(err, ErrNotFound)` — sees through `%w` |
| typed errors | `errors.As(err, &ve)` — carries fields |
| default | **500**, never 400 |

| Safety rule | Why |
|---|---|
| never send `err.Error()` | leaks credentials, hostnames, schema, file paths |
| always send a trace ID | the only correlation a user can report |
| log once, at the boundary | one line per failure, with full wrapped context |
| 5xx → `ERROR`, 4xx → `WARN` | alert on the first, trend the second |

| Header | With |
|---|---|
| `Content-Type: application/problem+json` | RFC 9457 bodies |
| `Allow` | every 405 |
| `WWW-Authenticate` | 401 |
| `Retry-After` | 429, 503 |

## Common Mistakes

**Putting `err.Error()` in the response body.** It leaks connection strings, internal
hostnames and schema names; send a fixed safe message plus a trace ID.

**Defaulting unclassified errors to 400.** Your outages then look like client errors and
your 5xx alerts stay silent; default to 500.

**Logging at every layer.** One failure produces four uncorrelated lines; wrap with `%w` and
log once in the adapter.

**Comparing errors with `==`.** A `%w`-wrapped sentinel fails `==` but passes `errors.Is`, so
a 404 silently becomes a 500.

**Mixing error shapes.** Handlers returning problem+json while the router returns plain-text
404s forces clients to handle two formats; override `NotFound`/`MethodNotAllowed` too.

**Omitting `Retry-After` on 429 and 503.** Clients retry instantly and amplify the overload.

**Putting HTTP status codes on domain errors.** A `StatusCode` field on a repository error
couples your database layer to HTTP; map at the boundary instead.

## Exercises

1. Convert the 16.5 CRUD API to the handler-returns-error pattern. Every handler must end in
   `return` statements only, and the status/message mapping must appear exactly once. Confirm
   every status from the original output is unchanged.
2. Add a `StatusError` type with `Unwrap`, so a handler can `return StatusError{409, err}` to
   override the classifier for one case. Make `classify` honour it with `errors.As` ahead of
   the sentinel checks, and prove both the override and the fallthrough.
3. Write a test asserting that for an error containing the literal string
   `"password=hunter2"`, the response body does not contain `"hunter2"` anywhere while the
   captured log line does. Use an `slog` handler writing to a `bytes.Buffer`.
4. Implement both response shapes behind a flag — RFC 9457 and `{"error":{...}}` — and a
   table-driven test that checks the status, content type and the trace ID for all twelve
   status codes in the table above.
5. Add `errors.Join` support ([8.5](../08-errors/05-join.md)) so a handler can return several
   independent failures at once, and have the classifier pick the **most severe** status
   among them. Show the real response for a joined 404 plus 422.

---

[← 16.8 Request Validation](08-validation.md) · [Course Index](../README.md) · [16.10 The net/http Client →](10-http-client.md)
