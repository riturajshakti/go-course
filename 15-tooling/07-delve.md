[← 15.6 pprof Profiling](06-pprof.md) · [Course Index](../README.md) · [15.8 go generate and Workspaces →](08-generate-workspaces.md)

# 15.7 — Delve, Briefly

## What You'll Learn

- `dlv debug`, `dlv test`, `dlv exec`, `dlv attach` — four ways in
- A real session finding a real off-by-one, as a terminal transcript
- `goroutines` is the command no print statement can replace
- `-ldflags "-s -w"` removes exactly what delve needs

## Coming From Another Language

| Other | Go |
|---|---|
| `node --inspect` + Chrome DevTools | `dlv debug` in the terminal, or an editor over the same API |
| `pdb.set_trace()` in the source | breakpoints set from outside; the source stays clean |
| `gdb ./prog` | `dlv`; gdb mostly cannot read Go's goroutines or maps |
| no first-class concurrency view | `goroutines` lists all of them with their current line |

## Why it exists, honestly

`fmt.Println` debugging is fine, extremely common in Go, and frequently faster
than opening a debugger — the compile is a second and you already know roughly
where to look. Delve earns its keep in exactly three situations: inspecting
**goroutine state** in a hung or deadlocked program, post-mortem analysis of a
**core dump**, and stepping through code whose control flow you genuinely do not
understand. For "what is this variable", print it.

## The buggy program

```go
package main

import "fmt"

type order struct {
  ID    int
  Items []int
}

func total(o order) int {
  sum := 0
  for i := 1; i < len(o.Items); i++ {
    sum += o.Items[i]
  }
  return sum
}

func main() {
  orders := []order{
    {ID: 1, Items: []int{10, 20, 30}},
    {ID: 2, Items: []int{5, 5}},
  }
  for _, o := range orders {
    fmt.Printf("order %d total %d\n", o.ID, total(o))
  }
}
```

```bash
go run .
```

Output:

```
order 1 total 50
order 2 total 5
```

50 and 5, not 60 and 10. It compiles, `go vet` is silent, and nothing in the
output says which element went missing.

## A real session

`dlv debug` compiles the package with optimisations and inlining off
(`-gcflags="all=-N -l"`, 15.1) and drops you at a prompt:

```bash
dlv debug
```

Real transcript. Addresses, thread IDs, and goroutine IDs vary on every run:

```
Type 'help' for list of commands.
(dlv) break main.total
Breakpoint 1 set at 0x1022dd6bc for main.total() ./main.go:10
(dlv) break main.go:13
Breakpoint 2 set at 0x1022dd704 for main.total() ./main.go:13
(dlv) continue
> [Breakpoint 1] main.total() ./main.go:10 (hits goroutine(1):1 total:1) (PC: 0x1022dd6bc)
     5:	type order struct {
     6:		ID    int
     7:		Items []int
     8:	}
     9:	
=>  10:	func total(o order) int {
    11:		sum := 0
    12:		for i := 1; i < len(o.Items); i++ {
    13:			sum += o.Items[i]
    14:		}
    15:		return sum
(dlv) args
o = main.order {ID: 1, Items: []int len: 3, cap: 3, [...]}
~r0 = 118153689611608
(dlv) locals
(no locals)
(dlv) next
> main.total() ./main.go:11 (PC: 0x1022dd6dc)
    10:	func total(o order) int {
=>  11:		sum := 0
    12:		for i := 1; i < len(o.Items); i++ {
(dlv) next
> main.total() ./main.go:12 (PC: 0x1022dd6e0)
    11:		sum := 0
=>  12:		for i := 1; i < len(o.Items); i++ {
    13:			sum += o.Items[i]
(dlv) print o.Items
[]int len: 3, cap: 3, [10,20,30]
(dlv) print len(o.Items)
3
(dlv) print i
Command failed: could not find symbol value for i
(dlv) clear 1
Breakpoint 1 cleared at 0x1022dd6bc for main.total() ./main.go:10
(dlv) continue
> [Breakpoint 2] main.total() ./main.go:13 (hits goroutine(1):1 total:1) (PC: 0x1022dd704)
    11:		sum := 0
    12:		for i := 1; i < len(o.Items); i++ {
=>  13:			sum += o.Items[i]
    14:		}
(dlv) locals
sum = 0
i = 1
(dlv) print o.Items[0]
10
(dlv) bt
0  0x00000001022dd704 in main.total
   at ./main.go:13
1  0x00000001022dd928 in main.main
   at ./main.go:24
2  0x0000000102268990 in runtime.main
   at /opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/proc.go:302
3  0x000000010229d7d4 in runtime.goexit
   at /opt/homebrew/Cellar/go/1.27.1/libexec/src/runtime/asm_arm64.s:1039
(dlv) goroutines
* Goroutine 1 - User: ./main.go:13 main.total (0x1022dd704) (thread 15120246)
  Goroutine 2 - User: .../runtime/proc.go:475 runtime.gopark (0x102298310) [force gc (idle)]
  Goroutine 3 - User: .../runtime/proc.go:475 runtime.gopark (0x102298310) [GOMAXPROCS updater (idle)]
  Goroutine 4 - User: .../runtime/proc.go:475 runtime.gopark (0x102298310) [finalizer wait]
  Goroutine 17 - User: .../runtime/proc.go:475 runtime.gopark (0x102298310) [GC sweep wait]
  Goroutine 18 - User: .../runtime/proc.go:475 runtime.gopark (0x102298310) [GC scavenge wait]
[6 goroutines]
(dlv) exit
```

