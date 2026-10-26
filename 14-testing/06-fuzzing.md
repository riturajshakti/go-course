[← 14.5 Benchmarks](05-benchmarks.md) · [Course Index](../README.md) · [14.7 Examples as Tests and Coverage →](07-examples-coverage.md)

# 14.6 — Fuzzing

## What You'll Learn

- `func FuzzXxx(f *testing.F)`, `f.Add` seeds, `f.Fuzz` targets, and the 15 argument types allowed
- Without `-fuzz` a fuzz target is an ordinary table test over its seed corpus; with `-fuzz` it generates inputs
- The real cycle: a fuzzer finding a genuine bug in 0.02s, writing the crasher to `testdata`, and that file becoming a regression test
- Round-trip and invariant properties, the two corpus locations, `-fuzztime`, and what fuzzing is bad at

## Coming From Another Language

| Other | Go |
|---|---|
| `hypothesis` (Python), `fast-check` (JS) | built into `go test`; coverage-guided, not type-driven generation |
| `cargo fuzz` / libFuzzer | the same coverage-guided idea, in the standard toolchain |
| property-based testing frameworks | no generators or shrinkers to write — but only 15 scalar types |
| AFL, with a separate harness binary | `go test -fuzz` on your normal test package |
| Jest snapshot of random input | failing inputs are *saved as files* and replay forever |

Two things make Go's fuzzing different from classic property testing. It is **coverage-guided**:
the engine watches which branches each input reaches and mutates the inputs that found new code,
so it does not sample blindly. And a failure is **persisted** — the input is written into
`testdata/`, where it becomes a permanent regression case you commit alongside the fix.

## The shape

A string reverser with a bug, and a fuzz target asserting two properties.

```go
package rev

func Reverse(s string) string {
  b := []byte(s)
  for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
    b[i], b[j] = b[j], b[i]
  }
  return string(b)
}
```

```go
package rev

import (
  "testing"
  "unicode/utf8"
)

func FuzzReverse(f *testing.F) {
  f.Add("hello")
  f.Add("")
  f.Add("a")
  f.Add("Go!")
  f.Fuzz(func(t *testing.T, in string) {
    once := Reverse(in)
    twice := Reverse(once)
    if in != twice {
      t.Errorf("Reverse(Reverse(%q)) = %q, want %q", in, twice, in)
    }
    if utf8.ValidString(in) && !utf8.ValidString(once) {
      t.Errorf("Reverse(%q) = %q, which is not valid UTF-8", in, once)
    }
  })
}
```

The pieces:

- `func FuzzXxx(f *testing.F)` — one fuzz target per function, named like a test.
- `f.Add(args...)` — a **seed corpus** entry. The argument types and count must match `f.Fuzz`'s
  function exactly.
- `f.Fuzz(func(t *testing.T, ...))` — the target. First parameter is always `*testing.T`; the rest
  are the fuzzed inputs. Called exactly once per fuzz target.
- Inside the target you get a `*testing.T`, not `*testing.F`. `f.Log`, `f.Error` and `f.Skip` are
  forbidden there; use the `t` methods.

Note what the assertions are: not "reversing 'hello' gives 'olleh'" but **properties that must hold
for every input**. Fuzzing cannot know the right answer for a generated string, so you assert
relationships instead.

## Without -fuzz it is a normal test

This surprises people. A fuzz target runs under plain `go test` — it just runs the seed corpus and
nothing else.

`go test -count=1 -v -run FuzzReverse .`. Output:

```
=== RUN   FuzzReverse
=== RUN   FuzzReverse/seed#0
=== RUN   FuzzReverse/seed#1
=== RUN   FuzzReverse/seed#2
=== RUN   FuzzReverse/seed#3
--- PASS: FuzzReverse (0.00s)
    --- PASS: FuzzReverse/seed#0 (0.00s)
    --- PASS: FuzzReverse/seed#1 (0.00s)
    --- PASS: FuzzReverse/seed#2 (0.00s)
    --- PASS: FuzzReverse/seed#3 (0.00s)
PASS
ok  	example.com/scratch	0.484s
```

Each `f.Add` became a subtest named `seed#N`. So a fuzz target is a table-driven test whose table
happens to be the seed corpus — it costs milliseconds in CI and it is where your regression cases
live. **This mode is the one that runs on every `go test ./...`.** Generation only happens with
`-fuzz`, which also requires exactly one matching target in exactly one package.

