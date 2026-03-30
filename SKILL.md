---
name: api-project-generator
description: >
  Generates complete, production-ready Node.js/Express REST API projects as a
  downloadable ZIP. Use this skill whenever the user asks to create, scaffold,
  generate, or bootstrap an API, REST service, backend, or Express app. Triggers
  on phrases like: generate an API, create an API project, scaffold a REST API,
  build an Express API, make an API for X, Node.js API for a topic. Always use
  this skill for API generation tasks rather than writing ad-hoc code inline.
---

# API Project Generator Skill

Generates a complete, downloadable Node.js/Express REST API project scaffold based on user requirements. Output is a ZIP file written to `/mnt/user-data/outputs/` and presented via `present_files`.

## Quick Decision Checklist

Before generating, confirm or infer these from the conversation:

| Setting | Options | Default |
|---|---|---|
| **API domain** | Any (e.g. contacts, products, orders) | Required — ask if missing |
| **Auth mode** | `none`, `apikey` | `apikey` |
| **Data mode** | `mock`, `postgres` | `mock` |
| **Pagination** | `offset` (always included) | on |
| **Extras** | docker, ratelimit, logging, validation | all on |

If the user gives enough context to infer these, proceed without asking. Only ask for the **API domain** if it's genuinely unclear.

---

## Project Structure to Generate

```
{project-name}/
├── .devcontainer/
│   └── devcontainer.json      # Codespaces config — Node.js image + postgres via docker run
├── scripts/
│   └── start-postgres.sh      # Idempotent postgres start script used by devcontainer
├── src/
│   ├── index.js               # Entry point
│   ├── app.js                 # Express app setup
│   ├── routes/
│   │   ├── health.js          # GET /health — always included
│   │   ├── apidocs.js         # GET /api-docs — Swagger UI, always included
│   │   └── {resource}.js      # Main resource routes
│   ├── controllers/
│   │   └── {resource}.js
│   ├── middleware/
│   │   ├── auth.js            # Auth middleware
│   │   └── pagination.js      # Offset/limit pagination
│   ├── db/
│   │   ├── client.js          # pg Pool setup
│   │   └── schema.sql         # Table definitions
│   └── data/
│       ├── {resource}.json    # Mock data seed file
│       └── seed.js            # Seed script
├── openapi.yaml               # OpenAPI 3.0 spec — always included
├── .env.example
├── .gitignore
├── Dockerfile
├── docker-compose.yml         # Postgres only (safe for Codespaces — no .env dependency)
├── docker-compose.full.yml    # Full stack (api + postgres) for local Docker use
├── package.json
└── README.md
```

---

## File Generation Instructions

### 1. `src/index.js`
Simple server start:
```js
require('dotenv').config();
const app = require('./app');
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

### 2. `src/app.js`
- Add `app.set('trust proxy', 1)` immediately after creating the Express app — required in Codespaces and any reverse-proxied environment so `express-rate-limit` can correctly identify client IPs from the `X-Forwarded-For` header. Without this, rate limiting throws `ERR_ERL_UNEXPECTED_X_FORWARDED_FOR` and crashes requests.
- Configure `express-rate-limit` with `validate: { xForwardedForHeader: false }` to suppress the proxy header warning (it is handled by `trust proxy` above)
- Import and configure express, cors, helmet (with `contentSecurityPolicy: false` so Swagger UI loads), morgan
- Apply rate limiter middleware — read limit from `RATE_LIMIT_WINDOW_MS` and `RATE_LIMIT_MAX` env vars
- Mount `/health` and `/api-docs` routes (unauthenticated, before auth middleware)
- Mount `/{resource}` route behind auth middleware
- Export app (not listen — that's in index.js)

### 3. Health Route (`src/routes/health.js`)
Always include. Returns:
```json
{ "status": "ok", "timestamp": "...", "version": "1.0.0", "service": "{project-name}" }
```
- Version read from `process.env.API_VERSION || '1.0.0'`
- No auth required on this endpoint

### 3b. API Docs Route (`src/routes/apidocs.js`) — Always include

Serve Swagger UI at `/api-docs`. No auth required. Uses `swagger-ui-express` and loads `openapi.yaml` from the project root.

```js
const express = require('express');
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');
const path = require('path');
const router = express.Router();

