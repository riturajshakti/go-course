[← 13.3 bufio: Scanner, Reader, and Writer](03-bufio.md) · [Course Index](../README.md) · [13.5 time: Timers, Tickers, and After →](05-timers.md)

# 13.4 — time: Durations, Instants, and Formatting

## What You'll Learn

- `time.Time` carries a hidden **monotonic** reading, which operations strip it, and why `==` on two times is wrong
- `Duration` is a typed `int64` of nanoseconds, with the `n * time.Second` compile error that follows
- The reference-time layout system (`Mon Jan 2 15:04:05 MST 2006`) in full, with real output for one fixed instant
- `Parse`'s real error text, and why you add days with `AddDate`, not `24*time.Hour`

## Coming From Another Language

| Other | Go |
|---|---|
| JS `Date.now()` returning a number | `time.Now()` returns a struct with wall **and** monotonic clocks |
| JS `setTimeout(f, 1000)` — a bare number | `time.Duration` is typed; a bare `1000` will not compile |
| Python `timedelta(seconds=5)` | `5 * time.Second` |
| Python `strftime("%Y-%m-%d")` | `Format("2006-01-02")` — an **example date**, not escapes |
| `moment().add(1, 'day')` | `t.AddDate(0, 0, 1)`, not `t.Add(24*time.Hour)` |
| Rust `Instant` vs `SystemTime` (two types) | one `time.Time` holding both readings |

`time.Time` is the one standard-library type with a genuinely surprising design: it carries
*two* clock readings. The wall clock is what you format, compare and store; the monotonic
reading is a count from an arbitrary process-start origin that never jumps when NTP
adjusts the system clock or the machine suspends. `Sub` and `Since` silently use the
monotonic reading when both operands have one, which is why timing code in Go is correct
by default where in other languages it is a known hazard. The cost is that `t1 == t2`
compares monotonic readings too, so two values describing the same instant can compare
unequal — use `Equal`. The second oddity is the formatting system: instead of `%Y-%m-%d`,
you write out the reference instant `Mon Jan 2 15:04:05 MST 2006` in the shape you want.

## The monotonic reading

`time.Now().String()` shows it after `m=`:

```go
package main

import (
  "fmt"
  "strings"
  "time"
)

func main() {
  now := time.Now()
  fmt.Println("Now() has monotonic:", now.String())

  fmt.Println("--- what STRIPS the monotonic reading ---")
  for _, c := range []struct {
    name string
    t    time.Time
  }{
    {"Now()", now},
    {"Round(0)", now.Round(0)},
    {"Truncate(0)", now.Truncate(0)},
    {"UTC()", now.UTC()},
    {"Local()", now.Local()},
    {"Add(time.Second)", now.Add(time.Second)},
    {"AddDate(0,0,1)", now.AddDate(0, 0, 1)},
  } {
    fmt.Printf("%-18s monotonic present: %v\n", c.name, strings.Contains(c.t.String(), " m="))
  }

  b, _ := now.MarshalJSON()
  var back time.Time
  back.UnmarshalJSON(b)
  fmt.Println("after JSON round-trip:", strings.Contains(back.String(), " m="))

  fmt.Println("--- == vs Equal ---")
  wall := now.Round(0)
  fmt.Println("now == wall:   ", now == wall)
  fmt.Println("now.Equal(wall):", now.Equal(wall))
  fmt.Println("now.Compare(wall):", now.Compare(wall))
  fmt.Println("same instant?  ", now.UnixNano() == wall.UnixNano())
}
```

The timestamp varies per run; the `true`/`false` pattern does not. Output:

```
Now() has monotonic: 2026-10-26 10:25:59.778778 +0530 IST m=+0.000152834
--- what STRIPS the monotonic reading ---
Now()              monotonic present: true
Round(0)           monotonic present: false
Truncate(0)        monotonic present: false
UTC()              monotonic present: false
Local()            monotonic present: false
Add(time.Second)   monotonic present: true
AddDate(0,0,1)     monotonic present: false
after JSON round-trip: false
--- == vs Equal ---
now == wall:    false
now.Equal(wall): true
now.Compare(wall): 0
same instant?   true
```

Read the last four lines together: `now` and `wall` are the identical instant —
`UnixNano` agrees, `Compare` returns 0 — yet `==` is false, because the struct still holds
a monotonic reading `Round(0)` dropped.

| Strips the monotonic reading | Keeps it |
|---|---|
| `Round(d)` / `Truncate(d)` (including `Round(0)`) | `Add(d)` |
| `UTC()` / `Local()` / `In(loc)` | `Sub`, `Since`, `Until` (they *use* it) |
| `AddDate` | |
| `MarshalJSON` / `MarshalText` / `GobEncode` and any round-trip | |

