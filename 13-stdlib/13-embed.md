[← 13.12 log/slog: Structured Logging](12-slog.md) · [Course Index](../README.md) · [13.14 iter: Range-over-Func Iterators →](14-iter.md)

# 13.13 — embed: Embedding Files in the Binary

## What You'll Learn

- The `//go:embed` directive rules, and the three target types it accepts
- The pattern rules, including the `all:` prefix — because dotfiles and `_`-files are **excluded** by default
- Every real compile error, captured: no matching files, `..`, an absolute path, a var inside a func, a missing blank import
- `embed.FS` satisfies `io/fs`, so `http.FileServerFS`, `template.ParseFS` and `fs.Sub` all just work
- The binary-size consequence, measured, and when not to embed

## Coming From Another Language

| Other | Go |
|---|---|
| webpack / Vite bundling assets | the **compiler** does it; no build step, no manifest |
| Node `fs.readFileSync(__dirname + ...)` | `//go:embed` — resolved at compile time, no runtime path |
| Python `importlib.resources` | `embed.FS`, but checked at compile time |
| `include_str!` / `include_bytes!` (Rust) | the closest analogue; Go adds whole directory trees |
| a Docker image with a `COPY assets/` | one static binary, nothing to copy |

The payoff is deployment: a single binary with its templates, static assets, SQL
migrations and default config inside it. No "works on my machine because the `templates/`
directory happened to be next to the binary", no volume mounts, no `COPY` layer. The cost
is that the assets are frozen at build time and the binary grows by their size.

## The directive rules

```go
import _ "embed"

//go:embed path/to/file
var name string
```

Four rules, all enforced by the compiler:

1. The file must **import `embed`** — as `_ "embed"` when the target is `string` or
   `[]byte`, and by name when the target is `embed.FS`.
2. The directive applies to a **package-level** `var` (not inside a function, not a
   `const`).
3. The target type must be `string`, `[]byte`, or `embed.FS`.
4. The directive must appear in the var's **doc comment**, before the declaration.

| Target type | Pattern must match | Use for |
|---|---|---|
| `string` | exactly one file | a template, a version, a default config |
| `[]byte` | exactly one file | a binary blob, an image, a certificate |
| `embed.FS` | any number of files and directories | a tree of assets, templates, migrations |

## Patterns, and the all: prefix

Patterns are `path.Match` globs, **relative to the directory containing the source file**.
A directory pattern embeds the tree recursively — but with one exclusion that surprises
everyone: files whose names begin with `.` or `_` are **left out**. The `all:` prefix
includes them.

This program runs in a module with these real files:

```
version.txt
assets/css/site.css
assets/js/app.js
assets/.hidden-config
assets/_internal.txt
templates/layout.html
templates/index.html
migrations/000001_create_users.up.sql
migrations/000001_create_users.down.sql
migrations/000002_add_plan.up.sql
migrations/000002_add_plan.down.sql
```

