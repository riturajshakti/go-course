[← 14.7 Examples as Tests and Coverage](07-examples-coverage.md) · [Course Index](../README.md) · [15.1 go build, run, install, and the Build Cache →](../15-tooling/01-build-run-install.md)

# 14.8 — Test Doubles with Interfaces

## What You'll Learn

- Define the interface where it is **consumed** and keep it small — then a hand-written double is trivial
- Stub, fake, spy, mock, function-field and interface-embedding doubles, all on one real service
- The stdlib doubles to reach for instead of writing your own: `httptest`, `fstest.MapFS`, `strings.Reader`, `iotest`
- Testing error paths — the single best reason doubles exist — and why over-mocking tests nothing

## Coming From Another Language

| Other | Go |
|---|---|
| `jest.mock('./module')` patches the module | no module patching; you pass a different value in |
| `unittest.mock.patch` monkeypatches an attribute | you define an interface and inject an implementation |
| `@InjectMocks` / a DI container | a constructor parameter or a struct field. No framework |
| `mockito`/`gomock` codegen as the default | hand-written fakes are the Go default; codegen is the exception |
| interfaces declared next to the implementation | interfaces declared next to the **consumer** |

That last row is the one that makes everything else easy, and it is the opposite of the Java/C#
habit. In Go the consumer declares the narrow interface it needs; the dependency satisfies it
implicitly ([5.1](../05-interfaces/01-implicit-satisfaction.md)). The database package does not know
it is being doubled, and your test does not need to know anything about the database.

## The principle: small interfaces at the consumer

Here is the service under test for this whole lesson. Note that `UserStore` and `Mailer` are
declared **in the package that uses them**, not in a storage or mail package, and that each has the
minimum number of methods.

```go
package signup

import (
  "context"
  "errors"
  "fmt"
  "strings"
  "time"
)

var (
  ErrNotFound = errors.New("signup: user not found")
  ErrTaken    = errors.New("signup: email already registered")
  ErrInvalid  = errors.New("signup: invalid email")
)

type User struct {
  ID       string
  Email    string
  Created  time.Time
  Welcomed bool
}

type UserStore interface {
  ByEmail(ctx context.Context, email string) (User, error)
  Create(ctx context.Context, u User) (User, error)
}

type Mailer interface {
  SendWelcome(ctx context.Context, to string) error
}

type Service struct {
  store  UserStore
  mailer Mailer
  now    func() time.Time
}

func NewService(store UserStore, mailer Mailer) *Service {
  return &Service{store: store, mailer: mailer, now: time.Now}
}

func (s *Service) Register(ctx context.Context, email string) (User, error) {
  email = strings.ToLower(strings.TrimSpace(email))
  if !strings.Contains(email, "@") || email == "" {
    return User{}, fmt.Errorf("register %q: %w", email, ErrInvalid)
  }

  switch _, err := s.store.ByEmail(ctx, email); {
  case err == nil:
    return User{}, fmt.Errorf("register %q: %w", email, ErrTaken)
  case !errors.Is(err, ErrNotFound):
    return User{}, fmt.Errorf("register %q: lookup: %w", email, err)
  }

  u, err := s.store.Create(ctx, User{Email: email, Created: s.now()})
  if err != nil {
    return User{}, fmt.Errorf("register %q: create: %w", email, err)
  }

  if err := s.mailer.SendWelcome(ctx, u.Email); err != nil {
    return u, fmt.Errorf("register %q: welcome mail: %w", email, err)
  }
  u.Welcomed = true
  return u, nil
}
```

Why the rule matters for testing: a 2-method `UserStore` needs a 2-method double. The same service
written against a 30-method `*sql.DB` or an all-queries `Repository` would need a 30-method double,
and you would reach for codegen out of desperation rather than judgement. **The size of your doubles
measures how well you drew the interface** ([5.5](../05-interfaces/05-composition.md)).

## 1. Stub: canned values

A stub returns fixed answers and records nothing. It is the right double when the test only cares
what the dependency *returns*.

```go
type stubStore struct {
  byEmailUser User
  byEmailErr  error
  createUser  User
  createErr   error
}

func (s stubStore) ByEmail(context.Context, string) (User, error) {
  return s.byEmailUser, s.byEmailErr
}

func (s stubStore) Create(_ context.Context, u User) (User, error) {
  if s.createErr != nil {
    return User{}, s.createErr
  }
  out := s.createUser
  out.Email = u.Email
  return out, nil
}

type stubMailer struct{ err error }

func (s stubMailer) SendWelcome(context.Context, string) error { return s.err }
```

