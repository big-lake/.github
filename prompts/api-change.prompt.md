---
description: "Make any change to the BigLake API — new endpoint, modified endpoint, service-layer refactor, or security fix. Covers context loading, design proposal, implementation, testing, and documentation."
mode: agent
---

# API change — context-first, confirm-before-implement

Use this prompt for any non-trivial change to the `api` repo: new endpoint, changed endpoint, service refactor, auth change, or security fix. Follow the phases in order. Do not skip ahead.

Change to make: ${input:changeDescription:Describe the change (e.g. "Add GET /notebooks/{id}/analytics returning view count and last viewed date")}

---

## Phase 1 — Load context

Read the following files before writing a single line of code:

1. [`api/README.md`](../api/README.md) — architecture diagram, design principles, downstream inventory, internal route map, agent notes
2. [`api/documentation/adr/README.md`](../api/documentation/adr/README.md) — ADR index; identify which ADRs are relevant to this change type and read them:
   - Auth/session changes → ADR-0003, 0014, 0015, 0016, 0017
   - SQL/query changes → ADR-0006, 0007, 0009
   - Chat/SSE changes → ADR-0012, 0019, 0020
   - Agent/notebook changes → ADR-0019, 0020
   - Storage changes → ADR-0004, 0010, 0018
   - OpenAPI/contract changes → ADR-0001, 0013
   - Any change touching security → ADR-0001, 0002, 0007

Do **not** load `BFF_API_REQUIREMENTS.md` or `risks.md` yet — those are only read in Phase 3 if the change warrants it.

While reading, form an initial view on:
- Which downstream surface(s) this change touches (OpenMetadata, intelligence, DuckDB, GCS, SQLite)
- Whether the public API surface (`openapi.json`) is likely to change
- Whether a new ADR might be warranted (only if the change introduces a non-obvious design choice others will need to understand)

---

## Phase 2 — Clarify intent and scope

Before proposing anything, ask the user the clarifying questions specific to this change. Do not proceed until you have enough information to write a concrete proposal. Consider:

- **What is the trigger?** Is this a new feature, a bug fix, a security fix, or a refactor?
- **Surface:** Does this change the public route surface, or only internal service logic?
- **Auth:** What auth tier is needed — `anon`, `user`, or service-to-service? Is the route scoped to the session owner?
- **Data shape:** What does the caller send, and what should the response contain? What are the types?
- **Error cases:** What can go wrong, and how should failures be communicated? (reference ADR-0001 error contract: `input | auth | forbidden | not_found | conflict | upstream | timeout`)
- **Idempotency:** If this is a write, is it idempotent? Can the caller retry safely?
- **Security:** Could this change introduce injection risk, IDOR, over-broad data exposure, or rate-limit bypass? Check against OWASP Top 10.
- **Downstream impact:** Does the change require the intelligence service, DuckDB, OpenMetadata, or GCS to do something new?
- **Existing risks:** Are there open SEC-NNN items in `risks.md` that this change should close or would interact with?

Summarise your understanding and proposed approach, and **explicitly ask the user to confirm before moving to Phase 3**.

---

## Phase 3 — Propose design

Present the full design before writing implementation code. Include all of the following that apply:

### Request / response shape

Show the proposed Marshmallow schema (not just prose). Example:

```python
# app/schemas/__init__.py
class GetAnalyticsResponse(Schema):
    notebook_id = fields.Str(required=True)
    view_count   = fields.Int(required=True)
    last_viewed  = fields.Str(allow_none=True)  # ISO 8601 or null
```

And the corresponding JSON shape as the caller will see it:

```json
{ "notebook_id": "abc", "view_count": 42, "last_viewed": "2026-06-08T03:00:00Z" }
```

### Route signature

```
GET /notebooks/{id}/analytics
Auth: session (user tier)
Response: 200 GetAnalyticsResponse | 404 not_found | 403 forbidden
```

### Service boundary

Which service module handles this, and what does it call downstream? One sentence per layer.

### Security considerations

List any OWASP Top 10 concerns that apply and how they are mitigated:
- SQL injection → parameterised queries / sqlglot validator
- IDOR → ownership check against session_sub before returning data
- Auth bypass → `@require_auth` applied; tier checked if `user`-only
- Sensitive data exposure → confirm no secrets or PII leak into response