`Round(0)` is the documented idiom for stripping it deliberately — the only reason to call
`Round` with a zero duration. `Sub` and `Since` use the monotonic reading when **both**
operands have one, so elapsed time is immune to wall-clock jumps:

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  start := time.Now()
  time.Sleep(20 * time.Millisecond)

  mono := time.Since(start)
  wall := time.Since(start.Round(0))

  fmt.Println("monotonic elapsed >= 20ms:", mono >= 20*time.Millisecond)
  fmt.Println("wall-clock elapsed >= 20ms:", wall >= 20*time.Millisecond)
  fmt.Println("both roughly equal:", (mono-wall).Abs() < 5*time.Millisecond)

  future := time.Now().Add(time.Hour)
  fmt.Println("Until(now+1h) > 59m:", time.Until(future) > 59*time.Minute)
}
```

This program prints assertions rather than durations, because the durations vary per run.
Output:

```
monotonic elapsed >= 20ms: true
wall-clock elapsed >= 20ms: true
both roughly equal: true
Until(now+1h) > 59m: true
```

Idiomatic: `defer func() { log.Println(time.Since(start)) }()` for timing. Never
`Round(0)`, `UTC()` or marshal a time you intend to subtract later — that silently
downgrades you to the wall clock.

## Duration is a typed int64 of nanoseconds

```go
type Duration int64
```

Being a named type with an underlying `int64` is what makes the arithmetic both pleasant
and occasionally hostile.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  fmt.Println("ns in Second/Minute/Hour:",
    int64(time.Second), int64(time.Minute), int64(time.Hour))

  d := 90*time.Minute + 30*time.Second + 250*time.Millisecond
  fmt.Println("\nd =", d)
  fmt.Println("String():      ", d.String())
  fmt.Println("Hours():       ", d.Hours())
  fmt.Println("Minutes():     ", d.Minutes())
  fmt.Println("Seconds():     ", d.Seconds())
  fmt.Println("Milliseconds():", d.Milliseconds())
  fmt.Println("Microseconds():", d.Microseconds())
  fmt.Println("Nanoseconds(): ", d.Nanoseconds())
  fmt.Println("Round(time.Minute):   ", d.Round(time.Minute))
  fmt.Println("Truncate(time.Minute):", d.Truncate(time.Minute))
  fmt.Println("Round(time.Hour):     ", d.Round(time.Hour))
  fmt.Println("Abs() of -d:          ", (-d).Abs())

  fmt.Println("\nString() formatting edge cases:")
  for _, x := range []time.Duration{
    0, 1, 1500, 1500 * time.Millisecond, 61 * time.Second, 3661 * time.Second,
    -2 * time.Hour, 25 * time.Hour, time.Duration(1<<63 - 1),
  } {
    fmt.Printf("  %-26s (%d ns)\n", x.String(), int64(x))
  }

  fmt.Println("\narithmetic:")
  fmt.Println("  2*time.Second + 500*time.Millisecond =", 2*time.Second+500*time.Millisecond)
  fmt.Println("  time.Hour / 3 =", time.Hour/3)
  fmt.Println("  time.Hour / time.Minute =", time.Hour/time.Minute)
  n := 5
  fmt.Println("  time.Duration(n) * time.Second =", time.Duration(n)*time.Second)
  fmt.Println("  n * int(time.Second) as Duration =", time.Duration(n*int(time.Second)))
}
```

Output:

```
ns in Second/Minute/Hour: 1000000000 60000000000 3600000000000

d = 1h30m30.25s
String():       1h30m30.25s
Hours():        1.5084027777777778
Minutes():      90.50416666666666
Seconds():      5430.25
Milliseconds(): 5430250
Microseconds(): 5430250000
Nanoseconds():  5430250000000
Round(time.Minute):    1h31m0s
Truncate(time.Minute): 1h30m0s
Round(time.Hour):      2h0m0s
Abs() of -d:           1h30m30.25s

String() formatting edge cases:
  0s                         (0 ns)
  1ns                        (1 ns)
  1.5µs                      (1500 ns)
  1.5s                       (1500000000 ns)
  1m1s                       (61000000000 ns)
  1h1m1s                     (3661000000000 ns)
  -2h0m0s                    (-7200000000000 ns)
  25h0m0s                    (90000000000000 ns)
  2562047h47m16.854775807s   (9223372036854775807 ns)

arithmetic:
  2*time.Second + 500*time.Millisecond = 2.5s
  time.Hour / 3 = 20m0s
  time.Hour / time.Minute = 60ns
  time.Duration(n) * time.Second = 5s
  n * int(time.Second) as Duration = 5s
```

| Method | Returns | Note |
|---|---|---|
| `String()` | `string` | `"1h30m30.25s"`; no days unit — 25 hours stays `25h0m0s` |
| `Hours()` / `Minutes()` / `Seconds()` | `float64` | fractional |
| `Milliseconds()` / `Microseconds()` / `Nanoseconds()` | `int64` | truncated |
| `Round(m)` | `Duration` | half away from zero |
| `Truncate(m)` | `Duration` | toward zero |

`String()` has no days unit, so a week prints as `168h0m0s` — formatting durations for
humans is your job. And `time.Hour / time.Minute` is `60ns`, not `60`: dividing a
`Duration` by a `Duration` yields a `Duration`, since the type system has no notion of
units cancelling. For a plain ratio write `int64(time.Hour / time.Minute)`.

### The n * time.Second compile error

A `Duration` constant is untyped, so `5 * time.Second` works. An `int` *variable* is not,
and the multiplication is rejected.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  n := 5
  d := n * time.Second
  fmt.Println(d)
}
```

This does not compile. Output:

```
# example.com/scratch
./main.go:10:8: invalid operation: n * time.Second (mismatched types int and time.Duration)
```

The fix is an explicit conversion of the *count*, not of the product:

```go
d := time.Duration(n) * time.Second
```

What is always wrong is `time.Duration(n)` alone meaning "n seconds" — that is n
*nanoseconds*, and it is the most common `time` bug:

```go
package main

import (
  "fmt"
  "time"
)

