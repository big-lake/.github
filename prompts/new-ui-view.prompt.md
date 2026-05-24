---
description: "Add a new view or significant UI feature that consumes existing API endpoints."
mode: agent
---

# New UI view / feature

Use this when adding UI that consumes endpoints that **already exist** in the API. If you need a new endpoint too, use [new-endpoint.prompt.md](new-endpoint.prompt.md) first.

Feature to add: ${input:featureDescription:Short description (e.g. "Dataset detail page with column list")}

## 1. Identify the endpoints you'll call

Open `api/openapi.json` in this workspace and locate the relevant paths. That is the authoritative shape — do not guess field names. If something is unclear, read the Marshmallow schema in `api/app/schemas/__init__.py`.

## 2. Add `client.js` functions (one per endpoint)

In `ui/src/api/client.js`:

- One function per endpoint. No wrappers, no helpers, no composables.
- `credentials: 'include'`. No `Authorization` header.
- Surface errors as thrown exceptions; let the view decide how to display them.

## 3. Build the view

In `ui/src/views/` (or `ui/src/components/` for a focused sub-component):

- Vue 3 Composition API, `<script setup>`.
- No new npm dependencies. No CSS framework. Scoped styles.
- Keep each `.vue` file under ~300 lines. If it grows beyond that, extract a focused child component — but only when there's a clear boundary.
- Inline over extract: if logic is used once, keep it in the component.

## 4. Add to the router

Edit `ui/src/router/index.js`. Match the existing pattern (lazy import, meta tags if needed).

## 5. Browser-verify

```powershell
# From VS Code: Tasks: Run Task → "Dev: API + UI"
```

Open `http://localhost:5173`. Walk the happy path. Check DevTools Network for 401s or unexpected 4xx.

## Checklist before commit

- [ ] All HTTP calls go through `client.js`
- [ ] No new npm dependencies
- [ ] No `Authorization` header anywhere — cookie auth only
- [ ] Field names in the view match `openapi.json` exactly
- [ ] View files under ~300 lines, or split into focused components
- [ ] Router updated if a new route was added
