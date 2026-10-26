[← 13.9 math, math/big, and math/rand](09-math-rand.md) · [Course Index](../README.md) · [13.11 flag: Command-Line Parsing →](11-flag.md)

# 13.10 — path/filepath and io/fs

## What You'll Learn

- `path` vs `path/filepath`: the rule is filesystem vs URL/import-path/`fs.FS` key
- The edge cases that bite: `Join` dropping empties, `Clean("")`, `Dir("/")`, `Rel` across roots
- `filepath.IsLocal` as the **path-traversal** defence, and why a `Rel`-prefix check is not enough
- `WalkDir` over `Walk`, `fs.SkipDir`/`fs.SkipAll`, and handling the error argument
- `io/fs` as the abstraction that makes `embed`, zip and `fstest.MapFS` interchangeable

## Coming From Another Language

| Other | Go |
|---|---|
| Node `path.join` | `filepath.Join` — but it **Cleans**, so `..` is resolved away |
| Python `os.path.join("/a", "/b")` → `"/b"` | `filepath.Join("/a", "/b")` → `"/a/b"`; absolute later elements do **not** reset |
| Python `pathlib.Path` objects | plain `string`; there is no path type |
| `glob("**/*.go")` recursing | Go's `Glob` does **not** support `**`; use `WalkDir` |
| shell `[!abc]` negated class | Go uses `[^abc]`; `!` is a **literal** |
| `os.walk` / `fs.readdir` | `filepath.WalkDir` with a callback, or `fs.WalkDir` over any `fs.FS` |

Paths in Go are strings, which means nothing stops you concatenating them wrongly and
nothing tells you whether a given string is slash-separated or OS-separated. The rule that
resolves almost every question:

**Use `path/filepath` for anything that touches the actual filesystem. Use `path` for URL
paths, import paths, and `fs.FS` keys — anything that is slash-separated by definition.**

On Linux and macOS the two are identical, which is exactly why the mistake survives until
someone runs your code on Windows.

## The filepath reference

| Func | Signature | Note |
|---|---|---|
| `Join` | `(elem ...string) string` | concatenates **and Cleans**; drops empty elements |
| `Split` | `(path string) (dir, file string)` | `dir` keeps its trailing separator |
| `Dir` | `(path string) string` | `"."` for a bare filename, `"/"` for `"/"` |
| `Base` | `(path string) string` | `"."` for `""`, `"/"` for `"/"` |
| `Ext` | `(path string) string` | from the **last** dot, including it; `""` if none |
| `Clean` | `(path string) string` | lexical only — never touches the disk |
| `Abs` | `(path string) (string, error)` | joins with the cwd and Cleans |
| `Rel` | `(base, targ string) (string, error)` | **errors** if one is absolute and the other is not |
| `IsAbs` | `(path string) bool` | |
| `IsLocal` | `(path string) bool` | stays inside the current directory — the security primitive |
| `Localize` | `(path string) (string, error)` | slash path → OS path, rejecting unsafe input |
| `ToSlash` / `FromSlash` | `(path string) string` | convert separators |
| `VolumeName` | `(path string) string` | `""` on Unix, `"C:"` on Windows |
| `SplitList` | `(path string) []string` | splits a `PATH`-style list on `ListSeparator` |
| `Glob` | `(pattern string) ([]string, error)` | **no** `**`, no recursion |
| `Match` | `(pattern, name string) (bool, error)` | one path element per `*` |
| `EvalSymlinks` | `(path string) (string, error)` | resolves links; **errors** if the path is missing |
| `WalkDir` | `(root string, fn fs.WalkDirFunc) error` | **preferred** |
| `Walk` | `(root string, fn WalkFunc) error` | older; stats every entry |

### Join, Clean, and the lexical functions

```go
package main

import (
  "fmt"
  "path"
  "path/filepath"
)

func main() {
  fmt.Println("--- Join cleans and drops empty elements ---")
  cases := [][]string{
    {"a", "b", "c"},
    {"a/", "/b"},
    {"a", "", "b"},
    {"", ""},
    {"a", "..", "b"},
    {"/a", "b"},
    {"a", "./b"},
    {"a//b", "c"},
  }
  for _, c := range cases {
    fmt.Printf("  Join(%-26s) = %q\n", fmt.Sprintf("%q", c), filepath.Join(c...))
  }

  fmt.Println("\n--- Clean ---")
  for _, p := range []string{"a/../../b", "", "a/./b", "a//b/", "/../a", "./a"} {
    fmt.Printf("  Clean(%-12q) = %q\n", p, filepath.Clean(p))
  }

  fmt.Println("\n--- Split, Dir, Base, Ext ---")
  fmt.Printf("  %-14s %-14s %-10s %-10s %s\n", "path", "Dir", "Base", "Ext", "Split")
  for _, p := range []string{"/a/b/c.txt", "/", "", "a/", "c.txt", "a/.hidden",
    "archive.tar.gz", "noext", "/a/b/", "."} {
    d, f := filepath.Split(p)
    fmt.Printf("  %-14q %-14q %-10q %-10q (%q, %q)\n",
      p, filepath.Dir(p), filepath.Base(p), filepath.Ext(p), d, f)
  }

  fmt.Println("\n--- IsAbs, IsLocal ---")
  for _, p := range []string{"/a/b", "a/b", "./a", "../a", "", "/", "a/../b", "a/../../b"} {
    fmt.Printf("  %-12q IsAbs=%-6v IsLocal=%v\n", p, filepath.IsAbs(p), filepath.IsLocal(p))
  }

  fmt.Println("\n--- path (slash-only) vs filepath (OS-specific) ---")
  fmt.Println("  path.Join(\"a\", \"b\"):     ", path.Join("a", "b"))
  fmt.Println("  filepath.Join(\"a\", \"b\"): ", filepath.Join("a", "b"))
  fmt.Println("  filepath.ToSlash(\"a/b\"):  ", filepath.ToSlash("a/b"))
  fmt.Println("  filepath.FromSlash(\"a/b\"):", filepath.FromSlash("a/b"))
  fmt.Println("  filepath.Separator:        ", string(filepath.Separator))
  fmt.Println("  filepath.ListSeparator:    ", string(filepath.ListSeparator))
  fmt.Println("  VolumeName(\"/a/b\"):      ", filepath.VolumeName("/a/b"))
  fmt.Printf("  SplitList(\"/a:/b:/c\"):    %q\n", filepath.SplitList("/a:/b:/c"))
  fmt.Printf("  SplitList(\"\"):            %q\n", filepath.SplitList(""))
}
```

