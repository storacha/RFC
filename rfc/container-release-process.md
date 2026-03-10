# Container Image Release Process

## Summary

This RFC defines the standard process for building and publishing container images across Storacha Go services.

## Status

- **Status**: Review
- **Applies to**: All deployable golang Storacha services

## Goals

1. Multi-architecture support — containers should work on both `linux/amd64` and `linux/arm64`
2. Consistent tagging across all services
3. Immutable tags for reproducibility
4. Mutable tags for convenience

## Container Registry

Images should be published to GitHub Container Registry (GHCR):

```
ghcr.io/storacha/<service-name>
```

## Tagging Strategy

The tagging scheme serves two audiences: humans who want the latest thing, and machines who want the exact thing. 

### Push to main branch

| Tag           | Mutable | Description                           |
|---------------|---------|---------------------------------------|
| `main`        | Yes     | Latest main branch                    |
| `sha-<short>` | No      | Specific commit, short SHA            |

### Release (tag v*)

| Tag         | Mutable | Description                      |
|-------------|---------|----------------------------------|
| `<version>` | No      | Semantic version (e.g., `1.2.3`) |

Note that container tags omit the `v` prefix common in git tags. A git tag of `v1.2.3` produces a container tag of `1.2.3`.
This matches the convention used by major container images (nginx, postgres, node, etc.).

SHA tags use the short form (7 characters by default, e.g., `sha-90dd603`) for readability.
The full 40-character SHA is preserved in the OCI label `org.opencontainers.image.revision` for precise traceability.
This is the default behavior of `docker/metadata-action`.

There is no `latest` tag. The `latest` tag is a convention that rewards ambiguity — it lets you deploy without knowing what you're deploying, which is the kind of convenience that generates incident reports.
All references to images should specify an explicit version.

### Pull Requests

PR builds should validate that the Dockerfile compiles but need not push images anywhere.
Building on a single platform (amd64) is sufficient here — the goal is fast feedback, not architectural completeness.

## Dockerfile Guidance

### Runtime base image

All runtime stages should use `alpine:latest` as the default base.
Alpine is roughly 5MB, ships with `wget` and CA certificates pre-installed, includes a shell for Docker Compose healthchecks, and requires no package installation in the runtime stage — which means no `RUN apt-get` and, consequently, no QEMU emulation for runtime layers during cross-platform builds.

This single choice resolves several concerns simultaneously: healthchecks work out of the box (via `wget`), the image is minimal, and multi-architecture builds are fast because the runtime stage contains no emulated commands beyond `adduser`.

For services where `wget` as a runtime dependency is undesirable, an alternative is to compile a small, statically-linked Go binary whose only job is to HTTP GET an endpoint and exit with the appropriate status code. 
Copy it into the image at build time alongside the service binary. 
This eliminates the dependency on Alpine's `wget` — or on Alpine at all — at the cost of building and maintaining one more binary. 
The tradeoff is reasonable for images with strict dependency requirements; for most services, `wget` is already there and works fine.

### Production image

The production image should be small, fast, and boring:

- Stripped binary (`-ldflags="-s -w"`)
- `alpine:latest` base (CA certificates and `wget` pre-installed)
- Non-root user
- No package installation required

### Debug images

A dev target with debug tooling — Delve, diagnostic utilities, an unstripped binary — is a useful thing to have. 
It is not standardized here. 
Individual services may define their own dev target as needed.
When that day comes, whoever needs it can write a Dockerfile. 
It will take twenty minutes.

### Entrypoint scripts

Services using Alpine must write entrypoint scripts for the POSIX shell:

```sh
#!/bin/sh
# NOT #!/bin/bash — Alpine uses ash
```

Alpine's shell (ash, via busybox) is POSIX-compliant but lacks bash-specific features like arrays, `[[` tests, and process substitution.
Most scripts work unchanged; those that don't are usually easy to fix.

### Non-root user

Production images should use the `USER` directive to run as a non-root user.
Running as root inside a container is the default, and like most defaults, it optimizes for the wrong thing.
The cost is a couple of lines; the benefit is a smaller blast radius when something goes sideways.

```dockerfile
RUN adduser -D -H appuser
USER appuser
```

