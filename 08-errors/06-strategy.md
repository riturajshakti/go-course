[← 8.5 errors.Join and Multiple Errors](05-join.md) · [Course Index](../README.md) · [9.1 Type Parameters on Functions →](../09-generics/01-type-parameters.md)

# 8.6 — Error Handling Strategy for Libraries vs Services

## What You'll Learn

- Libraries return errors and never log them; services handle and log once
- Mapping domain errors to HTTP status codes belongs at the transport boundary
- The public response and the internal log line carry different information
- Retry only what is retryable, and `context.Canceled` is not

## Coming From Another Language

| Other | Go |
|---|---|
| A global exception handler / Express error middleware | one mapping function at the transport boundary, called by each handler |
| Python `logging.exception()` inside the failing function | return the error; log at the handler, once |
| JS `err.statusCode` set on the thrown object | keep status out of domain errors; map at the edge |
| Rust `thiserror` in the library, `anyhow` in the binary | sentinels/typed errors in the library, wrap-and-log in the service |
| Sentry capturing from anywhere in the stack | capture where you log, which is where you handle |

The one rule that matters: **an error is handled exactly once.** Everywhere else you add
context and return it. "Log and return" is the most common mistake in Go services — it
produces the same failure in your logs five times at five different layers, and makes the
real handling site impossible to find.

## Libraries: the rules

| Do | Why |
|---|---|
| Return errors, never log them | you do not own the caller's logger, log level, or format |
| Wrap with context the caller cannot reconstruct | the ID, the path, the operation |
| Export sentinels and typed errors deliberately | each one is permanent API (8.4) |
| Document which errors each function returns | "returns an error wrapping [ErrNotFound] if ..." |
| Translate dependency errors at your boundary | wrapping `pgx` errors with `%w` makes pgx your API (8.2) |

| Don't | Why |
|---|---|
| `log.Printf` / `slog.Error` inside a library function | duplicate logs, wrong format, no way to turn off |
| `panic` across an API boundary | the exception is programmer error: a nil required argument, a bad regexp in `MustCompile` |
| `os.Exit` / `log.Fatal` anywhere | skips every `defer` in the caller's program |
| Depend on another package's error *strings* | not an API; use `errors.Is`/`errors.As` |
| Return a bare internal sentinel with no context | the caller gets `not found` with no idea of what |

Panic is acceptable for programmer error, and the convention is a `Must` prefix:
`regexp.MustCompile`, `template.Must`. Those are for package-level initialization where a
failure means the binary is broken, not the request.

## Services: the rules

| Do |
|---|
| Handle once, at the layer that can decide (usually the HTTP handler) |
| Log once, with the fully wrapped error plus a request ID |
| Map domain errors to status codes in one function at the transport boundary |
| Return a safe public message; keep internals in the log |
| Retry only retryable failures, and never a cancelled context |

A note on alerting: if every 5xx pages someone, nothing gets fixed. Decide a tolerable
error rate per endpoint (an error budget) and alert on the budget, not on individual
errors.

## The layered walkthrough

Four layers — storage, repository, service, handler — each wrapping with what it knows,
plus the status mapping and the log line. This is one runnable program using
`net/http/httptest`, so the output below is real.

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
  "strings"
)

var (
  ErrNotFound = errors.New("not found")
  ErrConflict = errors.New("already exists")
)

type ValidationError struct {
  Field  string
  Reason string
}

func (e *ValidationError) Error() string {
  return fmt.Sprintf("invalid %s: %s", e.Field, e.Reason)
}

type Retryable interface {
  Retryable() bool
}

type transientError struct{ Err error }

func (e *transientError) Error() string   { return "transient: " + e.Err.Error() }
func (e *transientError) Unwrap() error   { return e.Err }
func (e *transientError) Retryable() bool { return true }

type storage struct{}

func (storage) read(ctx context.Context, id string) (string, error) {
  switch id {
  case "1":
    return "ada", nil
  case "boom":
    return "", &transientError{Err: errors.New("dial tcp 10.0.0.5:5432: connection refused")}
  case "slow":
    return "", context.DeadlineExceeded
  }
  return "", fmt.Errorf("storage: select user %s: %w", id, ErrNotFound)
}

type repo struct{ st storage }

func (r repo) GetUser(ctx context.Context, id string) (string, error) {
  name, err := r.st.read(ctx, id)
  if err != nil {
    return "", fmt.Errorf("repo.GetUser %s: %w", id, err)
  }
  return name, nil
}

type service struct{ r repo }

