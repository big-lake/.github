# Local Development Guide

How to run the BigLake platform locally. You don't need a GCP account to work on the API or UI — a minimal local stack runs entirely on your machine.

---

## Minimal local stack (API + UI — no GCP required)

This is the fastest path. You'll have the full web interface running against a local API. Catalog, query, and AI features will be degraded (they call external services), but auth, navigation, and UI layout work fully.

### Prerequisites

- Python 3.8+
- Node.js 18+
- Git

### 1. Clone the repos

```bash
git clone https://github.com/big-lake/api
git clone https://github.com/big-lake/ui
```

### 2. Start the API

```powershell
cd api

# Create and activate a virtual environment
python -m venv .venv
.\.venv\Scripts\Activate.ps1    # Windows
# source .venv/bin/activate     # macOS/Linux

# Install dependencies
pip install -r requirements.txt

# Copy the example env file
cp .env.example .env
```

Edit `.env` and set:

```env
ENV=                            # leave empty — disables GCP Secret Manager
FLASK_SECRET_KEY=any-local-secret
JWT_SECRET=any-local-secret
```

The other values (`OPENMETADATA_*`, `GCS_BUCKET`, etc.) can stay as-is from `.env.example` — the API starts and returns graceful errors for those features when they're unavailable.

```powershell
python run.py
# API is now running at http://localhost:5000
```

### 3. Start the UI

```powershell
cd ui

npm install

# Point the UI at your local API
echo "VITE_API_BASE_URL=http://localhost:5000" > .env.local

npm run dev
# UI is now running at http://localhost:5173
```

Open `http://localhost:5173` — you should see the login page.

> **Default credentials:** The API seeds an admin user on first boot. Username: `admin`, password: whatever you set as `ADMIN_PASSWORD` in `.env` (or check `api/app/db/seed.py` for the dev default).

---

## Per-service local setup

### api

See [api/setup.md](https://github.com/big-lake/api/blob/main/setup.md) for GCP deployment.

Key env vars for local dev:

| Variable | Purpose | Local value |
|---|---|---|
| `ENV` | Enables GCP Secret Manager when set to `test` or `prod` | Leave empty |
| `JWT_SECRET` | Signing key for auth tokens | Any string |
| `FLASK_SECRET_KEY` | Flask session key | Any string |
| `OPENMETADATA_API_URL` | Catalog service | Point to a running OM instance, or leave as-is |
| `INTELLIGENCE_BASE_URL` | RAG service | Point to a running intelligence instance, or leave as-is |
| `GCS_BUCKET` | Data lake bucket for queries | Needs real GCP access for `/query` |

### ui

See [ui/setup.md](https://github.com/big-lake/ui/blob/main/setup.md) for GCP deployment.

The only required env var locally:

```env
VITE_API_BASE_URL=http://localhost:5000
```

---

## Testing during API + UI development

The platform deliberately has **no unit tests**. The testing strategy relies on:

1. **OpenAPI contract check** — the `validate-and-release` workflow diffs the live spec against `openapi.json`. Catches accidental request/response shape drift.
2. **Newman integration suite** — `api/postman/collections/ci.postman_collection.json` runs against the live API. Hits real routes, asserts on shape and status.
3. **Browser-driven UI checks** — Vite dev server with real Flask API. Visual + console verification.

### Local test loop (API or API + UI)

**Terminal 1 — start the API:**
```powershell
cd api
.\.venv\Scripts\Activate.ps1
python run.py            # http://localhost:5000
```

**Terminal 2 — run the Newman collection against local:**
```powershell
cd api
python -c "import sys, json, yaml; json.dump(yaml.safe_load(open(sys.argv[1], encoding='utf-8')), open(sys.argv[2], 'w'))" `
  "postman/environments/Big Lake — Local.environment.yaml" "newman-env.json"
npx --yes newman run "postman/collections/ci.postman_collection.json" --environment "newman-env.json"
# Cleanup:
Remove-Item newman-env.json
```

**Terminal 3 (only for UI work) — start the UI:**
```powershell
cd ui
npm run dev              # http://localhost:5173
```

### After changing the OpenAPI spec

```powershell
cd api
python scripts/update_openapi_baseline.py
# Commit the updated openapi.json alongside your code change
```

### After adding/changing an endpoint

Always update both:
- The Newman CI collection (`postman/collections/ci.postman_collection.json`) — see [`postman-ci.instructions.md`](https://github.com/big-lake/api/blob/main/.github/instructions/postman-ci.instructions.md)
- The OpenAPI baseline (`openapi.json`)

CI will fail loudly if either drifts from the code.

### Why no pytest?

The API is a thin BFF: route → service → external integration (OpenMetadata, DuckDB/GCS, intelligence). Stateless service functions don't benefit much from unit tests, and the route layer is best verified against a real Flask process anyway. Newman gives us that for free, and it's the same suite CI runs against the deployed test VM.

If a service function ever grows complex logic (parsing, scoring, ranking, etc.), drop a focused pytest for **that function** — but don't build out a test infrastructure pre-emptively.

---

### etl

ETL flows run on Prefect and require GCS access for all meaningful work. There is no local-only mode.

See [etl/setup.md](https://github.com/big-lake/etl/blob/main/setup.md) for full setup, including running a local Prefect server against GCS test data.

### intelligence

The intelligence service requires a running pgvector database and GCP credentials (for Vertex AI).

See [intelligence/setup.md](https://github.com/big-lake/intelligence/blob/main/setup.md) for full setup. A local pgvector instance can be run with Docker:

```bash
docker run -d \
  --name biglake-pgvector \
  -e POSTGRES_PASSWORD=dev \
  -p 5432:5432 \
  pgvector/pgvector:pg16
```

### knowledge

Knowledge pipelines require GCS access and (for embedding generation) GCP Vertex AI credentials. See [knowledge/setup.md](https://github.com/big-lake/knowledge/blob/main/setup.md).

### catalog

OpenMetadata is deployed via Docker Compose. See [catalog/setup.md](https://github.com/big-lake/catalog/blob/main/setup.md) for running it locally.

---

## Full GCP environment

To deploy the complete platform to a new GCP project, work through the `setup.md` files in this order:

1. `infra/setup.md` — provision all GCP resources (do this first)
2. `etl/setup.md`
3. `catalog/setup.md`
4. `knowledge/setup.md`
5. `intelligence/setup.md`
6. `api/setup.md`
7. `ui/setup.md`

Each `setup.md` documents its own GCP prerequisites and GitHub Actions secrets.

---

## Keeping this guide up to date

If you change how any service runs locally (new required env var, new prerequisite, changed port), update the relevant section above as part of the same PR.
