[← 16.6 Middleware from Scratch](06-middleware.md) · [Course Index](../README.md) · [16.8 Request Validation →](08-validation.md)

# 16.7 — chi: Routing, Groups, and Middleware

## What You'll Learn

- chi is `http.Handler` all the way down: every handler, every middleware, every subrouter is stdlib-compatible
- `Route`, `Group`, `Mount`, `With` — the four ways to scope middleware that the stdlib mux has no answer for
- `chi.Walk` to print your real route table, and custom `NotFound`/`MethodNotAllowed` handlers
- An honest table of what chi still buys you over the Go 1.22+ `ServeMux`

## Coming From Another Language

| Other | Go + chi |
|---|---|
| Express `app.use()` / `router.use()` | `r.Use(mw)` — same idea, but `mw` is `func(http.Handler) http.Handler` |
| Express `express.Router()` mounted with `app.use('/api', r)` | `r.Mount("/api", sub)` or `r.Route("/api", func(r chi.Router){...})` |
| Express `req.params.id` | `chi.URLParam(r, "id")` |
| Express route regex `/:id(\d+)` | `"/{id:[0-9]+}"` |
| FastAPI `APIRouter(prefix=..., dependencies=[...])` | `r.Route(prefix, ...)` with `r.Use(...)` inside |
| A framework that owns your `main` | chi owns nothing; it is a `http.Handler` you pass to `http.Server` |

## Installing it

```bash
go get github.com/go-chi/chi/v5@v5.5.0
```

Real output:

```
go: added github.com/go-chi/chi/v5 v5.5.0
```

Confirming the version actually in use:

```bash
go list -m github.com/go-chi/chi/v5
```

```
github.com/go-chi/chi/v5 v5.5.0
```

Everything in this lesson was run against **chi v5.5.0** on Go 1.27.1. chi has **zero
dependencies** outside the standard library, which is a large part of why this course picks
it.

## Why chi

Three reasons, in order of how much they matter:

**It is `http.Handler` everywhere.** `chi.Router` embeds `http.Handler`. Its middleware type
is `func(http.Handler) http.Handler` — the exact convention from 16.6, not a chi-specific
interface. So every middleware you wrote in the last lesson works unchanged, every stdlib
wrapper (`http.StripPrefix`, `http.TimeoutHandler`) works unchanged, and you can mount a
plain `*http.ServeMux` inside a chi router or vice versa. There is no lock-in: deleting chi
from a project means rewriting your route registrations and nothing else.

**No magic.** No reflection, no struct tags, no code generation, no context-key framework, no
`chi.Context` type you must accept instead of `*http.Request`. The entire API is about 30
methods.

**It is small.** One package plus a middleware subpackage, no transitive dependencies.

The alternatives, in one line: **gin** and **echo** are faster-benchmarking full frameworks
with their own `Context` type (so handlers are no longer `http.Handler` and the stdlib
ecosystem stops applying) — pick one if you want batteries like binding and rendering
built in; **gorilla/mux** is the venerable stdlib-compatible option whose main edge was
matchers on headers and queries, now largely redundant since Go 1.22.

## Routing basics, URL params, regexp patterns, and wildcards

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"

  "github.com/go-chi/chi/v5"
)