func (s service) Profile(ctx context.Context, id string) (string, error) {
  if id == "" {
    return "", fmt.Errorf("service.Profile: %w", &ValidationError{Field: "id", Reason: "must not be empty"})
  }
  name, err := s.r.GetUser(ctx, id)
  if err != nil {
    return "", fmt.Errorf("service.Profile %s: %w", id, err)
  }
  return strings.ToUpper(name), nil
}

type problem struct {
  Status int    `json:"status"`
  Title  string `json:"title"`
  Detail string `json:"detail,omitempty"`
  Field  string `json:"field,omitempty"`
}

func toProblem(err error) problem {
  switch {
  case errors.Is(err, ErrNotFound):
    return problem{Status: http.StatusNotFound, Title: "not found"}
  case errors.Is(err, ErrConflict):
    return problem{Status: http.StatusConflict, Title: "already exists"}
  case errors.Is(err, context.DeadlineExceeded):
    return problem{Status: http.StatusGatewayTimeout, Title: "upstream timeout"}
  case errors.Is(err, context.Canceled):
    return problem{Status: 499, Title: "client closed request"}
  }
  var ve *ValidationError
  if errors.As(err, &ve) {
    return problem{Status: http.StatusBadRequest, Title: "invalid request", Detail: ve.Reason, Field: ve.Field}
  }
  var rt Retryable
  if errors.As(err, &rt) && rt.Retryable() {
    return problem{Status: http.StatusServiceUnavailable, Title: "temporarily unavailable"}
  }
  return problem{Status: http.StatusInternalServerError, Title: "internal error"}
}

type handler struct {
  svc service
  log *slog.Logger
}

func (h handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  reqID := r.Header.Get("X-Request-Id")
  id := strings.TrimPrefix(r.URL.Path, "/users/")

  name, err := h.svc.Profile(r.Context(), id)
  if err != nil {
    p := toProblem(err)
    if p.Status >= 500 {
      h.log.Error("request failed", "request_id", reqID, "path", r.URL.Path, "status", p.Status, "err", err)
    } else {
      h.log.Warn("request rejected", "request_id", reqID, "path", r.URL.Path, "status", p.Status, "err", err)
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(p.Status)
    _ = json.NewEncoder(w).Encode(p)
    return
  }
  w.Header().Set("Content-Type", "application/json")
  _ = json.NewEncoder(w).Encode(map[string]string{"name": name})
}

func main() {
  log := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
      if a.Key == "time" {
        return slog.Attr{}
      }
      return a
    },
  }))
  h := handler{svc: service{r: repo{st: storage{}}}, log: log}

  srv := httptest.NewServer(h)
  defer srv.Close()

  for _, path := range []string{"/users/1", "/users/99", "/users/", "/users/boom", "/users/slow"} {
    req, _ := http.NewRequest(http.MethodGet, srv.URL+path, nil)
    req.Header.Set("X-Request-Id", "req-"+strings.TrimPrefix(path, "/users/"))
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      fmt.Println("client error:", err)
      continue
    }
    body, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("GET %-13s -> %d %s", path, resp.StatusCode, body)
  }
}
```

Output:

```
GET /users/1      -> 200 {"name":"ADA"}
level=WARN msg="request rejected" request_id=req-99 path=/users/99 status=404 err="service.Profile 99: repo.GetUser 99: storage: select user 99: not found"
GET /users/99     -> 404 {"status":404,"title":"not found"}
level=WARN msg="request rejected" request_id=req- path=/users/ status=400 err="service.Profile: invalid id: must not be empty"
GET /users/       -> 400 {"status":400,"title":"invalid request","detail":"must not be empty","field":"id"}
level=ERROR msg="request failed" request_id=req-boom path=/users/boom status=503 err="service.Profile boom: repo.GetUser boom: transient: dial tcp 10.0.0.5:5432: connection refused"
GET /users/boom   -> 503 {"status":503,"title":"temporarily unavailable"}
level=ERROR msg="request failed" request_id=req-slow path=/users/slow status=504 err="service.Profile slow: repo.GetUser slow: context deadline exceeded"
GET /users/slow   -> 504 {"status":504,"title":"upstream timeout"}
```

(The `time` attribute is stripped by `ReplaceAttr` so the output is reproducible; a real
service keeps it. `httptest.NewServer` is a real HTTP server on a loopback port, so these
are real requests and responses.)

Read the `/users/boom` pair carefully — this is the entire lesson in two lines:

- **Log:** `service.Profile boom: repo.GetUser boom: transient: dial tcp 10.0.0.5:5432: connection refused`
- **Client:** `{"status":503,"title":"temporarily unavailable"}`

The log names the operation, the layer, and the database host and port. The client gets a
status and three words. Leaking `err.Error()` into the response body would publish your
internal topology — hostnames, ports, table names, query fragments — to anyone who can
send a malformed request. The request ID is how you connect the two: the client can quote
it in a support ticket and you find the one log line.

Each layer adds exactly what it knows that the layer below did not: `storage` adds the
operation, `repo` adds its method and the ID, `service` adds its method. No layer logs.
Only `handler` decides.

Note the 4xx/5xx log level split: a 404 or a validation failure is the client's problem and
belongs at `warn` or lower, while a 503 or 500 is yours and belongs at `error`. Logging
every 404 at `error` is how you train yourself to ignore the error log.

13.12 covers `slog` properly, and 18.4 covers wiring it into a service. 16.9 builds the
same mapping as a reusable `problem+json` response helper, and 16.7 puts it behind chi
middleware so each handler is three lines.

## Retry only what is retryable

The distinction that matters: a deadline means you were too slow and a retry might
succeed; a cancellation means the caller walked away and a retry is wasted work on a
response nobody will read.

```go
package main