## Finding a real bug

`go test -count=1 -run FuzzReverse -fuzz FuzzReverse -fuzztime 60s .`. It did not need 60 seconds.
A real captured run — the *specific* input it lands on and therefore the corpus file's hash differ
every time (a re-run found U+0336 COMBINING LONG STROKE OVERLAY instead), but it is always a
multi-byte rune and it is always found in well under a second:

```
fuzz: elapsed: 0s, gathering baseline coverage: 0/4 completed
fuzz: elapsed: 0s, gathering baseline coverage: 4/4 completed, now fuzzing with 12 workers
fuzz: elapsed: 0s, execs: 752 (42014/sec), new interesting: 4 (total: 8)
--- FAIL: FuzzReverse (0.02s)
    --- FAIL: FuzzReverse (0.00s)
        rev_test.go:20: Reverse("։") = "\x89\xd6", which is not valid UTF-8
    
    Failing input written to testdata/fuzz/FuzzReverse/1f675d245585202e
    To re-run:
    go test -run=FuzzReverse/1f675d245585202e
FAIL
exit status 1
FAIL	example.com/scratch	0.529s
```

**752 executions and 0.02 seconds** to find a bug that four hand-written seed cases missed. The
input is U+0589 (ARMENIAN FULL STOP), encoded as the two bytes `d6 89`. Reversing *bytes* produces
`89 d6`, which is not valid UTF-8 — the classic byte-vs-rune bug, and the kind of input no one
thinks to write by hand.

Read the progress lines: `gathering baseline coverage` runs the existing corpus to learn which
branches it reaches; `new interesting: 4` counts inputs that found new coverage; `12 workers` is
one fuzzing process per CPU.

The crasher is now a file:

```
testdata/fuzz/FuzzReverse/1f675d245585202e
```

```
go test fuzz v1
string("։")
```

The format is a version line followed by one typed literal per argument. **Commit this file.** It is
the regression test for the bug you are about to fix.

## The saved input is a regression test

Re-run with no `-fuzz` at all. `go test -count=1 -v -run FuzzReverse .`. Output:

```
=== RUN   FuzzReverse
=== RUN   FuzzReverse/seed#0
=== RUN   FuzzReverse/seed#1
=== RUN   FuzzReverse/seed#2
=== RUN   FuzzReverse/seed#3
=== RUN   FuzzReverse/1f675d245585202e
    rev_test.go:20: Reverse("։") = "\x89\xd6", which is not valid UTF-8
--- FAIL: FuzzReverse (0.00s)
    --- PASS: FuzzReverse/seed#0 (0.00s)
    --- PASS: FuzzReverse/seed#1 (0.00s)
    --- PASS: FuzzReverse/seed#2 (0.00s)
    --- PASS: FuzzReverse/seed#3 (0.00s)
    --- FAIL: FuzzReverse/1f675d245585202e (0.00s)
FAIL
FAIL	example.com/scratch	0.211s
FAIL
```

The crasher became subtest `FuzzReverse/1f675d245585202e`, and it fails in the ordinary test run —
in CI, on every commit, forever, with no fuzzing budget. That is the payoff: fuzzing is how you
*find* the input, and `testdata` is how you *keep* it.

## The fix, and what fuzzing found next

The obvious fix is to reverse runes rather than bytes:

```go
func Reverse(s string) string {
  r := []rune(s)
  for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
    r[i], r[j] = r[j], r[i]
  }
  return string(r)
}
```

The saved crasher now passes. Fuzz again and the engine immediately finds a *second*, different
bug:

```
fuzz: minimizing 31-byte failing input file
fuzz: elapsed: 0s, gathering baseline coverage: 6/9 completed
--- FAIL: FuzzReverse (0.03s)
    --- FAIL: FuzzReverse (0.00s)
        rev_test.go:17: Reverse(Reverse("\xcb")) = "�", want "\xcb"
    
    Failing input written to testdata/fuzz/FuzzReverse/5a1ad6f2992f1d83
    To re-run:
    go test -run=FuzzReverse/5a1ad6f2992f1d83
FAIL
exit status 1
```

