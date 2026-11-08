# ADR-0004 — Profile in the semantics runner, not through the etl dispatch chain

- **Status:** provisional (2026-11-08; spike passed locally, first CI run as the semantics SA still to confirm)
- **Date:** 2026-11-07
- **Amends:** [ADR-0003](0003-ossie-semantic-layer-source-of-truth.md) decision 7 (the "Rejected: the GHA runner querying the lakehouse directly" bullet and the dispatch-through-etl trigger path). Decisions 1–6 and 8–10 are unchanged.
- **Tags:** semantics, etl, evidence, profiling, ci, perimeter

## Context

ADR-0003 decision 7 sent profiling through etl: `semantic-drift.yml` mints a GitHub App token for `etl`, dispatches `run-evidence.yml`, polls the run (up to 60 min per 50-table chunk), then copies profiles back from `gs://{project}-semantics/evidence/`. It was chosen partly because the runner had "no CI data-plane access" and to avoid re-learning the Prefect VM's DuckDB tuning.

What changed:

- infra #64 gave `biglake-sa-semantics-{env}` lakehouse `objectViewer` + `legacyBucketReader`, so the drift jobs already have data-plane read (they read Iceberg metadata today).
- The profile function is small and generic: `etl/src/evidence/core.py` `profile_table` runs in an in-memory DuckDB capped at 2 GB (`EVIDENCE_MEMORY_LIMIT`), well under a `ubuntu-latest` runner's ~7 GB, so the "VM tuning" concern does not apply.
- The chain costs more than it delivers: a cross-repo App permission (Actions write on `etl`), a dispatch/poll loop in bash, a GCS artifact hand-off, and a hard dependency on the Prefect VM being up (which is why the `profile` job is `continue-on-error` and the daily schedule is gated by `DRIFT_ENABLED`). Prod Terraform would replicate all of it.
- Profile freshness (`observed_codes`, `time_range`) gates what the planner may choose, yet depends on that fragile path (steer 8 of the plan's §0.2).

## Decision

1. **`profile_dataset` runs inside `semantic-drift.yml`** as a semantics module (`semantics.profile`) using DuckDB with the `iceberg` extension against the lakehouse, authenticated as `biglake-sa-semantics-{env}` through `CREATE SECRET (TYPE gcs, BEARER_TOKEN '<ADC access token>')` (spike, below). Profiles are produced as files in the job, not in `evidence/`.
2. **Retire from the daily path:** the `etl` App installation's Actions-write permission, the dispatch/poll step, the profile download from GCS, and the Prefect VM dependency. `DRIFT_ENABLED` is then only a cost switch for the runner, not for a VM.
3. **`semantics.integrity` and `profile_all` move with it** (integrity reads the same in-job profiles; `profile_all` becomes a plain loop). The integrity check's hard-coded incremental dataset list is replaced by reading etl's config from the sparse etl checkout, or dropped once bronze asserts the same thing (item 4 below).
4. **`run_probes` stays in etl.** Probe SQL derives from LLM-proposed expressions; executing it in a job holding WIF credentials is a larger attack surface than a Prefect flow with `guard.py` and an isolated DuckDB. Nothing currently calls it, so no change now; any future move needs its own ADR.
5. **etl's `profile_dataset` flow and `run-evidence.yml` are retired** in an etl change once semantics no longer calls them (confirm with the user before deleting). `run_probes` and its guard stay.
6. **Profile freshness is surfaced, not assumed:** each profile's snapshot id and capture time is recorded, and `profiled_at` per table is stamped into the published bundle so grounding can state it. Coupling profiling to ingestion (trigger the drift workflow when bronze finishes) is a follow-up, simpler now that profiling is one workflow.
7. **Data-integrity assertions belong in etl bronze** (fail the flow when a FULL_SNAPSHOT table has more than one ingestion batch or an ABS table more than one `meta_flow_ref`), not in a semantics report nobody is forced to read. `semantics.integrity` becomes a redundant cross-check and is retired once the bronze assertions exist.
8. **Doc drift settled in the same change:** the code reads `datasets/abs/{dataset}/overrides.yaml`; ADR-0003 and the old plan said `semantic.override.yaml`. The code wins; READMEs say `overrides.yaml` (ADR bodies are not rewritten).

## Spike result (2026-11-08, DuckDB 2.0.0, pyiceberg 0.12, test lakehouse)

| Option | Result |
|---|---|
| (a) `CREATE SECRET (TYPE gcs, PROVIDER credential_chain)` | Fails: DuckDB looks for an AWS profile, it does not read Google ADC. |
| **(a') `CREATE SECRET (TYPE gcs, BEARER_TOKEN '<ADC token>')`** then `iceberg_scan('gs://.../metadata.json')` | **Works.** `abs.cpi` count matched. No HMAC keys needed. |
| Native `ATTACH ... (TYPE iceberg, ENDPOINT ...)` with the token | 403 on `/v1/config` (needs the `x-goog-user-project` header, which the DuckDB secret does not set); not needed. |
| (c) pyiceberg `to_arrow_batch_reader()` into DuckDB | Works; kept as a fallback. |

- etl's own `evidence.core.profile_table` ran unchanged over `abs.pop_proj_region` (24.7M rows, 32 columns) at `memory_limit=2GB`, `threads=2`: 89 s from a laptop. Output matched the etl-produced profile of the same snapshot (same `snapshot_id`, row count, types, min/max, shapes, `distinct_values`); only `approx_count_distinct` differed (HLL estimates, e.g. 91 vs 106), which is not a semantic difference.
- The test lakehouse is 1.27 GiB of Parquet in total (`pop_proj_region` 0.31 GiB, `merch_exp` 0.17 GiB), so a runner in another region reading it costs cents of egress and fits the disk and time limits easily.
- Implementation notes: access tokens last about 1 h, so re-create the secret (or mint a fresh token) per table; the SA already holds `storage.objectViewer` + `legacyBucketReader` on the lakehouse bucket (`infra/modules/iam/semantics.tf`), which is what `iceberg_scan` needs; `evidence.core` imports only `duckdb` and `sqlglot` (via `guard`), so it can be vendored or copied rather than rewritten.
- Not verified: the run as `biglake-sa-semantics-test` itself (the operator has no `tokenCreator` on it). The first `semantics.profile` CI run proves it; if it 403s, grant is the first suspect, then fall back to (c).

## Consequences

- Positive: removes a cross-repo dispatch chain, one App permission and the VM dependency from a daily job; prod Terraform no longer has to replicate the grant; profiles and their freshness live next to the code that consumes them; the incremental-list duplication disappears.
- Negative: DuckDB-on-runner is a second place that reads silver (a bug in one, e.g. the `_shapes()` format logic, now needs fixing in two while etl's flow still exists, until item 5); runner memory is a hard ceiling (about 7 GB) instead of the VM's configurable limit.
- Follow-on: etl change to retire `profile_dataset`/`run-evidence.yml` (item 5); etl bronze integrity assertions (item 7); bundle `profiled_at` stamp and a grounding line (item 6); infra: remove the `etl` App permission and (separately) the unused drafter SA.

## Assumptions

- DuckDB on `ubuntu-latest` can read the lakehouse's Iceberg tables with the semantics SA's credentials. **Verified by spike 2026-11-08** (see below), locally with the operator's ADC token; the SA identity itself is confirmed only by the first CI run.
- The largest ABS tables (`abs.merch_*` 32–34M rows, `abs.pop_proj_region` 49M) profile within 2 GB and the runner's disk/time budget; etl already does it in 2 GB.
- Low hundreds of small tables (ADR-0003 scale assumption).

## Revisit when

- The spike fails or the largest table exceeds runner memory or the 6-hour job limit (fall back to etl, but then keep the dispatch for those tables only).
- Profiling needs more than deterministic stats (e.g. probes that read arbitrary silver data) — then the etl isolation argument returns.
- The lakehouse is no longer readable by the semantics SA, or table count grows by an order of magnitude.
