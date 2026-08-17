---
description: "Scan the intelligence service codebase for gaps and opportunities across the continuous-improvement categories, and append findings to the opportunity register (evaluation/results/OPPORTUNITIES.md). Read-only — does not rank, choose, or implement anything. Run this periodically/on demand; then use /intelligence-improve to triage and act on the register."
mode: agent
---

# intelligence — codebase scan (opportunity discovery)

Use this prompt to run **one scan pass** over the `intelligence` codebase per the categories and
checklist defined in [`intelligence/documentation/continuous-improvement.md`](../../intelligence/documentation/continuous-improvement.md).
This prompt is deliberately **separate from `/intelligence-improve`** so scanning (expensive,
code-reading work) and triage (cheap, register-reading work) can run on different cadences —
don't re-scan the whole codebase every time you just want to pick something to work on.

Output is new rows appended to the opportunity register. This prompt does **not** rank, choose,
present a shortlist for action, or write any implementation code.

Optional focus for this pass (leave blank to scan broadly):
${input:focus:Category (1–11) or symptom to concentrate on — e.g. "orchestration", "latency", "NL→SQL", "safety", or blank}

---

## Phase 1 — Load the framework and existing register

Read [`intelligence/documentation/continuous-improvement.md`](../../intelligence/documentation/continuous-improvement.md)
first. Internalise:
- The eleven review categories in §3 — grouped into capability stages (1–5), the substrate (6),
  and cross-cutting concerns (7–11) — plus the prompts-as-a-cross-cutting-lens note (§3.1) and
  the review checklist in §4.
- The evidence + axis expectations and the register format in §6 (`evaluation/results/OPPORTUNITIES.md`).

Read the existing [`OPPORTUNITIES.md`](../../intelligence/evaluation/results/OPPORTUNITIES.md)
register if it exists — this holds only the currently **active** (`open`/`chosen`/`in-progress`)
rows. Note every row already logged so you don't re-add a duplicate. Also check
[`OPPORTUNITIES-ARCHIVE.md`](../../intelligence/evaluation/results/OPPORTUNITIES-ARCHIVE.md) for
the highest `OPP-NNN` ID used there too — numbering is sequential across both files, so the next
new ID must be higher than the max of *either* file, not just the active one. If a previously
logged `open` row has since been fixed, move it to the archive with a `done` status and a
reference to the fix rather than leaving it (or a duplicate) in the active file.

Also skim, for grounding: [`DESIGN.md`](../../intelligence/DESIGN.md) (phasing + ADR-0008) and
[`README.md`](../../intelligence/README.md).

Check repo memory at `/memories/repo/intelligence-improve-state.md` (via the memory tool) for the
"Scan coverage — last pass per category" table — use it (over guessing from register row dates)
to decide which categories are most overdue for this pass's rotation (§2).

---

## Phase 2 — Scan the codebase

Review the actual code and config against the §4 checklist. If a focus was given, concentrate
there; otherwise rotate across categories (§2 "Rotation") so coverage stays balanced over time —
use the memory-tracked "Scan coverage" table from Phase 1 (not just register dates) to prioritise
categories that haven't been reviewed recently.

### 2a. Fan out across categories in parallel

A full multi-category pass is read-only and each category mostly touches different files, so
**dispatch one `explore_subagent` call per category (or small group of related categories) in
parallel** rather than scanning serially — this is the main lever for keeping a broad scan pass
fast. Guidance:

- **Group, don't over-split.** One subagent per category is fine for the five capability-stage
  categories (1–5, mostly disjoint files); categories that already share files or a review lens
  can go to one subagent together — e.g. 8+9 (both touch `orchestrator.py`/`tools/` failure and
  safety paths) or 10+11 (both are response-metadata/eval-ergonomics reviews). Use judgement, not
  a fixed rule — the goal is fewer round-trips without muddying what evidence belongs to which
  category.
- **Give each subagent the same brief**: the relevant §4 checklist questions for its category(ies)
  (copy the bullet text, don't just say "category 3"), the file/module pointers listed below, and
  an explicit instruction to **return findings only** (evidence, axis, impact/effort/risk/
  confidence, target repo) — not to rank, dedupe against the register, or write anything. Also
  tell it which rows are already logged for its category (from Phase 1) so it doesn't resurface
  them.
- **If focus was narrowed** (the `${input:focus}` above) to one category or symptom, skip fan-out
  entirely — a single-category pass doesn't need parallelism, just scan it directly.
- **Cross-repo categories (6)** still only inspect what's reachable from this repo (OM/manifest
  client code, DESIGN.md references) — the subagent shouldn't try to read `catalog`/`knowledge`
  source, just note the cross-repo lever per the existing pattern.

