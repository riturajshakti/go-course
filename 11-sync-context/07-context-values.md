[← 11.6 context: Cancellation and Timeouts](06-context-cancel.md) · [Course Index](../README.md) · [11.8 errgroup →](08-errgroup.md)

# 11.7 — context Values and Propagation

## What You'll Learn

- Plain `string` keys collide silently across packages — the real bug, then the `type ctxKey struct{}` fix
- Values are immutable and layered: `WithValue` shadows, never mutates, and lookup is O(depth)
- Typed getter/setter helpers are the only correct public API for a context value
- Request ID, trace ID, authenticated user, logger: yes. Config, options, dependencies: no.

## Coming From Another Language

| Other | Go |
|---|---|
| Python `contextvars.ContextVar` (typed, per-task) | `context.WithValue`, untyped, explicitly threaded |
| Node `AsyncLocalStorage` (implicit, follows async calls) | nothing implicit — the `ctx` argument *is* the mechanism |
| Java `ThreadLocal` / MDC | no thread-local storage exists in Go at all |
| Rust `tracing::Span` with structured fields | `context.WithValue` plus a logger you build from the ctx |
| Express `req.user = ...` | `r.WithContext(WithUser(r.Context(), u))` |

Go has no goroutine-local storage, by design, and no plans to add it. There is no way to
ask "what request am I serving" from deep in a call stack — the answer has to be handed
to you. `context.WithValue` is the sanctioned channel for that, and it is deliberately
awkward: the key and value are both `any`, so the compiler checks nothing, and the only
way a value reaches a function is if every function between here and there accepted a
`ctx` parameter. That awkwardness is the feature. It means data flow stays visible in
signatures, and it is why the Go community's rule is so narrow: a context carries
*request-scoped metadata that crosses API boundaries*, and nothing else.

## WithValue, and the collision bug

`context.WithValue(parent, key, val)` returns a child carrying one extra key. Keys are
compared with `==` on the `any` interface, which means **both the dynamic type and the
value** must match. Two packages using `"user"` as a key are using the same key.

```go
package main

import (
  "context"
  "fmt"
)

func authMiddleware(ctx context.Context) context.Context {
  return context.WithValue(ctx, "user", "alice")
}

func tracingMiddleware(ctx context.Context) context.Context {
  return context.WithValue(ctx, "user", "svc-account-7")
}

func main() {
  ctx := context.Background()
  ctx = authMiddleware(ctx)
  fmt.Println("after auth:    ", ctx.Value("user"))
  ctx = tracingMiddleware(ctx)
  fmt.Println("after tracing: ", ctx.Value("user"))

  v := ctx.Value("user")
  name, ok := v.(string)
  fmt.Printf("handler believes the user is %q (ok=%v)\n", name, ok)
  fmt.Println("missing key:", ctx.Value("tenant"))
  n, ok := ctx.Value("user").(int)
  fmt.Println("wrong type assertion:", n, ok)
}
```

Output:

```
after auth:     alice
after tracing:  svc-account-7
handler believes the user is "svc-account-7" (ok=true)
missing key: <nil>
wrong type assertion: 0 false
```

That is the bug in full. The auth middleware authenticated `alice`; the tracing
middleware — written by a different team, in a different package, that has never heard of
the auth middleware — stored its service account under the same string, and the handler
now runs with the wrong identity. Nothing failed. No compile error, no panic, no log
line. The type assertion even succeeds, because both values happen to be strings.

Swap the order of the two middlewares and the bug disappears, which is the worst
possible property for a bug to have.

The other two lines show the lookup semantics: a missing key gives `nil` (so the
comma-ok form is mandatory), and a wrong-typed assertion gives the zero value and
`false` rather than panicking.

## The fix: an unexported key type

Declare a package-local unexported type and use it as the key. Because the type is
unexported, no other package can construct a value of it, so no other package can collide
with you — the type system does the work.