Output (captured on darwin/arm64, so `Separator` is `/` and `VolumeName` is empty):

```
--- Join cleans and drops empty elements ---
  Join(["a" "b" "c"]             ) = "a/b/c"
  Join(["a/" "/b"]               ) = "a/b"
  Join(["a" "" "b"]              ) = "a/b"
  Join(["" ""]                   ) = ""
  Join(["a" ".." "b"]            ) = "b"
  Join(["/a" "b"]                ) = "/a/b"
  Join(["a" "./b"]               ) = "a/b"
  Join(["a//b" "c"]              ) = "a/b/c"

--- Clean ---
  Clean("a/../../b" ) = "../b"
  Clean(""          ) = "."
  Clean("a/./b"     ) = "a/b"
  Clean("a//b/"     ) = "a/b"
  Clean("/../a"     ) = "/a"
  Clean("./a"       ) = "a"

--- Split, Dir, Base, Ext ---
  path           Dir            Base       Ext        Split
  "/a/b/c.txt"   "/a/b"         "c.txt"    ".txt"     ("/a/b/", "c.txt")
  "/"            "/"            "/"        ""         ("/", "")
  ""             "."            "."        ""         ("", "")
  "a/"           "a"            "a"        ""         ("a/", "")
  "c.txt"        "."            "c.txt"    ".txt"     ("", "c.txt")
  "a/.hidden"    "a"            ".hidden"  ".hidden"  ("a/", ".hidden")
  "archive.tar.gz" "."            "archive.tar.gz" ".gz"      ("", "archive.tar.gz")
  "noext"        "."            "noext"    ""         ("", "noext")
  "/a/b/"        "/a/b"         "b"        ""         ("/a/b/", "")
  "."            "."            "."        "."        ("", ".")

--- IsAbs, IsLocal ---
  "/a/b"       IsAbs=true   IsLocal=false
  "a/b"        IsAbs=false  IsLocal=true
  "./a"        IsAbs=false  IsLocal=true
  "../a"       IsAbs=false  IsLocal=false
  ""           IsAbs=false  IsLocal=false
  "/"          IsAbs=true   IsLocal=false
  "a/../b"     IsAbs=false  IsLocal=true
  "a/../../b"  IsAbs=false  IsLocal=false

--- path (slash-only) vs filepath (OS-specific) ---
  path.Join("a", "b"):      a/b
  filepath.Join("a", "b"):  a/b
  filepath.ToSlash("a/b"):   a/b
  filepath.FromSlash("a/b"): a/b
  filepath.Separator:         /
  filepath.ListSeparator:     :
  VolumeName("/a/b"):       
  SplitList("/a:/b:/c"):    ["/a" "/b" "/c"]
  SplitList(""):            []
```

The lines worth memorizing.

**`Join` Cleans, which means it resolves `..`.** `Join("a", "..", "b")` is `"b"`, not
`"a/../b"` — convenient, and also the entire path-traversal attack surface: if an element
came from a user, `Join` walks out of your directory. Section below.

**`Join` drops empty elements** rather than doubling the separator, so
`Join("a", "", "b")` is `"a/b"` and `Join("", "")` is `""` — the one case where `Join`
does not return a Cleaned path, since `Clean("")` is `"."`.

**`filepath.Join("/a", "b")` is `/a/b`.** Unlike Python's `os.path.join`, a later absolute
element does **not** discard the earlier ones. Go just concatenates and Cleans.

**`Clean("")` is `"."`,** not `""`. An empty path means "the current directory"
everywhere in this package, and that is also why `Dir("")` and `Base("")` are both `"."`.

**`Dir("/")` and `Base("/")` are both `"/"`.** The root is its own parent. And `Base("a/")`
is `"a"` — a trailing separator is stripped first, so `Base` never returns `""` except for
a path that is entirely separators.

**`Ext` takes everything from the last dot.** `Ext("archive.tar.gz")` is `".gz"`, not
`".tar.gz"`. `Ext("a/.hidden")` is `".hidden"` — a dotfile looks like an extension with no
name, which breaks the naive "strip the extension to get the name" operation. And
`Ext(".")` is `"."`.

### Rel, Abs, Match and Glob

```go
package main

import (
  "fmt"
  "path/filepath"
)

func main() {
  fmt.Println("--- Rel ---")
  pairs := [][2]string{
    {"/a/b", "/a/b/c/d"},
    {"/a/b", "/a/x"},
    {"/a/b", "/a/b"},
    {"/a/b", "a/b"},
    {"a/b", "/a/b"},
    {"/", "/a"},
  }
  for _, pr := range pairs {
    r, err := filepath.Rel(pr[0], pr[1])
    fmt.Printf("  Rel(%-8q, %-12q) = %-12q err=%v\n", pr[0], pr[1], r, err)
  }

  fmt.Println("\n--- Match: one path element at a time, no recursion ---")
  type mc struct{ pat, name string }
  for _, c := range []mc{
    {"*.go", "main.go"},
    {"*.go", "sub/main.go"},
    {"*/*.go", "sub/main.go"},
    {"**/*.go", "sub/main.go"},
    {"**/*.go", "a/b/main.go"},
    {"m?in.go", "main.go"},
    {"[mn]ain.go", "main.go"},
    {"[^mn]ain.go", "main.go"},
    {"[!mn]ain.go", "main.go"},
    {"a[", "a["},
  } {
    ok, err := filepath.Match(c.pat, c.name)
    fmt.Printf("  Match(%-12q, %-14q) = %-6v err=%v\n", c.pat, c.name, ok, err)
  }
}
```

