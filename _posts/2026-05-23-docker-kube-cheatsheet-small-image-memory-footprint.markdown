---
layout: post
title: "Docker and Kubernetes Cheatsheet: Shrinking Image Size and Memory Footprint"
date: 2026-05-23 09:00:00 +0700
categories: infrastructure devops
tags: [docker, kubernetes, containers, optimization, memory, image-size, multi-stage-build, distroless, go]
description: "Practical recipes for building tiny container images and running them lean in Kubernetes. Multi-stage builds, distroless, static binaries, resource tuning, and the OOMKilled traps."
---

A 1.2 GB image to run a 12 MB Go binary is a confession. It says "I copied a Dockerfile from somewhere and didn't read it." I've shipped images like that. We all have. This post is the cheatsheet I wish I'd had earlier — concrete techniques, with the trade-offs, for shrinking both the image on disk and the memory the container actually uses at runtime.

Most of the examples are Go because that's what I write, but the principles port directly to Rust, C, and partially to Java, Python, and Node. Notes for the dynamic-language cases are at the end.

<!--more-->

## The two metrics that matter

People conflate "image size" and "memory footprint." They're related but different.

- **Image size** is bytes on disk in the registry. Affects pull time, registry cost, cold-start speed, attack surface.
- **Memory footprint** is RSS at runtime. Affects how many pods fit on a node, how cleanly autoscaling works, and whether your pod gets OOMKilled at 3 AM.

A small image does not guarantee a small memory footprint. A JVM app in a 200 MB Alpine image still wants 512 MB RAM at idle. Go the other direction too: a fat Ubuntu image running a Go binary that uses 30 MB RSS is wasteful on disk but fine at runtime.

Optimize both, but track them separately.

## Multi-stage builds: the single biggest win

If you take one thing from this post: build in one stage, ship from another. The build stage has compilers, headers, package managers, and your source code. The runtime stage has the binary and nothing else.

```dockerfile
# syntax=docker/dockerfile:1.6
# ---- build stage ----
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w" \
    -trimpath \
    -o /out/app ./cmd/app

# ---- runtime stage ----
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

What each piece earns:

- `CGO_ENABLED=0` — pure Go binary, no glibc dependency, runs on `scratch` or `distroless/static`.
- `-ldflags="-s -w"` — strips symbol table and DWARF debug info. Saves 20–40% on binary size. Lose stack traces with line numbers in panics, gain a smaller image.
- `-trimpath` — removes absolute build paths from the binary. Smaller and reproducible.
- `distroless/static` — no shell, no package manager, no busybox. Just the libraries needed to run a static binary, plus CA certs and tzdata.
- `nonroot` — uid 65532, doesn't need to be set in Kubernetes `securityContext` separately (though you should anyway).

Result for a typical service: ~15 MB image, ~12 MB binary, no shell to exploit.

## Base image cheatsheet

Pick the smallest base that still works. From smallest to largest:

| Base | Size | Has | Use when |
|---|---|---|---|
| `scratch` | 0 B | nothing | fully static binary, no CA certs needed |
| `gcr.io/distroless/static` | ~2 MB | CA certs, tzdata, /etc/passwd | static binary that makes HTTPS calls |
| `gcr.io/distroless/base` | ~20 MB | + glibc, libssl | dynamically-linked binary |
| `alpine:3.19` | ~7 MB | musl libc, busybox shell, apk | need a shell for debugging or musl-linked binary |
| `debian:12-slim` | ~75 MB | glibc, dpkg, basic tools | need apt for runtime deps |
| `ubuntu:22.04` | ~80 MB | glibc, apt, more tools | you really need Ubuntu specifically |

The real question isn't "which is smallest" but "what does my binary actually need." If you're calling out to HTTPS, you need CA certs. If you're parsing timezones, you need tzdata. If you're using cgo, you need glibc or musl. If you're shelling out to `curl`, stop and use a library instead.

## When Alpine bites you

Alpine uses musl libc, not glibc. Most Go programs don't care because they don't use cgo. The moment you turn cgo on — for `net` DNS resolution, for SQLite, for any C library — you're linking against musl. Things that work on glibc may not work on musl. The pathologies I've hit:

- DNS resolution edge cases (musl's resolver is stricter about `/etc/resolv.conf` than glibc).
- Subtle differences in `getaddrinfo` behavior under high concurrency.
- Some Go libraries that use cgo for crypto perform measurably worse on musl.

If you don't need a shell at runtime, skip Alpine entirely and use `distroless/static`. If you need the shell for debugging, prefer ephemeral debug containers (covered later) over baking a shell into the production image.

## Layering and the cache

Docker builds layers top to bottom. Each instruction creates a layer. Layers are cached by content. Order them from "rarely changes" to "frequently changes" so the cache survives normal edits.

```dockerfile
# WRONG — every code change invalidates dependency download
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY . .
RUN go mod download
RUN go build -o /out/app ./cmd/app
```

```dockerfile
# RIGHT — dependencies cached separately from source
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /out/app ./cmd/app
```

The `RUN go mod download` layer only invalidates when `go.mod` or `go.sum` changes, which is much less often than your source code. CI builds drop from minutes to seconds.

For Node, the equivalent is:

```dockerfile
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build
```

For Python with pip:

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
```

