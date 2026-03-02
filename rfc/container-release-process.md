# Container Image Release Process

## Summary

This RFC defines the standard process for building and publishing container images across Storacha Go services.

## Status

- **Status**: Review
- **Applies to**: piri, piri-signing-service, delegator, indexing-service, and more

## Goals

1. Multi-architecture support — containers should work on both `linux/amd64` and `linux/arm64`
2. Consistent tagging across all services
3. Clear separation of dev and prod images
4. Immutable tags for reproducibility
5. Mutable tags for convenience

## Container Registry

Images should be published to GitHub Container Registry (GHCR):

```
ghcr.io/storacha/<service-name>
```

## Tagging Strategy

The tagging scheme serves two audiences: humans who want the latest thing, and machines who want the exact thing. Both deserve to be happy.

### Push to main branch

A push to main should produce **both** dev and prod images:

| Tag | Target | Mutable | Description |
|-----|--------|---------|-------------|
| `main` | prod | Yes | Latest main branch (production build) |
| `main-dev` | dev | Yes | Latest main branch (dev build with debug tools) |
| `<sha>` | prod | No | Specific commit (production build) |
| `<sha>-dev` | dev | No | Specific commit (dev build) |

### Release (tag v*)

A release tag should produce **prod** images only. Nobody ships a debugger to production on purpose.

| Tag | Target | Mutable | Description |
|-----|--------|---------|-------------|
| `<version>` | prod | No | Semantic version (e.g., `1.2.3`) |

There is no `latest` tag. The `latest` tag is a convention that rewards ambiguity — it lets you deploy without knowing what you're deploying, which is the kind of convenience that generates incident reports. All references to images should specify an explicit version.

### Pull Requests

PR builds should validate that the Dockerfile compiles but need not push images anywhere. Building on a single platform (amd64) is sufficient here — the goal is fast feedback, not architectural completeness.

## Dockerfile Guidance

Each service should provide a multi-stage Dockerfile with at least two targets.

### `prod` target

The production target should be small, fast, and boring:

- Stripped binary (`-ldflags="-s -w"`)
- Minimal base image (`debian:bookworm-slim`)
- Only what's needed to run and health-check: `ca-certificates`, `curl`

### `dev` target

The dev target exists for the days when something has gone wrong and you need to figure out what. It should include:

- A debug-friendly binary (`-gcflags="all=-N -l"`)
- Delve debugger
- Enough diagnostic tools to be useful: `bash-completion`, `less`, `vim-tiny`, `procps`, `htop`, `strace`, `iputils-ping`, `dnsutils`, `net-tools`, `tcpdump`, `jq`

The precise selection of debugging tools is a matter of taste. The principle is not: include everything that may be required.

### Non-root user

Production images should use the `USER` directive to run as a non-root user. Running as root inside a container is the default, and like most defaults, it optimizes for the wrong thing. The cost is a couple of lines; the benefit is a smaller blast radius when something goes sideways.

```dockerfile
RUN useradd --system --no-create-home appuser
USER appuser
```

One thing to be aware of: a non-root user cannot bind to ports below 1024. If your service listens on port 80 or 443, you'll need to either remap the port at the orchestrator level or use a higher port internally. This is rarely a problem in practice — most services behind a load balancer can listen on whatever port they like — but it's the kind of thing that's confusing for exactly five minutes if you don't expect it.

### .dockerignore

Every repository with a Dockerfile should have a `.dockerignore` file. Without one, `COPY . .` in the build stage will send `.env` files, `.git` history, test fixtures, and whatever else happens to be lying around into the build context. This slows down builds, bloats layers, and occasionally leaks secrets — a trifecta of outcomes best avoided.

A reasonable starting point:

```
.git
.env*
*.md
LICENSE
docker-compose*.yml
.github
```

### Base image tags

This RFC recommends `debian:bookworm-slim` without pinning to a specific digest. Pinning to a digest (e.g., `debian:bookworm-slim@sha256:...`) would make builds fully reproducible and close a supply chain vector, but it also creates an ongoing maintenance burden: someone has to update the digest when security patches land.

For a team of Storacha's size, that tradeoff doesn't favor pinning. The `bookworm-slim` tag is maintained by Debian and widely relied upon across the industry. Tracking the tag and letting security patches arrive automatically is the pragmatic choice here. Teams with more headcount and stronger opinions about supply chain provenance may reasonably choose otherwise.

### Health checks

Whether to include a `HEALTHCHECK` directive in the Dockerfile is left to the implementer. Some services have an obvious health signal — an HTTP endpoint that returns 200 when things are fine. Others have more nuanced definitions of health that are better expressed in the orchestration layer, whether that's a Docker Compose healthcheck, a Kubernetes liveness probe, or something else entirely.

The RFC includes `curl` in the prod image to make container-level health checks possible. Whether to use it there or defer to the orchestrator is a judgment call per service.

## Image Signing

Image signing via cosign/sigstore is not part of this initial process but is a goal for later. Signing provides cryptographic verification that an image was produced by our CI pipeline and not by someone with access to the registry and a creative afternoon. As the team and infrastructure mature, this should be revisited.

## Cross-Compilation

For Go services, build stages should use the `--platform=$BUILDPLATFORM` pattern to avoid running the Go compiler under QEMU emulation. The difference between native cross-compilation and emulated compilation is the difference between waiting for your build and waiting for your retirement.

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.25-bookworm AS build
ARG TARGETARCH
ARG TARGETOS=linux
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -ldflags="-s -w" -o /app .

FROM debian:bookworm-slim AS prod
RUN apt-get update && apt-get install -y ca-certificates curl && rm -rf /var/lib/apt/lists/*
COPY --from=build /app /usr/bin/app
ENTRYPOINT ["/usr/bin/app"]
```

The key insight: `$BUILDPLATFORM` is the CI runner's architecture (x86_64), and `$TARGETPLATFORM` is the final image's architecture. Go handles cross-compilation natively, so the compiler runs fast while producing binaries for whatever architecture you need.

QEMU remains necessary for the runtime stages, where `apt-get` and friends must execute on the target architecture. These operations are lightweight compared to compilation, so the tradeoff is favorable.

## CI/CD Considerations

The workflow should live at `.github/workflows/publish-ghcr.yml` and be named `Container`.

Three jobs cover the necessary cases:

| Job | Trigger | Targets | Pushes Images |
|-----|---------|---------|---------------|
| Publish main | Push to main | dev + prod | Yes |
| Publish release | Push v* tag | prod only | Yes |
| Build check | Pull request | dev + prod | No |

A few things worth getting right:

**Multi-architecture builds** rely on QEMU for ARM64 emulation and Buildx for parallel builds. The publish jobs need both; PR builds can skip QEMU since they only target amd64.

**Caching** should use GitHub Actions cache (`type=gha`) with `mode=max` to export all layers. Each target (prod/dev) needs its own cache scope. Without scoping, parallel matrix jobs will cheerfully overwrite each other's caches — a form of cooperation that helps no one. PR builds should read from cache but not write to it.

**Concurrency control** should group builds by branch or tag and cancel in-progress runs when new commits arrive. This prevents the queue buildup that results from rapid pushes, a pattern familiar to anyone who has ever said "one more fix."

**Authentication** uses the automatic `GITHUB_TOKEN`, which requires `packages: write` permission for publish jobs.

Tag generation should go through `docker/metadata-action` rather than manual string interpolation. The action handles edge cases that you would rather not discover yourself.