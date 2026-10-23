---
description: "Fully exercise one named UI feature/area end-to-end via automated browser tools, verifying both API/network correctness and UI logic. Optimised to avoid expensive screenshot/snapshot spam — batches actions and network capture into consolidated Playwright scripts. Produces a findings report; does not fix bugs itself."
mode: agent
---

# /test-feature — Automated functional test of one UI feature

Feature or area to test: ${input:feature:Short name (e.g. "notebooks", "sidebar catalog", "chat")}

Use this to get broad, repeatable coverage of a specific feature without a human manually
clicking through it. This is **exploratory automated QA**, not a committed test suite — per the
`ui` repo's "no testing infrastructure unless asked" rule, nothing gets written to disk here
(no new `.spec.js`/Playwright test files) unless the user explicitly asks for that afterwards.

For diagnosing a single already-known bug reported by the user, use `/ui-e2e-bug-triage` instead
— that flow is cheaper for one issue because it skips browser automation entirely. This prompt is
for *finding* issues across a whole feature.

---

## Cost discipline — read this before touching the browser

Browser automation tools are token- and time-expensive. Default to the cheapest tool that answers
the question:

| Question | Cheapest tool |
|---|---|
| Does the API contract hold (status codes, response shape)? | `run_in_terminal` with `curl`/`Invoke-RestMethod`, or the `api: newman (local)` task — no browser needed |
| Does a UI flow work end-to-end (clicks → network → DOM updates)? | **One** consolidated `run_playwright_code` script per scenario: navigate, act, capture network, assert, return a small JSON/text summary |
| Is there a visual/layout bug? | `screenshot_page` targeted at the specific `ref`/`selector`, not the full viewport, and only after logic-level checks already pointed at that element |
| What's in the DOM right now? | Prefer `page.evaluate`/`page.$eval`-style queries inside `run_playwright_code` over `read_page`'s full accessibility tree, unless you genuinely need the whole tree to find an unfamiliar element |

Rules of thumb:

- **One script, many steps.** Don't do navigate → screenshot → click → screenshot → click →
  screenshot. Write a single Playwright snippet that performs the whole scenario and returns a
  compact result (e.g. `{status, responses: [...], finalText, errors}`).
- **Capture network inside the script**, not by inspecting devtools visually. Attach
  `page.on('request')` / `page.on('response')` (or `page.waitForResponse(...)` for a specific
  call) and collect `{url, method, status, bodyJson}` for the calls relevant to this feature only
  — don't dump every asset/tracking request.
- **No screenshots as a default sanity check.** Only take one when text/DOM evidence already
  suggests a rendering problem, and only of the relevant element.
- **Batch scenarios, don't interleave with re-reads.** Plan all scenarios up front (Step 2) so you
  aren't repeatedly re-discovering the DOM structure between runs.

---

## Step 1 — Establish the feature surface before opening a browser

Using code search (not browser tools), map out everything involved in "${input:feature}":

1. **UI layer** — which `ui/src/views/*.vue` and `ui/src/components/*.vue` implement it, and which
   functions in `ui/src/api/client.js` it calls.
2. **API layer** — which routes in `api/app/routes/` back those client calls, which service module
   they call into (`catalog_client.py`, `query_service.py`, `intelligence_client.py`, etc.), and
   the Marshmallow schemas in `api/app/schemas/__init__.py` that define the request/response
   contract. Cross-reference `api/openapi.json` for the committed baseline shape.
3. **Upstream dependencies** — does this feature depend on OpenMetadata, DuckDB/GCS, or
   `intelligence`? If `intelligence`, confirm the `iap: tunnel intelligence (:8001)` task is
   running — calls will fail with an upstream error otherwise.
4. **Auth tier** — does every route involved require `@require_auth`? Note it so you can verify
   unauthenticated/cross-user access is actually rejected (see Security pass below).

If the feature area is ambiguous or spans more than you'd expect, ask the user a short clarifying
question rather than guessing scope.

---

## Step 2 — Write the test plan before executing anything