func main() {
  r := chi.NewRouter()
  r.Get("/items", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "list")
  })
  r.Post("/items", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(201)
    fmt.Fprint(w, "created")
  })
  r.Get("/items/{id}", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "id=%q", chi.URLParam(r, "id"))
  })
  r.Get("/num/{id:[0-9]+}", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "numeric id=%q", chi.URLParam(r, "id"))
  })
  r.Get("/files/*", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "wildcard=%q", chi.URLParam(r, "*"))
  })
  r.Get("/ctx/{a}/{b}", func(w http.ResponseWriter, r *http.Request) {
    rctx := chi.RouteContext(r.Context())
    fmt.Fprintf(w, "pattern=%q routePath=%q keys=%v values=%v",
      rctx.RoutePattern(), rctx.RoutePath, rctx.URLParams.Keys, rctx.URLParams.Values)
  })
  r.NotFound(func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(404)
    fmt.Fprint(w, `{"error":"route not found"}`)
  })
  r.MethodNotAllowed(func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(405)
    fmt.Fprint(w, `{"error":"method not allowed"}`)
  })

  srv := httptest.NewServer(r)
  defer srv.Close()

  do := func(m, p string) {
    req, _ := http.NewRequest(m, srv.URL+p, nil)
    resp, _ := http.DefaultClient.Do(req)
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-6s %-20s %-26s allow=%-14q %s\n", m, p, resp.Status,
      resp.Header.Get("Allow"), strings.TrimSpace(string(b)))
  }
  do("GET", "/items")
  do("POST", "/items")
  do("DELETE", "/items")
  do("GET", "/items/42")
  do("GET", "/num/42")
  do("GET", "/num/abc")
  do("GET", "/files/a/b/c.txt")
  do("GET", "/ctx/x/y")
  do("GET", "/nothing")
}
```

Output:

```
GET    /items               200 OK                     allow=""             list
POST   /items               201 Created                allow=""             created
DELETE /items               405 Method Not Allowed     allow=""             {"error":"method not allowed"}
GET    /items/42            200 OK                     allow=""             id="42"
GET    /num/42              200 OK                     allow=""             numeric id="42"
GET    /num/abc             404 Not Found              allow=""             {"error":"route not found"}
GET    /files/a/b/c.txt     200 OK                     allow=""             wildcard="a/b/c.txt"
GET    /ctx/x/y             200 OK                     allow=""             pattern="/ctx/{a}/{b}" routePath="" keys=[a b] values=[x y]
GET    /nothing             404 Not Found              allow=""             {"error":"route not found"}
```

**Custom 404 and 405 handlers work**, which the stdlib mux cannot do at all. That is the
single most common reason people reach for chi on an API: without it, your 404s and 405s are
Go's plain-text `404 page not found` while every other error in your service is JSON.

**Watch the `allow=""` column.** chi's custom `MethodNotAllowed` handler does **not** set an
`Allow` header, whereas the stdlib mux computes one for you (16.2). If you need it — and the
HTTP spec says a 405 must have it — set it yourself in your handler, from
`chi.RouteContext(r.Context()).MethodsAllowed` or a hardcoded list.

**`/num/abc` is a 404, not a 405.** A regexp pattern that does not match means the route does
not exist, so there is no method mismatch to report.

**`{id:[0-9]+}`** keeps "is this an integer" out of your handler entirely. Useful, but note
it turns a bad input into a 404 rather than a 400 with a helpful message — for a public API
you often want the 400, so validate in the handler instead (16.8).

**`chi.RouteContext(r.Context())`** exposes the routing internals: `RoutePattern()` is the
matched pattern (perfect for a metrics label — bounded cardinality), and `URLParams` holds
the parallel key/value slices behind `chi.URLParam`.

Method helpers: `Get`, `Post`, `Put`, `Patch`, `Delete`, `Head`, `Options`, `Trace`,
`Connect`. Plus `Handle(pattern, h http.Handler)` and `HandleFunc` for **all** methods, and
`Method(verb, pattern, h)` / `MethodFunc(verb, pattern, f)` when the verb is a variable.

## Route, Group, Mount, With

This is what the stdlib mux has no answer for. Each middleware here tags the response with
its name, so the output shows exactly which middlewares ran for each route.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"

  "github.com/go-chi/chi/v5"
  "github.com/go-chi/chi/v5/middleware"
)

func tag(name string) func(http.Handler) http.Handler {
  return func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      w.Header().Add("X-Mw", name)
      next.ServeHTTP(w, r)
    })
  }
}

func ok(label string) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "%s userID=%q postID=%q", label,
      chi.URLParam(r, "userID"), chi.URLParam(r, "postID"))
  }
}

func adminRouter() http.Handler {
  r := chi.NewRouter()
  r.Use(tag("admin-only"))
  r.Get("/stats", ok("admin stats"))
  return r
}

func main() {
  r := chi.NewRouter()
  r.Use(middleware.RequestID)
  r.Use(middleware.CleanPath)
  r.Use(middleware.StripSlashes)
  r.Use(middleware.NoCache)
  r.Use(tag("global"))

  r.Get("/health", ok("health"))

  r.Route("/api/v1", func(r chi.Router) {
    r.Use(tag("api-v1"))

    r.Route("/users", func(r chi.Router) {
      r.Get("/", ok("list users"))
      r.Post("/", ok("create user"))
      r.Route("/{userID}", func(r chi.Router) {
        r.Use(tag("user-scope"))
        r.Get("/", ok("get user"))
        r.Route("/posts", func(r chi.Router) {
          r.Get("/", ok("list posts"))
          r.Get("/{postID}", ok("get post"))
          r.With(tag("inline-only")).Delete("/{postID}", ok("delete post"))
        })
      })
    })

    r.Group(func(r chi.Router) {
      r.Use(tag("group-scope"))
      r.Get("/reports", ok("reports"))
    })
  })

  r.Mount("/admin", adminRouter())

  r.NotFound(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(404)
    fmt.Fprint(w, `{"error":"not found"}`)
  })
  r.MethodNotAllowed(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(405)
    fmt.Fprint(w, `{"error":"method not allowed"}`)
  })

  srv := httptest.NewServer(r)
  defer srv.Close()

  do := func(m, p string) {
    req, _ := http.NewRequest(m, srv.URL+p, nil)
    resp, _ := http.DefaultClient.Do(req)
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-7s %-34s %-24s mw=%-52s %s\n", m, p, resp.Status,
      strings.Join(resp.Header.Values("X-Mw"), ","), strings.TrimSpace(string(b)))
  }

  do("GET", "/health")
  do("GET", "/api/v1/users")
  do("POST", "/api/v1/users")
  do("GET", "/api/v1/users/7")
  do("GET", "/api/v1/users/7/posts")
  do("GET", "/api/v1/users/7/posts/99")
  do("DELETE", "/api/v1/users/7/posts/99")
  do("GET", "/api/v1/reports")
  do("GET", "/admin/stats")
  do("PUT", "/api/v1/users/7")
  do("GET", "/api/v1/nope")
  do("GET", "/api/v1/users//7")

  fmt.Println("\n=== chi.Walk route table ===")
  chi.Walk(r, func(method, route string, h http.Handler, mws ...func(http.Handler) http.Handler) error {
    fmt.Printf("%-7s %-40s (%d middlewares)\n", method, route, len(mws))
    return nil
  })
}
```

