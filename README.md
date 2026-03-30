# API Project Generator — Claude Skill

A [Claude Skill](https://www.anthropic.com/claude) that generates complete, production-ready **Node.js/Express REST API projects** as a downloadable ZIP. Just describe the API you want and Claude scaffolds the entire project — ready to push to GitHub and launch in Codespaces.

---

## What It Does

When installed, this skill triggers automatically whenever you ask Claude to build an API. For example:

> *"Generate a contacts API"*
> *"Build me a Node.js API for managing invoices"*
> *"Create a REST API for a product catalog"*

Claude generates a full project and hands you a ZIP file containing every file you need — no boilerplate to write, no wiring to do.

---

## Generated Project Features

### 🔐 Authentication
- **API Key** (`x-api-key` header) — default, production-ready
- **None** — open access for development/internal use
- Runtime-switchable via `AUTH_MODE` environment variable

### 🗄️ Data Backend
- **PostgreSQL** (default) via `pg` node-postgres client
- **Mock** — in-memory data loaded from a JSON file, no database required
- Runtime-switchable via `DATA_MODE` environment variable

### 📄 Pagination
- Offset/limit pagination on all list endpoints (`?page=1&limit=10`)
- Consistent response envelope with `data` array and `pagination` metadata (`total`, `page`, `limit`, `totalPages`, `hasNext`, `hasPrev`)

### 🩺 Health Check
- `GET /health` — always included, always unauthenticated
- Returns service name, version, and timestamp

### 📖 Interactive API Docs
- `GET /api-docs` — Swagger UI served automatically, always unauthenticated
- Backed by a complete **OpenAPI 3.0 spec** (`openapi.yaml`) in the project root
- Full schemas, request bodies, response envelopes, and realistic examples

### 🛡️ Production Middleware
- **Rate limiting** (`express-rate-limit`) — configurable window and max requests via env vars
- **Request logging** (`morgan`) — combined or dev format based on `LOG_LEVEL`
- **Input validation** (`express-validator`) — domain-appropriate field validation on POST/PUT
- **Security headers** (`helmet`) — sane defaults with CSP disabled for Swagger UI compatibility
- **CORS** — enabled globally
- **Reverse proxy support** — `trust proxy` configured for Codespaces and other proxied environments

### 🐳 Docker
- **`docker-compose.yml`** — postgres-only, safe for Codespaces (no `.env` dependency at startup)
- **`docker-compose.full.yml`** — full stack (API + postgres) for local Docker deployments
- **`Dockerfile`** — node:20-alpine, production-optimized
- **`scripts/start-postgres.sh`** — always recreates the postgres container to ensure correct volume mounts regardless of environment (Codespaces vs. local Mac/Linux)

### 🚀 GitHub Codespaces Ready
- **`.devcontainer/devcontainer.json`** — fully configured with Node.js 20 + Docker-in-Docker
- PostgreSQL starts automatically on Codespace launch
- Ports 3000 (API) and 5432 (PostgreSQL) forwarded automatically
- SQLTools + SQLTools-pg extensions pre-installed for in-editor database access
- **Persistent PostgreSQL data** — stored in `.pgdata/` (workspace bind mount), survives Codespace stop/start cycles
- README includes a **Launch in Codespaces** badge linked to your GitHub account

### 📦 Complete Project Files
- `src/index.js` — entry point
- `src/app.js` — Express app setup
- `src/routes/` — health, api-docs, and resource routes
- `src/controllers/` — dual-mode (mock + postgres) CRUD controller
- `src/middleware/` — auth and pagination middleware
- `src/db/client.js` — pg Pool setup
- `src/db/schema.sql` — `CREATE TABLE IF NOT EXISTS` with indexes, auto-mounted into postgres on first run
- `src/data/{resource}.json` — 7–10 realistic domain-appropriate seed records
- `src/data/seed.js` — standalone seed script with `--clear` flag, safe to run repeatedly (`ON CONFLICT DO NOTHING`)
- `.env.example` — all environment variables documented with inline comments
- `.gitignore` — includes `node_modules/`, `.env`, `.pgdata/`
- `package.json` — with `start`, `dev`, `seed`, and `seed:clear` scripts

---

## Environment Variables

Every generated project is fully environmentalized. Key variables:

| Variable | Description | Default |
|---|---|---|
| `PORT` | Server port | `3000` |
| `AUTH_MODE` | `apikey` or `none` | `apikey` |
| `API_KEY` | API key value | — |
| `DATA_MODE` | `postgres` or `mock` | `postgres` |
| `DATABASE_URL` | PostgreSQL connection string | local docker |
| `DEFAULT_PAGE_SIZE` | Default page size | `10` |
| `MAX_PAGE_SIZE` | Max page size cap | `100` |
| `RATE_LIMIT_WINDOW_MS` | Rate limit window (ms) | `60000` |
| `RATE_LIMIT_MAX` | Max requests per window | `100` |
| `LOG_LEVEL` | `info` or `debug` | `info` |

---

## Quick Start (Codespaces)

1. Generate an API project by asking Claude: *"Generate a [domain] API"*
2. Download the ZIP and push it to a GitHub repo
3. Click the **Open in GitHub Codespaces** badge in the README
4. Wait for setup to complete (postgres starts automatically)
5. Open a terminal and run:
   ```bash
   npm run seed   # populate the database
   npm run dev    # start the API
   ```
6. Browse the live API docs at `http://localhost:3000/api-docs`

---

## Quick Start (Local)

```bash
cp .env.example .env          # review and configure
bash scripts/start-postgres.sh  # requires Docker Desktop running
npm install
npm run seed
npm run dev
```

---

## API Endpoint Pattern

Every generated API follows the same consistent structure:

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/health` | None | Health check |
| `GET` | `/api-docs` | None | Swagger UI |
| `GET` | `/{resource}` | API Key | List (paginated) |
| `GET` | `/{resource}/:id` | API Key | Get by ID |
| `POST` | `/{resource}` | API Key | Create |
| `PUT` | `/{resource}/:id` | API Key | Update |
| `DELETE` | `/{resource}/:id` | API Key | Delete |

---

## Installing the Skill

1. Download `api-project-generator.skill` from the [Releases](https://github.com/lbrenman/YOUR-NEW-REPO-NAME/releases) page
2. In Claude.ai, go to **Settings → Skills**
3. Click **Install Skill** and upload the `.skill` file
4. Start a conversation and ask Claude to generate an API

---

## Tech Stack

All generated projects use:

- **Runtime**: Node.js 20
- **Framework**: Express 4
- **Database**: PostgreSQL 16 (via `pg`)
- **Auth**: API key header (`x-api-key`)
- **Docs**: Swagger UI (`swagger-ui-express` + `yamljs`)
- **Validation**: `express-validator`
- **Rate limiting**: `express-rate-limit`
- **Logging**: `morgan`
- **Security**: `helmet`, `cors`
- **Dev**: `nodemon`, `dotenv`

---

## License

MIT
