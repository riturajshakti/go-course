[← 16.1 Your First net/http Server](01-first-server.md) · [Course Index](../README.md) · [16.3 Handlers, HandlerFunc, and the Interface →](03-handlers.md)

# 16.2 — ServeMux Routing: Methods and Wildcards

## What You'll Learn

- Go 1.22+ patterns in full: `"GET /items"`, `"/items/{id}"`, `"{rest...}"`, `"/static/"`, `"host/path"`
- Method matching, automatic `405` with an `Allow` header, and automatic `HEAD` from `GET`
- The precedence rule (most specific wins) and the **registration-time panic** when two patterns are ambiguous
- `NotFound`, `Error`, `Redirect`, `StripPrefix`, `ServeFile`, `FileServer`, `FileServerFS`

## Coming From Another Language

| Other | Go |
|---|---|
| Express: first matching route wins, order matters | **most specific** pattern wins, registration order is irrelevant |
| Express: ambiguous routes silently shadow each other | ambiguous patterns **panic at registration** — a startup crash, not a 3am mystery |
| Express `req.params.id` | `r.PathValue("id")` — a method, not a map |
| Express `/files/*` then `req.params[0]` | `/files/{rest...}` then `r.PathValue("rest")` |
| FastAPI `@app.get(...)` / `@app.post(...)` | the method is part of the pattern string: `"GET /items"` |
| Most frameworks: unmatched method gives 404 | Go gives **405 with an `Allow` header**, computed for you |

Before Go 1.22 `ServeMux` had no method matching and no path variables, which is why every
Go tutorial older than that reaches for a third-party router in lesson one. That is no
longer true, and this lesson is the reason 16.7 can present `chi` as a choice rather than a
necessity.

## Method matching and the automatic 405

