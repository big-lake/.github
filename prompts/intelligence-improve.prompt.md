---
description: "Triage the intelligence opportunity register: present a ranked shortlist of open opportunities for the user to choose from, deep-research and confirm the chosen one's current state, propose a concrete change plan, and (on confirmation) record it — including creating/updating ADRs. Does not scan the codebase for new opportunities; run /intelligence-scan for that."
mode: agent
---

# intelligence — continuous improvement triage

Use this prompt to triage the opportunity register produced by `/intelligence-scan` and take one
opportunity forward. This prompt is **cheap and register-driven** — it does not re-scan the whole
codebase (that's `/intelligence-scan`'s job, run on its own cadence). It reads what's already
been found, helps you choose, then does the deeper work only for the one thing you pick.

Optional filter for this pass (leave blank to consider the whole register):
${input:filter:Category (1–11), repo, or keyword to filter the register by — or blank for everything open}

---

## Phase 1 — Load the framework and register

Read [`intelligence/documentation/continuous-improvement.md`](../../intelligence/documentation/continuous-improvement.md)
for the category definitions (§3), ranking dimensions (§5), and the "from opportunity to change"
guidance (§7).

Read [`OPPORTUNITIES.md`](../../intelligence/evaluation/results/OPPORTUNITIES.md) — this is the
**active-only** register (open/chosen/in-progress rows; closed rows live separately in
[`OPPORTUNITIES-ARCHIVE.md`](../../intelligence/evaluation/results/OPPORTUNITIES-ARCHIVE.md) and
don't need to be read for a triage pass — only consult the archive if you need historical context
on a specific past fix, e.g. to check whether something related was already tried). If the active
file doesn't exist or has no `open` rows, **stop and tell the user to run `/intelligence-scan`
first** — do not scan the codebase yourself in this prompt.

Filter to `open` (and, if useful, `chosen`-but-stalled) rows, applying the optional filter input.

**Check for in-flight resume state first.** Before ranking from scratch, check repo memory at
`/memories/repo/intelligence-improve-state.md` (via the memory tool) for a "current item(s) in
flight" or "queued next picks" entry from a previous session. If one exists, **verify it against
the register** (the register is always the source of truth — the memory file can drift if the
register moved in another session) and, if still valid, offer to resume that item/queue directly
rather than re-running Phase 2 from a blank slate. If it's stale (item already closed, or no
longer matches the register), say so, update the memory file to clear it, and fall through to a
normal Phase 2 pass.

---

## Phase 2 — Rank and present opportunities

Turn the filtered register rows into a **ranked shortlist** (aim for 3–7, not an exhaustive
dump) using the existing impact/effort/risk/confidence/axis/repo fields already captured in the
register. For each, restate:

- **ID + one-line title**, category, axis
- Impact / Effort / Risk / Confidence, and the repo the change would land in
- The evidence already on record

Order by the §5 priority (high-impact, low-effort, low-risk, high-confidence first). Category
10/11 (observability/eval) gaps are force-multipliers and rank above direct tweaks; category 8
(safety/governance) findings are ranked on **risk**, not just impact.

**Tag fast-track-eligible rows.** Mark any shortlisted row `[fast-track]` if it meets **all** of:
S-effort, low risk, high confidence, not category 8, and mechanical/pattern-consistent enough
that no ADR is expected (per §7 — reuses an existing pattern/knob, no new design trade-off).
These are the rows where Phase 3's per-item rigour (3a research, verification method, docs) still
applies in full, but the *user confirmation checkpoint* can be shared across several of them at
once instead of one round-trip per row — see Phase 3. Anything M/L-effort, med/high-risk, or
category 8 is never fast-tracked, regardless of how small it looks; when in doubt, don't tag it.

Then **present the shortlist and ask the user which one (or, for `[fast-track]` rows, which
set) to take forward.** Stop here — do not implement anything, and do not read source files yet
beyond what the register already records.

**If the user wants to work through a large set (or "all") of open rows in one go**, offer
**Bulk auto-pilot mode** (see the dedicated section after Phase 4) as the mechanism — don't try to
run that many items through the normal one-at-a-time Phase 3/4 loop in this same conversation, it
will blow out this session's context. Bulk mode is opt-in and explicit: only use it if the user
asks to work through many/all items without a per-item confirmation checkpoint (that trade-off is
the whole point of the mode — see its own section for what stays non-negotiable regardless).

