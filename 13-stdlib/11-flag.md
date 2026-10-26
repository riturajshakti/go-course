[← 13.10 path/filepath and io/fs](10-filepath-fs.md) · [Course Index](../README.md) · [13.12 log/slog: Structured Logging →](12-slog.md)

# 13.11 — flag: Command-Line Parsing

## What You'll Learn

- All eight typed constructors plus the `*Var` forms, and `flag.Args`/`NArg`/`NFlag`
- The two classic bugs: `-v false` does not set `v` to false, and **flags after positional args are ignored**
- `flag.Value` for custom types — a comma-separated list and a validated enum
- `flag.FlagSet` for subcommands: the idiomatic stdlib `tool add|list`
- The env-var fallback via `Visit`, and parsing from a slice so `main` becomes testable

## Coming From Another Language

| Other | Go |
|---|---|
| `argparse` / `commander` / `clap` | `flag` — deliberately minimal; no required flags, no short+long pairs |
| `--flag` only | `-flag` and `--flag` are **identical**; there is no short-vs-long distinction |
| `-abc` meaning `-a -b -c` | not supported; `-abc` is a flag named `abc` |
| `prog --verbose file.txt` works either way | **flags must come first**; after the first positional, parsing stops |
| `--no-color` conventions | write `-color=false`, or define a second flag |
| `process.argv` / `sys.argv` | `flag.Args()` after `flag.Parse()`; `os.Args` is the raw form |

`flag` is small on purpose, and the two things to know up front are that `-flag` and
`--flag` mean the same thing (so there is no `-v`/`--verbose` pairing — you define one
name), and that argument order matters in a way it does not in most other ecosystems.

## The basics

| Constructor | Returns | `*Var` form |
|---|---|---|
| `flag.String(name, def, usage)` | `*string` | `StringVar(&s, …)` |
| `flag.Int(...)` | `*int` | `IntVar(&i, …)` |
| `flag.Int64(...)` | `*int64` | `Int64Var(...)` |
| `flag.Uint(...)` | `*uint` | `UintVar(...)` |
| `flag.Uint64(...)` | `*uint64` | `Uint64Var(...)` |
| `flag.Bool(...)` | `*bool` | `BoolVar(...)` |
| `flag.Float64(...)` | `*float64` | `Float64Var(...)` |
| `flag.Duration(...)` | `*time.Duration` | `DurationVar(...)` |
| `flag.TextVar(&v, name, def, usage)` | — | for any `encoding.TextUnmarshaler` |
| `flag.Var(&v, name, usage)` | — | for any `flag.Value` |

After `flag.Parse()`: `flag.Args()` gives the leftover positional arguments,
`flag.NArg()` their count, `flag.Arg(i)` one of them, and `flag.NFlag()` how many flags
were actually set.

```go
package main

import (
  "flag"
  "fmt"
  "time"
)

func main() {
  var (
    name    = flag.String("name", "world", "who to greet")
    count   = flag.Int("count", 1, "how many times")
    verbose = flag.Bool("v", false, "verbose output")
    ratio   = flag.Float64("ratio", 0.5, "a ratio")
    timeout = flag.Duration("timeout", 5*time.Second, "request timeout")
    size    = flag.Int64("size", 1<<20, "max size in bytes")
    workers = flag.Uint("workers", 4, "worker count")
    limit   = flag.Uint64("limit", 1000, "a big limit")
  )
  var bound string
  flag.StringVar(&bound, "bound", "default", "bound to an existing variable")

  flag.Parse()

  fmt.Printf("name=%q count=%d v=%v ratio=%v timeout=%v size=%d workers=%d limit=%d bound=%q\n",
    *name, *count, *verbose, *ratio, *timeout, *size, *workers, *limit, bound)
  fmt.Println("Args():", flag.Args())
  fmt.Println("NArg():", flag.NArg(), " NFlag():", flag.NFlag())
  if flag.NArg() > 0 {
    fmt.Println("Arg(0):", flag.Arg(0))
  }
}
```

Built as `./tool`. Single and double dashes are interchangeable, and `-flag value` and
`-flag=value` both work:

```bash
./tool -name Ada --count 3 -ratio=0.25 --timeout 2s
```