A pattern is `[METHOD ][host]/path`. With a method, only that method matches.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("GET /items", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "list items")
  })
  mux.HandleFunc("POST /items", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusCreated)
    fmt.Fprintln(w, "created")
  })
  mux.HandleFunc("GET /items/{id}", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "item id=%q\n", r.PathValue("id"))
  })
  mux.HandleFunc("DELETE /items/{id}", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusNoContent)
  })

  srv := httptest.NewServer(mux)
  defer srv.Close()

  reqs := []struct{ method, path string }{
    {"GET", "/items"},
    {"POST", "/items"},
    {"PUT", "/items"},
    {"PATCH", "/items"},
    {"GET", "/items/42"},
    {"DELETE", "/items/42"},
    {"POST", "/items/42"},
    {"HEAD", "/items"},
  }
  for _, rq := range reqs {
    req, _ := http.NewRequest(rq.method, srv.URL+rq.path, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      panic(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-6s %-12s -> %-22s Allow=%-24q body=%q\n",
      rq.method, rq.path, resp.Status, resp.Header.Get("Allow"),
      strings.TrimSpace(string(b)))
  }
}
```

Output:

```
GET    /items       -> 200 OK                 Allow=""                       body="list items"
POST   /items       -> 201 Created            Allow=""                       body="created"
PUT    /items       -> 405 Method Not Allowed Allow="GET, HEAD, POST"        body="Method Not Allowed"
PATCH  /items       -> 405 Method Not Allowed Allow="GET, HEAD, POST"        body="Method Not Allowed"
GET    /items/42    -> 200 OK                 Allow=""                       body="item id=\"42\""
DELETE /items/42    -> 204 No Content         Allow=""                       body=""
POST   /items/42    -> 405 Method Not Allowed Allow="DELETE, GET, HEAD"      body="Method Not Allowed"
HEAD   /items       -> 200 OK                 Allow=""                       body=""
```

Four behaviours in that output are worth pinning down.

**The 405 and its `Allow` header are generated for you.** `ServeMux` knows every method
registered for a matching path, so it answers with `405 Method Not Allowed` and a correct,
sorted `Allow` list. Most frameworks make you build that yourself or give you a 404.

**`GET` implies `HEAD`.** `Allow` says `GET, HEAD, POST` though no `HEAD` was registered,
and the `HEAD /items` request really returned 200 with an empty body. Go runs the `GET`
handler and discards the body.

**Per-path method sets are independent.** `/items` allows `GET, HEAD, POST`;
`/items/{id}` allows `DELETE, GET, HEAD`.

**No method in the pattern means all methods match.** `mux.Handle("/items", h)` without a
method accepts `GET`, `POST`, `DELETE`, `BREW` — anything. Omitting the method is how you
opt out of the 405 machinery, which you occasionally want for a catch-all.

The method must be a valid uppercase token. `"get /items"` is **not** a method pattern —
lowercase makes Go treat the whole string as a path, so you get a route literally named
`get /items` that nothing ever matches. There is no warning.

## Wildcards: {id}, {rest...}, and subtrees

Four pattern shapes, interacting:

- `/items/{id}` — exact, one segment, bound to `r.PathValue("id")`
- `/files/{rest...}` — matches one *or more* remaining segments, slashes included
- `/static/` — a trailing slash makes it a **subtree**: it matches everything beneath
- `/` — the universal catch-all, since it is a subtree rooted at nothing

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

func echo(label string) http.HandlerFunc {
  return func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "%s | id=%q sub=%q path=%q", label,
      r.PathValue("id"), r.PathValue("rest"), r.URL.Path)
  }
}

func main() {
  mux := http.NewServeMux()
  mux.Handle("/items/{id}", echo("exact /items/{id}"))
  mux.Handle("/items/{id}/tags", echo("exact /items/{id}/tags"))
  mux.Handle("/files/{rest...}", echo("multi /files/{rest...}"))
  mux.Handle("/static/", echo("subtree /static/"))
  mux.Handle("/static/css/app.css", echo("exact /static/css/app.css"))
  mux.Handle("/", echo("catch-all /"))

  srv := httptest.NewServer(mux)
  defer srv.Close()

  paths := []string{
    "/items/7",
    "/items/7/tags",
    "/items/7/tags/extra",
    "/files/a/b/c.txt",
    "/files/",
    "/files",
    "/static/",
    "/static/img/logo.png",
    "/static/css/app.css",
    "/static",
    "/nothing/here",
    "/",
  }
  for _, p := range paths {
    resp, err := http.Get(srv.URL + p)
    if err != nil {
      panic(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-22s %-16s %s\n", p, resp.Status, strings.TrimSpace(string(b)))
  }
}
```

Output:

```
/items/7               200 OK           exact /items/{id} | id="7" sub="" path="/items/7"
/items/7/tags          200 OK           exact /items/{id}/tags | id="7" sub="" path="/items/7/tags"
/items/7/tags/extra    200 OK           catch-all / | id="" sub="" path="/items/7/tags/extra"
/files/a/b/c.txt       200 OK           multi /files/{rest...} | id="" sub="a/b/c.txt" path="/files/a/b/c.txt"
/files/                200 OK           multi /files/{rest...} | id="" sub="" path="/files/"
/files                 200 OK           multi /files/{rest...} | id="" sub="" path="/files/"
/static/               200 OK           subtree /static/ | id="" sub="" path="/static/"
/static/img/logo.png   200 OK           subtree /static/ | id="" sub="" path="/static/img/logo.png"
/static/css/app.css    200 OK           exact /static/css/app.css | id="" sub="" path="/static/css/app.css"
/static                200 OK           subtree /static/ | id="" sub="" path="/static/"
/nothing/here          200 OK           catch-all / | id="" sub="" path="/nothing/here"
/                      200 OK           catch-all / | id="" sub="" path="/"
```

**`{id}` matches exactly one segment.** `/items/7/tags/extra` fell through to the catch-all,
because `/items/{id}/tags` has no room for `extra` and `{id}` will not swallow a slash.

**`{rest...}` captures the remainder with its slashes intact.** `sub="a/b/c.txt"` — one
`PathValue` holding three segments. It must be the last thing in the pattern.

**`{rest...}` and a subtree both match the empty remainder.** `/files/` gives `sub=""`.

