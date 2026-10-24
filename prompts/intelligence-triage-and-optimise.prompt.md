---
description: "Triage and optimise one sub-optimal intelligence response end-to-end: take the query, the actual response, and the ideal response; reproduce it against the deployed service with the e2e harness; find the first pipeline stage where the ideal answer became impossible; record OPP/BUG entries; fix with parallel subagents; deploy a branch to test; prove the improvement (and no regressions) with before/after harness runs; document, merge, and redeploy."
mode: agent
---

# /intelligence-triage-and-optimise — From one bad answer to a verified, regression-tested fix

Use this when an `intelligence` answer (chat or agent surface) **worked but wasn't good enough** —
wrong route, missed the right document, half-answered, hallucinated, refused when it shouldn't,
wrong SQL, wrong format. There may be no error at all.

| Situation | Use instead |
|---|---|
| Request fails / UI broken / wrong status code | `/ui-e2e-bug-triage` |
| Want to find improvement opportunities across the codebase | `/intelligence-scan` |
| Want to work an existing `OPP-NNN` from the register | `/intelligence-improve` |

The backbone of this flow is the **end-to-end regression harness**
([`intelligence/evaluation/e2e_eval.py`](../../intelligence/evaluation/e2e_eval.py),
[ADR-0024](../../intelligence/documentation/adr/0024-e2e-regression-harness-over-deployed-http-surface.md)).
Every triage leaves behind a permanent `E2E-NNN` regression case and committed before/after results —
the fix isn't done until the harness says so.

---

## Step 0 — Approval, intake, and context

### 0a. Session-wide approval

This run is long and unattended (harness runs, subagents, commits, a branch push, a test deploy,
a merge to `main`, a second deploy). Ask the user to approve "Allow all tools/commands for this
session" and confirm that approval covers **pushing a `triage/*` branch, merging to `main`,
pushing `main`, and deploying `intelligence` to `test`**. Wait for the go-ahead. This never
covers deploying to `prod`, force-pushing, or touching unrelated uncommitted changes.

### 0b. Intake — required

If the user hasn't supplied all three, ask for what's missing before doing anything else:

1. **Query** — verbatim, plus any **earlier turns** in the conversation (the chat surface rewrites
   follow-ups against history — ADR-0014 — so a follow-up can behave completely differently from
   the same text asked cold).
2. **Actual response** — verbatim, including citations/sources shown and any SQL/table output.
3. **Ideal response** — what it should have said, and ideally *why* (which document, page, or
   dataset holds the answer). Push back gently if it's vague ("better") — the ideal becomes the
   judge's reference and must name the key facts.

### 0c. Intake — optional (pull it yourself where possible, ask only if you can't)

- **Surface** — chat tab (`/chat/stream`) or notebook agent (`/agent/send`); for agent, whether
  it was a data cell, knowledge cell, or free-form terminal turn (`cell_type` null).
- **`trace_id`** — first SSE `meta` frame, or the `trace_id` column on `api`'s `messages` table
  (local DB: `C:\opt\biglake\api\app.db`).
- **Environment and rough time** — test vs prod; prod reports are reproduced on test unless the
  user explicitly says otherwise.

### 0d. Load context

Read, in parallel:

- [`continuous-improvement.md`](../../intelligence/documentation/continuous-improvement.md) §3
  (categories), §3.1 (prompts lens), §7 (opportunity → change).
- [`OPPORTUNITIES.md`](../../intelligence/evaluation/results/OPPORTUNITIES.md) and the max
  `OPP-NNN` in [`OPPORTUNITIES-ARCHIVE.md`](../../intelligence/evaluation/results/OPPORTUNITIES-ARCHIVE.md)
  — the symptom may already be a known row; update it rather than duplicate.
- [`e2e_cases.json`](../../intelligence/evaluation/e2e_cases.json) — next free `E2E-NNN`, and
  whether a similar case already exists.