Output:

```
GET     /health                            200 OK                   mw=global                                               health userID="" postID=""
GET     /api/v1/users                      200 OK                   mw=global,api-v1                                        list users userID="" postID=""
POST    /api/v1/users                      200 OK                   mw=global,api-v1                                        create user userID="" postID=""
GET     /api/v1/users/7                    200 OK                   mw=global,api-v1,user-scope                             get user userID="7" postID=""
GET     /api/v1/users/7/posts              200 OK                   mw=global,api-v1,user-scope                             list posts userID="7" postID=""
GET     /api/v1/users/7/posts/99           200 OK                   mw=global,api-v1,user-scope                             get post userID="7" postID="99"
DELETE  /api/v1/users/7/posts/99           200 OK                   mw=global,api-v1,user-scope,inline-only                 delete post userID="7" postID="99"
GET     /api/v1/reports                    200 OK                   mw=global,api-v1,group-scope                            reports userID="" postID=""
GET     /admin/stats                       200 OK                   mw=global,admin-only                                    admin stats userID="" postID=""
PUT     /api/v1/users/7                    405 Method Not Allowed   mw=global,api-v1,user-scope                             {"error":"method not allowed"}
GET     /api/v1/nope                       404 Not Found            mw=global,api-v1                                        {"error":"not found"}
GET     /api/v1/users//7                   200 OK                   mw=global,api-v1,user-scope                             get user userID="7" postID=""
```