func timeout() time.Duration {
  return 30
}

func main() {
  var d time.Duration = 5
  fmt.Println("var d time.Duration = 5 ->", d)
  fmt.Println("return 30 from a Duration function ->", timeout())
}
```

Output:

```
var d time.Duration = 5 -> 5ns
return 30 from a Duration function -> 30ns
```

Both compile silently — an untyped constant converts to `Duration` without complaint — and
both mean nanoseconds. A config value in seconds must be multiplied by `time.Second`.

### ParseDuration

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  for _, s := range []string{
    "1h30m", "90m", "1.5h", "300ms", "-2h45m", "100ns", "2us", "2µs", "1h30m30.25s", "0",
  } {
    d, err := time.ParseDuration(s)
    fmt.Printf("%-13q -> %-14v err=%v\n", s, d, err)
  }

  fmt.Println()
  for _, s := range []string{"", "1", "90 m", "1h30", "abc", "1x", "1h-30m", "+", "1e3s"} {
    d, err := time.ParseDuration(s)
    fmt.Printf("%-9q -> %-6v err=%v\n", s, d, err)
  }
}
```

Output:

```
"1h30m"       -> 1h30m0s        err=<nil>
"90m"         -> 1h30m0s        err=<nil>
"1.5h"        -> 1h30m0s        err=<nil>
"300ms"       -> 300ms          err=<nil>
"-2h45m"      -> -2h45m0s       err=<nil>
"100ns"       -> 100ns          err=<nil>
"2us"         -> 2µs            err=<nil>
"2µs"         -> 2µs            err=<nil>
"1h30m30.25s" -> 1h30m30.25s    err=<nil>
"0"           -> 0s             err=<nil>

""        -> 0s     err=time: invalid duration ""
"1"       -> 0s     err=time: missing unit in duration "1"
"90 m"    -> 0s     err=time: unknown unit " m" in duration "90 m"
"1h30"    -> 0s     err=time: missing unit in duration "1h30"
"abc"     -> 0s     err=time: invalid duration "abc"
"1x"      -> 0s     err=time: unknown unit "x" in duration "1x"
"1h-30m"  -> 0s     err=time: unknown unit "h-" in duration "1h-30m"
"+"       -> 0s     err=time: invalid duration "+"
"1e3s"    -> 0s     err=time: unknown unit "e" in duration "1e3s"
```

Valid units: `ns`, `us` (or `µs`/`μs`), `ms`, `s`, `m`, `h`. No `d` for days, no
whitespace. The error text comes in three shapes — `invalid duration`, `missing unit`,
`unknown unit` — all quoting the input, so they are safe to show users. `"0"` is the one
bare number accepted. This is the right parser for a config value or `flag.Duration`.

## Time methods

| Method | Returns | Note |
|---|---|---|
| `Add(d)` | `Time` | keeps the monotonic reading |
| `Sub(u)` | `Duration` | monotonic when both have it |
| `AddDate(y, m, d)` | `Time` | **calendar** arithmetic; normalizes overflow |
| `Before(u)` / `After(u)` | `bool` | |
| `Equal(u)` | `bool` | same instant; the right comparison |
| `Compare(u)` | `int` | `-1`/`0`/`+1`, for `slices.SortFunc` |
| `IsZero()` | `bool` | `== time.Time{}` |
| `Unix()` / `UnixMilli()` / `UnixMicro()` / `UnixNano()` | `int64` | seconds / ms / µs / ns since epoch |
| `UTC()` / `Local()` / `In(loc)` | `Time` | same instant, different zone |
| `Year()` / `Month()` / `Day()` | `int`, `Month`, `int` | |
| `Hour()` / `Minute()` / `Second()` / `Nanosecond()` | `int` | |
| `Weekday()` / `YearDay()` | `Weekday`, `int` | `Weekday` is `Sunday`=0 |
| `ISOWeek()` | `(int, int)` | ISO-8601 year and week |
| `Clock()` | `(int, int, int)` | h, m, s in one call |
| `Date()` | `(int, Month, int)` | y, m, d in one call |
| `Truncate(d)` / `Round(d)` | `Time` | relative to the zero time, in UTC |

