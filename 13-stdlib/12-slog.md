[← 13.11 flag: Command-Line Parsing](11-flag.md) · [Course Index](../README.md) · [13.13 embed: Embedding Files in the Binary →](13-embed.md)

# 13.12 — log/slog: Structured Logging

## What You'll Learn

- The three handlers (`TextHandler`, `JSONHandler`, and the bridge to the old `log` package) on one log line
- Levels as **integers**, custom levels, and `slog.LevelVar` for flipping the level at runtime
- Attributes: the key-value form, the typed form, `slog.Group`, and the real `!BADKEY` output
- `slog.LogValuer` for lazy and redacted values — plus the `fmt.Errorf("%v")` hole it does **not** close
- `ReplaceAttr` with real before/after output, a minimal custom `Handler`, and a benchmark that contradicts the usual advice

## Coming From Another Language

| Other | Go |
|---|---|
| `winston` / `pino` / `zap` | `slog` in the stdlib; `zap`/`zerolog` remain faster but `slog` is the default |
| `logger.info(f"user {id} logged in")` | `logger.Info("user logged in", "user_id", id)` — message constant, data as fields |
| `logging.getLogger(__name__)` hierarchy | no hierarchy; `logger.With(...)` derives a child |
| `logger.child({svc: "api"})` | `logger.With("svc", "api")` |
| `console.log` taking anything | an **even** number of alternating key/value args, or typed `Attr`s |
| Python's `%(asctime)s` format strings | a `Handler` decides the format; you never format a line |

`slog` arrived in Go 1.21 and is now the default answer. The design is three layers: a
`Logger` (what you call), a `Record` (level, message, attrs, time, caller PC), and a
`Handler` (what turns a Record into bytes). You swap the Handler to change format or
destination; you never touch formatting in your call sites.

[18.4](../18-production/04-service-logging.md) covers the decisions you make **once per
service** — which handler, the request-scoped logger in the context, what to log at which
level. This lesson is the API underneath it.

**On reproducibility:** every log line carries a timestamp, which would make this lesson's
output vary on every run. So each example installs a `ReplaceAttr` that **drops** the
`time` attribute (or freezes it to a fixed instant where the point is the field name).
That is why no output below has a `time` field unless the example is specifically about
one. In production you keep the timestamp.

## The three handlers, one log line

```go
package main

import (
  "fmt"
  "log"
  "log/slog"
  "os"
)

func stripTime(groups []string, a slog.Attr) slog.Attr {
  if a.Key == slog.TimeKey && len(groups) == 0 {
    return slog.Attr{}
  }
  return a
}

func main() {
  opts := &slog.HandlerOptions{ReplaceAttr: stripTime}

  fmt.Println("=== TextHandler ===")
  slog.New(slog.NewTextHandler(os.Stdout, opts)).
    Info("order created", "order_id", "ord_7", "total_cents", 4999, "paid", true)

  fmt.Println("\n=== JSONHandler ===")
  slog.New(slog.NewJSONHandler(os.Stdout, opts)).
    Info("order created", "order_id", "ord_7", "total_cents", 4999, "paid", true)

  fmt.Println("\n=== the default logger, before SetDefault (text, to stderr) ===")
  fmt.Println("(goes to stderr, with a real timestamp - not reproducible)")

  fmt.Println("\n=== and the log package bridged through slog ===")
  slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, opts)))
  slog.Info("via slog.Info", "k", "v")
  log.Println("via the log package")
  log.Printf("log.Printf with %d args", 2)

  fmt.Println("\n=== slog.NewLogLogger: a *log.Logger writing INTO a handler ===")
  lg := slog.NewLogLogger(slog.NewJSONHandler(os.Stdout, opts), slog.LevelWarn)
  lg.Println("third-party library output")
}
```

Output (stderr discarded):

```
=== TextHandler ===
level=INFO msg="order created" order_id=ord_7 total_cents=4999 paid=true

=== JSONHandler ===
{"level":"INFO","msg":"order created","order_id":"ord_7","total_cents":4999,"paid":true}

=== the default logger, before SetDefault (text, to stderr) ===
(goes to stderr, with a real timestamp - not reproducible)

=== and the log package bridged through slog ===
{"level":"INFO","msg":"via slog.Info","k":"v"}
{"level":"INFO","msg":"via the log package"}
{"level":"INFO","msg":"log.Printf with 2 args"}

=== slog.NewLogLogger: a *log.Logger writing INTO a handler ===
{"level":"WARN","msg":"third-party library output"}
```

Same call, three formats. `TextHandler` is logfmt-style and quotes only what needs it
(`msg="order created"` but `order_id=ord_7`); `JSONHandler` emits one object per line.
Text for a terminal, JSON for anything that ingests logs.

**The two bridges run in opposite directions and both matter.** `slog.SetDefault` makes
the package-level `slog.Info` work *and* redirects the standard `log` package into your
handler — notice `log.Println` came out as structured JSON with `"level":"INFO"`. That one
call captures output from `net/http`'s internal error logger and from every dependency
that still uses `log`. Going the other way, `slog.NewLogLogger(handler, level)` hands you
a `*log.Logger` to give to a library that demands one (such as `http.Server.ErrorLog`),
with its output arriving as `slog` records at the level you choose.