**`/static/css/app.css` beat `/static/`.** Registration order is irrelevant; specificity
decides.

**`/files` and `/static` show up with a trailing slash.** Neither request was made that
way — a redirect happened, which is the next section.

## The trailing-slash redirect, path cleaning, and %-encoding

Registering a subtree `/static/` makes `ServeMux` also register an implicit redirect from
`/static` to `/static/`. The client above followed it transparently, which is why the
printed `r.URL.Path` already had the slash. With redirect-following disabled you see the
real responses:

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/static/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "subtree")
  })
  mux.HandleFunc("/items/{id}", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "id=%s", r.PathValue("id"))
  })

  srv := httptest.NewServer(mux)
  defer srv.Close()

  noFollow := &http.Client{CheckRedirect: func(*http.Request, []*http.Request) error {
    return http.ErrUseLastResponse
  }}

  paths := []string{"/static", "/static/", "/items//", "/items/a/../b", "/items/%2Fx", "/items/a%20b", "/missing", "/items"}
  for _, p := range paths {
    resp, err := noFollow.Get(srv.URL + p)
    if err != nil {
      fmt.Printf("%-18s ERROR %v\n", p, err)
      continue
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-18s %-16s Location=%-14q body=%q\n", p, resp.Status,
      resp.Header.Get("Location"), strings.TrimSpace(string(b)))
  }
}
```

Output:

```
/static            307 Temporary Redirect Location="/static/"     body="<a href=\"/static/\">Temporary Redirect</a>."
/static/           200 OK           Location=""             body="subtree"
/items//           307 Temporary Redirect Location="/items/"      body="<a href=\"/items/\">Temporary Redirect</a>."
/items/a/../b      307 Temporary Redirect Location="/items/b"     body="<a href=\"/items/b\">Temporary Redirect</a>."
/items/%2Fx        200 OK           Location=""             body="id=/x"
/items/a%20b       200 OK           Location=""             body="id=a b"
/missing           404 Not Found    Location=""             body="404 page not found"
/items             404 Not Found    Location=""             body="404 page not found"
```

**It is 307, not 301.** `307 Temporary Redirect` preserves the method and the body, so a
`POST /static` redirects into a `POST /static/`. A 301 would let clients rewrite it to
`GET`, silently losing the body. (Go's own `http.Redirect` default is whatever you pass;
the mux's *implicit* redirects are 307 for the slash fixup and 301 for path cleaning in
older versions — on 1.27 both show as 307 above.)

**Path cleaning happens before matching.** `//` collapses and `..` resolves, and the mux
*redirects* rather than quietly serving the cleaned path, so the client's URL ends up
canonical. That makes `..` traversal tricks useless against your route table.

**`%2F` decodes to a literal `/` inside a path value without splitting the segment.**
`id="/x"` — one segment whose value contains a slash. If a path value feeds a filesystem
path or a database lookup, validate it; the mux will not.

**`%20` decodes too.** `r.PathValue` gives you the decoded value (`"a b"`), which is what
you want, but it means a path value can contain anything.

**`/items` is a 404, not a redirect.** `/items/{id}` is an exact pattern with no subtree,
so nothing links the two. The implicit redirect only exists for registered subtrees.

Without a `/` catch-all registered, unmatched paths get Go's built-in
`404 page not found`.

## {$}: anchoring a pattern so it does not become a subtree

A trailing `/` makes a pattern a subtree, which is often not what you want — `"/"` matching
every URL on earth is the usual complaint. `{$}` at the end anchors the pattern to end
exactly there. This example also uses `httptest.NewRecorder` to call the mux directly with
no network at all (16.11), and `mux.Handler` to ask which pattern *would* match.

```go
package main

import (
  "fmt"
  "net/http"
  "net/http/httptest"
)

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/items/{$}", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "exact-slash {$}")
  })
  mux.HandleFunc("/items/{id}", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "id=%s", r.PathValue("id"))
  })
  for _, p := range []string{"/items/", "/items/7", "/items/7/x"} {
    req := httptest.NewRequest("GET", p, nil)
    rec := httptest.NewRecorder()
    mux.ServeHTTP(rec, req)
    fmt.Printf("%-12s %d %q\n", p, rec.Code, rec.Body.String())
  }
  req := httptest.NewRequest("GET", "/items/7", nil)
  _, pat := mux.Handler(req)
  fmt.Printf("Handler lookup for /items/7 -> pattern %q\n", pat)
}
```