Output:

```
name="Ada" count=3 v=false ratio=0.25 timeout=2s size=1048576 workers=4 limit=1000 bound="default"
Args(): []
NArg(): 0  NFlag(): 4
```

`flag.Duration` accepts any `time.ParseDuration` string (13.4), so `2s`, `1h30m`, `500ms`.

### The bool exception

```bash
./tool -v
./tool -v=false
./tool -v false
```

Output:

```
name="world" count=1 v=true ratio=0.5 timeout=5s size=1048576 workers=4 limit=1000 bound="default"
Args(): []
NArg(): 0  NFlag(): 1
name="world" count=1 v=false ratio=0.5 timeout=5s size=1048576 workers=4 limit=1000 bound="default"
Args(): []
NArg(): 0  NFlag(): 1
name="world" count=1 v=true ratio=0.5 timeout=5s size=1048576 workers=4 limit=1000 bound="default"
Args(): [false]
NArg(): 1  NFlag(): 1
```

**`-v false` sets `v` to true and leaves `"false"` as a positional argument.** A boolean
flag never consumes the next argument — otherwise `./tool -v file.txt` could not work —
so the only way to set one to false is the `=` form, `-v=false`. The failure is silent:
your program runs with verbose *on* and an unexpected extra argument, which is a confusing
bug report to receive. If false-setting matters, document `-v=false` in the usage string.

### Flags must come before positional arguments

```bash
./tool file.txt -name Ada
```

Output:

```
name="world" count=1 v=false ratio=0.5 timeout=5s size=1048576 workers=4 limit=1000 bound="default"
Args(): [file.txt -name Ada]
NArg(): 3  NFlag(): 0
Arg(0): file.txt
```

**`-name Ada` was silently ignored** — `NFlag()` is 0 and `-name` is sitting in `Args()`
as a literal string. `Parse` stops at the first argument that is not a flag, and
everything after it is positional, dashes included. This is the single most common `flag`
bug: the program runs with defaults, no error, and the user swears they passed the flag.

There is no way to change this (cobra and pflag do allow interspersed flags). If your
users will get it wrong, scan `flag.Args()` for a leading `-` after parsing and reject it
with a pointed message.

The `--` terminator ends flag parsing explicitly:

```bash
./tool -name Ada -- -not-a-flag extra
```

Output:

```
name="Ada" count=1 v=false ratio=0.5 timeout=5s size=1048576 workers=4 limit=1000 bound="default"
Args(): [-not-a-flag extra]
NArg(): 2  NFlag(): 1
Arg(0): -not-a-flag
```

### Errors and the default usage text

`flag.CommandLine` uses `ExitOnError`, so a bad flag prints a message **and the full
usage** to stderr and exits **2**. `-h`/`-help` prints usage and exits **0**.

```bash
./tool -h
```

Output (exit code 0; the indented lines are real tabs):

```
Usage of ./tool:
  -bound string
    	bound to an existing variable (default "default")
  -count int
    	how many times (default 1)
  -limit uint
    	a big limit (default 1000)
  -name string
    	who to greet (default "world")
  -ratio float
    	a ratio (default 0.5)
  -size int
    	max size in bytes (default 1048576)
  -timeout duration
    	request timeout (default 5s)
  -v	verbose output
  -workers uint
    	worker count (default 4)
```

Flags are listed **alphabetically**, not in declaration order. The type name comes from
the flag's kind, the `(default …)` suffix is omitted when the default is the zero value
(note `-v` has none, and that it fits on one line because its usage is short), and a
backquoted word in your usage string becomes the placeholder name — `"set the `port`"`
renders as `-addr port`.

The four error messages, each followed by the same usage block and `exit 2`:

```
flag provided but not defined: -nope
invalid value "abc" for flag -count: parse error
invalid value "5" for flag -timeout: parse error
flag needs an argument: -name
```

`parse error` is all you get for a malformed number or duration — the underlying
`strconv`/`time` error is swallowed. A custom `flag.Value` can do better, as below.

## Custom types with flag.Value

```go
type Value interface {
  String() string
  Set(string) error
}
```

