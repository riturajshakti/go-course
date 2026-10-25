[← 4.7 Struct Tags](07-struct-tags.md) · [Course Index](../README.md) · [5.1 Implicit Satisfaction →](../05-interfaces/01-implicit-satisfaction.md)

# 4.8 — Constructors and the Functional Options Pattern

## What You'll Learn

- Go has no constructors; `NewT` is a naming convention and nothing more
- A useful zero value beats any constructor — `bytes.Buffer`, `sync.Mutex`, and `strings.Builder` need none
- The functional options pattern (`...Option`) is how you get optional, validated, extensible parameters
- `MustX` panicking variants are acceptable only at package initialisation

## Coming From Another Language

| Other | Go |
|---|---|
| `new Foo(...)` / `__init__` | a plain function, `NewFoo`, returning the value |
| Python keyword args with defaults | there are none; options structs or `...Option` |
| TS optional params `(a, b?)` | no optional params and no overloads |
| Rust `Default::default()` | the zero value, free and automatic |
| Rust builder (`.timeout().build()`) | functional options; builders exist but are not idiomatic |
| Exceptions from a constructor | `(*T, error)` |

Go has no constructors, no overloads, no default arguments. One function, one signature.
That constraint is what makes this lesson a design lesson rather than a syntax one.

## The NewT convention

```go
package main

import (
  "errors"
  "fmt"
)

type Point struct{ X, Y int }

func NewPoint(x, y int) Point { return Point{X: x, Y: y} }

type Cache struct {
  max   int
  items map[string]string
}

func NewCache(max int) *Cache {
  return &Cache{max: max, items: make(map[string]string, max)}
}

type Account struct {
  Email   string
  Balance int
}

func NewAccount(email string, balance int) (*Account, error) {
  if email == "" {
    return nil, errors.New("account: email is required")
  }
  if balance < 0 {
    return nil, fmt.Errorf("account: balance %d must not be negative", balance)
  }
  return &Account{Email: email, Balance: balance}, nil
}

func main() {
  fmt.Println(NewPoint(1, 2))
  c := NewCache(8)
  c.items["k"] = "v"
  fmt.Println(c.max, c.items)

  a, err := NewAccount("a@example.com", 100)
  fmt.Println(a, err)

  _, err = NewAccount("", 100)
  fmt.Println(err)
  _, err = NewAccount("a@example.com", -5)
  fmt.Println(err)
}
```

Output:

```
{1 2}
8 map[k:v]
&{a@example.com 100} <nil>
account: email is required
account: balance -5 must not be negative
```

Naming: `NewT` for a package with several types, and `New` alone when the package exists for
one type — `list.New()`, not `list.NewList()`. Nothing forces this; nothing enforces that
you call it either, which is why a type with mandatory invariants should keep its fields
unexported.

Which to return:

| Return | Use when |
|---|---|
| `T` | small immutable value, no invariants to protect (`NewPoint`) |
| `*T` | the value has identity, is mutated, is large, or contains a mutex |
| `(T, error)` / `(*T, error)` | construction can fail: validation, parsing, dialing, opening |
| `*T` with no error | nothing can fail; do not add an always-nil `error` |

`NewCache` must be a constructor because the zero `Cache` has a nil map, and writing to a
nil map panics. That is the real question for every type: is my zero value usable?

## The useful zero value

The best constructor is none. Several of the most-used standard library types are designed
so that `var x T` is immediately valid.

```go
package main

import (
  "bytes"
  "fmt"
  "log/slog"
  "os"
  "strings"
  "sync"
)

func main() {
  var buf bytes.Buffer
  buf.WriteString("usable with no constructor")
  fmt.Println(buf.String(), buf.Len())

  var mu sync.Mutex
  mu.Lock()
  mu.Unlock()
  fmt.Println("mutex ok")

  var wg sync.WaitGroup
  wg.Add(1)
  go func() { defer wg.Done() }()
  wg.Wait()
  fmt.Println("waitgroup ok")

  var sb strings.Builder
  sb.WriteString("builder")
  fmt.Println(sb.String())

  h := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{})
  _ = h
  fmt.Println("handler opts zero value accepted")
}
```

Output:

```
usable with no constructor 26
mutex ok
waitgroup ok
builder
handler opts zero value accepted
```

