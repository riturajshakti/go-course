[← 15.4 Build Tags and Conditional Compilation](04-build-tags.md) · [Course Index](../README.md) · [15.6 pprof Profiling →](06-pprof.md)

# 15.5 — Cross-Compilation and Static Binaries

## What You'll Learn

- Five platforms from one machine with two environment variables and no SDK
- `CGO_ENABLED=0` is what makes a Linux binary statically linked
- `-trimpath` is the difference between reproducible and not — proven with sha256
- `-s -w` buys ~35% of the size and costs you delve

## Coming From Another Language

| Other | Go |
|---|---|
| Docker build per target platform | `GOOS=linux GOARCH=arm64 go build` on your laptop |
| `cross` / `rustup target add` per triple | 47 targets available out of the box, nothing to install |
| `pkg`/PyInstaller bundling an interpreter | the runtime is linked in; one file, no dependencies |
| musl vs glibc static-linking pain | `CGO_ENABLED=0` and it is genuinely static |
| `.dll`/`.so` shipped alongside | one binary, scp it, run it |

## Five platforms, one command each

The program touches three things that behave differently per platform —
`runtime.GOOS`, `os/user` (historically cgo-backed), and DNS resolution:

```go
package main

import (
  "fmt"
  "net"
  "os/user"
  "runtime"
)

var version = "dev"

func main() {
  fmt.Printf("xc %s on %s/%s\n", version, runtime.GOOS, runtime.GOARCH)
  if u, err := user.Current(); err == nil {
    fmt.Println("user:", u.Username)
  }
  if ips, err := net.LookupHost("localhost"); err == nil {
    fmt.Println("localhost:", ips)
  }
}
```

```bash
go run .
```

Output (on the host, darwin/arm64):

```
xc dev on darwin/arm64
user: raj
localhost: [::1 127.0.0.1]
```

Now all five targets:

```bash
for t in darwin/arm64 darwin/amd64 linux/amd64 linux/arm64 windows/amd64; do
  os=${t%/*}; arch=${t#*/}
  ext=""; [ "$os" = windows ] && ext=.exe
  GOOS=$os GOARCH=$arch CGO_ENABLED=0 go build -o dist/xc-$os-$arch$ext .
done
ls -l dist
```

Output (sizes vary with Go version):

```
total 33080
-rwxr-xr-x@ 1 raj  staff  3475312 Oct 26 13:34 xc-darwin-amd64
-rwxr-xr-x@ 1 raj  staff  3341874 Oct 26 13:34 xc-darwin-arm64
-rwxr-xr-x@ 1 raj  staff  3390886 Oct 26 13:34 xc-linux-amd64
-rwxr-xr-x@ 1 raj  staff  3246219 Oct 26 13:35 xc-linux-arm64
-rwxr-xr-x@ 1 raj  staff  3477504 Oct 26 13:35 xc-windows-amd64.exe
```

Five real binaries, five seconds, no SDK, no container, no toolchain install.
This is Go's single most underrated property and the reason its release
engineering is boring.

```bash
for f in dist/*; do printf "%-28s " "$(basename $f)"; file -b "$f"; done
```

Output:

```
xc-darwin-amd64              Mach-O 64-bit executable x86_64
xc-darwin-arm64              Mach-O 64-bit executable arm64
xc-linux-amd64               ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=eQOPOt4crjMVDc50q7Rr/..., BuildID[sha1]=30656d0e54a2d804fa1aeadb3ddc7e17234bbba8, with debug_info, not stripped
xc-linux-arm64               ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, Go BuildID=e-_gGpXFEzFGkPA3qzcx/..., BuildID[sha1]=6f2d86f82dbdb51aca530fd3c725ebe3a7abdeb1, with debug_info, not stripped
xc-windows-amd64.exe         PE32+ executable (console) x86-64, for MS Windows
```

Three container formats — Mach-O, ELF, PE32+ — and two words in the Linux lines
that matter: **statically linked**, and **not stripped**. Hold both.

| Target | Format | Static | Size |
|---|---|---|---|
| darwin/arm64 | Mach-O arm64 | no (always links libSystem) | 3,341,874 |
| darwin/amd64 | Mach-O x86_64 | no | 3,475,312 |
| linux/amd64 | ELF x86-64 | yes | 3,390,886 |
| linux/arm64 | ELF aarch64 | yes | 3,246,219 |
| windows/amd64 | PE32+ console | yes | 3,477,504 |