`--no-cache-dir` is the pip equivalent of "don't keep the wheel cache around in the image." Saves 50–200 MB on a non-trivial dependency tree.

## BuildKit cache mounts

If you're on Docker 23+ or any modern CI, BuildKit cache mounts make package manager caches persist across builds without ending up in the final image.

```dockerfile
# syntax=docker/dockerfile:1.6
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /out/app ./cmd/app
```

The cache lives on the build host between builds. Final image doesn't contain it. Build times drop dramatically on CI runners that persist BuildKit cache.

## .dockerignore is not optional

Every byte you `COPY` into the build context goes over the wire to the Docker daemon and into the build cache key. A `.git` directory at 400 MB makes every build slow even before it starts. A `.dockerignore` that's just three lines:

```
.git
node_modules
*.log
```

…cuts build context size by 90%+ on most projects. Look at your context size with `docker build` — it prints `Sending build context to Docker daemon  XXX MB`. If that number is large, fix `.dockerignore` first, optimize Dockerfile second.

A more thorough one for a Go project:

```
.git
.gitignore
.github
.idea
.vscode
*.md
Dockerfile*
docker-compose*.yml
_site
node_modules
dist
build
*.log
*.tmp
coverage.out
.env*
```


## UPX is a trap

Someone will tell you to UPX-compress your binary for a 60% smaller image. Resist. UPX-compressed binaries decompress into RAM at startup, which means:

- Cold start gets slower (decompression time).
- Memory usage gets higher (compressed + decompressed both held briefly).
- Many security scanners flag UPX-packed binaries as malware-like.
- Distroless image layer compression already gets most of the win.

The genuine wins from `-ldflags="-s -w"` and `-trimpath` are free. UPX on top is rarely worth the trade.

## Go-specific runtime memory tuning

Image size is what you ship. Memory footprint is what you pay for at runtime. Go's runtime has a few knobs that matter inside containers.

**`GOMEMLIMIT`** — soft limit on the Go heap. Without it, Go's GC will let RSS climb until it hits the cgroup limit and the kernel kills the process. Set it to ~80% of the container's memory limit:

```yaml
# Kubernetes deployment snippet
env:
  - name: GOMEMLIMIT
    valueFrom:
      resourceFieldRef:
        resource: limits.memory
        divisor: 1
```

That gives Go the limit in bytes via the downward API. Then in your code, multiply by 0.8 — or just hardcode something like `GOMEMLIMIT=400MiB` if your limit is 512Mi. Go will run GC more aggressively as it approaches the limit instead of letting allocation outpace collection.

**`GOMAXPROCS`** — the number of OS threads Go schedules goroutines on. By default Go reads this from the host's CPU count, not the container's CPU limit. On a 64-core node with `cpu: 1` limit, Go thinks it has 64 cores, schedules 64 P's, and you get pathological lock contention. Fix:

```go
import _ "go.uber.org/automaxprocs"
```

That import alone reads cgroup CPU quota and sets `GOMAXPROCS` accordingly. One line, real impact.

**`GOGC`** — GC aggressiveness, default 100. Lower values trade CPU for less memory. I rarely touch this; `GOMEMLIMIT` is the better lever.


## Kubernetes resource requests and limits

This is where memory footprint stops being a Go problem and starts being a scheduling problem.

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
```

What each line does:

- `requests.cpu` — what the scheduler reserves on a node. Sum of requests determines pod density.
- `requests.memory` — same, for memory.
- `limits.cpu` — hard ceiling. Container is throttled (not killed) when it tries to use more.
- `limits.memory` — hard ceiling. Container is **OOMKilled** when it tries to use more.

The asymmetry is critical. CPU over-limit is throttled. Memory over-limit is killed. A pod that briefly spikes to 130 Mi against a 128 Mi limit dies, gets restarted, and your error rate spikes for the duration of the restart.

Set requests close to actual usage. Set limits with enough headroom for spikes. Don't make them equal unless you really know what you're doing.

## Measuring real memory usage

Don't guess. Measure. The two commands I run in every new service:

```bash
# Live RSS for every pod in a namespace
kubectl top pod -n my-namespace

