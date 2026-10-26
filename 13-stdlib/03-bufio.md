[← 13.2 io: Reader, Writer, and the Helpers](02-io.md) · [Course Index](../README.md) · [13.4 time: Durations, Instants, and Formatting →](04-time.md)

# 13.3 — bufio: Scanner, Reader, and Writer

## What You'll Learn

- Why buffering matters, with measured numbers: ~178x on a byte-at-a-time write
- `bufio.Scanner`, its split functions, a custom `SplitFunc`, and the 64KB token limit that silently eats your input
- That `Scanner` stops on error and reports success unless you check `Err()`
- `bufio.Reader.ReadString` **includes** the delimiter while `Scanner.Text` strips it
- That forgetting `Flush` on a `bufio.Writer` loses data with no error anywhere

## Coming From Another Language

| Other | Go |
|---|---|
| Python `for line in f:` (buffered by default) | `*os.File` is **unbuffered**; wrap it yourself |
| Node `readline` interface | `bufio.Scanner` with the default `ScanLines` |
| Python `f.readline()` keeping `"\n"` | `ReadString('\n')` keeps it, `Scanner.Text()` does not |
| Rust `BufReader`/`BufWriter` | `bufio.Reader`/`bufio.Writer`, same idea |
| Rust's `BufWriter` flushing on drop | Go has no destructors — `defer w.Flush()` or lose the data |

Go's `*os.File` is a raw file descriptor: every `Write` is a syscall, and every `Read` is
a syscall. There is no implicit buffering anywhere, unlike Python's file objects or Node's
streams, which is why wrapping in `bufio` is not an optimization you reach for later but
the normal way to do line-oriented I/O. The second Go-specific fact is the mirror of it:
nothing flushes a `bufio.Writer` for you. Rust's `BufWriter` flushes in `Drop`; Go has no
destructors, so an unflushed buffer is simply discarded when the program exits, silently
and with a nil error from every call you made.

## Why buffering

A byte-at-a-time loop against a bare `*os.File` makes one syscall per byte. The same loop
through `bufio` makes one per 4096 bytes.

```go
package main

import (
  "bufio"
  "io"
  "os"
  "testing"
)

const n = 200000

func BenchmarkUnbufferedWrite(b *testing.B) {
  for b.Loop() {
    f, _ := os.CreateTemp("", "bw-*")
    for i := 0; i < n; i++ {
      f.Write([]byte{'x'})
    }
    f.Close()
    os.Remove(f.Name())
  }
}

func BenchmarkBufioWrite(b *testing.B) {
  for b.Loop() {
    f, _ := os.CreateTemp("", "bw-*")
    w := bufio.NewWriter(f)
    for i := 0; i < n; i++ {
      w.WriteByte('x')
    }
    w.Flush()
    f.Close()
    os.Remove(f.Name())
  }
}

func BenchmarkUnbufferedRead(b *testing.B) {
  f := tempFile(b)
  for b.Loop() {
    r, _ := os.Open(f)
    one := make([]byte, 1)
    for {
      if _, err := r.Read(one); err == io.EOF {
        break
      }
    }
    r.Close()
  }
}

func BenchmarkBufioRead(b *testing.B) {
  f := tempFile(b)
  for b.Loop() {
    r, _ := os.Open(f)
    br := bufio.NewReader(r)
    for {
      if _, err := br.ReadByte(); err == io.EOF {
        break
      }
    }
    r.Close()
  }
}

func tempFile(b *testing.B) string {
  f, _ := os.CreateTemp("", "br-*")
  buf := make([]byte, n)
  for i := range buf {
    buf[i] = 'x'
  }
  f.Write(buf)
  f.Close()
  b.Cleanup(func() { os.Remove(f.Name()) })
  return f.Name()
}
```

`go test -bench . -benchmem`, 200,000 bytes per iteration. These numbers are
machine-specific (darwin/arm64, Apple M4 Pro) but the ratio is not. Output:

```
goos: darwin
goarch: arm64
pkg: example.com/scratch
cpu: Apple M4 Pro
BenchmarkUnbufferedWrite-12    	       6	 187279062 ns/op	     376 B/op	       7 allocs/op
BenchmarkBufioWrite-12         	    1106	   1078595 ns/op	    4471 B/op	       8 allocs/op
BenchmarkUnbufferedRead-12     	      20	  57294675 ns/op	     168 B/op	       3 allocs/op
BenchmarkBufioRead-12          	    3385	    357216 ns/op	    4264 B/op	       4 allocs/op
```