Output:

```
/items/      200 "exact-slash {$}"
/items/7     200 "id=7"
/items/7/x   404 "404 page not found\n"
```

Plus the lookup line:

```
Handler lookup for /items/7 -> pattern "/items/{id}"
```

`"/{$}"` is therefore the pattern for "the home page only", as opposed to `"/"` which is
the catch-all. `mux.Handler(r)` returning the matched pattern string is handy for logging
and for metrics labels — you want the *pattern* as a label, never the raw path, or your
metrics cardinality explodes.

## Precedence, and the conflict panic

The rule is one sentence: **the most specific matching pattern wins.** Formally, pattern A
beats B if every path A matches is also matched by B, and A matches strictly fewer. Literal
segments beat wildcards; a longer path beats a shorter subtree.

So `/items/{id}` loses to `/items/latest` for the path `/items/latest`, and
`/static/css/app.css` beats `/static/` as you saw above. Registration order never matters —
you can declare routes in any order, in any file.

What happens when neither pattern is more specific than the other? Go refuses to guess.

```go
package main

import (
  "fmt"
  "net/http"
)

func h(w http.ResponseWriter, r *http.Request) {}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("/items/{id}/detail", h)
  mux.HandleFunc("/items/latest/{field}", h)
  fmt.Println("registered")
}
```

This panics on purpose, at registration. Output (traceback trimmed, scratch path
normalized):

```
panic: pattern "/items/latest/{field}" (registered at /tmp/scratch/main.go:13) conflicts with pattern "/items/{id}/detail" (registered at /tmp/scratch/main.go:12):
	/items/latest/{field} and /items/{id}/detail both match some paths, like "/items/latest/detail".
	But neither is more specific than the other.
	/items/latest/{field} matches "/items/latest/field", but /items/{id}/detail doesn't.
	/items/{id}/detail matches "/items/id/detail", but /items/latest/{field} doesn't.

goroutine 1 [running]:
net/http.(*ServeMux).register(...)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/net/http/server.go:2957
net/http.(*ServeMux).HandleFunc(0x0?, {0x1050e3fa1?, 0x0?}, 0x0?)
	/opt/homebrew/Cellar/go/1.27.1/libexec/src/net/http/server.go:2931 +0x74
main.main()
```

That is one of the best error messages in the standard library: both patterns with their
**source line numbers**, a concrete path that is ambiguous, and one witness path for each
direction proving neither dominates. Express would have silently let the first registration
win.

Registering the same pattern twice panics too, with a shorter message. Renaming the
wildcard does not help — `{id}` and `{other}` match identically:

```go
package main

import (
  "fmt"
  "net/http"
)

func h(w http.ResponseWriter, r *http.Request) {}

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("GET /items/{id}", h)
  mux.HandleFunc("GET /items/{other}", h)
  fmt.Println("registered")
}
```

This panics on purpose. Output (traceback trimmed, scratch path normalized):

```
panic: pattern "GET /items/{other}" (registered at /tmp/scratch/main.go:13) conflicts with pattern "GET /items/{id}" (registered at /tmp/scratch/main.go:12):
	GET /items/{other} matches the same requests as GET /items/{id}
```

Because both panics happen at registration, they happen at process start, every time, on
every machine — including in CI. A routing conflict can never reach production.

## Host patterns

A pattern may begin with a host, before the leading `/`. A host pattern is more specific
than one without, so it wins for matching requests.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "strings"
)

