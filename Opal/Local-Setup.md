# Local Development Setup

> Complete guide to running every piece of Kyra infrastructure on your machine.
> This covers Redis, MongoDB, Supabase (optional), Python environment, and the
> two processes that need to run (FastAPI server + ARQ worker).

---

## Prerequisites

Install these once on your machine:

| Tool | Min version | Install |
|------|-------------|---------|
| Docker Desktop | 4.x | https://docs.docker.com/desktop |
| Python | 3.11+ | https://python.org or `brew install python@3.11` |
| Node.js | 18+ (for Supabase CLI) | https://nodejs.org or `brew install node` |
| Git | any | pre-installed on most systems |

---

## 1. Clone and enter the project

```bash
git clone <your-repo-url> opal-server
cd opal-server
```

---

## 2. One-time setup

```bash
make setup
```

This does three things:
- Creates `.venv/` with all Python dependencies installed
- Copies `.env.local` → `.env` (all local defaults pre-filled)

After it runs, open `.env` and fill in the three optional API keys:

```bash
OPENAI_API_KEY=sk-...              # paste your key here for live LLM
MOTIF_RUNWARE_API_KEY=...          # optional: AI image generation
MOTIF_UNSPLASH_ACCESS_KEY=...      # optional: venue photo search (free key)
```

You can skip all three and the server still runs fully — it just uses
deterministic/replay mode instead of calling real APIs.

---

## 3. Start Docker services

```bash
make services
```

This starts:

| Service | Port | What it does |
|---------|------|-------------|
| Redis | `6379` | Cache, session state, SSE replay, idempotency, job queue |
| MongoDB | `27017` | Conversation turns, session snapshots, moodboard versions, motif assets |

To also start browser GUIs for debugging:

```bash
make services-tools
# Redis Commander → http://localhost:8081  (browse all kyra:* keys)
# Mongo Express   → http://localhost:8082  (browse opal_db collections)
```

Verify everything is healthy:

```bash
docker compose ps
```

All services should show `healthy`.

---

## 4. (Optional) Set up Supabase locally

Supabase is only needed for the **V1 planning pipeline** (plans + sections tables).
If you're only working on the V2 assist flow, skip this step.

```bash
# Install Supabase CLI
npm install -g supabase

# Initialise local Supabase (first time only — creates supabase/ directory)
npx supabase init

# Start local Supabase stack (Postgres + Auth + Storage + Studio)
npx supabase start
```

The local Supabase keys are pre-filled in `.env.local` — they're the same
fixed demo keys for every local Supabase instance, safe to commit.

Run the V1 schema migrations:

```bash
npx supabase db push
```

If you need the `motif_assets` table in Supabase (not needed if using MongoDB):

```bash
# Paste the contents of infrastructure/db/motif_assets_migration.sql
# into the Supabase Studio SQL editor: http://localhost:54323
```

---

## 5. Start the FastAPI server

Open a terminal:

```bash
make dev
```

The server starts at **http://localhost:8000** with hot-reload.

- API docs: http://localhost:8000/docs
- Prometheus metrics: http://localhost:8000/metrics
- Health check: http://localhost:8000/health
- Keep-alive ping: http://localhost:8000/health/ping

---

## 6. Start the ARQ background worker

Open a **second terminal**:

```bash
make worker
```

The worker processes background jobs from the `kyra:queue` Redis queue. It's
needed for any endpoint that dispatches async work (image generation, etc.).

---

## 7. Verify everything is working

```bash
# Health check
curl http://localhost:8000/health

# Expected response
# {"status": "ok", ...}

# Run the test suite
make test
# Expected: 1371 passed
```

---

## Daily workflow

```bash
# Morning: start services (if not already running)
make services

# Start app (terminal 1)
make dev

# Start worker (terminal 2)
make worker

# Run tests
make test

# Check Redis keys
make redis-cli
# > KEYS kyra:*

# Check MongoDB data
make mongo-cli
# > use opal_db
# > db.conversation_turns.find().limit(5)
```

---

## LLM modes

Control how much AI you want to run locally via `.env`:

| Mode | `.env` settings | What happens |
|------|----------------|-------------|
| **Fully offline** (default) | `LLM_ENABLED=false` `LLM_V2_MODE=deterministic` | No API calls, instant deterministic responses, costs nothing |
| **Cheap online** | `LLM_ENABLED=true` `LLM_V2_MODE=hybrid` `OPENAI_API_KEY=sk-...` | Analyser model only (gpt-4o-mini, ~$0.0001/turn) |
| **Full online** | `LLM_ENABLED=true` `LLM_V2_MODE=full` `OPENAI_API_KEY=sk-...` | Both analyser + generator models, realistic responses |

---

## Motif (image generation)

Both providers are disabled by default in `.env.local` to avoid accidental API charges.

```bash
# Enable Unsplash (free, 50 req/hr demo key)
MOTIF_UNSPLASH_ENABLED=true
MOTIF_UNSPLASH_ACCESS_KEY=your_key   # https://unsplash.com/developers

# Enable Runware AI generation (~$0.001 per image)
MOTIF_IMAGE_GEN_ENABLED=true
MOTIF_RUNWARE_API_KEY=your_key       # https://runware.ai
```

---

## Resetting local data

```bash
# Stop services and wipe ALL local data (Redis + MongoDB volumes)
make reset

# Then restart fresh
make services
```

---

## Environment variable cheat sheet

| Variable | Local value | Notes |
|----------|-------------|-------|
| `REDIS_URL` | `redis://localhost:6379/0` | Matches docker-compose |
| `MONGODB_URL` | `mongodb://kyra:kyradev@localhost:27017` | Matches docker-compose |
| `MONGODB_DATABASE` | `opal_db` | All 4 collections live here |
| `SUPABASE_URL` | `http://127.0.0.1:54321` | Local Supabase CLI |
| `LLM_ENABLED` | `false` | Set `true` + add key for live LLM |
| `LLM_V2_MODE` | `deterministic` | Fastest offline dev mode |
| `MOTIF_IMAGE_GEN_ENABLED` | `false` | Enable for image generation |
| `MOTIF_UNSPLASH_ENABLED` | `false` | Enable for photo search |

Full reference: see `.env.example` and `docs/infrastructure.md`.

---

## Troubleshooting

**Port already in use**

```bash
# Find what is using port 6379 (Redis)
lsof -i :6379

# Find what is using port 27017 (MongoDB)
lsof -i :27017
```

**MongoDB auth fails**

The local MongoDB container uses `kyra:kyradev` credentials. If you get an
auth error, make sure `MONGODB_URL` in `.env` is exactly:

```
MONGODB_URL=mongodb://kyra:kyradev@localhost:27017
```

Do NOT also set `MONGODB_USER`/`MONGODB_PASSWORD` — the URL already embeds them.

**Redis connection refused**

```bash
# Check container is running
docker compose ps

# Restart Redis
docker compose restart redis
```

**Test failures**

```bash
# Two tests are known pre-existing failures (TestClient stream API issue)
# They are excluded by make test automatically.
# If you see other failures, check that services are running first:
docker compose ps
```

**`supabase start` fails**

Make sure Docker Desktop is running and has at least 4 GB RAM allocated
(Supabase local stack needs ~2 GB). Increase in Docker Desktop → Settings →
Resources.