One thing to be aware of: a non-root user cannot bind to ports below 1024 by default. 
The simplest fix is setting `net.ipv4.ip_unprivileged_port_start=0` via `sysctls` in your Compose file or orchestrator.
Alternatively, use a higher port internally and remap at the orchestrator level. 
Most services behind a load balancer can listen on whatever port they like.

### Health checks

All services should support a Docker Compose healthcheck.
These execute inside the container, which means the container must have a tool capable of making HTTP requests.

Alpine ships with `wget`, so the standard healthcheck pattern is:

```yaml
healthcheck:
  test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/readyz"]
  interval: 10s
  timeout: 5s
  retries: 3
```

Whether to include a `HEALTHCHECK` directive directly in the Dockerfile or defer to the orchestration layer (Compose, Kubernetes liveness probe, etc.) is left to the implementer.
The RFC ensures the capability exists in every image; where you wire it up is a judgment call per service.

### .dockerignore

Every repository with a Dockerfile should have a `.dockerignore` file.
Without one, `COPY . .` in the build stage will send `.env` files, `.git` history, test fixtures, and whatever else happens to be lying around into the build context.
This slows down builds, bloats layers, and occasionally leaks secrets — a trifecta of outcomes best avoided.

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

This RFC recommends `alpine:latest` without pinning to a specific digest.
Pinning to a digest (e.g., `alpine@sha256:...`) would make builds fully reproducible and close a supply chain vector, but it also creates an ongoing maintenance burden: someone has to update the digest when security patches land.

For a team of Storacha's size, that tradeoff doesn't favor pinning.
The `alpine:latest` tag tracks the most recent stable release and is widely relied upon across the industry.
Teams with more headcount and stronger opinions about supply chain provenance may reasonably choose otherwise.

## Image Signing

Image signing via cosign/sigstore is not part of this initial process but is a goal for later.
Signing provides cryptographic verification that an image was produced by our CI pipeline and not by someone with access to the registry and a creative afternoon.
As the team and infrastructure mature, this should be revisited.

## Cross-Compilation

For Go services, build stages should use the `--platform=$BUILDPLATFORM` pattern to avoid running the Go compiler under QEMU emulation.
The difference between native cross-compilation and emulated compilation is the difference between waiting for your build and waiting for your retirement.

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.25-bookworm AS build
ARG TARGETARCH
ARG TARGETOS=linux
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -ldflags="-s -w" -o /app .

FROM alpine:latest AS prod
RUN adduser -D -H appuser
USER appuser
COPY --from=build /app /usr/bin/app
ENTRYPOINT ["/usr/bin/app"]
```

The key insight: `$BUILDPLATFORM` is the CI runner's architecture (x86_64), and `$TARGETPLATFORM` is the final image's architecture.
Go handles cross-compilation natively, so the compiler runs fast while producing binaries for whatever architecture you need.

Because Alpine ships with everything the prod image needs — CA certificates, `wget`, a shell — the runtime stage requires no `RUN` commands that execute on the target architecture (apart from `adduser`, which writes to `/etc/passwd` and is trivial under emulation).
This eliminates QEMU as a meaningful factor in production image builds.

## CI/CD Considerations

The workflow should live at `.github/workflows/publish-ghcr.yml` and be named `Container`.

Three jobs cover the necessary cases:

| Job             | Trigger      | Pushes Images |
|-----------------|--------------|---------------|
| Publish main    | Push to main | Yes           |
| Publish release | Push v* tag  | Yes           |
| Build check     | Pull request | No            |

A few things worth getting right:

**Multi-architecture builds** rely on QEMU for ARM64 emulation and Buildx for parallel builds.
The publish jobs need both; PR builds can skip QEMU since they only target amd64.
With Alpine as the runtime base and Go cross-compilation in the build stage, QEMU does very little work — `adduser` is the only emulated command in the production image.

**Caching** should use GitHub Actions cache (`type=gha`) with `mode=max` to export all layers.
PR builds should read from cache but not write to it.

**Concurrency control** should group builds by branch or tag and cancel in-progress runs when new commits arrive.
This prevents the queue buildup that results from rapid pushes, a pattern familiar to anyone who has ever said "one more fix."

**Authentication** uses the automatic `GITHUB_TOKEN`, which requires `packages: write` permission for publish jobs.

Tag generation should go through `docker/metadata-action` rather than manual string interpolation.
The action handles edge cases that you would rather not discover yourself.