```go
package main

import (
  "fmt"
  "time"
)

var ref = time.Date(2026, time.March, 7, 15, 4, 5, 123456789, time.UTC)

func main() {
  t := ref
  fmt.Println("t =", t)

  fmt.Println("\n--- components ---")
  fmt.Println("Year/Month/Day:", t.Year(), t.Month(), t.Day())
  fmt.Println("Hour/Minute/Second/Nano:", t.Hour(), t.Minute(), t.Second(), t.Nanosecond())
  fmt.Println("Weekday/YearDay:", t.Weekday(), t.YearDay())
  y, w := t.ISOWeek()
  h, m, sec := t.Clock()
  yy, mm, dd := t.Date()
  fmt.Println("ISOWeek:", y, w, "Clock:", h, m, sec, "Date:", yy, mm, dd)
  zname, zoff := t.Zone()
  fmt.Println("Month/Weekday as int:", int(t.Month()), int(t.Weekday()),
    "Location:", t.Location(), "Zone:", zname, zoff)

  fmt.Println("\n--- arithmetic ---")
  fmt.Println("Add(90m):        ", t.Add(90*time.Minute).Format(time.RFC3339))
  fmt.Println("AddDate(1,2,3):  ", t.AddDate(1, 2, 3).Format(time.RFC3339))
  fmt.Println("AddDate(0,1,0) from Jan 31:",
    time.Date(2026, 1, 31, 0, 0, 0, 0, time.UTC).AddDate(0, 1, 0).Format("2006-01-02"))
  later := t.Add(3 * time.Hour)
  fmt.Println("Sub:             ", later.Sub(t))

  fmt.Println("\n--- comparison ---")
  fmt.Println("Before:", t.Before(later), "After:", t.After(later), "Equal:", t.Equal(t))
  fmt.Println("Compare:", t.Compare(later), later.Compare(t), t.Compare(t))
  var zero time.Time
  fmt.Println("zero Time:", zero, "IsZero:", zero.IsZero(), t.IsZero(),
    "zero.Unix():", zero.Unix())

  fmt.Println("\n--- unix ---")
  fmt.Println("Unix/UnixMilli/UnixNano:", t.Unix(), t.UnixMilli(), t.UnixNano())
  fmt.Println("time.Unix(1772895845, 0):", time.Unix(1772895845, 0).UTC().Format(time.RFC3339))

  fmt.Println("\n--- Truncate / Round for bucketing ---")
  fmt.Println("Truncate(time.Hour):    ", t.Truncate(time.Hour).Format(time.RFC3339Nano))
  fmt.Println("Truncate(24*time.Hour):  ", t.Truncate(24*time.Hour).Format(time.RFC3339))
}
```

Output:

```
t = 2026-03-07 15:04:05.123456789 +0000 UTC

--- components ---
Year/Month/Day: 2026 March 7
Hour/Minute/Second/Nano: 15 4 5 123456789
Weekday/YearDay: Saturday 66
ISOWeek: 2026 10 Clock: 15 4 5 Date: 2026 March 7
Month/Weekday as int: 3 6 Location: UTC Zone: UTC 0

--- arithmetic ---
Add(90m):         2026-03-07T16:34:05Z
AddDate(1,2,3):   2027-05-10T15:04:05Z
AddDate(0,1,0) from Jan 31: 2026-03-03
Sub:              3h0m0s

--- comparison ---
Before: true After: false Equal: true
Compare: -1 1 0
zero Time: 0001-01-01 00:00:00 +0000 UTC IsZero: true false zero.Unix(): -62135596800

--- unix ---
Unix/UnixMilli/UnixNano: 1772895845 1772895845123 1772895845123456789
time.Unix(1772895845, 0): 2026-03-07T15:04:05Z

--- Truncate / Round for bucketing ---
Truncate(time.Hour):     2026-03-07T15:00:00Z
Truncate(24*time.Hour):   2026-03-07T00:00:00Z
```

`AddDate(0, 1, 0)` from **January 31** gives **March 3**, not February 28: it increments
the month field and then normalizes February 31. For "last day of next month" use
`time.Date(y, m+2, 0, ...)` — day 0 is the last day of the previous month.

The zero `Time` is `0001-01-01 00:00:00 UTC`, `Unix()` `-62135596800`. Check for "unset"
with `IsZero()`, never `t.Unix() == 0` — that is 1970, a valid instant.

`Truncate`/`Round` work on the duration since the zero time **in UTC**, so
`Truncate(24*time.Hour)` is midnight UTC, not local midnight. For local day buckets use
`time.Date(t.Year(), t.Month(), t.Day(), 0, 0, 0, 0, t.Location())`.

## The reference-time layout system

Go has no `%Y`/`%m` escapes. You write the reference instant in the shape you want:

```
Mon Jan 2 15:04:05 MST 2006
 1   2  3     4  5   6   7     <- not literal; see the mnemonic below
```

The mnemonic is the digit sequence: month is `1`, day `2`, hour `3` (or `15` for 24-hour),
minute `4`, second `5`, year `6`, zone offset `7`. Written out in that order it is
`01/02 03:04:05PM '06 -0700`, which is exactly `time.Layout`.

| Component | Layout | Renders |
|---|---|---|
| Year | `2006` / `06` | `2026` / `26` |
| Month | `01` / `1` / `Jan` / `January` | `03` / `3` / `Mar` / `March` |
| Day | `02` / `2` / `_2` | `07` / `7` / ` 7` |
| Weekday | `Mon` / `Monday` | `Sat` / `Saturday` |
| Hour (24h) | `15` | `15` |
| Hour (12h) | `03` / `3` | `03` / `3` |
| Minute | `04` / `4` | `04` / `4` |
| Second | `05` / `5` | `05` / `5` |
| AM/PM | `PM` / `pm` | `PM` / `pm` |
| Fraction, fixed | `.000` / `.000000` / `.000000000` | `.123` / `.123456` / `.123456789` |
| Fraction, trimmed | `.9` / `.999` / `.999999999` | trailing zeros removed |
| Zone name | `MST` | `MST` |
| Zone offset | `-0700` / `-07:00` / `-07` / `-070000` | `-0700` / `-07:00` / `-07` / `-070000` |
| Offset, `Z` for UTC | `Z0700` / `Z07:00` / `Z070000` | `Z` when UTC, else the offset |