A struct and four short methods, no library. Unnamed parameters say "this double ignores it".

Because the stub is a plain struct it drops straight into a table ([14.2](02-table-driven.md)), and
this is where doubles earn their keep: **every error path becomes a table row**.

```go
func TestRegisterWithStubs(t *testing.T) {
  tests := []struct {
    name    string
    email   string
    store   stubStore
    mailer  stubMailer
    wantErr error
  }{
    {
      name:  "success",
      email: "Ada@Example.com ",
      store: stubStore{byEmailErr: ErrNotFound, createUser: User{ID: "u1"}},
    },
    {
      name:    "already registered",
      email:   "ada@example.com",
      store:   stubStore{byEmailUser: User{ID: "u1"}},
      wantErr: ErrTaken,
    },
    {
      name:    "invalid email",
      email:   "nope",
      wantErr: ErrInvalid,
    },
    {
      name:    "lookup fails",
      email:   "ada@example.com",
      store:   stubStore{byEmailErr: errors.New("connection refused")},
      wantErr: nil,
    },
    {
      name:    "create fails",
      email:   "ada@example.com",
      store:   stubStore{byEmailErr: ErrNotFound, createErr: errors.New("disk full")},
      wantErr: nil,
    },
    {
      name:    "mail fails",
      email:   "ada@example.com",
      store:   stubStore{byEmailErr: ErrNotFound, createUser: User{ID: "u1"}},
      mailer:  stubMailer{err: errors.New("smtp timeout")},
      wantErr: nil,
    },
  }
  for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
      svc := NewService(tt.store, tt.mailer)
      u, err := svc.Register(context.Background(), tt.email)
      switch {
      case tt.wantErr != nil:
        if !errors.Is(err, tt.wantErr) {
          t.Fatalf("Register() error = %v, want %v", err, tt.wantErr)
        }
      case tt.name == "success":
        if err != nil {
          t.Fatalf("Register() error = %v, want nil", err)
        }
        if u.Email != "ada@example.com" {
          t.Errorf("Email = %q, want %q", u.Email, "ada@example.com")
        }
        if !u.Welcomed {
          t.Error("Welcomed = false, want true")
        }
      default:
        if err == nil {
          t.Fatal("Register() error = nil, want an error")
        }
        t.Logf("got expected error: %v", err)
      }
    })
  }
}
```

`go test -count=1 -v -run TestRegisterWithStubs .`. Output:

```
=== RUN   TestRegisterWithStubs
=== RUN   TestRegisterWithStubs/success
=== RUN   TestRegisterWithStubs/already_registered
=== RUN   TestRegisterWithStubs/invalid_email
=== RUN   TestRegisterWithStubs/lookup_fails
    stub_test.go:100: got expected error: register "ada@example.com": lookup: connection refused
=== RUN   TestRegisterWithStubs/create_fails
    stub_test.go:100: got expected error: register "ada@example.com": create: disk full
=== RUN   TestRegisterWithStubs/mail_fails
    stub_test.go:100: got expected error: register "ada@example.com": welcome mail: smtp timeout
--- PASS: TestRegisterWithStubs (0.00s)
    --- PASS: TestRegisterWithStubs/success (0.00s)
    --- PASS: TestRegisterWithStubs/already_registered (0.00s)
    --- PASS: TestRegisterWithStubs/invalid_email (0.00s)
    --- PASS: TestRegisterWithStubs/lookup_fails (0.00s)
    --- PASS: TestRegisterWithStubs/create_fails (0.00s)
    --- PASS: TestRegisterWithStubs/mail_fails (0.00s)
PASS
ok  	example.com/signup	0.497s
```

Read those three logged errors. `lookup: connection refused`, `create: disk full`,
`welcome mail: smtp timeout` — three distinct failure modes, each reached deliberately, each
confirming its wrapping context is correct. **You cannot test any of them against a real database
and a real SMTP server**, and that is the single best argument for doubles. Not speed: reachability.

Note the `t.Logf` is not decoration — asserting the error *message* shape is how you catch a `%w`
you forgot or a context string that says the wrong operation.

## 2. Fake: real behaviour, in memory

A fake is a working implementation with a shortcut — state in a map instead of a database. It is the
most useful kind of double, because the test exercises real sequences rather than one canned answer.

