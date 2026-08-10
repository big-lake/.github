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

Read [`OPPORTUNITIES.md`](../../intelligence/evaluation/results/OPPORTUNITIES.md). If it doesn't
exist or has no `open` rows, **stop and tell the user to run `/intelligence-scan` first** — do not
scan the codebase yourself in this prompt.

Filter to `open` (and, if useful, `chosen`-but-stalled) rows, applying the optional filter input.

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

Then **present the shortlist and ask the user which one (if any) to take forward.** Stop here —
do not implement anything, and do not read source files yet beyond what the register already
records.

---

## Phase 3 — Deep-dive the chosen opportunity

Only after the user chooses one, do the deeper work — this phase has two parts, in order:

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
- Docs to update (`robots.md` always; `humans.md` if human-facing) and whether an ADR is
  warranted (see §4 — non-obvious design decisions, trade-offs, or reversals of prior ADRs
  usually warrant one).
- Any cross-repo issue to raise (for category 6 / knowledge-lakehouse findings).

Present the plan and **ask the user to confirm before writing any code**.

---

## Phase 4 — Record

Once the user confirms (or if they choose to stop after Phase 3 research alone):

- Update the register row's status (`chosen` → `in-progress`, or `done`/`wontfix` with a reason
  if the research in 3a showed it shouldn't proceed).
- **Create or update the relevant ADR(s)** in `documentation/adr/`:
  - If the change embodies a non-obvious design decision, a trade-off, or supersedes/extends an
    existing ADR (e.g. ADR-0008's reranker choice), write a new ADR following the repo's existing
    ADR format and add it to `documentation/adr/README.md`'s index.
  - If it extends or reverses a prior decision, mark the old ADR as **Superseded by** the new one
    (matching the pattern already used for ADR-0007 → ADR-0008 in DESIGN.md).
  - Minor/mechanical changes (e.g. a config tweak with no real trade-off) don't need a new ADR —
    note in the register why one wasn't warranted.
- Update `robots.md` / `humans.md` per the documentation standard if implementation proceeded.

Report a summary: what was recorded, which ADR(s) were touched, and the register's current state.

---

## Guardrails

- **This prompt does not scan the codebase for new opportunities** — it works from the register.
  If the register is empty or stale, point the user to `/intelligence-scan`.
- **Research before proposing** — Phase 3 always re-confirms the current state before writing a
  change plan; never propose against a stale register entry without checking it first.
- **Every plan needs evidence and a verification method** — no speculative changes.
- **Speed and quality are assessed together**; flag trade-offs explicitly.
- **ADRs are part of "done"** — a change that warrants one isn't complete until it's written.
- Respect the infra policy: intelligence VM / model infra changes go through `infra/` Terraform
  + GHA, never direct CLI.