Without `SetDefault`, the default handler writes `TextHandler`-ish output to **stderr**
with a real timestamp, which is why that section of the program just describes itself.

## Levels

Levels are plain `int`s, spaced by 4 so you can define your own in between.

| Constant | Value |
|---|---|
| `slog.LevelDebug` | **-4** |
| `slog.LevelInfo` | **0** |
| `slog.LevelWarn` | **4** |
| `slog.LevelError` | **8** |

```go
package main

import (
  "context"
  "fmt"
  "log/slog"
  "os"
)

func stripTime(groups []string, a slog.Attr) slog.Attr {
  if a.Key == slog.TimeKey && len(groups) == 0 {
    return slog.Attr{}
  }
  return a
}

const LevelTrace = slog.Level(-8)
const LevelFatal = slog.Level(12)

func main() {
  fmt.Println("=== the four levels and their numeric values ===")
  for _, l := range []slog.Level{slog.LevelDebug, slog.LevelInfo, slog.LevelWarn, slog.LevelError} {
    fmt.Printf("  %-5s = %d\n", l, int(l))
  }
  fmt.Printf("  custom LevelTrace = %d prints as %v\n", int(LevelTrace), LevelTrace)
  fmt.Printf("  custom LevelFatal = %d prints as %v\n", int(LevelFatal), LevelFatal)

  fmt.Println("\n=== Level filtering: default is Info, so Debug is dropped ===")
  def := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{ReplaceAttr: stripTime}))
  def.Debug("dropped")
  def.Info("kept")

  dbg := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    Level:       slog.LevelDebug,
    ReplaceAttr: stripTime,
  }))
  dbg.Debug("now kept")
  fmt.Println("  Enabled(Debug) on the default handler:",
    def.Handler().Enabled(context.Background(), slog.LevelDebug))
  fmt.Println("  Enabled(Debug) on the debug handler:  ",
    dbg.Handler().Enabled(context.Background(), slog.LevelDebug))

  fmt.Println("\n=== LevelVar: change the level at runtime ===")
  lv := new(slog.LevelVar)
  lv.Set(slog.LevelWarn)
  rt := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: lv, ReplaceAttr: stripTime}))
  rt.Info("invisible at WARN")
  rt.Warn("visible at WARN")
  lv.Set(slog.LevelDebug)
  rt.Debug("visible now, with no restart")
  fmt.Println("  lv.Level():", lv.Level(), " lv.String():", lv.String())

  fmt.Println("\n=== Log and LogAttrs take an explicit level ===")
  ctx := context.Background()
  rt.Log(ctx, LevelTrace, "a custom level")
  rt.Log(ctx, LevelFatal, "another custom level")
  rt.LogAttrs(ctx, slog.LevelInfo, "LogAttrs: no variadic any", slog.String("k", "v"), slog.Int("n", 1))

  fmt.Println("\n=== parsing a level from config ===")
  var l slog.Level
  for _, in := range []string{"debug", "INFO", "warn", "ERROR", "DEBUG+2", "nope"} {
    err := l.UnmarshalText([]byte(in))
    fmt.Printf("  %-8q -> %-8v err=%v\n", in, l, err)
  }

  fmt.Println("\n=== AddSource ===")
  src := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
    AddSource: true,
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      a = stripTime(g, a)
      if a.Key == slog.SourceKey {
        if s, ok := a.Value.Any().(*slog.Source); ok {
          return slog.String("source", fmt.Sprintf("%s:%d", "main.go", s.Line))
        }
      }
      return a
    },
  }))
  src.Info("with source")
}
```

Output:

```
=== the four levels and their numeric values ===
  DEBUG = -4
  INFO  = 0
  WARN  = 4
  ERROR = 8
  custom LevelTrace = -8 prints as DEBUG-4
  custom LevelFatal = 12 prints as ERROR+4

=== Level filtering: default is Info, so Debug is dropped ===
level=INFO msg=kept
level=DEBUG msg="now kept"
  Enabled(Debug) on the default handler: false
  Enabled(Debug) on the debug handler:   true

=== LevelVar: change the level at runtime ===
level=WARN msg="visible at WARN"
level=DEBUG msg="visible now, with no restart"
  lv.Level(): DEBUG  lv.String(): LevelVar(DEBUG)

=== Log and LogAttrs take an explicit level ===
level=ERROR+4 msg="another custom level"
level=INFO msg="LogAttrs: no variadic any" k=v n=1

=== parsing a level from config ===
  "debug"  -> DEBUG    err=<nil>
  "INFO"   -> INFO     err=<nil>
  "warn"   -> WARN     err=<nil>
  "ERROR"  -> ERROR    err=<nil>
  "DEBUG+2" -> DEBUG+2  err=<nil>
  "nope"   -> DEBUG+2  err=slog: level string "nope": unknown name

=== AddSource ===
level=INFO source=main.go:79 msg="with source"
```

Every level method has a `…Context` twin (`InfoContext(ctx, …)` and friends) producing
identical output while passing the `ctx` through to the Handler — which is what a Handler
needs to pull a trace ID out of it.