const spec = YAML.load(path.join(__dirname, '../../openapi.yaml'));

router.use('/', swaggerUi.serve);
router.get('/', swaggerUi.setup(spec));

module.exports = router;
```

Mount in `app.js` as: `app.use('/api-docs', apiDocsRouter);` — before the auth middleware so it's publicly accessible.

### 3c. `openapi.yaml` — Always include

Generate a complete OpenAPI 3.0 spec for the API. Place in the project root. Must include:

```yaml
openapi: 3.0.0
info:
  title: {Project Name} API
  version: 1.0.0
  description: |
    RESTful API for managing {resource}. Secured via API key.

servers:
  - url: http://localhost:{PORT}
    description: Local development

components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: x-api-key

  schemas:
    {Resource}:
      # All fields for the resource including id, created_at, updated_at
    {Resource}Input:
      # Required + optional input fields (no id/timestamps)
    PaginationMeta:
      type: object
      properties:
        total: { type: integer }
        page: { type: integer }
        limit: { type: integer }
        totalPages: { type: integer }
        hasNext: { type: boolean }
        hasPrev: { type: boolean }
    Error:
      type: object
      properties:
        error: { type: string }

security:
  - ApiKeyAuth: []

paths:
  /health:
    get:
      summary: Health check
      security: []   # No auth
      tags: [Health]
      responses:
        '200':
          description: Service is healthy

  /{resource}:
    get:
      summary: List {resource} (paginated)
      tags: [{Resource}]
      parameters:
        - in: query
          name: page
          schema: { type: integer, default: 1 }
        - in: query
          name: limit
          schema: { type: integer, default: 10 }
      responses:
        '200':
          description: Paginated list
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/{Resource}'
                  pagination:
                    $ref: '#/components/schemas/PaginationMeta'
        '401':
          $ref: '#/components/responses/Unauthorized'
    post:
      # ... create endpoint

  /{resource}/{id}:
    get: # ... getById
    put: # ... update
    delete: # ... delete

  /api-docs:
    get:
      summary: Swagger UI — interactive API documentation
      security: []
      tags: [Docs]
      responses:
        '200':
          description: Swagger UI HTML page