```go
type memStore struct {
  mu     sync.Mutex
  byID   map[string]User
  nextID int
}

func newMemStore(seed ...User) *memStore {
  m := &memStore{byID: make(map[string]User)}
  for _, u := range seed {
    m.nextID++
    if u.ID == "" {
      u.ID = fmt.Sprintf("u%d", m.nextID)
    }
    m.byID[u.ID] = u
  }
  return m
}

func (m *memStore) ByEmail(_ context.Context, email string) (User, error) {
  m.mu.Lock()
  defer m.mu.Unlock()
  for _, u := range m.byID {
    if u.Email == email {
      return u, nil
    }
  }
  return User{}, fmt.Errorf("by email %q: %w", email, ErrNotFound)
}

func (m *memStore) Create(_ context.Context, u User) (User, error) {
  m.mu.Lock()
  defer m.mu.Unlock()
  for _, existing := range m.byID {
    if existing.Email == u.Email {
      return User{}, fmt.Errorf("create %q: %w", u.Email, ErrTaken)
    }
  }
  m.nextID++
  u.ID = fmt.Sprintf("u%d", m.nextID)
  m.byID[u.ID] = u
  return u, nil
}

func (m *memStore) len() int {
  m.mu.Lock()
  defer m.mu.Unlock()
  return len(m.byID)
}
```

It enforces the real invariants: unique emails, server-assigned IDs, `ErrNotFound` wrapped as the
real store wraps it. The mutex makes it safe for parallel tests ([14.3](03-fixtures-helpers.md)) —
a detail people skip and then debug under `-race`.

```go
func TestRegisterWithFake(t *testing.T) {
  ctx := context.Background()
  store := newMemStore(User{Email: "existing@example.com"})
  svc := NewService(store, noopMailer{})

  u, err := svc.Register(ctx, "new@example.com")
  if err != nil {
    t.Fatalf("Register(new) error = %v, want nil", err)
  }
  if u.ID == "" {
    t.Error("Create did not assign an ID")
  }
  if store.len() != 2 {
    t.Errorf("store has %d users, want 2", store.len())
  }

  if _, err := svc.Register(ctx, "existing@example.com"); !errors.Is(err, ErrTaken) {
    t.Errorf("Register(existing) error = %v, want ErrTaken", err)
  }

  if _, err := svc.Register(ctx, "new@example.com"); !errors.Is(err, ErrTaken) {
    t.Errorf("re-registering error = %v, want ErrTaken", err)
  }
  if store.len() != 2 {
    t.Errorf("store has %d users after failures, want 2", store.len())
  }
  t.Logf("fake store ended with %d users", store.len())
}
```

Output:

```
=== RUN   TestRegisterWithFake
    fake_test.go:90: fake store ended with 2 users
--- PASS: TestRegisterWithFake (0.00s)
PASS
ok  	example.com/signup	0.466s
```

What the fake bought that a stub cannot: registering the same address **twice in one test** is
rejected the second time, and the store's size is checked after the failures to prove nothing was
half-written. A stub returning a fixed `ErrNotFound` would happily let you register the same user
forever.

**Idiomatic:** write the fake once per package, in a shared `_test.go` file (or an exported
`fooetest` package if other packages need it), and use it in most tests. Keep it honest — a fake
that accepts data the real store would reject makes your tests lie.

## 3. Spy: record the calls

A spy is a stub that remembers what it was asked to do, for when the *call* is the behaviour under
test. Sending exactly one welcome email is a requirement; the return value is not interesting.

```go
type spyMailer struct {
  sent []string
}

func (s *spyMailer) SendWelcome(_ context.Context, to string) error {
  s.sent = append(s.sent, to)
  return nil
}

func TestRegisterSpy(t *testing.T) {
  ctx := context.Background()
  store := newMemStore()
  spy := &spyMailer{}
  svc := NewService(store, spy)

  if _, err := svc.Register(ctx, "Ada@Example.com"); err != nil {
    t.Fatalf("Register() error = %v, want nil", err)
  }
  if _, err := svc.Register(ctx, "grace@example.com"); err != nil {
    t.Fatalf("Register() error = %v, want nil", err)
  }

  want := []string{"ada@example.com", "grace@example.com"}
  if len(spy.sent) != len(want) {
    t.Fatalf("sent %d mails, want %d: %q", len(spy.sent), len(want), spy.sent)
  }
  for i := range want {
    if spy.sent[i] != want[i] {
      t.Errorf("mail %d sent to %q, want %q", i, spy.sent[i], want[i])
    }
  }
  t.Logf("spy recorded: %q", spy.sent)
}

func TestRegisterNoMailOnInvalid(t *testing.T) {
  spy := &spyMailer{}
  svc := NewService(newMemStore(), spy)
  if _, err := svc.Register(context.Background(), "not-an-email"); err == nil {
    t.Fatal("Register() error = nil, want an error")
  }
  if len(spy.sent) != 0 {
    t.Errorf("sent %d mails on invalid input, want 0", len(spy.sent))
  }
}
```

