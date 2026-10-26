[← 18.7 Metrics with Prometheus](07-prometheus.md) · [Course Index](../README.md) · [18.9 CI with GitHub Actions →](09-ci.md)

# 18.8 — Docker Multi-Stage Builds

## What You'll Learn

- Why `COPY go.mod go.sum` comes **before** `COPY . .` — the layer cache, which is the whole reason multi-stage Dockerfiles look like this
- The real static Linux binary: `CGO_ENABLED=0 GOOS=linux`, with its genuine `file` and `ls -l` output, and `-trimpath` making the build byte-reproducible
- The four base-image choices in a table, and the two things that silently break in `scratch`: TLS and timezones
- A `HEALTHCHECK` in a shell-less image via a `-health` flag on your own binary

**No Docker daemon is running in this environment.** Every Go build below is real and
executed; every `docker` command is labelled as not executed. The Dockerfile and compose YAML
were both parsed and structurally validated.

## Coming From Another Language

| Other | Go |
|---|---|
| ship `node_modules` and a Node runtime (~200MB) | ship **one static binary**; the final image can be ~10MB with no OS at all |
| `python:3.12-slim` + `pip install` at runtime | nothing to install; the binary has no dependencies |
| a multi-stage build to drop dev dependencies | a multi-stage build to drop the **compiler** — 900MB of toolchain |
| `npm ci` cached by `package-lock.json` | `go mod download` cached by `go.mod` + `go.sum`; same trick |
| JVM: fat jar + a JRE base image | no runtime at all; `FROM scratch` is viable |

## The complete Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.27-alpine AS build

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .

ARG VERSION=dev
ARG COMMIT=none
ARG BUILD_TIME=unknown
ARG TARGETOS=linux
ARG TARGETARCH=amd64

RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -trimpath \
    -ldflags="-s -w -X main.version=${VERSION} -X main.commit=${COMMIT} -X main.buildTime=${BUILD_TIME}" \
    -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot AS final

COPY --from=build /out/server /server

USER nonroot:nonroot
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/server", "-health"]

ENTRYPOINT ["/server"]
```

Validated structurally: 17 instructions, 2 stages, no unknown directives. Every line is
explained below.

## The layer cache is the point

This is the reason the Dockerfile is shaped this way, and it is worth understanding rather than
copying.

Docker caches each instruction's result, keyed by the instruction text plus a hash of the files
it copies. When any key changes, that layer and **every layer after it** is rebuilt.

The naive version:

```dockerfile
COPY . .
RUN go mod download
RUN go build -o /out/server ./cmd/server
```

Change one line in one `.go` file and `COPY . .` is invalidated, so `go mod download` reruns —
re-fetching every dependency over the network on every single build. For a service with 40
transitive modules that is 30-90 seconds added to every iteration and to every CI run.

The correct version splits by **change frequency**:

```dockerfile
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build ...
```

`go.mod` and `go.sum` change when you add a dependency — weekly, maybe. Your source changes
every commit. So the `go mod download` layer is keyed only on the manifest files and survives
every source edit, and a normal build re-runs only `COPY . .` and `go build`. The general
rule, applicable to every language: **order instructions from least-frequently-changed to
most.**

With BuildKit you can go further and cache the module and build caches across builds entirely:

```dockerfile
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server
```

That persists Go's own build cache between `docker build` invocations, which turns a 40-second
cold compile into a 3-second incremental one. It needs BuildKit (the default since Docker
23) and the `# syntax=docker/dockerfile:1` line.

## The build flags, with real output

Each flag earns its place. Here is what they actually do, built for real on this machine
(darwin/arm64 host, cross-compiling to linux):

```bash
go build -o out/server-darwin ./cmd/server

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o out/server-linux-plain ./cmd/server

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
  -trimpath \
  -ldflags="-s -w -X main.version=1.4.2 -X main.commit=9f3a1c7 -X main.buildTime=2026-01-01T12:00:00Z" \
  -o out/server ./cmd/server

CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -trimpath -ldflags="-s -w" -o out/server-arm64 ./cmd/server
```