`Set` is called once per occurrence of the flag, which is what makes a repeatable flag
possible: accumulate into the receiver instead of overwriting. `String` renders the
default in the usage text, and it is called on the **zero value** of your type during
`PrintDefaults`, so it must not panic on a nil or empty receiver.

```go
package main

import (
  "flag"
  "fmt"
  "os"
  "slices"
  "strings"
)

type stringList []string

func (s *stringList) String() string {
  if s == nil {
    return ""
  }
  return strings.Join(*s, ",")
}

func (s *stringList) Set(v string) error {
  for _, part := range strings.Split(v, ",") {
    part = strings.TrimSpace(part)
    if part == "" {
      return fmt.Errorf("empty element in %q", v)
    }
    *s = append(*s, part)
  }
  return nil
}

type logLevel string

var validLevels = []string{"debug", "info", "warn", "error"}

func (l *logLevel) String() string { return string(*l) }

func (l *logLevel) Set(v string) error {
  if !slices.Contains(validLevels, v) {
    return fmt.Errorf("must be one of %s", strings.Join(validLevels, ", "))
  }
  *l = logLevel(v)
  return nil
}

type repeated []string

func (r *repeated) String() string { return fmt.Sprint(*r) }

func (r *repeated) Set(v string) error {
  *r = append(*r, v)
  return nil
}

func main() {
  var tags stringList
  level := logLevel("info")
  var headers repeated

  fs := flag.NewFlagSet("tool", flag.ContinueOnError)
  fs.Var(&tags, "tags", "comma-separated tags (repeatable)")
  fs.Var(&level, "level", "log level: debug|info|warn|error")
  fs.Var(&headers, "H", "a header (repeat the flag)")

  if err := fs.Parse(os.Args[1:]); err != nil {
    fmt.Println("parse failed:", err)
    os.Exit(2)
  }
  fmt.Printf("tags=%v level=%q headers=%v args=%v\n", []string(tags), string(level), []string(headers), fs.Args())
}
```

```bash
./tool -tags a,b -tags c -level warn -H 'X-A: 1' -H 'X-B: 2' pos
```

Output:

```
tags=[a b c] level="warn" headers=[X-A: 1 X-B: 2] args=[pos]
```

Both styles work at once: `-tags a,b -tags c` accumulated to `[a b c]` because `Set` splits
*and* appends, and `-H` repeated twice gave two headers. `-H 'X-A: 1'` keeps its space
because the shell passed one argument.

An invalid enum value produces your own message:

```bash
./tool -level nope
```

Output (exit 2):

```
invalid value "nope" for flag -level: must be one of debug, info, warn, error
Usage of tool:
  -H value
    	a header (repeat the flag)
  -level value
    	log level: debug|info|warn|error (default info)
  -tags value
    	comma-separated tags (repeatable)
parse failed: invalid value "nope" for flag -level: must be one of debug, info, warn, error
```

That is a genuinely good CLI error, and it came from four lines in `Set`. Compare it to
the bare `parse error` the built-in `Int` gives.

Two things in that output. Custom types render as `-level value` in the usage — the word
`value` is the placeholder for anything registered with `Var`, and you override it by
backquoting a word in the usage string. And **the error appears twice**: `ContinueOnError`
prints the message and usage to the FlagSet's output *and* returns the error, so printing
it yourself duplicates it. Either call `fs.SetOutput(io.Discard)` or just exit without
printing, as the subcommand example below does.

A `-tags a,,b` gives `invalid value "a,,b" for flag -tags: empty element in "a,,b"`, and
`-h` returns the sentinel `flag.ErrHelp` after printing usage — so a `ContinueOnError`
FlagSet must treat `ErrHelp` as a successful exit, not a failure.

Idiomatic: `flag.Var` with a small named type beats parsing a string yourself after
`Parse`, because validation failures become ordinary flag errors with the right exit code.
For a type that already implements `encoding.TextUnmarshaler` (`netip.Addr`, `time.Time`,
your own), `flag.TextVar` saves you writing `Value` at all.

## Subcommands with FlagSet

`flag` has no subcommand support, but `flag.FlagSet` plus a `switch` is the idiomatic
stdlib answer and it is about twenty lines. Each subcommand owns a `FlagSet`, so `add` and
`list` can both define `-limit` with different meanings.

