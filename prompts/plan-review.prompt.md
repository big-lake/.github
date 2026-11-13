---
description: "Mid-flight review of an active plan: are we on track to deliver its purpose (the functional value in its Why), progress by phase (with S/M/L effort), spaceship check, deviations, hindsight learnings, and a course correction if one is needed. Read-mostly; edits only the plan's status/progress sections after confirmation."
mode: agent
---

# Plan review — are we still building the right thing?

Plan to review: ${input:planPath:Path to the active plan (e.g. .github/documentation/plans/vocabulary-unified-intelligence.md)}

Review the work done so far against this plan. Base every claim on evidence: commits, files, CI runs, run records, and deployed `/health`. Do not rely on memory or earlier chat summaries. If a check can't be verified, say so.

**The question that matters most: are we on track to deliver the plan's purpose?** The purpose is the functional value the plan exists to give our system and its users. It is usually stated in the plan's **Why** (for example: "answer cross-publisher questions using compatible, traceable evidence from structured datasets and unstructured documents"). Phases, ADRs, gates and artefacts are only means to that end. Following the plan step by step is secondary. Deviating from the plan is fine when there is a good reason. Flag every deviation, but judge it by whether it moves the system closer to that value, not by whether it matches the plan text.

## 1. Establish the facts

- Read the whole plan. Note its **purpose** (the Why, and any stated non-goals such as "building a concept layer is not the end goal"), the current phase, its scope, the ADRs it names, its eval gate, the standing constraints and the open questions. If the plan has no explicit Why, infer it from the plan's framing and mark it as inferred.
- In every repo the plan touches, run `git status -sb` and `git log --oneline -5`. Look for uncommitted, unpushed and wrong-branch work.
- Check CI/eval runs: did they *actually* produce what the plan says? A green workflow is not a published result. Read the logs and the run records.
- Check deployed state against `main`: whether the commit `/health` reports matches `main`, and which infra exists versus infra that is only written.
- Compare the plan's progress table with reality. Flag every row that overstates (or understates) what is done.

## 2. Purpose — are we delivering the value?

Judge this from the user's side, not from the repos. Ask:
- **What can a user do today that they couldn't before the plan started?** Show it with evidence of real behaviour: official eval results, an actual answer from the deployed system, or a browser check. Merged code, flags that are off, and artefacts with no runtime consumer don't count.
- **How far is the system from the plan's first functional milestone** (for example, "one complete mixed answer")? Say what is still missing for a user to receive it on test.
- **Is each quality in the Why actually showing up in answers?** For example: cross-publisher, compatible, traceable, structured *and* unstructured. Flag any that exist only in contracts, schemas or tests.
- **Are the non-goals and exclusions holding?** For example: no graph database, no agent framework, no LLM-invented ontology, and the concept layer is not treated as the goal. Flag work that drifts toward building infrastructure for its own sake.
- **Is the current phase order still the shortest path to user value?** If something later in the plan now delivers value sooner, or something earlier no longer adds any, say so.

Give one status for the purpose overall (**on track**, **at risk** or **off track**), the single biggest risk to delivering it, and the evidence for both. If the purpose is no longer reachable, or no longer worth reaching, say so plainly.

## 3. Progress by phase

Give an overall progress breakdown across **all** phases in the plan, not just the current one:

| Phase | Effort | Completion | Status | Evidence / what's left |
|---|---|---|---|---|

- **Effort:** S, M or L for the whole phase, as originally scoped. Use the plan's own sizing if it has one. Otherwise estimate it and mark it `(est.)`. S means a few focused changes in one or two repos with no new infra. M means several repos, or one new component or pipeline, or one infra change. L means new infra or services, several repos, or an eval gate that needs new gold data or multiple official runs. If the remaining work has grown or shrunk a lot since the phase was scoped, show it as `M→L`.
- **Completion:** a percentage based on the phase's deliverables and gate items that are verified done. Don't count work that is written but not merged, deployed or run. A phase is 100% only when its eval gate has passed on an official run.
- **Status:** not started, in progress, blocked (say on what), done, or deferred/dropped (say why).
- Below the table, give an **overall completion** figure weighted by effort (S=1, M=3, L=5), plus one line on whether the pace is enough to deliver the purpose. Completion measures plan progress, not user value; section 2 judges value.

## 4. Spaceship check

Are we building more than the plan's current phase needs? For each artefact added since the phase started, ask:
- Does it serve the **current** phase's eval gate, or a later phase's?
- Is there a named consumer *now*? (No consumer means don't build it yet.)
- Is there new infra, a new service, a dependency or an abstraction that the phase didn't call for?
- Would a smaller version have moved the gate just as well?

List the spaceship items, if any, with a recommendation for each: keep, shrink, defer or remove. If there are none, say so plainly.

## 5. Deviation from plan

Flag every deviation, then classify it:
- **Justified:** there was a good reason and it helps, or doesn't hurt, the purpose. Name the reason. Recommend recording it in the plan (and in an ADR if it's a design decision) so the plan matches reality.
- **Unjustified or harmful:** there was no clear reason, or it moves user value further away. Recommend reverting or correcting it.

Types of deviation to look for:
- **Scope deviations:** work done that the plan doesn't list, or plan items silently skipped or reordered.
- **Constraint breaches:** anything that broke a standing rule (for example: unconfirmed edits to TODO.md or ADRs, infra changed outside Terraform/GHA, pushes or deploys without confirmation, held-out data touched).
- **Gate drift:** is the phase still aimed at its stated eval gate? Is any gate item unimplemented but implied done?
- **Binding design corrections:** does any work quietly resurrect something the plan marks as rejected?

## 6. What we learned (hindsight)

What do we know now that we didn't when the plan was written? Consider:
- Assumptions that proved false, or proved true but costly.
- Data or source facts discovered along the way (coverage gaps, upstream quirks, structures that are easier than expected).
- Operational lessons (run timing, quotas, deploy sequencing, flaky infrastructure).
- Places where a later phase's work is now cheap enough to pull forward, or an earlier item now looks unnecessary.
- ADRs whose Assumptions or Revisit-when triggers have now fired. Name each one and propose amending or superseding it; never silently ignore an ADR.

For each learning, give one line of evidence and one line on why it matters.

## 7. Course correction

Decide: **on track**, **minor correction** or **re-plan**. Base the decision on the purpose (section 2), not on how closely the work matches the plan text. A plan with many justified deviations can still be on track.
- If the call is on track, say so and give the next 3 actions in order.
- If a correction is needed, propose concrete changes: what to change in the plan (phase scope, order, gate, risks), what to stop, what to add, and which ADRs to write or amend. Keep it to the smallest change that fixes the problem.
- Present the proposal to the user and **wait for confirmation** before editing the plan's scope, phases or gates, or `TODO.md`. Without asking, you may only correct factual errors in the plan's progress/status rows, since those must match reality.

## Output

Keep it short and scannable. Use these headings, in this order: Verdict (one line, plus the purpose on track / at risk / off track), Purpose, Progress by phase (table + weighted overall %), Facts (with any progress rows corrected), Spaceship, Deviations (justified vs unjustified), Learnings, Proposed correction, Next actions. Report only what's useful: no praise, no restating the plan.