```go
package main

import (
  "embed"
  "fmt"
  "io/fs"
)

//go:embed version.txt
var version string

//go:embed version.txt
var versionBytes []byte

//go:embed assets
var assetsFS embed.FS

//go:embed all:assets
var assetsAllFS embed.FS

//go:embed templates/*.html
var templatesFS embed.FS

//go:embed migrations/*.up.sql
//go:embed migrations/*.down.sql
var migrationsFS embed.FS

//go:embed version.txt templates
var multiFS embed.FS

func list(label string, fsys fs.FS) {
  fmt.Println(label)
  _ = fs.WalkDir(fsys, ".", func(p string, d fs.DirEntry, err error) error {
    if err != nil {
      return err
    }
    if d.IsDir() {
      fmt.Printf("  DIR  %s\n", p)
    } else {
      info, _ := d.Info()
      fmt.Printf("  file %-28s %4d bytes\n", p, info.Size())
    }
    return nil
  })
}

func main() {
  fmt.Printf("as a string:  %q\n", version)
  fmt.Printf("as []byte:    %q (len %d)\n", versionBytes, len(versionBytes))

  list("\n--- //go:embed assets (dotfiles and _files EXCLUDED) ---", assetsFS)
  list("\n--- //go:embed all:assets (everything) ---", assetsAllFS)
  list("\n--- //go:embed templates/*.html ---", templatesFS)
  list("\n--- two directives on one var ---", migrationsFS)
  list("\n--- two patterns in one directive ---", multiFS)

  fmt.Println("\n--- embed.FS is read-only and satisfies fs.FS ---")
  b, err := fs.ReadFile(assetsFS, "assets/css/site.css")
  fmt.Printf("  fs.ReadFile: %d bytes, err=%v\n", len(b), err)
  _, err = fs.ReadFile(assetsFS, "assets/.hidden-config")
  fmt.Println("  the excluded dotfile:", err)
  _, err = fs.ReadFile(assetsAllFS, "assets/.hidden-config")
  fmt.Println("  present with all::   ", err == nil)

  fmt.Println("\n--- paths are always slash-separated, relative to the source file ---")
  _, err = fs.ReadFile(assetsFS, "assets/css/nope.css")
  fmt.Println("  missing:", err)

  fmt.Println("\n--- fs.Glob and fs.Sub work ---")
  g, _ := fs.Glob(migrationsFS, "migrations/*.up.sql")
  fmt.Println("  Glob up migrations:", g)
  sub, err := fs.Sub(assetsFS, "assets")
  fmt.Println("  fs.Sub err:", err)
  sg, _ := fs.Glob(sub, "*/*")
  fmt.Println("  Glob through Sub:  ", sg)
}
```

Output:

```
as a string:  "VERSION 1.4.2\n"
as []byte:    "VERSION 1.4.2\n" (len 14)

--- //go:embed assets (dotfiles and _files EXCLUDED) ---
  DIR  .
  DIR  assets
  DIR  assets/css
  file assets/css/site.css            67 bytes
  DIR  assets/js
  file assets/js/app.js               39 bytes

--- //go:embed all:assets (everything) ---
  DIR  .
  DIR  assets
  file assets/.hidden-config          11 bytes
  file assets/_internal.txt            9 bytes
  DIR  assets/css
  file assets/css/site.css            67 bytes
  DIR  assets/js
  file assets/js/app.js               39 bytes

--- //go:embed templates/*.html ---
  DIR  .
  DIR  templates
  file templates/index.html           93 bytes
  file templates/layout.html         159 bytes

--- two directives on one var ---
  DIR  .
  DIR  migrations
  file migrations/000001_create_users.down.sql   18 bytes
  file migrations/000001_create_users.up.sql   72 bytes
  file migrations/000002_add_plan.down.sql   36 bytes
  file migrations/000002_add_plan.up.sql   64 bytes

--- two patterns in one directive ---
  DIR  .
  DIR  templates
  file templates/index.html           93 bytes
  file templates/layout.html         159 bytes
  file version.txt                    14 bytes

--- embed.FS is read-only and satisfies fs.FS ---
  fs.ReadFile: 67 bytes, err=<nil>
  the excluded dotfile: open assets/.hidden-config: file does not exist
  present with all::    true

--- paths are always slash-separated, relative to the source file ---
  missing: open assets/css/nope.css: file does not exist

--- fs.Glob and fs.Sub work ---
  Glob up migrations: [migrations/000001_create_users.up.sql migrations/000002_add_plan.up.sql]
  fs.Sub err: <nil>
  Glob through Sub:   [css/site.css js/app.js]
```

**Compare the first two listings.** `//go:embed assets` embedded two files;
`//go:embed all:assets` embedded four, adding `.hidden-config` and `_internal.txt`. The
default exclusion exists because Go's own tooling treats `_` and `.` prefixes as "ignore
this" (a `_test` directory, an editor's dotfile), but it bites when you genuinely need a
`.env.example`, a `.well-known/` directory for ACME, or a Next.js-style `_next/` build
output. The symptom is a 404 in production and a file that is visibly present in your repo
— and the failure is at **runtime**, because an `embed.FS` with fewer files than you
expected is not a compile error.