## I cannot run the Linux binary here

Being plain about it, because the honest version is more useful than a faked
transcript:

```bash
./dist/xc-linux-amd64
```

Output:

```
exec format error: ./dist/xc-linux-amd64
```

That is the kernel refusing an ELF binary on a Mach-O system, as it should.
There is no Docker daemon in this environment either, so **nothing below shows
a Linux binary actually executing.** Everything about the Linux and Windows
binaries in this lesson is established by inspecting the files
(`file`, `ls -l`, `go version -m`, sha256), which is exactly what a release
pipeline's verification step does anyway. To run one yourself:

```bash
docker run --rm -v "$PWD/dist:/d" --platform linux/amd64 alpine /d/xc-linux-amd64
```

`alpine` works for this binary precisely because it is static — a dynamically
linked Go binary would fail there with a missing-interpreter error, since
Alpine ships musl rather than glibc. That is the whole practical point of the
next section.

## go tool dist list

```bash
go tool dist list | wc -l
go tool dist list | head -16
```

Output:

```
      47
aix/ppc64
android/386
android/amd64
android/arm
android/arm64
darwin/amd64
darwin/arm64
dragonfly/amd64
freebsd/386
freebsd/amd64
freebsd/arm
freebsd/arm64
illumos/amd64
ios/amd64
ios/arm64
js/wasm
```

47 pairs on Go 1.27, including `js/wasm` and `wasip1/wasm`. `go tool dist list
-json` adds a `FirstClass` field marking the ports with full CI coverage and
binary releases: linux and windows on 386/amd64/arm/arm64, and darwin on
amd64/arm64. A non-first-class port builds but has weaker guarantees.

## CGO_ENABLED=0 and static linking

With cgo on, Go links against the system C library for a handful of
packages — most importantly `net`'s DNS resolution and `os/user`'s passwd
lookups. That makes the binary dynamically linked, which makes it
non-portable across libc implementations.

You can see the swap in the file selection (15.4's `go list` technique applied
to the stdlib). For `GOOS=linux`:

```bash
GOOS=linux CGO_ENABLED=1 go list -f 'cgofiles={{.CgoFiles}}' net
```

Output:

```
cgofiles=[cgo_linux.go cgo_resnew.go cgo_socknew.go cgo_unix_cgo.go cgo_unix_cgo_res.go]
```

```bash
GOOS=linux CGO_ENABLED=0 go list -f 'cgofiles={{.CgoFiles}}' net
```

Output:

```
cgofiles=[]
```

And the Go file that takes their place:

```bash
GOOS=linux CGO_ENABLED=1 go list -f '{{.GoFiles}}' net | tr ' ' '\n' | grep cgo
GOOS=linux CGO_ENABLED=0 go list -f '{{.GoFiles}}' net | tr ' ' '\n' | grep cgo
```

Output:

```
cgo_unix.go
netcgo_off.go
cgo_stub.go
netcgo_off.go
```

`cgo_unix.go` (which calls `getaddrinfo`) is replaced by `cgo_stub.go`, and
the pure-Go resolver takes over. The `netgo` build tag forces the same
substitution **without** disabling cgo globally:

```bash
GOOS=linux CGO_ENABLED=1 go list -tags netgo -f '{{.GoFiles}}' net | tr ' ' '\n' | grep cgo
```

Output:

```
cgo_stub.go
netcgo_off.go
```

Identical to the `CGO_ENABLED=0` selection. `osusergo` does the same for
`os/user`. Use them when you need cgo for *something else* (a SQLite driver,
say) but still want a self-contained resolver:

```bash
CGO_ENABLED=1 go build -tags "netgo,osusergo" -ldflags '-extldflags "-static"' .
```

The resolver swap is a behaviour change, not just a linking one. The cgo
resolver uses `getaddrinfo`, so it honours `/etc/nsswitch.conf`, LDAP, mDNS,
and whatever else NSS is configured for. The pure-Go resolver reads
`/etc/resolv.conf` and `/etc/hosts` directly and speaks DNS itself. In a
container that is almost always what you want; on a corporate host with
sssd-backed name resolution it may not be. `GODEBUG=netdns=go+1` /
`netdns=cgo+1` prints which one is in use at runtime.

### On darwin, cgo makes no difference here

