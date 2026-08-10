---
description: "Work through the catalog → query → intelligence wiring plan systematically. Shows current state, lets the user pick the next item, then implements it end-to-end: API contract → service → UI → docs → Newman tests."
mode: agent
---

# /wire-up — Systematic end-to-end wiring

This prompt works through the BigLake BFF wiring plan one item at a time. Each run:
1. Shows the current state and suggests what to do next
2. Asks the user to confirm the item
3. Researches the actual code, validates architecture, flags any DB changes or gaps
4. Presents a detailed plan for confirmation
5. Implements everything end-to-end
6. Updates all docs and Newman tests
7. Tests against the local API (pointed at test-environment services)

---

## Step 1 — Show current state and suggest next items

Read the following to build an accurate status picture:
- `/memories/repo/wire-up-status.md` — persisted checklist of completed/remaining items
- `api/BFF_API_REQUIREMENTS.md` — authoritative status labels (Wired / Mocked / Partial / Not built)
- `ui/src/views/Workspace.vue` — which sidebar/terminal calls are still using mock data
- `ui/src/api/client.js` — which functions are implemented vs defined-but-not-called

Present a **current state table**:

| Surface | API | UI | Notes |
|---|---|---|---|
| Catalog sidebar | ✅ / 🟡 / ❌ | ✅ / 🟡 / ❌ | e.g. "API wired; OM token may be stale" |
| Knowledge sources | … | … | … |
| Catalog search | … | … | … |
| Agent terminal | … | … | … |
| … | … | … | … |

Then list the **next 3 items** from the remaining plan as options:

1. **P1.x — [Item name]**: One-line summary of what this involves and why it unblocks the next item.
2. **P1.y — [Item name]**: …
3. **P2.x — [Item name]**: …

Ask: "Which item would you like to work on?" and wait for the user's selection before proceeding.

---

## Step 2 — Research and validate the selected item

Once the user selects an item, **do not start implementing yet**. Read the code first.

### 2a. Read the actual current code

Read every file relevant to the selected item — route, service, schema, UI component, migration
files, test collection. Do not rely on the plan summary alone; the code may have changed.

Flag any differences between what the plan says and what the code actually contains.

### 2b. Check for SQLite schema changes

If the item requires a new table, column, or index:
- Read `api/app/db/schema.py` and all files in `api/app/db/migrations/` to understand the current
  schema
- Migrations follow the numbered pattern `NNNN_<description>.sql` (see existing files for style)
- Propose the exact new migration SQL
- Draw an **entity-relationship diagram** using Mermaid showing new table(s) and their relationships
  to existing tables
- State clearly: new columns, affected existing columns, foreign keys, indexes

Ask the user to confirm the DB design before proceeding.

### 2c. Identify architecture and design gaps

Flag anything that is:
- Not yet specified in `api/BFF_API_REQUIREMENTS.md` (missing error cases, unspecified pagination,
  missing fields)
- Architecturally inconsistent with similar endpoints (auth pattern, error shape, response wrapper)
- A cross-repo dependency that changes the implementation sequence (see 2d)
- A security concern (injection, unvalidated inputs, exposed credentials)

### 2d. Security review

Before implementing, check the proposed change against the OWASP Top 10 and the project's own
risk register (`api/documentation/security/risks.md`). Flag any of the following:

- **Injection** — any user-supplied value that reaches SQL, a shell, or an external API call must
  be validated or parameterised. Check every new service function for f-string interpolation into
  queries or URLs.
- **Broken access control** — new endpoints that return or mutate user data must check session
  ownership (same `session_sub` pattern as existing routes). Endpoints that should be
  user-tier-only must apply `@require_auth` and reject `tier == "anon"`.
- **Sensitive data exposure** — no credentials, tokens, GCS paths, or internal service URLs in
  API responses. Check that error messages don't leak stack traces or internal hostnames.