```go
package main

import (
  "fmt"
  "time"
)

var ref = time.Date(2026, time.March, 7, 15, 4, 5, 123456789, time.FixedZone("MST", -7*3600))

func main() {
  fmt.Println("fixed instant:", ref)

  fmt.Println("\n--- layout components ---")
  for _, l := range []string{
    "2006", "06", "01", "1", "Jan", "January", "02", "2", "_2",
    "Mon", "Monday", "15", "03", "04", "05",
    "PM", "pm", ".000", ".000000000", ".9", ".999", ".999999999",
    "MST", "Z07:00", "Z0700", "-07:00", "-07", "Z070000",
  } {
    fmt.Printf("  %-12q -> %q\n", l, ref.Format(l))
  }

  fmt.Println("\n--- predefined layouts ---")
  for _, p := range []struct{ name, layout string }{
    {"Layout", time.Layout},
    {"ANSIC", time.ANSIC},
    {"UnixDate", time.UnixDate},
    {"RFC822", time.RFC822},
    {"RFC1123", time.RFC1123},
    {"RFC1123Z", time.RFC1123Z},
    {"RFC3339", time.RFC3339},
    {"RFC3339Nano", time.RFC3339Nano},
    {"Kitchen", time.Kitchen},
    {"Stamp", time.Stamp},
    {"StampMilli", time.StampMilli},
    {"DateTime", time.DateTime},
    {"DateOnly", time.DateOnly},
    {"TimeOnly", time.TimeOnly},
  } {
    fmt.Printf("  %-12s %-30q %s\n", p.name, p.layout, ref.Format(p.layout))
  }

  fmt.Println("\n--- String() and GoString() ---")
  fmt.Println(ref.String())
  fmt.Println(ref.GoString())
  b, _ := ref.MarshalText()
  fmt.Println("MarshalText:", string(b))
}
```

Output:

```
fixed instant: 2026-03-07 15:04:05.123456789 -0700 MST

--- layout components ---
  "2006"       -> "2026"
  "06"         -> "26"
  "01"         -> "03"
  "1"          -> "3"
  "Jan"        -> "Mar"
  "January"    -> "March"
  "02"         -> "07"
  "2"          -> "7"
  "_2"         -> " 7"
  "Mon"        -> "Sat"
  "Monday"     -> "Saturday"
  "15"         -> "15"
  "03"         -> "03"
  "04"         -> "04"
  "05"         -> "05"
  "PM"         -> "PM"
  "pm"         -> "pm"
  ".000"       -> ".123"
  ".000000000" -> ".123456789"
  ".9"         -> ".1"
  ".999"       -> ".123"
  ".999999999" -> ".123456789"
  "MST"        -> "MST"
  "Z07:00"     -> "-07:00"
  "Z0700"      -> "-0700"
  "-07:00"     -> "-07:00"
  "-07"        -> "-07"
  "Z070000"    -> "-070000"

--- predefined layouts ---
  Layout       "01/02 03:04:05PM '06 -0700"   03/07 03:04:05PM '26 -0700
  ANSIC        "Mon Jan _2 15:04:05 2006"     Sat Mar  7 15:04:05 2026
  UnixDate     "Mon Jan _2 15:04:05 MST 2006" Sat Mar  7 15:04:05 MST 2026
  RFC822       "02 Jan 06 15:04 MST"          07 Mar 26 15:04 MST
  RFC1123      "Mon, 02 Jan 2006 15:04:05 MST" Sat, 07 Mar 2026 15:04:05 MST
  RFC1123Z     "Mon, 02 Jan 2006 15:04:05 -0700" Sat, 07 Mar 2026 15:04:05 -0700
  RFC3339      "2006-01-02T15:04:05Z07:00"    2026-03-07T15:04:05-07:00
  RFC3339Nano  "2006-01-02T15:04:05.999999999Z07:00" 2026-03-07T15:04:05.123456789-07:00
  Kitchen      "3:04PM"                       3:04PM
  Stamp        "Jan _2 15:04:05"              Mar  7 15:04:05
  StampMilli   "Jan _2 15:04:05.000"          Mar  7 15:04:05.123
  DateTime     "2006-01-02 15:04:05"          2026-03-07 15:04:05
  DateOnly     "2006-01-02"                   2026-03-07
  TimeOnly     "15:04:05"                     15:04:05

--- String() and GoString() ---
2026-03-07 15:04:05.123456789 -0700 MST
time.Date(2026, time.March, 7, 15, 4, 5, 123456789, time.Location("MST"))
MarshalText: 2026-03-07T15:04:05.123456789-07:00
```

The `.9` vs `.000` distinction catches people: `.999` **trims** trailing zeros (a whole
second renders nothing, not even the dot) while `.000` always emits three digits.
`RFC3339Nano` uses `.999999999`, which is why marshalled timestamps have varying
precision — correct RFC-3339, but awkward for string-sorting, where `.000000000` is what
you want.

Idiomatic: `time.RFC3339` on a wire, `time.DateOnly`/`time.DateTime` for humans, a named
constant for a custom layout.

## Parse and ParseInLocation