```go
package main

import (
  "context"
  "fmt"
)

type userKey struct{}

type traceKey struct{}

func WithUser(ctx context.Context, name string) context.Context {
  return context.WithValue(ctx, userKey{}, name)
}

func UserFrom(ctx context.Context) (string, bool) {
  name, ok := ctx.Value(userKey{}).(string)
  return name, ok
}

func WithTrace(ctx context.Context, id string) context.Context {
  return context.WithValue(ctx, traceKey{}, id)
}

func TraceFrom(ctx context.Context) string {
  id, _ := ctx.Value(traceKey{}).(string)
  return id
}

func main() {
  ctx := WithUser(context.Background(), "alice")
  ctx = WithTrace(ctx, "trace-abc")

  fmt.Println(UserFrom(ctx))
  fmt.Println("trace:", TraceFrom(ctx))

  fmt.Println(UserFrom(context.Background()))
  fmt.Printf("missing trace is the zero value: %q\n", TraceFrom(context.Background()))

  fmt.Println("userKey{} == userKey{}:", userKey{} == userKey{})
  fmt.Println("distinct types never collide:", any(userKey{}) == any(traceKey{}))

  shadowed := WithUser(ctx, "bob")
  fmt.Println("shadowed:", must(UserFrom(shadowed)))
  fmt.Println("parent is unchanged:", must(UserFrom(ctx)))
}

func must(s string, ok bool) string {
  if !ok {
    panic("missing")
  }
  return s
}
```

Output:

```
alice true
trace: trace-abc
 false
missing trace is the zero value: ""
userKey{} == userKey{}: true
distinct types never collide: false
shadowed: bob
parent is unchanged: alice
```

`type userKey struct{}` costs zero bytes, is comparable (so `userKey{} == userKey{}` is
true), and is impossible to forge from outside the package. `any(userKey{}) ==
any(traceKey{})` is false even though both are empty structs, because interface equality
compares the dynamic type first. Two independent packages can both declare
`type ctxKey struct{}` and they still do not collide.

The alternatives and why they are worse: `type ctxKey string` with constants is forgeable
if the type is exported and wastes a string comparison; `type ctxKey int` with
`iota` works but gives useless `%v` output; a package-level `var userKey = &struct{}{}`
works (pointer identity) but is mutable. `struct{}{}` is the common idiom, and
`type ctxKey int` with named constants is the other acceptable one when you have many
keys in one package.

Idiomatic: one unexported empty-struct type per key, or one `type ctxKey int` with
`iota` constants if there are several.

## Typed helpers are the API

Never export the key, and never make callers write a type assertion. Export a setter and
a getter; they are the only places the `any` boundary is crossed, so they are the only
places a type error can hide.

```go
type ctxKey struct{}

func WithUser(ctx context.Context, u *User) context.Context {
  return context.WithValue(ctx, ctxKey{}, u)
}

func UserFrom(ctx context.Context) (*User, bool) {
  u, ok := ctx.Value(ctxKey{}).(*User)
  return u, ok
}
```

Two getter shapes, both legitimate:

- **comma-ok** — `UserFrom(ctx) (*User, bool)` — when absence is a normal case the caller
  must handle (an unauthenticated request).
- **zero value** — `TraceFrom(ctx) string` — when absence is harmless and the zero value
  is a fine substitute (an empty trace ID just logs as empty).

A third shape, `MustUserFrom(ctx) *User` that panics, is reasonable in a handler that
sits behind a middleware guaranteeing the value — but only if that invariant is enforced
by routing, not by hope.

## Values are immutable and layered

`WithValue` never modifies anything. It allocates a new `valueCtx` wrapping the parent,
so lookup walks the chain from the newest link to the oldest and returns the first match.

```go
package main

import (
  "context"
  "fmt"
)

type key int

func main() {
  ctx := context.Background()
  for i := range 5 {
    ctx = context.WithValue(ctx, key(i), fmt.Sprintf("v%d", i))
  }
  fmt.Println(ctx)

  for i := range 6 {
    fmt.Printf("Value(key(%d)) = %v\n", i, ctx.Value(key(i)))
  }

  cctx, cancel := context.WithCancel(ctx)
  cancel()
  fmt.Println("after cancel, Err:", cctx.Err())
  fmt.Println("values survive cancellation:", cctx.Value(key(0)), cctx.Value(key(4)))

  vctx := context.WithValue(cctx, key(99), "added after cancel")
  fmt.Println("WithValue on a cancelled ctx, Err:", vctx.Err())
  fmt.Println("new value readable:", vctx.Value(key(99)))
  fmt.Println("old value still readable:", vctx.Value(key(2)))
  fmt.Println("Done already closed:", isClosed(vctx.Done()))
}

func isClosed(ch <-chan struct{}) bool {
  select {
  case <-ch:
    return true
  default:
    return false
  }
}
```