```

Write the full spec with all paths, request bodies, response schemas, and examples fully filled in — no placeholders. Use realistic example values matching the mock data.

### 4. Auth Middleware (`src/middleware/auth.js`)

Generate based on `AUTH_MODE` env var. The middleware should switch on `process.env.AUTH_MODE`:

**`none`** — pass through, no check. Useful for development or internal-only APIs.

**`apikey`** (default) — check `x-api-key` header against `process.env.API_KEY`. Return 401 if header is missing. Return 401 if key does not match. No other auth modes are supported.

> Auth mode is runtime-switchable via `AUTH_MODE` env var.

### 5. Pagination Middleware (`src/middleware/pagination.js`)
Parse `?page` and `?limit` from query string:
- Default: `page=1`, `limit=process.env.DEFAULT_PAGE_SIZE || 10`
- Max limit: `process.env.MAX_PAGE_SIZE || 100`
- Attach `{ page, limit, offset }` to `req.pagination`
- Include a `buildPaginationMeta(total, page, limit)` helper that returns `{ total, page, limit, totalPages, hasNext, hasPrev }`

### 6. Resource Routes + Controller

Generate CRUD routes for the named resource:
- `GET /{resource}` — list with pagination
- `GET /{resource}/:id` — get by id
- `POST /{resource}` — create
- `PUT /{resource}/:id` — update
- `DELETE /{resource}/:id` — delete

**Mock mode**: controller reads/writes the in-memory array loaded from `src/data/{resource}.json` at startup.

**Postgres mode**: controller uses `pg` Pool from `src/db/client.js`. Write real parameterized SQL queries. Use `RETURNING *` on insert/update.

Input validation (if enabled): use `express-validator` check chains on POST/PUT routes. Validate required fields inferred from the domain (e.g. for a contacts API: name, email).

### 7. Database Files (postgres mode only)

**`src/db/client.js`**:
```js
const { Pool } = require('pg');
module.exports = new Pool({ connectionString: process.env.DATABASE_URL });
```

**`src/db/schema.sql`**: CREATE TABLE IF NOT EXISTS with appropriate columns for the domain. Always include `id SERIAL PRIMARY KEY`, `created_at TIMESTAMPTZ DEFAULT NOW()`, `updated_at TIMESTAMPTZ DEFAULT NOW()`.

### 8. Mock Data

**`src/data/{resource}.json`**: Array of 5–10 realistic sample objects matching the domain. Include an `id` field on each.

**`src/data/seed.js`**:
- Postgres mode: reads the JSON file, connects to DB, inserts all rows. Runnable via `node src/data/seed.js`. Includes a `--clear` flag to truncate first.
- Mock mode: prints a friendly message explaining mock data is loaded from the JSON file automatically and no seeding is needed. Still valid to run, just informational.

### 9. `.env.example`

Always include ALL env vars the project uses, with placeholder values and inline comments:

```
# ─────────────────────────────────────────────
# Server
# ─────────────────────────────────────────────
PORT=3000
API_VERSION=1.0.0
NODE_ENV=development

# ─────────────────────────────────────────────
# Authentication
# Set AUTH_MODE to one of:
#   none    — No auth (open API, for development only)
#   apikey  — API key via x-api-key request header
# ─────────────────────────────────────────────
AUTH_MODE=apikey
API_KEY=your-api-key-here

# ─────────────────────────────────────────────
# Data
# Set DATA_MODE to one of:
#   mock     — In-memory data loaded from src/data/{resource}.json (no DB needed)
#   postgres — PostgreSQL via pg (use DATABASE_URL below)
#
# PostgreSQL is always available via docker-compose regardless of DATA_MODE.
# ─────────────────────────────────────────────
DATA_MODE=postgres

# PostgreSQL connection string
# In Codespaces / docker-compose (API inside container): hostname is "postgres"
#   DATABASE_URL=postgresql://api_user:api_pass@postgres:5432/{project_name}_db
# Running API natively (npm run dev) against docker-compose postgres: use "localhost"
#   DATABASE_URL=postgresql://api_user:api_pass@localhost:5432/{project_name}_db
# External (Neon, Supabase, Railway, etc.): use your provider's connection string
DATABASE_URL=postgresql://api_user:api_pass@localhost:5432/{project_name}_db

# ─────────────────────────────────────────────
# Pagination
# ─────────────────────────────────────────────
DEFAULT_PAGE_SIZE=10
MAX_PAGE_SIZE=100

# ─────────────────────────────────────────────
# Rate Limiting
# ─────────────────────────────────────────────
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX=100

# ─────────────────────────────────────────────
# Logging
# ─────────────────────────────────────────────
LOG_LEVEL=info
```

### 10. `.gitignore`

```
node_modules/
.env
*.log
dist/
.DS_Store
coverage/
.nyc_output/