`file` on each (the `Go BuildID` string is content-derived and will differ for your build):

Output:

```
out/server-darwin: Mach-O 64-bit executable arm64
out/server-linux-plain: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=__MPh1vn9YYE8qz961Uy/fvdhrOhvwMY5lNu0G00o/mzLgIatVfGXk0xeOD2F7/YA349vOko3OEHO0cMFgx, BuildID[sha1]=fb841f5e805f09c2804e43e884ce053b1c946d89, with debug_info, not stripped
out/server: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=WUKze3KioOHrdjxgu4Qr/TYPe_Ohil-uD83MWgav8/PYcnC81dOmuj2QjzLqWY/eCofmwUsTdEld6DC5J2p, BuildID[sha1]=6f250859ffa7a0b9d18678ffd9fd4d16bbbca754, stripped
out/server-arm64: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, Go BuildID=kugC6gjM06yb1rSp6U-c/2TbRkYIGTnBqjyPyNgh8/IYneM2JP5te4ttpcWkAc/T0vxfNsAh6kE-22EVqp2, BuildID[sha1]=ec3a7de94406dd6695c13662be6a63b865e50c09, stripped
```

and `ls -l`:

Output:

```
-rwxr-xr-x@ 1 raj  staff   7233696 Oct 26 14:45 server
-rwxr-xr-x@ 1 raj  staff   6684832 Oct 26 14:45 server-arm64
-rwxr-xr-x@ 1 raj  staff  10201874 Oct 26 14:45 server-darwin
-rwxr-xr-x@ 1 raj  staff  10682374 Oct 26 14:45 server-linux-plain
```

Everything that matters is in those two captures.

**`statically linked`.** `CGO_ENABLED=0` is what produces that, and it is the single most
important flag: a dynamically linked binary looks for `libc` at runtime, and `scratch` and
`distroless/static` have no `libc`, so the container exits immediately with
`no such file or directory` — a maddening error, because the file it cannot find is the
*linker*, not your binary. With CGO off, `net` and `os/user` use their pure-Go
implementations and the result depends on nothing.

**`-ldflags="-s -w"` took 10,682,374 bytes down to 7,233,696** — a 32% cut. `-s` strips the
symbol table, `-w` strips DWARF debug info; `file` reports `stripped` instead of
`with debug_info, not stripped`. The cost is that a stack trace in a core dump is unreadable
and `dlv` cannot attach usefully ([15.7](../15-tooling/07-delve.md)); the panic traces Go
prints at runtime still have function names and line numbers, because those come from a
separate table that `-s -w` does not remove. For most services that is the right trade.

**arm64 is slightly smaller than amd64** (6.68MB vs 7.23MB) for identical source — different
instruction encodings, nothing deeper.

**`GOOS=linux` on a Mac just works.** No toolchain to install, no container needed to build a
Linux binary ([15.5](../15-tooling/05-cross-compilation.md)). That is why building the real
artifact was possible here without a Docker daemon.

### -trimpath makes the build reproducible

`-trimpath` removes absolute filesystem paths from the binary, replacing
`/Users/raj/scratch/m18/cmd/server/main.go` with `github.com/acme/orders/cmd/server/main.go`.
Two reasons: your home directory is not something to ship to users, and the path is the main
thing that differs between two builds of the same commit on two machines.

Verified here by building the identical command twice and comparing bytes:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath \
  -ldflags="-s -w -X main.version=1.4.2 -X main.commit=9f3a1c7 -X main.buildTime=2026-01-01T12:00:00Z" \
  -o out/server2 ./cmd/server