**The paths keep the pattern's directory prefix.** `//go:embed assets` gives you keys like
`assets/css/site.css`, not `css/site.css`. Use `fs.Sub(assetsFS, "assets")` to strip it,
as the last section shows — that is what you want before handing the FS to a file server.

**Multiple directives stack onto one var**, and one directive takes multiple
space-separated patterns. Both forms union their matches, so
`//go:embed migrations/*.up.sql` plus `//go:embed migrations/*.down.sql` is equivalent to
one `//go:embed migrations/*.sql`.

**An `embed.FS` always has a `.` root directory** and a directory entry for each level,
which is what makes `fs.WalkDir` work on it.

The full pattern rules: no `..`, no absolute paths, no symlinks (they are not followed and
the file they point at is not embedded), and no path escaping the package directory. A
pattern may not be empty, and `*` does not cross a `/` — but a bare directory name
recurses.

### The compile errors

Every one of these was captured from a real `go build`.

A `string` or `[]byte` target needs the blank import. Forgetting it is the most common
first mistake:

```go
package main

import "fmt"

//go:embed version.txt
var version string

func main() {
  fmt.Println(version)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:5:3: go:embed requires import "embed" (or import _ "embed", if package is not used)
```

A pattern matching nothing:

```go
package main

import (
  _ "embed"
  "fmt"
)

//go:embed nope.txt
var missing string

func main() {
  fmt.Println(missing)
}
```

This does not compile. Output:

```
main.go:8:12: pattern nope.txt: no matching files found
```

A `..` in the pattern, and an absolute path — both rejected as syntax, not as
"not found":

```go
package main

import (
  _ "embed"
  "fmt"
)

//go:embed ../version.txt
var v string

func main() {
  fmt.Println(v)
}
```

This does not compile. Output:

```
main.go:8:12: pattern ../version.txt: invalid pattern syntax
```

With `//go:embed /etc/hosts` instead, the message is the same shape:
`main.go:8:12: pattern /etc/hosts: invalid pattern syntax`.

Note those three errors have **no `# example.com/scratch` header line and no `./` prefix**
on the filename — they come from the module loader resolving patterns, which runs before
the compiler, unlike the import error above.

A var inside a function:

```go
package main

import (
  _ "embed"
  "fmt"
)

func main() {
  //go:embed version.txt
  var v string
  fmt.Println(v)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:9:5: go:embed cannot apply to var inside func
```

And a target of the wrong type gives
`./main.go:9:5: go:embed cannot apply to var of type int`, while using `embed.FS` with
only `import "fmt"` gives the ordinary `./main.go:6:7: undefined: embed`.

**One piece of folklore that is not true on Go 1.27.** The directive is often described as
needing to be *immediately* above the var with nothing in between. In fact a blank line or
an ordinary comment between the directive and the declaration is tolerated — both of these
compile and embed correctly:

```go
package main

import (
  _ "embed"
  "fmt"
)

//go:embed version.txt

var v string

func main() {
  fmt.Printf("blank line between directive and var: %q\n", v)
}
```

Output:

```
blank line between directive and var: "VERSION 1.4.2\n"
```

The actual requirement is that the directive be part of the var's doc comment block. Write
it directly above anyway — it is clearer, and it is what every reader expects.

## embed.FS is an io/fs.FS

This is the design decision that makes `embed` useful rather than merely convenient:
`embed.FS` implements `fs.FS`, `fs.ReadDirFS` and `fs.ReadFileFS`, so every consumer of
`io/fs` (13.10) accepts it unchanged.