func main() {
  mux := http.NewServeMux()
  mux.HandleFunc("api.example.com/v1/ping", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "api host")
  })
  mux.HandleFunc("admin.example.com/v1/ping", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "admin host")
  })
  mux.HandleFunc("/v1/ping", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "no host pattern (Host=%s)", r.Host)
  })

  srv := httptest.NewServer(mux)
  defer srv.Close()

  for _, host := range []string{"api.example.com", "admin.example.com", "other.example.com", ""} {
    req, _ := http.NewRequest("GET", srv.URL+"/v1/ping", nil)
    if host != "" {
      req.Host = host
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
      panic(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    label := host
    if label == "" {
      label = "(default 127.0.0.1)"
    }
    fmt.Printf("%-22s %-8s %s\n", label, resp.Status, strings.TrimSpace(string(b)))
  }
}
```

Output (the port on the last line is kernel-assigned and differs every run):

```
api.example.com        200 OK   api host
admin.example.com      200 OK   admin host
other.example.com      200 OK   no host pattern (Host=other.example.com)
(default 127.0.0.1)    200 OK   no host pattern (Host=127.0.0.1:56481)
```

Setting `req.Host` (not the `Host` *header*, which Go overwrites) is how you test host
routing against a loopback server. Host patterns take no wildcards — there is no
`*.example.com` — so for subdomain routing you match on `r.Host` inside a handler or
middleware instead.

## NotFound, Error, Redirect, and serving files

Everything you need for the non-JSON parts of a service, verified in one run. The temp
directory holds a real `app.css`; `fstest.MapFS` provides an in-memory `fs.FS` for
`http.FileServerFS`.

```go
package main

import (
  "fmt"
  "io"
  "net/http"
  "net/http/httptest"
  "os"
  "path/filepath"
  "strings"
  "testing/fstest"
)

func main() {
  dir, err := os.MkdirTemp("", "assets")
  if err != nil {
    panic(err)
  }
  defer os.RemoveAll(dir)
  if err := os.WriteFile(filepath.Join(dir, "app.css"), []byte("body{margin:0}\n"), 0o644); err != nil {
    panic(err)
  }

  memFS := fstest.MapFS{
    "docs/readme.txt": {Data: []byte("from an fs.FS\n")},
  }

  mux := http.NewServeMux()
  mux.Handle("/gone/", http.NotFoundHandler())
  mux.HandleFunc("/notfound", func(w http.ResponseWriter, r *http.Request) {
    http.NotFound(w, r)
  })
  mux.HandleFunc("/boom", func(w http.ResponseWriter, r *http.Request) {
    http.Error(w, "upstream unavailable", http.StatusBadGateway)
  })
  mux.HandleFunc("/old", func(w http.ResponseWriter, r *http.Request) {
    http.Redirect(w, r, "/new", http.StatusMovedPermanently)
  })
  mux.HandleFunc("/one-file", func(w http.ResponseWriter, r *http.Request) {
    http.ServeFile(w, r, filepath.Join(dir, "app.css"))
  })
  mux.Handle("/assets/", http.StripPrefix("/assets/", http.FileServer(http.Dir(dir))))
  mux.Handle("/mem/", http.StripPrefix("/mem/", http.FileServerFS(memFS)))

  srv := httptest.NewServer(mux)
  defer srv.Close()

  noFollow := &http.Client{CheckRedirect: func(*http.Request, []*http.Request) error {
    return http.ErrUseLastResponse
  }}

  for _, p := range []string{"/gone/x", "/notfound", "/boom", "/old",
    "/one-file", "/assets/app.css", "/assets/nope.css", "/mem/docs/readme.txt"} {
    resp, err := noFollow.Get(srv.URL + p)
    if err != nil {
      panic(err)
    }
    b, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("%-22s %-24s ct=%-26q loc=%-6q body=%q\n", p, resp.Status,
      resp.Header.Get("Content-Type"), resp.Header.Get("Location"),
      strings.TrimSpace(string(b)))
  }
}
```

Output:

```
/gone/x                404 Not Found            ct="text/plain; charset=utf-8" loc=""     body="404 page not found"
/notfound              404 Not Found            ct="text/plain; charset=utf-8" loc=""     body="404 page not found"
/boom                  502 Bad Gateway          ct="text/plain; charset=utf-8" loc=""     body="upstream unavailable"
/old                   301 Moved Permanently    ct="text/html; charset=utf-8" loc="/new" body="<a href=\"/new\">Moved Permanently</a>."
/one-file              200 OK                   ct="text/css; charset=utf-8"  loc=""     body="body{margin:0}"
/assets/app.css        200 OK                   ct="text/css; charset=utf-8"  loc=""     body="body{margin:0}"
/assets/nope.css       404 Not Found            ct="text/plain; charset=utf-8" loc=""     body="404 page not found"
/mem/docs/readme.txt   200 OK                   ct="text/plain; charset=utf-8" loc=""     body="from an fs.FS"
```

Notes on each:

**`http.Error(w, msg, code)`** sets `Content-Type: text/plain; charset=utf-8`, sets
`X-Content-Type-Options: nosniff`, writes the status, and writes `msg` plus a newline. It is
the one-line error path, and 16.9 replaces it with something structured.

**`http.Redirect`** writes an HTML body too, for the benefit of clients that render it.
`http.Redirect` with a `GET`-only redirect code and a non-`GET` request still just sets the
header — choosing 301 vs 307 vs 308 is on you.

**`http.ServeFile`** gets you `Content-Type` from the extension (`text/css`), plus
`Last-Modified`, `ETag` handling, `Range` support and conditional requests — all of which
`fmt.Fprintf` on a file's bytes would not.

**`http.StripPrefix` is mandatory with `FileServer`.** `FileServer` resolves `r.URL.Path`
against its root, so mounting at `/assets/` without stripping would look for
`<dir>/assets/app.css`. The prefix you strip must match the mount pattern exactly.

**`http.FileServerFS` (Go 1.22+) takes an `fs.FS`**, which is what you want with
`//go:embed` ([13.13](../13-stdlib/13-embed.md)) — your assets end up inside the binary.
`http.Dir` is the filesystem-backed equivalent, and `http.FS` adapts an `fs.FS` to
`http.FileSystem` the other direction.

`http.ServeFileFS` and `http.ServeContent` are the single-file counterparts;
`ServeContent` takes an `io.ReadSeeker` and a modtime, so it works on content you generate
(16.3).

## What stdlib ServeMux does and does not do

| Capability | stdlib `ServeMux` (1.22+) | Need a router? |
|---|---|---|
| Method matching | yes — `"GET /items"` | no |
| Automatic 405 + `Allow` | yes, computed | no |
| Automatic `HEAD` from `GET` | yes | no |
| Single-segment path variables | yes — `{id}`, `r.PathValue` | no |
| Multi-segment wildcard | yes — `{rest...}` | no |
| Subtree matching | yes — trailing `/` | no |
| Host matching | yes — exact host only | wildcard hosts: yes |
| Specificity-based precedence | yes | no |
| Conflict detection | yes — panics at registration | no |
| Trailing-slash redirect, path cleaning | yes | no |
| **Regexp / typed constraints** (`{id:[0-9]+}`) | **no** | **yes** |
| **Middleware chaining** (`Use`) | no — you nest by hand (16.6) | **yes** |
| **Route groups with their own middleware** | no | **yes** |
| **Subrouters / `Mount`** | no | **yes** |
| **Custom 404 / 405 handlers** | no — the built-ins are fixed | **yes** |
| **Route introspection** (list the route table) | no | **yes** |
| Per-route param type conversion | no | sometimes |

The top half is why a small service can ship on the standard library alone. The bold rows
are what `chi` adds, and 16.7 goes through each of them.

## Quick Reference

| Pattern | Matches |
|---|---|
| `/items` | exactly `/items`, any method |
| `GET /items` | `GET` (and `HEAD`) on `/items` |
| `/items/{id}` | one segment; `r.PathValue("id")` |
| `/files/{rest...}` | one or more remaining segments, slashes included; must be last |
| `/static/` | `/static/` and everything below it (subtree) |
| `/` | every path (subtree at the root) |
| `api.example.com/v1/x` | only requests whose `Host` is exactly that |
| `/items/{$}` | **only** `/items/` exactly — opts a subtree out of matching children |

| Function | Signature | Note |
|---|---|---|
| `(*ServeMux).Handle` | `(pattern string, h Handler)` | panics on a conflict or duplicate |
| `(*ServeMux).HandleFunc` | `(pattern string, f func(w, r))` | same, wraps in `HandlerFunc` |
| `(*ServeMux).Handler` | `(r *Request) (Handler, pattern string)` | look up without serving |
| `(*Request).PathValue` | `(name string) string` | `""` if absent; decoded |
| `(*Request).SetPathValue` | `(name, value string)` | for tests (16.11) |
| `http.NotFound` | `(w, r)` | 404 + `404 page not found` |
| `http.NotFoundHandler` | `() Handler` | the handler form |
| `http.Error` | `(w, msg string, code int)` | plain text + `nosniff` |
| `http.Redirect` | `(w, r, url string, code int)` | sets `Location` + an HTML body |
| `http.RedirectHandler` | `(url string, code int) Handler` | the handler form |
| `http.StripPrefix` | `(prefix string, h Handler) Handler` | required in front of `FileServer` |
| `http.FileServer` | `(root FileSystem) Handler` | use with `http.Dir` |
| `http.FileServerFS` | `(fsys fs.FS) Handler` | 1.22+; pairs with `embed` |
| `http.ServeFile` | `(w, r, name string)` | one file from disk, with `Range`/`ETag` |
| `http.ServeFileFS` | `(w, r, fsys fs.FS, name string)` | 1.22+ |
| `http.ServeContent` | `(w, r, name string, modtime time.Time, c io.ReadSeeker)` | generated content |

## Common Mistakes

**Writing the method in lowercase.** `"get /items"` is parsed as a *path* containing a
space, so the route never matches and nothing warns you; methods are uppercase.

**Expecting `{id}` to match a slash.** It matches exactly one segment, so `/a/b` needs
`{rest...}` or a second wildcard; a one-segment wildcard silently falls through to your
catch-all.

**Forgetting `StripPrefix` in front of `FileServer`.** The file server resolves the full
request path against its root, so every request 404s until you strip the mount prefix.

**Relying on registration order.** Go picks the most specific pattern regardless of order,
and declares a tie a startup panic — so reorder nothing, and fix the ambiguity instead.

**Trusting `r.PathValue` to be safe.** `%2F` decodes to a literal slash and `%20` to a
space, so a path value can contain anything; validate before using it in a file path or a
query.

**Assuming `/items/{id}` also serves `/items`.** It does not, and there is no implicit
redirect between an exact pattern and its parent — register `/items` separately.

## Exercises

1. Register `GET /users/{id}`, `GET /users/me`, and `GET /users/{id}/posts/{postID}` on one
   mux. Request `/users/me`, `/users/42`, and `/users/42/posts/7` and print which pattern
   handled each plus every `PathValue`. Then add `GET /users/{id}/posts/latest` and
   `GET /users/me/posts/{postID}` and report whether it still starts.
2. Build the conflict panic deliberately with two patterns of your own design, capture the
   full message, and identify from the message alone which concrete path is ambiguous.
3. Show the difference between `/items/` and `/items/{$}` by registering both plus
   `/items/{id}`, then requesting `/items/`, `/items/7`, and `/items/7/x`. Explain the
   `{$}` result.
4. Serve a directory of files at `/public/` with `FileServer` plus `StripPrefix`, then
   serve the same files from an `embed.FS` at `/embedded/` with `FileServerFS`. Compare the
   `Content-Type`, `Last-Modified` and `ETag` headers the two produce for the same file.
5. Write a handler that rejects any `r.PathValue("id")` containing a `/` or a `..`, and
   prove it with requests to `/items/%2Fetc%2Fpasswd` and `/items/..%2F..%2Fsecret`. Print
   what `PathValue` returned in each case before your check ran.

---

[← 16.1 Your First net/http Server](01-first-server.md) · [Course Index](../README.md) · [16.3 Handlers, HandlerFunc, and the Interface →](03-handlers.md)
