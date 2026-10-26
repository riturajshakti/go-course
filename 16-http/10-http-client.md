[← 16.9 HTTP Errors and Problem Responses](09-error-responses.md) · [Course Index](../README.md) · [16.11 Testing Handlers with httptest →](11-httptest.md)

# 16.10 — The net/http Client

## What You'll Learn

- `http.Get` and `http.DefaultClient` have **no timeout at all** — the single most common production outage in Go services
- Go does **not** return an error for 4xx/5xx; you must check `resp.StatusCode` yourself
- Not closing — and not **draining** — `resp.Body` leaks goroutines and destroys connection reuse, measured here for real
- `RoundTripper` as client-side middleware: auth injection and retries with backoff
- The idiomatic typed API client: a struct holding `baseURL` plus one `*http.Client`

## Coming From Another Language

| Other | Go |
|---|---|
| `fetch()` rejects only on network failure | `Do` returns `err == nil` for a 500 too — same trap, but you *must* check `StatusCode` |
| `axios` throws on 4xx/5xx by default | nothing throws; you raise your own error |
| `requests.get(url, timeout=5)` | `&http.Client{Timeout: 5*time.Second}` — there is **no default** |
| Response bodies are garbage-collected | `resp.Body` is a live connection you must `Close()` |
| `axios.interceptors` | `http.RoundTripper` wrapping another `RoundTripper` |
| A new client per call is cheap | a new client per call destroys the connection pool |

## http.Get and the missing timeout

```go
fmt.Println("http.DefaultClient.Timeout =", http.DefaultClient.Timeout)
```

Output:

```
http.DefaultClient.Timeout = 0s
http.DefaultTransport is *http.Transport: true
```

`http.Get`, `http.Post`, `http.Head`, and `http.PostForm` all use `http.DefaultClient`,
whose `Timeout` is **zero — meaning no timeout, ever**. A dependency that accepts your
connection and then stops responding will hold your goroutine until the process restarts.
Do it under load and you exhaust your goroutines, your file descriptors, and then your
service.

Worse, `DefaultClient` is a *package-level variable*: any library in your dependency tree
can mutate it, and your "fix" of setting `http.DefaultClient.Timeout` affects every other
caller in the process.

The whole fix is one struct literal:

```go
client := &http.Client{Timeout: 10 * time.Second}
```

Here it is against a server that never responds:

```go
hang := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
  <-r.Context().Done()
}))
defer hang.Close()

timed := &http.Client{Timeout: 150 * time.Millisecond}
start := time.Now()
_, err := timed.Get(hang.URL)

var ue *url.Error
// ... print err, errors.As(err, &ue), ue.Timeout(), ue.Op
```

Output (server URL normalized to `SRV`):

```
client with Timeout=150ms against a hanging server:
  returned after >100ms: true
  err: Get "SRV": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
  errors.As(*url.Error): true
  ue.Timeout(): true  ue.Op="Get"
  errors.Is(os.ErrDeadlineExceeded): false
```

`Client.Timeout` bounds the **whole exchange**: connection, TLS handshake, request write,
response headers, and **reading the body**. That last part is the catch — if you are
streaming a large download or an SSE feed, a client-wide timeout will cut it off mid-body. For
those, leave `Client.Timeout` at zero and use a per-request `context.WithTimeout` that you
cancel after the headers arrive, or set the narrower `Transport` timeouts below.

Every client error is wrapped in `*url.Error`, which has `Timeout()` and `Temporary()`
methods and an `Op` field. Note `errors.Is(err, os.ErrDeadlineExceeded)` is **false** for a
`Client.Timeout` — use `ue.Timeout()` or `errors.Is(err, context.DeadlineExceeded)` instead.

## Go does not error on 4xx or 5xx

```go
for _, p := range []string{"/ok", "/404", "/500"} {
  resp, err := timed.Get(api.URL + p)
  if err != nil {
    fmt.Printf("  %-6s err=%v\n", p, err)
    continue
  }
  b, _ := io.ReadAll(resp.Body)
  resp.Body.Close()
  fmt.Printf("  %-6s err=%v  StatusCode=%d  body=%q\n", p, err, resp.StatusCode, b)
}
```

Output:

```
Go does NOT error on 4xx/5xx:
  /ok    err=<nil>  StatusCode=200  body="ok"
  /404   err=<nil>  StatusCode=404  body="nope\n"
  /500   err=<nil>  StatusCode=500  body="boom\n"
```

`err` is `nil` for all three. A non-nil error means the exchange *failed* — DNS, connection
refused, timeout, TLS. A 500 is a perfectly successful HTTP exchange that happens to carry
bad news.

This is **the** most common client bug in Go. The shape that goes wrong:

```go
resp, err := client.Do(req)
if err != nil {
  return err
}
defer resp.Body.Close()
var out Item
return json.NewDecoder(resp.Body).Decode(&out)   // decoding a 500's error body!
```

That decodes your server's `{"error":"..."}` into an `Item` and returns a zero-valued item
with **no error**. The check is mandatory:

```go
if resp.StatusCode < 200 || resp.StatusCode >= 300 {
  b, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
  return &APIStatusError{StatusCode: resp.StatusCode, Body: string(b)}
}
```

`io.LimitReader` there is deliberate: the error body comes from a *remote* server you do not
control, so bound it exactly as you bound inbound bodies (16.4).

## Closing and draining the body

`resp.Body` is a live reader over a network connection. Two obligations, and the second is
less known.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "runtime"
  "sort"
  "strings"
  "sync"
  "time"
)

func main() {
  var mu sync.Mutex
  var conns []string
  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    mu.Lock()
    conns = append(conns, r.RemoteAddr)
    mu.Unlock()
    io.WriteString(w, strings.Repeat("x", 2000))
  }))
  defer srv.Close()

  fmt.Println("=== leaking: never closing resp.Body ===")
  runtime.GC()
  before := runtime.NumGoroutine()
  leaky := &http.Client{Transport: &http.Transport{}}
  for range 20 {
    resp, err := leaky.Get(srv.URL)
    if err != nil {
      panic(err)
    }
    _ = resp
  }
  time.Sleep(200 * time.Millisecond)
  runtime.GC()
  leakedGoroutines := runtime.NumGoroutine() - before
  mu.Lock()
  leakedConns := len(uniq(conns))
  conns = nil
  mu.Unlock()
  fmt.Printf("20 requests, bodies never closed:\n")
  fmt.Printf("  goroutines above baseline: %d\n", leakedGoroutines)
  fmt.Printf("  distinct server-side connections used: %d\n", leakedConns)
  leaky.CloseIdleConnections()

  fmt.Println("\n=== correct: drain and close ===")
  runtime.GC()
  before = runtime.NumGoroutine()
  good := &http.Client{Transport: &http.Transport{}}
  for range 20 {
    resp, err := good.Get(srv.URL)
    if err != nil {
      panic(err)
    }
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
  }
  time.Sleep(200 * time.Millisecond)
  runtime.GC()
  fmt.Printf("20 requests, drained and closed:\n")
  fmt.Printf("  goroutines above baseline: %d\n", runtime.NumGoroutine()-before)
  mu.Lock()
  fmt.Printf("  distinct server-side connections used: %d\n", len(uniq(conns)))
  conns = nil
  mu.Unlock()
  good.CloseIdleConnections()

  fmt.Println("\n=== close without draining ===")
  partial := &http.Client{Transport: &http.Transport{}}
  for range 20 {
    resp, _ := partial.Get(srv.URL)
    buf := make([]byte, 10)
    resp.Body.Read(buf)
    resp.Body.Close()
  }
  time.Sleep(200 * time.Millisecond)
  mu.Lock()
  fmt.Printf("20 requests, only 10 bytes read then closed:\n")
  fmt.Printf("  distinct server-side connections used: %d\n", len(uniq(conns)))
  mu.Unlock()
  partial.CloseIdleConnections()
}

func uniq(s []string) []string {
  m := map[string]bool{}
  for _, v := range s {
    m[v] = true
  }
  out := make([]string, 0, len(m))
  for k := range m {
    out = append(out, k)
  }
  sort.Strings(out)
  return out
}
```

Output:

```
=== leaking: never closing resp.Body ===
20 requests, bodies never closed:
  goroutines above baseline: 60
  distinct server-side connections used: 20

