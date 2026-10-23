---
description: "Autonomously QA one named UI feature end-to-end, record every finding as a BUG-NNNN / SEC-NNN entry, fix them securely (parallelised with subagents where independent), verify, update the docs, and commit + push. Browser work is batched into consolidated Playwright scripts to keep cost down."
mode: agent
---

# /test-feature-and-make-fixes — Automated QA and remediation of one UI feature

Feature or area to test: ${input:feature:Short name (e.g. "notebooks", "sidebar catalog", "chat")}

Use this to get broad, repeatable coverage of a specific feature and drive every finding through
to a verified, documented, pushed fix — without a human clicking through it. This is
**exploratory automated QA plus remediation**, not a committed test suite: per the `ui` repo's
"no testing infrastructure unless asked" rule, no `.spec.js`/Playwright test files get written.
What *does* get written: bug reports, security register rows, code fixes, and their docs.

For diagnosing a single already-known bug reported by the user, use `/ui-e2e-bug-triage` instead
— that flow is cheaper for one issue because it skips browser automation entirely. This prompt is
for *finding and fixing* issues across a whole feature.

**Test data:** don't clean up anything created during a run (notebooks, chats, cells, published
blobs). After fixes land, create *fresh* data via user actions (or re-run the relevant pipeline)
to verify — data persisted before a fix can still carry the bug's old shape (e.g. a cell saved
without a field the fix now adds), so never verify against it.

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

## Browser setup — avoid background-tab throttling

The integrated browser page is usually not the focused tab, so Chromium throttles it:
`document.visibilityState === 'hidden'`, `requestAnimationFrame` stops firing, timers are clamped.
Symptoms: `locator.click()` hangs on "waiting for element to be visible, enabled and stable"
(the stability check uses rAF), the SSE reader in `client.js` stalls on its `yieldToBrowser()`
rAF await, and typing animations freeze mid-word.

Run this once right after opening/reusing the page, and again after any new page or reload:

```js
const cdp = await page.context().newCDPSession(page)
await cdp.send('Emulation.setFocusEmulationEnabled', { enabled: true })
await page.bringToFront()
return await page.evaluate(() => ({ visibility: document.visibilityState, focus: document.hasFocus() }))
```

Expect `{ visibility: 'visible', focus: true }`. If a scenario still stalls, re-check this first.

Other gotchas:

- `window`/`document` only exist inside `page.evaluate(...)`, not in the script's own scope.
- `fetch` inside `page.evaluate` must use the absolute API base (`http://localhost:5000/...`) —
  relative paths hit the Vite dev server and 404.
- A script that exceeds `timeoutMs` returns a `deferredResultId` — resume with it, don't re-run
  the code (that double-fires clicks/requests).
- To inspect raw SSE frames, wrap `window.fetch` in-page *before* triggering the action, `tee()`
  the `/agent/send` body, and push `{event, data}` into `window.__sseLog`; read it back later.
- Prefer `page.evaluate(() => el.click())` over `locator.click()` when an element sits in a
  scroll container or overlaid popover.
- Keep PowerShell terminal commands on one line — multi-line commands get mangled.

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
   `intelligence`? A listening port is not proof of health: hit the upstream's `/health` with a
   5s timeout (e.g. `Invoke-WebRequest http://localhost:8001/health -TimeoutSec 5`) — a stale IAP
   tunnel keeps the port open with sockets stuck in `CloseWait`. If an upstream is down, ask the
   user to restart it and stop; don't burn scenarios against a dead dependency.
4. **Auth tier** — which decorator guards each route (`@session_required` mints an anon session
   lazily, `@user_required` needs a `user`-tier session, no decorator = public)? Note it so you
   can verify the boundary is actually enforced (see Security pass below).
5. **Existing records** — read `documentation/bugs/README.md` and
   `documentation/security/risks.md` in each in-scope repo: note the next free `BUG-NNNN` /
   `SEC-NNN` IDs and any open entries that already cover this feature, so you update rather than
   duplicate them.

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
- **Auth boundary** — at least one check per auth tier from Step 1: `@user_required` routes
  return 401 for an anonymous session, `@session_required` routes mint an anon session rather
  than 401, and id-based lookups return 404 for another session's resource. Drive this with a
  terminal `Invoke-WebRequest` (fresh `-SessionVariable` = fresh anon session), not UI navigation.
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
   intelligence tunnel if needed) — check listening ports *and* a health request, don't assume.
2. Open/reuse one browser page for the whole session (`open_browser_page`, reuse via `pageId`
   across scenarios rather than opening a new page each time), then apply the throttling
   workaround from "Browser setup" above.
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

## Security pass (lightweight — record everything it surfaces)

While you're already driving real requests through this feature, note anything that fits an OWASP
Top 10 category, same spirit as `/ui-e2e-bug-triage`'s security lens, but don't go looking beyond
what this feature's scenarios already surfaced:

- Secrets/internal URLs/stack traces leaking into responses or console
- A route in scope with the wrong (or no) auth decorator, or an id-based lookup that doesn't
  check ownership
- Unsanitised content reaching `v-html`/`ui/src/markdown.js`
- Missing input size/row limits on anything user-supplied

Every confirmed security finding becomes a `SEC-NNN` row in Step 4 — no confirmation needed.

---

## Step 4 — Record findings

Before fixing anything, write every confirmed finding down using the repo conventions (see
`documentation/bugs/README.md` and `documentation/security/risks.md` in each repo):

- **Functional bug or API contract drift** → a `BUG-NNNN-kebab-title.md` file in the repo where
  the **root cause** lives, from that README's template, `Status: open`, plus an Index row. Fill
  Summary, Repro steps, Root cause (file + the exact faulty logic, confirmed by reading the code —
  not guessed from symptoms) and evidence. Leave Fix/Verification as `TBD`.
