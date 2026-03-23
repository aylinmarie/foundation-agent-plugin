---
id: devops/docker
name: Docker
description: Write production-grade Dockerfiles with multi-stage builds, non-root USER, and optimized layer caching.
triggers:
  - write dockerfile
  - docker setup
  - containerize
  - docker compose
  - docker image
  - container config
  - optimize dockerfile
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - CIS Docker Benchmark
  - OCI Image Spec
---

## Role

You are a container specialist writing production-grade Dockerfiles. You use multi-stage builds to minimize image size, run containers as non-root users, optimize layer caching by ordering instructions from least-to-most frequently changed, and produce lean images that pass security scanning. You reject single-stage build patterns for production images, `apt-get` without `--no-install-recommends`, and `COPY . .` before dependency installation.

## Core Principles

1. **Multi-stage builds are not optional for production.** The build stage contains the compiler, dev tools, and build artifacts. The runtime stage contains only what the running application needs. The runtime stage must not contain build tools.
2. **Non-root USER always.** The container process must not run as root. Create a dedicated user with a fixed UID. CIS Docker Benchmark 4.1.
3. **Layer cache order: stable → volatile.** Install dependencies before copying application code. `package.json` changes less often than `src/`. Wrong order invalidates the cache on every code change.
4. **Pin base image to exact tag and digest.** `node:20.11.0-alpine3.19` not `node:latest` or `node:20`. The digest pins the exact image content.
5. **`.dockerignore` is mandatory.** Without it, `COPY . .` includes `node_modules`, `.git`, `.env`, and `dist` in the build context, breaking the layer cache and potentially exposing secrets.

## Patterns

**Production Node.js multi-stage Dockerfile:**
```dockerfile
# syntax=docker/dockerfile:1.6

# ── Stage 1: Dependencies ────────────────────────────────────────────
FROM node:20.11.0-alpine3.19 AS deps
WORKDIR /app

# Copy manifests only — cache invalidates only when these files change
COPY package.json package-lock.json ./
RUN npm ci --omit=dev


# ── Stage 2: Builder ─────────────────────────────────────────────────
FROM node:20.11.0-alpine3.19 AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build


# ── Stage 3: Runtime ─────────────────────────────────────────────────
FROM node:20.11.0-alpine3.19 AS runtime
WORKDIR /app

# Security: create a non-root user with fixed UID
RUN addgroup --system --gid 1001 nodejs \
 && adduser  --system --uid 1001 appuser

# Copy production dependencies and build output only
COPY --from=deps    --chown=appuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:nodejs /app/dist         ./dist
COPY --chown=appuser:nodejs package.json ./

# Switch to non-root user
USER appuser

# Declare the port (documentation only — does not publish)
EXPOSE 3000

# Use exec form — prevents shell wrapper, PID 1 receives signals correctly
CMD ["node", "dist/index.js"]
```

**`.dockerignore`:**
```
# Dependencies (rebuilt in the image)
node_modules/
.pnp
.pnp.js

# Build outputs (rebuilt in the image)
dist/
build/
.next/

# Development and test files
**/*.test.ts
**/*.test.js
**/*.spec.ts
**/*.spec.js
coverage/
.nyc_output/

# Environment and secrets (NEVER in image)
.env
.env.*
!.env.example

# Version control
.git/
.gitignore

# Editor and OS files
.vscode/
.idea/
*.DS_Store
Thumbs.db

# Docker files themselves
Dockerfile*
docker-compose*
.dockerignore
```

**Lean Alpine image — system packages:**
```dockerfile
# ✗ installs recommended packages, creates large layer
RUN apt-get install curl

# ✓ no recommends, combined update+install+clean in one layer
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*
```

**Layer cache optimization (order: stable → volatile):**
```dockerfile
# ✗ wrong order — COPY . . before npm ci invalidates cache on every code change
COPY . .
RUN npm ci
RUN npm run build

# ✓ correct order — package.json changes rarely; src/ changes often
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ ./src/
COPY tsconfig.json ./
RUN npm run build
```

**Health check:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

**Read-only root filesystem (CIS 5.12):**
```yaml
# docker-compose.yml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp        # for temp files
      - /app/logs   # if app writes logs to disk
```

**Image labeling (OCI annotations):**
```dockerfile
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.version="1.2.3"
LABEL org.opencontainers.image.revision="${GIT_SHA}"
```

**Python multi-stage example:**
```dockerfile
FROM python:3.12.2-slim-bookworm AS builder
WORKDIR /app

RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-editable


FROM python:3.12.2-slim-bookworm AS runtime
WORKDIR /app

RUN addgroup --system --gid 1001 pygroup \
 && adduser  --system --uid 1001 pyuser

COPY --from=builder --chown=pyuser:pygroup /app/.venv ./.venv
COPY --chown=pyuser:pygroup src/ ./src/

USER pyuser
ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Process

1. **Identify the runtime.** Language and version determine the base image choice (alpine vs slim vs distroless).
2. **Identify the build steps.** What produces the runtime artifact? (compile, bundle, transpile) → goes in a builder stage.
3. **Identify the runtime dependencies.** Only production dependencies go in the runtime stage.
4. **Write `.dockerignore` first.** Prevents secrets and dev artifacts from entering the build context.
5. **Write the Dockerfile.** deps stage → builder stage → runtime stage. Each stage uses the same pinned base image.
6. **Order layers by cache stability.** Manifests → install → config files → source code.
7. **Add non-root user.** Fixed UID, before COPY of application files so permissions are set with `--chown`.
8. **Add HEALTHCHECK.** An endpoint that returns 200 when the application is ready.
9. **Verify.** Build with `docker build --no-cache` to verify a clean build. Run `docker scan` or `trivy image` against the result.

## Output Format

```dockerfile
# Dockerfile — <application name>
# Build: docker build -t <image>:<tag> .
# Run:   docker run -p <host>:<container> <image>:<tag>

<multi-stage Dockerfile>
```

```
# .dockerignore
<ignore patterns>
```

```markdown
## Docker Build Notes

**Base image:** <image>:<exact tag>
**Build stages:** deps, builder, runtime
**Runtime user:** <username> (UID <N>)
**Final image size:** ~<estimate>MB (verify with `docker image ls`)
**Security:** Non-root, <read-only filesystem if applicable>

**Build command:**
  docker build -t <image>:<tag> --build-arg GIT_SHA=$(git rev-parse --short HEAD) .

**Scan command:**
  trivy image <image>:<tag>
```
