[← 18.1 Project Layout: cmd/ and internal/](01-project-layout.md) · [Course Index](../README.md) · [18.3 Authentication with JWT →](03-auth-jwt.md)

# 18.2 — Configuration via Environment

## What You'll Learn

- `os.Getenv` vs `os.LookupEnv` — presence and emptiness are different questions
- A typed `Config` with `Load() (*Config, error)` that validates everything and reports **all** errors via `errors.Join`
- A `String()` method that redacts secrets, proven by a test that greps its own output
- Precedence (flags > env > file > defaults), a reflect/tag loader, and `t.Setenv` testing

## Coming From Another Language

| Other | Go |
|---|---|
| `process.env.PORT` — always `string \| undefined` | `os.Getenv` returns `""` for both unset and empty; use `LookupEnv` to tell them apart |
| `dotenv` loaded automatically at import | nothing is automatic; `.env` support is a library you opt into, for dev only |
| `pydantic-settings` / `zod` schema does the parsing | you write the parse-and-validate function; it is 60 lines and fully explicit |
| config object typed `any`, errors surface at first use | a typed struct validated at startup; the process refuses to boot |
| Rails `config/environments/production.rb` per env | one code path, different *values*; no `if env == "prod"` branches |

## Getenv versus LookupEnv

```go
package main

import (
  "fmt"
  "os"
)

func main() {
  os.Setenv("EMPTY", "")
  fmt.Printf("Getenv(%q)    = %q\n", "EMPTY", os.Getenv("EMPTY"))
  fmt.Printf("Getenv(%q)  = %q\n", "MISSING", os.Getenv("MISSING"))

  v, ok := os.LookupEnv("EMPTY")
  fmt.Printf("LookupEnv(%q)   = %q, %v\n", "EMPTY", v, ok)
  v, ok = os.LookupEnv("MISSING")
  fmt.Printf("LookupEnv(%q) = %q, %v\n", "MISSING", v, ok)
}
```

Output:

```
Getenv("EMPTY")    = ""
Getenv("MISSING")  = ""
LookupEnv("EMPTY")   = "", true
LookupEnv("MISSING") = "", false
```

`Getenv` cannot distinguish the two. That matters for any setting whose empty value is
meaningful — `ALLOWED_ORIGINS=` to mean "deliberately none" versus unset to mean "use the
default" — and for diagnostics, because "you forgot to set `DATABASE_URL`" and "you set
`DATABASE_URL` to the empty string" need different error messages.

Idiomatic: `LookupEnv` in a config loader, `Getenv` only for a throwaway lookup where unset
and empty really are the same.

## Twelve-factor config, in two sentences

Configuration is everything that differs between deploys — credentials, hostnames, ports,
feature flags — and it belongs in the environment, not in files committed to the repo, so the
same built artifact runs in staging and production with no rebuild. The environment is the
lowest common denominator every orchestrator, CI system, and PaaS can set, which is why it
won; that is the whole argument.

## A typed Config with full validation

The loader does four things: start from defaults, read and parse each variable, validate
cross-field rules, and return **every** error at once. The last point is the one most
loaders get wrong — reporting only the first missing variable means a five-variable
misconfiguration takes five deploy cycles to fix.