**A custom level prints relative to the nearest named one.** `slog.Level(-8)` renders as
`DEBUG-4` and `slog.Level(12)` as `ERROR+4`. If you want the word `TRACE` in your output,
rewrite the level attr in `ReplaceAttr`.

**`LevelTrace` did not appear in the output at all.** The `LevelVar` was at `Debug` (-4)
and `LevelTrace` is -8, so it was filtered — a custom level below `Debug` needs the
threshold lowered to match, which is the point of levels being integers.

**`*slog.LevelVar` is the mutable, atomic level holder.** Pass it as `Level` and
`lv.Set(...)` takes effect on the next call from every goroutine, with no restart. That is
what an admin endpoint toggles during an incident (18.4). Note `lv.String()` is
`LevelVar(DEBUG)`, not `DEBUG` — use `lv.Level()` when you want the level itself.

**`UnmarshalText` parses a level from config,** case-insensitively, and accepts the offset
form `DEBUG+2`. On failure it errors and **leaves the receiver unchanged** (note it still
holds `DEBUG+2` from the previous row), so check the error.

**`Handler.Enabled(ctx, level)` is the cheap pre-check** for genuinely expensive argument
construction, though `LogValuer` below is usually the better answer.

**`AddSource` attaches `source:{file,line,function}`,** at the cost of a `runtime.Callers`
per call. The raw value is a `*slog.Source`, which is why the example reformats it — the
real output would otherwise contain an absolute path that varies by machine.

## Attributes

Two ways to pass data, plus `slog.Group` to nest it.

| Constructor | For |
|---|---|
| `slog.String(k, v)` | `string` |
| `slog.Int(k, v)` / `slog.Int64` / `slog.Uint64` | integers |
| `slog.Float64(k, v)` | floats |
| `slog.Bool(k, v)` | bools |
| `slog.Duration(k, v)` | `time.Duration` |
| `slog.Time(k, v)` | `time.Time` |
| `slog.Any(k, v)` | anything else, including `error` |
| `slog.Group(k, attrsOrKV...)` | a nested object |

```go
package main

import (
  "errors"
  "fmt"
  "log/slog"
  "os"
  "time"
)

func stripTime(groups []string, a slog.Attr) slog.Attr {
  if a.Key == slog.TimeKey && len(groups) == 0 {
    return slog.Attr{}
  }
  return a
}

func main() {
  log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level:       slog.LevelDebug,
    ReplaceAttr: stripTime,
  }))

  fmt.Println("=== the alternating key-value form ===")
  log.Info("kv form", "user_id", 42, "plan", "pro", "active", true)

  fmt.Println("\n=== the typed Attr form: same output, no boxing ===")
  log.Info("attr form",
    slog.Int("user_id", 42),
    slog.String("plan", "pro"),
    slog.Bool("active", true))

  fmt.Println("\n=== every typed constructor ===")
  log.Info("types",
    slog.String("s", "text"),
    slog.Int("i", -1),
    slog.Int64("i64", 1<<40),
    slog.Uint64("u64", 18446744073709551615),
    slog.Float64("f", 1.5),
    slog.Bool("b", true),
    slog.Duration("d", 1500*time.Millisecond),
    slog.Time("t", time.Date(2026, 3, 7, 15, 4, 5, 0, time.UTC)),
    slog.Any("slice", []int{1, 2, 3}),
    slog.Any("map", map[string]int{"a": 1}),
    slog.Any("nil", nil),
    slog.Any("err", errors.New("boom")),
  )

  fmt.Println("\n=== THE most common slog bug: an odd number of arguments ===")
  log.Info("odd count", "user_id", 42, "orphan")
  log.Info("wrong order", 42, "user_id")
  log.Info("non-string key", 1, "one")

  fmt.Println("\n=== mixing the two forms works ===")
  log.Info("mixed", slog.Int("typed", 1), "loose", 2)

  fmt.Println("\n=== slog.Group nests ===")
  log.Info("request",
    slog.Group("http",
      slog.String("method", "POST"),
      slog.Int("status", 201),
      slog.Group("client", slog.String("ip", "203.0.113.7")),
    ),
    slog.Int("db_queries", 3),
  )
  log.Info("group, kv style", slog.Group("http", "method", "GET", "status", 200))
  log.Info("empty group is omitted", slog.Group("empty"), slog.String("kept", "yes"))

  fmt.Println("\n=== With derives a child logger ===")
  base := log.With("service", "orders", "version", "1.4.2")
  base.Info("from the child")
  deeper := base.With("component", "repo")
  deeper.Info("two levels of With")
  log.Info("the parent is unchanged")

  fmt.Println("\n=== WithGroup namespaces everything after it ===")
  dbLog := log.WithGroup("db").With("pool", "primary")
  dbLog.Info("slow query", "duration_ms", 1840, "table", "orders")
  log.WithGroup("a").WithGroup("b").Info("nested groups", "k", "v")

  fmt.Println("\n=== an empty Attr is dropped ===")
  log.Info("with an empty attr", slog.Attr{}, slog.String("k", "v"))
}
```

`go vet` flags the three deliberately-broken calls before the program even runs, which is
itself the headline:

```
main.go:50:3: call to slog.Logger.Info missing a final value
main.go:51:27: slog.Logger.Info arg "42" should be a string or a slog.Attr (possible missing key or value)
main.go:52:30: slog.Logger.Info arg "1" should be a string or a slog.Attr (possible missing key or value)
```

Output:

```
=== the alternating key-value form ===
{"level":"INFO","msg":"kv form","user_id":42,"plan":"pro","active":true}

=== the typed Attr form: same output, no boxing ===
{"level":"INFO","msg":"attr form","user_id":42,"plan":"pro","active":true}

=== every typed constructor ===
{"level":"INFO","msg":"types","s":"text","i":-1,"i64":1099511627776,"u64":18446744073709551615,"f":1.5,"b":true,"d":1500000000,"t":"2026-03-07T15:04:05Z","slice":[1,2,3],"map":{"a":1},"nil":null,"err":"boom"}

=== THE most common slog bug: an odd number of arguments ===
{"level":"INFO","msg":"odd count","user_id":42,"!BADKEY":"orphan"}
{"level":"INFO","msg":"wrong order","!BADKEY":42,"!BADKEY":"user_id"}
{"level":"INFO","msg":"non-string key","!BADKEY":1,"!BADKEY":"one"}

=== mixing the two forms works ===
{"level":"INFO","msg":"mixed","typed":1,"loose":2}

=== slog.Group nests ===
{"level":"INFO","msg":"request","http":{"method":"POST","status":201,"client":{"ip":"203.0.113.7"}},"db_queries":3}
{"level":"INFO","msg":"group, kv style","http":{"method":"GET","status":200}}
{"level":"INFO","msg":"empty group is omitted","kept":"yes"}

=== With derives a child logger ===
{"level":"INFO","msg":"from the child","service":"orders","version":"1.4.2"}
{"level":"INFO","msg":"two levels of With","service":"orders","version":"1.4.2","component":"repo"}
{"level":"INFO","msg":"the parent is unchanged"}

=== WithGroup namespaces everything after it ===
{"level":"INFO","msg":"slow query","db":{"pool":"primary","duration_ms":1840,"table":"orders"}}
{"level":"INFO","msg":"nested groups","a":{"b":{"k":"v"}}}

=== an empty Attr is dropped ===
{"level":"INFO","msg":"with an empty attr","k":"v"}
```

**`!BADKEY` is the single most common `slog` bug.** An odd number of arguments, or a
non-string where a key belongs, produces `"!BADKEY"` instead of an error — and in the
`"wrong order"` line it produced the key **twice in one JSON object**, which some
consumers will reject and others will silently de-duplicate. The protection is `go vet`,
which catches all three cases above; run it in CI and this bug never ships. `LogAttrs`
removes the possibility entirely, since its parameters are typed `...slog.Attr`.

**`slog.Duration` emits nanoseconds** (`1500000000`), not `"1.5s"`. Dashboards usually
want milliseconds, which is a one-line `ReplaceAttr` (below) or just
`slog.Int64("took_ms", d.Milliseconds())`.

**`slog.Any` on an `error` renders `err.Error()`** — `"boom"` — so a `%w`-wrapped chain
arrives as its full flattened message. There is no `slog.Error` attr constructor;
`slog.Any("err", err)` is the idiom.

**`With` returns a new logger; the parent is untouched.** `*slog.Logger` is immutable and
safe to share across goroutines, which makes the per-request-logger pattern (18.4)
race-free, and it computes the attrs **once** rather than per call — so base attributes
belong there.

**`WithGroup` prefixes every subsequent attr,** including those from a later `With`, and it
nests. In `TextHandler` the effect is dotted keys (`db.pool=primary`), in JSON a nested
object.

**Empty things vanish:** an empty `slog.Group` and a zero `slog.Attr{}` are both dropped,
so a conditional attribute can return `slog.Attr{}` instead of branching around the call.

## LogValuer: lazy and redacted values

```go
type LogValuer interface {
  LogValue() Value
}
```

`slog` calls `LogValue()` on any value implementing it, recursively, just before
formatting. Two uses: redaction that travels with the type, and deferring expensive work
until the record is known to be kept.