Output:

```
=== RUN   TestRegisterSpy
    spy_test.go:39: spy recorded: ["ada@example.com" "grace@example.com"]
--- PASS: TestRegisterSpy (0.00s)
=== RUN   TestRegisterNoMailOnInvalid
--- PASS: TestRegisterNoMailOnInvalid (0.00s)
PASS
ok  	example.com/signup	0.503s
```

The spy proves the normalization reached the mailer (`Ada@Example.com` arrived as
`ada@example.com`), and the second test proves a **negative**: no mail on an invalid address.
Negatives are what spies are uniquely good at — you cannot observe "did not happen" from a return
value.

Note the pointer receiver and `&spyMailer{}`: a value receiver would append to a copy and the
recorded slice would stay empty. This is the most common bug in hand-written spies.

## 4. Hand-written mock: expectations

A mock asserts on the interaction itself, failing if the expected call does not happen. In Go you
write it with a `*testing.T` and `t.Cleanup`.

```go
type mockMailer struct {
  t        *testing.T
  wantTo   string
  calls    int
  maxCalls int
}

func newMockMailer(t *testing.T, wantTo string) *mockMailer {
  t.Helper()
  m := &mockMailer{t: t, wantTo: wantTo, maxCalls: 1}
  t.Cleanup(func() {
    t.Helper()
    if m.calls != m.maxCalls {
      t.Errorf("SendWelcome called %d times, want %d", m.calls, m.maxCalls)
    }
  })
  return m
}

func (m *mockMailer) SendWelcome(_ context.Context, to string) error {
  m.t.Helper()
  m.calls++
  if to != m.wantTo {
    m.t.Errorf("SendWelcome(to=%q), want %q", to, m.wantTo)
  }
  if m.calls > m.maxCalls {
    m.t.Errorf("SendWelcome called %d times, want at most %d", m.calls, m.maxCalls)
  }
  return nil
}

func TestRegisterWithMock(t *testing.T) {
  mock := newMockMailer(t, "ada@example.com")
  svc := NewService(newMemStore(), mock)
  if _, err := svc.Register(context.Background(), "ada@example.com"); err != nil {
    t.Fatalf("Register() error = %v, want nil", err)
  }
}
```

```
=== RUN   TestRegisterWithMock
--- PASS: TestRegisterWithMock (0.00s)
```

The `t.Cleanup` is the mock's defining feature: the unmet-expectation check runs after the test
body, so a call that never happened still fails. Proving it — the same mock with an input that
never reaches the mailer:

```go
func TestMockUnmetExpectation(t *testing.T) {
  mock := newMockMailer(t, "ada@example.com")
  svc := NewService(newMemStore(), mock)
  if _, err := svc.Register(context.Background(), "not-an-email"); err == nil {
    t.Fatal("expected an error")
  }
}
```

This test fails on purpose. Output:

```
=== RUN   TestMockUnmetExpectation
    mockfail_test.go:9: SendWelcome called 0 times, want 1
--- FAIL: TestMockUnmetExpectation (0.00s)
FAIL
FAIL	example.com/signup	0.497s
FAIL
```

The failure is reported at **line 9**, the `newMockMailer` call, because of the `t.Helper()` in the
cleanup ([14.3](03-fixtures-helpers.md)) — without it you would get a line inside the closure.

`testify/mock` ([14.4](04-testify.md)) and `go.uber.org/mock` (gomock) generate this machinery from
an interface, with argument matchers and call ordering. Generation is worth it for a wide interface
you do not own and cannot shrink — a vendor SDK with 40 methods — or to keep a mock in sync with a
churning interface. For a 1-2 method interface you drew yourself, the generated code is longer than
the hand-written double and fails at runtime instead of compile time.

## 5. Function fields: the lightest double

When the dependency is a single function, make it a struct field instead of an interface. `Service`
already does this for the clock:

```go
type Service struct {
  store  UserStore
  mailer Mailer
  now    func() time.Time
}
```

`NewService` sets `now: time.Now`; a test replaces it with a constant.

```go
func TestRegisterFixedClock(t *testing.T) {
  fixed := time.Date(2024, time.March, 1, 12, 0, 0, 0, time.UTC)
  store := newMemStore()
  svc := NewService(store, noopMailer{})
  svc.now = func() time.Time { return fixed }

  u, err := svc.Register(context.Background(), "ada@example.com")
  if err != nil {
    t.Fatalf("Register() error = %v, want nil", err)
  }
  if !u.Created.Equal(fixed) {
    t.Errorf("Created = %v, want %v", u.Created, fixed)
  }
  t.Logf("Created = %s", u.Created.Format(time.RFC3339))
}
```