```go
package main

import (
  "errors"
  "fmt"
  "log/slog"
  "net/url"
  "os"
  "strconv"
  "strings"
  "time"
)

type Config struct {
  Env             string
  Port            int
  DatabaseURL     string
  JWTSecret       string
  LogLevel        slog.Level
  Debug           bool
  ShutdownTimeout time.Duration
  AllowedOrigins  []string
  UpstreamURL     *url.URL
}

func (c *Config) String() string {
  return fmt.Sprintf("Config{Env:%s Port:%d DatabaseURL:%s JWTSecret:%s LogLevel:%s Debug:%t ShutdownTimeout:%s AllowedOrigins:%v UpstreamURL:%s}",
    c.Env, c.Port, redactURL(c.DatabaseURL), redact(c.JWTSecret),
    c.LogLevel, c.Debug, c.ShutdownTimeout, c.AllowedOrigins, c.UpstreamURL)
}

func redact(s string) string {
  if s == "" {
    return ""
  }
  return "[REDACTED len=" + strconv.Itoa(len(s)) + "]"
}

func redactURL(raw string) string {
  u, err := url.Parse(raw)
  if err != nil || u.User == nil {
    return raw
  }
  u.User = url.UserPassword(u.User.Username(), "xxxxx")
  return u.String()
}

func Load() (*Config, error) {
  c := &Config{
    Env:             "development",
    Port:            8080,
    LogLevel:        slog.LevelInfo,
    ShutdownTimeout: 15 * time.Second,
  }
  var errs []error
  add := func(key string, err error) {
    if err != nil {
      errs = append(errs, fmt.Errorf("%s: %w", key, err))
    }
  }

  add("DATABASE_URL", required("DATABASE_URL", &c.DatabaseURL))
  add("JWT_SECRET", required("JWT_SECRET", &c.JWTSecret))
  add("ENV", optString("ENV", &c.Env))
  add("PORT", optInt("PORT", &c.Port))
  add("DEBUG", optBool("DEBUG", &c.Debug))
  add("SHUTDOWN_TIMEOUT", optDuration("SHUTDOWN_TIMEOUT", &c.ShutdownTimeout))
  add("ALLOWED_ORIGINS", optCSV("ALLOWED_ORIGINS", &c.AllowedOrigins))
  add("UPSTREAM_URL", optURL("UPSTREAM_URL", &c.UpstreamURL))
  add("LOG_LEVEL", optLevel("LOG_LEVEL", &c.LogLevel))

  if c.Port < 1 || c.Port > 65535 {
    errs = append(errs, fmt.Errorf("PORT: %d out of range 1-65535", c.Port))
  }
  switch c.Env {
  case "development", "staging", "production":
  default:
    errs = append(errs, fmt.Errorf("ENV: %q must be development, staging or production", c.Env))
  }
  if len(c.JWTSecret) > 0 && len(c.JWTSecret) < 32 {
    errs = append(errs, fmt.Errorf("JWT_SECRET: must be at least 32 bytes, got %d", len(c.JWTSecret)))
  }

  if len(errs) > 0 {
    return nil, errors.Join(errs...)
  }
  return c, nil
}

func required(key string, dst *string) error {
  v, ok := os.LookupEnv(key)
  if !ok || v == "" {
    return errors.New("required but not set")
  }
  *dst = v
  return nil
}

func optString(key string, dst *string) error {
  if v, ok := os.LookupEnv(key); ok {
    *dst = v
  }
  return nil
}

func optInt(key string, dst *int) error {
  v, ok := os.LookupEnv(key)
  if !ok || v == "" {
    return nil
  }
  n, err := strconv.Atoi(v)
  if err != nil {
    return err
  }
  *dst = n
  return nil
}

func optBool(key string, dst *bool) error {
  v, ok := os.LookupEnv(key)
  if !ok || v == "" {
    return nil
  }
  b, err := strconv.ParseBool(v)
  if err != nil {
    return err
  }
  *dst = b
  return nil
}

func optDuration(key string, dst *time.Duration) error {
  v, ok := os.LookupEnv(key)
  if !ok || v == "" {
    return nil
  }
  d, err := time.ParseDuration(v)
  if err != nil {
    return err
  }
  *dst = d
  return nil
}

func optCSV(key string, dst *[]string) error {
  v, ok := os.LookupEnv(key)
  if !ok || v == "" {
    return nil
  }
  parts := strings.Split(v, ",")
  out := make([]string, 0, len(parts))
  for _, p := range parts {
    if p = strings.TrimSpace(p); p != "" {
      out = append(out, p)
    }
  }
  *dst = out
  return nil
}

func optURL(key string, dst **url.URL) error {
  v, ok := os.LookupEnv(key)
  if !ok || v == "" {
    return nil
  }
  u, err := url.Parse(v)
  if err != nil {
    return err
  }
  if u.Scheme != "http" && u.Scheme != "https" {
    return fmt.Errorf("scheme %q must be http or https", u.Scheme)
  }
  *dst = u
  return nil
}

func optLevel(key string, dst *slog.Level) error {
  v, ok := os.LookupEnv(key)
  if !ok || v == "" {
    return nil
  }
  var l slog.Level
  if err := l.UnmarshalText([]byte(v)); err != nil {
    return err
  }
  *dst = l
  return nil
}

func main() {
  fmt.Println("=== valid environment ===")
  for k, v := range map[string]string{
    "DATABASE_URL":     "postgres://app:hunter2@db.internal:5432/prod?sslmode=require",
    "JWT_SECRET":       "0123456789abcdef0123456789abcdef",
    "ENV":              "production",
    "PORT":             "9000",
    "DEBUG":            "true",
    "SHUTDOWN_TIMEOUT": "30s",
    "ALLOWED_ORIGINS":  "https://app.example.com, https://admin.example.com",
    "UPSTREAM_URL":     "https://payments.internal/v1",
    "LOG_LEVEL":        "warn",
  } {
    os.Setenv(k, v)
  }
  cfg, err := Load()
  if err != nil {
    fmt.Println("err:", err)
  } else {
    fmt.Println(cfg)
  }

  fmt.Println("\n=== every value invalid: all errors at once ===")
  os.Unsetenv("DATABASE_URL")
  os.Unsetenv("JWT_SECRET")
  for k, v := range map[string]string{
    "ENV":              "prod",
    "PORT":             "eighty",
    "DEBUG":            "yes please",
    "SHUTDOWN_TIMEOUT": "30 seconds",
    "UPSTREAM_URL":     "ftp://payments.internal",
    "LOG_LEVEL":        "verbose",
  } {
    os.Setenv(k, v)
  }
  _, err = Load()
  fmt.Println(err)

  fmt.Println("\n=== short secret and out-of-range port ===")
  os.Setenv("DATABASE_URL", "postgres://localhost/dev")
  os.Setenv("JWT_SECRET", "tooshort")
  os.Setenv("ENV", "staging")
  os.Setenv("PORT", "70000")
  os.Setenv("DEBUG", "false")
  os.Setenv("SHUTDOWN_TIMEOUT", "5s")
  os.Setenv("UPSTREAM_URL", "https://x.internal")
  os.Setenv("LOG_LEVEL", "debug")
  _, err = Load()
  fmt.Println(err)
}
```

