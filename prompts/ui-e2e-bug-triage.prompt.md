---
description: "Triage UI end-to-end bugs, capture confirmed defects in each affected repo's bug register, create an ordered fix plan, and execute and validate fixes with subagents where useful. The user drives browser testing; security and deployment approval boundaries remain in force."
agent: agent
---

# /ui-e2e-bug-triage — Register, plan, fix, verify

Use this prompt once the stack is wired enough that a user can ask real questions in the `ui` and
expect real answers to come back from `api` → (`intelligence` and/or `catalog`/OpenMetadata and/or
the DuckDB query service). This prompt is for **diagnosing why that isn't working**, not for
building new surfaces — use `/wire-up` or `/new-endpoint` for that.

**Hard rule: do not use Playwright or any browser-automation tool in this flow.** The user drives
the browser manually (Chrome) and reports back. You may open the UI in VS Code's Simple Browser
purely to *look* at something together, but all real testing, clicking, and network-tab
inspection is done by the user.

Capture all reported bugs, then work through an **ordered fix plan**. Keep each root cause and
its verification distinct. Independent fixes may be delegated in parallel; dependent fixes must
run in order. Do not stop at diagnosis or a proposed plan: execute actionable local fixes unless
the user requests planning only. Pending manual retests do not block work on independent bugs.

---

## Security lens — applies to every step

Give security **extra consideration** throughout this flow, in two directions:

### Spotting existing exposures

End-to-end debugging puts you in front of real requests, responses, logs, and code paths — use
that to actively look for vulnerabilities, not just the functional bug. Examples worth checking
whenever they're in the path you're reading:

- **Data leakage** — secrets, tokens, HMAC keys, internal hostnames/URLs, stack traces, or raw
  upstream errors (OM, intelligence, DuckDB) appearing in response bodies, browser console, or
  logs. Error responses should use the `api/app/shared/errors.py` contract, not leak internals.