Output:

```
--- Rel ---
  Rel("/a/b"  , "/a/b/c/d"  ) = "c/d"        err=<nil>
  Rel("/a/b"  , "/a/x"      ) = "../x"       err=<nil>
  Rel("/a/b"  , "/a/b"      ) = "."          err=<nil>
  Rel("/a/b"  , "a/b"       ) = ""           err=Rel: can't make a/b relative to /a/b
  Rel("a/b"   , "/a/b"      ) = ""           err=Rel: can't make /a/b relative to a/b
  Rel("/"     , "/a"        ) = "a"          err=<nil>

--- Match: one path element at a time, no recursion ---
  Match("*.go"      , "main.go"     ) = true   err=<nil>
  Match("*.go"      , "sub/main.go" ) = false  err=<nil>
  Match("*/*.go"    , "sub/main.go" ) = true   err=<nil>
  Match("**/*.go"   , "sub/main.go" ) = true   err=<nil>
  Match("**/*.go"   , "a/b/main.go" ) = false  err=<nil>
  Match("m?in.go"   , "main.go"     ) = true   err=<nil>
  Match("[mn]ain.go", "main.go"     ) = true   err=<nil>
  Match("[^mn]ain.go", "main.go"     ) = false  err=<nil>
  Match("[!mn]ain.go", "main.go"     ) = true   err=<nil>
  Match("a["        , "a["          ) = false  err=syntax error in pattern
```

**`Rel` errors when one path is absolute and the other is not** — it is purely lexical,
with no access to the working directory, so it cannot bridge the two. Call `filepath.Abs`
on both first if you do not control the inputs. It can also return a `..`-prefixed result
(`"../x"`), which matters below.

**`*` never crosses a separator.** `Match("*.go", "sub/main.go")` is false. So `**` is not
a recursive wildcard — it is just two `*`s in one element, which is why `"**/*.go"`
matches `sub/main.go` (two elements) but **not** `a/b/main.go` (three). If you want
recursion, that is `WalkDir`.

**Go's negated character class is `[^abc]`, not `[!abc]`.** Look at the last two match
lines: `[^mn]ain.go` correctly fails to match `main.go`, while `[!mn]ain.go` **matches**
it — because `!` is not a metacharacter here, so that class means "one of `!`, `m`, `n`".
Every shell and most glob libraries use `!`, so this silently does the opposite of what
you meant. The only returned error is `filepath.ErrBadPattern`, which `errors.Is` matches.

### Glob and WalkDir over a real tree

These run against a tree created on disk:

```
tree/
  .env
  .git/objects/abc
  README.md
  cmd/app/main.go
  cmd/app/main_test.go
  docs/api.md
  docs/guide.md
  go.mod
  internal/store/db.go
  internal/store/db_test.go
  node_modules/pkg/index.js
```

```go
package main

import (
  "fmt"
  "io/fs"
  "path/filepath"
  "strings"
)

func main() {
  fmt.Println("--- Glob: one level per *, and ** is NOT special ---")
  for _, pat := range []string{
    "tree/*",
    "tree/*/*.go",
    "tree/*/*/*.go",
    "tree/**/*.go",
    "tree/nope/*",
    "tree/[",
  } {
    m, err := filepath.Glob(pat)
    fmt.Printf("  %-18q -> %v  err=%v\n", pat, m, err)
  }

  fmt.Println("\n--- WalkDir: every entry, in lexical order ---")
  err := filepath.WalkDir("tree", func(p string, d fs.DirEntry, err error) error {
    if err != nil {
      return err
    }
    kind := "file"
    if d.IsDir() {
      kind = "DIR "
    }
    fmt.Printf("  %s %s\n", kind, p)
    return nil
  })
  fmt.Println("  walk err:", err)

  fmt.Println("\n--- SkipDir prunes a subtree; the .go files that remain ---")
  var found []string
  err = filepath.WalkDir("tree", func(p string, d fs.DirEntry, err error) error {
    if err != nil {
      return err
    }
    name := d.Name()
    if d.IsDir() && (name == ".git" || name == "node_modules") {
      fmt.Println("  skipping dir:", p)
      return fs.SkipDir
    }
    if d.IsDir() && name != "tree" && strings.HasPrefix(name, ".") {
      return fs.SkipDir
    }
    if !d.IsDir() && filepath.Ext(p) == ".go" {
      found = append(found, p)
    }
    return nil
  })
  fmt.Println("  .go files:", found, "err:", err)

  fmt.Println("\n--- SkipAll stops the whole walk ---")
  count := 0
  err = filepath.WalkDir("tree", func(p string, d fs.DirEntry, err error) error {
    if err != nil {
      return err
    }
    count++
    if count == 4 {
      return fs.SkipAll
    }
    return nil
  })
  fmt.Println("  visited", count, "entries then stopped; err:", err)

  fmt.Println("\n--- a returned error (not a sentinel) aborts and propagates ---")
  sentinel := fmt.Errorf("found what I wanted")
  err = filepath.WalkDir("tree", func(p string, d fs.DirEntry, err error) error {
    if err != nil {
      return err
    }
    if d.Name() == ".env" {
      return sentinel
    }
    return nil
  })
  fmt.Println("  err:", err, " is sentinel:", err == sentinel)

  fmt.Println("\n--- walking a path that does not exist: the func is called WITH the error ---")
  err = filepath.WalkDir("tree/nope", func(p string, d fs.DirEntry, err error) error {
    fmt.Printf("  callback: path=%q d==nil:%v err=%v\n", p, d == nil, err)
    return err
  })
  fmt.Println("  walk returned:", err)
}
```