- `intelligence/README.md` → "Known Gotchas", and the ADR index
  (`intelligence/documentation/adr/README.md`).
- Repo memory `/memories/repo/intelligence-improve-state.md` (loose ends, in-flight items).

---

## Step 1 — Preflight

1. **Tunnel health** — `Invoke-WebRequest http://localhost:8001/health -TimeoutSec 5`. A
   listening port isn't proof (stale IAP tunnels keep the port open). If unhealthy, ask the user
   to restart the `iap: tunnel intelligence (:8001)` task and stop.
2. **Local api (optional)** — `http://localhost:5000/health`. Needed only so the harness can
   execute generated SQL for data cases; without it, data cases are judged on SQL alone.
3. **Harness venv** — `intelligence/.venv` exists with `evaluation/requirements-e2e.txt`
   installed; if not, create it (see the intelligence README "End-to-end regression harness").
4. **What's actually deployed** — from `intelligence/`:
   `gh run list --workflow=deploy.yml -L 3 --json headSha,headBranch,conclusion,createdAt`.
   Compare the last successful `test` deploy's SHA to `origin/main`. If they differ, the user saw
   behaviour from older/other code — say so, and read source at the **deployed** SHA
   (`git show <sha>:path`) when localising, not your working tree.
5. **Working tree** — `git status` in `intelligence` (and any other repo you may touch). Note
   pre-existing changes; never stage or revert them.

---

## Step 2 — Classify the gap

Before reading code, classify the symptom(s) from the actual-vs-ideal diff. This picks the
starting hypothesis and the framework category used for the register row:

| Symptom | Category | First place to look |
|---|---|---|
| Doc question went to SQL, or vice versa; wrong class | 1, 3 | `router.py`, `orchestrator.py` routing |
| Follow-up misread (coreference / "what about last year") | 1 | `query_rewriter.py` (chat only — agent has no history) |
| Right doc exists but wasn't retrieved / ranked low | 2 | `retrieval.py`, `query_expander.py`, `reranker*.py`, manifest tier selection |
| Retrieved but ignored, misread, or mis-cited | 4 | `synthesizer.py` prompts, context assembly |
| Hallucinated / unsupported claim | 4, 8 | `synthesizer.py`, faithfulness verdict |
| Half-answered multi-part / multi-hop question | 1, 3 | `decomposer.py`, `_combined_path`, class→pool widening |
| Refused when it shouldn't (or answered when it should refuse) | 4, 8 | synthesis rule 3, empty-retrieval guard, `data_query` unanswerable |
| Wrong dataset / wrong SQL / wrong filter values | 5 | `tools/data_discovery.py`, `tools/data_query.py`, `om_client.py` |
| Right content, wrong format / length / tone | 4 | synthesis prompts (prompts lens §3.1) |
| Answer simply isn't in the corpus / catalog | 6 | `knowledge` / `catalog` — cross-repo |
| Slow or expensive | 7 | trace `duration_ms`, model tier, pool sizes |

State the classification and one-line hypothesis to the user, then carry on (no need to wait).

---

## Step 3 — Capture the case and a baseline

### 3a. Write the regression case

Append an `E2E-NNN` case to `intelligence/evaluation/e2e_cases.json` (schema in the
`e2e_eval.py` module docstring):

- `question`, `history`, `surface`, `agent` options exactly as the user ran it.
- `ideal_answer` — the user's ideal, verbatim or lightly tidied; never weaken it.
- `expect` — encode the **intent of the ideal**, as tightly as the evidence allows:
  `route_class` if the correct class is unambiguous; `outcome`; `must_cite` (document title /
  label / collection id, or dataset FQN); `must_include` for the 1–3 key facts/figures;
  `must_not_include` for the specific wrong claim if there was one. Prefer stable substrings
  (a figure, a document name) over phrasing that a correct answer could legitimately word differently.
- **Never encode the current bad behaviour as expected**, and don't over-specify — an `expect`
  that only one exact wording could satisfy will make the case flaky forever.