187ms versus 1.08ms — about **178x** for writes, and about **160x** for reads. The cost is
one 4KB allocation. This is the clearest cost/benefit ratio in the standard library.

## bufio.Scanner

| Method | Signature | Note |
|---|---|---|
| `NewScanner` | `(r io.Reader) *Scanner` | default split is `ScanLines` |
| `Scan` | `() bool` | advances; `false` at EOF **or on error** |
| `Text` | `() string` | the token as a fresh string |
| `Bytes` | `() []byte` | the token; **aliases the internal buffer** |
| `Err` | `() error` | nil at clean EOF; the error otherwise |
| `Split` | `(f SplitFunc)` | must be called before the first `Scan` |
| `Buffer` | `(buf []byte, max int)` | must be called before the first `Scan` |

| Split function | Token |
|---|---|
| `ScanLines` | a line, with `\n` and a trailing `\r` stripped |
| `ScanWords` | a run of non-space, whitespace stripped |
| `ScanRunes` | one UTF-8 rune; invalid bytes become `U+FFFD` |
| `ScanBytes` | one byte |

```go
package main

import (
  "bufio"
  "fmt"
  "strings"
)

func main() {
  fmt.Println("--- ScanLines (default) ---")
  s := bufio.NewScanner(strings.NewReader("alpha\nbeta\r\ngamma\n"))
  for s.Scan() {
    fmt.Printf("%q\n", s.Text())
  }
  fmt.Println("Err:", s.Err())

  fmt.Println("--- no trailing newline ---")
  s = bufio.NewScanner(strings.NewReader("one\ntwo"))
  for s.Scan() {
    fmt.Printf("%q\n", s.Text())
  }

  fmt.Println("--- empty input ---")
  s = bufio.NewScanner(strings.NewReader(""))
  fmt.Println("Scan:", s.Scan(), "Err:", s.Err())

  fmt.Println("--- blank lines are tokens ---")
  s = bufio.NewScanner(strings.NewReader("a\n\n\nb\n"))
  for s.Scan() {
    fmt.Printf("%q\n", s.Text())
  }

  fmt.Println("--- ScanWords ---")
  s = bufio.NewScanner(strings.NewReader("  the   quick\tbrown\nfox  "))
  s.Split(bufio.ScanWords)
  for s.Scan() {
    fmt.Printf("%q ", s.Text())
  }
  fmt.Println()

  fmt.Println("--- ScanRunes ---")
  s = bufio.NewScanner(strings.NewReader("héllo"))
  s.Split(bufio.ScanRunes)
  for s.Scan() {
    fmt.Printf("%q(%d bytes) ", s.Text(), len(s.Bytes()))
  }
  fmt.Println()

  s = bufio.NewScanner(strings.NewReader("héllo"))
  s.Split(bufio.ScanBytes)
  n := 0
  for s.Scan() {
    n++
  }
  fmt.Println("ScanBytes token count:", n)
}
```

Output:

```
--- ScanLines (default) ---
"alpha"
"beta"
"gamma"
Err: <nil>
--- no trailing newline ---
"one"
"two"
--- empty input ---
Scan: false Err: <nil>
--- blank lines are tokens ---
"a"
""
""
"b"
--- ScanWords ---
"the" "quick" "brown" "fox" 
--- ScanRunes ---
"h"(1 bytes) "é"(2 bytes) "l"(1 bytes) "l"(1 bytes) "o"(1 bytes) 
ScanBytes token count: 6
```

`ScanLines` strips `\r\n` as well as `\n`, so Windows input needs no help. A missing final
newline still yields the last token. Empty input yields zero tokens and a nil `Err` — a
clean EOF, not a failure. Blank lines are real tokens (`""`), so filtering them is your
job, and `ScanRunes` counts 5 tokens for `"héllo"` where `ScanBytes` counts 6.

### Bytes() aliases the buffer

`s.Bytes()` points into the scanner's working buffer, which is reused and may be moved on
the next `Scan`. Retaining it is a real bug, not a theoretical one:

```go
package main

import (
  "bufio"
  "fmt"
  "strings"
)

func main() {
  var sb strings.Builder
  for i := 0; i < 4; i++ {
    sb.WriteString(strings.Repeat(string(rune('a'+i)), 30000))
    sb.WriteByte('\n')
  }
  input := sb.String()

  s := bufio.NewScanner(strings.NewReader(input))
  var keep [][]byte
  for s.Scan() {
    keep = append(keep, s.Bytes())
  }
  fmt.Print("WRONG (s.Bytes retained): ")
  for _, b := range keep {
    fmt.Printf("%c×%d ", b[0], len(b))
  }
  fmt.Println()

  s = bufio.NewScanner(strings.NewReader(input))
  var safe [][]byte
  for s.Scan() {
    safe = append(safe, append([]byte(nil), s.Bytes()...))
  }
  fmt.Print("RIGHT (copied):           ")
  for _, b := range safe {
    fmt.Printf("%c×%d ", b[0], len(b))
  }
  fmt.Println()

  s = bufio.NewScanner(strings.NewReader(input))
  var texts []string
  for s.Scan() {
    texts = append(texts, s.Text())
  }
  fmt.Print("RIGHT (s.Text):           ")
  for _, t := range texts {
    fmt.Printf("%c×%d ", t[0], len(t))
  }
  fmt.Println()
}
```

Output:

```
WRONG (s.Bytes retained): d×30000 d×30000 d×30000 d×30000 
RIGHT (copied):           a×30000 b×30000 c×30000 d×30000 
RIGHT (s.Text):           a×30000 b×30000 c×30000 d×30000 
```

All four retained slices show `d` — they are four views of the same reused memory. The
tokens are 30KB each specifically to force the buffer to be reused rather than regrown;
with short lines the bug often hides, which is what makes it dangerous.

Idiomatic: use `s.Text()` unless you measured the allocation and it mattered. If you need
`Bytes()`, copy with `append([]byte(nil), s.Bytes()...)`.

### The 64KB token limit

A `Scanner` refuses any token longer than `bufio.MaxScanTokenSize` (65536). This is the
single most common `bufio` bug, because the failure looks like "the file was empty".

```go
package main

import (
  "bufio"
  "errors"
  "fmt"
  "strings"
)

func main() {
  long := strings.Repeat("x", 100000) + "\nshort line\n"

  fmt.Println("--- default 64KB limit ---")
  s := bufio.NewScanner(strings.NewReader(long))
  count := 0
  for s.Scan() {
    count++
    fmt.Printf("token %d: %d bytes\n", count, len(s.Bytes()))
  }
  fmt.Println("tokens:", count)
  fmt.Println("Err():", s.Err())
  fmt.Println("is ErrTooLong:", errors.Is(s.Err(), bufio.ErrTooLong))

  fmt.Println("--- the silent-truncation bug ---")
  s2 := bufio.NewScanner(strings.NewReader(long))
  lines := 0
  for s2.Scan() {
    lines++
  }
  fmt.Printf("counted %d lines and reported success\n", lines)

  fmt.Println("--- fixed with Buffer ---")
  s3 := bufio.NewScanner(strings.NewReader(long))
  s3.Buffer(make([]byte, 0, 64*1024), 1024*1024)
  count = 0
  for s3.Scan() {
    count++
    fmt.Printf("token %d: %d bytes\n", count, len(s3.Bytes()))
  }
  fmt.Println("Err():", s3.Err())

  fmt.Println("--- MaxScanTokenSize ---")
  fmt.Println("default max:", bufio.MaxScanTokenSize)
  fmt.Println("ErrTooLong:", bufio.ErrTooLong)
  fmt.Println("ErrFinalToken:", bufio.ErrFinalToken)
  fmt.Println("ErrAdvanceTooFar:", bufio.ErrAdvanceTooFar)
  fmt.Println("ErrNegativeAdvance:", bufio.ErrNegativeAdvance)
  fmt.Println("ErrBufferFull:", bufio.ErrBufferFull)
}
```

Output:

```
--- default 64KB limit ---
tokens: 0
Err(): bufio.Scanner: token too long
is ErrTooLong: true
--- the silent-truncation bug ---
counted 0 lines and reported success
--- fixed with Buffer ---
token 1: 100000 bytes
token 2: 10 bytes
Err(): <nil>
--- MaxScanTokenSize ---
default max: 65536
ErrTooLong: bufio.Scanner: token too long
ErrFinalToken: final token
ErrAdvanceTooFar: bufio.Scanner: SplitFunc returns advance count beyond input
ErrNegativeAdvance: bufio.Scanner: SplitFunc returns negative advance count
ErrBufferFull: bufio: buffer full
```