```bash
otool -L dist/xc-darwin-arm64
```

Output:

```
dist/xc-darwin-arm64:
	/usr/lib/libSystem.B.dylib (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libresolv.9.dylib (compatibility version 0.0.0, current version 0.0.0)
```

`CGO_ENABLED=1` gives byte-identical output and, as it happens, a
byte-identical binary size (3,341,874 either way). macOS has no supported
static linking at all: Go on darwin always goes through `libSystem` for
syscalls, because Apple does not provide a stable syscall ABI. So
`CGO_ENABLED=0` is a *Linux* concern. On darwin it changes which resolver you
get, not whether you link.

### Cross-compiling with cgo needs a cross C compiler

```bash
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -o /dev/null .
```

This is an intentional failure. Output (truncated):

```
# runtime/cgo
gcc_amd64.S:27:8: error: unknown token in expression
 pushq %rbx
       ^
```

Apple's clang was handed x86-64 assembly and refused it. With cgo in the
picture, cross-compiling needs a real cross toolchain (`CC=x86_64-linux-musl-gcc`
or Zig's `zig cc -target x86_64-linux-musl`), and the one-command story
evaporates. That is the strongest practical argument for keeping cgo out of a
service: `CGO_ENABLED=0` is the setting that keeps cross-compilation free.

## -s -w: 35% of the size

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags "-s -w" -o dist/xc-linux-amd64-sw .
ls -l dist/xc-linux-amd64 dist/xc-linux-amd64-sw
file -b dist/xc-linux-amd64-sw
```

Output:

```
3390886 dist/xc-linux-amd64
2228384 dist/xc-linux-amd64-sw
ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=mraebkklJg6X7zMt9W69/..., BuildID[sha1]=ea1885b2c744dc123828c6671f5e6e986d9c73ae, stripped
```

3.39 MB → 2.23 MB, a 34% reduction, and `file` now says **stripped** instead of
`with debug_info, not stripped`.

What you lose: `-w` removes the DWARF debug info, so **delve cannot set
breakpoints or read locals** (15.7 shows that failure for real), and `-s`
removes the symbol table. Panic tracebacks still show function names, because
Go keeps its own `pclntab` for the runtime's own traceback and it is not part
of the symbol table — a detail that surprises people coming from C.

The rule: `-s -w` on release binaries that ship to users, never on anything you
might need to attach a debugger to. If you are already producing a container
image, 2.23 MB versus 3.39 MB on a compressed layer is less exciting than it
sounds; `-trimpath` matters more.

## -trimpath and reproducible builds

Same source, same flags, twice:

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /tmp/r1 .
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /tmp/r2 .
shasum -a 256 /tmp/r1 /tmp/r2
```

Output:

```
83f675094cfbf9bbc3e6bd405e8522dace15c452ec4627a366c54cfd1ebbdf97  /tmp/r1
83f675094cfbf9bbc3e6bd405e8522dace15c452ec4627a366c54cfd1ebbdf97  /tmp/r2
```

Now copy the source tree to a different directory and build it **without**
`-trimpath` from each location:

```bash
cp -r . /tmp/othersrc
(cd /tmp/othersrc && GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags "-s -w" -o /tmp/r3 .)
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags "-s -w" -o /tmp/r4 .
shasum -a 256 /tmp/r3 /tmp/r4
```

Output:

```
41ba55197ea056a8e52c06c92a4a81df157a97a4eca1d9f03a46119fab70e5aa  /tmp/r3
d946e2494157aed35a91b1faa3874b7eb5bc18b660b9fc52e0efc85a0fe6a264  /tmp/r4
```

Different bytes from identical source — the absolute build directory is baked
in. Add `-trimpath` and the difference disappears:

```bash
(cd /tmp/othersrc && GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /tmp/r5 .)
shasum -a 256 /tmp/r1 /tmp/r5
```

Output:

```
83f675094cfbf9bbc3e6bd405e8522dace15c452ec4627a366c54cfd1ebbdf97  /tmp/r1
83f675094cfbf9bbc3e6bd405e8522dace15c452ec4627a366c54cfd1ebbdf97  /tmp/r5
```

Byte-identical across directories. That is the reproducibility recipe in one
paragraph: Go builds are deterministic by construction — no timestamps, no
embedded build host, no randomised symbol ordering — so a build is reproducible
once you remove the three remaining inputs that vary. Pin the toolchain version
(`toolchain go1.27.1` in `go.mod`, 15.2), pin every dependency version
(`go.mod` + `go.sum`, and MVS guarantees the same resolution forever), and pass
`-trimpath`. Then anyone can rebuild your release and compare hashes.
`CGO_ENABLED=0` helps too, since a C compiler's output is not similarly
deterministic. The remaining variable is VCS stamping: `go build` embeds
`vcs.revision` and a dirty flag by default, so a clean checkout of the same
commit is required — `-buildvcs=false` removes even that.

## The full release build

Everything above, in one command:

```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build \
  -trimpath \
  -ldflags "-s -w -X main.version=1.4.2" \
  -o dist/rel-linux-amd64 .
ls -l dist/rel-linux-amd64
file -b dist/rel-linux-amd64
```

Output:

```
2212000 dist/rel-linux-amd64
ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=MGi3ldVXqJfKkcL-YHDJ/..., BuildID[sha1]=697d3aee932e9dcd32a43d0ffa67f9209a9128bc, stripped
```

2.21 MB, static, stripped, path-free, version-stamped. Slightly smaller than
the `-s -w` build above because `-trimpath` shortens the embedded path strings
too.

## Microarchitecture levels

`GOAMD64` and `GOARM64` let the compiler assume newer instructions:

```bash
go env GOAMD64 GOARM64
```

Output (GOAMD64 is empty here because the host is arm64):

```

v8.0
```

| Variable | Values | Default |
|---|---|---|
| `GOAMD64` | `v1`, `v2`, `v3`, `v4` | `v1` |
| `GOARM64` | `v8.0` … `v9.5`, with `,lse` / `,crypto` | `v8.0` |
| `GOARM` | `5`, `6`, `7`, with `,softfloat` / `,hardfloat` | `7` |
| `GO386` | `sse2`, `softfloat` | `sse2` |
| `GOMIPS`, `GOPPC64`, `GORISCV64`, `GOWASM` | per-arch | — |

`v3` is the useful amd64 level: AVX2, BMI, FMA — available on essentially every
server CPU since 2015, and typically worth a few percent on
numeric and string-heavy code. It is recorded in the binary:

```bash
GOOS=linux GOARCH=amd64 GOAMD64=v3 CGO_ENABLED=0 go build -o dist/v3 .
go version -m dist/v3 | grep -i amd64
```

Output:

```
	build	GOARCH=amd64
	build	GOAMD64=v3
```

The risk is `SIGILL` on an older CPU, with no graceful degradation. Default
`v1` unless you control the fleet and have measured a win.

## The CI release matrix

A `release.sh` that produces the full set, which 18.9 wires into a GitHub
Actions job:

```bash
#!/usr/bin/env bash
set -euo pipefail

VERSION="${1:-$(git describe --tags --always)}"
COMMIT="$(git rev-parse --short HEAD)"
LDFLAGS="-s -w -X main.version=$VERSION -X main.commit=$COMMIT"

TARGETS=(
  linux/amd64
  linux/arm64
  darwin/amd64
  darwin/arm64
  windows/amd64
)

rm -rf dist && mkdir -p dist
for t in "${TARGETS[@]}"; do
  os="${t%/*}"; arch="${t#*/}"
  ext=""; [ "$os" = windows ] && ext=".exe"
  out="dist/app_${VERSION}_${os}_${arch}${ext}"
  echo "building $out"
  CGO_ENABLED=0 GOOS="$os" GOARCH="$arch" \
    go build -trimpath -ldflags "$LDFLAGS" -o "$out" ./cmd/app
done

(cd dist && shasum -a 256 * > checksums.txt)
```

| Target | Why |
|---|---|
| `linux/amd64` | the default server and container target |
| `linux/arm64` | Graviton, Ampere, Apple-silicon Docker Desktop |
| `darwin/arm64` | developer laptops since 2020 |
| `darwin/amd64` | Intel Macs, still in the field |
| `windows/amd64` | only if you ship a CLI to end users |

Add `linux/arm/v7` for Raspberry Pi, `freebsd/amd64` if your users ask, and
`js/wasm` for a browser build. Skip the rest — a 47-row matrix is 47 artifacts
nobody downloads.

Two more notes for the pipeline. Cross-compilation is CPU-bound and perfectly
parallel, so run the matrix as parallel CI jobs sharing a warm module cache,
not a serial loop, once it grows past five targets. And emit `checksums.txt`:
combined with `-trimpath` and a pinned toolchain, it lets anyone verify your
release byte for byte.

Containerising the Linux binary is 18.8 — and the short version is that a
`CGO_ENABLED=0` static binary needs no base image at all beyond
`gcr.io/distroless/static` or `scratch` plus CA certificates. The
cross-compilation here is what makes that multi-stage build so small: you can
build on your host architecture and `COPY` the Linux binary in, rather than
compiling inside the image.

## Quick Reference

| Env | Purpose |
|---|---|
| `GOOS` | target operating system |
| `GOARCH` | target architecture |
| `CGO_ENABLED` | `0` for a static, freely cross-compilable binary |
| `CC` / `CXX` | the cross C compiler, when cgo is unavoidable |
| `GOAMD64` / `GOARM64` / `GOARM` | microarchitecture level |

| Flag / tag | Purpose |
|---|---|
| `-trimpath` | strip absolute paths; required for reproducibility |
| `-ldflags "-s -w"` | strip symbol table and DWARF (~34%) |
| `-ldflags "-X main.version=V"` | stamp a version |
| `-ldflags '-extldflags "-static"'` | force static linking **when cgo is on** |
| `-buildvcs=false` | do not embed git revision |
| `-tags netgo` | pure-Go DNS resolver even with cgo |
| `-tags osusergo` | pure-Go `os/user` even with cgo |

| Command | Does |
|---|---|
| `go tool dist list` | all 47 GOOS/GOARCH pairs |
| `go tool dist list -json` | the same, with `FirstClass` flags |
| `file BIN` | format, architecture, static/dynamic, stripped |
| `otool -L BIN` | dynamic libraries (macOS) |
| `ldd BIN` | dynamic libraries (Linux) |
| `go version -m BIN` | embedded modules and build settings |
| `shasum -a 256 BIN` | reproducibility check |

## Common Mistakes

**Leaving `CGO_ENABLED` at its default for a container build.** On a Linux
builder it defaults to 1, you get a glibc-linked binary, and it dies on Alpine
with a missing-interpreter error that looks nothing like a linking problem.

**Expecting `CGO_ENABLED=0` to produce a static binary on macOS.** Darwin has
no supported static linking; `otool -L` shows `libSystem` either way.

**Shipping `-s -w` binaries and then trying to debug one.** `-w` removes the
DWARF delve needs; build a second unstripped copy for the same commit.

**Assuming a reproducible build without `-trimpath`.** The build directory is
embedded, so two clones at different paths produce different bytes — shown
above with two different sha256 sums.

**Setting `GOAMD64=v3` for public release binaries.** An older CPU gets
`SIGILL` with no fallback; `v1` is the portable default for anything you do not
control.

**Testing only on your own GOOS.** `GOOS=windows go vet ./...` costs two
seconds and catches the `_windows.go` file that stopped compiling a month ago
(15.4).

## Exercises

1. Build the program from this lesson for every first-class port
   (`go tool dist list -json | jq -r '.[] | select(.FirstClass) | .GOOS+"/"+.GOARCH'`)
   and produce a table of sizes. Explain the two largest outliers.
2. Prove the `-trimpath` reproducibility result yourself: two clones at
   different paths, four builds, four sha256 sums. Then add `-buildvcs=false`
   and see whether it changes anything on a clean tree.
3. Write a program that calls `net.LookupHost` and build it four ways:
   cgo on/off crossed with `netgo` on/off. Use
   `go list -f '{{.GoFiles}}{{.CgoFiles}}' net` to show which resolver each
   build selected.
4. Build the same binary with `GOAMD64=v1` and `GOAMD64=v3` and compare sizes
   and a numeric benchmark. If you have an older x86 machine, run the `v3`
   build there and capture what happens.
5. Build with and without `-ldflags "-s -w"`, run both, and trigger a panic in
   each. Compare the tracebacks and explain which information survived
   stripping and why.
6. Write the `release.sh` from this lesson, run it, and verify every artifact
   with `file` and `go version -m`. Then verify `checksums.txt` on a second
   clone of the repo.

---

[← 15.4 Build Tags and Conditional Compilation](04-build-tags.md) · [Course Index](../README.md) · [15.6 pprof Profiling →](06-pprof.md)