Output:

```
=== valid environment ===
Config{Env:production Port:9000 DatabaseURL:postgres://app:xxxxx@db.internal:5432/prod?sslmode=require JWTSecret:[REDACTED len=32] LogLevel:WARN Debug:true ShutdownTimeout:30s AllowedOrigins:[https://app.example.com https://admin.example.com] UpstreamURL:https://payments.internal/v1}

=== every value invalid: all errors at once ===
DATABASE_URL: required but not set
JWT_SECRET: required but not set
PORT: strconv.Atoi: parsing "eighty": invalid syntax
DEBUG: strconv.ParseBool: parsing "yes please": invalid syntax
SHUTDOWN_TIMEOUT: time: unknown unit " seconds" in duration "30 seconds"
UPSTREAM_URL: scheme "ftp" must be http or https
LOG_LEVEL: slog: level string "verbose": unknown name
ENV: "prod" must be development, staging or production
```

```
=== short secret and out-of-range port ===
PORT: 70000 out of range 1-65535
JWT_SECRET: must be at least 32 bytes, got 8
```

Seven real parse errors and a cross-field one, in a single pass. The things worth noting:

**`errors.Join` gives one `error` whose `Error()` is the messages newline-separated**
([8.5](../08-errors/05-join.md)). Each is already prefixed with its variable name, so the
message is actionable without a stack trace.

**Every parse error comes from the standard library verbatim.** `strconv.ParseBool: parsing
"yes please": invalid syntax` names the function, the input and the problem. Do not rewrite
these — wrap them with the variable name and pass them through.

**`slog.Level` implements `encoding.TextUnmarshaler`**, so `optLevel` is three lines and
accepts `debug`/`info`/`warn`/`error` case-insensitively plus offsets like `warn+2`. Any type
with `UnmarshalText` can be parsed from the environment the same way.

**The duration error is the one that bites in production.** `SHUTDOWN_TIMEOUT=30` is a
*valid* `time.Duration` literal parse failure (`missing unit`), and `SHUTDOWN_TIMEOUT=30s`
is what you meant. Durations must carry a unit — `300ms`, `5s`, `2m`, `1h30m`
([13.4](../13-stdlib/04-time.md)).

### The redacting String()