After presenting, update `/memories/repo/intelligence-improve-state.md`'s "Queued next picks"
section with this pass's ranked shortlist (even before the user chooses) so a future session that
gets interrupted here can resume the same ranked list instead of re-deriving it.

---

## Phase 3 — Deep-dive the chosen opportunity (or fast-track batch)

Only after the user chooses one — or, for `[fast-track]`-tagged rows, chooses a small set to
batch (see Phase 2) — do the deeper work. **Every item, batched or not, still gets its own full
3a research pass and its own 3b plan bullet points** — batching only shares the confirmation
checkpoint at the end of 3b, never the research or the verification method. If 3a research on any
batched item reveals it doesn't actually qualify (e.g. it turns out M-effort, or touches something
riskier than expected), pull it out of the batch and treat it individually per the un-batching
precedent in continuous-improvement.md §5.1 — don't force a bad fit into the shared checkpoint.

This phase has two parts, in order, run once per item:

### 3a. Research and confirm current state

Before proposing anything, go re-examine the actual code/config the register row points to:

- Confirm the gap **still exists** as described (the register may be stale — code moves on).
- Quantify what you can: current behaviour, relevant numbers if visible (e.g. current top_k,
  current model tier, current timeout values, existing prompt text), and how it compares to the
  best-practice baseline implied by DESIGN.md / ADRs.
- Estimate the **uplift** — the realistic expected improvement in quality and/or speed/cost from
  closing this gap, and its confidence level. If the evidence doesn't support a confident
  estimate, say so rather than inventing a number.
- Check for any complication not visible from the register alone (e.g. the fix interacts with
  another in-flight phase, or a dependency has already changed).

If this research shows the opportunity is stale, already fixed, or less valuable than recorded,
**say so and update the register (§4)** instead of proceeding to a change plan — go back to
Phase 2's shortlist and offer the user the next-ranked item instead.

### 3b. Propose the change plan

Per §7 of the framework, write a concrete plan:

- The single lever to change, behind a flag/config where possible (reversible).
- Expected effect on **quality and speed/cost** (and risk, for category 8), grounded in the 3a
  research — not the register's original estimate alone.
- How success would be **verified** (eval delta via the RAGAS harness, manual check, or trace
  inspection) — describe it; don't run it here.
- **If verification requires a live/manual test against a real deployed environment** (real
  Vertex AI/Cohere calls, a real adversarial payload, real network timing — anything this
  workspace can't exercise because there are no local `.env`/credentials for `intelligence`, which
  normally only runs on its VM) **that can't be run in this session**, say so explicitly and plan
  to add it to [`intelligence/documentation/manual-verification-plan.md`](../../intelligence/documentation/manual-verification-plan.md)
  in Phase 4 rather than treating verification as optional or skipping it silently.
- **Whether a before/after RAGAS run applies.** The harness (`evaluation/ragas_eval.py`) only
  drives `knowledge_retrieval` + `synthesize` (per OPP-006), so it's a meaningful before/after
  signal for opportunities on that path (categories 1, 2, 4, and cross-cutting changes that feed
  synthesis quality/faithfulness, e.g. 8/9 when they touch the synthesis prompt or context). It's
  **not** a fit for the unevaluated NL→SQL/data path (category 5), or for changes with no
  quality-facing effect (pure logging/observability, e.g. 10/11, or mechanical config cleanup) —
  for those, say so explicitly and rely on the manual/trace verification method instead. If it
  applies, say so in the plan and flag that Phase 4 will run the harness once before
  implementing (baseline) and once after (delta), not automatically here.
- Docs to update (`README.md` always) and whether an ADR is
  warranted (see §4 — non-obvious design decisions, trade-offs, or reversals of prior ADRs
  usually warrant one).
- Any cross-repo issue to raise (for category 6 / knowledge-lakehouse findings).
- How the change will be **documented** once implemented (see the documentation bar below) —
  don't leave this as an afterthought in the plan.