=== correct: drain and close ===
20 requests, drained and closed:
  goroutines above baseline: 3
  distinct server-side connections used: 1

=== close without draining ===
20 requests, only 10 bytes read then closed:
  distinct server-side connections used: 2
```

Three measurements worth memorizing. (Goroutine counts can wobble by a couple because the GC
and the server's own accept loop are running; the magnitudes are the point.)

**Never closing: 60 leaked goroutines and 20 connections.** Each unclosed body pins a
read-loop goroutine and a write-loop goroutine on the client plus a handler goroutine on the
server, and the connection can never return to the pool. 20 requests, 20 TCP connections —
in a service doing hundreds per second, that is file-descriptor exhaustion within minutes.
`resp.Body` is **not** garbage-collected in any useful sense; the goroutines holding it keep
it alive.

**Draining and closing: 1 connection for 20 requests.** The connection went back to the idle
pool each time and was reused. That is a 20× reduction in TCP handshakes, and on HTTPS a 20×
reduction in TLS handshakes.

**Closing without draining: 2 connections.** This is the subtle one. `Close()` on a
partially-read body tells the transport the remaining bytes are unwanted, and since it cannot
know where the next response begins on that connection, it **discards the connection**. So
the ritual is:

```go
defer func() {
  io.Copy(io.Discard, resp.Body)
  resp.Body.Close()
}()
```

(It used 2 rather than 20 here only because the response is small enough to already sit in
the transport's read buffer. With a large body you would see 20.)

A plain `defer resp.Body.Close()` is correct and sufficient when you read the whole body —
`io.ReadAll` or a full `json.Decoder.Decode` already drains it. Add the explicit drain when
you might return early, which in practice means in your shared `do` helper.

And `resp.Body` is **never nil** when `err == nil`, so `defer resp.Body.Close()` placed
immediately after the error check is always safe.

## Always use NewRequestWithContext

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, body)
if err != nil {
  return err
}
req.Header.Set("Accept", "application/json")
resp, err := client.Do(req)
```

`http.NewRequest` (no context) is equivalent to passing `context.Background()`, which means
the request cannot be cancelled. In a server handler you always have `r.Context()`, and
passing it means a client hanging up on *you* also cancels the outbound call you made on
their behalf — the payoff example in 16.12. `go vet` does not catch this; make it a review
rule ([11.6](../11-sync-context/06-context-cancel.md)).

`body` is an `io.Reader`. Pass `nil` for GET. If you pass a `*bytes.Reader`,
`*bytes.Buffer`, or `*strings.Reader`, Go sets `ContentLength` *and* `GetBody`
automatically — the latter is what makes redirects and retries able to replay the body.

## The Transport, and connection reuse

```go
dt := http.DefaultTransport.(*http.Transport)
// print every field
```

Output:

```
http.DefaultTransport defaults:
  MaxIdleConns:           100
  MaxIdleConnsPerHost:    0 (0 means DefaultMaxIdleConnsPerHost = 2 )
  MaxConnsPerHost:        0 (0 = unlimited)
  IdleConnTimeout:        1m30s
  TLSHandshakeTimeout:    10s
  ExpectContinueTimeout:  1s
  ResponseHeaderTimeout:  0s (0 = none!)
  DisableKeepAlives:      false
  ForceAttemptHTTP2:      true
```

**`MaxIdleConnsPerHost` is 2.** That is the production gotcha. If your service talks to one
upstream at 500 requests/second, only 2 connections are kept idle; the rest are opened and
immediately discarded after use. You get the TCP and TLS handshake cost on nearly every
request, and it shows up as mysterious latency that disappears in load tests with low
concurrency.

A transport tuned for a service that hammers a few upstreams:

```go
transport := &http.Transport{
  MaxIdleConns:          200,
  MaxIdleConnsPerHost:   100,
  MaxConnsPerHost:       200,
  IdleConnTimeout:       90 * time.Second,
  TLSHandshakeTimeout:   5 * time.Second,
  ExpectContinueTimeout: 1 * time.Second,
  ResponseHeaderTimeout: 10 * time.Second,
  DialContext: (&net.Dialer{
    Timeout:   5 * time.Second,
    KeepAlive: 30 * time.Second,
  }).DialContext,
}
client := &http.Client{Transport: transport, Timeout: 30 * time.Second}
```

