---
description: "Add a new endpoint to the BigLake API and wire it into the UI using the contract-first loop."
mode: agent
---

# New API endpoint — contract-first loop

Use this prompt when adding a new endpoint that the UI will consume. Follow each step in order; do not skip ahead. The numbered artifacts are how the API and UI stay in sync — both for humans and for coding assistants working in parallel.

Endpoint to add: ${input:endpointDescription:Short description (e.g. "GET /catalog/datasets/{id}/columns")}

## 1. Design the contract

In the `api` repo:

- Add or update the request/response schemas in `app/schemas/__init__.py` (Marshmallow). Name them `<Verb><Resource>Request` / `<Verb><Resource>Response`.
- If a new top-level schema is added (not only referenced via `Nested`), register it in `app/openapi.py` with `_safe_register()`.

Do **not** write the UI yet. The schema is the contract.

## 2. Implement the route + service

- Create or update the route in `app/routes/` — keep routes thin (validate, call service, return).
- Put business logic in `app/services/`.
- Add the YAML docstring block to the view function. Copy the pattern from `app/routes/auth.py`. Fill in `requestBody`, `responses`, `parameters`.
- Apply `@require_auth` if the endpoint is not public.

## 3. Verify the spec

Start the API (`python run.py`) and check:

```powershell
curl http://localhost:5000/openapi.json | ConvertFrom-Json | Select-Object -ExpandProperty paths | Get-Member -MemberType NoteProperty | Select-Object Name
```

The new path should appear. Open `http://localhost:5000/docs` to eyeball the rendered spec.

## 4. Update the OpenAPI baseline

```powershell
cd api ; python scripts/update_openapi_baseline.py
```

Commit `openapi.json` alongside the code change. CI fails loudly if this drifts.

## 5. Add a Newman request

Edit `api/postman/collections/ci.postman_collection.json`. Follow the template in `.github/instructions/postman-ci.instructions.md`:

- Insert after Bootstrap (cookie auth carries over automatically — no per-request setup).
- Assert status code AND response shape (not values).
- If a later request needs an ID from this response, save it via `pm.environment.set(...)`.

Run locally to confirm:

```powershell
# From VS Code: Tasks: Run Task → "api: newman (local)"
```

## 6. Add the `client.js` function

In the `ui` repo:

- **Open `api/openapi.json` in this workspace** — that's the authoritative request/response shape. Do not guess field names or types.
- Add a function to `src/api/client.js`. Match the existing patterns (single fetch, `credentials: 'include'`, parse JSON, surface errors).
- Do not create wrapper helpers, composables, or new layers. The function lives in `client.js`.

## 7. Wire the view and update gap docs

- Use the new `client.js` function in the appropriate view in `src/views/` (or extract a focused child component into `src/components/` only if the view grows past ~300 lines).
- Update `api/BFF_API_REQUIREMENTS.md`:
  - Add or update the endpoint's section with the final shape.
  - Move the gap row to "resolved" or update its status.

## Checklist before commit

- [ ] Marshmallow schema updated and (if top-level) registered in `app/openapi.py`
- [ ] Route docstring YAML matches the code
- [ ] `openapi.json` regenerated and committed
- [ ] Newman request added and passing locally
- [ ] `client.js` function uses field names matching `openapi.json`
- [ ] `BFF_API_REQUIREMENTS.md` reflects current state
- [ ] No new ui dependencies (philosophy in `ui/.github/copilot-instructions.md`)
- [ ] No pytest unless the service function has non-trivial logic