`[]rune(s)` replaces each invalid byte with U+FFFD, so the round-trip is destroyed for input that
is not valid UTF-8. Note `fuzz: minimizing 31-byte failing input file` on the first line: the engine
found a 31-byte failure and shrank it to the single byte `\xcb` before saving. Minimization is
automatic, bounded by `-fuzzminimizetime` (default 60s).

Fix that by preserving invalid bytes:

```go
package rev

import "unicode/utf8"

func Reverse(s string) string {
  out := make([]byte, 0, len(s))
  for i := len(s); i > 0; {
    r, size := utf8.DecodeLastRuneInString(s[:i])
    if r == utf8.RuneError && size <= 1 {
      out = append(out, s[i-1])
      i--
      continue
    }
    out = append(out, s[i-size:i]...)
    i -= size
  }
  return string(out)
}
```

And fuzzing finds a **third** failure — this time telling you the *test* is wrong, not the code:

```
--- FAIL: FuzzReverse (0.03s)
    --- FAIL: FuzzReverse (0.00s)
        rev_test.go:17: Reverse(Reverse("\x92\xc3")) = "Ò", want "\x92\xc3"
```

Reversing the bytes `92 c3` yields `c3 92`, which *is* a valid two-byte encoding (U+00D2, `Ò`), so
the second reverse treats it as one rune and cannot restore the original. No implementation can
round-trip arbitrary byte strings under a rune-reversing definition. The honest fix is to scope the
property to valid input:

```go
func FuzzReverse(f *testing.F) {
  f.Add("hello")
  f.Add("")
  f.Add("a")
  f.Add("Go!")
  f.Fuzz(func(t *testing.T, in string) {
    if !utf8.ValidString(in) {
      t.Skip("not valid UTF-8")
    }
    once := Reverse(in)
    if !utf8.ValidString(once) {
      t.Errorf("Reverse(%q) = %q, which is not valid UTF-8", in, once)
    }
    if twice := Reverse(once); in != twice {
      t.Errorf("Reverse(Reverse(%q)) = %q, want %q", in, twice, in)
    }
    if n, m := utf8.RuneCountInString(in), utf8.RuneCountInString(once); n != m {
      t.Errorf("Reverse(%q) has %d runes, input has %d", in, m, n)
    }
  })
}
```

`t.Skip` inside the target tells the engine this input is not interesting. 40 seconds of fuzzing:

```
fuzz: elapsed: 36s, execs: 12703052 (2362634/sec), new interesting: 38 (total: 51)
fuzz: elapsed: 39s, execs: 15069688 (788933/sec), new interesting: 47 (total: 60)
fuzz: elapsed: 41s, execs: 15124065 (26711/sec), new interesting: 47 (total: 60)
PASS
ok  	example.com/scratch	41.551s
```

15 million executions, clean. Three findings from one target: two implementation bugs and one wrong
assumption in the test. That last one is typical and valuable — fuzzing forces you to state the
precondition you did not know you were relying on.

## Supported argument types

Only these 15, and the compiler enforces it:

`[]byte`, `string`, `bool`, `byte`, `rune`, `float32`, `float64`, `int`, `int8`, `int16`, `int32`,
`int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64` — `byte`/`uint8` and `rune`/`int32` being
aliases is why that reads as 15 distinct types.

**Structs are not supported.** Trying it:

```go
func FuzzStruct(f *testing.F) {
  f.Add(Pair{"a", "b"})
  f.Fuzz(func(t *testing.T, in Pair) {
    _ = Norm(in)
  })
}
```

This does not compile. Output:

```
# example.com/scratch
./p_test.go:7:32: fuzzing arguments can only have the following types: string, bool, float32, float64, int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64, []byte
```

A compile error rather than a runtime surprise. The workaround is to fuzz the *scalars* and build
the struct inside the target:

```go
f.Fuzz(func(t *testing.T, key, value string, retries int) {
  cfg := Config{Key: key, Value: value, Retries: retries}
  ...
})
```

Or, for a genuinely complex input, fuzz `[]byte` and derive the structure from it — feed the bytes
to your real decoder, or use them as a deterministic source of pseudo-random choices:

```go
f.Fuzz(func(t *testing.T, data []byte) {
  var cfg Config
  if err := json.Unmarshal(data, &cfg); err != nil {
    t.Skip()
  }
  ...
})
```

