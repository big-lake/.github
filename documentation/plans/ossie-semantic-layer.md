# Ossie semantic layer — status

- **Why:** [org ADR-0003](../adr/0003-ossie-semantic-layer-source-of-truth.md) (provisional). **How:** `semantics/README.md`. **Runtime:** intelligence [ADR-0028](https://github.com/big-lake/intelligence/blob/main/documentation/adr/0028-semantic-query-plan-and-compiler.md) (provisional). **Eval:** [big-lake/evaluation](https://github.com/big-lake/evaluation).
- **Collapsed 2026-11-07** (§0.2 steer 6): the earlier full plan (phases, contracts, §0 review, dated resume blocks) is in git history of this file. Do not append dated blocks; replace "PICK UP HERE".
- **Standing constraints:** `.github/TODO.md` is not edited without user confirmation; infra changes only via Terraform + GHA; `semantics` and `infra` `main` are protected (PR, user merges); confirm before anything prod or destructive; never `git add -A` in semantics.

## PICK UP HERE

State (2026-11-08): compiler + grounding ON on test; eval **73/81 pass^3**, 0 trap violations, 0 gold errors (run [42437018692](https://github.com/big-lake/evaluation/actions/runs/42437018692), data-only after the re-ingest; identical per-case to [42299639355](https://github.com/big-lake/evaluation/actions/runs/42299639355)); baseline 18/81 (run 41282922830). Prod undeployed (`engineering/go-to-prod.md`). Held-out set on hold, including for prod. Consolidation window (§0.2 steer 1): no new capability. No open PRs.

| # | Item | Status |
|---|---|---|
| 1 | Collapse this plan | done 2026-11-07 |
| 2 | `semantics.model` refactor | done ([semantics #45](https://github.com/big-lake/semantics/pull/45)) |
| 3 | Profile in the runner (ADR amending ADR-0003 decision 7) | done 2026-11-08. [ADR-0004](../adr/0004-profile-in-the-semantics-runner.md) `accepted`. `semantics.profile` profiles in `semantic-drift.yml` with DuckDB + the job's ADC token as a `gcs` `BEARER_TOKEN` secret, copying each snapshot to local Parquet first ([semantics #49](https://github.com/big-lake/semantics/pull/49); full run [42447755977](https://github.com/big-lake/semantics/actions/runs/42447755977): 132 tables in ~14 min, bundle identical to the etl-profiled one). etl `profile_dataset` retired (etl `0c30e8f`; `run_probes` and `run-evidence.yml` stay); `semantics.integrity` retired ([semantics #50](https://github.com/big-lake/semantics/pull/50)). **Open decision (user):** the `etl` App Actions-write grant stays while `semantic-verify.yml` dispatches `run_probes` (ADR decision 4); prod Terraform carries it for verify only unless verify moves off etl |
| 4 | Drafter deleted | done ([semantics #46](https://github.com/big-lake/semantics/pull/46)); SA `biglake-sa-sem-draft-{env}` removal merged ([infra #77](https://github.com/big-lake/infra/pull/77)) but **not yet applied**: the post-merge run skipped "Trigger Deploy (test)", so the SA still exists on test until a `Terraform Deploy` dispatch (destroys the SA; confirm with user) |
| 5 | Steer 7: bronze integrity assertions | done ([etl #40](https://github.com/big-lake/etl/pull/40), deployed to test); the semantics cross-check is retired (item 3) |
| 6 | Steer 8: profile freshness | stamp done ([semantics #47](https://github.com/big-lake/semantics/pull/47), [intelligence #22](https://github.com/big-lake/intelligence/pull/22)); `profiled_at` is in every bundle file on test. Ingestion-coupled profiling (trigger `semantic-drift.yml` after bronze) is now unblocked but is new capability: not before the consolidation window ends |
| 7 | Full-history re-ingest on test of `abs.merch_exp`, `abs.merch_imp`, `abs.pop_proj_region` | done 2026-11-08 ([semantics #48](https://github.com/big-lake/semantics/pull/48); eval unchanged). `pop_proj_region` also held rows of a removed flow, so it was dropped and re-seeded from the newest landing file; the two older ones are in `gs://big-lake-test-490405-raw/_landing_quarantine/abs/abs_data_api/pop_proj_region/` (delete when the user agrees). Fixed on the way: etl `9ab0650`, `558a24c` |
| 8 | DQ-0085/87/89 (wrong `index_code` on `abs.itpi_exp`) | done 2026-11-08, no change built (steer 4): `CL_ITPI_INDEX` has two near-identical codes, `3111001` "All groups" and `8093697` "SITC (Rev3) All Groups"; the gold uses the latter while the question says "All groups", so the exact-label match is the other code. Only one table is affected and any fix would be a new duplicate-label disambiguation capability, so recorded in Known limitations. **Stopped at 73** |

Follow-ups outside this plan: etl bronze/silver ABS triggers are hard-coded `run_all` (handed to another agent session as a prompt; not tracked here). Eval rule: one change-class per run, always.
## Phase status

| Phase | State |
|---|---|
| 0 Spike | Done; pins `apache/ossie@01a5488d`. Description-diff sign-off (user) and prod protection still open |
| 1 Infra | Test only; prod not set up |
| 2 Foundation + seed | Done (test) |
| 3 Catalog governance-only | Done |
| 4 etl evidence flows | `profile_dataset` retired (ADR-0004); `run_probes` stays, used only by `semantic-verify.yml` |
| 5 Generators + drift | Done; drafter deleted 2026-11-07 |
| 6 Clean rebuild | Not run; not needed for the eval gate |
| 7 intelligence | Done on test (grounding + compiler ON); see ADR-0028 *Results* |
| 7b Drill-across / roll-ups | On hold; named cases DQ-0030/31/33/76, DH-0001-3 |
| 8 Ontology + mappings | Not planned; needs a new ADR naming a consumer |
| 9 Knowledge linking | Not planned; needs a new ADR naming a consumer |

## Phase 0 — decide and spike

Spike complete (2026-10-25): both pilots load and round-trip in the pinned upstream package; the validator is callable as a library; upstream has no standalone ontology/mapping model (schema-only validation). Findings and rerun instructions: `semantics/README.md`.

## Known limitations

- Visible-case eval only (held-out on hold); run-to-run noise about ±4 cases; read per-case flips.
- Remaining fails: roll-ups (7b), DQ-0046 (`IB` vs `IM`, defensible), DQ-0085/87/89 (label choice: `abs.itpi_exp` has two "All groups" codes, `3111001` and `8093697`; the gold uses `8093697`, the question wording matches `3111001`).
- etl BUG-0001 latent (financial year and month share a shape). Bronze now fails on mixed flows or stacked snapshots, but cannot see existing duplicates.
- A code or period newly present in data is refused or invisible until the next profile run (steer 8).

## Gotchas

1. intelligence caches the bundle 300 s: wait ~5 min after a Publish before an eval; code changes need `deploy.yml`.
2. Prefect tunnel drops overnight: reopen `etl: iap tunnel prefect (:4200)`, then `$env:PREFECT_API_URL = 'http://127.0.0.1:4200/api'`; ABS ingest = flow run on deployment `ingest-abs` (`1e237ece-6f0c-4344-ab66-e8747e977298`) with `parameters={datasets:[...], run_all:false}`.
3. Profiles are the `profiles-out` artifact of the `semantic-drift.yml` run (no longer in `gs://…-semantics/evidence/`). `profile_all=true` refreshes every table (~14 min).
4. Local api `POST /auth/dev/session` hangs without `X-Dev-Auth`; query data via profiles or the eval.
5. etl has no PR checks: run `pytest tests/bronze tests/evidence` locally; etl deploys on push to main.
6. Verify loaders against the real published bundle, not synthetic fixtures.
7. `ingest-abs` takes bare dataset names (`merch_exp`, not `abs.merch_exp`); a full re-pull needs `full_history: true`. bronze-abs/silver-abs then run over every ABS dataset (~30+ min) because their triggers are `run_all`.
8. Dropping a bronze INCREMENTAL table makes the next bronze run re-read every landing file and can hit the VM's ~13.6 GiB spill cap: move older landing files aside first (etl ABS README).
9. A drift run from a branch (`--ref`) still pushes the bot PR; dispatch from `main` unless testing the workflow itself.

---

### 0.2 Course correction (2026-11-07, second independent review) — anti-spaceship steers

Written by an independent review session (user-approved). **This section survives the item-5 plan collapse.** Takes precedence over the open planner item in the PICK UP HERE block.

**Finding.** The core is validated — 18/81 → 73/81 pass^3, 0 trap violations (runs 41282922830 → 42299639355), two real data bugs found by the eval, boundaries held, no dead abstractions in the shipped code. The spaceship risk is no longer the design; it is **deferred simplification**: every session since the 2026-10-31 review shipped features or fixes while §0 items 3–5 stayed open, and the `validate._ext` private-import count grew from 6 to 7 in the same period. These steers add stopping rules, not workstreams.

1. **Consolidation window — no new capability until §0 items 3–5 are done.** Only two exceptions: the user-approved re-ingest of the 3 duplicate-carrying tables (data, not capability), and the DQ-0085/87/89 label-choice investigation, capped by steer 4. Order: (a) collapse the plan (item 5; user approves the collapsed form); (b) the `model.py` refactor (item 4); (c) the profile-in-runner ADR (item 3).
2. **Perimeter ratchet.** No new workflow, SA, GitHub App permission, cross-repo dispatch, or repo without an ADR naming what it *retires*. The item-3 ADR is the model: it removes the etl dispatch/poll chain and `run-evidence.yml` from the daily path; `semantics.integrity` and `profile_all` move with it (also fixing the hard-coded incremental-dataset constant that duplicates etl config). **Do item 3 before any prod Terraform** — otherwise the dispatch-chain grants are replicated into prod and must be unwound twice (cross-reference in `engineering/go-to-prod.md`).
3. **Decide the dormant drafter: delete or activate — no indefinite "parked" (user decision).** ~1,200 lines (`drafter_*.py`, `semantic-describe.yml`, `drafter-fetch-check.yml`) were built and never run live (§0 item 2). Dead code attracts maintenance (host allowlists, the SEC-001..005 surface) and misleads future readers. Recommendation: **delete** (git history preserves it; restore is one revert), mark the security-register rows superseded, and record the restore trigger — a description-less non-ABS table that `.github/prompts/describe-dataset.prompt.md` + a human can't cover.
4. **Stop rule for eval tuning.** The remaining 8 fails are 4 on-hold roll-ups (Phase 7b), 1 defensible reading (DQ-0046) and 3 label-choice (DQ-0085/87/89). Noise is ±4 and there is no held-out set, so each further visible-case iteration buys less and weakens the claim more. Rule: a change must plausibly fix a *class* (≥2 cases or a structural failure mode), else it is not built. If label-choice yields no general mechanism quickly, record it as a known limitation and **stop at 73 — that is a success, not an unfinished job**.
5. **Demote Phases 8–9 from phases to non-plans.** "Phase" implies it will be built; ontology/mappings (P8) and knowledge linking (P9) have zero consumers and no failing case needing them. In the collapsed plan they become one-line entries: *not planned; requires a new ADR with a named consumer*. Phase 7b stays a real, on-hold phase — it has named failing cases (DQ-0030/31/76, DH-0001–3).
6. **Status hygiene (binding once the plan is collapsed).** One PICK UP HERE block, **replaced not appended** — history lives in git, not dated strata. Every figure cited in the plan carries its run id (the stale "13 invented metrics" figure taught this). One change-class per eval run is a hard rule, not a convention.
7. **Move data-integrity checks into etl ingestion — fail loudly, not report-only (added 2026-11-07).** The two biggest wins of this effort were data bugs (BUG-0002/0003) that sat unnoticed for months; the guard built in response (`semantics.integrity`, semantics #44) is semantics checking etl's homework via a step summary nobody is forced to read, and it hard-codes etl's incremental dataset list. Bronze correctness is etl's job: assert at the end of a bronze run that a FULL_SNAPSHOT table has one ingestion batch and every ABS table one `meta_flow_ref`, and fail the flow when violated. `semantics.integrity` then becomes a redundant cross-check — retire it or let it move with the item-3 ADR.
8. **Couple profile freshness to ingestion, or surface the staleness (added 2026-11-07).** `observed_codes` and `time_range` now gate what the planner may choose and the compiler accepts, but they refresh only when `semantic-drift.yml` profiles a table — ingestion and profiling run on independent cadences with nothing in between. A new release's codes or periods are refused or invisible until the next profile run, i.e. answers silently degrade in exactly the window users most ask about (just-released data). Either trigger a profile refresh off ingestion (natural once item 3 puts profiling in the runner), or stamp each table's `profiled_at` into the bundle and have grounding state it, so staleness is visible instead of a footnoted caveat.