### ADR verdict

State explicitly:
- **No new ADR needed** — this follows established patterns (cite the ADR(s)). OR
- **New ADR warranted** — briefly describe the decision and the next ADR number to use.

### BFF_API_REQUIREMENTS verdict

State explicitly:
- **Not applicable** — this change doesn't touch the UI-facing surface (internal refactor, service change, etc). OR
- **BFF_API_REQUIREMENTS.md needs updating** — read it now (`api/BFF_API_REQUIREMENTS.md`), identify which section and what changes.

### Security verdict

State explicitly:
- **No security review needed** — this follows established patterns, no new inputs, no new data exposure. OR
- **Security review needed** — read `api/documentation/security/risks.md` now, check for open SEC-NNN items relevant to this change, and list any new concerns below with proposed mitigations.

**Ask the user to confirm this design before moving to Phase 4.**

---

## Phase 4 — Implement

After explicit confirmation, implement in this order:

1. **Schema** — add/update `app/schemas/__init__.py`. Register new top-level schemas in `app/openapi.py` via `_safe_register()`.
2. **Service** — add/update the service module in `app/services/`. Stateless functions only — no business logic in routes.
3. **Route** — add/update the route in `app/routes/`. Keep it thin: validate input → call service → return. Add `@require_auth` if needed. Add the YAML docstring block (follow the pattern in existing routes — `summary`, `requestBody`, `responses`, `parameters`).
4. **DB migration** — if SQLite schema changes, add a migration step (check `app/db/__init__.py` for the pattern).

Conventions to follow (from ADR-0001 and the copilot instructions):
- Routes do not contain business logic.
- Services are stateless functions, not classes.
- All errors use `make_error(message, kind=...)` from `app/shared/errors.py`.
- Log at WARNING for recoverable upstream failures; ERROR/exception for unexpected failures.
- Use `current_request_id()` for `X-Request-Id` propagation to downstreams.
- Session ownership checks use `session_sub` from `request.session`, not `user_id`.

---

## Phase 5 — Test

### OpenAPI baseline

If the public surface changed (new route, changed request/response shape, new status code):

```powershell
# In api/ with venv active:
python scripts/update_openapi_baseline.py
```

Verify the diff in `openapi.json` matches exactly what you intended. Commit it alongside the code.

If the surface did not change (internal refactor only), skip this step — say so explicitly.

### Newman CI collection

If a new route was added or an existing route's behaviour changed, add or update a Newman request in `postman/collections/ci.postman_collection.json`:

- Read `.github/instructions/postman-ci.instructions.md` for the exact format before editing.
- Insert after the Bootstrap request (session cookie carries automatically).
- Assert: status code AND response shape (not literal values, unless a value is load-bearing).
- Save any ID the next test needs via `pm.environment.set(...)`.

Then run the CI collection locally:

```powershell
# From VS Code: Tasks → "api: newman (local)"
```

All tests must pass (0 failures) before proceeding. If a test fails, fix the implementation — do not patch the test to hide the failure.

---

## Phase 6 — Document

Update every document that is now stale. Work through this checklist:

### Always

- [ ] **`api/README.md`** — update the downstream inventory, internal route map, and/or agent notes if any of those sections are now inaccurate. Do not rewrite sections that are still correct.

### If the UI-facing surface changed

- [ ] **`api/BFF_API_REQUIREMENTS.md`** — update the relevant section to reflect the new wired state. Mark previously mocked items as wired if applicable.

### If an architectural decision was made

- [ ] **New ADR** — create `api/documentation/adr/NNNN-kebab-title.md` following the MADR template (Status / Context / Decision / Consequences). Add a row to `api/documentation/adr/README.md`.

### If a security finding was introduced or resolved

- [ ] **`api/documentation/security/risks.md`** — add a new `SEC-NNN` row for any new finding. Change status to `resolved` for any finding this change closes; include the commit SHA in the resolved row (amend after committing if needed).

### Never

- Do not create new markdown files outside the structure above unless explicitly asked.
- Do not add docstrings or comments to code you did not touch.