Output:

```
--- Glob: one level per *, and ** is NOT special ---
  "tree/*"           -> [tree/.env tree/.git tree/README.md tree/cmd tree/docs tree/go.mod tree/internal tree/node_modules]  err=<nil>
  "tree/*/*.go"      -> []  err=<nil>
  "tree/*/*/*.go"    -> [tree/cmd/app/main.go tree/cmd/app/main_test.go tree/internal/store/db.go tree/internal/store/db_test.go]  err=<nil>
  "tree/**/*.go"     -> []  err=<nil>
  "tree/nope/*"      -> []  err=<nil>
  "tree/["           -> []  err=syntax error in pattern

--- WalkDir: every entry, in lexical order ---
  DIR  tree
  file tree/.env
  DIR  tree/.git
  DIR  tree/.git/objects
  file tree/.git/objects/abc
  file tree/README.md
  DIR  tree/cmd
  DIR  tree/cmd/app
  file tree/cmd/app/main.go
  file tree/cmd/app/main_test.go
  DIR  tree/docs
  file tree/docs/api.md
  file tree/docs/guide.md
  file tree/go.mod
  DIR  tree/internal
  DIR  tree/internal/store
  file tree/internal/store/db.go
  file tree/internal/store/db_test.go
  DIR  tree/node_modules
  DIR  tree/node_modules/pkg
  file tree/node_modules/pkg/index.js
  walk err: <nil>

--- SkipDir prunes a subtree; the .go files that remain ---
  skipping dir: tree/.git
  skipping dir: tree/node_modules
  .go files: [tree/cmd/app/main.go tree/cmd/app/main_test.go tree/internal/store/db.go tree/internal/store/db_test.go] err: <nil>

--- SkipAll stops the whole walk ---
  visited 4 entries then stopped; err: <nil>

--- a returned error (not a sentinel) aborts and propagates ---
  err: found what I wanted  is sentinel: true

--- walking a path that does not exist: the func is called WITH the error ---
  callback: path="tree/nope" d==nil:true err=lstat tree/nope: no such file or directory
  walk returned: lstat tree/nope: no such file or directory
```

**`Glob` does not recurse, and `**` is not supported.** `"tree/**/*.go"` returned
**nothing** — it is two `*`s in one element, so it looks for a directory whose name
matches `**` literally-ish, not a recursive descent. You must know the depth
(`"tree/*/*/*.go"` works) or use `WalkDir`. Also note `Glob` with no matches returns an
empty slice and a **nil error** — a missing directory is not an error. The only error is a
malformed pattern.

Note `"tree/*"` includes `.env` — unlike many shells, `Glob` does not hide dotfiles.

**The `WalkDirFunc` contract.** The signature is
`func(path string, d fs.DirEntry, err error) error`, and the three rules are:

1. **Check `err` first.** It is non-nil when the entry could not be read (permissions, a
   file deleted mid-walk, a broken root). When it is non-nil, `d` may be nil — so
   dereferencing `d` before the check is a nil panic. Return the error to abort, or return
   `nil` to **ignore the entry and continue**, which is what you want for a permissions
   error in a user-wide scan.
2. **Return `fs.SkipDir`** from a directory entry to prune that whole subtree. Returning it
   from a *file* entry skips the rest of that file's containing directory.
3. **Return `fs.SkipAll`** to stop the entire walk successfully — `WalkDir` returns `nil`,
   as the `SkipAll` output above shows. Any other non-nil error aborts the walk and is
   returned verbatim, which is the idiom for "stop, I found it".

Entries come in lexical order within each directory, which makes a walk deterministic —
unlike map iteration (7.6).

**`WalkDir` is preferred over `Walk`** because of what the callback receives. Swapping
`WalkDir` for `Walk` in the program above changes the second argument from
`fs.DirEntry` to `os.FileInfo`, and printing both for `tree/README.md` gives
`WalkDir: type=---------- isDir=false` (plus `d.Info()` on demand for `size=9`) versus
`Walk: size=9` already materialized.

`Walk` calls `lstat` on **every** entry to build the `FileInfo` it hands you. `WalkDir`
hands you a `fs.DirEntry`, whose name and type come from the directory read that already
happened — so a walk that only filters by name or extension does zero extra syscalls. On a
large tree that is a large difference, and `d.Info()` is there for the entries where you
actually need the size or mtime. `Walk` exists for compatibility; use `WalkDir`.

## Path traversal: the check that matters

`filepath.Join(root, userInput)` is a vulnerability. This is the most consequential thing
in the lesson for anyone writing a server.