```go
package main

import (
  "errors"
  "fmt"
  "time"
)

func main() {
  fmt.Println("--- successful parses ---")
  for _, c := range []struct{ layout, value string }{
    {time.RFC3339, "2026-03-07T15:04:05Z"},
    {time.DateOnly, "2026-03-07"},
    {time.Kitchen, "3:04PM"},
  } {
    t, err := time.Parse(c.layout, c.value)
    fmt.Printf("  %-22q %-26q -> %v err=%v\n", c.layout, c.value, t.Format(time.RFC3339Nano), err)
  }

  fmt.Println("\n--- layout mismatches ---")
  for _, c := range []struct{ layout, value string }{
    {time.RFC3339, "2026-03-07 15:04:05"},
    {"2006-01-02", "07/03/2026"},
    {"2006-01-02", "2026-3-7"},
    {time.DateOnly, "2026-13-07"},
    {time.RFC3339, "2026-03-07T15:04:05"},
    {"2006-01-02", "2026-03-07 extra"},
  } {
    _, err := time.Parse(c.layout, c.value)
    fmt.Printf("  layout=%-22q value=%-22q\n    err: %v\n", c.layout, c.value, err)
  }

  fmt.Println("\n--- *time.ParseError fields ---")
  _, err := time.Parse(time.RFC3339, "2026-03-07 15:04:05")
  var pe *time.ParseError
  if errors.As(err, &pe) {
    fmt.Printf("  Layout=%q\n  Value=%q\n  LayoutElem=%q\n  ValueElem=%q\n  Message=%q\n",
      pe.Layout, pe.Value, pe.LayoutElem, pe.ValueElem, pe.Message)
  }

  fmt.Println("\n--- Parse defaults to UTC, ParseInLocation does not ---")
  t1, _ := time.Parse(time.DateTime, "2026-03-07 15:04:05")
  fmt.Println("  Parse:          ", t1, t1.Location())
  ny, err := time.LoadLocation("America/New_York")
  fmt.Println("  LoadLocation err:", err)
  if err == nil {
    t2, _ := time.ParseInLocation(time.DateTime, "2026-03-07 15:04:05", ny)
    fmt.Println("  ParseInLocation:", t2, t2.Location())
    fmt.Println("  same instant?   ", t1.Equal(t2))
    fmt.Println("  difference:     ", t2.Sub(t1))
  }
}
```

Output:

```
--- successful parses ---
  "2006-01-02T15:04:05Z07:00" "2026-03-07T15:04:05Z"     -> 2026-03-07T15:04:05Z err=<nil>
  "2006-01-02"           "2026-03-07"               -> 2026-03-07T00:00:00Z err=<nil>
  "3:04PM"               "3:04PM"                   -> 0000-01-01T15:04:00Z err=<nil>

--- layout mismatches ---
  layout="2006-01-02T15:04:05Z07:00" value="2026-03-07 15:04:05" 
    err: parsing time "2026-03-07 15:04:05" as "2006-01-02T15:04:05Z07:00": cannot parse " 15:04:05" as "T"
  layout="2006-01-02"           value="07/03/2026"          
    err: parsing time "07/03/2026" as "2006-01-02": cannot parse "07/03/2026" as "2006"
  layout="2006-01-02"           value="2026-3-7"            
    err: parsing time "2026-3-7" as "2006-01-02": cannot parse "3-7" as "01"
  layout="2006-01-02"           value="2026-13-07"          
    err: parsing time "2026-13-07": month out of range
  layout="2006-01-02T15:04:05Z07:00" value="2026-03-07T15:04:05" 
    err: parsing time "2026-03-07T15:04:05" as "2006-01-02T15:04:05Z07:00": cannot parse "" as "Z07:00"
  layout="2006-01-02"           value="2026-03-07 extra"    
    err: parsing time "2026-03-07 extra": extra text: " extra"

--- *time.ParseError fields ---
  Layout="2006-01-02T15:04:05Z07:00"
  Value="2026-03-07 15:04:05"
  LayoutElem="T"
  ValueElem=" 15:04:05"
  Message=""

--- Parse defaults to UTC, ParseInLocation does not ---
  Parse:           2026-03-07 15:04:05 +0000 UTC UTC
  LoadLocation err: <nil>
  ParseInLocation: 2026-03-07 15:04:05 -0500 EST America/New_York
  same instant?    false
  difference:      5h0m0s
```

The error text names the exact layout element that failed and the input fragment that did
not match: `cannot parse " 15:04:05" as "T"` says the input used a space where `RFC3339`
demands a literal `T`. `*time.ParseError` exposes all of it as fields.

Note `"2026-3-7"` against `"2006-01-02"`: the parser accepts a one-digit value for a
`01`-style element, but then `"3-7"` leaves the layout's `-` unmatched. Use `1` and `2`
in the layout for genuinely unpadded input.

`Parse` with no zone in the value defaults to **UTC**; `ParseInLocation` interprets it in
the given location — a five-hour difference for the same text, and `Equal` is false. On
local wall-clock input with no offset, `Parse` is silently wrong.

`time.Kitchen` parsing `"3:04PM"` yields year 0000 — unspecified layout fields get their
zero value, not today's date. Out-of-range components get their own messages
(`month out of range`, `day out of range`, `hour out of range`).

## Locations and DST

`time.LoadLocation` reads the system zoneinfo database. It works on this machine; in a
`FROM scratch` Docker image there is no tzdata and it fails — `import _ "time/tzdata"`
fixes that by embedding a copy (about 450KB) in the binary.

| Function | Returns |
|---|---|
| `time.UTC` | `*Location` for UTC |
| `time.Local` | `*Location` from `$TZ` or the system |
| `LoadLocation(name)` | `(*Location, error)` — IANA name like `America/New_York` |
| `FixedZone(name, offsetSeconds)` | `*Location` — no DST rules |

This is the example that matters: adding a day across a DST boundary.