Look at the first output line. `DATABASE_URL` went in as
`postgres://app:hunter2@db.internal:5432/prod` and printed as
`postgres://app:xxxxx@db.internal:5432/prod` — username kept (useful for debugging), password
replaced. `JWT_SECRET` printed as `[REDACTED len=32]`, which tells you it is set and the right
length without revealing it.

This matters because `fmt.Println(cfg)`, `slog.Info("config", "cfg", cfg)`, `%v` in an error,
and a panic dump all go through `String()` once the method exists. Define it on the pointer
receiver and always log `cfg` rather than individual fields, and a secret cannot reach your log
aggregator by accident. [18.4](04-service-logging.md) does the same thing for domain types with
`slog.LogValuer`.

Secrets themselves never go in code, never in the repo, and never in a container image. They
come from the environment, injected by a secrets manager (Kubernetes Secret, AWS Secrets
Manager, Vault) at deploy time.

## Fail fast, with a message someone can act on

Validate at startup and refuse to run. A service that boots with a bad `DATABASE_URL` and
discovers it on the first request has turned a deploy-time failure into a user-facing outage.

```go
package main

import (
  "errors"
  "fmt"
  "os"
)

func Load() error {
  return errors.Join(
    errors.New("DATABASE_URL: required but not set"),
    errors.New("PORT: strconv.Atoi: parsing \"eighty\": invalid syntax"),
  )
}

func main() {
  if err := Load(); err != nil {
    fmt.Fprintf(os.Stderr, "fatal: invalid configuration:\n")
    for _, line := range splitJoined(err) {
      fmt.Fprintf(os.Stderr, "  - %s\n", line)
    }
    os.Exit(2)
  }
}

func splitJoined(err error) []string {
  var out []string
  if u, ok := err.(interface{ Unwrap() []error }); ok {
    for _, e := range u.Unwrap() {
      out = append(out, e.Error())
    }
    return out
  }
  return []string{err.Error()}
}
```

Output (on stderr; `go run` then reports the non-zero exit):

```
fatal: invalid configuration:
  - DATABASE_URL: required but not set
  - PORT: strconv.Atoi: parsing "eighty": invalid syntax
exit status 2
```

Configuration failure is the one place where exiting from `main` beats returning an error up a
stack — there is no stack yet. Use a distinct exit code (2 here) so a supervisor can tell
"misconfigured, do not retry" from "crashed, restart me". Write to **stderr**, not through
`slog`, because the logger's own level comes from the config you just failed to load.

`splitJoined` relies on `errors.Join`'s value implementing `Unwrap() []error` — that is the
documented interface, and it is how you get the individual errors back out of a join.

## Precedence: flags > env > file > defaults

Apply the sources in increasing order of priority, each overwriting the last. Flags win
because they are the most specific thing a human just typed.

```go
package main

import (
  "bufio"
  "flag"
  "fmt"
  "os"
  "strings"
)

type Settings struct {
  Port     int
  LogLevel string
  Env      string
}

func resolve(args []string, file map[string]string) (*Settings, error) {
  s := &Settings{Port: 8080, LogLevel: "info", Env: "development"}

  if v, ok := file["PORT"]; ok {
    fmt.Sscanf(v, "%d", &s.Port)
  }
  if v, ok := file["LOG_LEVEL"]; ok {
    s.LogLevel = v
  }
  if v, ok := file["ENV"]; ok {
    s.Env = v
  }

  if v, ok := os.LookupEnv("PORT"); ok {
    fmt.Sscanf(v, "%d", &s.Port)
  }
  if v, ok := os.LookupEnv("LOG_LEVEL"); ok {
    s.LogLevel = v
  }
  if v, ok := os.LookupEnv("ENV"); ok {
    s.Env = v
  }

  fs := flag.NewFlagSet("server", flag.ContinueOnError)
  fs.SetOutput(os.Stdout)
  port := fs.Int("port", s.Port, "listen port")
  level := fs.String("log-level", s.LogLevel, "log level")
  env := fs.String("env", s.Env, "environment")
  if err := fs.Parse(args); err != nil {
    return nil, err
  }
  s.Port, s.LogLevel, s.Env = *port, *level, *env
  return s, nil
}

func parseDotEnv(r *bufio.Scanner) map[string]string {
  out := map[string]string{}
  for r.Scan() {
    line := strings.TrimSpace(r.Text())
    if line == "" || strings.HasPrefix(line, "#") {
      continue
    }
    k, v, ok := strings.Cut(line, "=")
    if !ok {
      continue
    }
    out[strings.TrimSpace(k)] = strings.Trim(strings.TrimSpace(v), `"'`)
  }
  return out
}

