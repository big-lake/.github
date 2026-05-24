---
description: "Modify an existing API endpoint (request/response shape, query params, auth tier) without breaking the UI. Walks through schema update, route fix, OpenAPI rebaseline, Newman update, and UI client/view migration."
agent: "agent"
---

Modify an existing API endpoint. The risk here is **silent breakage of the UI** when the response shape changes. This prompt enforces the cross-repo loop in [.github/DEVELOPMENT.md](../DEVELOPMENT.md#cross-repo-workflow--the-contract-first-loop).

Before starting, confirm with the user:
- Which endpoint (method + path)
- What's changing (field added, renamed, removed, type changed, auth tier changed, new query param, etc.)
- Whether it's an additive change (safe) or breaking (requires UI fix in the same change)

## Step 1 — Find all current consumers

Search the UI for the endpoint:
- `grep` `ui/src/api/client.js` for the path
- `grep` the entire `ui/src/` for every field of the response that gets read (especially `v-if`, `?.`, destructuring)

Show the user the list of consuming components and the fields they use. **The user needs to know what will break.**

## Step 2 — Update the schema

- Edit [api/app/schemas/__init__.py](../../api/app/schemas/__init__.py) — add/rename/remove fields. Preserve dump/load semantics.
- For a breaking rename, prefer keeping the old field as `dump_only` for one release alongside the new one, then remove next change.

## Step 3 — Update the route + service

- Update the route's YAML docstring to match the new schema (parameters, requestBody, responses).
- Update the service function logic.
- If auth tier changed, update `@require_auth` accordingly.

## Step 4 — Refresh OpenAPI baseline

```powershell
cd api
.\.venv\Scripts\Activate.ps1
python scripts/update_openapi_baseline.py
```

Confirm `openapi.json` reflects every change. Commit it.

## Step 5 — Update the Newman request

Edit [api/postman/collections/ci.postman_collection.json](../../api/postman/collections/ci.postman_collection.json):
- Update the request body / query params for the changed endpoint.
- Update the `pm.test` assertions — assert the **new** shape.
- Add an assertion for any new field that's load-bearing in the UI.

Run the suite locally (`api: newman (local)` task). Green = the API change is correct.

## Step 6 — Migrate the UI client + views

For every consumer found in Step 1:
- Update [ui/src/api/client.js](../../ui/src/api/client.js) if the function signature changes.
- Update the view's template/script to use the new field names.
- Test in the browser — confirm the screens that consume this endpoint still work.

If a field was removed, search-and-destroy all references in the UI before declaring done.

## Step 7 — Update BFF_API_REQUIREMENTS.md

Edit [api/BFF_API_REQUIREMENTS.md](../../api/BFF_API_REQUIREMENTS.md):
- Update the endpoint's response shape.
- Update field usage notes (which component uses what).
- Update the **Gaps / TODO** table if this resolves a gap or creates a new one.

## Final check

- Newman green
- OpenAPI baseline updated
- All UI consumers updated and visually verified
- BFF_API_REQUIREMENTS.md current

Summarise: what changed, which UI files were touched, evidence (Newman pass, screenshot).