Present the plan and **ask the user to confirm before writing any code**. For a fast-track batch,
present all the batched items' plans together (clearly separated per `OPP-NNN`) and ask for **one
confirmation covering the whole batch** — this is the efficiency gain of fast-tracking. If the
user wants to drop one item from the batch or asks a clarifying question about just one, treat
that as narrowing the batch, not rejecting all of it — proceed with whatever subset is confirmed.

Once the user chooses (Phase 2) and 3a/3b are underway, update the "Current item(s) in flight"
section of `/memories/repo/intelligence-improve-state.md` with the `OPP-NNN`(s) and the current
step (e.g. "3a research complete, plan proposed, awaiting confirmation") — this is what lets a
session that gets interrupted mid-plan resume at the right point instead of restarting Phase 3.

### Documentation bar for any implementation

If the plan proceeds to actual code changes, hold every change to a high documentation standard
— this is non-negotiable, not optional polish:

- **In-code comments:** every non-obvious change gets a comment explaining *why*, not just what
  (the rationale, the trade-off accepted, the alternative rejected and why). Any new flag/config
  knob is documented at its definition (valid values, default, effect). Any workaround or
  gotcha introduced gets a comment flagging it as such.
- **README:** update `intelligence/README.md` (endpoints, config, deployment, gotchas) if
  human-observable behaviour changed, per the documentation standard — don't just append a line,
  integrate the change into the existing narrative so the docs stay coherent, not a changelog
  bolted on.
- **ADRs:** per the §4 guidance above — write one whenever the change embodies a non-obvious
  decision or trade-off, and mark any superseded ADR explicitly.
- Treat documentation completeness as part of the deliverable, not a follow-up task — the change
  isn't done until the comments and docs are in place alongside the code.

---

## Phase 4 — Record