Read the second section carefully: **zero** lines, and the loop exited normally. The
`short line` after the long one was never seen either, because `Scan` stops at the first
error and never resumes. Without the `s.Err()` check, a program hitting a 100KB JSON line
in a log reports "0 records, no errors".

`Buffer(buf, max)` sets the starting buffer and the ceiling; `max` is what matters, since
the scanner grows from `cap(buf)` up to it.

Idiomatic: if the input is not under your control, either raise the limit explicitly with
`s.Buffer(make([]byte, 0, 64*1024), 1<<20)`, or use `bufio.Reader`, which has no token
limit at all.

### Scanner stops on error — always check Err()

`Scan` returning `false` means "no more tokens", which covers both a clean EOF and a hard
failure. The two are indistinguishable without `Err()`.

```go
package main

import (
  "bufio"
  "errors"
  "fmt"
  "io"
  "strings"
)

type failAfter struct {
  r io.Reader
  n int
}

func (f *failAfter) Read(p []byte) (int, error) {
  if f.n <= 0 {
    return 0, errors.New("disk went away")
  }
  f.n--
  if len(p) > 4 {
    p = p[:4]
  }
  return f.r.Read(p)
}

func countBuggy(r io.Reader) int {
  s := bufio.NewScanner(r)
  n := 0
  for s.Scan() {
    n++
  }
  return n
}

func countCorrect(r io.Reader) (int, error) {
  s := bufio.NewScanner(r)
  n := 0
  for s.Scan() {
    n++
  }
  return n, s.Err()
}

func main() {
  src := func() io.Reader {
    return &failAfter{r: strings.NewReader("a\nb\nc\nd\ne\n"), n: 1}
  }

  fmt.Println("buggy count:  ", countBuggy(src()))
  n, err := countCorrect(src())
  fmt.Println("correct count:", n, "err:", err)

  n, err = countCorrect(strings.NewReader("a\nb\nc\nd\ne\n"))
  fmt.Println("clean input:  ", n, "err:", err)
}
```

Output:

```
buggy count:   2
correct count: 2 err: disk went away
clean input:   5 err: <nil>
```

The input has five lines. Both functions counted **2** and stopped; only the second one
knows that is wrong.

### A custom SplitFunc

```go
type SplitFunc func(data []byte, atEOF bool) (advance int, token []byte, err error)
```

The contract: return `(0, nil, nil)` to ask for more data, `(n, token, nil)` to emit a
token and consume `n` bytes, or `(n, token, bufio.ErrFinalToken)` to emit one last token
and stop. `advance` may exceed `len(token)` — that is how delimiters get discarded.

```go
package main

import (
  "bufio"
  "bytes"
  "fmt"
  "strings"
)

func scanSemicolons(data []byte, atEOF bool) (advance int, token []byte, err error) {
  if atEOF && len(data) == 0 {
    return 0, nil, nil
  }
  if i := bytes.IndexByte(data, ';'); i >= 0 {
    return i + 1, bytes.TrimSpace(data[:i]), nil
  }
  if atEOF {
    return len(data), bytes.TrimSpace(data), nil
  }
  return 0, nil, nil
}

func scanParagraphs(data []byte, atEOF bool) (int, []byte, error) {
  if atEOF && len(data) == 0 {
    return 0, nil, nil
  }
  if i := bytes.Index(data, []byte("\n\n")); i >= 0 {
    return i + 2, data[:i], nil
  }
  if atEOF {
    return len(data), bytes.TrimRight(data, "\n"), nil
  }
  return 0, nil, nil
}

func main() {
  fmt.Println("--- custom SplitFunc: semicolons ---")
  s := bufio.NewScanner(strings.NewReader("SELECT 1; SELECT 2 ;  SELECT 3"))
  s.Split(scanSemicolons)
  for s.Scan() {
    fmt.Printf("%q\n", s.Text())
  }
  fmt.Println("Err:", s.Err())

  fmt.Println("--- custom SplitFunc: paragraphs ---")
  s = bufio.NewScanner(strings.NewReader("line a\nline b\n\npara two\n\npara three\n"))
  s.Split(scanParagraphs)
  for s.Scan() {
    fmt.Printf("%q\n", s.Text())
  }
  fmt.Println("Err:", s.Err())

  fmt.Println("--- ErrFinalToken stops cleanly ---")
  s = bufio.NewScanner(strings.NewReader("a\nb\nSTOP\nc\nd\n"))
  s.Split(func(data []byte, atEOF bool) (int, []byte, error) {
    adv, tok, err := bufio.ScanLines(data, atEOF)
    if string(tok) == "STOP" {
      return adv, nil, bufio.ErrFinalToken
    }
    return adv, tok, err
  })
  for s.Scan() {
    fmt.Printf("%q\n", s.Text())
  }
  fmt.Println("Err:", s.Err())
}
```