```go
package main

import (
  "errors"
  "fmt"
  "io/fs"
  "path/filepath"
  "strings"
)

func unsafeResolve(root, userPath string) string {
  return filepath.Join(root, userPath)
}

func safeResolve(root, userPath string) (string, error) {
  if !filepath.IsLocal(userPath) {
    return "", fmt.Errorf("refusing non-local path %q", userPath)
  }
  return filepath.Join(root, userPath), nil
}

func safeResolveByPrefix(root, userPath string) (string, error) {
  full := filepath.Join(root, userPath)
  rel, err := filepath.Rel(root, full)
  if err != nil || strings.HasPrefix(rel, "..") {
    return "", fmt.Errorf("path %q escapes %q", userPath, root)
  }
  return full, nil
}

func main() {
  const root = "/srv/uploads"
  inputs := []string{
    "photo.jpg",
    "sub/photo.jpg",
    "../../etc/passwd",
    "..",
    "/etc/passwd",
    "sub/../photo.jpg",
    "sub/../../etc/passwd",
    "",
    "./photo.jpg",
  }

  fmt.Printf("%-24s %-28s %-10s %s\n", "user input", "Join (UNSAFE)", "IsLocal", "safeResolve")
  for _, in := range inputs {
    got, err := safeResolve(root, in)
    res := got
    if err != nil {
      res = "REJECTED"
    }
    fmt.Printf("%-24q %-28q %-10v %s\n", in, unsafeResolve(root, in), filepath.IsLocal(in), res)
  }

  fmt.Println("\n--- the Rel-based check does NOT agree ---")
  for _, in := range []string{"photo.jpg", "../../etc/passwd", "/etc/passwd", "sub/../../x"} {
    got, err := safeResolveByPrefix(root, in)
    fmt.Printf("  %-20q -> %q err=%v\n", in, got, err)
  }

  fmt.Println("\n--- Localize converts a slash path to an OS path, rejecting the unsafe ---")
  for _, in := range []string{"a/b", "../a", "/a", "a/../b", ""} {
    got, err := filepath.Localize(in)
    fmt.Printf("  Localize(%-10q) = %-8q err=%v\n", in, got, err)
  }

  fmt.Println("\n--- fs.ValidPath: the rule for fs.FS keys ---")
  for _, in := range []string{"a/b", ".", "", "/a", "a/", "./a", "../a", "a//b"} {
    fmt.Printf("  ValidPath(%-8q) = %v\n", in, fs.ValidPath(in))
  }

  fmt.Println("\n--- the error sentinels work with errors.Is ---")
  _, err := filepath.Match("[", "x")
  fmt.Println("  Match bad pattern is ErrBadPattern:", errors.Is(err, filepath.ErrBadPattern))
}
```

Output:

```
user input               Join (UNSAFE)                IsLocal    safeResolve
"photo.jpg"              "/srv/uploads/photo.jpg"     true       /srv/uploads/photo.jpg
"sub/photo.jpg"          "/srv/uploads/sub/photo.jpg" true       /srv/uploads/sub/photo.jpg
"../../etc/passwd"       "/etc/passwd"                false      REJECTED
".."                     "/srv"                       false      REJECTED
"/etc/passwd"            "/srv/uploads/etc/passwd"    false      REJECTED
"sub/../photo.jpg"       "/srv/uploads/photo.jpg"     true       /srv/uploads/photo.jpg
"sub/../../etc/passwd"   "/srv/etc/passwd"            false      REJECTED
""                       "/srv/uploads"               false      REJECTED
"./photo.jpg"            "/srv/uploads/photo.jpg"     true       /srv/uploads/photo.jpg

--- the Rel-based check does NOT agree ---
  "photo.jpg"          -> "/srv/uploads/photo.jpg" err=<nil>
  "../../etc/passwd"   -> "" err=path "../../etc/passwd" escapes "/srv/uploads"
  "/etc/passwd"        -> "/srv/uploads/etc/passwd" err=<nil>
  "sub/../../x"        -> "" err=path "sub/../../x" escapes "/srv/uploads"

--- Localize converts a slash path to an OS path, rejecting the unsafe ---
  Localize("a/b"     ) = "a/b"    err=<nil>
  Localize("../a"    ) = ""       err=invalid path
  Localize("/a"      ) = ""       err=invalid path
  Localize("a/../b"  ) = ""       err=invalid path
  Localize(""        ) = ""       err=invalid path

--- fs.ValidPath: the rule for fs.FS keys ---
  ValidPath("a/b"   ) = true
  ValidPath("."     ) = true
  ValidPath(""      ) = false
  ValidPath("/a"    ) = false
  ValidPath("a/"    ) = false
  ValidPath("./a"   ) = false
  ValidPath("../a"  ) = false
  ValidPath("a//b"  ) = false

--- the error sentinels work with errors.Is ---
  Match bad pattern is ErrBadPattern: true
```

Read the `Join (UNSAFE)` column. `"../../etc/passwd"` became **`/etc/passwd`**, outside
the root entirely, and `".."` became `/srv`. There is no error, no warning: `Join` Cleaned
the `..` away exactly as documented, and your handler now serves `/etc/passwd`. This is
CWE-22, and it is the single most common file-serving bug.

**`filepath.IsLocal` is the fix.** It returns true only for a path that stays within the
directory it is evaluated in: not absolute, not starting with `..`, not escaping via
interior `..`, and not empty. It rejects every dangerous row above while accepting
`"sub/../photo.jpg"`, which is legitimately inside the root. One call, before the `Join`.

**The `Rel`-prefix idiom you will find on the internet is wrong.** Look at the middle
block: `safeResolveByPrefix` **accepted** `"/etc/passwd"`, returning
`/srv/uploads/etc/passwd`. It is not a traversal in this case — `Join` treated the leading
slash as just another separator — but the check reported "safe" for an absolute path, and
the moment you use the user's string anywhere other than that exact `Join` (passing it to
`os.Open` directly, logging it, forwarding it) the absolute path is live. Check the
**input** with `IsLocal`, do not try to validate the output.

**`filepath.Localize` is the stricter cousin** for converting a slash-separated path (from
a URL, a zip entry, an `fs.FS` key) into an OS path. It rejects `..`, absolute paths, the
empty string, **and** `"a/../b"` which `IsLocal` allows — it wants a path that is already
clean. Use it at the boundary where slash-world becomes filesystem-world.

**`fs.ValidPath` is the rule for `fs.FS` keys**, and it is stricter still: unrooted,
slash-separated, Cleaned, no leading `./`, no trailing slash, no doubled slashes, with `.`
as the only way to say "the root". Every `fs.FS` implementation must reject anything else,
which is why the custom FS below checks it first.