# Local postgres data — persists in Codespaces but should not be committed
.pgdata/
```

### 11. `package.json`

Include scripts: `start`, `dev` (nodemon), `seed` (`node src/data/seed.js`).

Dependencies based on selections:
- Always: `express`, `dotenv`, `cors`, `helmet`, `swagger-ui-express`, `yamljs`
- Logging: `morgan`
- Rate limiting: `express-rate-limit`
- Validation: `express-validator`
- Postgres: `pg`
- Dev: `nodemon`

### 12. `Dockerfile`

node:20-alpine, WORKDIR /app, COPY package*, RUN npm ci, COPY ., EXPOSE $PORT, CMD node src/index.js

### 13. `docker-compose.yml` — Postgres only for Codespaces

**Important**: In Codespaces, docker-compose is used only to run the postgres service. The API itself runs natively via `npm run dev`. Do NOT include an `api` service with `env_file: - .env` — docker-compose validates the config before the Codespace starts, and `.env` doesn't exist yet (it's gitignored), causing container creation to fail.

Provide two compose files:

**`docker-compose.yml`** — postgres only (used in Codespaces and for local dev without the API containerized):

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: {project_name}_db
      POSTGRES_USER: api_user
      POSTGRES_PASSWORD: api_pass
    ports:
      - "5432:5432"
    volumes:
      - ./.pgdata:/var/lib/postgresql/data
      - ./src/db/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U api_user -d {project_name}_db"]
      interval: 5s
      timeout: 5s
      retries: 5
```

**`docker-compose.full.yml`** — full stack including the API (for local Docker-only deployments where `.env` is present):

```yaml
version: '3.9'

services:
  api:
    build: .
    ports:
      - "${PORT:-3000}:3000"
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: {project_name}_db
      POSTGRES_USER: api_user
      POSTGRES_PASSWORD: api_pass
    ports:
      - "5432:5432"
    volumes:
      - ./.pgdata:/var/lib/postgresql/data
      - ./src/db/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U api_user -d {project_name}_db"]
      interval: 5s
      timeout: 5s
      retries: 5
```

> No `volumes:` top-level block needed — the bind mount (`./.pgdata`) is self-contained.

> **Note for README**: When the API runs natively (`npm run dev`) against the docker-compose postgres, use `localhost` as the DATABASE_URL hostname. When using `docker-compose.full.yml`, use `postgres` as the hostname.

### 14. `.devcontainer/devcontainer.json` — Always include

Do NOT use `dockerComposeFile` — it causes the devcontainer to attach to the wrong container (e.g. postgres alpine, which has no Node.js or npm). Instead use a Node.js base image directly and start postgres as a sidecar via `postCreateCommand`.

```json
{
  "name": "{Project Name} API",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:20",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {
      "moby": false
    }
  },
  "forwardPorts": [3000, 5432],
  "portsAttributes": {
    "3000": { "label": "API Server", "onAutoForward": "notify" },
    "5432": { "label": "PostgreSQL", "onAutoForward": "silent" }
  },
  "postCreateCommand": "npm install && cp .env.example .env && bash scripts/start-postgres.sh && npm run seed",
  "postStartCommand": "bash scripts/start-postgres.sh && npm run seed",
  "customizations": {
    "vscode": {
      "extensions": [
        "dbaeumer.vscode-eslint",
        "PKief.material-icon-theme",
        "mtxr.sqltools",
        "mtxr.sqltools-driver-pg"
      ],
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash"
      }
    }
  }
}
```

> **Notes**:
> - `image: mcr.microsoft.com/devcontainers/javascript-node:20` — gives Node 20 and npm
> - `features: docker-in-docker:2` with `"moby": false` — installs Docker CLI and daemon; `moby: false` is required because the `javascript-node:20` image is based on Debian Trixie which dropped Moby packages
> - `scripts/start-postgres.sh` blocks until `pg_isready` confirms postgres is accepting connections — no `sleep` needed
> - `postCreateCommand` runs once on first setup: installs deps, copies `.env`, starts postgres, seeds initial data
> - `postStartCommand` runs on every Codespace restart: restarts/recreates postgres (preserving `.pgdata/`), then seeds (safe — `ON CONFLICT DO NOTHING`)
> - After the Codespace starts, open a terminal and run `npm run dev`
> - `DATABASE_URL` must use `localhost` since postgres is exposed on the Codespace host network
> - `app.js` must include `app.set('trust proxy', 1)` and `validate: { xForwardedForHeader: false }` on the rate limiter — Codespaces proxies requests and sets `X-Forwarded-For`, which crashes requests without these settings