```go
package main

import (
  "errors"
  "flag"
  "fmt"
  "os"
  "time"
)

func usage() {
  fmt.Fprintf(os.Stderr, `tool - manage items

Usage:
  tool [global flags] <command> [command flags] [args]

Commands:
  add     add an item
  list    list items

Global flags:
`)
  flag.PrintDefaults()
  fmt.Fprintf(os.Stderr, "\nRun 'tool <command> -h' for command flags.\n")
}

func cmdAdd(global bool, args []string) error {
  fs := flag.NewFlagSet("add", flag.ContinueOnError)
  fs.SetOutput(os.Stderr)
  priority := fs.Int("priority", 1, "priority 1-5")
  due := fs.String("due", "", "due date (YYYY-MM-DD)")
  fs.Usage = func() {
    fmt.Fprintln(os.Stderr, "Usage: tool add [-priority n] [-due date] <title>...")
    fs.PrintDefaults()
  }
  if err := fs.Parse(args); err != nil {
    return err
  }
  if fs.NArg() == 0 {
    fs.Usage()
    return errors.New("add: a title is required")
  }
  fmt.Printf("ADD title=%q priority=%d due=%q verbose=%v\n", fs.Arg(0), *priority, *due, global)
  return nil
}

func cmdList(global bool, args []string) error {
  fs := flag.NewFlagSet("list", flag.ContinueOnError)
  fs.SetOutput(os.Stderr)
  all := fs.Bool("all", false, "include completed")
  limit := fs.Int("limit", 10, "max rows")
  if err := fs.Parse(args); err != nil {
    return err
  }
  fmt.Printf("LIST all=%v limit=%d extra=%v verbose=%v\n", *all, *limit, fs.Args(), global)
  return nil
}

func main() {
  verbose := flag.Bool("v", false, "verbose output")
  timeout := flag.Duration("timeout", 10*time.Second, "overall timeout")
  flag.Usage = usage
  flag.Parse()

  if flag.NArg() == 0 {
    usage()
    os.Exit(2)
  }

  cmd, rest := flag.Arg(0), flag.Args()[1:]
  var err error
  switch cmd {
  case "add":
    err = cmdAdd(*verbose, rest)
  case "list":
    err = cmdList(*verbose, rest)
  default:
    fmt.Fprintf(os.Stderr, "unknown command %q\n\n", cmd)
    usage()
    os.Exit(2)
  }
  if err != nil {
    if errors.Is(err, flag.ErrHelp) {
      os.Exit(0)
    }
    fmt.Fprintln(os.Stderr, "error:", err)
    os.Exit(1)
  }
  _ = timeout
}
```

```bash
./tool -v add -priority 3 "write lesson"
./tool list -all -limit 2
```

Output:

```
ADD title="write lesson" priority=3 due="" verbose=true
LIST all=true limit=2 extra=[] verbose=false
```

Each subcommand rejects the other's flags:

```bash
./tool add -all x
```

Output (exit 1):

```
flag provided but not defined: -all
Usage: tool add [-priority n] [-due date] <title>...
  -due string
    	due date (YYYY-MM-DD)
  -priority int
    	priority 1-5 (default 1)
error: flag provided but not defined: -all
```

With no command, the custom `flag.Usage`:

```
tool - manage items

Usage:
  tool [global flags] <command> [command flags] [args]

Commands:
  add     add an item
  list    list items

Global flags:
  -timeout duration
    	overall timeout (default 10s)
  -v	verbose output

Run 'tool <command> -h' for command flags.
```

And `./tool add -h` exits **0** with the subcommand's own usage:

```
Usage: tool add [-priority n] [-due date] <title>...
  -due string
    	due date (YYYY-MM-DD)
  -priority int
    	priority 1-5 (default 1)
```

The mechanics worth copying. **Global flags go before the command** (`tool -v add …`),
because the global `Parse` stops at the first positional — which is the command name, and
`flag.Args()[1:]` is exactly the subcommand's argument slice. **Each subcommand parses its
own slice**, so flag names never collide. **`flag.Usage` is a `var` you assign**, and the
default implementation prints `Usage of <name>:` then `PrintDefaults()`. **Return
`flag.ErrHelp` upward and exit 0** — `-h` is a successful outcome, not a failure. And the
whole thing has no dependencies.