Output:

```
--- custom SplitFunc: semicolons ---
"SELECT 1"
"SELECT 2"
"SELECT 3"
Err: <nil>
--- custom SplitFunc: paragraphs ---
"line a\nline b"
"para two"
"para three"
Err: <nil>
--- ErrFinalToken stops cleanly ---
"a"
"b"
Err: <nil>
```

Two rules to internalize. The `atEOF && len(data) == 0` guard must come first or the
scanner spins. And `ErrFinalToken` is the one "error" that leaves `Err()` nil — it means
"stop here, successfully", which is how you implement an early-terminating parser without
a sentinel of your own.

## bufio.Reader

No token limit, more control, more bookkeeping.

| Method | Signature | Note |
|---|---|---|
| `NewReader` | `(rd io.Reader) *Reader` | 4096-byte buffer |
| `NewReaderSize` | `(rd io.Reader, size int) *Reader` | minimum 16 |
| `ReadString` | `(delim byte) (string, error)` | **includes** the delimiter |
| `ReadBytes` | `(delim byte) ([]byte, error)` | includes it; fresh allocation |
| `ReadSlice` | `(delim byte) ([]byte, error)` | includes it; **aliases the buffer**; `ErrBufferFull` |
| `ReadLine` | `() ([]byte, bool, error)` | strips the newline; `isPrefix` for long lines |
| `ReadByte` | `() (byte, error)` | |
| `UnreadByte` | `() error` | one byte back; errors if the last op was not a read |
| `ReadRune` | `() (rune, int, error)` | size in bytes |
| `UnreadRune` | `() error` | only valid right after `ReadRune` |
| `Peek` | `(n int) ([]byte, error)` | look ahead without consuming; `ErrBufferFull` if `n > size` |
| `Discard` | `(n int) (int, error)` | skip `n` bytes efficiently |
| `Buffered` | `() int` | bytes available without a read |
| `Size` | `() int` | buffer size |
| `Reset` | `(r io.Reader)` | reuse the reader on a new source |

### ReadString includes the delimiter; Scanner strips it

This is the most common point of confusion between the two APIs. Same input, both ways:

```go
package main

import (
  "bufio"
  "fmt"
  "strings"
)

const input = "alpha\nbeta\ngamma"

func main() {
  fmt.Println("--- Scanner STRIPS the newline ---")
  s := bufio.NewScanner(strings.NewReader(input))
  for s.Scan() {
    fmt.Printf("%q\n", s.Text())
  }

  fmt.Println("--- ReadString INCLUDES the newline ---")
  r := bufio.NewReader(strings.NewReader(input))
  for {
    line, err := r.ReadString('\n')
    fmt.Printf("%q err=%v\n", line, err)
    if err != nil {
      break
    }
  }

  fmt.Println("--- Peek / Discard / Buffered ---")
  r = bufio.NewReaderSize(strings.NewReader("0123456789abcdef"), 16)
  p, err := r.Peek(4)
  fmt.Printf("Peek(4)=%q err=%v Buffered=%d\n", p, err, r.Buffered())
  n, err := r.Discard(3)
  fmt.Printf("Discard(3)=%d err=%v\n", n, err)
  p, _ = r.Peek(4)
  fmt.Printf("Peek(4) after discard=%q Buffered=%d\n", p, r.Buffered())
  p, err = r.Peek(100)
  fmt.Printf("Peek(100)=%q err=%v\n", p, err)

  fmt.Println("--- Unread ---")
  r = bufio.NewReader(strings.NewReader("abc"))
  c, _ := r.ReadByte()
  fmt.Printf("ReadByte=%q UnreadByte=%v\n", c, r.UnreadByte())
  c, _ = r.ReadByte()
  fmt.Printf("ReadByte again=%q\n", c)
  fmt.Println("UnreadByte twice:", r.UnreadByte(), r.UnreadByte())

  r = bufio.NewReader(strings.NewReader("héllo"))
  r.ReadRune()
  ru, size, _ := r.ReadRune()
  fmt.Printf("ReadRune=%q size=%d UnreadRune=%v\n", ru, size, r.UnreadRune())
  ru, size, _ = r.ReadRune()
  fmt.Printf("ReadRune again=%q size=%d\n", ru, size)
}
```