- `links.opp` / `links.bug` filled in at Step 6; `note` = one line on why the case exists.

### 3b. Reproduce

From `intelligence/`:

```powershell
.\.venv\Scripts\python.exe -m evaluation.e2e_eval --case E2E-NNN --label E2E-NNN-before
```

(3 repeats by default.) Then read the result JSON's `observation` blocks — `route_class`,
`trace_steps`, `citations`, `answer`/`sql`/`sql_result`, `faithfulness_*`, `judge.missing` /
`judge.incorrect`, `latency_ms`.

- **Reproduces (fail/flaky)** → continue.
- **Passes 3/3** → it doesn't reproduce. Check, in order: history not supplied; different
  surface/`cell_type`; deployed SHA changed since the user saw it; prod vs test corpus difference;
  `expect`/ideal too loose. Report to the user and ask how to proceed rather than "fixing"
  something you can't observe.
- **Harness itself errors** (auth, tunnel, judge failure) → fix the environment, not the case.

### 3c. Baseline sweep

Run the full case set once so Step 9 can detect regressions:

```powershell
.\.venv\Scripts\python.exe -m evaluation.e2e_eval --label E2E-NNN-baseline
```

Skip only if a sweep against the **same deployed SHA** already exists in
`evaluation/results/e2e/`. If the set is larger than `--max-turns` allows, sweep the classes the
fix could plausibly touch (`--class`) and say so.

---

## Step 4 — Is the ideal answer reachable at all?

Before blaming the pipeline, prove the ideal is grounded in what the service can see. Tuning
prompts to produce facts the corpus doesn't contain is the most expensive mistake in this flow.

