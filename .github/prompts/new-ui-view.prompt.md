---
description: "Add a new UI view that consumes existing API endpoints. Follows the disposable-by-design rules: no new dependencies, inline-first, single-file Vue component, modular only when over the size threshold."
agent: "agent"
---

Add a new UI view in the `ui` repo, consuming endpoints that **already exist** in the API. If the endpoint doesn't exist yet, use `/new-endpoint` instead.

Before starting, confirm with the user:
- View name and route (e.g. `/datasets/:id`)
- Which endpoint(s) it consumes — list them
- Whether it appears in the main nav

## Step 1 — Verify the API contract

Open `../api/openapi.json` (or `../api/app/schemas/__init__.py`) and read the exact request/response shape for every endpoint this view uses.

**Do not guess field names.** If the shape is unclear, run the Newman request for that endpoint locally and inspect the response.

## Step 2 — Ensure `client.js` covers the endpoints

Check [ui/src/api/client.js](../../ui/src/api/client.js) for functions matching the endpoints. If any are missing or have a wrong shape:
- Add or fix them now (do not work around in the view).
- Use the existing `request()` helper. `credentials: 'include'` is already set.

## Step 3 — Create the view

Create `ui/src/views/<Name>.vue` as a single-file component:

- `<script setup>` Composition API only
- `reactive()` for state — no Pinia, no composables, no helpers
- Inline templates and scoped styles
- Call `client.js` functions in `onMounted` or in response to user actions
- Use `v-if` / `?.` for fields that may be absent
- Keep under ~300 lines; if you cross that, extract focused child components into `ui/src/components/`

Follow the disposable-by-design rules in [ui/.github/copilot-instructions.md](../../ui/.github/copilot-instructions.md):
- **No new npm dependencies**
- **No wrapper/base components**
- **No TypeScript**
- **No testing infrastructure**

## Step 4 — Register the route

Edit [ui/src/router/index.js](../../ui/src/router/index.js):
- Import the view
- Add the route entry
- Mark `meta: { public: true }` only if the view is intentionally pre-auth

## Step 5 — Add nav entry (if needed)

If the user said the view appears in the main nav, edit [ui/src/App.vue](../../ui/src/App.vue) to add the link.

## Step 6 — Visual verification

Start the dev server (`Dev: API + UI` task) and walk through the new view in the browser:
- All endpoint calls succeed (check DevTools network panel)
- Empty states render
- Loading states render
- Error states render (kill the API briefly to test)

## Step 7 — Update BFF_API_REQUIREMENTS.md

Edit [api/BFF_API_REQUIREMENTS.md](../../api/BFF_API_REQUIREMENTS.md):
- Update field usage notes for the consumed endpoints (add your new component to the "used by" lists).
- If you discovered a gap during development (missing field, wrong shape), add it to the **Gaps / TODO** table.

## Final check

Summarise: new files, modified files, screenshot of the view, list of endpoints consumed.