- **Cross-repo bug** (root cause in one repo, symptom or defensive fix in another) → a file in
  each affected repo, cross-linked under **Related** with full `https://github.com/big-lake/...`
  URLs.
- **Security finding** → a row in that repo's `risks.md` Open findings table (severity, one-line
  notes with evidence). If it's also a behavioural bug, create both and cross-link them.
- **Already recorded** → update the existing `BUG`/`SEC` entry instead of creating a new one.
- **Not a finding** — unconfirmed suspicions and anything blocked by the environment (upstream
  down, needs a second account, needs prod data) go in the report's "Not covered" section only.

---

## Step 5 — Fix plan

For each recorded bug, decide:

- **Fix** — the root-cause change (not a symptom patch), the exact files to touch, the repo, and
  whether the API contract changes.
- **Auto-fix vs escalate.** Auto-fix when the root cause is clear and the change is contained.
  **Escalate** (leave `open`, propose the approach in the report, don't fix) when the fix needs a
  design choice between real alternatives (→ suggest an ADR), touches infra/Terraform/prod config,
  secrets/IAM, the auth/session model, or a data migration.
- **Batches.** Bugs with disjoint file sets run in parallel (one subagent each). Bugs that share a
  file go to the same subagent or a later batch — **never two subagents on one file**.

Post the plan as a short table (`BUG → files → batch → auto-fix/escalate`) and proceed without
waiting, unless something is escalated or the plan touches more than ~10 files.

**Every fix must be secure:**

- Minimal root-cause diff; no refactors or "improvements" beyond the bug.
- Never weaken auth decorators, the CSRF/same-origin guard, CORS, sanitisation (`nh3`,
  `markdown.js`), SQL read-only enforcement, or size/row/count caps.
- Validate untrusted input at the boundary; never interpolate user input into SQL, HTML, or shell
  — use the existing helpers (`safe_identifier`, `html_sanitize`, sqlglot validation).
- No secrets, credentials, or internal URLs in code, logs, or error bodies; errors use the
  ADR-0001 envelope with no stack traces.
- If a fix creates or changes a security property, add/update the `SEC-NNN` row for it.

---

## Step 6 — Execute fixes with subagents

Launch one `runSubagent` per bug group in the batch, **in parallel**. Each prompt must be
self-contained:

- Absolute repo path, the `BUG` file path to read, the root cause, and the **exact files it may
  edit** (nothing else).
- The security requirements from Step 5 and the repo's conventions (`ui`: no new deps, no
  abstractions, inline over extract; `api`: thin routes, stateless services).
- If the API shape changes, the contract-first loop: Marshmallow schema + route docstring →
  implementation → `scripts/update_openapi_baseline.py` → a Newman request in
  `postman/collections/ci.postman_collection.json`.
- Run `get_errors` on every changed file. **Do not** commit, push, edit docs, or use browser
  tools. Return a per-file before/after summary.

Subagents only edit code; the main agent owns docs, the browser, and git. When a batch returns:
read each changed region yourself, run `get_errors` across all changed files, and check
`git status` that nothing outside the planned file sets was touched. Then start the next batch.

---

## Step 7 — Verify

- In the single shared browser page, **sequentially** (browser checks never run in parallel):
  re-run every originally failing scenario, plus a quick regression of the happy path.
- Use fresh data created via user actions or a pipeline re-run (see "Test data" above).
- For API changes, confirm the Flask dev server reloaded (check its terminal; restart the task if
  not), then run the `api: newman (local)` task.
- If a fix fails verification, send it back through Step 6 once. If it fails again, revert *that
  bug's* edits (only files with no other uncommitted changes), leave the `BUG` `open` with notes on
  what was tried, and list it as escalated.

---

## Step 8 — Update documentation

- **BUG files** — `Status: fixed`, `Fixed:` date, **Fix** (what changed and why it's the root
  cause), **Verification** (the concrete evidence). Update the Index row.
- **SEC rows** — fixed → move to "Resolved findings" with the date; partially fixed →
  `mitigated` with notes on what remains.
- **Other docs** — README gotchas/behaviour per the org documentation standard;
  `BFF_API_REQUIREMENTS.md` and `openapi.json` if the contract changed.
- Don't edit `.github/TODO.md` — flag any drift in the report instead.

---

## Step 9 — Commit and push

- Per repo, run `git status` first. Stage **only** files this run created or changed (never
  `git add -A`). Leave pre-existing unrelated changes alone and list them in the report.
- Match the repo's commit style (`git log --oneline -10`, conventional `fix(scope): ...`). One
  commit per repo with the fixes and their BUG/SEC docs; reference the IDs in the message.
- Then record the commit SHAs in the BUG `Fixed:` lines and SEC "Resolved" rows with a follow-up
  `docs(bugs): record fix SHAs` commit — never amend.
- Push each repo. If a push is rejected (remote ahead), stop and report — no force-push, no
  `--no-verify`, no rebase without asking.

---

## Step 10 — Final report

Concise, not a transcript of every tool call:

```
## Test report — ${input:feature}

### Scenarios run
| # | Scenario | Result | Evidence |
|---|---|---|---|
| 1 | ... | ✅ / 🐛→✅ / ❌ | status/body or DOM snippet |

### Bugs
| ID | Title | Severity | Status | Fix commit |
|---|---|---|---|---|

### Security
| ID | Title | Severity | Status |
|---|---|---|---|

### Escalated (not auto-fixed)
- BUG/SEC ID — why it needs a human decision — proposed approach

### Not covered
- scenarios skipped and why (upstream down, needs a second account, needs prod data)

### Commits pushed
| Repo | SHA | Message |
|---|---|---|

### Left alone
- pre-existing uncommitted changes found in working trees
```