func main() {
  dotenv := parseDotEnv(bufio.NewScanner(strings.NewReader(`
# local development only
PORT=3000
LOG_LEVEL="debug"
ENV=development
`)))
  fmt.Printf(".env file     -> %v\n", dotenv)

  s, _ := resolve(nil, nil)
  fmt.Printf("defaults only -> %+v\n", *s)

  s, _ = resolve(nil, dotenv)
  fmt.Printf("+ file        -> %+v\n", *s)

  os.Setenv("PORT", "9000")
  s, _ = resolve(nil, dotenv)
  fmt.Printf("+ env         -> %+v\n", *s)

  s, _ = resolve([]string{"-port=7777", "-env=staging"}, dotenv)
  fmt.Printf("+ flags       -> %+v\n", *s)
}
```

Output:

```
.env file     -> map[ENV:development LOG_LEVEL:debug PORT:3000]
defaults only -> {Port:8080 LogLevel:info Env:development}
+ file        -> {Port:3000 LogLevel:debug Env:development}
+ env         -> {Port:9000 LogLevel:debug Env:development}
+ flags       -> {Port:7777 LogLevel:debug Env:staging}
```

The trick that makes `flag` compose cleanly: register each flag with the **already-resolved**
value as its default ([13.11](../13-stdlib/11-flag.md)). An unspecified flag then leaves the
env/file value intact, and a specified one wins — no "was this flag set?" bookkeeping. Note
`fmt` prints maps with sorted keys, which is why the first line is reproducible even though
map iteration is not ([7.6](../07-slices-maps/06-maps.md)).

`.env` files are a **local development** convenience only: never commit one with real
credentials, and never read one in production, where the orchestrator sets the environment
directly. `github.com/joho/godotenv` does the parsing properly (quoting, `export` prefixes,
multi-line values) if you want more than the twelve lines above.

## Different environments without an `if env == "prod"` thicket

The anti-pattern:

```go
if cfg.Env == "production" {
  logger = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
  timeout = 30 * time.Second
} else {
  logger = slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug}))
  timeout = 5 * time.Second
}
```

Every one of those branches is untested in the other environment, and every new difference
adds a branch. Make them **values**, not code paths:

```go
type Config struct {
  LogFormat       string        // "json" | "text"
  LogLevel        slog.Level
  ShutdownTimeout time.Duration
}
```

One code path reads `cfg.LogFormat`; staging and production differ only in what they set.
`cfg.Env` then survives for exactly two legitimate uses — as a log attribute and a metrics
label — and for the small set of guards that genuinely must not be possible in production,
like enabling a debug endpoint. Keep that list short enough to read in one screen.

## A reflect/tag-driven loader

Nine `optX` calls per config is fine; sixty is not. Struct tags
([4.7](../04-types-structs/07-struct-tags.md)) plus `reflect`
([13.15](../13-stdlib/15-reflect.md)) collapse the whole thing into a declaration.

```go
package main

import (
  "errors"
  "fmt"
  "os"
  "reflect"
  "strconv"
  "strings"
  "time"
)

type AppConfig struct {
  Port        int           `env:"PORT" default:"8080"`
  Host        string        `env:"HOST" default:"0.0.0.0"`
  DatabaseURL string        `env:"DATABASE_URL,required"`
  Timeout     time.Duration `env:"TIMEOUT" default:"5s"`
  Debug       bool          `env:"DEBUG" default:"false"`
  Origins     []string      `env:"ORIGINS" default:"http://localhost:3000"`
}