`bytes.Buffer` grows its slice lazily; `sync.Mutex` starts unlocked; `strings.Builder` starts
empty. `slog.HandlerOptions{}` means "all defaults", which is why `slog.NewTextHandler`
accepts a nil or empty options pointer.

Design for this when you can: lazily initialise maps in the method that writes to them,
treat `0` as "use the default" at the point of use, and your type needs no constructor at
all. When you cannot — a required address, a connection to dial, a map that several methods
read — write the constructor and keep the fields unexported.

## The config-struct parameter

The first answer to "too many parameters": one struct, keyed at the call site.

```go
package main

import (
  "fmt"
  "time"
)

type ClientConfig struct {
  BaseURL string
  Timeout time.Duration
  Retries int
}

type Client struct {
  cfg ClientConfig
}

func NewClient(cfg ClientConfig) (*Client, error) {
  if cfg.BaseURL == "" {
    return nil, fmt.Errorf("client: BaseURL is required")
  }
  if cfg.Timeout == 0 {
    cfg.Timeout = 5 * time.Second
  }
  if cfg.Retries == 0 {
    cfg.Retries = 3
  }
  return &Client{cfg: cfg}, nil
}

func main() {
  c, err := NewClient(ClientConfig{BaseURL: "https://api.example.com"})
  fmt.Println(c.cfg, err)

  c2, _ := NewClient(ClientConfig{BaseURL: "https://api.example.com", Retries: 1, Timeout: time.Second})
  fmt.Println(c2.cfg)

  _, err = NewClient(ClientConfig{})
  fmt.Println(err)
}
```

Output:

```
{https://api.example.com 5s 3} <nil>
{https://api.example.com 1s 1}
client: BaseURL is required
```

Cheap, readable, and adding a field is backward compatible. The flaw is in the
default-filling: there is no way to request `Retries: 0`, because zero is indistinguishable
from "not set" — the constructor would quietly turn it into 3. Expressing it needs `*int`, a
separate `RetriesSet bool`, or a sentinel like `-1`, all of them ugly. `http.Server` and
`tls.Config` use this pattern and live with it.

## Functional options

An option is a function that mutates the value being built. Defaults are applied first, then
each option in order, and an option can validate and return an error.

```go
package main

import (
  "fmt"
  "io"
  "log/slog"
  "os"
  "time"
)

type Server struct {
  addr     string
  timeout  time.Duration
  retries  int
  logger   *slog.Logger
  tlsOn    bool
  maxConns int
}

type Option func(*Server) error

func WithTimeout(d time.Duration) Option {
  return func(s *Server) error {
    if d <= 0 {
      return fmt.Errorf("server: timeout %v must be positive", d)
    }
    s.timeout = d
    return nil
  }
}

func WithRetries(n int) Option {
  return func(s *Server) error {
    if n < 0 {
      return fmt.Errorf("server: retries %d must not be negative", n)
    }
    s.retries = n
    return nil
  }
}

func WithLogger(l *slog.Logger) Option {
  return func(s *Server) error {
    if l == nil {
      return fmt.Errorf("server: logger must not be nil")
    }
    s.logger = l
    return nil
  }
}

func WithTLS() Option {
  return func(s *Server) error {
    s.tlsOn = true
    return nil
  }
}

func WithMaxConns(n int) Option {
  return func(s *Server) error {
    if n < 1 {
      return fmt.Errorf("server: maxConns %d must be at least 1", n)
    }
    s.maxConns = n
    return nil
  }
}

func NewServer(addr string, opts ...Option) (*Server, error) {
  if addr == "" {
    return nil, fmt.Errorf("server: addr is required")
  }
  s := &Server{
    addr:     addr,
    timeout:  30 * time.Second,
    retries:  3,
    maxConns: 100,
    logger:   slog.New(slog.NewTextHandler(io.Discard, nil)),
  }
  for _, opt := range opts {
    if err := opt(s); err != nil {
      return nil, err
    }
  }
  return s, nil
}

func (s *Server) describe() string {
  return fmt.Sprintf("addr=%s timeout=%v retries=%d tls=%t maxConns=%d",
    s.addr, s.timeout, s.retries, s.tlsOn, s.maxConns)
}

func main() {
  s, err := NewServer(":8080")
  fmt.Println(s.describe(), err)

  s2, err := NewServer(":9090",
    WithTimeout(2*time.Second),
    WithRetries(0),
    WithTLS(),
    WithMaxConns(10),
    WithLogger(slog.New(slog.NewTextHandler(os.Stdout, nil))),
  )
  fmt.Println(s2.describe(), err)

  _, err = NewServer(":1", WithTimeout(-1))
  fmt.Println(err)

  _, err = NewServer(":1", WithMaxConns(0))
  fmt.Println(err)

  _, err = NewServer("")
  fmt.Println(err)
}
```