The bug is in the `locals` output after the second `continue`: on the **first**
execution of line 13, `sum = 0` and `i = 1`. The loop starts at index 1, so
`o.Items[0]` — which `print` confirms is 10 — is never added. `i := 1` should
be `i := 0`.

Three details worth noticing. `args` showed `~r0`, the unnamed return slot,
holding garbage because the function has not returned yet. `print i` failed at
line 12 because `i` is not yet in scope at the top of the `for` statement —
delve reports lexical scope accurately, which is a feature, not a bug.
And `goroutines` listed six goroutines in a program that started none: the GC
sweeper, scavenger, finalizer, and the Go 1.27 GOMAXPROCS updater are always
there.

## Conditional breakpoints

Named breakpoints plus `condition` is how you skip the first 10,000 iterations:

```
(dlv) break b1 main.go:13
Breakpoint b1 set at 0x102b31704 for main.total() ./main.go:13
(dlv) condition b1 sum > 15
(dlv) continue
> [b1] main.total() ./main.go:13 (hits goroutine(1):1 total:1) (PC: 0x102b31704)
    12:		for i := 1; i < len(o.Items); i++ {
=>  13:			sum += o.Items[i]
(dlv) locals
sum = 20
i = 2
(dlv) print o.ID
1
```

`sum > 15` first held on the second iteration, so it stopped there. `on b1 print
sum` attaches a command that runs at each hit without stopping — a print
statement you can add and remove without recompiling, which is usually what you
actually wanted.

## Debugging a test

`dlv test` builds the test binary and debugs it. Same bug, through its test:

```go
package main

import "testing"

func TestTotal(t *testing.T) {
  got := total(order{ID: 1, Items: []int{10, 20, 30}})
  if got != 60 {
    t.Fatalf("total = %d, want 60", got)
  }
}
```

```bash
go test ./...
```

Output:

```
    total_test.go:8: total = 50, want 60
FAIL	example.com/scratch	1.881s
FAIL
```

```bash
dlv test
```

```
(dlv) break main.total
Breakpoint 1 set at 0x102f8668c for example.com/scratch.total() ./main.go:10
(dlv) continue
> [Breakpoint 1] example.com/scratch.total() ./main.go:10 (hits goroutine(5):1 total:1) (PC: 0x102f8668c)
=>  10:	func total(o order) int {
(dlv) args
o = example.com/scratch.order {ID: 1, Items: []int len: 3, cap: 3, [...]}
~r0 = 0
```

Two differences from `dlv debug`. Symbols are qualified by the **import path**
(`example.com/scratch.total`, not `main.total`) — though `break main.total`
still resolved it. And the breakpoint hit on **goroutine 5**, because `go test`
runs each test in its own goroutine. Pass test flags after `--`:
`dlv test -- -run TestTotal -v`.

## Debugging goroutines

This is the part no print statement replaces. In a hung program, `goroutines`
plus `-with` finds the blocked one:

```
(dlv) goroutines -with running
(dlv) goroutines -s             # sorted by goroutine ID
(dlv) goroutine 42              # switch to it
(dlv) bt                        # its stack
(dlv) frame 2                   # select a caller's frame
(dlv) locals                    # that frame's variables
```

Switch to a goroutine, walk its frames with `frame N`, and read the locals that
show what it is waiting for. 12.5's deadlock and leak hunt is exactly this
workflow, and `goroutineleak` (15.6) narrows the candidate list first.

For a process already running — including one in a container — attach instead
of restarting:

```bash
dlv attach $(pgrep myservice)
dlv exec ./myservice -- --config=prod.yaml
dlv debug --headless --listen=:2345 --api-version=2 ./cmd/app
```

The `--headless` form is what editors and remote debugging use. `dlv core
./myservice core.1234` opens a core dump post-mortem, which with
`GOTRACEBACK=crash` is how you debug a panic that only happens in production.

## Stripped binaries do not work

15.5's release flags remove the DWARF information delve reads:

```bash
go build -ldflags "-s -w" -o stripped .
dlv exec ./stripped
```

Output:

```
Warning: no debug info found, some functionality will be missing such as stack traces and variable evaluation.
Type 'help' for list of commands.
(dlv) break main.total
Command failed: location "main.total" not found
(dlv) continue
order 1 total 50
order 2 total 5
Process 52174 has exited with status 0
```

No breakpoints, no variables, no stack traces. If you may need to debug a
release build, keep an unstripped binary from the *same commit* and use
`dlv exec` against a copy, or build without `-w` and accept the size.

## Editor integration

**VS Code**: the official Go extension speaks delve's headless API; `F5` with a
`launch.json` of `{"mode": "debug", "program": "${workspaceFolder}"}` is the
whole setup, and breakpoints in the gutter map to the commands above.
**GoLand**: its own frontend over delve, with goroutine and channel views that
are genuinely better than the CLI's.

## Quick Reference

| Command | Short | Does |
|---|---|---|
| `break FUNC` / `break file:line` | `b` | set a breakpoint |
| `break name file:line` | | named breakpoint |
| `condition name EXPR` | `cond` | only stop when EXPR is true |
| `on name print x` | | run a command at each hit, do not stop |
| `clear N` / `clearall` | | remove breakpoints |
| `breakpoints` | `bp` | list them |
| `continue` | `c` | run to the next breakpoint |
| `next` | `n` | step over |
| `step` | `s` | step into |
| `stepout` | `so` | finish the current function |
| `restart` | `r` | rerun from the start |
| `print EXPR` | `p` | evaluate a Go expression |
| `set x = 5` | | assign to a variable |
| `locals` / `args` | | variables in scope / parameters |
| `whatis EXPR` | | its type |
| `list` / `list file:line` | `l` | show source |
| `bt` | | stack trace |
| `frame N` | | select a frame |
| `goroutines` | `grs` | all goroutines |
| `goroutine N` | `gr` | switch to one |
| `threads` / `thread N` | | OS threads |
| `exit` | `q` | quit |

| Entry point | For |
|---|---|
| `dlv debug [pkg]` | build and debug a main package |
| `dlv test [pkg]` | build and debug a test binary |
| `dlv exec ./bin` | an already-built binary |
| `dlv attach PID` | a running process |
| `dlv core ./bin core` | a core dump, post-mortem |
| `dlv debug --headless --listen=:2345` | for an editor or remote session |

## Common Mistakes

**Debugging a `-ldflags "-s -w"` binary.** The DWARF info is gone; nothing
resolves. Keep an unstripped build of the same commit.

**Expecting optimised code to step sensibly.** `dlv debug` passes `-N -l` for
you, but `dlv exec` on a normally-built binary gives inlined frames and
"optimized out" variables.

**Printing a loop variable before its scope opens.** `print i` at the `for`
line fails; step onto the body first.

**Reaching for delve when a print would do.** For "what is this value", recompile
with a `fmt.Printf`; it is faster. Save delve for goroutine state and
post-mortems.

## Exercises

1. Reproduce the session above and find the off-by-one yourself. Then use
   `set i = 0` at the breakpoint and `continue` to confirm the expected total
   without editing the source.
2. Write a program that deadlocks on two mutexes taken in opposite orders. Run
   it under `dlv`, interrupt it, and use `goroutines` plus `goroutine N` and
   `bt` to identify both halves of the cycle.
3. Use a conditional breakpoint to stop on the 500th iteration of a loop.
   Then do the same with `on` instead, printing the value without stopping,
   and say which you would reach for.
4. Debug a failing table-driven test with `dlv test -- -run TestX/subtest`.
   Confirm which goroutine your breakpoint lands on and explain why.
5. Build the same program stripped and unstripped. Confirm that `print` and
   `bt` work on one and not the other, then check whether a panic traceback
   still shows function names in the stripped build (15.5).

---

[← 15.6 pprof Profiling](06-pprof.md) · [Course Index](../README.md) · [15.8 go generate and Workspaces →](08-generate-workspaces.md)