| ErrorHandling | On a parse error |
|---|---|
| `flag.ContinueOnError` | prints the error + usage, **returns** the error |
| `flag.ExitOnError` | prints, then `os.Exit(2)` — the default for `flag.CommandLine` |
| `flag.PanicOnError` | prints, then panics — useful in tests that assert a panic |

Use `ContinueOnError` for every `FlagSet` you create, so your `main` can decide the exit
code and your tests can call the parser without killing the test binary.

## Env-var fallback, precedence, and testable parsing

Two patterns in one program. `Visit` reports only the flags that were **actually set**,
which is how you layer env vars underneath flags without clobbering an explicit value.
And taking `args []string` instead of reading `os.Args` makes the whole parser a pure
function you can table-test.

```go
package main

import (
  "bytes"
  "flag"
  "fmt"
  "os"
  "time"
)

type config struct {
  Addr    string
  Timeout time.Duration
  Debug   bool
}

func parseConfig(name string, args []string, env func(string) (string, bool), out *bytes.Buffer) (config, error) {
  var c config
  fs := flag.NewFlagSet(name, flag.ContinueOnError)
  fs.SetOutput(out)
  fs.StringVar(&c.Addr, "addr", ":8080", "listen address")
  fs.DurationVar(&c.Timeout, "timeout", 5*time.Second, "request timeout")
  fs.BoolVar(&c.Debug, "debug", false, "enable debug logging")

  if err := fs.Parse(args); err != nil {
    return c, err
  }

  set := map[string]bool{}
  fs.Visit(func(f *flag.Flag) { set[f.Name] = true })

  for _, m := range []struct{ flagName, envName string }{
    {"addr", "APP_ADDR"},
    {"timeout", "APP_TIMEOUT"},
    {"debug", "APP_DEBUG"},
  } {
    if set[m.flagName] {
      continue
    }
    if v, ok := env(m.envName); ok {
      if err := fs.Set(m.flagName, v); err != nil {
        return c, fmt.Errorf("%s: %w", m.envName, err)
      }
    }
  }
  return c, nil
}

func main() {
  envs := map[string]string{"APP_ADDR": ":9999", "APP_TIMEOUT": "30s"}
  lookup := func(k string) (string, bool) { v, ok := envs[k]; return v, ok }
  none := func(string) (string, bool) { return "", false }
  var out bytes.Buffer

  fmt.Println("--- precedence: flag > env > default ---")
  for _, tc := range []struct {
    label string
    args  []string
    env   func(string) (string, bool)
  }{
    {"nothing set", nil, none},
    {"env only", nil, lookup},
    {"flag only", []string{"-addr", ":3000"}, none},
    {"flag overrides env", []string{"-addr", ":3000"}, lookup},
    {"flag sets the DEFAULT value explicitly", []string{"-addr", ":8080"}, lookup},
  } {
    c, err := parseConfig("app", tc.args, tc.env, &out)
    fmt.Printf("  %-38s %+v err=%v\n", tc.label, c, err)
  }

  fmt.Println("\n--- Visit vs VisitAll ---")
  fs := flag.NewFlagSet("demo", flag.ContinueOnError)
  fs.SetOutput(&out)
  fs.String("a", "da", "flag a")
  fs.String("b", "db", "flag b")
  fs.Parse([]string{"-a", "set"})
  fmt.Print("  Visit (only those SET):   ")
  fs.Visit(func(f *flag.Flag) { fmt.Printf("%s=%q ", f.Name, f.Value) })
  fmt.Print("\n  VisitAll (every flag):    ")
  fs.VisitAll(func(f *flag.Flag) { fmt.Printf("%s=%q(def=%q) ", f.Name, f.Value, f.DefValue) })
  fmt.Println()
  fmt.Println("  Lookup(\"a\").Usage:", fs.Lookup("a").Usage)
  fmt.Println("  Lookup(\"zz\") == nil:", fs.Lookup("zz") == nil)

  fmt.Println("\n--- bad env value surfaces as an error, not a silent default ---")
  badEnv := func(k string) (string, bool) {
    if k == "APP_TIMEOUT" {
      return "not-a-duration", true
    }
    return "", false
  }
  c, err := parseConfig("app", nil, badEnv, &out)
  fmt.Printf("  %+v\n  err=%v\n", c, err)

  fmt.Println("\n--- ContinueOnError returns the error instead of exiting ---")
  out.Reset()
  _, err = parseConfig("app", []string{"-nope"}, none, &out)
  fmt.Println("  err:", err)
  fmt.Printf("  first line written to our buffer: %q\n", firstLine(out.String()))

  fmt.Println("\n--- flag.CommandLine is just the default FlagSet ---")
  fmt.Println("  Name() == os.Args[0]:", flag.CommandLine.Name() == os.Args[0])
  fmt.Println("  ErrorHandling() == ExitOnError:", flag.CommandLine.ErrorHandling() == flag.ExitOnError)
}

func firstLine(s string) string {
  if i := bytes.IndexByte([]byte(s), '\n'); i >= 0 {
    return s[:i]
  }
  return s
}
```

