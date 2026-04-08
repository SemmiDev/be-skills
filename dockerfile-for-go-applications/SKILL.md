---
name: dockerfile-for-go-applications
description: Production-grade, secure, and minimal Dockerfile best practices for Go backend projects (2026 standards). Uses multi-stage builds with `golang:1.xx-alpine` as builder and `scratch` (or `distroless/static`) for the final image. Incorporates layer caching optimization, static binary compilation (CGO_ENABLED=0), binary stripping, non-root user, HEALTHCHECK, labels, .dockerignore, and seamless integration with your existing skills (migrations, logging, database, etc.). Always use this template instead of single-stage or Alpine-only Dockerfiles.
---

# Dockerfile for Go Applications (Best Practices)

**Skill purpose**  
This skill defines the **official SemmiDev pattern** for writing production-ready `Dockerfile` for all Go services.  

It follows 2026 community and security best practices:
- **Multi-stage build** — keep build tools out of the final image.
- **Static binary** — `CGO_ENABLED=0` + `GOOS=linux`.
- **Minimal runtime** — `scratch` by default (or `gcr.io/distroless/static-debian12` when you need CA certificates / tzdata).
- **Layer caching** — copy `go.mod`/`go.sum` first.
- **Security** — non-root user, no shell in final image, minimal attack surface.
- **Observability** — labels, HEALTHCHECK, exposed port.
- **Reproducibility** — build args for version, commit, build time.

Result: Final image size typically **< 15 MB** (often ~8-12 MB), fast startup, excellent security posture, and fast CI builds.

## When to use this skill
- Any Go HTTP API, worker, CLI tool, or microservice.
- You want consistent, auditable, and secure container images across all projects.
- You are deploying to Kubernetes, Docker Swarm, Cloud Run, or any container platform.

**Never** use a single-stage Dockerfile, run as root, or include the full Go toolchain in production images.

## Recommended Project Files

Create these at the root of your Go project:

1. **Dockerfile** (main file)
2. **.dockerignore** (critical for build speed & security)

### .dockerignore (always include)

```
.git
.gitignore
README.md
Dockerfile
.dockerignore
*.md
.env
.env.*
*.test
coverage.out
db/migration/*.sql   # migrations are run separately
tmp/
```

## Canonical Dockerfile (2026 Best Practice)

```dockerfile
# =============================================
# Stage 1: Builder
# =============================================
FROM golang:1.23-alpine AS builder

# Install minimal build dependencies (ca-certificates for HTTPS in builder if needed)
RUN apk --no-cache add ca-certificates git

WORKDIR /app

# Copy dependency files first → best layer caching
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build static binary with optimizations
# - CGO_ENABLED=0     : fully static (no glibc dependency)
# - -ldflags="-w -s"  : strip debug symbols and DWARF info
# - -trimpath         : remove file system paths from binary
ARG VERSION="dev"
ARG COMMIT="unknown"
ARG BUILD_TIME="unknown"

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath \
    -ldflags="-w -s \
              -X main.Version=${VERSION} \
              -X main.Commit=${COMMIT} \
              -X main.BuildTime=${BUILD_TIME}" \
    -o /app/main ./cmd/api   # ← change path if your main is elsewhere (e.g. cmd/server)

# =============================================
# Stage 2: Final Runtime Image
# =============================================
# Option A: Ultra-minimal (recommended default for pure Go apps)
FROM scratch AS final

# Option B: If you need CA certificates or timezone data → use distroless instead
# FROM gcr.io/distroless/static-debian12 AS final

# Copy CA certificates (required for HTTPS calls to external APIs)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# (Optional) Copy timezone data if your app uses time.LoadLocation()
# COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Create non-root user (scratch doesn't have adduser, so we set UID/GID manually)
COPY --from=builder /app/main /main

# Metadata labels (standard OCI labels)
LABEL org.opencontainers.image.source="https://github.com/yourorg/yourrepo" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.created="${BUILD_TIME}" \
      org.opencontainers.image.revision="${COMMIT}"

# Security: run as non-root
USER 65532:65532

# Expose port (change according to your app)
EXPOSE 8080

# Healthcheck (optional but highly recommended)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/main", "health"]   # or use a simple TCP check if no health endpoint

# Run the binary directly (no shell)
ENTRYPOINT ["/main"]
```

## Build Commands (add to Makefile)

```makefile
.PHONY: docker-build
docker-build: ## Build Docker image with build args
	docker build \
		--build-arg VERSION=$(git describe --tags --always) \
		--build-arg COMMIT=$(git rev-parse --short HEAD) \
		--build-arg BUILD_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
		-t yourapp:$(git describe --tags --always) \
		--file Dockerfile .

.PHONY: docker-build-dev
docker-build-dev:
	docker build -t yourapp:dev --target builder .
```

## Key Best Practices (2026 Summary)

**Always**:
- Use **multi-stage** builds.
- Set `CGO_ENABLED=0` for static linking.
- Strip symbols with `-ldflags="-w -s"`.
- Copy `go.mod` + `go.sum` before source code (maximizes cache hits).
- Run as non-root user (UID 65532 is common convention).
- Use `scratch` for pure Go apps (smallest & most secure).
- Switch to `gcr.io/distroless/static-debian12` only if you need CA certs or tzdata.
- Add meaningful OCI labels.
- Include `.dockerignore`.
- Add `HEALTHCHECK`.

**Decision Tree**:
1. Pure static Go binary, no external HTTPS? → Use `scratch`.
2. Needs HTTPS calls (e.g. to payment gateways)? → Copy CA certificates + use `scratch` or `distroless/static`.
3. Needs time zones? → Copy `/usr/share/zoneinfo` from builder.
4. Needs debugging in image? → Use `alpine` only for dev/staging, never production.

**Never**:
- Use `golang` image as final stage.
- Run container as root.
- Include build tools or source code in production image.
- Omit `.dockerignore`.
- Hardcode version/commit without build args.

## Integration Notes for Your Backend Projects
- Place the `Dockerfile` at project root.
- Update the build path (`./cmd/api`) according to your project layout.
- Combine with `database-migrations-with-golang-migrate` — run migrations in CI/CD or entrypoint script (separate from app image if possible).
- Use the same image for all environments (dev uses different compose setup).
- Scan images with Trivy or Docker Scout in CI before pushing.
- For Kubernetes: add resource limits and liveness/readiness probes.

Follow this skill and every Docker image in your Go backends will be:
- Extremely small (< 15 MB)
- Secure by default
- Fast to build and deploy
- Consistent across the organization