```go
package main

import (
  "fmt"
  "log/slog"
  "os"
  "strings"
)

func stripTime(groups []string, a slog.Attr) slog.Attr {
  if a.Key == slog.TimeKey && len(groups) == 0 {
    return slog.Attr{}
  }
  return a
}

type Token string

func (t Token) LogValue() slog.Value {
  if t == "" {
    return slog.StringValue("")
  }
  return slog.StringValue(fmt.Sprintf("[REDACTED %d chars]", len(t)))
}

type Email string

func (e Email) LogValue() slog.Value {
  name, domain, ok := strings.Cut(string(e), "@")
  if !ok {
    return slog.StringValue("[invalid-email]")
  }
  if len(name) <= 1 {
    return slog.StringValue("*@" + domain)
  }
  return slog.StringValue(name[:1] + strings.Repeat("*", len(name)-1) + "@" + domain)
}

type User struct {
  ID    int
  Email Email
  Token Token
  Plan  string
}

func (u User) LogValue() slog.Value {
  return slog.GroupValue(
    slog.Int("id", u.ID),
    slog.Any("email", u.Email),
    slog.String("plan", u.Plan),
  )
}

type expensive struct{ calls *int }

func (e expensive) LogValue() slog.Value {
  *e.calls++
  return slog.StringValue("computed")
}

func main() {
  log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{ReplaceAttr: stripTime}))

  u := User{ID: 42, Email: "ada.lovelace@example.com", Token: "sk_live_51H8xQ2abc", Plan: "pro"}

  fmt.Println("=== without LogValue, the whole struct leaks ===")
  type Unsafe struct {
    ID    int
    Email string
    Token string
    Plan  string
  }
  log.Info("user", "user", Unsafe{42, "ada.lovelace@example.com", "sk_live_51H8xQ2abc", "pro"})

  fmt.Println("\n=== with LogValue: an allowlist of fields ===")
  log.Info("user", "user", u)

  fmt.Println("\n=== the typed fields redact themselves anywhere ===")
  log.Info("auth", "email", u.Email, "token", u.Token)
  log.Info("edges", "empty", Token(""), "bad_email", Email("nonsense"), "short", Email("a@x.com"))

  fmt.Println("\n=== THE TRAP: fmt looks for Stringer, not LogValuer ===")
  log.Error("charge failed", "err", fmt.Errorf("declined for %v", u.Token))
  fmt.Printf("fmt.Printf: %v\n", u.Token)

  fmt.Println("\n=== LogValue is LAZY: not called for a filtered-out record ===")
  calls := 0
  warnOnly := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelWarn, ReplaceAttr: stripTime,
  }))
  warnOnly.Info("dropped", "v", expensive{&calls})
  fmt.Println("  after a dropped Info, LogValue calls:", calls)
  warnOnly.Warn("kept", "v", expensive{&calls})
  fmt.Println("  after a kept Warn, LogValue calls:    ", calls)
}
```

Output:

```
=== without LogValue, the whole struct leaks ===
{"level":"INFO","msg":"user","user":{"ID":42,"Email":"ada.lovelace@example.com","Token":"sk_live_51H8xQ2abc","Plan":"pro"}}

=== with LogValue: an allowlist of fields ===
{"level":"INFO","msg":"user","user":{"id":42,"email":"a***********@example.com","plan":"pro"}}

=== the typed fields redact themselves anywhere ===
{"level":"INFO","msg":"auth","email":"a***********@example.com","token":"[REDACTED 18 chars]"}
{"level":"INFO","msg":"edges","empty":"","bad_email":"[invalid-email]","short":"*@x.com"}

=== THE TRAP: fmt looks for Stringer, not LogValuer ===
{"level":"ERROR","msg":"charge failed","err":"declined for sk_live_51H8xQ2abc"}
fmt.Printf: sk_live_51H8xQ2abc

=== LogValue is LAZY: not called for a filtered-out record ===
  after a dropped Info, LogValue calls: 0
{"level":"WARN","msg":"kept","v":"computed"}
  after a kept Warn, LogValue calls:     1
```

**`LogValue` returning a `GroupValue` is an allowlist.** The `Unsafe` struct dumped a live
API token into the log; the `User` with `LogValue` emitted `id`, a masked `email` and
`plan`. A field added to the struct next year is **invisible by default**, which is the
property that makes this hold up over time — an allowlist fails safe, a denylist does not.

**The redaction travels with the type.** `Token` and `Email` mask themselves in *any* log
line in the program, including ones written by code that has never heard of this rule.
Declaring the struct field as `Token` rather than `string` solves the problem once.

**The trap, and it is a real hole: `fmt` does not know about `LogValuer`.** Look at the
error line — the token came out as `sk_live_51H8xQ2abc`, completely unredacted, because
`fmt.Errorf("%v", ...)` looks for `fmt.Stringer`, and `Token` has `LogValue()` but no
`String()`. The same leak happens in `fmt.Printf`. The fix is to **implement both**:

```go
func (t Token) String() string { return t.LogValue().String() }
```

[18.4](../18-production/04-service-logging.md) shows this exact leak being caught and
fixed, and makes the rule explicit: a sensitive type with only one of the two methods has
a hole, and the hole is in error messages — which is precisely where sensitive values end
up.

**`LogValue` is lazy.** The dropped `Info` record did not call it at all (0 calls), so a
`LogValue` that marshals a struct or hits a cache costs nothing at a filtered level. The
flip side: it *is* called for every record that passes the level, including `Debug`, so
keep it cheap and never let it panic — a panicking `LogValue` takes down the log call.

## ReplaceAttr

`ReplaceAttr func(groups []string, a slog.Attr) slog.Attr` sees every attribute before it
is written. Return a modified `Attr` to rename or reformat, or a zero `slog.Attr{}` to drop
it. `groups` is the enclosing group path, so you can scope a rule to the top level.