### 14b. `scripts/start-postgres.sh` — Always include

Always removes and recreates the container on each run. This is safe because all data lives in `.pgdata/` (bind mount) — the container itself holds nothing. Recreating ensures volume mount paths are always correct regardless of whether running in Codespaces, local Docker, or switching between them.

```bash
#!/bin/bash
set -e

CONTAINER_NAME="postgres"
DB_NAME="{project_name}_db"
DB_USER="api_user"
DB_PASS="api_pass"
DATA_DIR="$(pwd)/.pgdata"
SCHEMA_FILE="$(pwd)/src/db/schema.sql"

# Check Docker is available and running
if ! docker info > /dev/null 2>&1; then
  echo ""
  echo "ERROR: Docker is not running or not installed."
  echo ""
  echo "  - Mac/Windows: Start Docker Desktop and wait for it to fully launch"
  echo "  - Linux: run 'sudo systemctl start docker'"
  echo ""
  exit 1
fi

# Always remove and recreate the container — data is safe in .pgdata/ (bind mount).
# This avoids stale path mounts when switching between Codespaces and local dev,
# or when the container was created from a different working directory.
if docker ps -a --format '{{.Names}}' | grep -q "^${CONTAINER_NAME}$"; then
  echo "Removing existing postgres container (data preserved in .pgdata/)..."
  docker rm -f "$CONTAINER_NAME" > /dev/null
fi

# Create container fresh — volume mounts always resolve to current working directory
echo "Creating postgres container..."
docker run -d \
  --name "$CONTAINER_NAME" \
  --restart unless-stopped \
  -e POSTGRES_DB="$DB_NAME" \
  -e POSTGRES_USER="$DB_USER" \
  -e POSTGRES_PASSWORD="$DB_PASS" \
  -p 5432:5432 \
  -v "$DATA_DIR":/var/lib/postgresql/data \
  -v "$SCHEMA_FILE":/docker-entrypoint-initdb.d/01-schema.sql \
  postgres:16-alpine

# Wait for postgres to be ready
echo "Waiting for postgres to be ready..."
until docker exec "$CONTAINER_NAME" pg_isready -U "$DB_USER" -d "$DB_NAME" 2>/dev/null; do
  sleep 1
done
echo "Postgres is ready."
```

### 15. `README.md`

Must include all of the following sections:

```markdown
# {Project Name} API

> One-line description of what this API does.

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/lbrenman/{project-name})

## Overview
Brief description.

## Prerequisites
- Node.js 20+
- Docker + Docker Compose (for local postgres)
- (Optional) External PostgreSQL — Neon, Supabase, Railway, etc.

## Quick Start

### Option A — Codespaces (recommended)
1. Click the Open in GitHub Codespaces badge
2. Wait for setup to complete — it automatically: installs npm deps, copies `.env.example` to `.env` (defaults to `DATA_MODE=postgres`), runs `scripts/start-postgres.sh`, seeds the database
3. Open a terminal and run `npm run seed` manually — the automatic seed during setup runs before `.env` is fully active, so a manual seed ensures postgres is populated
4. Run `npm run dev`
5. Browse API docs at `http://localhost:3000/api-docs`
Note: on every Codespace restart, postgres and seed run automatically — no manual steps needed after first setup. If postgres is unresponsive, run `bash scripts/start-postgres.sh` manually.

### Option B — Local with Docker
bash scripts/start-postgres.sh, npm install, npm run seed, npm run dev

### Option C — Local with external DB
cp .env.example .env, set DATABASE_URL, npm install, npm run seed, npm run dev

## Environment Variables
Table of all env vars, their purpose, and defaults.

## Authentication
Explain the two AUTH_MODE options:
- `apikey` (default) — include `x-api-key: <key>` header on all requests; set API_KEY in .env
- `none` — open, no auth required (development only)
Note that the health endpoint is always unauthenticated regardless of AUTH_MODE.

## API Endpoints