cmp -s out/server out/server2 && echo "IDENTICAL (reproducible)" || echo "DIFFER"
```

Output:

```
IDENTICAL (reproducible)
```

Byte-for-byte identical, which means the Docker layer containing it is cache-identical too.
Note that `buildTime` had to be a **fixed** value for this to hold — injecting
`$(date -u +...)` makes every build differ by construction, so a genuinely reproducible
pipeline uses the commit's timestamp (`git log -1 --format=%cI`) rather than the build's.

## Choosing the final stage

| Base | Size added | Shell | CA certs | tzdata | libc | When |
|---|---|---|---|---|---|---|
| `scratch` | **0** | no | **no** | **no** | no | smallest possible; you must add certs/tzdata yourself |
| `gcr.io/distroless/static-debian12` | ~2MB | no | **yes** | **yes** | no | **the default recommendation** |
| `alpine:3.21` | ~8MB | yes (`sh`) | via `apk add ca-certificates` | via `apk add tzdata` | musl | you need a shell for debugging |
| `debian:12-slim` | ~75MB | yes (`bash`) | yes | yes | glibc | you need glibc (so CGO) or real tooling |

With a 7MB binary, the final image is roughly 7MB on `scratch`, 9MB on distroless, 15MB on
alpine, 82MB on debian-slim. For comparison, the `golang:1.27-alpine` build stage is around
250MB and a full `golang:1.27` is over 800MB — which is what multi-stage saves you.

**Recommendation: `gcr.io/distroless/static-debian12:nonroot`.** It is `scratch` plus the
three things you always end up needing — CA certificates, timezone data, and `/etc/passwd`
with a `nonroot` user — and nothing else. No shell, no package manager, no `curl`, so the
attack surface after a remote-code-execution bug is close to nil.

The argument for `alpine` is debugging: `docker exec -it ... sh` works, and during an incident
that is worth something. The counter-argument is that `kubectl debug --image=busybox
--target=...` gives you a shell in a sidecar sharing the process namespace, so you get the
debugging without shipping the shell. Pick alpine only if your team will not use ephemeral
containers.

### The two things that silently break on scratch

**CA certificates.** Without them, every outbound HTTPS call fails with
`x509: certificate signed by unknown authority`. Your service works perfectly in local
development and cannot reach Stripe in production.

**Timezone data.** Without `/usr/share/zoneinfo`, `time.LoadLocation("America/New_York")`
returns `unknown time zone America/New_York` ([13.4](../13-stdlib/04-time.md)). UTC keeps
working, so this only breaks the code paths doing local-time formatting — often a monthly
report.

On `scratch`, copy both from the build stage:

```dockerfile
FROM alpine:3.21 AS certs
RUN apk add --no-cache ca-certificates tzdata

FROM scratch AS final
COPY --from=certs /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=certs /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=build /out/server /server
USER 65534:65534
ENTRYPOINT ["/server"]
```

Or, Go-only, embed the zoneinfo database into the binary with a single import — about 450KB,
and it removes the filesystem dependency entirely:

```go
import _ "time/tzdata"
```

Distroless `static` includes both already, which is the main reason to prefer it.

## USER, EXPOSE, ENTRYPOINT

**`USER nonroot:nonroot`** (uid 65532 in distroless). Containers run as root by default, and
root in a container is root on the host if anything escapes the namespace — a kernel CVE, a
careless `--privileged`, a writable hostPath mount. Your Go binary needs no privileges: it
listens on 8080, not 80, and writes nothing to disk. Many clusters enforce this with a
`runAsNonRoot: true` Pod Security Standard, and an image with no non-root user then simply
fails to schedule. On `scratch` there is no `/etc/passwd`, so use the numeric form
`USER 65534:65534`.

**`EXPOSE 8080`** is **documentation only**. It publishes nothing; `docker run -p` and the
Kubernetes `containerPort` do that. Keep it because tooling reads it.

**`ENTRYPOINT` versus `CMD`:**

| Form | Behaviour |
|---|---|
| `ENTRYPOINT ["/server"]` | always runs; `docker run img -flag` **appends** `-flag` |
| `CMD ["/server"]` | `docker run img /bin/sh` **replaces** it entirely |
| `ENTRYPOINT ["/server"] CMD ["-addr=:8080"]` | the `CMD` is default arguments the user can override |

Use **exec form** (a JSON array), always. Shell form (`ENTRYPOINT /server`) wraps your process
in `/bin/sh -c`, so your binary is **not PID 1**, the shell does not forward `SIGTERM`, and
everything in [18.5](05-graceful-shutdown.md) stops working: your process never receives the
signal and gets `SIGKILL`ed after the grace period. There is also no shell in distroless, so
the shell form fails outright — which is at least a loud failure rather than a silent one.

## HEALTHCHECK without a shell

`HEALTHCHECK` normally runs `curl` or `wget`, and distroless has neither. The trick is to make
your **own binary** the health probe:

```go
func main() {
  health := flag.Bool("health", false, "probe the local server and exit")
  flag.Parse()

  if *health {
    os.Exit(selfCheck())
  }
  if err := run(); err != nil {
    slog.Error("fatal", "err", err)
    os.Exit(1)
  }
}

