---
description: "Make a change that touches the UI and/or API using the mock-first, contract-first loop. Handles new endpoints, changed endpoints, and UI-only changes."
mode: agent
---

# API + UI change — mock-first, contract-first loop

Use this prompt for any change that touches the UI, the API, or both. Follow the phases in order. Do not skip ahead — the mockup and contract steps exist to catch shape mismatches before implementation, not after.

Change to make: ${input:changeDescription:Describe the change (e.g. "Add a dataset detail panel to the Catalog view showing row count and last updated date")}

---

## Phase 1 — Clarify before starting

Before writing any code, ask the user clarifying questions. Do not proceed until you have enough context to propose a clear plan. Good questions to consider:

- **Scope:** Does this change require a new endpoint, a change to an existing endpoint, or only UI changes?
- **Data:** What data needs to be shown or submitted? Where does it come from (existing field, new query, external service)?
- **UX:** Where does this appear in the UI — new view, new panel, new column, inline edit? Is there a mobile/responsive concern?
- **Edge cases:** What should the UI show while loading, on error, or when there's no data?
- **Auth:** Is this available to all authenticated users, or restricted?

Make suggestions where the user may not have considered implications (e.g. "this field isn't in the current response — we'd need to either extend the endpoint or add a new one; I'd suggest X because Y").

Summarise your understanding and proposed approach, and ask the user to confirm before moving to Phase 2.

---

## Phase 2 — Mock up the UI first

Implement the UI change using **hardcoded mock data**. Do not call the API yet. The goal is to validate layout and UX before committing to an API shape.

- Use inline `const mockData = { ... }` or a `ref([...])` with static values in the component.
- Keep the mock realistic — use the same field names you expect the real API to return.
- Apply scoped styles. No CSS frameworks. No new npm dependencies.
- If adding to an existing view, show the user what the view looks like before and after.

**Show the result in the browser** (`http://localhost:5173`). Walk the user through the mockup and ask:
- Does the layout match what you had in mind?
- Are there any fields missing or labels that need changing?
- Any interaction (click, hover, expand) that isn't there yet?

**Do not proceed to Phase 3 until the user is happy with the mockup.**

---

## Phase 3 — Determine the API scope

Based on the confirmed mockup, identify which of the three cases applies and tell the user which path you're taking:

| Case | What it means | Phases that follow |
|---|---|---|
| **A — New endpoint** | The data the mockup needs doesn't exist in any current endpoint | 4 → 5 → 6 → 7 → 8 → 9 |
| **B — Changed endpoint** | An existing endpoint needs new fields, changed shape, or different behaviour | 4 → 5 → 6 → 7 → 8 → 9 |
| **C — UI only** | All data is already available via existing endpoints; no API change needed | Skip to 8 → 9 |

> **API-only changes (no UI involved):** If it turns out no UI change is needed at all, this prompt is the wrong tool. Stop here and use `/new-api-endpoint` instead — it covers the full API-only contract-first loop without UI phases.

For cases A and B, read `api/openapi.json` and `api/app/schemas/__init__.py` now to understand the current contract before proposing changes.

---

## Phase 4 — Design the contract (Cases A and B only)

In the `api` repo:

- Add or update the Marshmallow schema in `app/schemas/__init__.py`. Name new schemas `<Verb><Resource>Request` / `<Verb><Resource>Response`.
- If a **new top-level schema** is added (not only referenced via `fields.Nested(...)`), register it in `app/openapi.py` with `_safe_register()`. Never register nested schemas manually — `MarshmallowPlugin` handles them; doing it again raises `DuplicateComponentNameError`.
- Add or update the YAML docstring block on the view function — copy the pattern from `app/routes/auth.py`. Cover `requestBody`, `responses`, and `parameters`.

Show the proposed schema diff and ask if the shape looks right before implementing the route.

---

## Phase 5 — Implement the route + service (Cases A and B only)

- Routes live in `app/routes/` — keep them thin: validate input, call service, return response.
- Business logic lives in `app/services/`.
- Apply `@require_auth` if the endpoint is not public.
- For case B (changed endpoint), check `api/BFF_API_REQUIREMENTS.md` and `ui/src/api/client.js` for any existing callers that may be affected by the shape change — flag them to the user before proceeding.

---

## Phase 6 — Refresh the OpenAPI baseline (Cases A and B only)

```powershell
# From VS Code: Tasks: Run Task → "api: update OpenAPI baseline"
# Or:
cd api ; python scripts/update_openapi_baseline.py
```

Commit `openapi.json` alongside the route change. CI fails if they drift.

Verify the new or changed path appears:

```powershell
curl http://localhost:5000/openapi.json | ConvertFrom-Json | Select-Object -ExpandProperty paths | Get-Member -MemberType NoteProperty | Select-Object Name
```

---

## Phase 7 — Add a Newman request (Cases A and B only)

Edit `api/postman/collections/ci.postman_collection.json`. Follow the template in `.github/instructions/postman-ci.instructions.md`:

- Insert after Bootstrap — the session cookie carries over automatically, no per-request auth setup needed.
- Assert status code **and** response shape (not hardcoded values).
- If a later Newman request needs an ID from this response, stash it with `pm.environment.set(...)`.

Run the CI suite locally to confirm green:

```powershell
# From VS Code: Tasks: Run Task → "api: newman (local)"
```

Do not proceed until Newman is green.

---

## Phase 8 — Wire the real data into the UI (all cases)

Replace the hardcoded mock data from Phase 2 with real API calls.

- **Open `api/openapi.json`** in this workspace to get exact field names and types. Do not guess.
- Add or update the function in `ui/src/api/client.js` — one function per endpoint, `credentials: 'include'`, no `Authorization` header, surface errors as thrown exceptions.
- Update the component to call the `client.js` function, handle loading and error states.
- Keep `.vue` files under ~300 lines. If the component has grown beyond that, extract a focused child component into `src/components/` — but only where there's a clear boundary.

**Verify in the browser** (`http://localhost:5173`). Check:
- Happy path renders correctly with real data.
- Loading state shows while the request is in-flight.
- Error state handles a 4xx / 5xx gracefully.
- DevTools Network tab shows no unexpected 401s.

---

## Phase 9 — Update gap docs (all cases)

Update `api/BFF_API_REQUIREMENTS.md`:

- Add or update the endpoint section with the final request/response shape.
- Update the **Status** line (e.g. `Mocked` → `Wired`, or `Planned` → `Wired`).
- Update the **Gaps / TODO** table — mark resolved gaps as done, add any new ones discovered during implementation.
- Note which Vue component uses which field.

---

## Checklist before commit

**API (cases A and B):**
- [ ] Marshmallow schema updated; new top-level schemas registered in `app/openapi.py`
- [ ] Route docstring YAML matches the implementation
- [ ] `openapi.json` regenerated and committed in the same change
- [ ] Newman request added and passing locally

**UI (all cases):**
- [ ] All HTTP calls go through `client.js` — no direct fetch in components
- [ ] `credentials: 'include'` on every call; no `Authorization` header
- [ ] Field names in the view match `openapi.json` exactly
- [ ] Loading and error states handled
- [ ] No new npm dependencies
- [ ] View files under ~300 lines, or split into focused child components
- [ ] Router updated if a new route was added

**Wrap-up:**
- [ ] `api/BFF_API_REQUIREMENTS.md` updated with final shape and status