Output:

```
=== RUN   TestRegisterFixedClock
    clock_test.go:22: Created = 2024-03-01T12:00:00Z
--- PASS: TestRegisterFixedClock (0.00s)
```

A deterministic timestamp with no `Clock` interface, no `fakeClock` type and no library — one field
and one closure. It generalizes to any single-function dependency:

```go
type Fetcher struct {
  fetch func(ctx context.Context, url string) ([]byte, error)
  sleep func(time.Duration)
  newID func() string
}
```

and in a test: `f.newID = func() string { return "fixed-id" }`. For time specifically, a function
field handles "what time is it"; controlling timers and tickers needs more
([13.5](../13-stdlib/05-timers.md)).

**Idiomatic:** function field for one function, interface for two or more related methods. The
field must be unexported with a constructor default, so production code cannot accidentally leave
it nil.

## 6. Interface embedding: stub only what you need

Embed the interface in your double and you satisfy it entirely, implementing only the methods the
test actually exercises. Anything else panics — loudly, which is the point.

```go
type readOnlyStore struct {
  UserStore
  user User
  err  error
}

func (s readOnlyStore) ByEmail(context.Context, string) (User, error) {
  return s.user, s.err
}

func TestEmbeddedStubOnlyImplementsByEmail(t *testing.T) {
  store := readOnlyStore{user: User{ID: "u1", Email: "ada@example.com"}}
  svc := NewService(store, noopMailer{})

  _, err := svc.Register(context.Background(), "ada@example.com")
  if !errors.Is(err, ErrTaken) {
    t.Fatalf("Register() error = %v, want ErrTaken", err)
  }
  t.Log("ByEmail was enough; Create was never reached")
}

func TestEmbeddedStubPanicsIfCreateCalled(t *testing.T) {
  store := readOnlyStore{err: ErrNotFound}
  svc := NewService(store, noopMailer{})

  defer func() {
    r := recover()
    if r == nil {
      t.Fatal("expected a nil-pointer panic from the embedded interface")
    }
    t.Logf("recovered: %v", r)
  }()
  _, _ = svc.Register(context.Background(), "new@example.com")
}
```

Output:

```
=== RUN   TestEmbeddedStubOnlyImplementsByEmail
    embed_test.go:27: ByEmail was enough; Create was never reached
--- PASS: TestEmbeddedStubOnlyImplementsByEmail (0.00s)
=== RUN   TestEmbeddedStubPanicsIfCreateCalled
    embed_test.go:39: recovered: runtime error: invalid memory address or nil pointer dereference
--- PASS: TestEmbeddedStubPanicsIfCreateCalled (0.00s)
PASS
ok  	example.com/signup	0.512s
```

The embedded `UserStore` field is nil, so calling the promoted `Create` dereferences nil and panics
with `invalid memory address or nil pointer dereference`. That is a *useful* failure: it tells you
the code under test called something you asserted it would not.

The trade-off, and it is real: embedding means **adding a method to the interface silently keeps
your double compiling**, and the test fails at runtime instead. The explicit stub in section 1
would have failed to compile, which is better. Use embedding for a wide third-party interface where
implementing everything is impractical; prefer explicit methods for your own narrow interfaces.

## Stdlib doubles: do not write your own

For the most commonly doubled dependencies the standard library already has the double, and it is
better than yours.

### httptest.NewServer — a real HTTP server

```go
func TestRateWithHTTPTest(t *testing.T) {
  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path != "/rates/EURUSD" {
      w.WriteHeader(http.StatusNotFound)
      return
    }
    w.Header().Set("Content-Type", "application/json")
    _, _ = w.Write([]byte(`{"pair":"EURUSD","value":1.0842}`))
  }))
  t.Cleanup(srv.Close)

  c := &Client{BaseURL: srv.URL, HTTP: srv.Client()}
  got, err := c.Rate("EURUSD")
  if err != nil {
    t.Fatalf("Rate() error = %v, want nil", err)
  }
  if got.Pair != "EURUSD" || got.Value != 1.0842 {
    t.Errorf("Rate() = %+v, want {EURUSD 1.0842}", got)
  }
}

func TestRateServerError(t *testing.T) {
  srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusInternalServerError)
  }))
  t.Cleanup(srv.Close)

  c := &Client{BaseURL: srv.URL, HTTP: srv.Client()}
  _, err := c.Rate("EURUSD")
  if err == nil {
    t.Fatal("Rate() error = nil, want an error")
  }
  t.Logf("got: %v", err)
}
```