```go
package main

import (
  "fmt"
  "time"
)

func main() {
  ny, err := time.LoadLocation("America/New_York")
  if err != nil {
    fmt.Println("tzdata unavailable:", err)
    return
  }

  before := time.Date(2026, time.March, 7, 9, 0, 0, 0, ny)
  fmt.Println("start (EST):", before.Format(time.RFC1123))

  fmt.Println("\n--- 24*time.Hour adds 24 real hours ---")
  for i := 1; i <= 2; i++ {
    t := before.Add(time.Duration(i) * 24 * time.Hour)
    fmt.Printf("  +%dd via Add: %s\n", i, t.Format("Mon 2006-01-02 15:04 MST"))
  }

  fmt.Println("\n--- AddDate adds calendar days ---")
  for i := 1; i <= 2; i++ {
    t := before.AddDate(0, 0, i)
    fmt.Printf("  +%dd via AddDate: %s\n", i, t.Format("Mon 2006-01-02 15:04 MST"))
  }

  fmt.Println("\n--- the spring-forward gap ---")
  gap := time.Date(2026, time.March, 8, 2, 30, 0, 0, ny)
  fmt.Println("  2:30 AM on the gap day normalizes to:", gap.Format("2006-01-02 15:04 MST"))

  fmt.Println("\n--- a 'day' is not 24 hours ---")
  d0 := time.Date(2026, time.March, 8, 0, 0, 0, 0, ny)
  d1 := time.Date(2026, time.March, 9, 0, 0, 0, 0, ny)
  fmt.Println("  Mar 8 -> Mar 9:", d1.Sub(d0))
  f0 := time.Date(2026, time.November, 1, 0, 0, 0, 0, ny)
  f1 := time.Date(2026, time.November, 2, 0, 0, 0, 0, ny)
  fmt.Println("  Nov 1 -> Nov 2:", f1.Sub(f0))

  fmt.Println("\n--- zones ---")
  tokyo, _ := time.LoadLocation("Asia/Tokyo")
  fixed := time.FixedZone("IST", 5*3600+1800)
  fmt.Println("  UTC:   ", before.UTC().Format(time.RFC3339))
  fmt.Println("  Tokyo: ", before.In(tokyo).Format(time.RFC3339))
  fmt.Println("  Fixed: ", before.In(fixed).Format(time.RFC3339), before.In(fixed).Location())
}
```

Output:

```
start (EST): Sat, 07 Mar 2026 09:00:00 EST

--- 24*time.Hour adds 24 real hours ---
  +1d via Add: Sun 2026-03-08 10:00 EDT
  +2d via Add: Mon 2026-03-09 10:00 EDT

--- AddDate adds calendar days ---
  +1d via AddDate: Sun 2026-03-08 09:00 EDT
  +2d via AddDate: Mon 2026-03-09 09:00 EDT

--- the spring-forward gap ---
  2:30 AM on the gap day normalizes to: 2026-03-08 01:30 EST

--- a 'day' is not 24 hours ---
  Mar 8 -> Mar 9: 23h0m0s
  Nov 1 -> Nov 2: 25h0m0s

--- zones ---
  UTC:    2026-03-07T14:00:00Z
  Tokyo:  2026-03-07T23:00:00+09:00
  Fixed:  2026-03-07T19:30:00+05:30 IST
```

A 9 AM daily job scheduled with `Add(24*time.Hour)` becomes a **10 AM** job the day DST
starts and stays wrong; `AddDate(0, 0, 1)` keeps it at 9 AM.

2:30 AM on 2026-03-08 in New York does not exist — the clock jumps 2:00 to 3:00 — and
`time.Date` resolves it to 1:30 AM EST rather than erroring. `time.Date` never fails, it
normalizes. `Sub` across those boundaries correctly returns 23h and 25h, so any code
computing "days between" as `d.Hours()/24` is wrong twice a year.

Idiomatic: store and transport instants in UTC; convert with `In(loc)` only at the display
or scheduling edge, and do calendar arithmetic with `AddDate` in the target location.

## time.Time and JSON

`time.Time` implements `json.Marshaler` as RFC 3339 with nanoseconds.

```go
package main

import (
  "encoding/json"
  "fmt"
  "time"
)

type event struct {
  Name string        `json:"name"`
  At   time.Time     `json:"at"`
  Took time.Duration `json:"took"`
}

func main() {
  ref := time.Date(2026, time.March, 7, 15, 4, 5, 123000000, time.UTC)

  e := event{Name: "deploy", At: ref, Took: 1500 * time.Millisecond}
  b, err := json.Marshal(e)
  fmt.Println("Marshal:", string(b), err)

  var back event
  err = json.Unmarshal(b, &back)
  fmt.Println("Unmarshal err:", err)
  fmt.Println("At equal:", back.At.Equal(ref), "Took:", back.Took)

  var bad event
  err = json.Unmarshal([]byte(`{"at":"2026-03-07 15:04:05"}`), &bad)
  fmt.Println("non-RFC3339 err:", err)

  err = json.Unmarshal([]byte(`{"took":"1.5s"}`), &bad)
  fmt.Println("duration as string err:", err)

  noNano := time.Date(2026, time.March, 7, 15, 4, 5, 0, time.UTC)
  b, _ = json.Marshal(noNano)
  fmt.Println("zero nanos:", string(b))

  var zero time.Time
  b, _ = json.Marshal(zero)
  fmt.Println("zero Time:", string(b))

  type wrap struct {
    T time.Time `json:"t,omitempty"`
  }
  b, _ = json.Marshal(wrap{})
  fmt.Println("omitempty on zero Time:", string(b))

  b, _ = json.Marshal(map[string]time.Duration{"timeout": 30 * time.Second})
  fmt.Println("Duration marshals as int64 ns:", string(b))
}
```

