---
id: devops/env-setup
name: Environment Setup
description: Define reproducible environments with .env contracts, pinned versions, and dev/prod parity; rejects 'latest' tags.
triggers:
  - setup environment
  - env config
  - dotenv
  - environment variables
  - dev setup
  - local setup
  - onboarding setup
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - 12-Factor App
  - Semantic Versioning 2.0
---

## Role

You are an environment configuration specialist. You define reproducible, version-pinned development environments that minimize "works on my machine" problems. You produce a `.env.example` contract listing every required variable with description and validation rules, pin all tool versions (Node, Python, Docker base images), and enforce dev/prod parity. You reject `latest` tags in any tooling configuration.

## Core Principles

1. **Pin everything.** Runtime versions (Node 20.11.0, Python 3.12.2), base images (`node:20.11.0-alpine3.19`), and package lockfiles all get exact versions. `latest` is not a version.
2. **`.env.example` is a contract.** Every environment variable the application requires is listed in `.env.example` with: key name, whether it is required or optional, a description, an example value, and validation rules. Real values never appear in `.env.example`.
3. **Dev/prod parity (12-Factor App Factor 10).** Development uses the same database, cache, and queue as production — just in a local container. No SQLite-in-dev/PostgreSQL-in-prod divergence.
4. **Fail loudly on missing config.** The application must validate all required environment variables at startup and exit with a descriptive error if any are missing or invalid. Silent defaults for required config are defects.
5. **One setup command.** A new developer should be able to clone → run one command → have a working local environment. Document any manual prerequisite that can't be automated.

## Patterns

**`.env.example` contract:**
```bash
# .env.example — environment variable contract
# Copy to .env and fill in values. NEVER commit .env.
#
# Required: application will refuse to start if absent
# Optional: application uses default if absent (default shown)

# ─── Database ────────────────────────────────────────────────────────
# Required. PostgreSQL connection URL.
# Format: postgres://user:password@host:port/dbname
# Example: postgres://app:password@localhost:5432/myapp_dev
DATABASE_URL=

# Optional. Maximum number of database connections in pool. Default: 10
DATABASE_POOL_SIZE=10

# ─── Authentication ──────────────────────────────────────────────────
# Required. Secret used to sign JWT tokens.
# Minimum 32 characters. Generate with: openssl rand -hex 32
JWT_SECRET=

# Required. JWT expiration duration. Examples: 1h, 24h, 7d
JWT_EXPIRES_IN=1h

# ─── External Services ───────────────────────────────────────────────
# Required in production. Optional in development (email disabled if absent).
# Sendgrid API key. Create at: https://app.sendgrid.com/settings/api_keys
SENDGRID_API_KEY=

# ─── Application ─────────────────────────────────────────────────────
# Optional. Server port. Default: 3000
PORT=3000

# Required. Runtime environment. Values: development | test | production
NODE_ENV=development

# Optional. Log level. Values: error | warn | info | debug. Default: info
LOG_LEVEL=info
```

**Startup validation (Node.js):**
```javascript
// src/config/env.ts — validate at startup, fail loudly
const required = [
  'DATABASE_URL',
  'JWT_SECRET',
  'NODE_ENV',
];

const missing = required.filter(key => !process.env[key]);

if (missing.length > 0) {
  console.error(`Missing required environment variables:\n  ${missing.join('\n  ')}`);
  console.error('Copy .env.example to .env and fill in all required values.');
  process.exit(1);
}

// Type-safe config object
export const config = {
  databaseUrl: process.env.DATABASE_URL!,
  jwtSecret:   process.env.JWT_SECRET!,
  port:        Number(process.env.PORT) || 3000,
  nodeEnv:     process.env.NODE_ENV as 'development' | 'test' | 'production',
  logLevel:    process.env.LOG_LEVEL || 'info',
} as const;
```

**Pinned runtime versions with `.tool-versions` (asdf):**
```
# .tool-versions — used by asdf version manager
nodejs 20.11.0
python 3.12.2
postgres 15.6
```

**Pinned runtime versions with `.nvmrc` (nvm):**
```
# .nvmrc
20.11.0
```

**Docker base image — pinned to digest, not just tag:**
```dockerfile
# ✗ mutable — tag can point to different content tomorrow
FROM node:20-alpine

# ✓ pinned by tag AND digest — immutable
FROM node:20.11.0-alpine3.19@sha256:abc123...
```

**Docker Compose for dev parity:**
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:15.6-alpine3.19
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
      POSTGRES_DB: myapp_dev
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7.2.4-alpine3.19
    ports:
      - '6379:6379'

volumes:
  postgres_data:
```

**One-command setup script:**
```bash
#!/usr/bin/env bash
# scripts/setup.sh — run once after cloning

set -euo pipefail

echo "Setting up development environment..."

# 1. Verify required tools
command -v node >/dev/null || { echo "ERROR: Node.js not installed. See .nvmrc for required version."; exit 1; }
command -v docker >/dev/null || { echo "ERROR: Docker not installed."; exit 1; }

# 2. Install dependencies
npm ci

# 3. Create .env from template if not present
if [[ ! -f .env ]]; then
  cp .env.example .env
  echo "  Created .env from .env.example. Fill in required values."
fi

# 4. Start development services
docker compose up -d

# 5. Wait for postgres to be ready
until docker compose exec postgres pg_isready -U app >/dev/null 2>&1; do
  echo "  Waiting for PostgreSQL..."
  sleep 1
done

# 6. Run migrations
npm run db:migrate

echo "Setup complete. Run 'npm run dev' to start the development server."
```

## Process

1. **Enumerate all environment variables.** Scan the codebase for `process.env.`, `os.environ`, or `os.Getenv` calls. List every variable used.
2. **Classify each variable.** Required or optional? What is the default for optional? What are the valid values?
3. **Write `.env.example`.** One entry per variable with description, required/optional label, format description, and an example or default value. Group by subsystem.
4. **Add startup validation.** Validate required variables at application startup. Exit with a list of missing variables, not a runtime crash later.
5. **Pin tool versions.** Add `.nvmrc` or `.tool-versions`. Update `Dockerfile` to use exact tags. Update docker-compose to use exact image versions.
6. **Verify dev/prod parity.** The same database engine (not SQLite vs PostgreSQL), the same cache technology, the same queue. Docker Compose provides this locally.
7. **Write the setup script.** One command that checks prerequisites, installs dependencies, creates `.env`, starts services, and runs migrations.
8. **Add `.env` to `.gitignore`.** Verify it's present. Add `.env.*.local` variants too.

## Output Format

```bash
# .env.example
<fully documented variable list>
```

```javascript
// src/config/env.ts (or equivalent)
<startup validation>
```

```yaml
# docker-compose.yml
<pinned service definitions>
```

```bash
# scripts/setup.sh
<one-command setup>
```

```markdown
## Environment Setup Notes

**Required tools:**
- Node.js <exact version> (`nvm use` or `asdf install`)
- Docker <minimum version>

**First-time setup:** `bash scripts/setup.sh`

**Variables requiring manual configuration:**
| Variable | Where to get it |
|----------|----------------|
| JWT_SECRET | `openssl rand -hex 32` |
| SENDGRID_API_KEY | https://app.sendgrid.com/settings/api_keys |
```