That shape is also the standard way to fuzz a parser: most inputs are rejected and skipped, and the
coverage guidance quickly learns to produce inputs that get past the decoder.

## The two corpus locations

| Location | Checked in? | When it runs | Written by |
|---|---|---|---|
| `f.Add(...)` in the source | yes (it is code) | always | you |
| `testdata/fuzz/FuzzName/` | **yes, commit it** | always, as subtests | the engine, on failure |
| `$GOCACHE/fuzz/<pkg>/FuzzName/` | no | only under `-fuzz` | the engine, continuously |

The cache corpus is where coverage-expanding inputs accumulate between runs. On this machine:

```bash
go env GOCACHE
```

```
/Users/raj/Library/Caches/go-build
```

and after the runs above, `$GOCACHE/fuzz/example.com/scratch/FuzzReverse/` held 4 files. That
directory makes a later fuzzing session resume from what the earlier one learned, which is why the
second run of a target often finds things faster. It is machine-local, disposable, and cleared by
`go clean -fuzzcache`.

The distinction that matters: **`testdata/fuzz` is your test suite, the cache is the fuzzer's
scratch space.** Commit the former, never the latter.

## The invariant styles that work

Fuzzing is only as good as the property you assert. The four that pay off:

**Round trip.** `decode(encode(x)) == x`. The strongest property available, because it needs no
oracle. A key-value codec with escaping:

```go
package kv

import "testing"

func FuzzRoundTrip(f *testing.F) {
  f.Add("name", "Ada")
  f.Add("", "")
  f.Add("a=b", "c;d")
  f.Add(`back\slash`, "=;=")
  f.Fuzz(func(t *testing.T, key, value string) {
    in := Pair{Key: key, Value: value}
    s := Format(in)
    got, err := Parse(s)
    if err != nil {
      t.Fatalf("Parse(Format(%#v)) = error %v, want no error (encoded: %q)", in, err, s)
    }
    if got != in {
      t.Errorf("round trip: Parse(Format(%#v)) = %#v (encoded %q)", in, got, s)
    }
  })
}
```

Seeds only, `go test -count=1 -v -run FuzzRoundTrip .`:

```
=== RUN   FuzzRoundTrip
=== RUN   FuzzRoundTrip/seed#0
=== RUN   FuzzRoundTrip/seed#1
=== RUN   FuzzRoundTrip/seed#2
=== RUN   FuzzRoundTrip/seed#3
--- PASS: FuzzRoundTrip (0.00s)
    --- PASS: FuzzRoundTrip/seed#0 (0.00s)
    --- PASS: FuzzRoundTrip/seed#1 (0.00s)
    --- PASS: FuzzRoundTrip/seed#2 (0.00s)
    --- PASS: FuzzRoundTrip/seed#3 (0.00s)
PASS
ok  	example.com/scratch	0.518s
```

Two fuzzed arguments — note the seeds pass two values to `f.Add`. Then
`-fuzz FuzzRoundTrip -fuzztime 30s`:

```
fuzz: elapsed: 27s, execs: 22439712 (814259/sec), new interesting: 20 (total: 24)
fuzz: elapsed: 30s, execs: 24832632 (797670/sec), new interesting: 20 (total: 24)
PASS
ok  	example.com/scratch	30.624s
```

24.8 million key/value pairs through escape-and-parse with no failure. That is meaningful evidence
about the escaping logic, and `new interesting` stabilising at 20 says the engine stopped finding
new branches — a sign the target is saturated and more time would add little.

**Never panics.** The weakest but cheapest property: just call the function. Any panic is a bug.
This is the right target for parsers, decoders, and anything handling untrusted input.

**Stability / idempotence.** `format(parse(format(x))) == format(x)`, or `normalize(normalize(x))
== normalize(x)`. Useful when a full round trip is impossible because the encoding is lossy.

**Differential.** Run two implementations — your optimized version against the obvious one, or your
parser against the standard library's — and assert they agree. The single highest-value fuzzing
pattern when you have a reference to compare against.

## Finding a panic, and minimization

The "never panics" style, on an off-by-one waiting to happen:

```go
package idx

import "strings"

func FieldAt(csv string, n int) string {
  parts := strings.Split(csv, ",")
  return parts[n]
}
```