For the strongest guarantee, `os.Root` (13.1) confines all operations beneath a directory
**at the syscall level**, which also defeats symlinks pointing out of the tree — something
no amount of string checking can do.

## io/fs

`io/fs` is the abstraction over "a tree of files". The core interface is one method:

```go
type FS interface {
  Open(name string) (File, error)
}
```

Everything else is an optional extension (`ReadDirFS`, `ReadFileFS`, `StatFS`, `GlobFS`,
`SubFS`) that the package-level helpers use when present and emulate when absent. That is
why `fs.ReadFile` works on **any** `FS`, even one that only implements `Open`.

The payoff is that the same code runs over the real disk, an embedded tree (13.13), a zip
archive (`archive/zip`), and an in-memory map — so file-reading logic becomes testable
without touching a disk.

| Func | Note |
|---|---|
| `fs.ReadFile(fsys, name)` | whole file as `[]byte` |
| `fs.ReadDir(fsys, name)` | `[]DirEntry`, **sorted by filename** |
| `fs.Stat(fsys, name)` | `FileInfo` |
| `fs.Glob(fsys, pattern)` | as `filepath.Glob`, over an `FS` |
| `fs.WalkDir(fsys, root, fn)` | as `filepath.WalkDir`, over an `FS` |
| `fs.Sub(fsys, dir)` | an `FS` rooted at `dir` |
| `fs.ValidPath(name)` | the key rule above |
| `os.DirFS(dir)` | the real filesystem as an `FS` |
| `fstest.MapFS` | an in-memory `FS` for tests |

```go
package main

import (
  "errors"
  "fmt"
  "io/fs"
  "os"
  "testing/fstest"
)

func countGo(fsys fs.FS) (int, error) {
  n := 0
  err := fs.WalkDir(fsys, ".", func(p string, d fs.DirEntry, err error) error {
    if err != nil {
      return err
    }
    if !d.IsDir() && len(p) > 3 && p[len(p)-3:] == ".go" {
      n++
    }
    return nil
  })
  return n, err
}

func main() {
  fmt.Println("--- os.DirFS: the real filesystem as an fs.FS ---")
  disk := os.DirFS("tree")
  b, err := fs.ReadFile(disk, "README.md")
  fmt.Printf("  fs.ReadFile(README.md) = %q err=%v\n", b, err)

  entries, _ := fs.ReadDir(disk, ".")
  fmt.Print("  fs.ReadDir(\".\"):")
  for _, e := range entries {
    fmt.Printf(" %s", e.Name())
  }
  fmt.Println()

  info, _ := fs.Stat(disk, "README.md")
  fmt.Printf("  fs.Stat: name=%s size=%d isDir=%v mode=%v\n",
    info.Name(), info.Size(), info.IsDir(), info.Mode())

  g, _ := fs.Glob(disk, "*/*/*.go")
  fmt.Println("  fs.Glob(\"*/*/*.go\"):", g)

  n, err := countGo(disk)
  fmt.Println("  .go files on disk:", n, "err:", err)

  fmt.Println("\n--- keys are ALWAYS slash-separated and relative, with no leading slash ---")
  _, err = fs.ReadFile(disk, "/README.md")
  fmt.Println("  ReadFile(\"/README.md\"):", err)
  _, err = fs.ReadFile(disk, "./README.md")
  fmt.Println("  ReadFile(\"./README.md\"):", err)
  _, err = fs.ReadFile(disk, "../f3/tree/README.md")
  fmt.Println("  ReadFile(\"../...\"):", err)

  fmt.Println("\n--- fs.Sub strips a prefix ---")
  sub, err := fs.Sub(disk, "docs")
  fmt.Println("  fs.Sub(disk, \"docs\") err:", err)
  se, _ := fs.ReadDir(sub, ".")
  fmt.Print("  entries:")
  for _, e := range se {
    fmt.Printf(" %s", e.Name())
  }
  fmt.Println()
  sb, _ := fs.ReadFile(sub, "guide.md")
  fmt.Printf("  ReadFile(\"guide.md\") through Sub = %q\n", sb)

  fmt.Println("\n--- fstest.MapFS: the same code, no disk at all ---")
  mem := fstest.MapFS{
    "README.md":            {Data: []byte("# in-memory\n")},
    "cmd/app/main.go":      {Data: []byte("package main\n")},
    "internal/store/db.go": {Data: []byte("package store\n")},
    "docs/guide.md":        {Data: []byte("notes\n")},
    "sub/deep/deeper/x.go": {Data: []byte("package deep\n")},
  }
  mb, _ := fs.ReadFile(mem, "README.md")
  fmt.Printf("  fs.ReadFile(README.md) = %q\n", mb)
  mn, _ := countGo(mem)
  fmt.Println("  the SAME countGo over MapFS:", mn)
  mg, _ := fs.Glob(mem, "*/*/*.go")
  fmt.Println("  fs.Glob(\"*/*/*.go\"):", mg)
  me, _ := fs.ReadDir(mem, "cmd/app")
  fmt.Println("  implied directories exist:", len(me), me[0].Name())

  fmt.Println("\n--- fstest.TestFS validates an implementation ---")
  fmt.Println("  TestFS(mem, \"README.md\"):", fstest.TestFS(mem, "README.md"))
  fmt.Println("  TestFS(mem, \"nope.md\") err:", fstest.TestFS(mem, "nope.md") != nil)

  fmt.Println("\n--- the error sentinels ---")
  _, err = fs.ReadFile(mem, "missing.txt")
  fmt.Println("  err:", err)
  fmt.Println("  errors.Is(err, fs.ErrNotExist):", errors.Is(err, fs.ErrNotExist))
  var pe *fs.PathError
  if errors.As(err, &pe) {
    fmt.Printf("  PathError: Op=%q Path=%q Err=%v\n", pe.Op, pe.Path, pe.Err)
  }
  _, err = fs.ReadFile(disk, "nope")
  fmt.Println("  os.DirFS missing file is also ErrNotExist:", errors.Is(err, fs.ErrNotExist))
}
```