```go
package main

import (
  "fmt"
  "io"
  "log/slog"
  "os"
  "strings"
  "time"
)

func main() {
  fixed := time.Date(2026, 3, 7, 15, 4, 5, 0, time.UTC)

  fmt.Println("=== BEFORE: the default field names (time frozen, not stripped) ===")
  before := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      if a.Key == slog.TimeKey && len(g) == 0 {
        return slog.Time(slog.TimeKey, fixed)
      }
      return a
    },
  }))
  before.Warn("disk nearly full", "free_pct", 4, "password", "hunter2",
    slog.Group("host", slog.String("name", "web-1")))

  fmt.Println("\n=== AFTER: renamed, re-formatted, and a field removed ===")
  after := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
    ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
      if len(groups) == 0 {
        switch a.Key {
        case slog.TimeKey:
          return slog.String("@timestamp", fixed.Format(time.RFC3339Nano))
        case slog.LevelKey:
          lvl := a.Value.Any().(slog.Level)
          return slog.String("severity", strings.ToLower(lvl.String()))
        case slog.MessageKey:
          return slog.String("message", a.Value.String())
        }
      }
      if a.Key == "password" || a.Key == "secret" {
        return slog.Attr{}
      }
      if a.Value.Kind() == slog.KindDuration {
        return slog.Int64(a.Key+"_ms", a.Value.Duration().Milliseconds())
      }
      return a
    },
  }))
  after.Warn("disk nearly full", "free_pct", 4, "password", "hunter2",
    "took", 1500*time.Millisecond,
    slog.Group("host", slog.String("name", "web-1")))

  fmt.Println("\n=== ReplaceAttr is NOT called for a Group's own key, only its contents ===")
  seen := []string{}
  spy := slog.New(slog.NewTextHandler(io.Discard, &slog.HandlerOptions{
    ReplaceAttr: func(g []string, a slog.Attr) slog.Attr {
      seen = append(seen, fmt.Sprintf("groups=%v key=%s", g, a.Key))
      return a
    },
  }))
  spy.Info("m", slog.Group("outer", slog.String("inner", "v")))
  fmt.Println("  calls:")
  for _, s := range seen {
    fmt.Println("   ", s)
  }
}
```

Output:

```
=== BEFORE: the default field names (time frozen, not stripped) ===
{"time":"2026-03-07T15:04:05Z","level":"WARN","msg":"disk nearly full","free_pct":4,"password":"hunter2","host":{"name":"web-1"}}

=== AFTER: renamed, re-formatted, and a field removed ===
{"@timestamp":"2026-03-07T15:04:05Z","severity":"warn","message":"disk nearly full","free_pct":4,"took_ms":1500,"host":{"name":"web-1"}}

=== ReplaceAttr is NOT called for a Group's own key, only its contents ===
  calls:
    groups=[] key=time
    groups=[] key=level
    groups=[] key=msg
    groups=[outer] key=inner
```

The `AFTER` line is ready for an Elastic-style index: `time` → `@timestamp`, `level` →
lowercase `severity`, `msg` → `message`, `password` gone, and `took` converted from
nanoseconds to `took_ms`. That is the entire "make slog match our log schema" problem,
solved in one function.

Three mechanics. **`groups` is why the `len(g) == 0` guard matters** — without it, a
user attribute that happens to be named `msg` inside a group would also be rewritten. The
built-in keys are `slog.TimeKey`, `slog.LevelKey`, `slog.MessageKey` and
`slog.SourceKey`, all only at the top level. **The level arrives as `slog.Level` inside an
`Any` value**, hence the type assertion. And **`ReplaceAttr` is not called for a group's
own key**, only for the attrs inside it — the spy shows one call for `inner` with
`groups=[outer]` and none for `outer` itself, so you cannot rename a group this way.

Dropping or freezing the `time` attribute is also the honest way to make log output
testable, which is exactly what every example in this lesson does.

## A custom Handler

```go
type Handler interface {
  Enabled(context.Context, Level) bool
  Handle(context.Context, Record) error
  WithAttrs(attrs []Attr) Handler
  WithGroup(name string) Handler
}
```

Four methods. The contract: `WithAttrs`/`WithGroup` must return a **new** Handler and not
mutate the receiver, and `Handle` should not be called if `Enabled` returned false.