- **Broken access control** — routes missing `@require_auth`/`session_required`, or resources
  fetched by id without checking ownership (IDOR, e.g. another user's notebook or chat).
- **Injection** — user/LLM-supplied values reaching DuckDB SQL, OM queries, or shell/paths without
  `safe_identifier()`/parameterisation; read-only enforcement in `query_service.py` being bypassable.
- **XSS** — untrusted content (LLM output, catalog descriptions, query results) rendered via
  `v-html` / `ui/src/markdown.js` without sanitisation.
- **Session/CORS/cookie weaknesses** — overbroad credentialed CORS, missing cookie flags, CSRF
  exposure on state-changing endpoints.
- **SSRF / prompt injection** — proxied URLs or retrieved content steering upstream calls or
  agent/tool behaviour.
- **Abuse limits** — missing input size limits, row limits, or rate limits.

When you find one:

- **Surface it to the user explicitly**, separately from the bug you're working on — with the
  evidence, an OWASP Top 10 category, and a suggested severity. Never silently fix or ignore it.
- Propose recording it as a `SEC-NNN` row in `{repo}/documentation/security/risks.md` (confirm
  with the user first). Don't put it in `BFF_API_REQUIREMENTS.md` gap tables — link the `SEC-NNN`.
- Keep it separate from the functional fix: log it and continue, unless it's actively
  exploitable/high severity — then recommend the user prioritise it ahead of the current bug.

### Keeping fixes secure

- **Never make a bug go away by weakening a control** — e.g. loosening CORS, dropping
  `@require_auth`, widening `safe_identifier()`/validation, disabling read-only SQL checks,
  relaxing cookie flags, or adding `v-html` for unsanitised content. If the only apparent fix
  weakens a control, stop and discuss alternatives with the user.
- Validate at the boundary, parameterise queries, encode/sanitise output, fail closed, and keep
  least privilege. Don't log secrets or full request bodies containing user data while debugging —
  and remove any temporary diagnostic logging before finishing.
- Infra-side security fixes (IAM, Cloud Armor, networking, secrets) go through `infra/` Terraform +
  GHA, never ad-hoc CLI changes.
- If the fix resolves an existing `risks.md` row, update its status in the **same commit**.

---

## Step 1 — Get a repro from the user

Use the details already provided. For each reported bug, collect the following where relevant;
ask only for missing information that blocks diagnosis or verification:

1. **What view and action** — which page/tab, what question or operation was attempted
2. **Expected vs. actual** — what should have happened vs. what actually happened on screen
3. **Browser console errors** — any red errors in DevTools Console, pasted verbatim
4. **Network tab details** for the failing request(s): request URL, method, status code, request
   payload, and response body (especially the `error`/`kind` fields the API uses — see
   `api/app/shared/errors.py` error contract: `input | auth | forbidden | not_found | conflict |
   upstream | timeout`)
5. Whether the dev stack is actually running — the three relevant background tasks are
   `api: dev (Flask :5000)`, `ui: dev (Vite :5173)`, and `iap: tunnel intelligence (:8001)`
   (catalog/OM calls go through the API's own OM client config, not a tunnel)

Don't invent missing details. Use available local logs, tasks, code, and focused non-browser
checks before asking the user for evidence they need to collect manually. Never request secrets
or unredacted cookies/tokens.

---

## Step 2 — Localise the fault to a layer

Form a hypothesis before reading code. The BFF chain for a "real" UI interaction is:

```
ui (Vue view) → ui/src/api/client.js → api route → api service → upstream
                                                                    ├─ intelligence (chat/agent, SSE)
                                                                    ├─ catalog_client.py → OpenMetadata
                                                                    └─ query_service.py → DuckDB/GCS
```

Narrow down using the evidence from Step 1:

- **Request never left the browser / wrong URL / wrong payload shape** → `ui` bug (`client.js` or
  the calling view). Compare the actual request against `api/openapi.json` and
  `api/app/schemas/__init__.py` for the real contract — don't trust `client.js` comments.
- **Request reached the API but got a 4xx before touching any upstream** → `api` route/schema
  validation bug, or an auth/session problem (check cookie present, `@require_auth` tier).
- **Request reached the API, returned `kind: upstream` or `502`/`504`** → an upstream dependency
  problem. Check which one from the route (`catalog.py` → OM, `query.py` → DuckDB/GCS,
  `chat.py`/`agent.py` → intelligence) and go to Step 2a–c.
- **Request succeeded (200) but the UI rendered it wrong or not at all** → `ui` bug in the view
  component — read the actual response shape returned and compare to what the component expects.
- **SSE response (`/chat`, `/agent/send`) looked fine in Network tab but UI never updated** → check
  the EventSource/fetch-stream parsing in the relevant view/client code — a malformed event or an
  unhandled event type is a common culprit.

### 2a. If it points at `intelligence`

- The local API talks to intelligence via `INTELLIGENCE_URL=http://localhost:8001`, which is the
  **IAP tunnel to the deployed test VM** — not a local process. If that task isn't running, every
  intelligence-backed call will fail with a connection error, which `api` will surface as
  `kind: upstream`.
- If intelligence code was recently changed, remember it requires a GHA deploy before the tunnel
  will reflect it — check `intelligence/documentation/continuous-improvement.md` and recent
  `intelligence` deploy workflow runs if behaviour doesn't match the current source.

### 2b. If it points at `catalog` (OpenMetadata)

- `api/app/services/catalog_client.py` talks to OM directly (no tunnel). A `502 kind:upstream`
  here is frequently a stale/expired OM token rather than a real OM outage — check for a 401 from
  OM in the api server's terminal log before assuming OM itself is broken.
- Confirm the dataset/table being queried actually has a `curation/datasets/<dataset>/table.yaml`
  in the `catalog` repo — without one it's invisible to OM regardless of what's in the Iceberg
  catalog (see `catalog/.github/copilot-instructions.md`).

### 2c. If it points at the DuckDB query service

- `api/app/services/query_service.py` queries GCS parquet directly via DuckDB — check GCS HMAC
  creds are loaded (Secret Manager) and the GCS path/table name resolved by the service matches
  what's actually in the bucket.
- SQL write attempts should be rejected at the route level — if a read is failing, check
  `safe_identifier()`/validation isn't over-rejecting a legitimate identifier.

---

## Step 3 — Capture confirmed bugs in the register

As soon as evidence confirms a functional defect, record it; don't wait until the fix is done.
An unproven report remains a hypothesis, not a confirmed bug or invented root cause.

- Read `{repo}/documentation/bugs/README.md` for each affected repo and follow its template,
  status vocabulary, and index conventions. Check existing reports first; update a matching
  report instead of creating a duplicate. If the register is missing, create it using the
  org-wide bug-register convention and the UI register as a template.
- Create `BUG-NNNN-kebab-case-title.md` using the next unused, monotonically increasing ID in
  that repo, and add its index row. Inspect existing filenames as well as the index; never reuse
  an ID or delete a report.
- Include the reported date, severity, impact, expected vs. actual behaviour, repro steps,
  redacted evidence, affected files, and known root cause. Label unknowns as pending diagnosis.
  Start with `Status: open`; populate Fix and Verification as work progresses.
- For a cross-repo defect, create a companion report in each affected repo and cross-link under
  Related using full `https://github.com/big-lake/{repo}/blob/main/...` URLs. A UI symptom alone
  doesn't require a UI report if only another repo needs a change.
- Security weaknesses belong in `documentation/security/risks.md`, following the confirmation
  rule above. A functional bug with security implications gets both records, cross-linked.
- Keep register writes and ID allocation with the coordinating agent so concurrent subagents
  cannot allocate duplicate IDs or overwrite the index.

---

## Step 4 — Confirm root cause and make an executable fix plan

State plainly which layer and file(s) are at fault and why, citing the actual evidence from
Step 1 (status code, response body, log line) — not speculation. If you're not sure, say so and
propose the smallest next diagnostic step (e.g. "add a log line here and reproduce again") rather
than guessing at a fix.

For each confirmed bug, identify one falsifiable local hypothesis and the cheapest focused check
that could disprove it. Once the owning code path and check are clear, stop exploring and make
the smallest testable fix.

Present a short ordered plan keyed by repo and `BUG-NNNN`. For each item include:

- Root cause and evidence, files to change, intended behaviour, and why the change addresses it.
- Dependencies and order, plus any independent work suitable for a subagent.
- Focused validation command/check and the manual UI retest needed from the user.
- Security impact ("none" is valid), exposures spotted, and any approval or environment blocker.

Then **execute the plan without an extra approval round for ordinary local edits**. Ask only
when a material ambiguity blocks a safe fix or an action needs explicit approval. Do not push,
merge, deploy, run destructive operations, or weaken security controls without the required
confirmation. Never edit the cross-repo `TODO.md` without confirmation.

---

## Step 5 — Execute fixes, delegating where useful

Use subagents when independent bugs or clearly bounded implementation tasks benefit from
separate context. Keep a simple, local fix in the coordinating agent when delegation adds no
value.

- Give each subagent the bug ID/report, repro evidence, root-cause hypothesis, relevant repo
  instructions, exact file ownership, expected change, validation command, and approval limits.
- Use a read-only agent such as `Explore` only for investigation. Delegate implementation only
  to an agent that can edit and validate; if none is available, implement in the coordinator.
- Parallelise only tasks with disjoint file ownership and no contract dependencies. Execute API
  contract changes before dependent UI changes. Keep shared files and register/index updates
  coordinator-owned; all agents share the working tree and must preserve others' changes.
- Require a return summary of root cause, changed files, checks run/results, remaining risks,
  and blockers. Review returned changes and run any necessary integration check yourself.
- Subagents must obey the no-browser-automation rule and must not commit, push, merge, deploy,
  or perform destructive actions. Do not create a commit unless the user requests one.

- Fix only the bug identified — do not refactor unrelated code while you're in there.
- Re-check the diff against the Security lens "Keeping fixes secure" rules before handing back.
- Follow the relevant repo's existing conventions (`api/.github/copilot-instructions.md`,
  `ui/.github/copilot-instructions.md`) for where the fix belongs (route vs. service vs. view vs.
  `client.js`).
- If the fix changes the API's request/response contract, update the schema, the route's YAML
  docstring, refresh `openapi.json` (`api: update OpenAPI baseline` task), and add/update a Newman
  request per `api/.github/instructions/postman-ci.instructions.md`.
- If the bug reveals a real gap in `api/BFF_API_REQUIREMENTS.md` (e.g. a status the UI never
  expected), update it.
- If the root cause was a genuine design flaw worth remembering (not just a typo), add a short
  gotcha note to the relevant README per `etl/.github/instructions/documentation.instructions.md`
  (applies org-wide) — don't write an ADR for a plain bug fix, only for an actual decision.
- Immediately after each substantive edit, run the cheapest focused executable validation
  before further exploration or edits. Repair local failures and rerun the same check. Use
  existing checks and repo testing conventions; do not add test infrastructure unnecessarily.
- Record implemented changes and actual validation results in the bug report, including failed
  checks and remaining blockers. Keep the report and index `open` while manual verification or
  a required deployment is pending; don't invent a pending status outside the repo vocabulary.
- Continue other independent planned fixes while waiting for a manual retest or deployment
  approval. Explicitly report blocked work rather than silently skipping it.

---

## Step 6 — Manual retest and register close-out

For each implemented fix, tell the user exactly what to redo in the UI (same view/action from
Step 1), what result to expect, and whether the running stack contains the changed code. A local
source edit is not proof that the deployed intelligence service is fixed. Request new redacted
network/console evidence if the bug persists and resume diagnosis against the same report.

Do not mark a bug `fixed` until the focused checks pass and the user confirms the manual retest.
Then update both the report and its index, with the verification date and evidence. If a commit
is requested, record its real SHA according to the repo convention; never fabricate a SHA or
commit merely to fill the field. Until committed, state that the verified fix is local/uncommitted.
User-deferred bugs stay `open` with the reason and next step; use `wontfix` only for an explicit
decision not to fix.

Finish with a compact summary keyed by repo and bug ID: changes made, checks and outcomes,
manual retests awaiting the user, and any deployment/approval blockers. Keep the fix plan current
as items complete; do not claim all bugs resolved while any required verification remains.