Output (run from the directory containing `tree/`):

```
--- os.DirFS: the real filesystem as an fs.FS ---
  fs.ReadFile(README.md) = "# README\n" err=<nil>
  fs.ReadDir("."): .env .git README.md cmd docs go.mod internal node_modules
  fs.Stat: name=README.md size=9 isDir=false mode=-rw-r--r--
  fs.Glob("*/*/*.go"): [cmd/app/main.go cmd/app/main_test.go internal/store/db.go internal/store/db_test.go]
  .go files on disk: 4 err: <nil>

--- keys are ALWAYS slash-separated and relative, with no leading slash ---
  ReadFile("/README.md"): readfile /README.md: invalid argument
  ReadFile("./README.md"): readfile ./README.md: invalid argument
  ReadFile("../..."): readfile ../f3/tree/README.md: invalid argument

--- fs.Sub strips a prefix ---
  fs.Sub(disk, "docs") err: <nil>
  entries: api.md guide.md
  ReadFile("guide.md") through Sub = "notes\n"

--- fstest.MapFS: the same code, no disk at all ---
  fs.ReadFile(README.md) = "# in-memory\n"
  the SAME countGo over MapFS: 3
  fs.Glob("*/*/*.go"): [cmd/app/main.go internal/store/db.go]
  implied directories exist: 1 main.go

--- fstest.TestFS validates an implementation ---
  TestFS(mem, "README.md"): <nil>
  TestFS(mem, "nope.md") err: true

--- the error sentinels ---
  err: open missing.txt: file does not exist
  errors.Is(err, fs.ErrNotExist): true
  PathError: Op="open" Path="missing.txt" Err=file does not exist
  os.DirFS missing file is also ErrNotExist: true
```

**`countGo` is written once and runs over both.** That is the whole argument for `io/fs`:
take an `fs.FS` parameter instead of a directory string and your function becomes testable
with `fstest.MapFS` — no `t.TempDir()`, no cleanup, no disk, and the fixture visible in the
test source. This is the pattern 14.8 builds on, and the reason `embed.FS` (13.13) works
with everything here.

**`fs.FS` keys reject the forms you habitually type.** `"/README.md"`, `"./README.md"` and
any `..` all fail with `invalid argument` (`fs.ErrInvalid` in a `*fs.PathError`). The
narrow key space is what makes `fs.FS` traversal-safe by construction.

**`fstest.MapFS` synthesizes the intermediate directories.** The map has no `"cmd"` or
`"cmd/app"` key, yet `fs.ReadDir(mem, "cmd/app")` works and `WalkDir` descends normally.
So you only list the files.

**`fstest.TestFS` is a conformance checker** for an `fs.FS` you wrote: it exercises
`Open`, `Stat`, `ReadDir`, seeking and the error paths, and reports every deviation from
the contract.

**The sentinels are `fs.ErrNotExist`, `fs.ErrExist`, `fs.ErrPermission`, `fs.ErrInvalid`
and `fs.ErrClosed`.** Always match with `errors.Is` — the concrete value is a
`*fs.PathError` wrapping the sentinel, so `==` fails. Note that `os.ErrNotExist` **is**
`fs.ErrNotExist` (the `os` names are aliases), so one check covers both worlds.

### A custom FS

Implementing `fs.FS` is one method. This one uppercases every file it serves, which is a
silly transformation but exactly the shape of a real decorating FS (decrypting, rendering
templates, rewriting asset URLs):

```go
package main

import (
  "bytes"
  "fmt"
  "io"
  "io/fs"
  "strings"
  "time"
)

type upperFS map[string]string

type upperFile struct {
  name string
  *bytes.Reader
}

type upperInfo struct {
  name string
  size int64
}

func (i upperInfo) Name() string       { return i.name }
func (i upperInfo) Size() int64        { return i.size }
func (i upperInfo) Mode() fs.FileMode  { return 0o444 }
func (i upperInfo) ModTime() time.Time { return time.Time{} }
func (i upperInfo) IsDir() bool        { return false }
func (i upperInfo) Sys() any           { return nil }

func (f upperFile) Stat() (fs.FileInfo, error) {
  return upperInfo{name: f.name, size: f.Size()}, nil
}

func (f upperFile) Close() error { return nil }

func (u upperFS) Open(name string) (fs.File, error) {
  if !fs.ValidPath(name) {
    return nil, &fs.PathError{Op: "open", Path: name, Err: fs.ErrInvalid}
  }
  data, ok := u[name]
  if !ok {
    return nil, &fs.PathError{Op: "open", Path: name, Err: fs.ErrNotExist}
  }
  return upperFile{name: name, Reader: bytes.NewReader([]byte(strings.ToUpper(data)))}, nil
}

func main() {
  var u fs.FS = upperFS{"greet.txt": "hello there", "bye.txt": "goodbye"}

  b, err := fs.ReadFile(u, "greet.txt")
  fmt.Printf("fs.ReadFile: %q err=%v\n", b, err)

  f, _ := u.Open("bye.txt")
  st, _ := f.Stat()
  fmt.Printf("Stat: name=%s size=%d mode=%v\n", st.Name(), st.Size(), st.Mode())
  all, _ := io.ReadAll(f)
  fmt.Printf("io.ReadAll: %q\n", all)
  f.Close()

  _, err = fs.ReadFile(u, "missing.txt")
  fmt.Println("missing:", err)
  _, err = fs.ReadFile(u, "/absolute")
  fmt.Println("invalid path:", err)

  fmt.Println("\n--- fs.FileMode bits ---")
  for _, m := range []fs.FileMode{0o644, 0o755, fs.ModeDir | 0o755, fs.ModeSymlink | 0o777} {
    fmt.Printf("  %-12v IsDir=%-6v IsRegular=%-6v Perm=%04o\n",
      m, m.IsDir(), m.IsRegular(), m.Perm())
  }
}
```