```go
func FuzzFieldAt(f *testing.F) {
  f.Add("a,b,c", 1)
  f.Fuzz(func(t *testing.T, csv string, n int) {
    if n < 0 {
      t.Skip()
    }
    _ = FieldAt(csv, n)
  })
}
```

`go test -count=1 -run FuzzFieldAt -fuzz FuzzFieldAt -fuzztime 20s .`, tail of the output:

```
        testing.go:2076: panic: runtime error: index out of range [31] with length 1
            goroutine 28 [running]:
            runtime/debug.Stack()
            	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/debug/stack.go:26 +0xc4
            testing.tRunner.func1()
            	/opt/homebrew/Cellar/go/1.27.1/libexec/src/testing/testing.go:2076 +0x1c8
            panic({0x102b0a370?, 0x52753f3280f0?})
            	/opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/panic.go:859 +0x120
            example.com/scratch.FieldAt(...)
            	/tmp/scratch/idx.go:7
            example.com/scratch.FuzzFieldAt.func1(0x52753f388908, {0x10291edc0, 0x1}, 0x1f)
            	/tmp/scratch/idx_test.go:11 +0x134
    
    Failing input written to testdata/fuzz/FuzzFieldAt/96c17b7b3a4045e5
    To re-run:
    go test -run=FuzzFieldAt/96c17b7b3a4045e5
FAIL
exit status 1
FAIL	example.com/scratch	0.535s
```

A panic inside a fuzz target is caught and reported with the stack, not allowed to kill the run. The
saved file shows how aggressively minimization worked:

```
go test fuzz v1
string("0")
int(31)
```

A one-character string and a small index — the engine shrank both arguments to a minimal
reproducer. Compare that to debugging a 31-byte blob.

## Flags

| Flag | Effect |
|---|---|
| `-fuzz regexp` | fuzz the one matching target; requires exactly one match in one package |
| `-fuzztime 30s` | stop after a duration (default: forever) |
| `-fuzztime 1000x` | stop after N executions |
| `-fuzzminimizetime 10s` | cap each minimization attempt (default `60s`); `0` disables |
| `-parallel n` | number of fuzzing worker processes (default `GOMAXPROCS`) |
| `-race` | race detector while fuzzing, at ~12x the cost |
| `-run FuzzName/<hash>` | replay one saved corpus entry |
| `go clean -fuzzcache` | discard the generated corpus |

Fuzzing runs **after** all tests, benchmarks and other targets' seed corpora. Without `-fuzztime`
it runs until it fails or you interrupt it, so CI always sets a bound.

`-race` is worth it periodically: a fuzz target is a concurrency test if the code under test uses
goroutines, and the generated inputs reach paths your tests do not. The cost is real — the same
`FuzzRoundTrip` target managed 69,013 execs/sec under `-race` versus roughly 814,000 without, about
12x slower:

```
fuzz: elapsed: 9s, execs: 632460 (69013/sec), new interesting: 1 (total: 25)
fuzz: elapsed: 11s, execs: 701337 (33675/sec), new interesting: 1 (total: 25)
PASS
ok  	example.com/scratch	12.578s
```

## Coverage-guided mutation, briefly

The engine instruments the package to record which branches each execution reaches. It starts from
the seed corpus, measures baseline coverage, then repeatedly picks a corpus entry, mutates it
(flipping bits, splicing values, inserting interesting constants like 0, -1, maxint), and runs it.
If a mutated input reaches a branch no previous input reached, it is "interesting" and joins the
corpus to be mutated further. That feedback loop is what lets it find `d6 89` in 752 executions
rather than waiting out the astronomically unlikely odds of random generation: each step toward a
new branch is kept and built on. The practical implication is that **seeds matter** — a seed that
reaches deep into your code gives the mutator a much better starting point than an empty string.

## What fuzzing is bad at

- **Semantic correctness.** It cannot know that `Add(2, 3)` should be 5. It only checks properties
  you can state for all inputs, which is why round-trip and differential targets dominate.
- **Slow functions.** At 800k execs/sec a 1ms function gets you 1,000 executions per second
  instead. Fuzz the fast, pure core; test the slow orchestration conventionally.