The `mw=` column is the whole lesson. Read down it:

**`Route` nests a subrouter under a path prefix** and scopes middleware to it.
`/api/v1/users/7` ran `global,api-v1,user-scope` — three middlewares from three nesting
levels, accumulated automatically. `/health`, outside the `/api/v1` tree, ran only `global`.

**Nested `Route` calls compose**, which is how `/api/v1/users/{userID}/posts/{postID}` stays
readable. Both `userID` and `postID` are available at the leaf, from different nesting
levels. Compare with writing that path out longhand on a flat mux twelve times.

**`Group` scopes middleware with no path prefix.** `/api/v1/reports` got `group-scope` but
sits at the same path level as `/api/v1/users`. That is the "these six routes need auth, those
three do not, all under the same prefix" case.

**`With` applies middleware to one route inline**, returning a router rather than mutating
one. `DELETE .../posts/{postID}` got `inline-only`; the `GET` on the identical pattern did
not.

**`Mount` attaches an independent `http.Handler`.** `/admin/stats` ran `global,admin-only` —
it inherited the parent's middleware but *not* `api-v1`, because it is mounted at the root,
not inside `/api/v1`. `Mount` takes any `http.Handler`, so you can mount a plain
`*http.ServeMux`, an `http.FileServer`, a `/debug/pprof` handler, or a router from another
package. This is how you compose a service out of independently-testable modules
([18.1](../18-production/01-project-layout.md)).

**`/api/v1/users//7` still worked**, thanks to `middleware.CleanPath`. And the route is
registered as `/` inside `Route("/{userID}")`, which `middleware.StripSlashes` makes
reachable as `/api/v1/users/7` without a trailing slash.

**Middleware still runs on 404 and 405.** The `GET /api/v1/nope` line shows
`global,api-v1` — chi found the subrouter, ran its middleware, then failed to match. So your
logging and request-ID middleware cover error responses too, which is what you want.

Important ordering rule: **`r.Use` must come before any route registration on that router.**
chi panics if you call `Use` after a route is defined, because the middleware chain is
compiled at registration time.

## chi.Walk: printing the real route table

The tail of the same run:

```
=== chi.Walk route table ===
GET     /admin/stats                             (6 middlewares)
GET     /api/v1/reports                          (7 middlewares)
POST    /api/v1/users/                           (6 middlewares)
GET     /api/v1/users/                           (6 middlewares)
GET     /api/v1/users/{userID}/                  (7 middlewares)
GET     /api/v1/users/{userID}/posts/            (7 middlewares)
GET     /api/v1/users/{userID}/posts/{postID}    (7 middlewares)
DELETE  /api/v1/users/{userID}/posts/{postID}    (8 middlewares)
GET     /health                                  (5 middlewares)
```

(The relative order of `GET` and `POST` on the same pattern varies between runs, since chi
iterates a method map.)

`chi.Walk` is genuinely useful and has no stdlib equivalent. Three real uses: a
`--routes` flag that prints the table so you can diff it in code review and catch an
accidentally-deleted route; generating API documentation; and asserting in a test that a
route you require still exists with the middleware count you expect. The middleware counts
confirm the nesting arithmetic — 5 global, +1 for `api-v1`, +1 for `user-scope`, +1 for the
inline `With` on the DELETE.

`chi.Walk` returns an error if your callback does, so returning a non-nil error stops the
walk early.

## chi's bundled middleware

The `middleware` subpackage ships the ones everybody writes. Here are the most useful, run
for real.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
  "time"

  "github.com/go-chi/chi/v5"
  "github.com/go-chi/chi/v5/middleware"
)