Output:

```
--- precedence: flag > env > default ---
  nothing set                            {Addr::8080 Timeout:5s Debug:false} err=<nil>
  env only                               {Addr::9999 Timeout:30s Debug:false} err=<nil>
  flag only                              {Addr::3000 Timeout:5s Debug:false} err=<nil>
  flag overrides env                     {Addr::3000 Timeout:30s Debug:false} err=<nil>
  flag sets the DEFAULT value explicitly {Addr::8080 Timeout:30s Debug:false} err=<nil>

--- Visit vs VisitAll ---
  Visit (only those SET):   a="set" 
  VisitAll (every flag):    a="set"(def="da") b="db"(def="db") 
  Lookup("a").Usage: flag a
  Lookup("zz") == nil: true

--- bad env value surfaces as an error, not a silent default ---
  {Addr::8080 Timeout:0s Debug:false}
  err=APP_TIMEOUT: parse error

--- ContinueOnError returns the error instead of exiting ---
  err: flag provided but not defined: -nope
  first line written to our buffer: "flag provided but not defined: -nope"

--- flag.CommandLine is just the default FlagSet ---
  Name() == os.Args[0]: true
  ErrorHandling() == ExitOnError: true
```

**`Visit` is the mechanism that makes precedence work.** It walks only the flags present
on the command line, so "was this set explicitly?" is answerable — and the layering is
then just `fs.Set(name, envValue)` for the ones that were not. `VisitAll` walks every
registered flag and is what you want for dumping the effective configuration at startup.

**Look at the fifth precedence row.** Passing `-addr :8080` — the same string as the
default — still beats the env var, because `Visit` reports it as set. That is the correct
behaviour (the user said it explicitly) and it is a behaviour you cannot implement by
comparing against the default value, which is why `Visit` exists.

**A bad env value becomes a real error** rather than silently leaving the default, because
`fs.Set` runs the same parser the flag would. Note `Timeout` is `0s` in that row, not
`5s`: a failed `Set` has already zeroed the target, so the returned config is garbage —
return on the error, never use the partial value.

**`parseConfig` takes `args` and an `env` function and writes to a buffer.** All three
make it a pure function, so the table above *is* the test. The rule for CLIs: `main` reads
`os.Args` and `os.LookupEnv` and immediately delegates; everything else takes parameters.

This precedence order — **flags > env > config file > defaults** — is what
[18.2](../18-production/02-config.md) builds out with a file layer and struct tags. The
`Visit` mechanism here is the piece that makes it work.

## What flag does not do

No required flags (check `Visit` or the zero value after `Parse`), no short+long aliases
(define two flags writing to the same variable with `*Var`), no interspersed flags after
positionals, no subcommand help tree, no `--no-x` negation, no flag groups or mutual
exclusion, no shell completion.

If you need those, the ecosystem answers are `spf13/cobra` (the big one: subcommand trees,
completion, used by kubectl and docker), `urfave/cli`, `alecthomas/kingpin`, and
`spf13/pflag` (GNU-style flags, drop-in-ish for `flag`). Reach for cobra when you have a
real subcommand tree; `flag` plus a `switch` genuinely covers most tools, and
[18.10](../18-production/10-project-layout.md) covers the layout.