Enumerate concrete scenarios as a checklist, covering at minimum:

- **Happy path(s)** — the primary intended flow(s), including any multi-step sequences (e.g.
  create → edit → delete a notebook).
- **Empty / loading states** — first-load with no data, in-flight request state.
- **Validation / error states** — invalid input client-side, and a deliberately bad request to
  confirm the API rejects it correctly (schema validation, row limits, SQL write rejection, etc.)
  and the UI surfaces that error sensibly rather than crashing or silently failing.
- **Auth boundary** — at least one check that the feature's routes 401/redirect when the session
  cookie is missing or invalid (drive this via a `run_playwright_code` fetch with credentials
  stripped, not full UI navigation, to keep it cheap).
- **Edge cases specific to this feature** — pagination/large result sets, concurrent
  updates, special characters in input, long-running operations (SSE streams for chat/agent), etc.
  Use your Step 1 mapping to decide what's actually relevant — don't pad the list with
  scenarios that don't apply.

Present the scenario list to the user briefly (one line each) before running — this is cheap and
lets them redirect you if you've misunderstood the feature's scope. Don't wait for explicit
sign-off unless the plan looks unusually large; a short "running these N scenarios now" is enough.

---

## Step 3 — Execute

For each scenario:

1. Confirm the dev stack is up (`api: dev (Flask :5000)`, `ui: dev (Vite :5173)`, and the
   intelligence tunnel if needed) — check via the task list/terminal state, don't just assume.
2. Open/reuse one browser page for the whole session (`open_browser_page`, reuse via `pageId`
   across scenarios rather than opening a new page each time).
3. If the feature requires auth, rely on the existing dev auto-login (`POST /auth/dev/session`
   fired from `main.js` on load) rather than re-implementing login per scenario.
4. Run the scenario as one `run_playwright_code` script, returning:
   - the sequence of captured `{url, method, status, bodyJson}` for relevant requests
   - any `console.error`/unhandled promise rejections seen on the page
   - a short extracted text/state snapshot proving the expected end state (e.g. "notebook list
     contains the new title", "error banner text equals X") — not a screenshot
5. Compare captured responses against the schema/`openapi.json` contract from Step 1 — flag any
   drift (extra/missing fields, wrong status code, wrong `kind` in error envelope) even if the UI
   happened to render fine anyway.
6. Only escalate to `screenshot_page` (targeted element) if something looks visually broken that
   text/DOM evidence can't confirm.

---

## Security pass (lightweight, don't fix — just flag)

While you're already driving real requests through this feature, note anything that fits an OWASP
Top 10 category, same spirit as `/ui-e2e-bug-triage`'s security lens, but don't go looking beyond
what this feature's scenarios already surfaced:

- Secrets/internal URLs/stack traces leaking into responses or console
- A route in scope missing `@require_auth`, or an id-based lookup that doesn't check ownership
- Unsanitised content reaching `v-html`/`ui/src/markdown.js`
- Missing input size/row limits on anything user-supplied

Surface findings separately from functional bugs, with evidence (status/body), and suggest a
`SEC-NNN` row in the relevant repo's `documentation/security/risks.md` — don't add one yourself
without confirming with the user first.

---

## Step 4 — Report findings

Produce a concise report, not a transcript of every tool call:

```
## Test report — ${input:feature}

### Scenarios run
| # | Scenario | Result | Evidence |
|---|---|---|---|
| 1 | ... | ✅ / ❌ | status/body or DOM snippet |

### Bugs found
- [severity] description — repro steps — evidence (request/response or console error) — suspected
  layer (ui view / client.js / api route / service / upstream)

### API contract drift
- any response shape that didn't match schemas/openapi.json, even if the UI tolerated it

### Security notes
- anything from the Security pass, with evidence

### Not covered
- scenarios you deliberately skipped and why (e.g. requires prod data, requires a second user
  account, requires an upstream service not available locally)
```

Do not fix anything in this pass. If the user wants a bug fixed, hand it to `/ui-e2e-bug-triage`
(one bug at a time) or ask which finding to tackle first.