- **Anything non-deterministic.** The engine re-runs inputs to confirm failures and to minimize. A
  target depending on time, randomness, the network or goroutine scheduling produces flaky
  crashers that will not reproduce.
- **Stateful or side-effecting code.** The target must not retain the input slices between calls
  (the engine reuses their memory) and should not write to shared files or databases.
- **Finding the bug quickly in a large input space.** It is a budget: minutes find shallow bugs,
  hours find deep ones. Fuzz continuously, not once.

Good candidates: parsers, decoders, encoders, validators, text processing, state machines, anything
that takes bytes from a user. Bad candidates: business logic with database calls, HTTP handlers end
to end, anything where "correct" needs a human.

## Quick Reference

| Form | Meaning |
|---|---|
| `func FuzzXxx(f *testing.F)` | a fuzz target |
| `f.Add(args...)` | one seed corpus entry; types must match `f.Fuzz` |
| `f.Fuzz(func(t *testing.T, a, b string){})` | the target body; called once |
| `t.Skip()` inside the target | this input is uninteresting |
| `f.Cleanup`, `f.Setenv`, `f.Helper` | allowed on `f`, before `f.Fuzz` |
| `testdata/fuzz/FuzzXxx/<hash>` | a saved failing input; commit it |
| `$GOCACHE/fuzz/...` | the generated corpus; do not commit |

| Command | Effect |
|---|---|
| `go test ./...` | runs fuzz targets over their seed + `testdata` corpus only |
| `go test -fuzz FuzzX -fuzztime 30s .` | actually fuzz, bounded |
| `go test -run 'FuzzX/<hash>' .` | replay one saved input |
| `go test -fuzz FuzzX -race .` | fuzz under the race detector |
| `go clean -fuzzcache` | drop the generated corpus |

| Property style | Assertion |
|---|---|
| never panics | just call it |
| round trip | `decode(encode(x)) == x` |
| stability | `f(f(x)) == f(x)` |
| differential | `fast(x) == reference(x)` |
| invariant | output length, validity, ordering preserved |

## Common Mistakes

**Thinking a fuzz target does nothing without `-fuzz`.** It runs the seed corpus and every saved
crasher as subtests on every `go test` — that is where its regression value lives.

**Not committing `testdata/fuzz/`.** The crasher the fuzzer spent an hour finding is then lost, and
the bug can return silently.

**Asserting a specific expected output.** The engine generates inputs you cannot predict; assert
properties, not answers.

**Writing a property that is not actually true.** The `\x92\xc3` case above was the test's bug, not
the code's; when a finding looks absurd, check the precondition before the implementation.

**Fuzzing something non-deterministic.** Confirmation and minimization re-run the input, so a
time- or network-dependent target yields crashers that will not reproduce.

**Trying to fuzz a struct.** Only the 15 scalar types are allowed; fuzz the fields, or fuzz
`[]byte` and decode.

**Retaining the fuzzed slice.** The engine may reuse its backing memory after the call; copy it if
you need to keep it.

**Running `-fuzz` in CI without `-fuzztime`.** It runs until it fails, which means forever.

## Exercises

1. Write a byte-reversing `Reverse` and the fuzz target from this lesson. Find the crasher yourself,
   commit `testdata/fuzz/`, fix the bug, and confirm the saved input passes with no `-fuzz`.
2. Add a `FuzzParse` target asserting only "never panics" to a hand-written CSV or INI parser.
   Report how long it took to find the first panic and what the minimized input was.
3. Write a differential fuzz target comparing your own `strconv.Atoi` replacement against the
   standard library's, agreeing on both the value and whether an error occurred.
4. Fuzz a JSON config decoder by fuzzing `[]byte` and `t.Skip`-ing inputs that fail to unmarshal.
   Report `new interesting` after 30s with and without a realistic seed, and explain the difference.
5. Take a fuzz target that passes for 60s. Add `-race` and measure the execs/sec drop. Then
   introduce a data race in the code under test and see whether fuzzing finds it.
6. Run `go clean -fuzzcache`, fuzz for 30s, then fuzz again for 30s without cleaning. Compare the
   `new interesting` counts and explain what the cache corpus bought you.

---

[← 14.5 Benchmarks](05-benchmarks.md) · [Course Index](../README.md) · [14.7 Examples as Tests and Coverage →](07-examples-coverage.md)
