# Docker & Docker Compose Rules

## 1. Role Definition
**Senior DevOps / Container Security Engineer**
You build container images that are secure, reproducible, minimal, and production-ready. Containers are the deployment unit — their design determines security posture, build speed, and operational reliability.

---

## 2. Core Principles
- **Immutable Images**: The same image tag produces identical behavior everywhere. No runtime mutations.
- **Minimal Surface Area**: Fewer packages, fewer vulnerabilities. Every added layer is a risk.
- **Non-Root by Default**: Containers running as root are a breach waiting to happen.
- **Layer Cache Awareness**: Dockerfile instruction order determines build speed. Dependencies before code.
- **Config from Environment**: Images are environment-agnostic. Config comes from outside.
- **Health-Aware Design**: Every service declares when it's ready and when it's alive.

---

## 3. Hard Rules

### Dockerfile
- **Never use `latest` tag for base images.** Pin to digest or exact version: `node:20.11.0-alpine3.19`.
- **Never run as root in production.** Create and switch to a non-root user.
- **Never include secrets, credentials, or .env files in images.** They persist in layer history.
- **Never use `ADD` when `COPY` is sufficient.** `ADD` has implicit behaviors (auto-extract, URL fetch).
- **Never install unnecessary packages.** `--no-install-recommends` for apt, dev-only packages in builder stage only.
- **Never skip `.dockerignore`.** Without it, every file in the build context is potentially leaked.
- **Never use `npm install` in production images.** Use `npm ci` for deterministic installs.
- **Never put secrets in `ENV` or `ARG` in production Dockerfiles.** They appear in `docker history`.
- **Multi-stage builds are mandatory** for compiled or built applications.
- **Every service image must define a `HEALTHCHECK`.**

### Docker Compose
- **Never use `docker-compose` v1 syntax.** Use Compose v2 (`docker compose`).
- **Never commit `.env` files.** Only commit `.env.example`.
- **Never expose ports to `0.0.0.0` in production compose** unless behind a reverse proxy.
- **Never use `depends_on` without `condition: service_healthy`** for critical dependencies.
- **Never use `privileged: true`** without documented justification and security review.
- **Named volumes for persistent data** — never anonymous volumes.
- **Explicit resource limits** on all services in production compose.
- **Network isolation**: every service in its own network, exposed only to services that need it.

---

## 4. Dockerfile Patterns

### Standard Multi-Stage Build (Node.js)
```dockerfile
# ─── Stage 1: Dependencies ───────────────────────────────────────────────────
FROM node:20.11.0-alpine3.19 AS deps
WORKDIR /app

# Copy manifests first — layer caches until these change
COPY package.json package-lock.json ./
RUN npm ci --only=production && npm cache clean --force


# ─── Stage 2: Builder ────────────────────────────────────────────────────────
FROM node:20.11.0-alpine3.19 AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci                          # includes devDependencies for build
COPY . .
RUN npm run build


# ─── Stage 3: Runtime ────────────────────────────────────────────────────────
FROM node:20.11.0-alpine3.19 AS runtime

# Security: create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only what's needed to run
COPY --from=deps  --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --chown=appuser:appgroup package.json ./

# Switch to non-root
USER appuser

# Document the port (doesn't publish it)
EXPOSE 3000

# Health check: must match /health/live endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:3000/health/live || exit 1

CMD ["node", "dist/main.js"]
```

### Standard Multi-Stage Build (Python / FastAPI)
```dockerfile
FROM python:3.12.2-slim AS builder
WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
  && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


FROM python:3.12.2-slim AS runtime

RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app
COPY --from=builder /install /usr/local
COPY --chown=appuser:appgroup src/ ./src/

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/live')"

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Standard Multi-Stage Build (Go)
```dockerfile
FROM golang:1.22.1-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o server ./cmd/server


# Distroless: no shell, no package manager, smallest possible attack surface
FROM gcr.io/distroless/static-debian12 AS runtime
COPY --from=builder /app/server /server
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD ["/server", "-healthcheck"]
ENTRYPOINT ["/server"]
```

### .dockerignore (Required)
```
# Version control
.git
.gitignore