```go
package main

import (
  "context"
  "fmt"
  "io"
  "log/slog"
  "os"
  "strings"
)

type countingHandler struct {
  inner  slog.Handler
  counts map[slog.Level]int
}

func (h *countingHandler) Enabled(ctx context.Context, l slog.Level) bool {
  return h.inner.Enabled(ctx, l)
}

func (h *countingHandler) Handle(ctx context.Context, r slog.Record) error {
  h.counts[r.Level]++
  return h.inner.Handle(ctx, r)
}

func (h *countingHandler) WithAttrs(as []slog.Attr) slog.Handler {
  return &countingHandler{inner: h.inner.WithAttrs(as), counts: h.counts}
}

func (h *countingHandler) WithGroup(name string) slog.Handler {
  return &countingHandler{inner: h.inner.WithGroup(name), counts: h.counts}
}

type lineHandler struct {
  w     io.Writer
  level slog.Level
  attrs []slog.Attr
}

func (h *lineHandler) Enabled(_ context.Context, l slog.Level) bool { return l >= h.level }

func (h *lineHandler) Handle(_ context.Context, r slog.Record) error {
  var b strings.Builder
  fmt.Fprintf(&b, "%-5s %s", r.Level, r.Message)
  for _, a := range h.attrs {
    fmt.Fprintf(&b, " | %s=%v", a.Key, a.Value)
  }
  r.Attrs(func(a slog.Attr) bool {
    fmt.Fprintf(&b, " | %s=%v", a.Key, a.Value)
    return true
  })
  fmt.Fprintln(h.w, b.String())
  return nil
}

func (h *lineHandler) WithAttrs(as []slog.Attr) slog.Handler {
  return &lineHandler{w: h.w, level: h.level, attrs: append(append([]slog.Attr{}, h.attrs...), as...)}
}

func (h *lineHandler) WithGroup(string) slog.Handler { return h }

func main() {
  fmt.Println("=== a minimal custom Handler ===")
  lh := slog.New(&lineHandler{w: os.Stdout, level: slog.LevelInfo})
  lh.Info("server started", "addr", ":8080")
  lh.With("service", "orders").Warn("pool busy", "in_use", 18)
  lh.Debug("filtered out by Enabled")

  fmt.Println("\n=== a wrapping Handler that counts records ===")
  counts := map[slog.Level]int{}
  ch := slog.New(&countingHandler{
    inner:  slog.NewTextHandler(io.Discard, &slog.HandlerOptions{Level: slog.LevelDebug}),
    counts: counts,
  })
  ch.Debug("d")
  ch.Info("i1")
  ch.Info("i2")
  ch.With("k", "v").Error("e")
  for _, l := range []slog.Level{slog.LevelDebug, slog.LevelInfo, slog.LevelWarn, slog.LevelError} {
    fmt.Printf("  %-5s %d\n", l, counts[l])
  }
}
```

Output:

```
=== a minimal custom Handler ===
INFO  server started | addr=:8080
WARN  pool busy | service=orders | in_use=18

=== a wrapping Handler that counts records ===
  DEBUG 1
  INFO  2
  WARN  0
  ERROR 1
```

**A `Record`'s attrs are iterated, not indexed.** `r.Attrs(func(slog.Attr) bool)` is a
callback that stops when you return false, and `r.NumAttrs()` counts only the call-site
attrs. The `WithAttrs` attrs are stored separately on the Handler and must be emitted by
you, which is why `lineHandler` prints both lists. The other `Record` fields are `Level`,
`Message`, `Time` and `PC` (the caller, for `AddSource`).

**The wrapping Handler is the pattern worth internalizing.** `countingHandler` delegates
every method to an inner Handler and adds behaviour in `Handle` — that is how you build
sampling, metrics from log records, redaction enforcement, or a fan-out to two
destinations, without reimplementing formatting. Note `WithAttrs` must delegate *and*
rewrap, or the added behaviour disappears from derived loggers.

**`WithGroup` returning the receiver (as `lineHandler` does) is a shortcut, not correct.**
A conforming Handler must prefix subsequent attrs with the group name; the proper
implementation is in the stdlib's `slogtest` harness, and `testing/slogtest.TestHandler`
will tell you exactly which contract rules you broke.

## Performance

The usual advice is "typed `Attr`s are faster than key-value pairs". Here is a real
measurement.

```go
package main

import (
  "context"
  "io"
  "log/slog"
  "testing"
)

var (
  jsonLog = slog.New(slog.NewJSONHandler(io.Discard, nil))
  warnLog = slog.New(slog.NewJSONHandler(io.Discard, &slog.HandlerOptions{Level: slog.LevelWarn}))
)

func BenchmarkKeyValue(b *testing.B) {
  for b.Loop() {
    jsonLog.Info("request", "method", "GET", "path", "/orders", "status", 200)
  }
}

func BenchmarkTypedAttrs(b *testing.B) {
  for b.Loop() {
    jsonLog.Info("request",
      slog.String("method", "GET"), slog.String("path", "/orders"), slog.Int("status", 200))
  }
}

func BenchmarkLogAttrs(b *testing.B) {
  ctx := context.Background()
  for b.Loop() {
    jsonLog.LogAttrs(ctx, slog.LevelInfo, "request",
      slog.String("method", "GET"), slog.String("path", "/orders"), slog.Int("status", 200))
  }
}

func BenchmarkDisabledKeyValue(b *testing.B) {
  for b.Loop() {
    warnLog.Info("request", "method", "GET", "path", "/orders", "status", 200)
  }
}
```