Output:

```
context.Background.WithValue(main.key, v0).WithValue(main.key, v1).WithValue(main.key, v2).WithValue(main.key, v3).WithValue(main.key, v4)
Value(key(0)) = v0
Value(key(1)) = v1
Value(key(2)) = v2
Value(key(3)) = v3
Value(key(4)) = v4
Value(key(5)) = <nil>
after cancel, Err: context canceled
values survive cancellation: v0 v4
WithValue on a cancelled ctx, Err: context canceled
new value readable: added after cancel
old value still readable: v2
Done already closed: true
```

The `String()` output is the chain itself, which makes `fmt.Println(ctx)` a genuinely
useful debugging tool: you can see every layer and which key type each holds.

The last four lines are the part people are unsure about. Cancellation and values are
**orthogonal**. Cancelling a context does not clear its values, and `WithValue` on an
already-cancelled context returns a context that is *still cancelled* (`Err()` is
`context.Canceled`, `Done()` is already closed) but whose values — old and new — read
back fine. That matters for the `WithoutCancel` pattern in
[11.6](06-context-cancel.md): detached audit work reads the request ID off a context
whose cancellation it has deliberately dropped.

Since each `WithValue` allocates, and shadowing a key just adds a link rather than
replacing one, a loop that calls `WithValue` per iteration builds an arbitrarily long
chain. Set your values once, in middleware.

## Lookup cost

`Value` is a linked-list walk, so it is O(depth), and a **miss costs the most** because
it walks all the way to `Background()`.

```
BenchmarkDepth1-12         	468712770	         2.407 ns/op
BenchmarkDepth8-12         	77418105	        14.97 ns/op
BenchmarkDepth64-12        	10836501	       111.3 ns/op
BenchmarkDepth64Miss-12    	10818794	       112.3 ns/op
```

Machine-specific numbers (darwin/arm64, Apple M4 Pro); the shape is what matters — dead
linear in depth, and the miss at depth 64 costs the same as the deepest hit. Real request
chains are 3-8 deep, so this is a non-issue in practice; it becomes one if you call
`ctx.Value` inside a hot loop, which is the actual rule: read your values **once** at the
top of a function into locals, not per iteration.

## Propagation through a real handler chain

A `net/http` server with two middlewares, a service and a repository, plus goroutines
spawned from the handler — the whole picture, run against `httptest` so the output below
is a real capture. The `slog` time attribute is stripped via `ReplaceAttr` purely to make
the output reproducible here; in production you keep it.

```go
package main

import (
  "context"
  "fmt"
  "io"
  "log/slog"
  "net/http"
  "net/http/httptest"
  "os"
  "slices"
  "sync"
)

type reqIDKey struct{}

type userKey struct{}

func WithRequestID(ctx context.Context, id string) context.Context {
  return context.WithValue(ctx, reqIDKey{}, id)
}

func RequestID(ctx context.Context) string {
  id, _ := ctx.Value(reqIDKey{}).(string)
  return id
}

func WithUser(ctx context.Context, u string) context.Context {
  return context.WithValue(ctx, userKey{}, u)
}

func User(ctx context.Context) (string, bool) {
  u, ok := ctx.Value(userKey{}).(string)
  return u, ok
}

var log = slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
  ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
    if a.Key == slog.TimeKey {
      return slog.Attr{}
    }
    return a
  },
}))

func logger(ctx context.Context) *slog.Logger {
  return log.With("request_id", RequestID(ctx))
}

type Repository struct{}

func (Repository) Load(ctx context.Context, id int) (string, error) {
  logger(ctx).Info("repository query", "table", "orders", "id", id)
  return fmt.Sprintf("order-%d", id), nil
}

type Service struct{ repo Repository }

func (s Service) Fetch(ctx context.Context, id int) (string, error) {
  u, ok := User(ctx)
  logger(ctx).Info("service fetch", "user", u, "authenticated", ok)
  return s.repo.Load(ctx, id)
}

func requestIDMiddleware(next http.Handler) http.Handler {
  var n int
  var mu sync.Mutex
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    mu.Lock()
    n++
    id := fmt.Sprintf("req-%03d", n)
    mu.Unlock()
    ctx := WithRequestID(r.Context(), id)
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}

func authMiddleware(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    if u := r.Header.Get("X-User"); u != "" {
      ctx = WithUser(ctx, u)
    }
    next.ServeHTTP(w, r.WithContext(ctx))
  })
}

func handler(svc Service) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    logger(ctx).Info("handler start", "path", r.URL.Path)

    var mu sync.Mutex
    var sideEffects []string
    var wg sync.WaitGroup
    for _, task := range []string{"audit", "metrics"} {
      wg.Go(func() {
        mu.Lock()
        sideEffects = append(sideEffects, task+":"+RequestID(ctx))
        mu.Unlock()
      })
    }

    out, err := svc.Fetch(ctx, 7)
    if err != nil {
      http.Error(w, err.Error(), 500)
      return
    }
    wg.Wait()
    slices.Sort(sideEffects)
    logger(ctx).Info("spawned goroutines saw the same id", "tasks", sideEffects)
    fmt.Fprint(w, out)
  })
}

func main() {
  srv := httptest.NewServer(requestIDMiddleware(authMiddleware(handler(Service{}))))
  defer srv.Close()

  for _, user := range []string{"alice", ""} {
    req, _ := http.NewRequest(http.MethodGet, srv.URL+"/orders/7", nil)
    if user != "" {
      req.Header.Set("X-User", user)
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      panic(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("response body: %s\n\n", b)
  }
}
```