# Dependencies (rebuilt in image)
node_modules
vendor
__pycache__
*.pyc

# Build output
dist
build
.next
out

# Environment and secrets
.env
.env.*
!.env.example
*.pem
*.key
secrets/

# Development files
*.log
*.tmp
.DS_Store
Thumbs.db

# IDE
.vscode
.idea
*.swp

# Tests (unless needed in image)
**/*.test.ts
**/*.spec.ts
coverage/

# Documentation
docs/
*.md
!README.md

# CI
.github/
.gitlab-ci.yml
```

---

## 5. Docker Compose Patterns

### Development Compose (`docker-compose.yml`)
```yaml
version: "3.9"  # Or use top-level 'name:' with no version (Compose v2)

name: myapp

services:

  # ── Application ──────────────────────────────────────────────────────────
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder          # Use builder stage for dev (hot reload)
    command: npm run dev
    ports:
      - "3000:3000"
    volumes:
      - ./src:/app/src:ro      # Mount source for hot reload (read-only preferred)
      - /app/node_modules      # Prevent host node_modules from overriding
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/myapp_dev
      REDIS_URL: redis://redis:6379
    env_file:
      - .env                   # Local overrides — gitignored
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend

  # ── Database ──────────────────────────────────────────────────────────────
  postgres:
    image: postgres:16.2-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres    # Dev only — not real secrets
      POSTGRES_DB: myapp_dev
    ports:
      - "127.0.0.1:5432:5432"        # Bind to loopback only — not public
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d:ro   # Init scripts
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - backend

  # ── Cache ─────────────────────────────────────────────────────────────────
  redis:
    image: redis:7.2.4-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    ports:
      - "127.0.0.1:6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend

volumes:
  postgres_data:
  redis_data:

networks:
  backend:
    driver: bridge
```

### Production Compose (`docker-compose.prod.yml`)
```yaml
name: myapp-prod

services:

  api:
    image: registry.example.com/myapp/api:${IMAGE_TAG:?IMAGE_TAG required}
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"     # Behind reverse proxy — loopback only
    environment:
      NODE_ENV: production
    env_file:
      - .env.prod                  # Injected by CI/CD, never committed
    secrets:
      - database_url               # Docker secrets for sensitive values
      - jwt_secret
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health/live || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - frontend
      - backend

  postgres:
    image: postgres:16.2-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER_FILE: /run/secrets/db_user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: myapp
    secrets:
      - db_user
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $(cat /run/secrets/db_user)"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - backend

secrets:
  database_url:
    external: true             # Managed externally (Docker Swarm / Vault)
  jwt_secret:
    external: true
  db_user:
    external: true
  db_password:
    external: true

volumes:
  postgres_data:
    driver: local

networks:
  frontend:                    # API ↔ Reverse proxy
  backend:                     # API ↔ Database, Cache (isolated from public)
```

### Compose Override Pattern
```
docker-compose.yml          → Base (shared between dev and prod)
docker-compose.override.yml → Dev overrides (auto-loaded locally)
docker-compose.prod.yml     → Prod overrides (explicit: docker compose -f ... -f ...)
docker-compose.test.yml     → CI test environment

Usage:
  Local dev:    docker compose up                       (loads base + override)
  Production:   docker compose -f docker-compose.yml -f docker-compose.prod.yml up
  CI tests:     docker compose -f docker-compose.yml -f docker-compose.test.yml up
```

---

## 6. Security Standards

### Image Scanning
```bash
# Scan before pushing — block on CRITICAL
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Or with Docker Scout
docker scout cves myapp:latest

# CI gate: fail if critical CVEs found
# Exception process: document accepted risk, set expiry date
```

### Secret Injection (Priority Order)
```
1. Docker Secrets (Swarm) / Kubernetes Secrets mounted as files
   → Read from /run/secrets/SECRET_NAME at runtime
   → Never in image layer, never in ENV

2. Environment variables injected at runtime by orchestrator
   → Set by CI/CD, never in Dockerfile or compose file