| Field | Bounds | Note |
|---|---|---|
| `MaxIdleConns` | idle connections across all hosts | 100 default is usually fine |
| `MaxIdleConnsPerHost` | idle connections per host | **default 2** — raise it for a busy upstream |
| `MaxConnsPerHost` | total (not just idle) per host | 0 = unlimited; set it to protect an upstream |
| `IdleConnTimeout` | how long an idle connection survives | 90s; keep it under the upstream's own idle timeout |
| `TLSHandshakeTimeout` | the TLS handshake | 10s default |
| `ExpectContinueTimeout` | wait for `100 Continue` after `Expect: 100-continue` | 1s |
| `ResponseHeaderTimeout` | request write → response headers | **0 by default**; the right knob for streaming clients, since it excludes the body |
| `DisableKeepAlives` | set `true` to use one connection per request | only for debugging or a one-shot CLI |
| `DisableCompression` | stop automatic gzip | Go transparently gzips/ungzips by default |

`ResponseHeaderTimeout` is the field people are looking for when they ask "how do I time out
a request without cutting off a large download". It bounds everything up to the headers and
nothing after.

## Reuse ONE client

**Create your `*http.Client` once** and share it. A client is safe for concurrent use by
multiple goroutines — that is documented and intended. Creating one per request creates a new
`Transport` with a new empty connection pool, so you get zero reuse and the pool's
connections are never collected until `IdleConnTimeout` expires. The 20-connections-for-20-
requests result above is exactly what per-request clients produce, permanently.

The corollary: if you *must* make a short-lived client (a test, a one-shot migration), call
`client.CloseIdleConnections()` when done, as the example above does.

## RoundTripper: client-side middleware

`http.RoundTripper` is a one-method interface — the client-side mirror of `http.Handler`:

```go
type RoundTripper interface {
  RoundTrip(*Request) (*Response, error)
}
```

Wrapping one gives you interceptors. Two genuinely useful ones.

**Auth and user-agent injection**, so no call site can forget them:

```go
type authTransport struct {
  base  http.RoundTripper
  token string
}

func (t *authTransport) RoundTrip(r *http.Request) (*http.Response, error) {
  clone := r.Clone(r.Context())
  clone.Header.Set("Authorization", "Bearer "+t.token)
  clone.Header.Set("User-Agent", "go-course-client/1.0")
  return t.base.RoundTrip(clone)
}
```

`r.Clone(r.Context())` is **mandatory**. The documented contract is that `RoundTrip` must not
modify the request — the caller may still be holding it, and the retry logic below needs the
original intact. Mutating `r.Header` directly is a race.

**Retries with exponential backoff** ([12.4](../12-concurrency-patterns/04-rate-limiting.md)):

```go
type retryTransport struct {
  base     http.RoundTripper
  attempts int
  backoff  time.Duration
  sleep    func(time.Duration)
}

func (t *retryTransport) RoundTrip(r *http.Request) (*http.Response, error) {
  if r.Body != nil && r.GetBody == nil {
    return t.base.RoundTrip(r)
  }
  var lastErr error
  for attempt := 1; attempt <= t.attempts; attempt++ {
    req := r.Clone(r.Context())
    if r.GetBody != nil {
      b, err := r.GetBody()
      if err != nil {
        return nil, err
      }
      req.Body = b
    }
    resp, err := t.base.RoundTrip(req)
    if err == nil && resp.StatusCode < 500 && resp.StatusCode != 429 {
      return resp, nil
    }
    if err == nil {
      io.Copy(io.Discard, resp.Body)
      resp.Body.Close()
      lastErr = fmt.Errorf("status %d", resp.StatusCode)
    } else {
      if r.Context().Err() != nil {
        return nil, err
      }
      lastErr = err
    }
    if attempt < t.attempts {
      t.sleep(t.backoff * time.Duration(1<<(attempt-1)))
    }
  }
  return nil, fmt.Errorf("after %d attempts: %w", t.attempts, lastErr)
}
```

The four rules that make a retry safe, all visible there:

**Only retry when the body can be replayed.** `r.GetBody == nil` with a non-nil body means
the body is a one-shot stream, so retrying would send an empty or truncated request. Bail out
and do a single attempt.