func selfCheck() int {
  addr := os.Getenv("HEALTH_ADDR")
  if addr == "" {
    addr = "http://127.0.0.1:8080/healthz"
  }
  c := &http.Client{Timeout: 2 * time.Second}
  resp, err := c.Get(addr)
  if err != nil {
    fmt.Fprintln(os.Stderr, "health:", err)
    return 1
  }
  defer resp.Body.Close()
  io.Copy(io.Discard, resp.Body)
  if resp.StatusCode != http.StatusOK {
    fmt.Fprintln(os.Stderr, "health: status", resp.Status)
    return 1
  }
  return 0
}
```

Then `HEALTHCHECK CMD ["/server", "-health"]`. Verified for real, outside Docker — with
nothing listening:

Output:

```
exit=1
health: Get "$HEALTH_ADDR/healthz": dial tcp 127.0.0.1:18501: connect: connection refused
```

and with the server running:

Output:

```
exit=0
orders dev (none)
```

Exit 0 means healthy, exit 1 means unhealthy — that is the entire `HEALTHCHECK` contract, and
the binary already contains an HTTP client. It costs about 20 lines and no extra image content.

One honest caveat: **in Kubernetes, `HEALTHCHECK` is ignored entirely.** The kubelet uses the
probes from [18.6](06-health-checks.md) and never reads the image's healthcheck. It matters for
plain `docker run`, for compose (`depends_on: condition: service_healthy`), for Swarm and for
some PaaS platforms. Include it; do not rely on it under Kubernetes.

## .dockerignore

```
.git
.github
.gitignore
Dockerfile
.dockerignore
docker-compose.yml
out/
bin/
*_test.go
testdata/
README.md
*.md
.env
.env.*
coverage.out
.idea/
.vscode/
```

Three reasons it is not optional. **`.git` is often larger than your source** and gets sent to
the daemon as build context on every build, then invalidates `COPY . .` on every commit even
when no `.go` file changed. **Secrets leak**: a `.env` with real credentials copied into a
layer is extractable from the image forever, even if a later layer deletes it — layers are
append-only. **Build speed**: the entire context is tarred and transferred before the build
starts.

Keep `*_test.go` out of the image but note it does not break the build — `go build` ignores
test files anyway.

## Build args for version injection

`ARG` values flow into the `-ldflags -X` from [18.6](06-health-checks.md) and
[15.1](../15-tooling/01-build-run-install.md):

Expected output (requires a running Docker daemon; not executed in this environment):

```bash
$ docker build \
    --build-arg VERSION=$(git describe --tags --always) \
    --build-arg COMMIT=$(git rev-parse --short HEAD) \
    --build-arg BUILD_TIME=$(git log -1 --format=%cI) \
    -t ghcr.io/acme/orders:1.4.2 .
[+] Building 34.1s (16/16) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load metadata for gcr.io/distroless/static-debian12:nonroot
 => [internal] load metadata for docker.io/library/golang:1.27-alpine
 => [internal] load .dockerignore
 => [build 1/6] FROM docker.io/library/golang:1.27-alpine
 => [internal] load build context
 => CACHED [build 2/6] WORKDIR /src
 => CACHED [build 3/6] COPY go.mod go.sum ./
 => CACHED [build 4/6] RUN go mod download
 => [build 5/6] COPY . .
 => [build 6/6] RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath ...
 => [final 1/1] COPY --from=build /out/server /server
 => exporting to image
 => => naming to ghcr.io/acme/orders:1.4.2