import (
  "context"
  "errors"
  "fmt"
)

type Retryable interface {
  Retryable() bool
}

type transient struct{ Err error }

func (e *transient) Error() string   { return "transient: " + e.Err.Error() }
func (e *transient) Unwrap() error   { return e.Err }
func (e *transient) Retryable() bool { return true }

func retryable(err error) bool {
  if errors.Is(err, context.Canceled) {
    return false
  }
  if errors.Is(err, context.DeadlineExceeded) {
    return true
  }
  var r Retryable
  return errors.As(err, &r) && r.Retryable()
}

func Do(ctx context.Context, attempts int, f func(int) error) error {
  var last error
  for i := 1; i <= attempts; i++ {
    if err := ctx.Err(); err != nil {
      return fmt.Errorf("retry aborted after %d: %w", i-1, err)
    }
    last = f(i)
    if last == nil {
      fmt.Printf("  attempt %d: ok\n", i)
      return nil
    }
    fmt.Printf("  attempt %d: %v (retryable=%v)\n", i, last, retryable(last))
    if !retryable(last) {
      return fmt.Errorf("retry gave up after %d: %w", i, last)
    }
  }
  return fmt.Errorf("retry exhausted after %d: %w", attempts, last)
}

func main() {
  ctx := context.Background()

  fmt.Println("flaky then ok:")
  fmt.Println(Do(ctx, 4, func(i int) error {
    if i < 3 {
      return &transient{Err: errors.New("connection refused")}
    }
    return nil
  }))

  fmt.Println("permanent:")
  fmt.Println(Do(ctx, 4, func(int) error {
    return errors.New("invalid credentials")
  }))

  fmt.Println("always transient:")
  fmt.Println(Do(ctx, 3, func(int) error {
    return &transient{Err: errors.New("connection refused")}
  }))

  fmt.Println("canceled:")
  fmt.Println(Do(ctx, 3, func(int) error {
    return context.Canceled
  }))

  cctx, cancel := context.WithCancel(context.Background())
  cancel()
  fmt.Println("ctx already canceled:")
  fmt.Println(Do(cctx, 3, func(int) error { return nil }))
}
```

Output:

```
flaky then ok:
  attempt 1: transient: connection refused (retryable=true)
  attempt 2: transient: connection refused (retryable=true)
  attempt 3: ok
<nil>
permanent:
  attempt 1: invalid credentials (retryable=false)
retry gave up after 1: invalid credentials
always transient:
  attempt 1: transient: connection refused (retryable=true)
  attempt 2: transient: connection refused (retryable=true)
  attempt 3: transient: connection refused (retryable=true)
retry exhausted after 3: transient: connection refused
canceled:
  attempt 1: context canceled (retryable=false)
retry gave up after 1: context canceled
ctx already canceled:
retry aborted after 0: context canceled
```

Four behaviours worth naming. A transient failure retries and succeeds. An unknown error
is treated as **not** retryable — the safe default, since retrying a failed write may
duplicate it. Exhausting the attempts wraps the last error so the caller can still match
it. And `Do` checks `ctx.Err()` *before* each attempt, so an already-cancelled context
costs zero calls.

A real retry adds exponential backoff with jitter and a budget, and only applies to
idempotent operations. 12.4 covers the rate-limiting side.

## Testing errors

Assert on identity, never on text. The table test below checks a sentinel with
`errors.Is` and a typed error's field with `errors.As`.

`app.go`:

```go
package tst