**Only retry retryable failures.** 5xx, 429, and transport errors. A 400 or 404 will be a
400 or 404 forever, and retrying a 409 can double-apply a change.

**Drain and close the discarded response.** Otherwise the retry loop itself leaks connections
— the exact bug from the measurement above.

**Stop immediately if the context is done.** Otherwise your "resilience" keeps a cancelled
request alive and your backoff outlives the caller.

The *fifth* rule is not in the code because it is an architectural one: **only retry
idempotent requests.** GET, HEAD, PUT and DELETE are idempotent by definition; POST is not. A
retried POST can create two orders. Either restrict retries by method, or require an
`Idempotency-Key` header that the server deduplicates on. Real production code also adds
**jitter** to the backoff so a thousand clients do not retry in lockstep.

## Redirects and cookies

```go
jar, _ := cookiejar.New(nil)
tracking := &http.Client{
  Timeout: 2 * time.Second,
  Jar:     jar,
  CheckRedirect: func(req *http.Request, via []*http.Request) error {
    fmt.Printf("  redirect #%d -> %s\n", len(via), req.URL.Path)
    if len(via) >= 5 {
      return errors.New("too many redirects")
    }
    return nil
  },
}
```

Against a server that sets a cookie then redirects twice:

Output:

```
following redirects with a cookie jar:
  redirect #1 -> /hop1
  redirect #2 -> /final
  final status=200 OK final URL path=/final
  body="arrived; cookie=session=abc err=<nil>"
  jar now holds: [session=abc]

with ErrUseLastResponse: status=302 Found Location="/hop1"
```

**Redirects are followed automatically**, up to 10 by default. `resp.Request.URL` is the URL
that finally answered, not the one you asked for — useful, and a trap if you assumed
otherwise.

**`CheckRedirect`** is called before each hop with the next request and the chain so far. Return
an error to stop; return `http.ErrUseLastResponse` to stop and hand back the 3xx response
*unmodified*, which is how you inspect a `Location` header (and what every `noFollow` client
in this module does).

**`Jar`** makes the client stateful. The cookie set on `/start` was sent to `/final`
automatically. `cookiejar.New(nil)` is in-memory and respects path and domain scoping. Without
a jar, cookies are simply dropped — the client is stateless by default, which for an API
client is what you want.

Note: Go strips the `Authorization` header on a redirect to a *different* host, which is
correct and occasionally surprising.

## The idiomatic typed API client

Put it all together: one struct, one `*http.Client`, typed methods, and one shared `do`
helper that handles encoding, the status check, draining, and decoding.

```go
type APIClient struct {
  baseURL string
  http    *http.Client
}

func NewAPIClient(baseURL, token string) *APIClient {
  tr := &authTransport{base: http.DefaultTransport, token: token}
  rt := &retryTransport{base: tr, attempts: 3, backoff: 100 * time.Millisecond,
    sleep: time.Sleep}
  return &APIClient{
    baseURL: strings.TrimRight(baseURL, "/"),
    http:    &http.Client{Timeout: 5 * time.Second, Transport: rt},
  }
}

type APIStatusError struct {
  StatusCode int
  Body       string
}

func (e *APIStatusError) Error() string {
  return fmt.Sprintf("api returned %d: %s", e.StatusCode, e.Body)
}

func (c *APIClient) do(ctx context.Context, method, path string, in, out any) error {
  var body io.Reader
  if in != nil {
    buf, err := json.Marshal(in)
    if err != nil {
      return fmt.Errorf("encoding request: %w", err)
    }
    body = bytes.NewReader(buf)
  }
  req, err := http.NewRequestWithContext(ctx, method, c.baseURL+path, body)
  if err != nil {
    return fmt.Errorf("building request: %w", err)
  }
  if in != nil {
    req.Header.Set("Content-Type", "application/json")
  }
  req.Header.Set("Accept", "application/json")

  resp, err := c.http.Do(req)
  if err != nil {
    return fmt.Errorf("%s %s: %w", method, path, err)
  }
  defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
  }()

  if resp.StatusCode >= 400 {
    b, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
    return &APIStatusError{StatusCode: resp.StatusCode, Body: strings.TrimSpace(string(b))}
  }
  if out == nil {
    return nil
  }
  if err := json.NewDecoder(resp.Body).Decode(out); err != nil {
    return fmt.Errorf("decoding response: %w", err)
  }
  return nil
}

func (c *APIClient) GetItem(ctx context.Context, id int) (Item, error) {
  var it Item
  err := c.do(ctx, http.MethodGet, fmt.Sprintf("/items/%d", id), nil, &it)
  return it, err
}

func (c *APIClient) ListItems(ctx context.Context, q url.Values) ([]Item, error) {
  var out []Item
  path := "/items"
  if len(q) > 0 {
    path += "?" + q.Encode()
  }
  err := c.do(ctx, http.MethodGet, path, nil, &out)
  return out, err
}

func (c *APIClient) CreateItem(ctx context.Context, it Item) (Item, error) {
  var created Item
  err := c.do(ctx, http.MethodPost, "/items", it, &created)
  return created, err
}
```