$ docker images ghcr.io/acme/orders
REPOSITORY             TAG     IMAGE ID       CREATED         SIZE
ghcr.io/acme/orders    1.4.2   8c1f2a3b4d5e   2 seconds ago   9.3MB

$ docker run --rm -p 8080:8080 \
    -e DATABASE_URL=postgres://app:pw@host.docker.internal:5432/orders \
    -e JWT_SECRET=dev-only-signing-key-at-least-32b!! \
    ghcr.io/acme/orders:1.4.2
{"time":"2026-01-01T12:00:00Z","level":"INFO","msg":"starting","version":"1.4.2","commit":"9f3a1c7","build_time":"2026-01-01T12:00:00Z"}

$ curl localhost:8080/buildinfo
{"version":"1.4.2","commit":"9f3a1c7","build_time":"2026-01-01T12:00:00Z","go_version":"go1.27.1"}
```

The **9.3MB** is the expected total: the 7.2MB binary measured above plus distroless `static`'s
~2MB. Note `CACHED` on the `go mod download` layer even though the source changed — that is the
layer-ordering payoff, visible in the build output.

`docker build` also injects `TARGETOS` and `TARGETARCH` automatically when building with
buildx, which is why they are declared as `ARG`s with defaults rather than hardcoded.

## docker compose for local development

Validated as YAML (parsed with `yaml.safe_load`). Agrees with
[17.2](../17-database/02-pgx.md)'s Postgres setup.

```yaml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: orders
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d orders"]
      interval: 5s
      timeout: 3s
      retries: 10

  api:
    build:
      context: .
      args:
        VERSION: dev
        COMMIT: local
    ports:
      - "8080:8080"
    environment:
      ADDR: ":8080"
      ENV: development
      LOG_FORMAT: text
      LOG_LEVEL: debug
      DATABASE_URL: postgres://app:devpassword@db:5432/orders?sslmode=disable
      JWT_SECRET: dev-only-signing-key-at-least-32b!!
      SHUTDOWN_TIMEOUT: 5s
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

Expected output (requires a running Docker daemon; not executed in this environment):

```bash
$ docker compose up --build
[+] Running 3/3
 ✔ Network orders_default   Created
 ✔ Container orders-db-1    Healthy
 ✔ Container orders-api-1   Started
orders-db-1   | database system is ready to accept connections
orders-api-1  | level=INFO msg=starting version=dev commit=local
orders-api-1  | level=INFO msg="server listening" addr=:8080
```

Three details that matter. **`condition: service_healthy`**, not the bare
`depends_on: [db]` — the bare form waits only for the container to *start*, and Postgres takes
several seconds after that to accept connections, so your API crashes on its config validation
([18.2](02-config.md)). **`host: db`** in the `DATABASE_URL`: compose provides DNS for service
names on the shared network. **`sslmode=disable`** for local only; never in production.

The named `pgdata` volume survives `docker compose down`; use `down -v` to wipe it, which is
what you want after changing a migration.

## Multi-arch builds

One line, because it is a one-line concern:
`docker buildx build --platform linux/amd64,linux/arm64 --push -t ghcr.io/acme/orders:1.4.2 .`
builds both architectures (Go cross-compiles each natively, so this is fast) and pushes a
manifest list that resolves per-architecture at pull time — worth doing the moment anyone on
the team runs an Apple Silicon Mac or you deploy to Graviton.

## Quick Reference