`go test -bench=. -benchmem`. Machine-specific (darwin/arm64, Apple M4 Pro) and a few
percent noisy; read the allocation columns.

Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkKeyValue-12            	 3152918	       366.0 ns/op	       0 B/op	       0 allocs/op
BenchmarkTypedAttrs-12          	 3176691	       377.1 ns/op	     144 B/op	       3 allocs/op
BenchmarkLogAttrs-12            	 3429542	       349.4 ns/op	       0 B/op	       0 allocs/op
BenchmarkDisabledKeyValue-12    	176746255	         6.797 ns/op	       0 B/op	       0 allocs/op
```

**This contradicts the common advice, so read it carefully.** Passing typed `Attr`s to
`Info` is the **worst** of the three: 3 allocations, 144 B/op, because `Info` takes
`...any` and each `slog.Attr` struct must be boxed into an interface. The plain key-value
form boxes small values the runtime has cached, so it allocates **nothing**. The typed
form only pays off through **`LogAttrs`**, whose parameter is `...slog.Attr` with no
interface conversion at all — the zero-allocation, fastest path.

So the practical ranking: `LogAttrs` with typed attrs is fastest; plain key-value `Info` is
essentially tied and far more readable; typed attrs through `Info` is the one combination
to avoid. For ordinary code, **use the key-value form** — shorter, allocation-free, and
guarded by `go vet`. Reach for `LogAttrs` in a genuinely hot path. (`TextHandler`
benchmarks within noise of `JSONHandler`, so format is not the deciding factor.)

**A filtered-out call costs ~7 ns and zero allocations**, roughly 50x cheaper than an
emitted one. That is what makes leaving `Debug` calls in production code free, and it is
why an `Enabled` guard is usually unnecessary.

`zap` and `zerolog` are still faster than `slog` for high-volume logging; `slog` is fast
enough for essentially every service and it is in the standard library, which is why this
course uses it.

## log.Logger still fits some cases

For a short-lived CLI, a build script, or a test helper where the output is for a human
reading a terminal and nothing will ever query it, the old `log` package is one line and
perfectly fine — `log.Fatalf` in particular has no `slog` equivalent. Everything that runs
as a service gets `slog`.

## Quick Reference

| Need | Call |
|---|---|
| A JSON logger | `slog.New(slog.NewJSONHandler(os.Stdout, opts))` |
| Set the global + redirect `log` | `slog.SetDefault(logger)` |
| A `*log.Logger` for a library | `slog.NewLogLogger(handler, slog.LevelError)` |
| Fastest, vet-proof form | `log.LogAttrs(ctx, level, msg, slog.String(...), ...)` |
| A child logger | `log.With("service", "api")` |
| Minimum level | `&slog.HandlerOptions{Level: slog.LevelDebug}` |
| Runtime-adjustable level | `v := new(slog.LevelVar); v.Set(...)`; pass `v` as `Level` |
| Rename / drop / reformat a field | `HandlerOptions.ReplaceAttr` |
| Redact a type everywhere | implement `LogValue() slog.Value` **and** `String() string` |
| Log an error | `slog.Any("err", err)` |
| Reproducible output in tests | `ReplaceAttr` dropping `slog.TimeKey` + a `bytes.Buffer` |

| Trap | Reality |
|---|---|
| An odd number of k/v args | `"!BADKEY"` in the output, no error — `go vet` catches it |
| Typed `Attr`s passed to `Info` | 3 allocations; use `LogAttrs` or plain k/v |
| `slog.Duration` in a dashboard | emits **nanoseconds** |
| `LogValue` without `String` | leaks through `fmt.Errorf("%v")` and `fmt.Printf` |
| `ReplaceAttr` without the `groups` guard | rewrites same-named attrs inside groups |
| Trying to rename a group with `ReplaceAttr` | it is never called for the group's own key |

## Common Mistakes

**Passing an odd number of key-value arguments.** The output gets `!BADKEY` instead of an
error; run `go vet`, which detects it.

**Using typed `Attr`s with `Info`.** That is the slowest combination — either use plain
key-value pairs or switch to `LogAttrs`.

**Implementing `LogValue` without `String`.** The value still leaks through every `%v` in
an error message; implement both, with `String` delegating to `LogValue`.

**Interpolating data into the message.** `slog.Info(fmt.Sprintf("user %d ...", id))`
produces one unqueryable field and destroys `msg` as a group-by key; pass attributes.

**Forgetting the `len(groups) == 0` guard in `ReplaceAttr`.** A user attribute named
`time` or `msg` inside a group gets silently rewritten.

**Asserting on log output that contains a timestamp.** The test is flaky by construction;
strip or freeze `slog.TimeKey`.

## Exercises

1. Write `func NewTestLogger() (*slog.Logger, *bytes.Buffer)` whose output is fully
   deterministic, then test that a function logs exactly one `WARN` record with a given
   field — and that a secret never appears in the raw buffer.
2. Implement a `slog.Handler` that writes human-coloured output to a terminal and JSON
   otherwise, and make `testing/slogtest.TestHandler` pass on it. Report which contract
   rules you failed first.
3. Build a sampling Handler wrapper: pass everything at `Warn` and above, 1-in-N at
   `Info`, adding a `sample_rate` attribute. Verify the ratio over 10,000 records.
4. Add `LogValue()` and `String()` to an `IPAddress` type that masks the last octet, then
   write the test proving no unmasked form appears in either a log line or an
   `fmt.Errorf("%v")` message.
5. Write a `ReplaceAttr` that converts every `time.Duration` attribute to a `_ms` integer
   field and every `time.Time` to Unix milliseconds, and table-test it against attributes
   nested two groups deep.
6. Benchmark your own hot path three ways — `Info` with k/v, `Info` with typed attrs and
   `LogAttrs` — at an enabled and a disabled level, and explain each allocation count.

---

[← 13.11 flag: Command-Line Parsing](11-flag.md) · [Course Index](../README.md) · [13.13 embed: Embedding Files in the Binary →](13-embed.md)