Output:

```
addr=:8080 timeout=30s retries=3 tls=false maxConns=100 <nil>
addr=:9090 timeout=2s retries=0 tls=true maxConns=10 <nil>
server: timeout -1ns must be positive
server: maxConns 0 must be at least 1
server: addr is required
```

The pieces, each of which is load-bearing:

Required arguments stay positional (`addr`). Options are only for the optional.

Defaults are set on the struct *before* the loop, so an absent option means "default" and a
present one always wins. Note `WithRetries(0)` produced `retries=0` — the zero value is
expressible here, which is exactly what the config struct could not do.

Options are applied in order, so a later option overrides an earlier one, and a caller can
hold `[]Option` and append to it:

```go
base := []Option{WithTLS(), WithRetries(5)}
prod := append(base, WithTimeout(time.Minute))
s, err := NewServer(":443", prod...)
```

The constructor returns `(*Server, error)` and stops at the first failing option. The
fields stay unexported, so a `Server` cannot be built around the validation.

Naming: `With*` for setters, `Option` for the type. Keep the type exported so callers in
other packages can write their own options against your exported setters.

## The simpler func(*T) form

If no option can fail, drop the error and the pattern becomes much lighter.

```go
package main

import (
  "fmt"
  "time"
)

type Dialer struct {
  timeout   time.Duration
  keepAlive bool
}

type DialOption func(*Dialer)

func DialTimeout(d time.Duration) DialOption {
  return func(dl *Dialer) { dl.timeout = d }
}

func DialKeepAlive() DialOption {
  return func(dl *Dialer) { dl.keepAlive = true }
}

func NewDialer(opts ...DialOption) *Dialer {
  d := &Dialer{timeout: 10 * time.Second}
  for _, o := range opts {
    o(d)
  }
  return d
}

func main() {
  fmt.Printf("%+v\n", *NewDialer())
  fmt.Printf("%+v\n", *NewDialer(DialTimeout(time.Second), DialKeepAlive()))

  base := []DialOption{DialKeepAlive()}
  prod := append(base, DialTimeout(2*time.Second))
  fmt.Printf("%+v\n", *NewDialer(prod...))
}
```

Output:

```
{timeout:10000000000 keepAlive:false}
{timeout:1000000000 keepAlive:true}
{timeout:2000000000 keepAlive:true}
```

`timeout` prints as a raw nanosecond count rather than `10s` because `fmt` cannot call the
`String` method of an *unexported* field through reflection. Export the field, or print
`d.timeout` directly, and you get `10s`.

Idiomatic: use `func(*T)` when nothing can fail, `func(*T) error` when something can.
`grpc.Dial`, `slog`, and the AWS SDK all use the first; `pgxpool` and most database drivers
the second.

## Comparing the three

| Approach | Optional params | Zero value expressible | Backward compatible | Call-site cost | Discoverability |
|---|---|---|---|---|---|
| Positional params | no — new param breaks callers | yes | no | lowest | signature says it all |
| Config struct | yes, keyed literal | **no** — 0 means "unset" | yes, add a field | low | one struct to read |
| Functional options | yes | yes | yes, add a `With*` | highest — one closure each | must browse `With*` funcs |

The tradeoff in one line: options buy you expressible zero values, per-option validation,
and third-party extensibility, at the cost of a closure allocation per option and an API
surface spread over a dozen functions.

Rule of thumb: fewer than four optional settings, or a type that is pure configuration,
take the config struct. A long-lived type with growing optional behaviour, especially a
library others extend, takes options. A type with no optional settings takes neither.

## Must variants