Output:

```
--- Scanner STRIPS the newline ---
"alpha"
"beta"
"gamma"
--- ReadString INCLUDES the newline ---
"alpha\n" err=<nil>
"beta\n" err=<nil>
"gamma" err=EOF
--- Peek / Discard / Buffered ---
Peek(4)="0123" err=<nil> Buffered=16
Discard(3)=3 err=<nil>
Peek(4) after discard="3456" Buffered=13
Peek(100)="3456789abcdef" err=bufio: buffer full
--- Unread ---
ReadByte='a' UnreadByte=<nil>
ReadByte again='a'
UnreadByte twice: <nil> bufio: invalid use of UnreadByte
ReadRune='é' size=2 UnreadRune=<nil>
ReadRune again='é' size=2
```

The `"alpha"` vs `"alpha\n"` difference is the headline. Writing
`strings.TrimSuffix(line, "\n")` after every `ReadString` is a sign you wanted a
`Scanner`.

`ReadLine` is the third option: it strips the newline like `Scanner` but returns a
buffer-aliasing slice plus an `isPrefix` flag for lines longer than the buffer. It exists
for low-level use; prefer the other two.

The final `ReadString` returns `"gamma"` **with** `io.EOF` — data and EOF together, exactly
the `io.Reader` subtlety from 13.2. Breaking out of the loop before printing would lose
`gamma`.

`Peek(100)` on a 16-byte buffer returns what it has plus `bufio.ErrBufferFull` — `Peek`
never sees further than `Size()`, so longer lookahead needs a bigger `NewReaderSize`.

`UnreadByte` steps back one byte, and only right after a read — two in a row gives
`bufio: invalid use of UnreadByte`.

`ReadSlice` (not shown above) returns a view into the buffer that the next read
invalidates, and fails with `ErrBufferFull` on a token longer than the buffer — it is the
zero-allocation primitive `ReadBytes` and `ReadString` are built on. `Size()` is 4096 by
default, and `NewReaderSize(r, 1)` is silently promoted to the 16-byte minimum.

### Scanner vs Reader

| Want | Use |
|---|---|
| Lines, words or runes, simply | `Scanner` |
| Tokens that may exceed 64KB | `Reader` (or `Scanner` + `Buffer`) |
| The delimiter kept | `Reader.ReadString` |
| The delimiter stripped | `Scanner.Text` or `Reader.ReadLine` |
| Lookahead before deciding | `Reader.Peek` |
| Push a byte or rune back | `Reader.UnreadByte` / `UnreadRune` |
| A hand-written tokenizer | `Reader.ReadRune` + `UnreadRune` |
| Error detail and position | `Reader` — `Scanner` collapses everything into `Err()` |

Idiomatic: reach for `Scanner` first. Switch to `Reader` when you hit its limits.

## bufio.Writer

| Method | Signature | Note |
|---|---|---|
| `NewWriter` | `(w io.Writer) *Writer` | 4096-byte buffer |
| `NewWriterSize` | `(w io.Writer, size int) *Writer` | |
| `WriteString` | `(s string) (int, error)` | no `[]byte` conversion |
| `WriteByte` | `(c byte) error` | returns only an error |
| `WriteRune` | `(r rune) (int, error)` | returns bytes written |
| `Flush` | `() error` | **mandatory** |
| `Buffered` | `() int` | bytes waiting |
| `Available` | `() int` | room left |
| `Size` | `() int` | buffer size |
| `Reset` | `(w io.Writer)` | **discards** anything buffered |

### Forgetting Flush loses data, silently