Exercised against a real `httptest.Server` that echoes what it received, with one endpoint
that fails twice before succeeding. The `sleep` function is injected so the backoff delays are
recorded rather than actually waited on, making the run both fast and deterministic.

Output (the `[retry]` lines are the retry transport's own tracing):

```
=== GetItem(1)
  SERVER saw GET /items/1 auth="Bearer secret-token" ua="go-course-client/1.0" accept="application/json"
    [retry] attempt 1 -> 200, done
  -> {ID:1 Name:widget Price:9.99} err=<nil>

=== ListItems(limit=10)
  SERVER saw GET /items?limit=10 auth="Bearer secret-token" ua="go-course-client/1.0" accept="application/json"
    [retry] attempt 1 -> 200, done
  -> [{ID:1 Name:widget Price:9.99}] err=<nil>

=== CreateItem
  SERVER saw POST /items auth="Bearer secret-token" ua="go-course-client/1.0" accept="application/json"
  SERVER body={"id":0,"name":"new","price":1.5}
    [retry] attempt 1 -> 201, done
  -> {ID:7 Name:new Price:1.5} err=<nil>

=== GetItem(99) -- 404 is an error WE raise
  SERVER saw GET /items/99 auth="Bearer secret-token" ua="go-course-client/1.0" accept="application/json"
    [retry] attempt 1 -> 404, done
  err=api returned 404: {"error":"not found"}
  errors.As(*APIStatusError)=true status=404

=== /flaky -- retry with backoff
  SERVER saw GET /flaky auth="Bearer secret-token" ua="go-course-client/1.0" accept="application/json"
    [retry] attempt 1 -> 503, retrying
  SERVER saw GET /flaky auth="Bearer secret-token" ua="go-course-client/1.0" accept="application/json"
    [retry] attempt 2 -> 503, retrying
  SERVER saw GET /flaky auth="Bearer secret-token" ua="go-course-client/1.0" accept="application/json"
    [retry] attempt 3 -> 200, done
  -> {ID:3 Name:eventually Price:2} err=<nil>
  backoff delays requested: [10ms 20ms]
```

What the output proves:

**The auth transport injected headers on every call** — no method had to remember.

**`/items/99` became a typed Go error** carrying the status and the remote body, recoverable
with `errors.As`. That is the interface your calling code wants, not a `*http.Response`.

**The retry transport replayed the 503 twice and succeeded on attempt 3**, with delays
`[10ms 20ms]` — exponential doubling, and recorded rather than slept.

**`CreateItem` sent `"id":0`** because `Item.ID` has no `omitempty`. A real client would use a
separate request type without an `ID` field — the same transport/domain separation as 16.8.

Three structural notes on the shape itself:

**`baseURL` is a field, not a parameter**, so tests point it at `srv.URL` and production
points it at the real host — the simplest possible seam (16.11).

**One `do` helper carries the invariants.** The status check, the drain, the context, and the
`Content-Type` are written once. A new endpoint is three lines and cannot forget them.

**The methods take `ctx` first.** Always — so cancellation flows from your handler through
your client to the upstream (16.12).

For a multipart upload from the client, build the body with `multipart.Writer` into a
`bytes.Buffer` and use `mw.FormDataContentType()` as the `Content-Type` — the same code as the
test-side body builder in 16.4.

## Quick Reference

| Call | Note |
|---|---|
| `http.Get/Post/Head/PostForm` | use `DefaultClient` — **no timeout**; avoid |
| `&http.Client{Timeout: d}` | bounds the whole exchange **including the body** |
| `http.NewRequestWithContext(ctx, m, u, body)` | **always**, not `http.NewRequest` |
| `client.Do(req)` | `err == nil` for 4xx/5xx — check `resp.StatusCode` |
| `resp.Body.Close()` | mandatory; `resp.Body` is never nil when `err == nil` |
| `io.Copy(io.Discard, resp.Body)` before close | required for connection reuse on an early return |
| `client.CloseIdleConnections()` | for a short-lived client |
| `resp.Request.URL` | where the request *ended* after redirects |

| Response field | Holds |
|---|---|
| `StatusCode` / `Status` | `404` / `"404 Not Found"` |
| `Header` | canonical keys (16.4) |
| `ContentLength` | `-1` if unknown/chunked |
| `TransferEncoding` | `["chunked"]` when streamed |
| `Body` | `io.ReadCloser` — close it |
| `Uncompressed` | true if Go transparently ungzipped it |

| Error check | For |
|---|---|
| `errors.As(err, &urlErr)` | `*url.Error` wraps every client error; `.Timeout()`, `.Op` |
| `errors.Is(err, context.DeadlineExceeded)` | a context or `Client.Timeout` deadline |
| `errors.Is(err, context.Canceled)` | the caller gave up |
| `errors.As(err, &netErr)` / `net.Error` | `.Timeout()` on transport errors |

## Common Mistakes

**Using `http.Get` or `http.DefaultClient`.** There is no timeout, so one unresponsive
upstream pins goroutines until restart; construct `&http.Client{Timeout: ...}`.

**Not checking `resp.StatusCode`.** Go returns `err == nil` for a 500, so you decode the error
body into your success type and return a zero value with no error.

**Not closing `resp.Body`.** Each one pins goroutines and a connection — 20 requests leaked 60
goroutines and 20 connections in the measurement above.

**Closing without draining on an early return.** The transport discards the connection rather
than pooling it, so you lose all reuse; `io.Copy(io.Discard, resp.Body)` first.

**Creating a client per request.** Each gets a fresh empty connection pool, so nothing is ever
reused; build one client and share it.

**Using `http.NewRequest` instead of `NewRequestWithContext`.** The call cannot be cancelled,
so a client disconnect leaves your outbound request running (16.12).

**Mutating the request inside `RoundTrip`.** The contract forbids it and the caller may still
hold it; `r.Clone(r.Context())` first.

**Retrying a POST.** It is not idempotent, so a retry can create a duplicate record; restrict
retries by method or require an idempotency key.

**Leaving `MaxIdleConnsPerHost` at its default of 2.** A busy upstream then pays a handshake
on nearly every request; raise it to match your concurrency.

## Exercises

1. Write a client that fetches a 10 MB response and bound it so the *headers* must arrive in 2
   seconds but the body may take 60. Show which of `Client.Timeout`,
   `Transport.ResponseHeaderTimeout` and `context.WithTimeout` achieves it, and capture the
   real error from each of the three when the server stalls at each stage.
2. Reproduce the leak measurement with a 1 MB response body and report the connection counts
   for all three cases. Explain why "close without draining" now behaves differently from the
   2000-byte case in this lesson.
3. Add jitter to `retryTransport` and honour a `Retry-After` header on a 429 (16.9). Verify
   against a server that sends `Retry-After: 2` that the delay used is 2 seconds, not the
   backoff — using an injected clock so the test is fast.
4. Write a `RoundTripper` that records every request and response to a slice for assertions,
   and one that returns canned responses from a map without any network at all. Use the second
   to unit-test your API client's error classification with no server.
5. Build a client method that uploads a file as multipart and streams it with `io.Pipe` instead
   of buffering, then prove with a 50 MB file and `runtime.ReadMemStats` that peak memory stays
   flat. State why `GetBody` is nil in that case and what it means for retries.

---

[← 16.9 HTTP Errors and Problem Responses](09-error-responses.md) · [Course Index](../README.md) · [16.11 Testing Handlers with httptest →](11-httptest.md)