A `MustX` function panics instead of returning an error, for use where a failure is a
programming bug that must stop the process.

```go
package main

import (
  "fmt"
  "regexp"
)

type Pattern struct {
  re *regexp.Regexp
}

func NewPattern(expr string) (*Pattern, error) {
  re, err := regexp.Compile(expr)
  if err != nil {
    return nil, fmt.Errorf("pattern %q: %w", expr, err)
  }
  return &Pattern{re: re}, nil
}

func MustPattern(expr string) *Pattern {
  p, err := NewPattern(expr)
  if err != nil {
    panic(err)
  }
  return p
}

var digits = MustPattern(`\d+`)

func main() {
  fmt.Println(digits.re.FindString("abc123"))

  _, err := NewPattern(`[`)
  fmt.Println(err)

  defer func() { fmt.Println("recovered:", recover()) }()
  MustPattern(`[`)
}
```

Output:

```
123
pattern "[": error parsing regexp: missing closing ]: `[`
recovered: pattern "[": error parsing regexp: missing closing ]: `[`
```

Acceptable: package-level `var` initialisation (as `digits` above) and test setup, where the
input is a literal in the source and a failure means the binary is broken. Not acceptable:
anywhere the input comes from a user, a file, a flag, or the network. Always ship the
error-returning version alongside, as the standard library does with `regexp.Compile` and
`regexp.MustCompile`, `template.Parse` and `template.Must`.

## The builder, briefly

A chained builder (`New().Timeout(d).Retries(3).Build()`) is possible but rare in Go — it
needs a second type, defers all errors to `Build`, and gains nothing over options; reach for
it only when porting an API that already looks like that.

## Quick Reference

| Form | Signature |
|---|---|
| plain constructor | `func NewT(required ...) *T` |
| failing constructor | `func NewT(required ...) (*T, error)` |
| must variant | `func MustT(required ...) *T` (panics) |
| config struct | `func NewT(cfg Config) (*T, error)` |
| options, infallible | `type Option func(*T)` + `func NewT(req string, opts ...Option) *T` |
| options, fallible | `type Option func(*T) error` + `func NewT(req string, opts ...Option) (*T, error)` |
| option constructor | `func WithX(v V) Option { return func(t *T) error { ... } }` |

| Checklist for a new type | |
|---|---|
| Is the zero value usable? | then do not write a constructor |
| Can construction fail? | return `(T, error)`; never an always-nil error |
| Are there invariants? | unexport the fields so the constructor cannot be bypassed |
| More than ~3 optional settings? | functional options |
| Does the type contain a mutex or a map? | return `*T`, and initialise the map |

## Common Mistakes

**Writing a constructor for a type whose zero value already works.** It forces every caller
through your function for nothing; design for `var x T` instead.

**Returning an `error` that is always nil.** Every caller now has a pointless `if err != nil`;
drop it until something can actually fail.

**Using a config struct where `0` is a meaningful value.** You cannot distinguish "set to
zero" from "unset"; use functional options or a `*int`.

**Applying options before the defaults.** The defaults then overwrite the caller's choices;
set defaults first, then loop.

**Using `MustX` on runtime input.** It panics the process on bad data; reserve it for
package-level variables and tests.

## Exercises

1. Write `NewRateLimiter(rps int, opts ...Option)` with `WithBurst`, `WithClock`, and
   `WithJitter`. Make `WithBurst(0)` a validation error and prove the zero value of `jitter`
   is still reachable through an option.
2. Take the `ClientConfig` version and the options version of the same type, then add a
   `WithProxy` setting to both. Count the lines changed and the callers broken in each.
3. Design a type whose zero value is fully usable with a lazily initialised map and a `0`
   that means "default size". Write a test that uses it via `var c Cache` only.
4. Implement `Option` as an interface with an `apply(*Server) error` method instead of a
   function type. Note what this buys (named options, introspection, comparability) and
   what it costs.
5. Write `MustLoadConfig(path string) *Config` and `LoadConfig(path string) (*Config, error)`.
   Argue in two sentences which one `main` should call and which one a library must expose.

---

[← 4.7 Struct Tags](07-struct-tags.md) · [Course Index](../README.md) · [5.1 Implicit Satisfaction →](../05-interfaces/01-implicit-satisfaction.md)