import (
  "errors"
  "fmt"
)

var ErrNotFound = errors.New("not found")

type ValidationError struct {
  Field  string
  Reason string
}

func (e *ValidationError) Error() string {
  return fmt.Sprintf("invalid %s: %s", e.Field, e.Reason)
}

func Lookup(id string) error {
  switch id {
  case "1":
    return nil
  case "":
    return fmt.Errorf("lookup: %w", &ValidationError{Field: "id", Reason: "must not be empty"})
  default:
    return fmt.Errorf("lookup %s: %w", id, ErrNotFound)
  }
}
```

`app_test.go`:

```go
package tst

import (
  "errors"
  "testing"
)

func TestLookup(t *testing.T) {
  tests := []struct {
    name      string
    id        string
    wantErr   error
    wantField string
  }{
    {name: "ok", id: "1"},
    {name: "missing", id: "99", wantErr: ErrNotFound},
    {name: "empty id", id: "", wantField: "id"},
  }

  for _, tc := range tests {
    t.Run(tc.name, func(t *testing.T) {
      err := Lookup(tc.id)

      if tc.wantErr == nil && tc.wantField == "" {
        if err != nil {
          t.Fatalf("Lookup(%q) = %v, want nil", tc.id, err)
        }
        return
      }
      if err == nil {
        t.Fatalf("Lookup(%q) = nil, want error", tc.id)
      }
      if tc.wantErr != nil && !errors.Is(err, tc.wantErr) {
        t.Errorf("Lookup(%q) = %v, want errors.Is(..., %v)", tc.id, err, tc.wantErr)
      }
      if tc.wantField != "" {
        var ve *ValidationError
        if !errors.As(err, &ve) {
          t.Fatalf("Lookup(%q) = %v, want *ValidationError", tc.id, err)
        }
        if ve.Field != tc.wantField {
          t.Errorf("Field = %q, want %q", ve.Field, tc.wantField)
        }
      }
    })
  }
}
```

`go test -v ./...` output:

```
=== RUN   TestLookup
=== RUN   TestLookup/ok
=== RUN   TestLookup/missing
=== RUN   TestLookup/empty_id
--- PASS: TestLookup (0.00s)
    --- PASS: TestLookup/ok (0.00s)
    --- PASS: TestLookup/missing (0.00s)
    --- PASS: TestLookup/empty_id (0.00s)
PASS
ok  	example.com/tst	(cached)
```

The `wantErr error` field holds the *sentinel*, not a message. Adding a wrap layer in
`Lookup` cannot break this test. Compare with the string version:

```go
package tst

import "testing"

func TestLookupFragile(t *testing.T) {
  err := Lookup("99")
  if err.Error() != "lookup 99: user not found" {
    t.Errorf("got %q", err.Error())
  }
}
```

This test fails on purpose — the message says `not found`, not `user not found`. Output:

```
--- FAIL: TestLookupFragile (0.00s)
    frag_test.go:8: got "lookup 99: not found"