- **Knowledge answers** — probe `POST /retrieve` over the tunnel with (a) the user's query, (b)
  the key facts of the ideal phrased as a query, (c) the document/page the user named, at a
  generous `top_k` (20–50). Write a throwaway script in `$env:TEMP` that imports
  `_load_config`/`_TokenCache` from `evaluation.e2e_eval` for auth (don't add it to the repo).
  Also try `"tool": "search_propositions"`. If the right chunk never appears at any depth, check
  whether the document is in the manifest / indexed at all (`knowledge` source configs,
  `catalog/curation/knowledge/`).
- **Data answers** — confirm the dataset exists and is curated (`catalog/curation/datasets/<dataset>/`
  `table.yaml` + `cols.yaml`), that column descriptions/units make the question answerable, and,
  via the local api's `POST /query`, that a hand-written SQL query actually returns the ideal's
  numbers.

Outcome:
- **Not reachable** → category 6 (cross-repo). Record it (Step 6), propose the
  `knowledge`/`catalog` fix (e.g. `/curate-dataset`, a source config addition), and **stop the
  intelligence-fix path** for this root cause — escalate, don't fake it in prompts.
- **Ideal is wrong** (contradicts the source) → tell the user with evidence; agree a corrected
  ideal before continuing.
- **Reachable** → the pipeline lost it somewhere. Continue.

---

## Step 5 — Localise: the first stage where the ideal became impossible

Walk the pipeline in order and stop at the **first** stage whose output no longer contained what
the ideal needed. Downstream symptoms of an upstream loss are not separate root causes.

```
chat:  s0 rewrite → s1 classify → expand → retrieve (dense+BM25+RRF) → cap → rerank → parent
       → s3 synthesis (model tier, prompt, context) → faithfulness → citations
agent: s1 classify (or forced cell_type) → [clarify?] → knowledge_path | data_path | combined_path
       data: data_discovery → (sources.request) → data_query (schema block → SQL → sqlglot) → BFF exec
```

### 5a. Evidence sources (gather them yourself)

- **Harness observations** from 3b — the primary evidence.
- **Counterfactual probes** over the tunnel, each isolating one stage: `/retrieve` with the
  rewritten vs raw query; the agent with `cell_type` forced to `knowledge`/`data`; the question
  re-phrased to the class you believe is correct; the dataset pre-selected via a temp case with
  `auto_confirm_sources` and a hand-picked selection. Keep probes in `$env:TEMP` throwaway case
  files (`--cases $env:TEMP\probe.json --output $env:TEMP\probe-out.json`) so they don't pollute
  the committed case set or results.
- **Service logs by `trace_id`** (read-only) — every log line is stamped with it (ADR-0012), incl.
  per-call token usage (`llm_usage.py`) and data_query attempts:
  `gcloud compute ssh biglake-intelligence-test --tunnel-through-iap --zone=australia-southeast2-b --project=big-lake-test-490405 --command "sudo journalctl -u biglake-intelligence.service --since '-2 hours' --no-pager | grep <trace_id>"`.
  Read-only only — never restart, edit, or patch anything on the VM. If SSH is denied, say so and
  continue without logs.
- **Source at the deployed SHA** (Step 1.4).

### 5b. Parallel read-only investigation

When more than one stage is plausible, dispatch one `Explore` subagent per suspected stage **in
parallel** (read-only; tell each to return findings only). Each brief must include: the case's
question/actual/ideal, the relevant harness observation excerpt, the exact files for that stage
(from the Step 2 table), the deployed SHA to read at, and the question to answer ("does this
stage's output still contain X? cite file:line"). Don't fan out when the evidence already points
at one stage.

### 5c. State the root cause(s)

For each: stage, file:line, the specific logic/prompt/config at fault, and the evidence
(observation field, probe result, log line). If not confident, name the smallest next probe
instead of guessing. There may be more than one independent root cause (e.g. discovery ranked the
right dataset 5th **and** the SQL step mishandles declines) — list each separately.

Also note anything spotted along the way that's a security issue (prompt injection via
ingested text, SQL boundary, data leakage) — it becomes a `SEC-NNN`, never silently fixed or ignored.

---

## Step 6 — Record findings

Before fixing anything, write every confirmed root cause down:

- **Quality/optimisation gap** (behaviour works as designed but the design produces a worse
  answer) → an `OPP-NNN` row in `OPPORTUNITIES.md` per the register format (§6): category, axis,
  impact/effort/risk/confidence, repo, status `in-progress` if fixing now or `open` if escalated.
  Notes: `source: /intelligence-triage-and-optimise`, the `E2E-NNN` id, and the evidence.
- **Defect** (code doesn't do what it, its docs, or its ADR say — e.g. a dead code path, a
  parser rejecting valid output) → a `BUG-NNNN` file in `intelligence/documentation/bugs/`. If
  that folder doesn't exist yet, create it with a `README.md` mirroring
  `api/documentation/bugs/README.md` (rules, template, index) — the org convention requires it.
- **Cross-repo root cause** (`knowledge`, `catalog`, `api`, `ui`) → record it in the repo where
  the root cause lives (OPP row with that repo, or that repo's `BUG`/curation task), cross-linked
  with full `https://github.com/big-lake/...` URLs.
- **Security** → `SEC-NNN` row in that repo's `documentation/security/risks.md`.
- Fill the case's `links` with the IDs.

---

## Step 7 — Fix plan

For each root cause, decide:

- **The single lever** (§7): one prompt, one parameter, one code path — behind an existing
  flag/config where possible, so it's reversible.
- **Generalisation check (non-negotiable).** The fix must improve the *class* of question, not
  this question. Reject any fix that keys on this query's words, adds this case (or a paraphrase)
  as a few-shot example, hardcodes a document/dataset name, or special-cases a figure. Name 2+
  other existing cases (or describe question types) that share the root cause and should also
  improve, and which cases could plausibly regress.
- **ADR check.** Read the ADR(s) governing the stage. ADRs are reasoning, not rules — if the fix
  conflicts with one, say so explicitly, state which Assumption no longer holds, and plan a new
  ADR that amends/supersedes it. A non-obvious trade-off (quality vs cost/latency, new LLM call,
  model-tier change) gets a new ADR even without a conflict.
- **Cost/latency effect** — any new LLM call, larger pool, or Pro-tier routing is a trade; state it.
- **Auto-fix vs escalate.** Auto-fix when the root cause is clear and the change is contained to
  `intelligence` code/prompts/config. **Escalate** (record, propose, don't implement) when it
  needs: a product decision between real alternatives, an `infra/` change (env vars in
  `startup.sh`, model/VM changes), a cross-repo contract change (`api`/`ui` event shapes), content
  curation in `knowledge`/`catalog`, or a category 8 (safety) change.
- **Batches.** Root causes with disjoint file sets → parallel subagents. Shared files → same
  subagent, sequentially. **Never two subagents on one file.**

Post a short table (`root cause → OPP/BUG → files → batch → auto-fix/escalate → expected effect`)
and proceed without waiting unless something is escalated or the plan touches more than ~8 files.

---

## Step 8 — Implement with subagents

Create the branch first: `git -C <intelligence> switch -c triage/E2E-NNN` (from up-to-date `main`).

Launch one `runSubagent` per batch, **in parallel** where batches are disjoint. Each prompt must
be self-contained:

- Absolute repo path, the `OPP`/`BUG` file/row to read, the root cause with evidence, and the
  **exact files it may edit** (nothing else).
- The generalisation rules from Step 7, verbatim.
- Conventions: services are stateless functions; prompts stay centralised in their module;
  every new knob is documented at its definition (valid values, default, effect); in-code
  comments explain *why*, one short line where possible; untrusted text (chunks, OM descriptions,
  user input) stays delimited as data in any prompt it touches.
- Run `get_errors` on every changed file. **Do not** commit, push, deploy, edit docs/registers,
  or run the harness. Return a per-file before/after summary.

When each returns: read the changed regions yourself, run `get_errors`, and check `git status`
shows nothing outside the planned files. Then commit on the branch
(`fix(<scope>): ... (E2E-NNN, OPP-NNN/BUG-NNNN)`, matching `git log --oneline -10` style).

---

## Step 9 — Deploy the branch to test and verify

1. Push the branch: `git push -u origin triage/E2E-NNN`.
2. Deploy it: `gh workflow run deploy.yml --ref triage/E2E-NNN -f environment=test`, then watch
   `gh run watch $(gh run list --workflow=deploy.yml -L1 --json databaseId -q '.[0].databaseId')`.
   On failure: stop, report the run URL, don't retry blindly.
3. Confirm health over the tunnel (the deploy restarts the service — the tunnel may need a
   restart too).
4. Re-run the target and compare:
   ```powershell
   .\.venv\Scripts\python.exe -m evaluation.e2e_eval --case E2E-NNN --label E2E-NNN-after --compare evaluation/results/e2e/<E2E-NNN-before file>
   ```
5. Re-run the sweep and compare against the Step 3c baseline (`--label E2E-NNN-after-sweep --compare ...`).

**Acceptance:**
- Target case is `pass` (3/3). `flaky` is not done — it means the fix is partial or the case is
  over-specified; investigate which.
- No sweep case drops in status (`pass` → `flaky`/`fail`). A single-run sweep drop may be noise:
  re-run just that case with `--case X` (3 repeats) before calling it a regression.
- Mean judge scores flat or up; latency/cost change consistent with what Step 7 predicted.

**If it fails:** one more iteration (back to Step 5 with the new evidence — not a blind tweak).
If it fails again: revert that root cause's commits on the branch (new revert commits, never
history rewrites), leave its OPP/BUG `open` with what was tried, mark it escalated, and continue
with any root causes that did pass.

**Always** finish this step by making sure `test` won't be left running an abandoned branch:
either it proceeds to Step 11's merge + redeploy, or redeploy `main` now.

---

## Step 10 — Document

- **Register / bugs** — OPP row → `done` with outcome + before/after numbers, then move it
  verbatim to `OPPORTUNITIES-ARCHIVE.md`; BUG file → `Status: fixed`, Fix, Verification (cite
  the result files), index row. Escalated items stay `open` with the proposed approach.
- **ADR** — written/updated per Step 7; index updated; superseded ADRs get a Status line change
  only.
- **README** — `intelligence/README.md` behaviour/config/gotcha sections updated where
  human-observable behaviour changed (integrate into existing bullets; don't bolt on a changelog).
  Correct any README claim this triage proved false.
- **Case** — `e2e_cases.json` entry final (links filled, `note` accurate). It stays forever as
  a regression case — never delete or weaken it to make a run pass.
- **Results** — commit the `E2E-NNN-*` result JSONs from `evaluation/results/e2e/` (not
  `$env:TEMP` probe outputs).
- **Manual verification** — only if something genuinely can't be exercised by the harness (e.g.
  the UI rendering of a new event), add an `MVP-NNN` to `manual-verification-plan.md`.
- **Memory** — update `/memories/repo/intelligence-improve-state.md` "Loose ends" if anything
  stays open; add a user-memory note only for a genuinely recurring gotcha.
- Don't edit `.github/TODO.md` — flag drift in the report instead.

Commit docs on the branch: `docs(...): record E2E-NNN triage outcome`.

---

## Step 11 — Merge, push, redeploy main

1. `git switch main; git pull --ff-only; git merge --ff-only triage/E2E-NNN` (if not
   fast-forwardable, `git merge --no-ff` — never rebase shared history; if conflicts, stop and ask).
2. Record the fix SHA in the BUG `Fixed:` line / OPP notes with a follow-up
   `docs(bugs): record fix SHAs` commit — never amend.
3. `git push origin main`. If rejected (remote ahead), stop and report.
4. Redeploy `test` from `main` (`gh workflow run deploy.yml --ref main -f environment=test`),
   watch it, confirm `/health`. **Never deploy to prod** without the user's explicit confirmation.
5. Leave the `triage/E2E-NNN` remote branch in place and mention it — deleting branches needs the
   user's say-so.
6. Same flow per repo for any cross-repo commits that were in scope (each repo's own deploy
   workflow, `test` only).

---

## Step 12 — Report

Concise, evidence-first:

```
## Triage report — E2E-NNN

### The case
Question · surface · actual (1 line) · ideal (1 line)

### Root cause(s)
| # | Stage | File:line | Evidence | Recorded as |
|---|---|---|---|---|

### Fixes
| Root cause | Change (one lever) | ADR | Status |
|---|---|---|---|

### Before → after (harness)
| Case | Before | After | Δ judge | Δ latency |
|---|---|---|---|---|
| E2E-NNN | fail 0/3 (0.20) | pass 3/3 (0.90) | +0.70 | +0.4 s |
| sweep | 7 pass / 2 flaky / 1 fail | ... | | |

### Escalated / cross-repo
- ID — why it needs a human / other repo — proposed approach

### Security
- SEC-NNN rows added (or "none observed")

### Commits & deploys
| Repo | SHA | Message | Deployed to test |
|---|---|---|---|

### Left alone
- pre-existing uncommitted changes; the triage branch (not deleted)
```

---

## Guardrails

- **Evidence over intuition** — every root cause cites an observation, probe, or log line.
- **Generalise, never overfit** — no query-specific keywords, few-shots copied from the case,
  hardcoded names, or weakened `expect` blocks to make a run pass.
- **Never weaken a control to improve an answer** — read-only SQL validation, row caps, prompt
  delimiting of untrusted text, faithfulness signalling, auth.
- **Infra changes go through `infra/` Terraform + GHA**, never ad-hoc on the VM; logs are
  read-only access only.
- **Cost-aware** — stay inside the harness's `--max-turns`; prefer `/retrieve` probes and single
  `--repeats 1` probe runs while localising; save 3-repeat runs for before/after.
- **Shared `test` VM** — keep concurrency at the default; never leave `test` on a branch build at
  the end of the run.