func Unmarshal(dst any) error {
  v := reflect.ValueOf(dst)
  if v.Kind() != reflect.Pointer || v.Elem().Kind() != reflect.Struct {
    return errors.New("env: destination must be a pointer to struct")
  }
  v = v.Elem()
  t := v.Type()

  var errs []error
  for i := range t.NumField() {
    f := t.Field(i)
    tag, ok := f.Tag.Lookup("env")
    if !ok || !f.IsExported() {
      continue
    }
    name, opts, _ := strings.Cut(tag, ",")
    required := strings.Contains(opts, "required")

    raw, present := os.LookupEnv(name)
    if !present || raw == "" {
      if required {
        errs = append(errs, fmt.Errorf("%s: required but not set", name))
        continue
      }
      raw = f.Tag.Get("default")
      if raw == "" {
        continue
      }
    }
    if err := setField(v.Field(i), raw); err != nil {
      errs = append(errs, fmt.Errorf("%s=%q: %w", name, raw, err))
    }
  }
  return errors.Join(errs...)
}

func setField(fv reflect.Value, raw string) error {
  if fv.Type() == reflect.TypeFor[time.Duration]() {
    d, err := time.ParseDuration(raw)
    if err != nil {
      return err
    }
    fv.SetInt(int64(d))
    return nil
  }
  switch fv.Kind() {
  case reflect.String:
    fv.SetString(raw)
  case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
    n, err := strconv.ParseInt(raw, 10, fv.Type().Bits())
    if err != nil {
      return err
    }
    fv.SetInt(n)
  case reflect.Bool:
    b, err := strconv.ParseBool(raw)
    if err != nil {
      return err
    }
    fv.SetBool(b)
  case reflect.Slice:
    if fv.Type().Elem().Kind() != reflect.String {
      return fmt.Errorf("unsupported slice element %s", fv.Type().Elem())
    }
    parts := strings.Split(raw, ",")
    for i := range parts {
      parts[i] = strings.TrimSpace(parts[i])
    }
    fv.Set(reflect.ValueOf(parts))
  default:
    return fmt.Errorf("unsupported kind %s", fv.Kind())
  }
  return nil
}