Look at, as relevant to the categories in scope (hand these pointers to whichever subagent covers
each category, or use them directly for a narrow-focus/serial pass):

- **1 Query understanding & routing** — `app/services/router.py`, `query_expander.py`, `orchestrator.py`
- **2 Retrieval & ranking** — `app/services/retrieval.py`, `reranker*.py`, `manifest.py`
- **3 Orchestration & tool use** — `app/services/orchestrator.py`, `router.py`, `tools/`
- **4 Generation & grounding** — `app/services/synthesizer.py`, streaming/event ordering
- **5 Data & analytics (NL→SQL)** — `app/services/tools/data_discovery.py`, `tools/data_query.py`, `om_client.py`
- **6 Knowledge & lakehouse curation** (cross-repo) — thin OpenMetadata descriptions or missing
  `knowledge` tiers that are the real lever, even though the fix lands in `knowledge` / `catalog`
- **7 Runtime, latency & cost** — `app/config.py`, `infra/modules/intelligence/startup.sh` (model tiers, VM, backends, flags)
- **8 Safety, security & governance** — injection surface in `synthesizer.py` (untrusted doc text) and
  `tools/data_query.py` (NL→SQL read-only + row-cap boundary), data sovereignty (ADR-0008), out-of-corpus refusal
- **9 Reliability & resilience** — upstream failure/timeout/retry/degrade paths in `orchestrator.py`,
  `tools/`, clients; boot-time manifest parity checks
- **10 Observability & instrumentation** — response models / logging: trace IDs, per-stage timing, token/cost, attribution metadata
- **11 Evaluation, feedback & production learning** — `evaluation/` test-set coverage & harness ergonomics; any user-feedback / trace-sampling signal

Apply the **prompts lens** (§3.1) within stages 1/3/4/5 rather than treating prompts as a separate scan
(fold this into each relevant subagent's brief).

For each finding, capture **evidence** (file + line, ADR, known gotcha, or symptom) and **which
axis** it moves (quality / speed / cost / risk / diagnosability). Score it on the §5 dimensions
(impact / effort / risk / confidence) and note the repo it would land in. Do not propose detailed
fixes or implement anything — that's `/intelligence-improve`'s job.

### 2b. Merge and dedupe subagent results

Once all subagents report back, merge their findings **before** touching the register:

- Skim across subagent reports for near-duplicate findings (two subagents occasionally surface
  the same underlying gap from different angles, e.g. a category-3 orchestration gap that's also
  visible as a category-9 resilience gap) — merge into one row tagged with the more specific
  category, noting the other axis in its Notes rather than filing two rows for one root cause.
  This is the same principle as the register's existing "don't duplicate" rule, just applied
  before rows are even written instead of after.
- Assign sequential `OPP-NNN` IDs **only at this merge step** (never let a subagent assign IDs
  itself — running them in parallel makes collisions likely if each one guesses the "next" ID
  independently).

---

## Phase 3 — Append to the register

Append each new finding as a row to `evaluation/results/OPPORTUNITIES.md` (create it using the §6
table format if it doesn't exist yet), status `open`. Assign the next sequential `OPP-NNN` ID
(checking both `OPPORTUNITIES.md` and `OPPORTUNITIES-ARCHIVE.md` for the current max — see Phase 1).

Do not remove or re-rank other still-open rows — this pass only adds. If you identified that a
previously `open` row is now stale, obsolete, or already fixed, move its entire row to
`OPPORTUNITIES-ARCHIVE.md` with the corrected `done`/`wontfix` status and a one-line note, and
delete it from the active file — don't leave a closed row behind in `OPPORTUNITIES.md`, and don't
add a duplicate.

Report a short summary: how many new opportunities were added, which categories they fall under,
and which categories still haven't been scanned recently (for next time).

Update `/memories/repo/intelligence-improve-state.md`'s "Scan coverage" table with today's date
for every category actually covered in this pass, so the next scan's rotation decision (and the
next triage session's context) doesn't have to be reconstructed from register dates.

---

## Guardrails

- **Read-only with respect to `intelligence` behaviour** — this prompt only reads code and writes
  to the register markdown file. No implementation changes, no eval runs.
- **Subagents report, they don't write** — fanned-out `explore_subagent` calls (§2a) return
  findings only; only the main pass (after §2b's merge) writes to `OPPORTUNITIES.md`, and only it
  assigns `OPP-NNN` IDs. Never let a subagent touch the register file directly.
- **Every finding needs evidence** — a path, ADR, gotcha, or symptom. No speculative items.
- **No ranking or shortlisting here** — that happens in `/intelligence-improve`, working from the
  register.
- **Route cross-repo findings correctly** — note the target repo (`knowledge` / `catalog` /
  `infra`) even though this scan only reads `intelligence` code.