```go
package main

import (
  "bufio"
  "fmt"
  "os"
  "strings"
)

func writeForgettingFlush(path string) {
  f, _ := os.Create(path)
  defer f.Close()
  w := bufio.NewWriter(f)
  for i := 0; i < 3; i++ {
    fmt.Fprintf(w, "line %d\n", i)
  }
}

func writeWithFlush(path string) error {
  f, err := os.Create(path)
  if err != nil {
    return err
  }
  defer f.Close()
  w := bufio.NewWriter(f)
  for i := 0; i < 3; i++ {
    fmt.Fprintf(w, "line %d\n", i)
  }
  return w.Flush()
}

func main() {
  dir, _ := os.MkdirTemp("", "buflesson")
  defer os.RemoveAll(dir)

  writeForgettingFlush(dir + "/lost.txt")
  data, _ := os.ReadFile(dir + "/lost.txt")
  fmt.Printf("no Flush:   %d bytes %q\n", len(data), data)

  writeWithFlush(dir + "/kept.txt")
  data, _ = os.ReadFile(dir + "/kept.txt")
  fmt.Printf("with Flush: %d bytes %q\n", len(data), data)

  fmt.Println("--- Buffered / Available ---")
  var sb strings.Builder
  w := bufio.NewWriterSize(&sb, 16)
  fmt.Println("Size:", w.Size(), "Available:", w.Available(), "Buffered:", w.Buffered())
  w.WriteString("hello")
  fmt.Println("after 5 bytes -> Available:", w.Available(), "Buffered:", w.Buffered())
  fmt.Printf("downstream so far: %q\n", sb.String())
  w.WriteString("world, this overflows")
  fmt.Println("after overflow  -> Available:", w.Available(), "Buffered:", w.Buffered())
  fmt.Printf("downstream now:    %q\n", sb.String())
  fmt.Println("Flush:", w.Flush())
  fmt.Printf("downstream final:  %q\n", sb.String())

  fmt.Println("--- Reset discards buffered data ---")
  var a, b strings.Builder
  w2 := bufio.NewWriter(&a)
  w2.WriteString("goes nowhere")
  w2.Reset(&b)
  w2.WriteString("goes to b")
  w2.Flush()
  fmt.Printf("a=%q b=%q\n", a.String(), b.String())
}
```

Output:

```
no Flush:   0 bytes ""
with Flush: 21 bytes "line 0\nline 1\nline 2\n"
--- Buffered / Available ---
Size: 16 Available: 16 Buffered: 0
after 5 bytes -> Available: 11 Buffered: 5
downstream so far: ""
after overflow  -> Available: 6 Buffered: 10
downstream now:    "helloworld, this"
Flush: <nil>
downstream final:  "helloworld, this overflows"
--- Reset discards buffered data ---
a="" b="goes to b"
```

**Zero bytes.** Every `Fprintf` returned a nil error, the file was created, `Close`
succeeded, and the data is gone. Nothing in Go will flush that buffer for you. The
`downstream so far: ""` line shows the mechanism: nothing reached the destination until
the 16-byte buffer overflowed, and the remainder waited for `Flush`.

`Reset` is a reuse primitive, not a clear-and-keep-writing one — it throws away the 12
buffered bytes destined for `a`.

### Flushing correctly

`defer w.Flush()` is the common form, but it throws the error away — and a flush is
exactly when a disk-full or broken-pipe error surfaces. Make `Flush` the last statement
and return its error:

```go
func writeReport(path string, rows []string) (err error) {
  f, err := os.Create(path)
  if err != nil {
    return err
  }
  defer func() {
    if cerr := f.Close(); err == nil {
      err = cerr
    }
  }()

  w := bufio.NewWriter(f)
  for i, r := range rows {
    if _, err := fmt.Fprintf(w, "%d,%s\n", i, r); err != nil {
      return err
    }
  }
  return w.Flush()
}
```

The named return plus the deferred `Close` captures a close error too, without masking a
real write error.

Idiomatic: return `w.Flush()` as the function's last act. Use `defer w.Flush()` only in a
function whose output nobody depends on.

## bufio.ReadWriter and a complete program

`bufio.ReadWriter` is a struct embedding both, for protocol code that reads a request and
writes a response on one connection — `net/http`'s `Hijacker` hands you exactly this.
`bufio.NewReadWriter(bufio.NewReader(c), bufio.NewWriter(c))` gives a value with every
method of both, and you still have to `Flush` it.

The shape worth copying is a buffered filter with both error checks in place:

```go
package main

import (
  "bufio"
  "fmt"
  "os"
  "strings"
)

func filterErrors(in, out string) error {
  src, err := os.Open(in)
  if err != nil {
    return err
  }
  defer src.Close()

  dst, err := os.Create(out)
  if err != nil {
    return err
  }
  defer dst.Close()

  w := bufio.NewWriter(dst)
  defer w.Flush()

  sc := bufio.NewScanner(src)
  n := 0
  for sc.Scan() {
    if code, _, ok := strings.Cut(sc.Text(), " "); ok && code >= "500" {
      n++
      fmt.Fprintf(w, "%d: %s\n", n, sc.Text())
    }
  }
  if err := sc.Err(); err != nil {
    return err
  }
  return w.Flush()
}

func main() {
  dir, _ := os.MkdirTemp("", "buflesson")
  defer os.RemoveAll(dir)
  src := dir + "/access.log"
  os.WriteFile(src, []byte(
    "200 /index\n404 /missing\n500 /boom\n200 /about\n503 /down\n"), 0o644)

  if err := filterErrors(src, dir+"/errors.log"); err != nil {
    fmt.Println("error:", err)
    return
  }
  data, _ := os.ReadFile(dir + "/errors.log")
  fmt.Printf("%s", data)
}
```

Output:

```
1: 500 /boom
2: 503 /down
```

Buffered read, buffered write, `sc.Err()` checked before declaring success, and
`w.Flush()`'s error returned. The `defer w.Flush()` covers the early-return paths; the
explicit one at the end is what reports failure.

## Quick Reference

| Need | Call |
|---|---|
| Read lines | `s := bufio.NewScanner(r); for s.Scan() { s.Text() }` |
| Check for a read error | `if err := s.Err(); err != nil` |
| Read words | `s.Split(bufio.ScanWords)` |
| Raise the token limit | `s.Buffer(make([]byte, 0, 64*1024), 1<<20)` |
| Keep the token past the next `Scan` | `s.Text()`, or copy `s.Bytes()` |
| Lines with the newline attached | `r.ReadString('\n')` |
| Look ahead | `r.Peek(n)` |
| Push back one rune | `r.UnreadRune()` |
| Buffer writes | `w := bufio.NewWriter(f); defer w.Flush()` |
| Report a flush error | `return w.Flush()` as the last statement |
| Read and write one conn | `bufio.NewReadWriter(r, w)` |

| Trap | Reality |
|---|---|
| No `Flush` | every byte silently discarded |
| `defer w.Flush()` only | the flush error is thrown away |
| Not calling `s.Err()` | a mid-file failure looks like a clean EOF |
| Lines over 64KB | `bufio.Scanner: token too long`, zero tokens, no panic |
| Retaining `s.Bytes()` | the next `Scan` overwrites it |
| `s.Split` after the first `Scan` | panics: `Split called after Scan` |
| `TrimSuffix(line, "\n")` after `ReadString` | you wanted a `Scanner` |

## Common Mistakes

**Not calling `Flush` on a `bufio.Writer`.** The data is discarded with no error anywhere;
`return w.Flush()` as the function's last statement.

**Ignoring `Scanner.Err()`.** `Scan` returning `false` covers both EOF and failure, so a
truncated read reports success.

**Assuming a `Scanner` handles any line length.** Tokens over 64KB fail with
`bufio.ErrTooLong` and *no* tokens are produced; call `Buffer` or use `bufio.Reader`.

**Storing `Scanner.Bytes()` in a slice.** It aliases a reused buffer; use `Text()` or copy
it.

**Expecting `ReadString('\n')` to strip the newline.** It includes the delimiter by
design — `Scanner.Text()` is the one that strips.

## Exercises

1. Write `func wordFreq(r io.Reader) (map[string]int, error)` with `ScanWords`. Return
   `s.Err()` and prove with a failing reader that the error propagates.
2. Implement a `SplitFunc` for a length-prefixed format: a `uint16` big-endian length,
   then that many payload bytes. Distinguish a truncated prefix from a truncated payload.
3. Write `func tailFollow(path string, out io.Writer) error` that reads to the end with a
   `bufio.Reader`, then polls for appended lines. Explain why `Scanner` cannot.
4. Benchmark `Scanner.Text()` against `Scanner.Bytes()` on a 10-million-line file with
   `-benchmem`, and report the allocation difference.
5. Build `func csvRoundTrip(in, out string) error` that reverses each line's fields.
   Inject a failing `io.Writer` and assert `Flush`'s error reaches the caller.

---

[← 13.2 io: Reader, Writer, and the Helpers](02-io.md) · [Course Index](../README.md) · [13.4 time: Durations, Instants, and Formatting →](04-time.md)