func main() {
  fmt.Println("=== defaults, with the one required var set ===")
  os.Setenv("DATABASE_URL", "postgres://localhost/dev")
  var c AppConfig
  if err := Unmarshal(&c); err != nil {
    fmt.Println("err:", err)
  }
  fmt.Printf("%+v\n", c)

  fmt.Println("\n=== overrides ===")
  os.Setenv("PORT", "9443")
  os.Setenv("TIMEOUT", "250ms")
  os.Setenv("DEBUG", "1")
  os.Setenv("ORIGINS", "https://a.example.com, https://b.example.com")
  var c2 AppConfig
  if err := Unmarshal(&c2); err != nil {
    fmt.Println("err:", err)
  }
  fmt.Printf("%+v\n", c2)

  fmt.Println("\n=== bad values and a missing required var ===")
  os.Unsetenv("DATABASE_URL")
  os.Setenv("PORT", "99999999999999999999")
  os.Setenv("TIMEOUT", "soon")
  os.Setenv("DEBUG", "maybe")
  var c3 AppConfig
  fmt.Println(Unmarshal(&c3))
}
```

Output:

```
=== defaults, with the one required var set ===
{Port:8080 Host:0.0.0.0 DatabaseURL:postgres://localhost/dev Timeout:5s Debug:false Origins:[http://localhost:3000]}

=== overrides ===
{Port:9443 Host:0.0.0.0 DatabaseURL:postgres://localhost/dev Timeout:250ms Debug:true Origins:[https://a.example.com https://b.example.com]}

=== bad values and a missing required var ===
PORT="99999999999999999999": strconv.ParseInt: parsing "99999999999999999999": value out of range
DATABASE_URL: required but not set
TIMEOUT="soon": time: invalid duration "soon"
DEBUG="maybe": strconv.ParseBool: parsing "maybe": invalid syntax
```

Four things to notice. `time.Duration` has to be checked **before** the `Kind()` switch,
because its kind is `Int64` and it would otherwise parse `5s` as an integer and fail.
`DEBUG=1` is true — `strconv.ParseBool` accepts `1 t T TRUE true True 0 f F FALSE false
False` and nothing else, so `yes`/`on` are errors. `reflect.TypeFor[time.Duration]()` is the
generic form that avoids the old `reflect.TypeOf(time.Duration(0))` dance. And the loader
still collects every error, which is the property worth preserving when you trade explicitness
for tags.

The trade-off: the tag version is shorter and self-documenting, but validation rules
(`PORT` in range, `JWT_SECRET` long enough) have nowhere to live except a separate
`Validate() error` method, and tag typos fail silently rather than at compile time. For a
large config, use tags for parsing and a hand-written `Validate()` for the rules.

## Testing config loading with t.Setenv

`t.Setenv` sets a variable for one test and restores it on cleanup
([14.3](../14-testing/03-fixtures-helpers.md)). It also marks the test as unable to run in
parallel, which is correct — the environment is process-global.

```go
package main

import (
  "log/slog"
  "strings"
  "testing"
  "time"
)

func setValid(t *testing.T) {
  t.Helper()
  t.Setenv("DATABASE_URL", "postgres://localhost/test")
  t.Setenv("JWT_SECRET", "0123456789abcdef0123456789abcdef")
}

func TestLoadDefaults(t *testing.T) {
  setValid(t)
  cfg, err := Load()
  if err != nil {
    t.Fatalf("Load() = %v", err)
  }
  if cfg.Port != 8080 {
    t.Errorf("Port = %d, want 8080", cfg.Port)
  }
  if cfg.Env != "development" {
    t.Errorf("Env = %q, want development", cfg.Env)
  }
  if cfg.LogLevel != slog.LevelInfo {
    t.Errorf("LogLevel = %v, want INFO", cfg.LogLevel)
  }
  if cfg.ShutdownTimeout != 15*time.Second {
    t.Errorf("ShutdownTimeout = %v, want 15s", cfg.ShutdownTimeout)
  }
}

func TestLoadOverrides(t *testing.T) {
  setValid(t)
  t.Setenv("PORT", "9000")
  t.Setenv("ENV", "production")
  t.Setenv("LOG_LEVEL", "error")
  t.Setenv("ALLOWED_ORIGINS", "a,b , c")
  cfg, err := Load()
  if err != nil {
    t.Fatalf("Load() = %v", err)
  }
  if cfg.Port != 9000 || cfg.Env != "production" || cfg.LogLevel != slog.LevelError {
    t.Errorf("got %+v", cfg)
  }
  want := []string{"a", "b", "c"}
  if len(cfg.AllowedOrigins) != 3 {
    t.Fatalf("AllowedOrigins = %v, want %v", cfg.AllowedOrigins, want)
  }
  for i := range want {
    if cfg.AllowedOrigins[i] != want[i] {
      t.Errorf("AllowedOrigins[%d] = %q, want %q", i, cfg.AllowedOrigins[i], want[i])
    }
  }
}

func TestLoadCollectsAllErrors(t *testing.T) {
  t.Setenv("PORT", "eighty")
  t.Setenv("DEBUG", "nope")
  _, err := Load()
  if err == nil {
    t.Fatal("Load() succeeded, want error")
  }
  for _, want := range []string{"DATABASE_URL", "JWT_SECRET", "PORT", "DEBUG"} {
    if !strings.Contains(err.Error(), want) {
      t.Errorf("error missing %q:\n%v", want, err)
    }
  }
}

func TestStringRedactsSecrets(t *testing.T) {
  setValid(t)
  t.Setenv("DATABASE_URL", "postgres://app:hunter2@db:5432/prod")
  cfg, err := Load()
  if err != nil {
    t.Fatal(err)
  }
  s := cfg.String()
  for _, secret := range []string{"hunter2", "0123456789abcdef"} {
    if strings.Contains(s, secret) {
      t.Errorf("String() leaked %q:\n%s", secret, s)
    }
  }
}
```

`go test -v` (the elapsed time varies run to run):

Output:

```
=== RUN   TestLoadDefaults
--- PASS: TestLoadDefaults (0.00s)
=== RUN   TestLoadOverrides
--- PASS: TestLoadOverrides (0.00s)
=== RUN   TestLoadCollectsAllErrors
--- PASS: TestLoadCollectsAllErrors (0.00s)
=== RUN   TestStringRedactsSecrets
--- PASS: TestStringRedactsSecrets (0.00s)
PASS
ok  	example.com/scratch	0.511s
```

`TestStringRedactsSecrets` is the one to copy into your own project. It is a four-line test
that permanently prevents a credential from reaching a log aggregator, and it keeps working as
people add fields — as long as you also assert on the whole `String()` output rather than
field by field.

`TestLoadCollectsAllErrors` asserts the *collecting* behaviour specifically: all four variable
names present in one error. That is the property that saves deploy cycles, so test it.

## Libraries

| Library | When it earns its place |
|---|---|
| `kelseyhightower/envconfig` | you want the tag loader above and nothing else; tiny, stable, unmaintained-but-done |
| `knadh/koanf` | several sources (env + YAML + consul) merged, with a small dependency tree |
| `spf13/viper` | you need live file watching and are already using cobra; otherwise its dependency weight is hard to justify |

For a service configured purely from the environment, `os.LookupEnv` plus 60 lines is the
right answer and has no dependencies at all. Reach for a library when you genuinely have
multiple config sources to merge.

## Quick Reference

| Function | Returns | Note |
|---|---|---|
| `os.Getenv(k)` | `string` | `""` for unset and for empty |
| `os.LookupEnv(k)` | `(string, bool)` | the only way to detect presence |
| `os.Setenv(k, v)` / `os.Unsetenv(k)` | `error` | process-global; prefer `t.Setenv` in tests |
| `os.Environ()` | `[]string` of `"K=V"` | useful for a debug dump — **redact before logging** |
| `os.ExpandEnv(s)` | `string` | substitutes `$VAR` in a template string |
| `strconv.Atoi` / `ParseInt` / `ParseFloat` | value, error | error text names function and input |
| `strconv.ParseBool` | `(bool, error)` | accepts only `1 t T TRUE true True` and the false forms |
| `time.ParseDuration` | `(Duration, error)` | unit is **required**: `30s` not `30` |
| `url.Parse` | `(*url.URL, error)` | very permissive; check `Scheme` and `Host` yourself |
| `(*slog.Level).UnmarshalText` | `error` | parses `debug`/`info`/`warn`/`error`, plus `warn+2` |
| `errors.Join(errs...)` | `error` | newline-separated; `Unwrap() []error` to get them back |
| `t.Setenv(k, v)` | — | auto-restored; makes the test non-parallel |

## Common Mistakes

**Using `os.Getenv` and treating `""` as unset.** `FEATURE_X=` then silently means "use the
default" instead of "off"; use `os.LookupEnv`.

**Returning on the first config error.** A five-variable misconfiguration takes five deploys
to fix; collect them all with `errors.Join`.

**Logging the whole config without a redacting `String()`.** Your database password ends up
in your log aggregator forever; redact in `String()` and test it.

**Validating lazily, on first use.** A bad `DATABASE_URL` becomes a user-facing 500 instead of
a failed deploy; validate everything in `Load()` and exit non-zero.

**`SHUTDOWN_TIMEOUT=30`.** `time.ParseDuration` requires a unit and rejects it; always write
`30s`.

**Branching on `cfg.Env` throughout the code.** Each branch is untested in the other
environment; make the difference a config *value* read by one code path.

**Reading a `.env` file in production.** It masks a missing orchestrator variable with a stale
committed default; load `.env` only when `ENV=development`.

## Exercises

1. Extend the `Config` above with `MAX_UPLOAD_BYTES` (accepting `10MB`-style suffixes),
   `RATE_LIMIT_RPS` as a float, and `FEATURE_FLAGS` as a `map[string]bool` from
   `a=true,b=false`. Keep the all-errors-at-once property and capture the real error text for
   three bad values of each.
2. Write the `Validate() error` method for the tag-driven loader so that the port range,
   secret length and env allowlist checks live outside `reflect`. Prove a struct can fail
   parsing and validation simultaneously and report both.
3. Make `redact` safe for a short secret: a 4-byte secret must not reveal its length in a way
   that helps an attacker. Decide what to print and write the test that enforces it.
4. Add a `-config FILE` flag that reads JSON, and wire it into the precedence chain so the
   order becomes flags > env > JSON file > defaults. Show all four levels winning for
   different keys in one run.
5. Write a `Config.Diff(other *Config) []string` that reports which fields changed, with
   secrets shown only as `changed`/`unchanged`. Use it to log a config change at startup
   against the previous deploy's values.

---

[← 18.1 Project Layout: cmd/ and internal/](01-project-layout.md) · [Course Index](../README.md) · [18.3 Authentication with JWT →](03-auth-jwt.md)