func main() {
  r := chi.NewRouter()
  r.Use(middleware.RequestID)
  r.Use(middleware.RealIP)
  r.Use(middleware.Recoverer)
  r.Use(middleware.NoCache)
  r.Use(middleware.Timeout(80 * time.Millisecond))

  r.Get("/ping", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "RemoteAddr=%s reqID-in-ctx=%v",
      strings.Split(r.RemoteAddr, ":")[0],
      middleware.GetReqID(r.Context()) != "")
  })
  r.Get("/panic", func(w http.ResponseWriter, r *http.Request) {
    panic("chi recoverer test")
  })
  r.Get("/slow", func(w http.ResponseWriter, r *http.Request) {
    select {
    case <-time.After(400 * time.Millisecond):
      fmt.Fprint(w, "slow done")
    case <-r.Context().Done():
      fmt.Println("  handler observed:", r.Context().Err())
    }
  })
  r.With(middleware.Throttle(1)).Get("/throttled", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "throttled ok")
  })

  srv := httptest.NewServer(r)
  defer srv.Close()

  do := func(p string, hdr map[string]string) {
    req, _ := http.NewRequest("GET", srv.URL+p, nil)
    for k, v := range hdr {
      req.Header.Set(k, v)
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      fmt.Printf("%-12s client err %v\n", p, err)
      return
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-12s %-26s cache-control=%q\n   body=%q\n",
      p, resp.Status, resp.Header.Get("Cache-Control"), strings.TrimSpace(string(b)))
  }

  do("/ping", nil)
  do("/ping", map[string]string{"X-Forwarded-For": "203.0.113.7"})
  do("/panic", nil)
  do("/slow", nil)
  do("/throttled", nil)
}
```

Output, with `middleware.Recoverer`'s pretty-printed stack trace removed (about 25 lines of
file paths that differ on every machine; only its `panic:` line is kept):

```
/ping        200 OK                     cache-control="no-cache, no-store, no-transform, must-revalidate, private, max-age=0"
   body="RemoteAddr=127.0.0.1 reqID-in-ctx=true"
/ping        200 OK                     cache-control="no-cache, no-store, no-transform, must-revalidate, private, max-age=0"
   body="RemoteAddr=203.0.113.7 reqID-in-ctx=true"
 panic: chi recoverer test
/panic       500 Internal Server Error  cache-control="no-cache, no-store, no-transform, must-revalidate, private, max-age=0"
   body=""
  handler observed: context deadline exceeded
/slow        504 Gateway Timeout        cache-control="no-cache, no-store, no-transform, must-revalidate, private, max-age=0"
   body=""
/throttled   200 OK                     cache-control="no-cache, no-store, no-transform, must-revalidate, private, max-age=0"
   body="throttled ok"