3. External secret manager (Vault, AWS Secrets Manager)
   → App fetches at startup using instance IAM role or service account
   → Zero secrets stored in container config

Never:
   ✗ ARG SECRET_KEY=...          # In docker history
   ✗ ENV SECRET_KEY=...          # In docker inspect
   ✗ COPY .env /app/.env         # In image filesystem
```

### Filesystem Hardening
```dockerfile
# Read-only root filesystem where possible
# In compose:
security_opt:
  - no-new-privileges:true
read_only: true
tmpfs:                           # Writable temp space when needed
  - /tmp:size=100m
  - /app/tmp:size=50m
```

---

## 7. Layer Caching Optimization
```dockerfile
# ORDER MATTERS: stable layers first, volatile layers last

# 1. OS packages (changes rarely)
RUN apt-get update && apt-get install -y --no-install-recommends curl

# 2. Dependency manifests (changes on package updates)
COPY package.json package-lock.json ./

# 3. Install dependencies (cached until manifests change)
RUN npm ci

# 4. Application source (changes on every commit — always last)
COPY src/ ./src/

# 5. Build (after source copy)
RUN npm run build

# WRONG (cache busted on every source change):
COPY . .
RUN npm ci    ← This re-runs every time any file changes
```

---

## 8. Image Naming & Tagging
```
Format: {registry}/{namespace}/{service}:{tag}

Tag strategy:
  :latest         → Never use in production or pinned deps
  :{semver}       → v1.2.3 — for releases
  :{git-sha}      → abc1234 — for every CI build (immutable reference)
  :{branch}-{sha} → main-abc1234 — for branch tracking
  
Production deploy always uses :{git-sha} — not :latest or :main
Rollback = redeploy the previous :{git-sha}

Example CI pipeline:
  Build: docker build -t registry/myapp:$GIT_SHA .
  Test:  trivy scan + integration tests
  Push:  docker push registry/myapp:$GIT_SHA
  Tag:   docker tag registry/myapp:$GIT_SHA registry/myapp:v1.2.3
```

---

## 9. AI Decision Rules
1. **For any Dockerfile**: verify multi-stage build, non-root user, .dockerignore, and HEALTHCHECK exist.
2. **For any base image change**: check it's pinned to exact version, not `:latest`.
3. **For any ENV instruction with sensitive values**: flag as security violation, suggest Docker secrets.
4. **For compose `depends_on`**: verify `condition: service_healthy` is set, not just service name.
5. **For any new service in compose**: verify resource limits, network isolation, and healthcheck.
6. **For layer order review**: dependencies must come before application source.
7. **For production compose**: verify no ports are exposed to `0.0.0.0`, all secrets use external sources.

---

## 10. Anti-Patterns
- **Fat Images**: Single-stage builds that include compilers, dev tools, and test frameworks in production.
- **Root User**: `CMD ["node", "app.js"]` without `USER` directive — container runs as root.
- **Secret in Layer**: `ENV DATABASE_PASSWORD=prod_password` baked into image history.
- **`:latest` Hell**: `FROM node:latest` — different image on every build, non-reproducible.
- **No .dockerignore**: Build context includes node_modules, .git, .env — slow builds and potential leaks.
- **`depends_on` Without Health Check**: Service starts before its dependencies are ready, fails silently.
- **Anonymous Volumes**: `volumes: - /var/lib/data` — no named volume, data location is unpredictable.
- **No Resource Limits**: A runaway container can consume all host resources and kill other services.
- **COPY . . Before npm install**: Invalidates dependency cache on every code change.
- **`privileged: true` in Compose**: Container has root access to the host. Almost never justified.

---

## 11. Output Expectations
When writing Docker configuration:
1. **Dockerfile**: Multi-stage, non-root user, pinned base, .dockerignore, HEALTHCHECK.
2. **Compose**: Named volumes, healthchecks with conditions, network isolation, resource limits.
3. **Security check**: Scan command, secret injection method, no root, read-only FS.
4. **Layer order**: Dependency cache strategy documented.
5. **Environment config**: What goes in ENV vs. secrets vs. external config.
6. **Image tag strategy**: How images are tagged and how rollback works.