| Instruction | Note |
|---|---|
| `FROM img AS name` | names a stage for `COPY --from=name` |
| `WORKDIR /src` | creates and `cd`s; use absolute paths |
| `COPY go.mod go.sum ./` | before `COPY . .`, so `go mod download` stays cached |
| `COPY --from=build /out/server /server` | pulls one file from another stage; the toolchain stays behind |
| `ARG X=default` | build-time variable; not present at runtime |
| `ENV X=v` | runtime environment variable, baked into the image |
| `USER nonroot:nonroot` | or `USER 65534:65534` on scratch |
| `EXPOSE 8080` | documentation only |
| `ENTRYPOINT ["/server"]` | exec form, so your binary is PID 1 and gets SIGTERM |
| `CMD ["-addr=:8080"]` | default args appended to `ENTRYPOINT` |
| `HEALTHCHECK CMD ["/server","-health"]` | exit 0 healthy, 1 unhealthy; ignored by Kubernetes |

| Build flag | Effect |
|---|---|
| `CGO_ENABLED=0` | **static** binary; required for scratch/distroless-static |
| `GOOS=linux GOARCH=amd64` | cross-compile; no toolchain needed |
| `-ldflags="-s -w"` | strip symbols and DWARF; ~30% smaller |
| `-ldflags="-X main.version=v"` | inject into a package-level `string` var |
| `-trimpath` | remove absolute paths; makes builds **byte-reproducible** |
| `-tags netgo,osusergo` | force pure-Go resolvers even if CGO is somehow on |

| Base image | Size | Use |
|---|---|---|
| `scratch` | 0 | smallest; add CA certs + tzdata yourself |
| `gcr.io/distroless/static-debian12:nonroot` | ~2MB | **default**; certs, tzdata, nonroot user |
| `alpine:3.21` | ~8MB | you want a shell in the image |
| `debian:12-slim` | ~75MB | glibc/CGO required |

## Common Mistakes

**`COPY . .` before `go mod download`.** Every source edit re-downloads every dependency; copy
`go.mod`/`go.sum` first.

**Forgetting `CGO_ENABLED=0` with a scratch or distroless-static base.** The container exits
with `no such file or directory` — the missing file is the dynamic linker, not your binary.

**No CA certificates in a `scratch` image.** Every HTTPS call fails with
`x509: certificate signed by unknown authority`; copy `ca-certificates.crt` or use distroless.

**No tzdata.** `time.LoadLocation` fails for every non-UTC zone; copy `/usr/share/zoneinfo` or
`import _ "time/tzdata"`.

**Shell-form `ENTRYPOINT`.** `sh -c` becomes PID 1 and swallows `SIGTERM`, so graceful shutdown
never runs; always use the JSON array form.

**Running as root.** A container escape becomes host root, and `runAsNonRoot` clusters refuse
the image; add a `USER`.

**No `.dockerignore`.** `.git` bloats the context, invalidates the cache on every commit, and a
`.env` becomes permanently extractable from a layer.

**`depends_on: [db]` without `condition: service_healthy`.** The API starts before Postgres
accepts connections and exits on config validation.

**Injecting `$(date)` as the build time while claiming reproducible builds.** Every build then
differs by construction; use the commit timestamp.

## Exercises

1. Build the same service onto all four bases (`scratch` with certs, distroless-static, alpine,
   debian-slim), and record the real `docker images` sizes. Then `docker run` each and confirm
   an outbound HTTPS request succeeds on all four — fixing the one that fails.
2. Prove the layer-cache claim: build once, touch one `.go` file, build again with
   `--progress=plain`, and show `CACHED` on the `go mod download` step. Then move `COPY . .`
   above it and show the same step re-running.
3. Add the BuildKit `--mount=type=cache` lines for `/go/pkg/mod` and `/root/.cache/go-build`,
   and measure the cold versus warm build time for a one-line source change.
4. Make the build fully reproducible: pin the builder by digest, use the commit timestamp for
   `BUILD_TIME`, and confirm two builds of the same commit produce the same image digest.
5. Write a `docker-compose.override.yml` for local development that mounts the source and runs
   `go run ./cmd/server` with live reload, while the base compose file keeps the production-shaped
   image build. Show both working.

---

[← 18.7 Metrics with Prometheus](07-prometheus.md) · [Course Index](../README.md) · [18.9 CI with GitHub Actions →](09-ci.md)