```go
package main

import (
  "embed"
  "fmt"
  "html/template"
  "io"
  "io/fs"
  "net/http"
  "net/http/httptest"
  "strings"
)

//go:embed assets
var assetsFS embed.FS

//go:embed templates/*.html
var templatesFS embed.FS

func main() {
  fmt.Println("=== http.FileServerFS serving embedded assets (via httptest) ===")
  static, err := fs.Sub(assetsFS, "assets")
  if err != nil {
    panic(err)
  }
  mux := http.NewServeMux()
  mux.Handle("/static/", http.StripPrefix("/static/", http.FileServerFS(static)))

  srv := httptest.NewServer(mux)
  defer srv.Close()

  for _, p := range []string{"/static/css/site.css", "/static/js/app.js", "/static/nope.css", "/static/"} {
    resp, err := http.Get(srv.URL + p)
    if err != nil {
      fmt.Println("  client err:", err)
      continue
    }
    body, _ := io.ReadAll(resp.Body)
    resp.Body.Close()
    fmt.Printf("  GET %-22s %d  content-type=%-24s %d bytes\n",
      p, resp.StatusCode, resp.Header.Get("Content-Type"), len(body))
  }

  resp, _ := http.Get(srv.URL + "/static/css/site.css")
  body, _ := io.ReadAll(resp.Body)
  resp.Body.Close()
  fmt.Printf("\n  the actual bytes served:\n%s", indent(string(body)))

  fmt.Println("=== html/template.ParseFS on embedded templates ===")
  tmpl, err := template.ParseFS(templatesFS, "templates/*.html")
  if err != nil {
    panic(err)
  }
  fmt.Print("  templates defined:")
  for _, t := range tmpl.Templates() {
    fmt.Printf(" %q", t.Name())
  }
  fmt.Println()

  data := struct {
    Title string
    Items []string
  }{Title: "Embedded <Page>", Items: []string{"alpha", "beta & gamma"}}

  var out strings.Builder
  if err := tmpl.ExecuteTemplate(&out, "layout.html", data); err != nil {
    panic(err)
  }
  fmt.Printf("  rendered:\n%s\n", indent(out.String()))

  fmt.Println("=== ParseFS with a pattern matching nothing is an error ===")
  _, err = template.ParseFS(templatesFS, "templates/*.tmpl")
  fmt.Println("  err:", err)
}

func indent(s string) string {
  var b strings.Builder
  for _, line := range strings.Split(strings.TrimRight(s, "\n"), "\n") {
    b.WriteString("    " + line + "\n")
  }
  return b.String()
}
```

Output (a real HTTP round trip through `httptest`, not a simulation):

