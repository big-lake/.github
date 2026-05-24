# Local Development Guide

How to run the BigLake platform locally. You don't need a GCP account to work on the API or UI — a minimal local stack runs entirely on your machine.

---

## One-click setup (recommended)

Open `biglake.code-workspace` in VS Code from the parent `biglake/` folder. It loads all repos as a multi-root workspace and ships with tasks for:

- **`Dev: API + UI`** — starts both servers in parallel in dedicated panels
- **`api: newman (local)`** — runs the CI integration suite against your local API
- **`api: update OpenAPI baseline`** — regenerates `openapi.json` after endpoint changes

Run via `Tasks: Run Task` in the command palette. Most day-to-day work needs only `Dev: API + UI` and `api: newman (local)`.

---

## Cross-repo workflow — the contract-first loop

When a feature touches both `api` and `ui`, follow this 7-step sequence. Each step has a verifiable artifact, which is what makes it work cleanly for both humans and coding assistants.

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│  api/app/schemas/__init__.py  ──►  api/app/routes/*  ──►  openapi.json     │
│         (1) design contract       (2) implement       (3) baseline         │
│                                                            │                │
│                                                            ▼                │
│              ui/src/views/*  ◄──  ui/src/api/client.js  ◄──  Newman ✓       │
│                 (6) wire view      (5) add client fn    (4) integration    │
│                                                                             │
│                  (7) update api/BFF_API_REQUIREMENTS.md                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

1. **Design the contract first.** Edit/add the Marshmallow schema in [api/app/schemas/__init__.py](https://github.com/big-lake/api/blob/main/app/schemas/__init__.py) and the route's YAML docstring. Do not write UI code yet.
2. **Implement the route + service.** Routes validate input and call services; services hold logic. See [api/.github/copilot-instructions.md](https://github.com/big-lake/api/blob/main/.github/copilot-instructions.md).
3. **Refresh the OpenAPI baseline.** Run the `api: update OpenAPI baseline` task (or `python scripts/update_openapi_baseline.py`). Commit `openapi.json` alongside the route change.
4. **Add a Newman request** to [api/postman/collections/ci.postman_collection.json](https://github.com/big-lake/api/blob/main/postman/collections/ci.postman_collection.json) per [postman-ci.instructions.md](https://github.com/big-lake/api/blob/main/.github/instructions/postman-ci.instructions.md). Run the `api: newman (local)` task — green = contract works end-to-end.
5. **Add the `client.js` function** in `ui`. Point your assistant at `../api/openapi.json` and `../api/app/schemas/__init__.py` for exact shapes — never guess field names.
6. **Wire the view.** Keep components inline-first (no extraction unless a section exceeds the boundary rules in [ui/.github/copilot-instructions.md](https://github.com/big-lake/ui/blob/main/.github/copilot-instructions.md)).
7. **Update [api/BFF_API_REQUIREMENTS.md](https://github.com/big-lake/api/blob/main/BFF_API_REQUIREMENTS.md)** — status (Mocked → Wired), field usage notes, gaps closed/opened.

### Why this order works for AI-assisted dev

- The OpenAPI spec and Newman collection are the **handshake between sessions** — when both are green, both repos are aligned, even if you're running a separate assistant for each.
- Steps 1–4 are entirely in `api`. Steps 5–6 are entirely in `ui`. Step 3's `openapi.json` is the single artifact that crosses the boundary.
- An assistant pointed at the spec can produce correct `client.js` calls without seeing the API source.

### Reusable prompts

The org `.github` repo ships with prompt files in [.github/prompts/](https://github.com/big-lake/.github/tree/main/.github/prompts) that walk through these flows step-by-step:

- `new-endpoint.prompt.md` — adding a new endpoint end-to-end (steps 1–7)
- `change-endpoint.prompt.md` — modifying an existing endpoint without breaking the UI
- `new-ui-view.prompt.md` — adding a UI surface that consumes existing endpoints

Invoke with `/new-endpoint`, `/change-endpoint`, or `/new-ui-view` in Copilot Chat from the multi-root workspace.

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

# Copy the example env file — the defaults are dev-safe, no edits needed
cp .env.example .env
```

The `.env.example` ships with sensible local defaults: `ENV=` (unset, disables Secret Manager), dev-only secrets, and the OpenMetadata / GCS values can stay as-is (the API returns graceful errors for those features when unavailable).

```powershell
python run.py
# API is now running at http://localhost:5000
```

### 3. Start the UI

```powershell
cd ui

npm install

# Point the UI at your local API (dev-safe defaults are in .env.example)
cp .env.example .env.local

npm run dev
# UI is now running at http://localhost:5173
```

Open `http://localhost:5173`.

> **Auth in local dev:** The platform is cookie-based passwordless (see [api ADR-0014](https://github.com/big-lake/api/blob/main/documentation/adr/0014-two-tier-session-cookie.md) / [ADR-0015](https://github.com/big-lake/api/blob/main/documentation/adr/0015-passwordless-auth-methods.md)). Locally:
>
> - The router currently skips its guard when `import.meta.env.DEV` is true, so navigation works without a session.
> - Endpoints that require a real session need a cookie. To mint one, hit `POST /auth/dev/session` with `X-Dev-Auth: dev-session-secret-change-me` and body `{"email": "you@biglake.local"}` ([api ADR-0017](https://github.com/big-lake/api/blob/main/documentation/adr/0017-dev-session-endpoint.md)). The Newman suite does this automatically.
> - The UI dev-mode auto-bootstrap that mints a session on app load is staged as [ui/DESIGN.md Phase 2 Step 3](https://github.com/big-lake/ui/blob/main/DESIGN.md) — landing alongside the broader auth rewire.

---

## Per-service local setup

### api

See [api/setup.md](https://github.com/big-lake/api/blob/main/setup.md) for GCP deployment.

Key env vars for local dev:

| Variable | Purpose | Local value |
|---|---|---|
| `ENV` | Enables GCP Secret Manager when set to `test` or `prod` | Leave empty |
| `FLASK_SECRET_KEY` | Flask session key | Any string (`.env.example` default is fine) |
| `OPENMETADATA_API_URL` | Catalog service | Point to a running OM instance, or leave as-is |
| `INTELLIGENCE_URL` | RAG service | Point to a running intelligence instance (or IAP tunnel — see "Local end-to-end coverage" below) |
| `GCS_BUCKET` | Data lake bucket for queries | Needs real GCP access for `/query` |
| `PUBLIC_BASE_URL` | Base URL used in magic-link emails | `http://localhost:5000` |

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

## Local end-to-end coverage — pointing at deployed dependencies

The minimal local stack above gives you working auth and UI navigation but degraded catalog / query / chat. To exercise the **full** end-to-end flow against real data (real OpenMetadata, real GCS parquet, real RAG with Gemini + Cohere) without standing up the whole platform locally, the local API can point at the deployed `test` environment dependencies.

### Reachability matrix

| Dependency | Reachable from laptop? | How the local API connects |
|---|---|---|
| GCS (parquet datalake) | Direct (public API) | HMAC keys auto-resolved from Secret Manager via ADC |
| Vertex AI / Gemini | Direct (public API) | ADC (not actually used by api locally — intelligence uses it) |
| Secret Manager | Direct (public API) | ADC, only consulted for unset env vars |
| OpenMetadata VM | Direct (VM has external IP, 0.0.0.0/0 on port 8585) | `OPENMETADATA_API_URL` + `OPENMETADATA_API_TOKEN` in `api/.env` |
| Intelligence VM | **Internal only** (no external IP) | IAP tunnel to `localhost:8001` |
| pgvector / Neo4j (knowledge_db) | **Internal only** | Not connected directly — the deployed intelligence service handles this |

### One-time prereqs

1. `gcloud auth login` and `gcloud auth application-default login`
2. Grant your user account on the api SA:
   ```powershell
   gcloud iam service-accounts add-iam-policy-binding `
     biglake-sa-api-test@big-lake-test-490405.iam.gserviceaccount.com `
     --member="user:you@example.com" `
     --role="roles/iam.serviceAccountTokenCreator" `
     --project=big-lake-test-490405
   ```
3. Grant your user `roles/iap.tunnelResourceAccessor` on the intelligence VM (one-time, per developer). See [intelligence/setup.md](https://github.com/big-lake/intelligence/blob/main/setup.md).

### Populate `api/.env` for chat (intelligence via IAP tunnel)

Uncomment / set in `api/.env`:
```env
INTELLIGENCE_URL=http://localhost:8001
INTELLIGENCE_TOKEN_AUDIENCE=http://biglake-intelligence-test.australia-southeast2-b.c.big-lake-test-490405.internal:8001
```

`INTELLIGENCE_URL` is the **connect** URL (the local end of the IAP tunnel). `INTELLIGENCE_TOKEN_AUDIENCE` is the **token** audience the deployed intelligence verifies the JWT against — keep them split. The api's `id_token_client` will impersonate the api SA via ADC (no key file) to mint the token.

### Populate `api/.env` for catalog (direct to OpenMetadata VM)

Fetch the OM external IP and JWT token once:
```powershell
$omIp = gcloud compute instances describe biglake-openmetadata-test `
  --zone=australia-southeast2-b --project=big-lake-test-490405 `
  --format='value(networkInterfaces[0].accessConfigs[0].natIP)'
$omToken = gcloud secrets versions access latest `
  --secret=api-openmetadata-token-test --project=big-lake-test-490405
"OPENMETADATA_API_URL=http://$omIp:8585/api/v1"
"OPENMETADATA_API_TOKEN=$omToken"
```
Paste both lines into `api/.env`.

### Query / GCS

No `api/.env` changes needed. With ADC active, `secrets.py` resolves the GCS HMAC keys from Secret Manager automatically the first time DuckDB initialises the GCS extension. The `GCS_BUCKET` and `GOOGLE_CLOUD_PROJECT` defaults already point at `test`.

### Run it

Open the command palette → `Tasks: Run Task` → **`Dev: Full Stack`**. This launches:

- `api: dev (Flask :5000)`
- `ui: dev (Vite :5173)`
- `iap: tunnel intelligence (:8001)`

Then visit `http://localhost:5173`. Catalog, query, and chat should all work end-to-end against `big-lake-test-490405`.

### Caveats

- Chat round-trips hit a real VM via IAP — expect a few seconds of network + reranker latency. This is faithful to prod, not a defect.
- The IAP tunnel terminates if `gcloud` re-auths or your laptop sleeps. Re-run the task to reconnect.
- The OpenMetadata external IP is ephemeral by default — if the VM gets recreated, refetch and update `api/.env`.
- This setup is for **dev only**. Do not commit the OM token or any other fetched secret to git (`.env` is gitignored; `.env.example` is the safe template).

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