- **SSRF** — any new outbound HTTP call (e.g. to OM, intelligence, GCS) must only call
  allow-listed internal URLs from config/env vars, never from user input.
- **XSS** — any field that the UI renders via `v-html` (e.g. OM search snippet `<em>` highlights)
  must be sanitised server-side before returning. Flag the field and confirm the sanitisation
  approach with the user.
- **New findings** — if a real finding is identified, add a `SEC-NNN` row to
  `api/documentation/security/risks.md` before proceeding. Link to it rather than describing the
  finding inline in `BFF_API_REQUIREMENTS.md`.

If no issues are found, state "No new security findings" explicitly so it's clear the check ran.

### 2e. Flag cross-repo deploy requirements

If the item touches the **intelligence** service:

> ⚠️ **Intelligence changes require a GHA deploy before they can be tested end-to-end.**
> The local API's `INTELLIGENCE_URL=http://localhost:8001` points at the **deployed** intelligence
> VM via the IAP tunnel — not a local process. After pushing intelligence changes, wait for
> `.github/workflows/deploy.yml` to complete before running integration tests.

If the item touches the **OM token** (catalog work):

> ⚠️ **Verify the OM token before implementing.** The catalog API returns `502 kind:upstream` when
> OM responds with 401, which is hard to distinguish from real failures. Run the pre-check in
> Step 5 before starting.

Present a concise **updated implementation plan** for this specific item and ask the user to
confirm before writing any code.

---

## Step 3 — Implement end-to-end

Follow the contract-first loop from `DEVELOPMENT.md`. Implement in this order.

### 3a. API contract first (if API changes are involved)

1. Add or update Marshmallow schemas in `api/app/schemas/__init__.py`
   - New top-level schemas → register in `api/app/openapi.py` with `_safe_register()`
   - **Never manually register nested schemas** (`fields.Nested(...)` inside another schema) —
     `MarshmallowPlugin` auto-registers them; doing it again raises `DuplicateComponentNameError`
2. Write or update the YAML docstring block on the view function — copy pattern from
   `api/app/routes/auth/session.py`. Cover `requestBody`, `responses`, `parameters`, `security`.
3. Implement the route in `api/app/routes/` — thin: validate input, call service, return response
4. Implement the service in `api/app/services/` — all logic here, not in routes
5. Register the blueprint in `api/app/__init__.py` if a new route file was created
6. If a SQLite migration is needed, add `api/app/db/migrations/NNNN_<description>.sql`

### 3b. Refresh the OpenAPI baseline

```powershell
# VS Code: Tasks: Run Task → "api: update OpenAPI baseline"
# Or:
cd api ; python scripts/update_openapi_baseline.py
```

Verify the new or changed path appears in `openapi.json`. Commit it in the same change as the
route. CI fails if they drift.

### 3c. UI wiring

1. Add or update the function in `ui/src/api/client.js`
   - Consult `api/openapi.json` for exact field names and types — do not guess
   - `credentials: 'include'` on every call; no `Authorization: Bearer` header
   - 401 responses must clear `authStore` and redirect to `/login` (see existing pattern)
2. Wire the view in `ui/src/views/` — replace the mock call / mock data with the real client call
3. Handle loading, error, and empty states inline (no shared utility)
4. Keep `.vue` files under ~300 lines; extract to `src/components/` only when a section has a
   clear standalone boundary with its own template, styles, and logic

### 3d. Update documentation

Update **all** of the following in the same commit:

- **`api/BFF_API_REQUIREMENTS.md`** — update Status label (→ Wired), update request/response shape,
  note which Vue component uses which field, close resolved gaps, open new ones discovered during
  implementation
- **ADR** (if a new architectural decision was made) — create
  `api/documentation/adr/NNNN-<description>.md` following the existing ADR format
- **`api/README.md`** — update the endpoint list if a new route domain was added
- **`intelligence/README.md`** (if intelligence was changed) — update
  architecture, endpoints, SSE event shapes per the documentation instructions