```

Four behaviours to note, two of them surprises:

**`middleware.RealIP` really rewrote `r.RemoteAddr`** to `203.0.113.7` from the
`X-Forwarded-For` header. That is convenient and **dangerous**: it trusts the header
unconditionally, so without a trusted proxy in front of you, any client can forge their IP
and defeat your rate limiter or audit log (16.4). Use it only behind a load balancer that
overwrites the header.

**`middleware.RequestID` puts the ID in the context, not in the response header.** Read it
with `middleware.GetReqID(r.Context())`. If you want it echoed to the client — and you do,
for support tickets (16.6) — add a one-line middleware that copies it into
`w.Header().Set("X-Request-Id", ...)`.

**`middleware.Recoverer` returns 500 with an *empty body*.** It prints a colourized stack to
stderr and sends no response body at all. For a JSON API that is wrong — clients get a 500
they cannot parse — so write your own (16.6) or wrap this one. This is the clearest example
of chi's middleware being "reasonable defaults for a web app", not "correct for your API".

**`middleware.Timeout` returns 504 Gateway Timeout**, where the stdlib
`http.TimeoutHandler` returns 503 (16.6). Neither is more correct; just know which one your
clients will see. The handler's `r.Context()` was cancelled with `context deadline exceeded`,
so `middleware.Timeout` is a thin wrapper over `context.WithTimeout` — it does *not* buffer
the response the way `http.TimeoutHandler` does, so it works in front of a streaming
handler.

The catalogue:

| Middleware | Does | Caution |
|---|---|---|
| `middleware.RequestID` | generates an ID into the context | not in the response header |
| `middleware.RealIP` | rewrites `r.RemoteAddr` from `X-Forwarded-For`/`X-Real-IP` | **trusts the header**; only use behind a proxy |
| `middleware.Logger` | logs method, path, status, bytes, duration | human-readable text, not structured; use your own `slog` one in production |
| `middleware.Recoverer` | panic → 500 | **empty body**; stack to stderr |
| `middleware.Timeout(d)` | `context.WithTimeout` per request | returns **504** |
| `middleware.Compress(level, types...)` | gzip/deflate by content type | handles the `Accept-Encoding` and size pitfalls from 16.6 |
| `middleware.StripSlashes` | `/users/` → `/users` before matching | pair with `CleanPath` |
| `middleware.RedirectSlashes` | 301-redirects instead of stripping | pick one, not both |
| `middleware.CleanPath` | collapses `//` and resolves `..` | |
| `middleware.NoCache` | sets the full no-store header set | see the real value above |
| `middleware.Heartbeat(path)` | answers one path with `"."` | a zero-dependency liveness probe |
| `middleware.Throttle(n)` | caps **concurrent** in-flight requests, 503 beyond | not a rate limiter — it limits concurrency |
| `middleware.ThrottleBacklog(n, backlog, d)` | same, with a queue | |
| `middleware.AllowContentType(types...)` | 415 for anything else | |
| `middleware.SetHeader(k, v)` | sets a response header | |
| `middleware.GetHead` | routes `HEAD` to the `GET` handler | the stdlib mux does this itself |
| `middleware.WithValue(k, v)` | puts a static value in the context | config belongs in a struct, not a context (11.7) |

Note `middleware.Throttle` limits **concurrency**, not rate — it is a semaphore
([12.4](../12-concurrency-patterns/04-rate-limiting.md)), useful for protecting a downstream
with a connection limit. For requests-per-second you want a real rate limiter.

## chi vs the Go 1.22+ ServeMux

| Capability | stdlib `ServeMux` | chi v5 |
|---|---|---|
| Method matching | yes | yes |
| Path variables | yes, `r.PathValue` | yes, `chi.URLParam` |
| Multi-segment wildcard | yes, `{rest...}` | yes, `/*` |
| Host matching | yes (exact host) | no (do it in middleware) |
| Automatic 405 + `Allow` header | **yes, computed** | 405 yes, **`Allow` not set** with a custom handler |
| Automatic `HEAD` from `GET` | yes | via `middleware.GetHead` |
| Conflict detection | **yes, panics at registration** | no — later registration wins silently |
| **Regexp patterns** (`{id:[0-9]+}`) | no | **yes** |
| **Middleware chaining** (`Use`) | no — nest by hand | **yes** |
| **Path-scoped subrouters** (`Route`) | no | **yes** |
| **Middleware-only scopes** (`Group`) | no | **yes** |
| **Mounting another handler** (`Mount`) | partially, via `StripPrefix` | **yes, first-class** |
| **Per-route inline middleware** (`With`) | manual wrapping | **yes** |
| **Custom 404 / 405 handlers** | no | **yes** |
| **Route introspection** (`Walk`) | no | **yes** |
| Matched pattern available | `mux.Handler(r)` | `chi.RouteContext(r).RoutePattern()` |
| Dependencies | none | none |

The honest summary: **the stdlib mux is enough for a service with a flat route list and
global middleware.** The moment you want per-group middleware, a JSON 404, nested resources,
or a route table you can print, chi pays for itself — and because it is `http.Handler` all
the way down, adopting it costs you nothing you cannot undo.

Two things the stdlib does *better*: the registration-time conflict panic (chi silently lets
a later registration shadow an earlier one, so a typo'd route can be dead and you will not
be told), and the computed `Allow` header on a 405.

## Quick Reference