Output:

```
fs.ReadFile: "HELLO THERE" err=<nil>
Stat: name=bye.txt size=7 mode=-r--r--r--
io.ReadAll: "GOODBYE"
missing: open missing.txt: file does not exist
invalid path: open /absolute: invalid argument

--- fs.FileMode bits ---
  -rw-r--r--   IsDir=false  IsRegular=true   Perm=0644
  -rwxr-xr-x   IsDir=false  IsRegular=true   Perm=0755
  drwxr-xr-x   IsDir=true   IsRegular=false  Perm=0755
  Lrwxrwxrwx   IsDir=false  IsRegular=false  Perm=0777
```

`fs.File` needs `Stat`, `Read` and `Close`; embedding a `*bytes.Reader` supplies `Read`
(plus `Seek` and `ReadAt`, which `http.FileServerFS` requires — 13.13). Validate with
`fs.ValidPath` and return a `*fs.PathError` wrapping the right sentinel so callers'
`errors.Is` works. `fs.ReadFile` worked here despite only `Open` being implemented,
because the helper falls back to `Open` + `io.ReadAll`.

`fs.FileMode` packs the type bits above the permission bits, which is why
`fs.ModeDir | 0o755` prints as `drwxr-xr-x`. Use `IsDir()`, `IsRegular()` and `Perm()`
rather than masking by hand.

## Quick Reference

| Need | Call |
|---|---|
| Build a filesystem path | `filepath.Join(parts...)` |
| File extension | `filepath.Ext(p)` — from the **last** dot |
| Path of A relative to B | `filepath.Rel(base, targ)` — errors across roots |
| **Validate user-supplied path** | `filepath.IsLocal(p)` **before** joining |
| Slash path → OS path, safely | `filepath.Localize(p)` |
| Recurse a tree | `filepath.WalkDir(root, fn)` |
| Prune a subtree mid-walk | `return fs.SkipDir` |
| Treat a directory as an `fs.FS` | `os.DirFS(dir)` |
| Read from any `fs.FS` | `fs.ReadFile`, `fs.ReadDir`, `fs.Stat`, `fs.Glob`, `fs.WalkDir` |
| An `fs.FS` for tests | `fstest.MapFS{"a/b.txt": {Data: []byte("…")}}` |
| Test for a missing file | `errors.Is(err, fs.ErrNotExist)` |

| Trap | Reality |
|---|---|
| `Join(root, userInput)` | **traversal**; `"../../etc/passwd"` escapes silently |
| A `Rel`-prefix safety check | accepts absolute input; use `IsLocal` on the input |
| `Dir("/")`, `Base("/")` | both `"/"` |
| `Ext("a.tar.gz")` | `".gz"` |
| `Match("[!abc]x", "ax")` | **true** — Go negates with `^`, not `!` |
| Dereferencing `d` before checking `err` | `d` can be nil — nil panic |
| `fs.FS` key `"./a"` or `"/a"` | `invalid argument`; keys are unrooted and Cleaned |
| `err == fs.ErrNotExist` | fails; it is wrapped — use `errors.Is` |

## Common Mistakes

**Joining a user-supplied path onto a root.** `filepath.Join` resolves `..`, so the result
can leave your directory; call `filepath.IsLocal` on the input first, or use `os.Root`.

**Using `filepath` for URL paths or `fs.FS` keys.** It is a no-op on Unix and corrupts
them on Windows; use `path` for anything slash-separated by definition.

**Expecting `Glob("**/*.go")` to recurse.** `**` is not supported; use `filepath.WalkDir`.

**Writing `[!abc]` in a `Match` pattern.** Go negates with `^`; `!` is a literal, so the
class matches the opposite of what you intended.

**Ignoring the `err` argument in a `WalkDirFunc`.** `d` may be nil, so touching it panics;
check `err` first and return `nil` to skip an unreadable entry.

**Taking a directory path where an `fs.FS` would do.** You lose `fstest.MapFS` testing
and the ability to serve from `embed.FS`.

## Exercises

1. Write `func SafeOpen(root string, userPath string) (*os.File, error)` that rejects
   traversal, and table-test it against `"a.txt"`, `"../a.txt"`, `"/etc/passwd"`,
   `"sub/../a.txt"`, `""` and `"sub/../../a.txt"`. Then rewrite it with `os.Root` and
   explain which attack only the second version stops.
2. Implement `func FindFiles(fsys fs.FS, exts, skipDirs []string) ([]string, error)` with
   `fs.WalkDir` and `fs.SkipDir`, tested entirely with `fstest.MapFS` — including an
   unreadable entry your error handling must skip rather than abort on.
3. Write `func GlobRecursive(root, pattern string) ([]string, error)` that supports `**`
   by walking the tree and matching with `filepath.Match` per element. Verify it agrees
   with `filepath.Glob` on non-`**` patterns.
4. Build a `fs.FS` that wraps another one and serves `.md` files rendered to HTML, and run
   `fstest.TestFS` against it. Report which contract rules you got wrong on the first try.
5. Write a table test proving the six `Dir`/`Base`/`Ext` edge cases above (`"/"`, `""`,
   `"a/"`, `".hidden"`, `"a.tar.gz"`, `"."`), then implement `func StripExt(p string)
   string` handling `.tar.gz` as a single extension.

---

[← 13.9 math, math/big, and math/rand](09-math-rand.md) · [Course Index](../README.md) · [13.11 flag: Command-Line Parsing →](11-flag.md)