FAIL
FAIL	example.com/tst	2.655s
FAIL
```

One word of rewording and the test breaks while the behaviour is unchanged. That is a test
testing the wrong thing. 14.2 covers table tests in depth.

## Quick Reference

| Decision | Library | Service |
|---|---|---|
| On failure | return the wrapped error | handle it, once, at the handler |
| Logging | never | once per failed request, with the full chain + request ID |
| `panic` | only documented programmer error (`MustXxx`) | recover in middleware, return 500 |
| `log.Fatal` / `os.Exit` | never | `main` only, during startup |
| Status codes | never mentioned | one `toProblem`-style mapper at the transport boundary |
| Dependency errors | translate at your boundary | wrap with `%w` freely; it is your own code |
| Error identity | exported sentinels + typed errors, documented | matched with `errors.Is` / `errors.As` |

| Where it goes | Internal log | Public response |
|---|---|---|
| Fully wrapped `err` | yes | no |
| Hostnames, ports, queries, table names | yes | no |
| Status code | yes | yes |
| Safe title ("not found") | yes | yes |
| Validation field + reason | yes | yes |
| Request ID | yes | yes |

| Retry? | Error |
|---|---|
| yes | `Retryable() bool` reporting true, `context.DeadlineExceeded`, 5xx from an idempotent upstream |
| no | `context.Canceled`, `*ValidationError`, auth failures, any unrecognized error |

| Status | Mapped from |
|---|---|
| 400 | `*ValidationError` via `errors.As` |
| 404 | `errors.Is(err, ErrNotFound)` |
| 409 | `errors.Is(err, ErrConflict)` |
| 499 | `errors.Is(err, context.Canceled)` |
| 500 | anything unrecognized |
| 503 | a `Retryable` error |
| 504 | `errors.Is(err, context.DeadlineExceeded)` |

## Common Mistakes

**Logging an error and then returning it.** The same failure appears at every layer and
the real handling site is invisible; log only where you handle.

**Putting `err.Error()` in the HTTP response body.** It publishes hostnames, ports, and
query fragments; return a safe title plus a request ID and log the detail.

**Setting a status code on a domain error type.** It couples your business logic to HTTP;
map domain error to status in one function at the transport boundary.

**Logging 404s and validation failures at `error` level.** The error log fills with the
client's mistakes and you stop reading it; 4xx belongs at `warn` or below.

**Retrying every error.** A failed non-idempotent write may be duplicated and a cancelled
context wastes work; gate retries on an explicit retryability check.

**Asserting on `err.Error()` in tests.** One reworded message breaks a passing test while
the behaviour is unchanged; assert with `errors.Is` / `errors.As`.

## Do and Don't

| Don't | Do |
|---|---|
| `slog.Error(...)` then `return err` | return the wrapped error; log where you handle |
| `panic` on a bad request | return an error, map it to 400 |
| Put `err.Error()` in the HTTP response body | a safe title + a request ID; the detail goes to the log |
| `if strings.Contains(err.Error(), "not found")` | `errors.Is(err, ErrNotFound)` |
| Set `StatusCode` on your domain error type | map domain error to status at the transport boundary |
| `log.Fatal` in a library | return the error; `log.Fatal` only in `main` |
| Wrap third-party errors with `%w` in exported funcs | translate to your own sentinel at the boundary |
| Log every 404 at `error` level | 4xx at `warn` or `info`, 5xx at `error` |
| Retry anything that failed | retry only `Retryable()`/`DeadlineExceeded`; never `Canceled` |
| `assert err.Error() == "..."` in tests | `errors.Is` / `errors.As` |
| Return `nil` and a sentinel log line | return the error |
| One error handler per layer | one per request, at the handler |

## Checklist

Library:

- [ ] No logging, no `panic` (except documented programmer error), no `os.Exit`
- [ ] Every fallible exported function returns `error` as its last result
- [ ] Each error wrapped with context the caller cannot reconstruct
- [ ] Exported sentinels and error types are intentional and documented ("wrapping [ErrX]")
- [ ] Dependency errors translated at the boundary, not re-exported via `%w`
- [ ] Tests assert with `errors.Is` / `errors.As`

Service:

- [ ] One mapping function from domain error to status code, at the transport boundary
- [ ] One log line per failed request, with the full wrapped error and a request ID
- [ ] Public responses carry a safe title; no internal text, hostnames, or queries
- [ ] 4xx logged below `error`; 5xx logged at `error`
- [ ] Retries gated on a retryability check, with backoff, on idempotent operations only
- [ ] `context.Canceled` distinguished from `context.DeadlineExceeded`
- [ ] Goroutine errors have a declared channel or an `errgroup` (11.8)
- [ ] Alerting on an error budget, not on individual errors

## Exercises

1. Add a `POST /users` route to the walkthrough program that returns `ErrConflict` for a
   duplicate and a joined `*ValidationError` set (8.5) for a bad body. Extend `toProblem`
   to emit a `fields` map and verify every status with `httptest`.
2. Extract `toProblem` into its own package with no `net/http` import, returning a status
   `int` and a title. Argue in two sentences whether that is an improvement.
3. Write a test for the walkthrough handler using `httptest.NewRecorder` instead of
   `NewServer`, asserting status and decoded JSON body for all five cases. No string
   comparison on error messages.
4. Add exponential backoff with jitter to `Do`, bounded by the context deadline, and make
   it stop early when the remaining deadline is shorter than the next sleep. Prove both
   behaviours with a test using `context.WithTimeout`.
5. Audit a service you have written in another language: count how many times a single
   failure is logged on its way out. Rewrite one path so it is logged exactly once, and
   note what information you had to add to the error to keep the log line as useful.

---

[← 8.5 errors.Join and Multiple Errors](05-join.md) · [Course Index](../README.md) · [9.1 Type Parameters on Functions →](../09-generics/01-type-parameters.md)