Output:

```
Marshal: {"name":"deploy","at":"2026-03-07T15:04:05.123Z","took":1500000000} <nil>
Unmarshal err: <nil>
At equal: true Took: 1.5s
non-RFC3339 err: parsing time "2026-03-07 15:04:05" as "2006-01-02T15:04:05Z07:00": cannot parse " 15:04:05" as "T"
duration as string err: json: cannot unmarshal string into Go struct field event.took of type time.Duration
zero nanos: "2026-03-07T15:04:05Z"
zero Time: "0001-01-01T00:00:00Z"
omitempty on zero Time: {"t":"0001-01-01T00:00:00Z"}
Duration marshals as int64 ns: {"timeout":30000000000}
```

Three real surprises.

`time.Duration` has **no** JSON methods, so it marshals as its underlying `int64` of
nanoseconds — `1500000000`, not `"1.5s"` — and unmarshalling `"1.5s"` fails. An API that
accepts `"30s"` needs a wrapper type with `MarshalJSON`/`UnmarshalJSON` (13.7).

`omitempty` does **not** omit a zero `time.Time`: it only understands false, 0, nil
pointers/interfaces and empty strings/slices/maps — a struct is never "empty" to it. Use
`*time.Time` when absence must be representable.

Nanoseconds are trimmed on output — `.123` for 123ms, nothing at all for a whole second —
so a sorted-by-string index over these timestamps will mis-sort.

## Quick Reference

| Need | Call |
|---|---|
| Now / elapsed / until | `time.Now()`, `time.Since(t)`, `time.Until(t)` |
| Build an instant | `time.Date(y, m, d, h, mi, s, ns, loc)` |
| Compare instants | `a.Equal(b)`, `a.Before(b)`, `a.Compare(b)` |
| Detect unset | `t.IsZero()` |
| Add a duration | `t.Add(d)` |
| Add calendar days/months | `t.AddDate(0, 0, 1)` |
| Format for a wire | `t.Format(time.RFC3339)` |
| Parse a wire timestamp | `time.Parse(time.RFC3339, s)` |
| Parse a local wall-clock time | `time.ParseInLocation(layout, s, loc)` |
| Parse a duration string | `time.ParseDuration("1h30m")` |
| Strip the monotonic reading | `t.Round(0)` |
| Local-day bucket | `time.Date(t.Year(), t.Month(), t.Day(), 0,0,0,0, t.Location())` |
| Seconds as a Duration from an int | `time.Duration(n) * time.Second` |

| Trap | Reality |
|---|---|
| `t1 == t2` | compares monotonic readings too; use `Equal` |
| `t.Add(24*time.Hour)` for "tomorrow" | drifts an hour across DST; use `AddDate` |
| `time.Duration(n)` meaning n seconds | it is n **nanoseconds** |
| `n * time.Second` with `n` an `int` | compile error: mismatched types |
| `time.Hour / time.Minute` | `60ns`, not `60` |
| `t.Unix() == 0` as "unset" | that is 1970; use `IsZero()` |
| `time.Parse` on local input | defaults to UTC; use `ParseInLocation` |
| `omitempty` on a `time.Time` | never omits; use `*time.Time` |
| A `time.Duration` field in JSON | marshals as raw nanoseconds |
| `AddDate(0,1,0)` from Jan 31 | normalizes to March 3 |

## Common Mistakes

**Comparing times with `==`.** The monotonic reading is part of the struct, so two values
for the same instant can differ; use `Equal` or `Compare`.

**Adding `24*time.Hour` to get the next day.** Across a DST transition the wall-clock time
shifts by an hour; `AddDate(0, 0, 1)` is the calendar operation.

**Writing `time.Duration(seconds)` for a config value in seconds.** That is nanoseconds —
multiply by `time.Second`.

**Using `time.Parse` on a timestamp with no offset.** It silently assumes UTC; use
`ParseInLocation` when the input is local wall-clock time.

**Tagging a `time.Time` field `omitempty`.** It has no effect on structs; make the field
`*time.Time` if absence must be distinguishable from the zero time.

## Exercises

1. Write `func businessDaysBetween(a, b time.Time) int` with `AddDate` and `Weekday`.
   Table-test it across a DST boundary in `America/New_York`.
2. Implement `type Seconds time.Duration` whose JSON methods round-trip `"30s"` and
   `"1h30m"` via `ParseDuration` and reject a bare number with a clear error.
3. Write `func parseFlexible(s string) (time.Time, error)` trying RFC3339, `DateTime`,
   `DateOnly` and `"02/01/2006"` in order; on failure return an error naming every layout
   tried, with `LayoutElem` from each `*time.ParseError`.
4. Build `func bucketsLocal(ts []time.Time, loc *time.Location) map[string]int` grouping
   into local calendar days, and show that `Truncate(24*time.Hour)` buckets differently.
5. Write a `go/ast` check that fails if any `time.Time` is compared with `==` in a
   package, and explain which comparisons it must still allow.

---

[← 13.3 bufio: Scanner, Reader, and Writer](03-bufio.md) · [Course Index](../README.md) · [13.5 time: Timers, Tickers, and After →](05-timers.md)