## Quick Reference

| Need | Call |
|---|---|
| A typed flag | `flag.String/Int/Bool/Float64/Duration/Int64/Uint/Uint64(name, def, usage)` |
| Bind to an existing variable | `flag.StringVar(&s, name, def, usage)` |
| A custom type | implement `flag.Value`, register with `flag.Var(&v, name, usage)` |
| A `TextUnmarshaler` type | `flag.TextVar(&v, name, def, usage)` |
| Parse | `flag.Parse()` — or `fs.Parse(args)` for a `FlagSet` |
| Positional arguments | `flag.Args()`, `flag.NArg()`, `flag.Arg(i)` |
| How many flags were set | `flag.NFlag()` |
| Which flags were set | `flag.Visit(fn)` |
| Every registered flag | `flag.VisitAll(fn)` |
| Set a flag programmatically | `fs.Set(name, value)` |
| One flag's definition | `fs.Lookup(name)` — nil if undefined |
| Custom usage | assign `flag.Usage` / `fs.Usage`; call `PrintDefaults()` inside |
| Subcommands | one `flag.NewFlagSet(name, flag.ContinueOnError)` per command |
| Don't exit on error | `flag.NewFlagSet(name, flag.ContinueOnError)` |
| Capture usage output | `fs.SetOutput(&buf)` |

| Trap | Reality |
|---|---|
| `-v false` | sets `v` **true**, leaves `"false"` positional; use `-v=false` |
| `prog file -flag x` | `-flag` is **ignored** and lands in `Args()` |
| `-abc` | one flag named `abc`, not `-a -b -c` |
| Expecting declaration order in `-h` | flags print **alphabetically** |
| A bad int's error message | just `parse error`; write a `flag.Value` for better |
| `ContinueOnError` + printing the error | prints twice; discard the output or don't print |
| `-h` with `ContinueOnError` | returns `flag.ErrHelp`; treat it as exit 0 |
| Using the config after a failed `fs.Set` | the target is already zeroed |
| Comparing a flag to its default to detect "set" | wrong; use `Visit` |
| Reading `os.Args` deep in your code | untestable; pass `args []string` down |

## Common Mistakes

**Documenting `-v false` to disable a boolean.** It does the opposite; only `-v=false`
works.

**Putting flags after positional arguments.** Parsing stops at the first positional, so
the flags are silently ignored — validate `flag.Args()` for a leading dash.

**Using `ExitOnError` on a `FlagSet` you built.** Your `main` loses control of the exit
code and tests cannot call the parser; use `ContinueOnError`.

**Forgetting `flag.Parse()`.** Every flag keeps its default and `flag.Args()` is empty,
with no warning.

**Reading flag values before `Parse` returns.** The pointers hold defaults until then, so
a package-level `var x = *flag.String(...)` captures the default forever.

**Detecting "the user set this" by comparing to the default.** A user who passes the
default value explicitly is indistinguishable; use `flag.Visit`.

## Exercises

1. Extend the `tool add|list` CLI with a `delete` subcommand taking `-force` and a
   required ID, plus a `tool help <command>` form that prints that subcommand's usage.
   Make every error path exit with the right code.
2. Write `func Required(fs *flag.FlagSet, names ...string) error` that returns an error
   naming every required flag the user omitted, using `Visit`. Table-test it.
3. Implement a `flag.Value` for `map[string]string` accepting `-label k=v` repeatedly and
   rejecting a missing `=` with a useful message. Capture the real error text.
4. Make `parseConfig` testable end to end: write table tests covering flag-only,
   env-only, both, a malformed env value, an unknown flag, and `-h`, asserting on the
   returned error and on the usage text written to the buffer.
5. Add a `-config file.json` layer to `parseConfig` so precedence becomes
   flags > env > file > defaults, and prove the full ordering with one table test per
   layer combination.

---

[← 13.10 path/filepath and io/fs](10-filepath-fs.md) · [Course Index](../README.md) · [13.12 log/slog: Structured Logging →](12-slog.md)