Output:

```
=== RUN   TestRateWithHTTPTest
--- PASS: TestRateWithHTTPTest (0.00s)
=== RUN   TestRateServerError
    client_test.go:45: got: get rate EURUSD: unexpected status 500 Internal Server Error
--- PASS: TestRateServerError (0.00s)
```

A **real** server on a real loopback port, so you exercise your actual `http.Client`, real headers,
real status codes and real JSON decoding — no `RoundTripper` double that quietly differs from the
network. `srv.URL` carries a per-test random port, `srv.Client()` is preconfigured for it, and
`t.Cleanup(srv.Close)` rather than `defer` because of the parallel-subtest ordering rule.

### httptest.NewRecorder — doubling the ResponseWriter

For testing a handler, go the other way: fake the `http.ResponseWriter`.

```go
func TestHealthHandlerRecorder(t *testing.T) {
  req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
  rec := httptest.NewRecorder()

  HealthHandler(rec, req)

  res := rec.Result()
  if res.StatusCode != http.StatusOK {
    t.Errorf("status = %d, want %d", res.StatusCode, http.StatusOK)
  }
  if ct := res.Header.Get("Content-Type"); ct != "application/json" {
    t.Errorf("Content-Type = %q, want %q", ct, "application/json")
  }
  if got := rec.Body.String(); got != `{"status":"ok"}` {
    t.Errorf("body = %q, want %q", got, `{"status":"ok"}`)
  }
  t.Logf("status=%d body=%s", res.StatusCode, rec.Body.String())
}

func TestHealthHandlerWrongMethod(t *testing.T) {
  req := httptest.NewRequest(http.MethodPost, "/healthz", nil)
  rec := httptest.NewRecorder()
  HealthHandler(rec, req)
  if rec.Code != http.StatusMethodNotAllowed {
    t.Errorf("status = %d, want %d", rec.Code, http.StatusMethodNotAllowed)
  }
  t.Logf("status=%d body=%q", rec.Code, rec.Body.String())
}
```

Output:

```
=== RUN   TestHealthHandlerRecorder
    rec_test.go:25: status=200 body={"status":"ok"}
--- PASS: TestHealthHandlerRecorder (0.00s)
=== RUN   TestHealthHandlerWrongMethod
    rec_test.go:35: status=405 body="method not allowed\n"
--- PASS: TestHealthHandlerWrongMethod (0.00s)
```

No server, no port, no goroutine — just a call. `rec.Code`, `rec.Body` and `rec.Header()` are the
three things you assert on. Full treatment in [16.11](../16-http/11-httptest.md).

### fstest.MapFS — a filesystem in a map

```go
func TestLoadConfigNamesMapFS(t *testing.T) {
  fsys := fstest.MapFS{
    "conf.d/app.yaml":  {Data: []byte("name: app\n")},
    "conf.d/db.yaml":   {Data: []byte("name: db\n")},
    "conf.d/notes.txt": {Data: []byte("ignore me\n")},
    "other/thing.yaml": {Data: []byte("elsewhere\n")},
  }
  got, err := LoadConfigNames(fsys)
  if err != nil {
    t.Fatalf("LoadConfigNames() error = %v", err)
  }
  want := []string{"conf.d/app.yaml", "conf.d/db.yaml"}
  if len(got) != len(want) {
    t.Fatalf("got %v, want %v", got, want)
  }
  for i := range want {
    if got[i] != want[i] {
      t.Errorf("got[%d] = %q, want %q", i, got[i], want[i])
    }
  }
  t.Logf("matched %v", got)
}
```

Output:

```
=== RUN   TestLoadConfigNamesMapFS
    client_test.go:87: matched [conf.d/app.yaml conf.d/db.yaml]
--- PASS: TestLoadConfigNamesMapFS (0.00s)
```

No `t.TempDir`, no files written, and the fixture is visible in the test. The precondition is that
your function takes `fs.FS` rather than a path — which is the lesson:
**accept an interface and the double comes free** ([13.10](../13-stdlib/10-filepath-fs.md)).

### io doubles: strings.Reader, bytes.Buffer, iotest

For anything reading or writing, do not build a double at all.

```go
func TestCountLinesFromStringsReader(t *testing.T) {
  got, err := CountLines(strings.NewReader("a\nb\nc\n"))
  if err != nil {
    t.Fatalf("CountLines() error = %v", err)
  }
  if got != 3 {
    t.Errorf("CountLines() = %d, want 3", got)
  }
}

func TestCountLinesReadError(t *testing.T) {
  want := errors.New("disk on fire")
  _, err := CountLines(iotest.ErrReader(want))
  if !errors.Is(err, want) {
    t.Errorf("CountLines() error = %v, want %v", err, want)
  }
  t.Logf("got: %v", err)
}
```