# Per-container with the metrics-server
kubectl top pod my-pod --containers
```

For a more honest picture over time, use the cAdvisor metric `container_memory_working_set_bytes` in your Prometheus stack. That's what the kernel actually considers "in use" and what triggers OOMKill, not the misleading `container_memory_usage_bytes` (which includes reclaimable cache).

A common surprise: `kubectl top` shows 80 Mi but Prometheus shows 200 Mi working set. The difference is page cache from disk reads. If your service does a lot of file I/O, working set is what matters for the OOM killer.

## Distroless debugging without baking a shell

The downside of `distroless/static`: no shell, can't `kubectl exec -it pod -- sh`. The fix is ephemeral debug containers (Kubernetes 1.25+):

```bash
kubectl debug -it my-pod --image=busybox --target=my-container
```

This attaches a busybox container that shares the target's process namespace. You can `ps aux`, look at `/proc/<pid>/`, run `netstat`, all without the production image carrying any of those tools. The pod terminates the debug container when you exit, and the production image stays minimal.

## Minimal scratch example

For the absolute smallest image, Go binary on `scratch`:

```dockerfile
# syntax=docker/dockerfile:1.6
FROM golang:1.22-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -trimpath -o /app ./cmd/app

FROM scratch
COPY --from=build /app /app
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
ENTRYPOINT ["/app"]
```

That's it. ~12 MB total. The CA certs line matters if your app makes outbound HTTPS calls — without it, every TLS handshake fails with "certificate signed by unknown authority." A common stumble.


## Notes for non-Go runtimes

The principles port. The numbers don't.

**Rust** — same idea as Go. Static MUSL build, `distroless/static` or `scratch`, similar 5–15 MB images. Build with `cargo build --release --target x86_64-unknown-linux-musl`. Strip with `strip` or `cargo` profile settings.

**Java** — the JVM is heavy. Two real options:
- **jlink** to build a custom JRE with only the modules you need. Cuts a base JRE from 200 MB to 60–80 MB.
- **GraalVM native-image** to compile to a static binary. 30–80 MB, low memory, slow build. Not every framework supports it well; Spring Boot does, but check your dependencies.

For a normal JVM app, set `-XX:MaxRAMPercentage=75.0` (or similar) so the JVM respects the cgroup limit rather than the host's RAM. Older JVMs (pre-10) need `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap` instead.

**Python** — the multi-stage trick is `pip install` in a builder, then copy `/usr/local/lib/python3.x/site-packages` and the binary into a runtime stage. Use `python:3.12-slim` (~50 MB) over `python:3.12` (~1 GB). For pure-Python apps, distroless has `gcr.io/distroless/python3-debian12`. CPython itself is the floor on memory — expect 30–50 MB RSS minimum even for hello-world.

**Node** — `node:20-alpine` is around 50 MB, `node:20-slim` around 90 MB. Multi-stage with `npm ci --omit=dev` in the runtime stage gets you below 100 MB total for most apps. Node's V8 will happily use more memory than your container limit by default; set `--max-old-space-size=384` (in MB) for a 512 Mi container.

## Quick reference

The cheatsheet I keep open in another tab when I'm Dockerfile-tuning:

```
GO IMAGE BUILD
  CGO_ENABLED=0           # static binary
  -ldflags="-s -w"        # strip symbols
  -trimpath               # reproducible paths
  scratch / distroless/static  # smallest runtime base
  COPY ca-certificates.crt     # if making HTTPS calls

DOCKERFILE HYGIENE
  multi-stage build       # always
  cache-friendly order    # deps before source
  --mount=type=cache      # BuildKit cache mounts
  .dockerignore           # before optimizing anything else
  no UPX                  # not worth it

GO RUNTIME IN K8S
  GOMEMLIMIT from limits.memory * 0.8
  go.uber.org/automaxprocs    # respect cpu limit
  resources.limits.memory > peak working set + 20%

K8S RESOURCES
  requests = typical usage
  limits   = peak + headroom
  cpu over-limit = throttle
  memory over-limit = OOMKilled
  measure with kubectl top + container_memory_working_set_bytes

DEBUGGING DISTROLESS
  kubectl debug -it pod --image=busybox --target=container
```

## Wrapping up

The order I optimize in, when I take over a service with a 800 MB image and a pod that gets OOMKilled twice a day:

1. Add a `.dockerignore` and check the build context size. Often this alone fixes 50% of the problem.
2. Convert to multi-stage, move runtime to distroless or scratch.
3. Add `GOMEMLIMIT` from the downward API and `automaxprocs`. Memory behavior gets sane.
4. Look at actual memory usage in Prometheus over a week. Set requests to typical, limits to peak + 20%.
5. If it's a JVM or Node app, tune the language's memory flags to match the cgroup.

That sequence usually takes a service from "scary" to "boring" in a couple of afternoons. Boring is the goal. A boring container is one that pulls fast, starts fast, uses what it asked for, doesn't die at 3 AM, and has nothing inside it that an attacker can use as a stepping stone.
