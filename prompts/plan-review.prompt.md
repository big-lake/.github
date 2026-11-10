---
description: "Mid-flight review of an active plan: spaceship check, deviation from plan, hindsight learnings, and a course correction if one is needed. Read-mostly; edits only the plan's status/progress sections after confirmation."
mode: agent
---

# Plan review — are we still building the right thing?

Plan to review: ${input:planPath:Path to the active plan (e.g. .github/documentation/plans/vocabulary-unified-intelligence.md)}

Review the work done so far against this plan. Base every claim on evidence: commits, files, CI runs, run records, and deployed `/health`. Do not rely on memory or earlier chat summaries. If a check can't be verified, say so.

## 1. Establish the facts

- Read the whole plan. Note the current phase, its scope, the ADRs it names, its eval gate, the standing constraints and the open questions.
- In every repo the plan touches, run `git status -sb` and `git log --oneline -5`. Look for uncommitted, unpushed and wrong-branch work.
- Check CI/eval runs: did they *actually* produce what the plan says? A green workflow is not a published result. Read the logs and the run records.
- Check deployed state against `main`: whether the commit `/health` reports matches `main`, and which infra exists versus infra that is only written.
- Compare the plan's progress table with reality. Flag every row that overstates (or understates) what is done.

## 2. Spaceship check

Are we building more than the plan's current phase needs? For each artefact added since the phase started, ask:
- Does it serve the **current** phase's eval gate, or a later phase's?
- Is there a named consumer *now*? (No consumer means don't build it yet.)
- Is there new infra, a new service, a dependency or an abstraction that the phase didn't call for?
- Would a smaller version have moved the gate just as well?

List the spaceship items, if any, with a recommendation for each: keep, shrink, defer or remove. If there are none, say so plainly.

## 3. Deviation from plan

- **Scope deviations:** work done that the plan doesn't list, or plan items silently skipped or reordered.
- **Constraint breaches:** anything that broke a standing rule (for example: unconfirmed edits to TODO.md or ADRs, infra changed outside Terraform/GHA, pushes or deploys without confirmation, held-out data touched).
- **Gate drift:** is the phase still aimed at its stated eval gate? Is any gate item unimplemented but implied done?
- **Binding design corrections:** does any work quietly resurrect something the plan marks as rejected?

## 4. What we learned (hindsight)

What do we know now that we didn't when the plan was written? Consider:
- Assumptions that proved false, or proved true but costly.
- Data or source facts discovered along the way (coverage gaps, upstream quirks, structures that are easier than expected).
- Operational lessons (run timing, quotas, deploy sequencing, flaky infrastructure).
- Places where a later phase's work is now cheap enough to pull forward, or an earlier item now looks unnecessary.
- ADRs whose Assumptions or Revisit-when triggers have now fired. Name each one and propose amending or superseding it; never silently ignore an ADR.

For each learning, give one line of evidence and one line on why it matters.

## 5. Course correction

Decide: **on track**, **minor correction** or **re-plan**.
- If the call is on track, say so and give the next 3 actions in order.
- If a correction is needed, propose concrete changes: what to change in the plan (phase scope, order, gate, risks), what to stop, what to add, and which ADRs to write or amend. Keep it to the smallest change that fixes the problem.
- Present the proposal to the user and **wait for confirmation** before editing the plan's scope, phases or gates, or `TODO.md`. Without asking, you may only correct factual errors in the plan's progress/status rows, since those must match reality.

## Output

Keep it short and scannable. Use these headings: Facts (with any progress rows corrected), Spaceship, Deviations, Learnings, Verdict + proposed correction, Next actions. Report only what's useful: no praise, no restating the plan.