Once the user confirms the plan, implement it (per §7/3b), then close out the register row —
this phase isn't done until the row reflects the actual outcome, not just the intent to proceed.
**For a confirmed fast-track batch, implement and close out each item in turn** (one at a time,
in the shortlist's ranked order) — a shared confirmation does not mean a shared implementation
step; each item still gets its own code change (or its own file/section within one change if they
truly share files per §5.1), its own verification, and its own register closure below:

- **If 3b determined a before/after RAGAS run applies:** run `evaluation/ragas_eval.py` against
  `test_set.json` **before** making the code change to capture a baseline (skip this if a recent,
  still-valid baseline already exists), implement the change, then run it again **after** and
  report the per-metric delta (faithfulness, answer_relevancy, etc.) in the Phase 4 summary. If
  the delta is flat or negative where an uplift was expected, say so plainly — don't paper over a
  null result — and note it in the register row rather than marking it a clean `done`.
- Update the register row's status:
  - `chosen` → `in-progress` as soon as the plan is confirmed and implementation starts — update
    it **in place in `OPPORTUNITIES.md`** (still active, not yet closed).
  - `in-progress` → **`done`** once the code change, docs, and ADR (if warranted) are all
    actually in place — don't leave it sitting at `in-progress` after finishing the work.
  - `done`/`wontfix` with a one-line reason if the Phase 3a research showed it shouldn't proceed
    (in which case skip straight there and never mark it `in-progress`).
  - **The moment a row reaches `done` or `wontfix`, move its entire row (verbatim, same `OPP-NNN`
    ID, full notes) from `OPPORTUNITIES.md` to `OPPORTUNITIES-ARCHIVE.md` and delete it from the
    active file** — closing a row isn't finished until it's out of the active register; don't
    leave closed rows sitting in `OPPORTUNITIES.md`.
  - Clear this row from `/memories/repo/intelligence-improve-state.md`'s "Current item(s) in
    flight" the moment it reaches `done`/`wontfix` — a resume pointer to a closed row is worse
    than no pointer at all. If it's part of a fast-track batch and other items are still
    in-progress, update the entry to reflect only what's left, don't clear the whole line.
- **Create or update the relevant ADR(s)** in `documentation/adr/`:
  - If the change embodies a non-obvious design decision, a trade-off, or supersedes/extends an
    existing ADR (e.g. ADR-0008's reranker choice), write a new ADR following the repo's existing
    ADR format and add it to `documentation/adr/README.md`'s index.
  - If it extends or reverses a prior decision, mark the old ADR as **Superseded by** the new one
    (matching the pattern already used for ADR-0007 → ADR-0008 in DESIGN.md).
  - Minor/mechanical changes (e.g. a config tweak with no real trade-off) don't need a new ADR —
    note in the register why one wasn't warranted.
- Update `intelligence/README.md` per the documentation standard if implementation proceeded,
  and verify in-code comments meet the documentation bar (3b) — check the diff, don't assume.
- **If 3b identified a deferred live/manual test**, add it to
  [`intelligence/documentation/manual-verification-plan.md`](../../intelligence/documentation/manual-verification-plan.md)
  as a new `MVP-NNN` entry (next sequential number) — precondition, exact steps, expected result,
  status `pending` — and reference the `MVP-NNN` id back in the register row's Notes/link column.
  A change that needed live verification isn't fully "done" until that test case exists, even
  though the test itself hasn't been *run* yet; don't let it become an untracked TODO in prose.
- **Commit the change once the row is `done`/`wontfix` and docs/ADR are in place** — one commit
  per closed item (or one commit per fast-track batch, if its items were already grouped for
  sharing a file per §5.1 — don't split a genuinely shared-file batch into multiple commits that
  each leave the repo in a half-updated state). Message references the `OPP-NNN`(s) and a
  one-line summary of the fix, mirroring the register row's own one-line opportunity text.
  Commit freely — this repo's convention is to commit as you go; **never `push`, merge, or open a
  PR without the user's explicit confirmation first** (per the repo's infra/git policy — that
  goes for every commit made by this prompt, not just bulk mode).

Report a summary: what was recorded, which ADR(s) were touched, which docs/comments were updated,
which `MVP-NNN` manual-verification entries (if any) were added, and the register's **final**
status for this row (confirm it's `done`, not left at `in-progress`). For a fast-track batch,
report this once per item so each `OPP-NNN`'s outcome is individually traceable, even though they
shared one confirmation checkpoint.

---

## Bulk auto-pilot mode — working through many/all items in one request

**Opt-in only.** Use this instead of the normal Phase 3/4 loop when the user explicitly asks to
work through a large set (or "all") of open rows **without** a per-item confirmation checkpoint —
if they haven't said that, default to the one-at-a-time (or small fast-track batch) flow above.
The reason this exists: running many items through Phase 3/4 serially, inline, in one conversation
would load every item's research/implementation into this session's context — expensive and, at
enough items, likely to degrade quality as context grows. Delegating each item to a **subagent**
(`runSubagent`) keeps this session's context to just the final summaries: the subagent's full
research/edit trail lives in its own isolated context and its file edits land in the real
workspace regardless.

### How it works

1. **Cluster the selected items by shared files/theme**, using the same test as continuous-
   improvement.md §5.1 (same file(s), same root cause). Each cluster becomes one subagent
   dispatch; items with no file overlap with anything else are their own single-item cluster.
2. **Dispatch one `runSubagent` call per cluster.** Clusters confirmed to touch disjoint files run
   **in parallel** (issue their `runSubagent` calls together, per the parallelization guidance);
   a cluster with 2+ items sharing files runs those items **sequentially inside that one subagent
   call** (give it all the items in the cluster and tell it to work through them in order,
   updating docs/register after each — same discipline as a fast-track batch, just unsupervised).
3. **Give every subagent the full context it needs to act autonomously**, since it cannot pause to
   ask the user anything mid-run:
   - The register row(s) it owns (full text — evidence, impact/effort/risk/confidence).
   - A pointer to `continuous-improvement.md` §7 (from-opportunity-to-change) and the
     "Documentation bar" section above — tell it explicitly this is non-negotiable, auto-pilot
     removes the *human confirmation checkpoint*, not the rigour (3a research, verification
     method, in-code comments, README updates, ADRs where warranted).
   - Explicit instruction: **do not stop to ask for confirmation** (it can't reach the user
     anyway) — if 3a research shows the opportunity is stale/already-fixed/lower-value than
     recorded, it should update the register to `wontfix`/`done` with a reason and stop there for
     that item, rather than forcing a low-value change through.
   - Explicit instruction to update `OPPORTUNITIES.md`/`OPPORTUNITIES-ARCHIVE.md` and
     `/memories/repo/intelligence-improve-state.md` itself for its own item(s) before returning —
     don't leave that to the parent session, since the parent's job here is dispatch + aggregate,
     not re-derive per-item state from subagent prose.
   - Tell it whether to expect writing code (yes) per the tool's own guidance for clarity.
   - **Tell it to commit its own cluster's change before returning** (one commit per item, or one
     per cluster if the cluster is a genuine shared-file batch) — same message convention as
     Phase 4 (`OPP-NNN` + one-line summary) — and explicitly **never to `push`, merge, or open a
     PR**; that stays a user-confirmed action outside any subagent's authority.
4. **As each subagent returns, immediately reconcile** (don't wait for all of them): confirm its
   register/memory updates actually landed (spot-check, don't just trust the summary — same
   principle as Phase 4's "check the diff, don't assume"), and note anything it flagged as
   un-batched, stale, or needing a deferred `MVP-NNN`.
5. **Report one consolidated summary** at the end: per item, what happened (done/wontfix/partial),
   any ADRs written, any `MVP-NNN` entries added, and anything that came back needing your
   attention (e.g. a subagent that couldn't confidently complete 3a, or found a scope surprise
   like OPP-023's cross-repo discovery).

### What bulk mode does NOT relax

- The §7 discipline (one lever, evidenced, verified, documented) — every item still gets it, just
  without a human gate in between research and implementation.
- Category 8 (safety/governance) items — these are high enough risk that they should stay out of
  bulk mode even if the user asked for "everything"; flag them back for individual attention
  instead of silently including them.
- The RAGAS before/after caveat: if multiple dispatched clusters touch the harness's evaluated
  path (categories 1/2/4, per Phase 3b's guidance) and run in parallel, a clean **per-item**
  before/after delta isn't possible — capture one baseline before dispatching any of them and one
  combined after-run once they've all returned, and say so plainly in the summary (a shared delta
  across several changes, not an isolated one) rather than presenting it as a clean per-item
  measurement.
- Anything the register already marks as needing a live/manual verification still gets its
  `MVP-NNN` entry — bulk mode doesn't skip that, it just can't run the live test either (same
  no-local-creds constraint as always).

---

## Guardrails

- **This prompt does not scan the codebase for new opportunities** — it works from the register.
  If the register is empty or stale, point the user to `/intelligence-scan`.
- **Research before proposing** — Phase 3 always re-confirms the current state before writing a
  change plan; never propose against a stale register entry without checking it first.
- **Every plan needs evidence and a verification method** — no speculative changes.
- **Speed and quality are assessed together**; flag trade-offs explicitly.
- **ADRs are part of "done"** — a change that warrants one isn't complete until it's written.
- **Documentation is part of "done"** — thorough in-code comments, `intelligence/README.md`
  updates, and ADRs are as mandatory as the code change itself; don't ship one without the other.
- **Commit along the way, never push unconfirmed** — each closed item (or fast-track/bulk
  cluster) gets its own commit once its code, docs, and ADR are in place; committing is routine
  and doesn't need to wait for user sign-off, but `push`/merge/PR always does.
- **Bulk mode delegates, it doesn't dilute** — subagents dispatched per the section above still
  owe the same evidence/verification/documentation bar as an inline Phase 3/4 run; only the human
  confirmation checkpoint is removed, and only when the user explicitly opted into that trade-off.
- **Fast-tracking shares the confirmation, never the rigour** — batched items still each get a
  full 3a research pass, their own verification method, and their own docs/ADR check; only the
  "ask before writing code" checkpoint is combined. The eligibility bar (S-effort, low-risk,
  high-confidence, not category 8, no ADR expected) is deliberately strict — if a batched item
  turns out not to qualify once researched, un-batch it rather than lowering the bar.
- **Deferred live verification is tracked, not skipped** — if a change's only meaningful
  verification requires a real deployed environment unavailable in this session, it gets a
  concrete `MVP-NNN` entry in `intelligence/documentation/manual-verification-plan.md` (steps +
  expected result), not a vague note that verification "should happen later".
- Respect the infra policy: intelligence VM / model infra changes go through `infra/` Terraform
  + GHA, never direct CLI.