Output:

```
level=INFO msg="handler start" request_id=req-001 path=/orders/7
level=INFO msg="service fetch" request_id=req-001 user=alice authenticated=true
level=INFO msg="repository query" request_id=req-001 table=orders id=7
level=INFO msg="spawned goroutines saw the same id" request_id=req-001 tasks="[audit:req-001 metrics:req-001]"
response body: order-7

level=INFO msg="handler start" request_id=req-002 path=/orders/7
level=INFO msg="service fetch" request_id=req-002 user="" authenticated=false
level=INFO msg="repository query" request_id=req-002 table=orders id=7
level=INFO msg="spawned goroutines saw the same id" request_id=req-002 tasks="[audit:req-002 metrics:req-002]"
response body: order-7
```

Clean under `go run -race .`. Six things this demonstrates at once.

**`r.WithContext(ctx)` is how middleware adds a value.** `*http.Request` is immutable
in the sense that matters: you build a shallow copy carrying the new context and pass
that down. Mutating `r` in place would be a race against anything else holding the
request.

**The request ID reaches the repository without the repository knowing about HTTP.**
`Repository.Load` takes a `context.Context`, not an `*http.Request`. That is the point of
putting it in the context rather than in a parameter — the plumbing is uniform and the
bottom layer stays decoupled.

**The logger is derived from the context, not stored in it.** `logger(ctx)` builds a
`*slog.Logger` with `request_id` attached. A pre-built logger *can* go in the context
too (that is a legitimate use), but deriving one from the IDs is simpler and keeps the
context holding data rather than behavior.

**Goroutines spawned from the handler inherit the context by closure.** `RequestID(ctx)`
works inside `wg.Go` because `ctx` is captured, giving `audit:req-001` and
`metrics:req-001`. The results are collected under a mutex and sorted, so the line is
deterministic. The hazard: if those goroutines outlive the handler, they are reading a
*cancelled* context — fine for values, fatal if they also need to do I/O. That is
`context.WithoutCancel` plus its own deadline
([11.6](06-context-cancel.md)).

**The missing value degrades correctly.** The second request sends no `X-User` header, so
`User(ctx)` returns `("", false)` and the log says `user="" authenticated=false`. No
panic, no nil dereference — because the getter uses comma-ok.

**Each request gets a fresh chain.** `req-001` and `req-002` never see each other's
values. The values live on the context, which lives exactly as long as the request.

## What belongs in a context, and what does not

**Yes:**

| Value | Why |
|---|---|
| request ID | needed by every log line at every layer; purely diagnostic |
| trace / span ID | OpenTelemetry propagation; crosses every API boundary |
| authenticated user or claims | established by middleware, needed by authorization deep down |
| tenant / org ID | same: set once at the edge, consumed everywhere |
| a request-scoped logger | already carrying the IDs above |
| the client's locale or timezone | request-scoped, read in many places |

**No:**

| Value | Use instead |
|---|---|
| an optional function parameter | a parameter, or a functional-options struct |
| application config (timeouts, feature flags) | a field on the struct that needs it |
| a database handle, HTTP client, cache | constructor injection |
| anything a function *requires* to work | an explicit parameter |
| a mutable value other code writes to | a value with its own synchronization |