| chi method | Purpose |
|---|---|
| `chi.NewRouter() *chi.Mux` | a router; it is an `http.Handler` |
| `r.Get/Post/Put/Patch/Delete/Head/Options/Trace/Connect(pat, f)` | method + pattern + `func(w, r)` |
| `r.Handle(pat, h)` / `r.HandleFunc(pat, f)` | **all** methods |
| `r.Method(verb, pat, h)` / `r.MethodFunc(verb, pat, f)` | verb as a variable |
| `r.Use(mw...)` | middleware for this router; **before** any route |
| `r.With(mw...) chi.Router` | returns a router with extra middleware, for one chain |
| `r.Group(fn)` | a middleware scope with no path prefix |
| `r.Route(pat, fn) chi.Router` | a subrouter mounted at `pat` |
| `r.Mount(pat, h)` | mount any `http.Handler` at `pat` |
| `r.NotFound(f)` / `r.MethodNotAllowed(f)` | custom error handlers |
| `chi.URLParam(r, name) string` | a path variable; `"*"` for the wildcard |
| `chi.URLParamFromCtx(ctx, name) string` | same, from a bare context |
| `chi.RouteContext(ctx) *chi.Context` | `RoutePattern()`, `RoutePath`, `URLParams` |
| `chi.Walk(r, fn) error` | iterate the route table |
| `chi.NewRouteContext() *chi.Context` | build one by hand, for tests (16.11) |

| Pattern | Matches |
|---|---|
| `/items` | exactly |
| `/items/{id}` | one segment, into `id` |
| `/items/{id:[0-9]+}` | one segment matching the regexp |
| `/files/*` | the rest of the path, into `"*"` |
| `/` inside `Route("/users")` | `/users` (with `StripSlashes`) and `/users/` |

## Common Mistakes

**Calling `r.Use` after registering a route on that router.** chi panics, because the chain
is compiled at registration; declare all middleware for a router first.

**Expecting `middleware.Recoverer` to send a JSON body.** It sends an empty 500, so your
clients get an unparseable response; use the recovery middleware from 16.6.

**Using `middleware.RealIP` without a trusted proxy.** It believes `X-Forwarded-For`
unconditionally, letting any client forge their IP; only enable it behind a load balancer
that overwrites the header.

**Relying on a custom 405 to set `Allow`.** chi does not set it, and the spec requires it, so
add it in your `MethodNotAllowed` handler.

**Assuming a duplicate route is an error.** chi silently lets the later registration win, so
a typo leaves dead code and no warning; `chi.Walk` in a test is the defence.

**Treating `middleware.Throttle` as a rate limiter.** It bounds concurrent in-flight
requests, not requests per second; use a token-bucket limiter for rate.

## Exercises

1. Port the CRUD API from 16.5 to chi with `Route("/api/v1")`, a `Group` requiring auth for
   the write methods, JSON `NotFound`/`MethodNotAllowed` handlers that *do* set `Allow`, and
   `chi.Walk` printing the table. Confirm every status from the 16.5 output is unchanged.
2. Build `/api/v1/users/{userID}/posts/{postID}/comments/{commentID}` with a middleware at
   each nesting level that loads the parent resource into the context and 404s if it is
   missing. Show the real responses for a valid path, a missing user, and a missing post.
3. Write a test using `chi.Walk` that fails if any route lacks the auth middleware, by
   comparing middleware counts against an expected map. Then delete an `r.Use` line and show
   the test catching it.
4. Mount a plain `*http.ServeMux` under `/legacy` and a chi subrouter under `/v2` on the same
   parent, with a global logging middleware. Prove with real output that both receive the
   global middleware and that path variables work in each.
5. Replace `middleware.Recoverer` with your own from 16.6 and `middleware.Logger` with an
   `slog` version. Compare the real output of both pairs for a panicking route, and state
   which fields you gained.

---

[← 16.6 Middleware from Scratch](06-middleware.md) · [Course Index](../README.md) · [16.8 Request Validation →](08-validation.md)