```
=== http.FileServerFS serving embedded assets (via httptest) ===
  GET /static/css/site.css   200  content-type=text/css; charset=utf-8  67 bytes
  GET /static/js/app.js      200  content-type=text/javascript; charset=utf-8 39 bytes
  GET /static/nope.css       404  content-type=text/plain; charset=utf-8 19 bytes
  GET /static/               200  content-type=text/html; charset=utf-8 127 bytes

  the actual bytes served:
    body { font-family: system-ui; margin: 2rem; }
    h1 { color: #0a7; }
=== html/template.ParseFS on embedded templates ===
  templates defined: "index.html" "content" "layout.html"
  rendered:
    <!doctype html>
    <html><head><title>Embedded &lt;Page&gt;</title>
    <link rel="stylesheet" href="/static/css/site.css">
    </head>
    <body><h1>Embedded &lt;Page&gt;</h1><ul><li>alpha</li><li>beta &amp; gamma</li></ul></body></html>

=== ParseFS with a pattern matching nothing is an error ===
  err: template: pattern matches no files: `templates/*.tmpl`
```

**`http.FileServerFS` plus `fs.Sub` is the whole static-asset story.** Four lines, and the
`Content-Type` is sniffed from the extension, `404`s are handled, directory listings are
generated, and `Range` requests work — because `embed.FS` files implement `io.Seeker`,
which `FileServerFS` requires. (`http.FS(fsys)` is the older adapter returning an
`http.FileSystem`; `FileServerFS` takes the `fs.FS` directly and is what to use now.)

**`fs.Sub` is doing the important work.** Without it the URL would have to be
`/static/assets/css/site.css`, because the embedded keys carry the `assets/` prefix. One
`fs.Sub` makes the URL space match the directory layout.

**`template.ParseFS` parses straight out of the binary,** and note `tmpl.Templates()`
lists three names: the two files plus `"content"` from the `{{define}}` inside
`index.html`. The rendered output shows `html/template`'s contextual escaping doing its
job — `<Page>` became `&lt;Page&gt;` and `beta & gamma` became `beta &amp; gamma`.

**`ParseFS` errors when the pattern matches nothing**, which is a useful second line of
defence: the embed succeeded (the files are there) but your parse pattern was wrong, and
you find out at startup rather than on the first request.

`text/template.ParseFS` works identically for non-HTML output, and `fs.WalkDir`,
`fs.Glob`, `fs.ReadFile`, `fs.ReadDir` and `fstest.TestFS` all accept an `embed.FS` too.

## Embedded SQL migrations

The canonical non-asset use. Shipping schema migrations inside the binary means the
deployed artifact can always migrate itself, with no second file to copy and no version
skew between binary and SQL.

```go
package main

import (
  "embed"
  "fmt"
  "io/fs"
  "path"
  "sort"
  "strings"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

type migration struct {
  Version string
  Name    string
  Up      string
  Down    string
}

func loadMigrations(fsys fs.FS, dir string) ([]migration, error) {
  entries, err := fs.ReadDir(fsys, dir)
  if err != nil {
    return nil, fmt.Errorf("reading %s: %w", dir, err)
  }
  byVersion := map[string]*migration{}
  var order []string
  for _, e := range entries {
    name := e.Name()
    if !strings.HasSuffix(name, ".sql") {
      continue
    }
    version, rest, ok := strings.Cut(name, "_")
    if !ok {
      return nil, fmt.Errorf("%s: expected <version>_<name>.<up|down>.sql", name)
    }
    body, err := fs.ReadFile(fsys, path.Join(dir, name))
    if err != nil {
      return nil, err
    }
    m, seen := byVersion[version]
    if !seen {
      m = &migration{Version: version}
      byVersion[version] = m
      order = append(order, version)
    }
    switch {
    case strings.HasSuffix(rest, ".up.sql"):
      m.Name = strings.TrimSuffix(rest, ".up.sql")
      m.Up = string(body)
    case strings.HasSuffix(rest, ".down.sql"):
      m.Down = string(body)
    default:
      return nil, fmt.Errorf("%s: missing .up.sql or .down.sql suffix", name)
    }
  }
  sort.Strings(order)
  out := make([]migration, 0, len(order))
  for _, v := range order {
    m := byVersion[v]
    if m.Up == "" || m.Down == "" {
      return nil, fmt.Errorf("version %s: missing an up or down file", v)
    }
    out = append(out, *m)
  }
  return out, nil
}

func main() {
  ms, err := loadMigrations(migrationsFS, "migrations")
  if err != nil {
    panic(err)
  }
  fmt.Println("loaded", len(ms), "migrations from the binary:")
  for _, m := range ms {
    fmt.Printf("  %s %-13s up=%-2d down=%-2d bytes\n", m.Version, m.Name, len(m.Up), len(m.Down))
    fmt.Printf("     up:   %s", m.Up)
    fmt.Printf("     down: %s", m.Down)
  }
}
```

Output:

```
loaded 2 migrations from the binary:
  000001 create_users  up=72 down=18 bytes
     up:   CREATE TABLE users (id SERIAL PRIMARY KEY, email TEXT NOT NULL UNIQUE);
     down: DROP TABLE users;
  000002 add_plan      up=64 down=36 bytes
     up:   ALTER TABLE users ADD COLUMN plan TEXT NOT NULL DEFAULT 'free';
     down: ALTER TABLE users DROP COLUMN plan;
```

Note `fs.ReadDir` returns entries **sorted by filename**, which is why the zero-padded
`000001`/`000002` naming convention gives you correct ordering for free — and why
`sort.Strings` on the versions is belt-and-braces rather than essential.

In a real service you do not hand-roll this. [17.6](../17-database/06-migrations.md) wires
the same `embed.FS` into golang-migrate through its `iofs` source driver, which is three
lines:

```go
//go:embed migrations/*.sql
var migrationsFS embed.FS

src, err := iofs.New(migrationsFS, "migrations")
m, err := migrate.NewWithSourceInstance("iofs", src, dbURL)
```

`iofs.New` takes any `fs.FS` plus a subdirectory, so the embedded tree and a real
directory are interchangeable — and 17.6 also captures what `iofs.New` does with a
duplicate version or a missing directory. The embed half is identical to what is above.

## The binary-size consequence

Embedded files are stored **uncompressed** in the binary's data section. Two builds of the
same trivial program, one with a 2 MiB asset embedded and one without:

```bash
ls -l bin-noembed bin-withembed
```

Output (darwin/arm64, go1.27.1 — absolute sizes are toolchain- and platform-specific; the
*delta* is the point):

```
-rwxr-xr-x@ 1 raj  staff  2429714 Oct 26 16:06 bin-noembed
-rwxr-xr-x@ 1 raj  staff  4561730 Oct 26 16:06 bin-withembed
```

The asset is 2,097,152 bytes and the binary grew by 2,132,016 bytes — a 2.00 MiB file costs
2.03 MiB of binary, the small excess being the file table and padding. There is no
compression and no deduplication, so this is a reliable linear rule: **embedding N bytes
of assets adds N bytes to your binary.**

That matters in three places. Container image layers and registry pulls get bigger. Your
CI artifact upload gets slower. And memory: the embedded data is in the binary's mapped
pages, so it is demand-paged by the OS rather than heap-allocated — reading a 100 MB
embedded file does not allocate 100 MB of heap, but it does mean a 100 MB binary.

### When not to embed

- **Large or numerous media.** Images, video, fonts at scale belong on a CDN or in object
  storage. A 200 MB binary is a deployment problem.
- **Anything you want to change without a rebuild.** Operator-editable config, feature
  flags, email templates the marketing team owns, locale files added after release. Embed
  the *defaults* and read an override from disk or a config service.
- **Secrets.** `strings` on the binary reveals them; anyone with the artifact has them.
  Use environment variables or a secret manager ([18.2](../18-production/02-config.md)).
- **Content that differs per deployment.** A tenant's branding, environment-specific
  endpoints. That is configuration, not an asset.

Embed when the file is logically part of the program: templates, migrations, static assets
for the app's own UI, a default config, a timezone or lookup table, a test fixture.

A hybrid pattern that gets you both, and which is what a real service does:

```go
//go:embed templates
var embeddedTemplates embed.FS

func templateFS(dir string) (fs.FS, error) {
  if dir != "" {
    return os.DirFS(dir), nil
  }
  return fs.Sub(embeddedTemplates, "templates")
}
```

With `-templates ./templates` you edit a file and reload; with the flag unset you get the
embedded copy. The code downstream takes an `fs.FS` and cannot tell the difference, which
is the `io/fs` abstraction paying for itself.

For version strings, embedding a `version.txt` is one option, but `-ldflags "-X
main.version=$(git describe)"` ([15.1](../15-tooling/01-go-build.md)) injects it at link
time with no file to keep in sync — use ldflags for build metadata, `embed` for content.

## Quick Reference

| Need | Write |
|---|---|
| One file as text | `import _ "embed"` + `//go:embed f.txt` + `var s string` |
| One file as bytes | same, with `var b []byte` |
| A directory tree | `import "embed"` + `//go:embed dir` + `var f embed.FS` |
| Include dotfiles / `_`-files | `//go:embed all:dir` |
| Several patterns | `//go:embed a b/*.html` or stacked `//go:embed` lines |
| Strip the directory prefix | `sub, err := fs.Sub(f, "dir")` |
| Read one file | `fs.ReadFile(f, "dir/name")` — slash-separated, with the prefix |
| List a directory | `fs.ReadDir(f, "dir")` — sorted by filename |
| Walk the tree | `fs.WalkDir(f, ".", fn)` |
| Serve static assets | `http.Handle("/static/", http.StripPrefix("/static/", http.FileServerFS(sub)))` |
| Parse templates | `template.ParseFS(f, "templates/*.html")` |
| Migrations via golang-migrate | `iofs.New(f, "migrations")` (17.6) |
| Build metadata instead | `-ldflags "-X main.version=..."` (15.1) |

| Trap | Reality |
|---|---|
| `//go:embed dir` and a missing `.env.example` | dotfiles and `_`-files are **excluded**; use `all:` |
| Expecting `css/site.css` as the key | keys keep the pattern prefix (`assets/css/site.css`) |
| `//go:embed ../shared` | `invalid pattern syntax`; nothing outside the package dir |
| A symlink in an embedded directory | not followed, not embedded, no error |
| `var s string` with no `embed` import | `go:embed requires import "embed"` |
| `//go:embed` on a var inside a function | `cannot apply to var inside func` |
| A typo'd pattern | **compile** error for no matches, but a *wrong* match is silent |
| Embedding a secret | `strings` on the binary reveals it |
| Expecting compression | stored uncompressed; N bytes of assets = N bytes of binary |
| Editing an asset and restarting | you must **rebuild**; `go run` does re-embed |

## Common Mistakes

**Forgetting `import _ "embed"` for a `string` or `[]byte` target.** The compiler tells you
exactly this, but it is the first thing everyone hits.

**Assuming a directory pattern includes dotfiles.** `.well-known/`, `.env.example` and
`_next/` are silently skipped; use the `all:` prefix.

**Using the un-prefixed path as the `fs.FS` key.** `//go:embed assets` produces
`assets/...` keys; either include the prefix or strip it with `fs.Sub`.

**Embedding assets you want to edit in production.** They are frozen at build time; embed
the defaults and allow a directory override.

**Embedding large media into the binary.** The binary grows byte for byte with no
compression; put media behind a CDN.

**Putting a secret in an embedded file.** Anyone with the binary can read it; use the
environment or a secret manager.

## Exercises

1. Build a small web app whose templates and static assets are embedded, served with
   `template.ParseFS` and `http.FileServerFS`, and prove with `httptest` that a CSS file
   returns 200 with `text/css` and a missing one returns 404.
2. Implement `func AssetFS(devDir string) (fs.FS, error)` returning `os.DirFS(devDir)` when
   the flag is set and an `fs.Sub` of the embedded tree otherwise. Write one test that runs
   the same assertions against both.
3. Add a `.well-known/security.txt` to an embedded tree, show with `fs.WalkDir` that
   `//go:embed assets` omits it, then fix it with `all:` and show the output change.
4. Write a test that walks the embedded template FS and asserts every `.html` file parses
   and that every `{{define}}` name is unique — a build-time guarantee that a renamed
   template cannot ship broken.
5. Measure it yourself: build the same program with a 0, 1 MiB and 10 MiB embedded asset,
   tabulate the binary sizes, and confirm the linear relationship. Then try `gzip`-ing the
   asset before embedding and decompressing at startup, and report the trade-off.
6. Extend `loadMigrations` to reject a duplicate version, a gap in the sequence, and a
   missing down file, with a test per failure mode using `fstest.MapFS` instead of real
   embedded files.

---

[← 13.12 log/slog: Structured Logging](12-slog.md) · [Course Index](../README.md) · [13.14 iter: Range-over-Func Iterators →](14-iter.md)
