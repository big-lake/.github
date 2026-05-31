---
description: "Add a new API endpoint end-to-end across the api and ui repos using the contract-first 7-step loop. Updates schemas, route, OpenAPI baseline, Newman CI suite, UI client function, view wiring, and BFF_API_REQUIREMENTS.md."
agent: "agent"
---

Add a new API endpoint following the **contract-first 7-step loop** documented in [.github/DEVELOPMENT.md](../DEVELOPMENT.md#cross-repo-workflow--the-contract-first-loop).

Before starting, confirm with the user:
- Endpoint method + path (e.g. `GET /catalog/datasets/{id}/columns`)
- Purpose in one sentence
- Which UI view(s) will consume it (if any) — UI work can be deferred
- Auth tier (`anon` ok, or `user`/`user_recovery` required)

Then execute the steps in order. **Do not skip ahead** — each step has a verifiable artifact.

## Step 1 — Design the contract (api only)

- Add a `XxxRequest` and/or `XxxResponse` Marshmallow schema to [api/app/schemas/__init__.py](../../api/app/schemas/__init__.py). Match the patterns of existing schemas exactly (snake_case fields, `Nested` for sub-objects, `dump_only`/`load_only` where appropriate).
- Show the user the schema diff and confirm field names/types before proceeding.

## Step 2 — Implement the route + service

- Add the route in the appropriate blueprint under [api/app/routes/](../../api/app/routes/). Routes validate input and call services; **no business logic in the route**.
- Add or extend a function in [api/app/services/](../../api/app/services/) for the logic.
- Include a full YAML docstring on the view function (copy a neighbour for the exact format — `summary`, `description`, `tags`, `requestBody`/`parameters`, `responses` including the `$ref` to the schemas from Step 1).
- Use `@require_auth` (or the matching tier helper) unless the endpoint is genuinely public.
- If a new top-level schema was added (not just `Nested`), register it in [api/app/openapi.py](../../api/app/openapi.py) via `_safe_register()`.

## Step 3 — Refresh the OpenAPI baseline

Run from `api/`:
```powershell
.\.venv\Scripts\Activate.ps1
python scripts/update_openapi_baseline.py
```

Verify the new path appears in `api/openapi.json` and the schemas are present in `components.schemas`. Commit `openapi.json` alongside the route change — CI fails if it drifts.

## Step 4 — Add a Newman request

Edit [api/postman/collections/ci.postman_collection.json](../../api/postman/collections/ci.postman_collection.json) per [postman-ci.instructions.md](../../api/.github/instructions/postman-ci.instructions.md):
- Place the new request after the `Bootstrap dev session` request, grouped logically by domain.
- Always assert status code; assert shape (field existence + types), not values.
- For POST/PUT, include a `body` block.
- Save any IDs needed by later requests via `pm.environment.set(...)`.

Then run the suite against your local API:
```powershell
# In one terminal (from api/):
.\.venv\Scripts\Activate.ps1
python run.py

# In another terminal (from api/):
python -c "import sys, json, yaml; json.dump(yaml.safe_load(open(sys.argv[1], encoding='utf-8')), open(sys.argv[2], 'w'))" "postman/environments/Big Lake — Local.environment.yaml" "newman-env.json"
npx --yes newman run "postman/collections/ci.postman_collection.json" --environment "newman-env.json"
Remove-Item newman-env.json
```

Or use the `api: newman (local)` VS Code task from the multi-root workspace.

**Green here means the contract works end-to-end.** Steps 5–6 only happen if the user said the UI consumes this endpoint.

## Step 5 — Add the `client.js` function (ui)

- Open `../api/openapi.json` and `../api/app/schemas/__init__.py` for exact shapes. **Never guess field names.**
- Add the function to [ui/src/api/client.js](../../ui/src/api/client.js) using the existing patterns:
  - `request(path, options)` helper for JSON endpoints
  - `credentials: 'include'` is already set in the helper
  - For SSE endpoints, follow the `streamChatMessage` pattern
- Keep the function name aligned with the endpoint's purpose (e.g. `getDatasetColumns(datasetId)`).

## Step 6 — Wire the view (ui)

- Find the view in [ui/src/views/](../../ui/src/views/) and add the call.
- Inline-first — do not extract helpers or composables (see [ui/.github/copilot-instructions.md](../../ui/.github/copilot-instructions.md) for the disposable-by-design rules).
- Only extract a child component if the view crosses ~300 lines and the new section has a clear template/script/style boundary.

Visually verify in the browser at `http://localhost:5173`.

## Step 7 — Update BFF_API_REQUIREMENTS.md

Edit [api/BFF_API_REQUIREMENTS.md](../../api/BFF_API_REQUIREMENTS.md):
- Add the endpoint section with the request/response shape (or reference the OpenAPI types).
- Update the **Status** line.
- Update the **Gaps / TODO** table — close any resolved gaps, add any new ones.
- Note which UI component(s) use which fields.

## Final check

Run once more before handing back:
- `python scripts/update_openapi_baseline.py` — ensure no drift
- `npx newman run ...` — ensure suite still green
- Browser smoke test if UI was touched

Summarise the diff and the verifiable evidence (Newman pass, OpenAPI updated, UI screenshot if relevant).