---

## Step 4 — Add Newman CI tests

Edit `api/postman/collections/ci.postman_collection.json`.

Rules (see `.github/instructions/postman-ci.instructions.md` for the full template):
- Insert after Bootstrap — the session cookie carries automatically, no per-request auth needed
- **Assert status code and response shape** — not hardcoded values or timestamps
- **SSE endpoints** (`POST /chat`, `POST /agent/send`): assert `status(200)` and
  `Content-Type: text/event-stream`; do not try to parse the stream body
- If the endpoint calls an external dependency (OM, intelligence) that may be down in CI, allow
  `[200, 502]` with a comment explaining why
- Use `pm.environment.set(...)` to stash IDs needed by later requests in the same run
- Negative-path tests (e.g. rejects missing field with 400) are valuable — add them as separate
  named requests

Current collection order:
```
Bootstrap dev session → Health → Who am I → Catalog → Query → Chat → [new domains here]
```

---

## Step 5 — Test

### Pre-flight: confirm test-environment services are reachable

The local API (`localhost:5000`) calls **test-environment services** via `api/.env`. Each
dependency has a different reachability path:

| Dependency | How local API connects | Pre-check |
|---|---|---|
| OpenMetadata | HTTPS: `https://om.test.biglake.au/api/v1` | See OM token check below |
| Intelligence | IAP tunnel: `localhost:8001` | IAP tunnel task must be running |
| GCS / DuckDB | Direct via ADC | `gcloud auth application-default login` |
| Secret Manager | Direct via ADC | Same ADC login |

**Check the OM token (for catalog work):**
```powershell
$token = gcloud secrets versions access latest --secret=api-openmetadata-token-test --project=big-lake-test-490405
curl.exe -s -o NUL -w "%{http_code}" -H "Authorization: Bearer $token" "https://om.test.biglake.au/api/v1/tables?limit=1"
# Expect 200. If 401, the token needs refreshing via the OM UI → update the Secret Manager value.
```

**Start the IAP tunnel (for intelligence / agent work):**
```powershell
# VS Code: Tasks: Run Task → "Dev: Remote Dependencies"
# Or manually:
gcloud compute start-iap-tunnel biglake-intelligence-test 8001 `
  --local-host-port=localhost:8001 `
  --zone=australia-southeast2-b `
  --project=big-lake-test-490405
```

> ⚠️ If intelligence code was changed in this session, it must be **deployed via GHA first**
> (`git push` → wait for `deploy.yml` to complete). The tunnel connects to the deployed VM —
> pushing without deploying means you're testing the old version.

### Run Newman against the local API

```powershell
# VS Code: Tasks: Run Task → "api: newman (local)"
```

All requests must be green before marking the item done. If a test fails on an upstream dependency
(OM / intelligence), check the service first rather than changing the test assertion.

### Browser verification

Open `http://localhost:5173` and verify:
- The wired surface shows real data (not mock data or empty state)
- Loading state shows while the request is in-flight
- Error state handles a 4xx / 5xx gracefully (no blank screen, no unhandled exception)
- DevTools Network: expected requests appear, all 200, no unexpected 401s

---

## Step 6 — Mark done and summarise

1. Update `/memories/repo/wire-up-status.md` — move the completed item from "Remaining" to
   "Completed" with today's date (YYYY-MM-DD)

2. Present a **summary of changes**:

```
## Done: [Item name] — YYYY-MM-DD

### API endpoints now ready
- METHOD /path — one-line description

### Response objects
- SchemaName: { field1: type, field2: type, ... }

### UI surfaces now wired
- ViewName.vue: replaced [mock source] with [client.js function]

### Newman CI
- N new request(s) added to ci.postman_collection.json

### Docs updated
- api/BFF_API_REQUIREMENTS.md §N.N: Mocked → Wired
- [ADR created if applicable]
```

3. Return to **Step 1** — present the updated state table and next suggestions.