Output:

```
=== RUN   TestCountLinesFromStringsReader
--- PASS: TestCountLinesFromStringsReader (0.00s)
=== RUN   TestCountLinesReadError
    client_test.go:64: got: count lines: disk on fire
--- PASS: TestCountLinesReadError (0.00s)
```

`iotest.ErrReader` reaches the `io.ReadAll` error branch that is otherwise almost impossible to
trigger — exactly the path coverage reports as red ([14.7](07-examples-coverage.md)).

| Need | Use |
|---|---|
| an `io.Reader` with content | `strings.NewReader` / `bytes.NewReader` |
| an `io.Writer` to inspect | `bytes.Buffer`, then `buf.String()` |
| a reader that fails | `iotest.ErrReader(err)` |
| a reader that times out mid-stream | `iotest.TimeoutReader(r)` |
| a reader returning one byte at a time | `iotest.OneByteReader(r)` |
| to verify a reader's contract | `iotest.TestReader(r, want)` |
| a filesystem | `fstest.MapFS`, and `fstest.TestFS` to verify one |
| an HTTP server | `httptest.NewServer` / `NewTLSServer` |
| an `http.ResponseWriter` | `httptest.NewRecorder` |
| an `*http.Request` | `httptest.NewRequest` |

## How to inject

| Mechanism | Verdict |
|---|---|
| Constructor parameter (`NewService(store, mailer)`) | **preferred** — dependencies are explicit and non-nil |
| Exported struct field | fine when optional with a sane default; nil-check or default it |
| Unexported field set by the test (same package) | fine for an internal test package; `svc.now = ...` above |
| Functional options (`NewService(WithMailer(m))`) | good for many optional dependencies ([4.8](../04-types-structs/08-constructors-options.md)) |
| Package-level `var` | **avoid** |

Why the package-level `var` is bad, concretely:

```go
var Now = time.Now

func (s *Service) stamp() time.Time { return Now() }
```

It looks like the easiest injection point and it costs you: the dependency is invisible in every
signature; two tests mutating it race and cannot be `t.Parallel`; forgetting to restore it leaks
into later tests (see `-shuffle` in [14.3](03-fixtures-helpers.md)); and nothing stops production
code reassigning it. If you inherit one, at least restore it with `t.Cleanup`:

```go
old := Now
Now = func() time.Time { return fixed }
t.Cleanup(func() { Now = old })
```

and plan to replace it with a field.

## Over-mocking: the antipattern

The failure mode is a test that asserts only that the doubles were called, and therefore asserts
nothing about your code. Concretely:

```go
func TestRegisterBad(t *testing.T) {
  store := &mockStore{}
  mailer := &mockMailer{}
  store.On("ByEmail", mock.Anything, "ada@example.com").Return(User{}, ErrNotFound)
  store.On("Create", mock.Anything, mock.Anything).Return(User{ID: "u1"}, nil)
  mailer.On("SendWelcome", mock.Anything, mock.Anything).Return(nil)

  svc := NewService(store, mailer)
  _, err := svc.Register(context.Background(), "ada@example.com")

  require.NoError(t, err)
  store.AssertExpectations(t)
  mailer.AssertExpectations(t)
}
```

What this test actually verifies: that `Register` calls `ByEmail`, then `Create`, then
`SendWelcome`. That is a restatement of the implementation, which means:

- It would still pass if `Register` lowercased nothing, so `Ada@Example.com` and
  `ada@example.com` register as two different users. **The real bug this test misses.**
- It would still pass if `Welcomed` were never set to `true`.
- It fails the moment you reorder the calls or add a cache lookup — a refactor with no behaviour
  change breaks the test. Tests that break on refactors and pass on bugs are worse than no tests.
- `mock.Anything` on the `Create` argument means the `User` being written is never checked at all.