### Health
GET /health — no auth required

### {Resource}
Table of all endpoints with method, path, auth required, description.

## Pagination
Explain ?page and ?limit query params and the response envelope format.

## Mock vs Database Mode
Explain DATA_MODE and how to switch.
Note that postgres is always available via docker-compose regardless of DATA_MODE.

## Running in Codespaces
Explain that .devcontainer/devcontainer.json starts only the postgres container (not the API).
After the Codespace starts, open a terminal and run `npm run dev` to start the API.
Note DATABASE_URL: since the API runs natively (not inside docker-compose), always use `localhost` as the hostname:
  DATABASE_URL=postgresql://api_user:api_pass@localhost:5432/{project_name}_db
Explain persistence: postgres data is stored in `.pgdata/` (bind mount to Codespace persistent disk). Records survive restarts. `.pgdata/` is gitignored.
Note that `.env` is auto-created from `.env.example` on first Codespace create — edit it to set API_KEY and other values.

## Docker
docker-compose up instructions. Note seed command.

## Development
npm run dev, npm run seed, npm run seed:clear, etc.
```

Replace `{GITHUB_USERNAME}` with `lbrenman` (the user's GitHub username).

---

## Response Envelope Format

All list responses must follow this format:
```json
{
  "data": [...],
  "pagination": {
    "total": 42,
    "page": 1,
    "limit": 10,
    "totalPages": 5,
    "hasNext": true,
    "hasPrev": false
  }
}
```

Single-item and mutation responses:
```json
{ "data": { ... } }
```

Error responses:
```json
{ "error": "message here" }
```

---

## Generation Steps

1. **Infer or confirm** the API domain and settings from the conversation
2. **Create project directory** at `/home/claude/{project-name}/` including `.devcontainer/`
3. **Generate all files** per the instructions above — be thorough and write real, working code (not stubs)
4. **ZIP the project**: `cd /home/claude && zip -r {project-name}.zip {project-name}/`
5. **Copy ZIP to outputs**: `cp /home/claude/{project-name}.zip /mnt/user-data/outputs/`
6. **Call `present_files`** with the output ZIP path
7. **Summarize** what was generated: auth mode, data mode, endpoints, Codespaces quick-start instructions

---

## Quality Rules

- All files must be **complete and runnable** — no `// TODO` stubs
- All env vars must appear in `.env.example` with comments
- Auth middleware must handle missing/invalid credentials gracefully with correct HTTP status codes (401 for auth failures, 400 for validation failures, 404 for not found, 500 for server errors)
- Mock data must be realistic for the domain (not just `item1`, `item2`)
- README Codespaces badge must use `lbrenman` as the GitHub username
- Health endpoint must always be unauthenticated
- The seed script must use `ON CONFLICT DO NOTHING` so it is safe to run multiple times
- `docker-compose.yml` must contain **only the postgres service** — no `api` service, no `env_file` reference — so Codespaces can start without a `.env` file present
- `docker-compose.full.yml` must contain the full stack (api + postgres) for local Docker-only use
- `docker-compose.yml` must use a **bind mount** (`./.pgdata:/var/lib/postgresql/data`) not a named volume — named volumes are ephemeral in Codespaces
- `.devcontainer/devcontainer.json` must use `image: mcr.microsoft.com/devcontainers/javascript-node:20` — NOT `dockerComposeFile`
- `postCreateCommand` and `postStartCommand` must both call `scripts/start-postgres.sh`
- `scripts/start-postgres.sh` must always remove and recreate the container (never `docker start`) — recreating ensures volume mounts resolve correctly regardless of environment (Codespaces vs. local Mac/Linux)
- `openapi.yaml` must be a complete, fully-filled OpenAPI 3.0 spec with all paths, schemas, and examples — no placeholders
- `/api-docs` route must be unauthenticated and mount Swagger UI
- `swagger-ui-express` and `yamljs` must be in `package.json` dependencies
- `.pgdata/` must be in `.gitignore`