The dividing line is not taste, it is two concrete properties.

**It is untyped and invisible to the compiler.** A function taking `(ctx context.Context,
id int)` and panicking because a middleware did not install a `*sql.DB` has a signature
that lies. Delete the middleware and nothing fails to build; it fails in production, on
one route, at 3am. An explicit `db *sql.DB` parameter fails at compile time. Context
values trade compile-time safety for the ability to cross API boundaries you do not
control — spend that only where you actually need it.

**It is invisible to readers and to tooling.** `ctx.Value` has no "find all references"
you can trust, because the key is a private type in a package that may not be the one
doing the lookup. A context value is in effect a global variable scoped to a call tree.
Request ID earns that; a retry count does not.

A good test: *if a caller forgot to set this, should the code fail to compile?* If yes,
it is a parameter.

## Quick Reference

| Function | Signature | Note |
|---|---|---|
| `context.WithValue(parent, key, val)` | `(Context, any, any) Context` | one key per call; allocates one link |
| `ctx.Value(key)` | `(any) any` | `nil` when absent; O(depth) walk |

| Key type | Verdict |
|---|---|
| `type ctxKey struct{}` | **idiomatic** — zero size, unforgeable, one type per key |
| `type ctxKey int` + `iota` consts | fine, especially with many keys in one package |
| `type ctxKey string` (unexported) | acceptable; slower compare, no real benefit |
| a plain `string` or `int` literal | **wrong** — collides silently across packages |
| an exported key type | **wrong** — other packages can forge your key |

| Getter shape | Use when |
|---|---|
| `UserFrom(ctx) (*User, bool)` | absence is a case the caller must handle |
| `TraceFrom(ctx) string` | absence is harmless; zero value is fine |
| `MustUserFrom(ctx) *User` (panics) | a middleware guarantees it, enforced by routing |

| Property | Behaviour |
|---|---|
| mutability | immutable; `WithValue` shadows, never overwrites |
| lookup | newest link first; the first match wins |
| cost | O(depth); a miss walks the whole chain |
| after cancellation | values still readable; `WithValue` on a cancelled ctx works |
| across goroutines | inherited by closure capture; safe to read concurrently |

## Common Mistakes

**Using a plain `string` or `int` as a key.** Two packages picking `"user"` silently
share one slot and the later middleware wins; use an unexported empty-struct type.

**Exporting the key type.** Any package can then construct your key and overwrite your
value; export a setter and a getter instead.

**Making callers write the type assertion.** Every call site becomes a place a wrong
type can hide; put the single assertion in your getter.

**Putting dependencies or config in a context.** It turns a compile-time error into a
production panic on one route; pass a database handle, a client or a config as a
parameter or a struct field.

**Calling `ctx.Value` inside a hot loop.** It is a linked-list walk, and a miss walks
the entire chain; read it once into a local.

**Expecting `WithValue` to mutate the parent.** It returns a *new* context; if you drop
the return value or forget `r.WithContext(ctx)`, nothing downstream sees the value.

## Exercises

1. Build a `requestid` package exporting only `Middleware`, `WithID` and `FromContext`,
   keyed by an unexported type. Write a test that another package cannot reach the value
   except through `FromContext`, and that two independent packages both using
   `type ctxKey struct{}` do not collide.
2. Reproduce the collision bug from this lesson in a test — two packages with
   `string` keys — then fix it and show the test now distinguishes the two values.
3. Write middleware that puts a `*slog.Logger` (not just the ID) in the context, with
   `request_id` and `trace_id` pre-attached, plus a `Logger(ctx)` helper that falls back
   to `slog.Default()` on a miss. Verify the fallback with `context.Background()`.
4. Benchmark `ctx.Value` at depths 1, 4, 16 and 64 for a hit on the newest key, a hit on
   the oldest key, and a miss. Explain which of the three is cheapest at depth 64 and
   why.
5. Take the handler chain in this lesson and spawn an audit goroutine that must survive
   the request. Make it work with `context.WithoutCancel` plus its own 5s timeout, and
   prove with a cancelled request that the audit still logs the right request ID.

---

[← 11.6 context: Cancellation and Timeouts](06-context-cancel.md) · [Course Index](../README.md) · [11.8 errgroup →](08-errgroup.md)