The fix is to assert on **outcomes**: use the fake and check its final state (`store.len()`, the
stored user's email), check the returned `User`, and use a spy only for genuinely unobservable
effects such as "exactly one mail, to this address". That is what `TestRegisterWithFake` and
`TestRegisterSpy` above do.

Rules of thumb: double what you cannot control (network, clock, filesystem, third-party APIs) and
what you cannot otherwise reach (error paths). Do not double your own pure logic, value types, or
anything fast and deterministic — use a `Validator` or a `slices` helper for real.

## When to use the real dependency

Doubles test your code against your *belief* about the dependency. At some point you need to test
that belief. A fake `UserStore` cannot tell you that your SQL has a typo, that the unique index is
missing, or that the driver maps `NULL` the way you assumed.

The layered approach that works:

| Layer | Dependency | Speed | Catches |
|---|---|---|---|
| Unit tests | stubs and fakes | milliseconds | logic, error handling, edge cases |
| Repository tests | a real database | seconds | SQL, schema, constraints, type mapping |
| Integration tests | real services, build-tag gated | slower | wiring, config, migrations |

For the middle layer: in-process SQLite for simple SQL, `testcontainers-go` or a `docker compose`
Postgres for real fidelity (the right answer with Postgres-specific features), or a shared CI
database with per-test schemas. Gate it behind `//go:build integration`
([14.3](03-fixtures-helpers.md)) so the fast suite stays fast;
[17.8](../17-database/08-testing-db.md) covers it in full. The practical split: doubles for the many
logic tests, a few real-dependency tests for the queries. Not one or the other.

## Quick Reference

| Double | Returns | Records | Asserts | Use when |
|---|---|---|---|---|
| **Stub** | canned values | no | no | the return value is all that matters; error paths in a table |
| **Fake** | real behaviour | state | no | you need sequences, invariants, or the end state |
| **Spy** | canned values | calls | in the test | the call is the behaviour; proving something did *not* happen |
| **Mock** | canned values | calls | itself | a strict protocol you must not break; be sparing |
| **Real** | real behaviour | — | — | it is fast and deterministic, or the thing under test *is* the integration |

| Dependency | Reach for |
|---|---|
| HTTP server | `httptest.NewServer` |
| HTTP handler | `httptest.NewRecorder` + `NewRequest` |
| Filesystem | `fstest.MapFS`, or `t.TempDir` for real files |
| `io.Reader`/`Writer` | `strings.Reader`, `bytes.Buffer`, `iotest` |
| Clock | a `now func() time.Time` field |
| Database | a fake for logic, a real DB for the queries |
| Third-party SDK | a narrow interface you own, stubbed |

## Common Mistakes

**Declaring the interface next to the implementation.** Declare it where it is consumed, with only
the methods that consumer uses — then the double is three lines.

**A value receiver on a spy.** `func (s spyMailer) Send(...)` appends to a copy, the recording stays
empty, and the test passes vacuously. Use a pointer receiver and `&spyMailer{}`.

**A fake with no mutex used in parallel tests.** It is a data race on the map; `-race` will fail the
suite, usually in CI rather than locally.

**Asserting only that mocks were called.** The test restates the implementation: it breaks on
refactors and passes on real bugs like the missing lowercase above.

**Injecting through a package-level `var`.** Invisible dependency, cannot be parallel, leaks between
tests; use a constructor parameter or a struct field.

**`defer srv.Close()` with parallel subtests.** The parent returns first; use `t.Cleanup(srv.Close)`.

**Writing a `RoundTripper` double instead of using `httptest.NewServer`.** You lose real header,
status and connection behaviour for no gain.

**Never testing against the real database.** Every double encodes an assumption; something must
check the assumptions.

## Exercises

1. Take a function that calls `http.Get` directly and refactor it to accept a narrow interface or an
   `*http.Client`. Test it with `httptest.NewServer` for 200, 404, 500 and a malformed JSON body.
2. Write a fake for a 2-method store that enforces a unique constraint, and a test proving a
   duplicate insert fails *and* leaves the store unchanged. Then run it with `t.Parallel()` and
   `-race` and fix whatever that reveals.
3. Convert a spy-based test that asserts "SendWelcome was called" into one that asserts an outcome
   instead. State what behaviour change each version would catch and miss.
4. Give a service a `now func() time.Time` field and write a test that asserts an exact timestamp.
   Then do the same with a `Clock` interface and argue which you would keep for a single method.
5. Build a double by embedding a 5-method interface and implementing one method. Add a sixth method
   to the interface and show that the double still compiles. Then write the explicit version and
   show the compile error.
6. Use `iotest.ErrReader` and `iotest.TimeoutReader` to cover the error branches of a function that
   reads a stream. Report the coverage percentage before and after.
7. Write the same repository test twice: once against an in-memory fake, once against a real
   database behind `//go:build integration`. Introduce a SQL typo and report which one caught it.

---

[← 14.7 Examples as Tests and Coverage](07-examples-coverage.md) · [Course Index](../README.md) · [15.1 go build, run, install, and the Build Cache →](../15-tooling/01-build-run-install.md)
