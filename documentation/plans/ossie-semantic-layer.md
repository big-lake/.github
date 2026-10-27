# Implementation plan — Apache Ossie semantic layer

- **Status:** finalised 2026-10-25 — all §7 decisions closed; **nothing implemented**. Ready for implementation starting at Phase 0.
- **Decision record:** [org ADR-0003](../adr/0003-ossie-semantic-layer-source-of-truth.md) (provisional — revisited after implementation)
- **Repos touched:** **semantics (new)**, catalog (refactor), etl, intelligence, infra, .github · follow-on: knowledge · unaffected: api, ui
- **Code state verified against:** local working trees on 2026-10-25 — some repos had feature branches checked out, so re-check file/line refs against `main` before implementing
- **Implementer notes:** read ADR-0003 first (ownership split + its *Revisit when* triggers gate Phase 0). Session context is in repo memory `ossie-semantic-layer-plan.md`. **No migration:** the initial implementation lands via a clean rebuild of all datasets and the etl + knowledge pipelines (§3.9) — don't build compatibility with existing data/OM state. Standing constraints: confirm with the user before any push/merge/deploy and before every destructive step of the rebuild; infra changes only via Terraform + GHA; GitHub App / secrets are human-created in the GitHub UI (record in `SETUP.md`); `.github/TODO.md` must not be edited without explicit user confirmation (§9 rows not applied).

---

## 1. Target responsibilities

| Component | Owns | Does **not** own |
|---|---|---|
| **semantics** (new repo) | Ossie files: descriptions, fields, temporal coverage, metrics, relationships, `ai_context`; SDMX extracts + shared codelists; ontology + mappings; generators; drift detection; evidence orchestration; LLM drafting; bundle assembly, OM projection and publish | Governance/scope; writing to OM; reading silver data (evidence flows do that); runtime query generation |
| **catalog** (refactored) | Governance + scope (`table.yaml`: `namespace`/`table_name`, `display_name`, `domain`, `tier`, `certification`, `owner`, `tags`, `logo`, `links`); taxonomy; Iceberg connector; **projecting** the published bundle's descriptions and temporal coverage into OM — still the **only OM writer** | Descriptions, column docs, temporal coverage; parsing Ossie |
| **OpenMetadata** | Discovery/search, lineage, governance (domains, tiers, owners, tags), descriptions *as received* | Semantic source of truth; anything executable |
| **etl** (Prefect) | Producing silver; **generic** evidence capabilities: `profile_dataset`, `run_probes` | Any knowledge of Ossie or curation formats |
| **GCS `{project}-semantics`** | Published bundles (`bundles/`, incl. `projections/om.json`), transient evidence runs (`evidence/`) | — |
| **intelligence** | NL → semantic plan (LLM) → SQL (deterministic compiler); raw-SQL fallback; OM for discovery only | Executing SQL (api does) |
| **api / ui** | Unchanged — api executes `data_sql_ready` SQL and reads descriptions from OM as today | — |
| **knowledge** | *(follow-on ADR)* concept tagging of documents/chunks against the shared ontology | — |
| **Humans** | Reviewing and merging semantics bot PRs and catalog governance PRs; dispatching prod publishes | Authoring semantic content |

```
 catalog git ── sparse checkout: table.yaml, curation/utils/abs/datasets.yml ──┐
 etl git ────── sparse checkout: ABS config (dsd_id/version), transform files ─┤
                                                                               ▼
 semantics GHA: semantic-drift.yml (schedule + dispatch; environment input)
   1 introspect Iceberg REST (schema, snapshot) ─ diff vs Ossie files + catalog scope ─► targets
   2 gh workflow run etl/run-evidence.yml ──► Prefect: profile_dataset ──► gs://…/evidence
   3 draft in layers (existing / SDMX / evidence / LLM-from-cited-inputs)   [job holds no write credentials]
   4 gh workflow run etl/run-evidence.yml ──► Prefect: run_probes ──► gs://…/evidence
   5 validate ─► branch bot/semantics + gh pr create (evidence env, gaps, declined metrics)
                                   │ human review + merge
                                   ▼
 semantics GHA: publish.yml (one workflow, environment input; push to main → test, dispatch → prod)
   validate ─► assemble per root domain + projections/om.json ─► gs://{project}-semantics/bundles
          └─► gh workflow run catalog/sync_catalog.yml (same environment, jobs=curate)
                                   │                                    │
                                   ▼                                    ▼
 catalog: patch_metadata ◄─ projections/om.json            intelligence: bundle (TTL) + OM discovery
          + table.yaml governance ─► OpenMetadata                       │
                                                       api executes SQL ◄── data_sql_ready
```

### Extraction order

Generators build each dataset's semantics in layers. Each layer only fills what earlier layers left open, and every generated item records its layer (shown in the PR body).

| # | Source | Supplies | Why here |
|---|---|---|---|
| 1 | Reviewed text already in semantics — for the 9 hand-authored datasets, **seeded once** from catalog's existing `description` / `cols.yaml` / `temporal` text (Phase 2); ABS is regenerated from SDMX, not seeded | Human-written descriptions worth keeping | OM is **not** read back — it's a projection (and is rebuilt from scratch, §3.9), so reading it adds a loop and no information, and it holds none of the layer-2 structure |
| 2 | Publisher structural metadata (ABS SDMX, §3.7) | Verbatim descriptions, concept/codelist identities, measure codes, unit attachment, hierarchies, availability, categorisation, geography URIs | Authoritative and deterministic — no LLM |
| 3 | Data evidence (Iceberg schema, `profile_dataset`, `run_probes`) | Types, time roles, keys, observed values, temporal coverage; confirms or rejects layer-2 candidates | Proves what's in *our* snapshot |
| 4 | LLM proposals (cited inputs only) | Non-SDMX descriptions, synonyms, examples, metrics where no publisher structure exists, ontology mappings | Fills remaining gaps; validated by layer 3 before the PR |

Human overrides always win. Otherwise, where layers disagree (a reviewed description contradicts current SDMX text, an SDMX categorisation contradicts catalog's `domain:`), the PR flags it — the generator never silently picks a winner over reviewed content.

---

## 2. Current state (verified) and gaps

### catalog

| Area | Today | Change in this plan |
|---|---|---|
| Curation loading | `ingestion/curation_loader.py` — `load_curated_datasets()` (L158) yields `CuratedDataset(name, table_def, column_defs, source)` (L42) from per-folder `table.yaml`+`cols.yaml` (`_iter_per_folder_datasets`, L91) and ABS `_generated/**` + `overrides/**` (`_iter_consolidated_datasets`, L108; `_deep_merge` L51, `_merge_columns` L69). Generated files without `columns:` load cleanly | Overlay `description`, `column_defs` and `temporal` from the published bundle's `projections/om.json` (§3.8). **`CuratedDataset` shape unchanged**, so `patch_metadata.py` logic is unchanged. Catalog never parses Ossie |
| OM patching | `ingestion/patch_metadata.py` — `patch_table_metadata()` (L103) **skips an empty description** (L113); `temporal` only patched when present (L219); `patch_table_columns()` (L256) skips empty column descriptions and **makes no call for an empty list** | No logic change. Skip-on-empty means a missing projection never blanks OM; on a freshly rebuilt OM it simply leaves descriptions empty until the first publish |
| Scope | `ingestion/run_ingestion.py` `_load_curated_iceberg_tables()` (L155) from `namespace`/`table_name` | Unchanged — `table.yaml` stays the allowlist; semantics reads it |
| Iceberg REST access | `run_ingestion.py` `_patch_iceberg_rest_catalog_for_biglake()` (L110) + `_get_adc_token()` (L96), pyiceberg `load_rest` with `x-goog-user-project` | Unchanged. semantics reimplements the same pattern (no cross-repo import) |
| ABS generator | `curation/utils/abs/generate.py` — 2 SDMX calls/dataset, no LLM, writes `_generated/{category}/{subcategory}/{dataset}.yaml` (`table:` + `columns:`, ~117 files); `--check`, `--list-new`, `--dataset`. Uses dataflow name/description, methodology `EXT_RESOURCE` annotation and concept descriptions; the DSD response it fetches also carries concept/codelist URNs, attribute attachment, codelists with parents/ordering/annotations — all discarded. Constraints and categorisations not fetched | **Slimmed to governance only** in Phase 3: keeps the dataflow call (→ `display_name`, methodology link, credit-free governance fields); drops `fetch_dsd_children`, `build_columns`, `_build_concept_lookup`, `_concept_description`, the `PROVENANCE_COLUMNS`/`META_COLUMNS`/`NUMERIC_CAST_COLUMNS`/`PASSTHROUGH_TEXT_COLUMNS` constants, the `TODO` description marker, and `temporal` emission. `_generated` files become `table:`-only. Descriptions + structure move to semantics' ABS generator |
| ABS overrides | `curation/datasets/abs/overrides/` holds only `.gitkeep` | Nothing to migrate |
| Onboarding helpers | `check_etl_onboarding.py` stubs `datasets.yml` from etl config (`ETL_REPO_READ_TOKEN`); `abs_domains.py` resolves domain FQNs | Unchanged. semantics reads `datasets.yml` for the ABS dataset list |
| Hand-authored datasets | 9 per-folder datasets: `cash_flow_aggregates`, `expenses_by_function`, `fiscal_accrual_aggregates`, `historical_debt` (fiscal_policy); `critical_mineral_deposits` (mining_and_energy); `declared_{private,public}_hospitals` (health_disability_ageing); `federal_electoral_divisions` (electoral); `individual_income_and_taxation` (taxation). Each has `description` + `temporal` in `table.yaml` and a `cols.yaml` | `description` + `temporal` removed and `cols.yaml` deleted in Phase 3 — their text is carried over once as the semantics seed (Phase 2) |
| Workflows | `sync_catalog.yml` (**dispatch only**, `environment` choice input; jobs `taxonomy` → `ingest` → `curate` → `knowledge` chained by `needs`; concurrency `ingest-{env}`); `abs-curation-and-domain-drift.yml` (`generate.py --check` on PR/dispatch + domain-tree regen PR); `onboard-new-abs-datasets.yml` (PR body tells reviewers to fill `TODO(curation)` descriptions via catalog overrides); `onboard-new-knowledge-collections.yml` | `sync_catalog.yml` gains a `jobs` input (`all`/`curate`) so semantics can dispatch a curate-only run; ABS drift `--check` becomes governance-only; onboarding PR text points description work at semantics; new `validate-curation.yml` guard |
| Copilot prompt | `.github/prompts/curate-dataset.prompt.md` generates `table.yaml` + column descriptions from ETL code and fetched web pages | Becomes governance-only; description authoring (incl. the "credit fetched sources" `Source:` line rule) moves to semantics |
| Knowledge containers | `curation/knowledge/**` (`source.yaml`, `documents.generated.yaml`) via `patch_knowledge.py` | **Out of scope** — unchanged |
| Validation | Runtime-only (loader errors); **no tests dir, no schema checks** | `validate-curation.yml`: rejects `description`/`temporal` in `table.yaml`, any `cols.yaml`, `columns:` in `_generated` |
| Docs drift spotted | catalog ADR-0003 says patch runs at end of `ingest.yml`; reality is manual `sync_catalog.yml` | Note when amending ADR-0003 |

### semantics (new repo)

Doesn't exist. Needs: repo creation (human, GitHub UI), branch protection + CODEOWNERS, WIF binding, `biglake-sa-semantics-{env}`, GitHub App installation, and the org-standard docs skeleton (`README.md`, `SETUP.md`, `.github/copilot-instructions.md`, `.github/WORKFLOWS.md`, `documentation/{adr,bugs,security}/`). Layout in §3.1.

### etl

| Area | Today | Gap |
|---|---|---|
| DuckDB | `src/transformation/utils/gcs_duckdb.py` `get_duckdb_con()` — `DUCKDB_PATH` default `/opt/biglake/data/bronze.duckdb`, `DUCKDB_MEMORY_LIMIT` 12GB, threads 1, `DUCKDB_TEMP_DIRECTORY` `/tmp/duckdb_spill` **cleared on every connect** (L93–99), HMAC from Secret Manager, `iceberg` extension | Env-only config is worker-wide. Profiling **must not** share the db file *or* the temp dir: cleanup-on-connect is only safe under the file lock, so a profiling connection with its own db but the shared temp dir would wipe a concurrent silver run's spill files. Add optional `db_path`/`temp_dir`/`memory_limit` params |
| Silver reads | `iceberg_scan(metadata_location)` with `get_table_metadata_location()` (`utils/iceberg_publish.py` L537–560) | Reuse as-is |
| Deployments | `orchestration/{ingest,bronze,silver}.yaml`, all pipeline stages, `biglake-process-pool`; `deploy-prefect.yml` deploys every `orchestration/*.yaml` | New `orchestration/evidence.yaml` picked up automatically |
| Profiling/probes | None (row-count log lines only); no sqlglot; `safe_identifier()` in `1_bronze/utils/identifiers.py` | New flows; add `sqlglot` |
| Remote trigger | No etl workflow triggers runs; knowledge's `promote-documents.yml` does (SSH + `prefect deployment run`) via `biglake-sa-prefect-deploy-{env}` | New dispatch-only workflow |

### intelligence

| Area | Today | Gap |
|---|---|---|
| Data path | `orchestrator._data_path()` → `tools/data_discovery.py` (OM search, ≤3 auto-proceed) → `tools/data_query.py` (OM `get_table_schema()` per table, ≤3 tables, Gemini Flash JSON `{sql,title}`, 2 attempts, `_validate_sql()` incl. OPP-015 unknown-column check, skipped for CTEs/subqueries) → `data_sql_ready`; **intelligence never executes SQL** | Bundle grounding; plan → compile path; fallback |
| Settings | `app/config.py`, `@lru_cache` `get_settings()`; OM disabled when URL empty; manifest fetched from `knowledge_bucket` per ADR-0023 | Add `SEMANTIC_BUNDLE_URI` (empty = disabled), `ENABLE_SEMANTIC_COMPILER` |
| Injection hardening | `describe_sql.py` + `data_query._delimit()` (SEC-002/SEC-004) | Reuse for plan prompt; semantics' LLM drafter needs an equivalent |
| Eval | `evaluation/data_query_eval.py` checks `expected_fqn_contains` + `expect_success` only — **no SQL/result correctness oracle** | Gold results needed before the compiler gate means anything |

### infra

| Area | Today | Gap |
|---|---|---|
| Buckets (`modules/storage`) | raw, lakehouse, api-data, knowledge-data, intelligence-traces | New `{project_id}-semantics` |
| `biglake-sa-catalog-{env}` (`modules/iam/catalog.tf`) | `storage.objectAdmin` + `legacyBucketReader` on **lakehouse** ("upload the storage manifest" — DataLake-connector era), `compute.viewer`, project-level `iap.tunnelResourceAccessor`, `biglake.viewer`; WIF-bound to catalog repo; **no OS Login** | Only `objectViewer` on `semantics/bundles/` (conditional prefix; exact-path reads of `current.json` + `projections/om.json`). No Vertex, no semantics write. ⚠ Lakehouse `objectAdmin` looks over-privileged post-ADR-0005 — candidate catalog `SEC-NNN`, out of scope here |
| `biglake-sa-semantics-{env}` | **New** (25 chars — within the 30 limit) | WIF-bound to `big-lake/semantics`; `objectAdmin` on the semantics bucket; `aiplatform.user`; `biglake.viewer` + `objectViewer` on lakehouse **conditioned to Iceberg `metadata/` objects** (schema/snapshot introspection only — verify the condition against pyiceberg's reads; data reads stay in Prefect evidence flows) |
| `biglake-sa-prefect-deploy-{env}` | IAP + `osAdminLogin`, WIF-bound to etl + knowledge | Used by the new etl dispatch workflow — no change |
| `biglake-sa-data-{env}` (Prefect runtime) | R/W raw, lakehouse, knowledge | Write on `semantics/evidence/` |
| `biglake-sa-intelligence-{env}` | Reads knowledge bucket, writes traces | Read `semantics/bundles/` |
| intelligence VM env | `modules/intelligence/startup.sh` writes `.env` from metadata | Add `SEMANTIC_BUNDLE_URI` |

### knowledge (follow-on only)

Enrichment (`src/enrichment/enrich.py`, schema `utils/schema.py`) emits generic entity types (PERSON/ORG/AMOUNT/YEAR…); Neo4j has `Entity`/`Chunk`/`Proposition` with `MENTIONED_IN`/`DERIVED_FROM`/predicate edges (`indexing/utils/neo4j_writer.py`); `setup/taxonomy/` exists but isn't wired into enrichment. No concept IRIs anywhere.

---

## 3. Contracts

### 3.1 Repo layout (`big-lake/semantics`)

```
datasets/{namespace}/{table}/semantic.ossie.yaml     one physical dataset (+ derived); generated for ABS, bot-maintained otherwise
datasets/{namespace}/{table}/semantic.override.yaml  sparse human corrections (ABS only — generated files are never hand-edited)
datasets/abs/{table}/sdmx.yaml                       SDMX structural extract (generated, §3.7)
reference/abs/codelists/{id}@{version}.yaml          shared ABS codelists (generated, §3.7)
relationships/{root_domain}.yaml                     relationships + model-scoped / drill-across metrics
ontology/biglake.ontology.yaml
mappings/{root_domain}.mapping.yaml
_spec/                                               ossie-schema.json, ontology.json, mapping.json + PINNED.md (the single upstream commit also used for the Python package + validate.py; licence)
src/semantics/                                       ossie_io, iceberg, load/validate, seed (one-shot), generators (abs/, fields/, llm/), evidence client, assemble, project, publish
tests/                                               generator + projection + validator tests (fixtures incl. births_summary, historical_debt)
.github/workflows/                                   validate.yml (PR), semantic-drift.yml, publish.yml
```

Files are keyed by **identity** (`{namespace}/{table}` = the Iceberg identifier), not by taxonomy: domain reclassification in catalog never moves files. Domain is read from catalog at assembly time.

Conventions:
- Each `semantic.ossie.yaml` is a valid standalone Ossie model holding exactly one **physical** dataset (`source == {namespace}.{table}`, `datasets[0].name == {table}`), plus optional **derived** datasets (query `source` over that physical dataset only — e.g. `historical_debt_latest`), marked `custom_extensions` BIGLAKE `{"role":"derived"}`. The 1:1 catalog `table.yaml` ↔ dataset rule applies to physical datasets; derived datasets have no `table.yaml` and are never projected to OM. **Derived datasets are never materialised** — they compile to an inline CTE over their one certified table, and may not reference other derived datasets (traceability principle, §3.6).
- **Pass-through fields:** one per Iceberg column, `name == column`, expression `ANSI_SQL: <column>`. Only these are projected to OM column descriptions. Computed fields are allowed in addition.
- Provenance columns (`meta_*`, `biglake_ingestion_*`) carry `custom_extensions: [{vendor_name: BIGLAKE, data: '{"role":"provenance"}'}]` so the planner excludes them.
- Metrics carry `custom_extensions` BIGLAKE `{"additive_over": [...]}` — the dimensions the metric may be summed across. Empty by default (fail closed); a dimension is added only with evidence (§3.7).
- `dataset.description` is the OM table description; model-level `description` unused at dataset level. Descriptions drawn from fetched web content end with a plain-text `Source: …` credit line (rule moves here from catalog's copilot instructions).
- **Temporal coverage** lives on the dataset as `custom_extensions` BIGLAKE `{"temporal":{"date_field":"…","earliest_data":"YYYY-MM-DD","most_recent_data":"YYYY-MM-DD"}}`; `date_field` must name an `is_time` field. Seeded from catalog (Phase 2) where present, then maintained from profiles (Phase 5).
- Model membership at publish time = root segment of catalog `table.yaml` `domain:` (e.g. `people.population.births` → `people`).

### 3.2 Worked example — `historical_debt` (shows why evidence matters)

```yaml
version: 0.2.0.dev0
name: historical_debt
datasets:
  - name: historical_debt
    source: parliamentary_budget_office.historical_debt
    primary_key: [year, item, uom, meta_budget_release_key]  # confirmed by profile uniqueness probe
    description: >
      Australian Government Securities (AGS) face value and debt breakdown since 1900-01 ...
      Source: PBO Historical Budget Data (https://www.pbo.gov.au/...).
    fields:
      - name: year
        expression: {dialects: [{dialect: ANSI_SQL, expression: year}]}
        datatype: String
        dimension: {is_time: true}            # String-valued grain; profile shows "YYYY-YY"
        description: Australian financial year ...
        ai_context: {synonyms: [financial year, fiscal year, FY]}
      # item, uom, value, record_type, meta_* ...
```

Evidence the generator must use, all from today's `cols.yaml` knowledge:
- `item` mixes total / subtotal / components → **no `SUM(value)` metric**; reconciliation probe (`total = commonwealth + states`, `commonwealth = war + other`) confirms the hierarchy before any metric is proposed.
- The same `(year, item, uom)` repeats across `meta_budget_release_key` with revisions → metrics need a **latest-vintage dataset** (Ossie allows a query `source`), e.g. `historical_debt_latest` filtered to the most recent release.
- Debt is a **stock** → metric is a per-year value, never summed over years. Without a cited source for that, the metric is not generated (fail closed).

### 3.3 Published bundle (GCS)

```
gs://{project}-semantics/bundles/{bundle_id}/        bundle_id = {utc_ts}-{git_sha7}
  index.json            {bundle_id, git_sha, built_at, ossie_spec_version, spec_sha,
                         models: [{name, path, datasets: [source...]}], ontology, mappings: [...]}
  models/{root_domain}.ossie.yaml       assembled: all datasets of the domain + relationships + model metrics
  ontology/biglake.ontology.yaml
  mappings/{root_domain}.mapping.yaml
  projections/om.json   OM projection for catalog (§3.8) — Ossie-free
gs://{project}-semantics/bundles/current.json        {"bundle_id": "..."} — written last (atomic switch)
```

Readers fetch exact paths only (no `list`), so prefix-conditional IAM works (see repo memory: conditional IAM can't scope `objects.list`).

- **Compatibility:** consumers check `index.json`'s `ossie_spec_version`/`spec_sha` against what they were built for; unknown version → keep serving the last good bundle and log loudly (never crash the request path). `projections/om.json` carries its own `projection_version`; catalog checks it.
- **Rollback:** old bundles are retained (lifecycle: keep last N / 90 days); a bad publish is rolled back by re-pointing `current.json` to a previous `bundle_id` via a `publish.yml` dispatch input — no rebuild needed. The rollback also dispatches catalog's curate-only sync, so OM follows.
- **Deployment = this publish.** Ossie has no runtime; `publish.yml` is one workflow run identically for test and prod with an `environment` input (push to `main` publishes test; prod by dispatch).

### 3.4 Evidence flows (etl)

**`profile_dataset`** — params `{run_id, tables: ["ns.table", ...], key_candidates: {"ns.table": [[col,...], ...]}, sample_rows: 20}` → `gs://{project}-semantics/evidence/{run_id}/profiles/{ns}.{table}.json`:

```json
{"table": "ns.table", "snapshot_id": 0, "row_count": 0, "profiled_at": "...",
 "columns": [{"name": "", "type": "", "null_frac": 0.0, "distinct_approx": 0,
              "min": null, "max": null, "top_values": [{"value": "", "count": 0}]}],
 "key_candidates": [{"columns": [], "unique": true, "duplicate_groups": 0}],
 "sample_rows": [{}]}
```
Caps: top_values ≤ 20, strings truncated to 200 chars, sample ≤ 20 rows.

**`run_probes`** — params `{run_id, probes_uri}`; input written by semantics:

```json
{"probes": [{"id": "rel:...", "kind": "inclusion|uniqueness|executes|assert_zero_rows",
             "sql": "SELECT ...", "tables": ["ns.table"]}]}
```
→ `evidence/{run_id}/probes/results.json` `[{id, status: pass|fail|error, value, error, elapsed_ms}]`.

Execution rules inside the flow (defence in depth — probe SQL derives from LLM-proposed expressions):
- sqlglot: single `SELECT`, no DDL/DML/`COPY`/`ATTACH`/`PRAGMA`/`SET`, no file/table functions in the probe text; logical `ns.table` names resolved by the flow to `iceberg_scan(...)` views.
- Own DuckDB: in-memory db, **dedicated temp dir** (`/tmp/duckdb_evidence/{run_id}`), memory limit 2GB, per-probe timeout, result `LIMIT`.
- Deployments on a work-queue with concurrency 1 — this serialises evidence jobs only; they can still overlap a silver run, hence the 2GB cap and isolated db + temp dir.

### 3.5 Trigger path (**decided 2026-10-25: B, authenticated by a GitHub App**)

| Option | How | Verdict |
|---|---|---|
| A. Semantics SSHes to Prefect VM | Grant instance-scoped `osAdminLogin` + `serviceAccountUser` on data SA; mirror `promote-documents.yml` | ✗ Root on the Prefect VM = data SA (R/W raw, lakehouse, knowledge) — privilege escalation for semantics CI |
| C. Semantics calls the Prefect API over an IAP tunnel | would need `iap.tunnelResourceAccessor` | ✗ Self-hosted Prefect API is unauthenticated; API access can create deployments with shell pull steps ≈ code execution as the data SA — same exposure as A |
| D. Event-driven (request file in GCS) | Prefect OSS has no GCS trigger → needs a poller flow or Cloud Function | ✗ More infra, no advantage over B |
| **B. Semantics dispatches an etl workflow** | `gh workflow run run-evidence.yml -R big-lake/etl -f environment=… -f run_id=… -f flow=… -f params=…`; etl workflow uses its existing `biglake-sa-prefect-deploy-{env}`, runs `prefect deployment run … --watch` (verify flag on our Prefect version); semantics finds the run via `run-name: evidence ${{ inputs.run_id }}` and `gh run watch --exit-status`, then reads GCS | ✓ No VM access; etl owns its VM |

**Credential — GitHub App, not a PAT.** Org-owned (not tied to a person), short-lived installation tokens minted per run with the official `actions/create-github-app-token`. Secrets `SEMANTICS_APP_ID` / `SEMANTICS_APP_PRIVATE_KEY` in the semantics repo (human creates the App and secrets in the GitHub UI — semantics `SETUP.md`). Installations and permissions (Metadata read on all):

| Installed on | Permissions | Used for |
|---|---|---|
| `big-lake/etl` | Actions read/write, Contents read | dispatch `run-evidence.yml`; sparse-checkout ABS config + transform files |
| `big-lake/catalog` | Actions read/write, Contents read | dispatch `sync_catalog.yml` (`jobs=curate`); sparse-checkout `table.yaml` + `datasets.yml` |
| `big-lake/semantics` | Contents read/write, Pull requests read/write | bot branch + PR (alternatively the workflow's own `GITHUB_TOKEN`) |

**Containing the token** — Actions write lets it dispatch *any* workflow in etl/catalog on any existing ref:
- `run-evidence.yml` validates inputs: `environment` ∈ {test, prod}, `flow` ∈ {profile_dataset, run_probes}, `run_id` pattern-checked, `params` schema-validated; runs only from `main`.
- `sync_catalog.yml` already restricts `environment` to a choice; the new `jobs` input is a choice too.
- Pre-req: confirm etl's and catalog's deploy/sync workflows for prod are behind GitHub environment protection where intended, so a leaked token can't run an unmerged branch against prod — fix first if not.
- The LLM drafting job never receives this token (Phase 5 guardrails).

### 3.5.1 Evidence environment (**decided 2026-10-25: workflow input, prod preferred**)

Test and prod lakehouses can differ (tables, freshness, release vintages), so evidence must come from a named environment — ideally prod, which may not be available yet.

- semantics' `semantic-drift.yml` takes `environment` (test|prod) as a dispatch input; scheduled runs use repo variable `vars.EVIDENCE_ENVIRONMENT` (set `test` until the prod lakehouse is ready, then flip to `prod` — no code change). The same value drives Iceberg introspection, `run-evidence.yml` and which `{project}-semantics` bucket evidence is read from.
- Every profile/probe result records `environment` + `snapshot_id`; the bot PR body states which environment the evidence came from.
- Bundles are built from git, so they're environment-independent, but a bundle may describe tables absent from an environment. intelligence plans only over bundle datasets that the **same environment's OM** returned in discovery — OM is the per-environment existence check.

### 3.6 Semantic query plan + compiler (intelligence)

```json
{"model": "fiscal_policy",
 "metrics": ["historical_debt_latest.ags_total_face_value"],
 "dimensions": ["historical_debt_latest.year"],
 "filters": [{"field": "historical_debt_latest.year", "op": ">=", "value": "2015-16"}],
 "order_by": [{"field": "historical_debt_latest.year", "dir": "asc"}],
 "limit": 100}
```
or `{"fallback": true, "reason": "..."}`.

Compiler rules: every name must resolve in the bundle (else fallback); field → its expression; metric → its expression; joins via relationships, **many-to-one only** (refuse fan-out paths); filter values rendered as sqlglot literals (never string-spliced); `source` → DuckDB view `{namespace}.{table}` (the api's view naming); query-sourced datasets become CTEs; output passes `_validate_sql()`.

**Traceability principle (decided 2026-10-25):** compiled SQL reads **only certified silver tables, by their `{namespace}.{table}` names**. The semantic layer never creates, materialises or depends on new tables or views after silver — derived datasets and cross-dataset metrics are inline CTEs over certified tables. Query efficiency is traded for direct traceability. The compiler returns the certified source tables alongside the SQL and records them in the intelligence trace.

**Scope:** compiler v1 handles **single-dataset plans only**; any plan spanning datasets falls back to the raw-SQL path. Cross-dataset metrics arrive later (§4 Phase 7b).

Statistical safety rules (apply wherever the facts exist; ABS supplies most of them):
- **Additivity:** every dimension not in a metric's `additive_over` must be grouped by or filtered to one value, else fallback.
- **Hierarchy:** never aggregate a code with its ancestor (e.g. `REGION` `AUS` with states `1`–`8` in `births_summary`); "by state" resolves to the child level via codelist parents.
- **Units:** never aggregate across differing `unit_measure_code`; scale by `POWER(10, unit_mult)` when mixed multipliers are in scope.
- **Code membership:** filter codes must exist in the referenced codelist, else fallback. Publisher **availability** is a planner hint ("these codes should have data"), **not** a compile-time reject — our snapshot can legitimately differ from the current constraint (ingestion lag, retained expired releases), and the constraint names release windows, not our data. The authority on what's actually present is the profile, not the constraint.

### 3.7 ABS SDMX structural extract

Generated by semantics' ABS generator (`src/semantics/generators/abs/`) from public endpoints (shapes verified live against `BIRTHS_SUMMARY`, 2026-10-25). The ABS dataset list comes from catalog's `curation/utils/abs/datasets.yml` (scope); `dsd_id`/`dsd_version` come from **etl's ABS config** (what silver actually decoded with) and are cross-checked against `datasets.yml` — a mismatch is flagged in the PR:
- `GET /dataflow/ABS/{flow_ref}/latest?references=none` — dataflow description (→ `dataset.description`, verbatim + credit line) and annotations.
- `GET /datastructure/ABS/{dsd_id}/{dsd_version}?references=children` — concept descriptions (→ field descriptions, verbatim), structure, codelists. Uses the **configured** `dsd_version` so the extract matches what silver decoded; a newer DSD behind the latest dataflow is reported as drift, not adopted.
- `GET /dataflow/ABS/{flow_ref}/latest?references=contentconstraint` — availability.
- `GET /dataflow/ABS/{flow_ref}/latest?references=categorisation` — category placement.

The ABS silver column contract (provenance/meta columns, `{dim}_code`/`{dim}_name` pairs, numeric-cast and passthrough attributes) moves with it from catalog's `generate.py` — still a deliberate duplicate of etl's `abs_silver_transformation.py` constants (no cross-repo import), now checked against the live Iceberg schema by the drift detector instead of by convention.

```yaml
# datasets/abs/births_summary/sdmx.yaml — generated, never hand-edited
dataflow: urn:sdmx:org.sdmx.infomodel.datastructure.Dataflow=ABS:BIRTHS_SUMMARY(1.0.0)
categorisation: urn:sdmx:org.sdmx.infomodel.categoryscheme.Category=ABS:PEOPLE(1.0.0).POPULATION.BIRTHS
components:
  - {id: MEASURE, role: dimension, column: measure_code, concept: ABS:CS_COMMON(1.0.0).MEASURE, codelist: CL_BIRTHS_MEASURE@1.0.0}
  - {id: REGION, role: dimension, column: region_code, concept: ABS:CS_GEOGRAPHY(1.0.0).REGION, codelist: CL_STATE@1.0.0}
  - {id: UNIT_MEASURE, role: attribute, column: unit_measure_code, attached_to: {dimensions: [MEASURE]}, codelist: CL_UNIT_MEASURE@1.0.0}
  - {id: OBS_STATUS, role: attribute, column: obs_status, attached_to: observation, codelist: CL_OBS_STATUS@1.0.0}
measures:                 # available MEASURE codes → one metric candidate each
  - {code: "1", name: Births}
  - {code: "3", name: Crude birth rate}
availability:             # the Actual constraint valid now — by validFrom/validTo, never "first returned"
  constraint: CR_B_BIRTHS_SUMMARY
  release_version: B
  obs_count: 4500
  codes: {MEASURE: ["1", ...], REGION: ["1", ..., AUS], FREQ: [A]}   # per-dimension list only when ≤200 codes;
                                                                     # else {count: N, codes_omitted: true} — planner falls back to the codelist
  time: {start: "1975", end: "<from constraint>"}
context_notes: []         # CONTEXT annotations, verbatim
default_view: "MEASURE=1+...,ASGS_2011=...,FREQ=A,TIME_PERIOD_START=2010"   # DEFAULT annotation, verbatim, unvalidated
```

```yaml
# reference/abs/codelists/CL_STATE@1.0.0.yaml — shared by every dataset using it
urn: urn:sdmx:org.sdmx.infomodel.codelist.Codelist=ABS:CL_STATE(1.0.0)
codes:
  - {id: AUS, name: Australia}
  - {id: "1", name: New South Wales, parent: AUS, order: 20}
```
Only known annotation types are copied (`ORDER`, `ASGS_LOCI_URI`, `AREA_ALBERS_SQKM`, codelist-level `EXTERNAL_RESOURCE`); unknown types are counted in the PR body, not copied.

These files are committed and published in the bundle (decision 8): the drift job diffs a fresh ABS fetch against git and proposes changes as bot PRs; a codelist file is written once per `{id}@{version}` and never overwritten by a newer version, so datasets on an older edition keep their own file.

| Fact | Used for | Not evidence of |
|---|---|---|
| Concept URN | Ontology concept mapping (Phase 8); join candidates | A safe join |
| Codelist id + version | Drill-across key candidates only on identical id **and** version — geography editions never cross (`CL_LGA_2016` ≠ `CL_LGA_2021`); see decision 3 / Phase 7b | Same population, coverage, period or time basis |
| Measure codes | One metric candidate per code: label as name, filter on `measure_code`, scaled by `unit_mult`, `additive_over: []` | An aggregation rule |
| Attribute attachment | Unit rule in §3.6; units shown to the planner | — |
| Code hierarchy | Hierarchy rule in §3.6; level synonyms ("states", "national") | That children sum to the parent — needs a reconciliation probe |
| Availability | Planner hint for likely-populated filter codes; publisher time coverage vs profile min/max (gap = ingestion lag); `release_version` change triggers regeneration | What our snapshot contains — the profile is the authority (§3.6 code-membership rule) |
| Categorisation | Cross-check of catalog's `domain:` (mismatch flagged in the semantics PR; catalog stays the writer) | — |
| ASGS URIs, area | Region IRIs in the ontology; area enables density-style metric *proposals* | Boundary comparability across editions |
| CONTEXT / DEFAULT annotations | Verbatim description notes; default view as an `ai_context` example | Correct ids — births' `DEFAULT` cites `ASGS_2011`, absent from its DSD; validate before use |

A dimension enters a metric's `additive_over` only when the unit is count-like **and** a reconciliation probe shows the child codes sum to the parent code. `TIME_PERIOD` additionally needs a cited flow (not stock) source.

### 3.8 OM projection + catalog consumption

The only interface between semantics and catalog. Ossie-free, so catalog stays ignorant of the Ossie spec and its churn.

```json
// gs://{project}-semantics/bundles/{bundle_id}/projections/om.json
{"projection_version": 1, "bundle_id": "…", "git_sha": "…",
 "tables": {
   "abs.births_summary": {
     "description": "Dataset: BIRTHS_SUMMARY. …\n\nSource: Australian Bureau of Statistics …",
     "columns": [{"name": "region_code", "description": "ABS `REGION` dimension code — …"}],
     "temporal": {"date_field": "period_date", "earliest_data": "1975-01-01", "most_recent_data": "2023-12-31"}}}}
```

- Keys are `{namespace}.{table}`; only physical datasets; only pass-through field descriptions; `temporal` omitted when unknown. Shapes match `CuratedDataset.table_def["description"]`, `column_defs` and `table_def["temporal"]` exactly, so `patch_metadata.py` is unchanged.
- **catalog loader** (`curation_loader.py`, new `apply_semantic_projection(datasets, env)`): reads `bundles/current.json` → `projections/om.json` from the env's semantics bucket (exact-path reads; `google-cloud-storage` added to `ingestion/requirements.txt`); sets `description`, `column_defs` and `temporal` per dataset matched on `namespace`+`table_name`. It is the **only** source of those fields — catalog files no longer carry them (no precedence or transition logic).
- **Failure modes:** bucket/`current.json` missing or unknown `projection_version` → log a warning and apply no semantic fields; governance patching still runs. Skip-on-empty in `patch_metadata.py` means OM is never blanked — it shows whatever it last received (nothing, on a freshly rebuilt OM).
- **Trigger:** semantics `publish.yml` dispatches `sync_catalog.yml` with the same `environment` and `jobs=curate`. Catalog's existing concurrency group (`ingest-{env}`) queues it behind any running full sync.

### 3.9 Clean rebuild (decided 2026-10-25 — replaces any migration)

The initial implementation does **not** migrate existing data or carry compatibility code for state that doesn't match the target. Instead, once the new code is in place, every environment is cleared and rebuilt from scratch: all datasets, then the etl and knowledge RAG pipelines. No byte-identity checks against current OM, no transition precedence, no staged pilot removal — catalog switches to governance-only in one change.

The only thing carried forward is **source text in git**: the 9 hand-authored datasets' existing descriptions/`cols.yaml`/`temporal` are copied once into semantics as starting content (Phase 2 seed), because they are reviewed human writing, not data. Everything ABS is regenerated from SDMX.

Rebuild sequence (per environment; **test first, then prod**; every destructive step needs explicit user confirmation at execution time and follows the accidental-data-loss rules):

| # | Step | Owner |
|---|---|---|
| 1 | Pre-reqs merged and deployed: Phases 1–5 (semantics repo, governance-only catalog, evidence flows, generators) | — |
| 2 | **Clear** all datasets (lakehouse Iceberg tables + landing/raw as the user decides) and the corresponding OM entities; clear knowledge stores as required for a from-scratch RAG run. Mechanism chosen with the user at execution time — via the repos' existing workflows where they exist, never ad-hoc infra changes | human-confirmed |
| 3 | Re-run etl pipelines from scratch (ingest → bronze → silver) | etl / Prefect |
| 4 | Re-run knowledge RAG pipelines from scratch | knowledge / Prefect |
| 5 | catalog `sync_catalog.yml` `jobs=all` — taxonomy, Iceberg ingestion, governance patching (no descriptions yet) | catalog |
| 6 | semantics `semantic-drift.yml` for the environment — generates/updates every dataset's semantics against the fresh tables (evidence, SDMX, seeded text, LLM gaps) → bot PR → human review + merge | semantics |
| 7 | semantics `publish.yml` → bundle + projection → auto-dispatched catalog `jobs=curate` → OM descriptions populated | semantics → catalog |
| 8 | Verify: every curated table has a description in OM (gaps listed, not silent); intelligence (Phase 7, once built) can load the bundle | — |

Steps 3–4 are independent of semantics and can run in parallel. The semantics PR in step 6 is generated once for test; prod reuses the merged git content — step 6 for prod only produces a PR if prod's tables differ.

---

## 4. Phases

Each phase ships independently. Exit criteria are gates for the next.

### Phase 0 — Decide and spike (no production change)

- [ ] Run the ADR-0003 Phase 0 exit triggers (pilot exit-check, tooling spike); record outcomes in the ADR. Technical spike complete; description-diff review still requires human sign-off (see below).
- [x] Spike (decision 7): pin one upstream `apache/ossie` commit for `core-spec/ossie-schema.json`, `ontology/ontology.json`, `ontology/mapping.json`, the `python/` package and `validation/validate.py`. Confirm: (a) the package loads both pilot files, (b) `validate.py` is callable as a library, (c) our emitter → package load round-trips equal, (d) whether the package models ontology documents. Any failure → fall back to `jsonschema` + own loader.
- [x] Hand-write `historical_debt` `semantic.ossie.yaml` from its `table.yaml` + `cols.yaml` and compare diffs for reviewability (**pilot exit-check** input). Pilot written and compared; human verdict still open.
- [ ] Hand-write `births_summary` `semantic.ossie.yaml` + `sdmx.yaml` (ABS pilot: exercises §3.7, the `additive_over` rule and the `AUS`/state hierarchy trap). Preliminary model and SDMX subset validate; additivity remains unasserted until data probes.
- [x] SDMX coverage survey (read-only, public API) across every flow in `datasets.yml`: count Actual constraints (and how many are currently valid), parent codes, `CONTEXT`/`DEFAULT` annotations, categorisations, geography URIs, and configured `dsd_version` behind the dataflow's latest DSD. Sizes §3.7's value and surfaces shape variance before Phase 5.
- [ ] Verify prod-protection pre-reqs in etl and catalog (§3.5). Checked: both `prod` GitHub environments have empty protection rules and no deployment branch policy; configure before production dispatch.

2026-10-25 spike results: `semantics` commit `d31055b` contains the first pilots; the follow-up pins `apache/ossie@01a5488d840deda309aac68cc0b312c195369b8d` for Python, schemas and validator. Both models load and round-trip, and the callable validator passes. Upstream Python has no standalone ontology/mapping model, so those documents need schema-only validation. The 124-flow survey found zero request failures, 235 Actual constraints (124 currently valid), 51,400 per-flow parent-code entries, 114 DEFAULT and zero dataflow CONTEXT annotations, 124 categorisations, 17,910 per-flow ASGS URIs and no configured/latest DSD mismatches. Polaris importer `bc4eb5d051dcc1825d90aea7f0c2c4cf89015d3c` is a mapping reference only: OAuth, source names and unproven identifier keys differ from BigLake. Full findings and rerun instructions live in `semantics/README.md`.

**Exit:** spike files validate; reviewability judged acceptable — i.e. none of ADR-0003's Phase 0 triggers fired.

**Implementation status (2026-10-26):** Phases 2–4 code is merged to `main`; the first bundle is published to test and the publish → catalog dispatch path has run.
- **P1:** infra applied to **test** only (bucket `{project}-semantics`, `biglake-sa-semantics-test`, bundle/evidence grants; infra `0db5a24`). Prod not applied. Two plan deviations: (1) no lakehouse `/metadata/` read for the semantics SA — GCS IAM conditions support only `startsWith`/`endsWith` (not `contains`/`matches`) and `loadTable` returns schema and snapshot inline; (2) lifecycle deletes only `evidence/` after 30 days — an age rule on `bundles/` would delete the live bundle after a quiet period, so old bundles need pruning logic. **Prod is deliberately not set up** (no prod environment while not live, to save cost): no prod infra apply, no `prod` GitHub environment, no prod protection rules; everything below is test-only until prod exists.
- **P2:** `semantics` has the validator, assembler, projector and publisher, `validate.yml`/`publish.yml` (publish gated by `PUBLISH_ENABLED`), and the 9 seeded datasets (one-shot seed script deleted after use). **GitHub setup done (test only):** App transferred to the `big-lake` org and installed on semantics/catalog/etl; repo secrets/variables and `test` environment (WIF secrets, `PROJECT_ID`) set; org upgraded to Team so the `main` ruleset is enforced (PR + status check; owner bypass since a sole author cannot self-approve); `CODEOWNERS` merged (PR #1). First `publish.yml` run succeeded: bundle `20261025T191114Z-1ed1d2e` and `bundles/current.json` in `big-lake-test-490405-semantics`.
- **P3:** catalog is governance-only (catalog `72ce346`): loader overlay, `jobs` input on `sync_catalog.yml`, `validate-curation.yml`, slimmed ABS generator (124 files regenerated; only `description`/`temporal` removed, verified), prompts split, ADR-0003/0008 amended. **Exit partially met:** publish dispatched a test `jobs=curate` sync (run 40224832112, success). The projection fetch raised no "unavailable" warning (the bundle was read), but every curated table logged "Table not found in OM" because test OM has no Iceberg tables (the 2026-10-24 ingestion also processed 0 records), so no description was actually patched. Exit can only be proven after test OM is populated (Phase 6 rebuild, or a test ingestion run).
- **P4:** etl evidence flows, guard, tests (28 pass), `run-evidence.yml`, docs (etl `c130242`). One plan deviation: `run_probes` takes only `run_id` and derives `evidence/{run_id}/probes/input.json` itself rather than accepting a `probes_uri`, so a caller cannot point it at an arbitrary object. **Deployed to test** (VM restarted by the user; `deploy-prefect.yml` run 40219388306 registered `evidence-profile-dataset` and `evidence-run-probes`). The profile path is proven on test (runs 40226437061, 40226686678): `profile_dataset` via `run-evidence.yml` wrote real profiles (row count, snapshot id, per-column stats) for `declared_public_hospitals` and `individual_income_and_taxation` to `evidence/{run_id}/profiles/`. The PBO tables named in the exit are **not in the test lakehouse** (no `parliamentary_budget_office` namespace or landing), so they profile as "not found"; the concurrent-silver-run and DDL-rejection checks remain unexercised end to end (the guard is unit-tested).
- **P5 (in progress, semantics PRs #2-#4):** done: Iceberg introspection (REST, ADC + `x-goog-user-project`; prefix is a resource path and must not be URL-encoded) and drift detector (`new_table`, `table_absent`, `new_column`, `removed_column`, `type_changed`); field drafter (folded into the ABS generator: names/types from the Iceberg schema, `is_time` from type); ABS generator (`semantics.generators.abs`: `semantic.ossie.yaml`, `sdmx.yaml`, shared codelists, `overrides.yaml` merge, `--check`) — **exit part met**: live run on test generated 123/124 flows, all schema-valid, 1 skipped (table absent from the lakehouse); output deliberately not committed (it reflects the test silver vintage, e.g. `ingestion_date_utc_*`, and arrives via the bot PR); temporal auto-maintenance (`semantics.temporal`, date/timestamp columns only) and `semantic-drift.yml` (two-job split, `prepare` has no repo-write token; PR #4 open, **not yet dispatched**). Live drift run on test found 123 new ABS tables, 5 PBO tables absent from the lakehouse, and ingestion-column renames between the seeded files and test silver. Deviations: the SDMX availability time-range comparison against profiled data (gap shown in the PR) is not built; only changed tables are profiled, so data-only changes need a dispatch. **Not started:** non-SDMX description drafter (Vertex guardrails), relationship inference, metric templates, `ai_context` drafter. Exit "drift job on a table with an added column yields a correct bot PR" is unproven until `semantic-drift.yml` has run.
- **Phase 0 gates still open:** description-diff review sign-off (compare the seeded files, e.g. `semantics/datasets/parliamentary_budget_office/historical_debt/semantic.ossie.yaml`, with the old `table.yaml`/`cols.yaml` in catalog git history). `prod` protection is deferred until a prod environment exists.

### Converter implementation references

Use [apache/ossie-converters](https://github.com/apache/ossie-converters) as an implementation reference, not a required runtime dependency. Its current converters do not import SDMX; ABS still needs the deterministic adapter in §3.7, without an intermediate vendor format.

- **Phase 0 evaluation:** inspect the [Polaris importer](https://github.com/apache/ossie-converters/tree/main/polaris) for reusable Iceberg-to-Ossie mappings (fields, logical datatypes, temporal roles and identifier fields). Check BigLake REST catalog authentication, headers and source naming before considering direct reuse. Iceberg identifier fields are not proof of uniqueness; keep the profile/probe gate. Evaluate import only: its export path creates namespaces and tables and is outside this plan.
- **Phase 5 implementation:** use converter mapping patterns and fixtures to inform the field drafter and SDMX adapter tests. Keep ABS-specific concepts, units, hierarchies and availability extraction in the adapter. Record the upstream commit used as a reference; pin any adopted dependency and preserve required licence/notice attribution for reused code.
- **Ontology boundary:** the [ontology converters](https://github.com/apache/ossie-converters/tree/main/ontology) target Palantir and Spec YAML, not general SDMX, RDF/OWL or SKOS ingestion. They do not replace ABS concept/ASGS identity mapping.
- **Future interoperability:** vendor exporters may distribute our completed Ossie models to supported analytics tools. Before adoption, test fidelity for expressions, `ai_context`, mappings and BigLake safety metadata; unsupported rules must be reported, not silently dropped. No exporter is required for the initial BigLake runtime.
- **Optional upstream contribution:** once the ABS adapter is tested, consider a general SDMX importer contribution separately. Keep direct publisher-fact translation distinct from BigLake-specific metric/join proposals and their evidence checks.

### Phase 1 — stand up the `semantics` repo + infra

| Task | Where |
|---|---|
| Create `big-lake/semantics` (human, GitHub UI): branch protection on `main` (PR + review required, no force-push), CODEOWNERS, environments `test`/`prod` | GitHub |
| GitHub App (human, GitHub UI): create, install per §3.5 table; secrets `SEMANTICS_APP_ID`/`SEMANTICS_APP_PRIVATE_KEY`; variable `EVIDENCE_ENVIRONMENT=test`; WIF secrets per env | GitHub |
| `{project_id}-semantics` bucket; versioning; lifecycle: delete `evidence/` after 30 days, retain `bundles/` 90 days | `infra/modules/storage/` |
| `biglake-sa-semantics-{env}` + WIF binding for `big-lake/semantics`; grants per §2 infra table | new `infra/modules/iam/semantics.tf`; WIF pool config |
| catalog SA: `objectViewer` on `bundles/` (conditional prefix) | `infra/modules/iam/catalog.tf` |
| data SA: create/get on `evidence/` (conditional prefix) | `infra/modules/iam/data.tf` |
| intelligence SA: `objectViewer` on `bundles/` (conditional prefix) | `infra/modules/iam/intelligence.tf` |
| Repo skeleton: Python 3.11 package `src/semantics/`, `requirements.txt` (`pyyaml`, `jsonschema`, `pyiceberg`, `google-cloud-storage`, `requests`), `tests/`, org-standard docs (`README.md`, `SETUP.md`, `.github/copilot-instructions.md`, `.github/WORKFLOWS.md`, `documentation/{adr,bugs,security}/README.md`), `.github/prompts/describe-dataset.prompt.md` (moved from catalog's prompt, description half) | `semantics/` |
| Org docs: add `semantics` to the repo map and diagram | `.github/ARCHITECTURE.md`, `.github/.github/copilot-instructions.md`, `biglake.code-workspace` |

**Exit:** a no-op `publish.yml` dry run authenticates via WIF in both envs and writes/reads a test object; an App installation token is minted and can read contents + list workflows in etl and catalog (dispatch is exercised later — the target inputs don't exist until Phases 3–4); Terraform applied through GHA only.

### Phase 2 — semantics foundation + seed

| Task | Files (semantics) |
|---|---|
| Ossie I/O adapter: load via upstream `OssieDocument`, upstream `validate.py`, deterministic YAML emitter (fixed key order) + round-trip test | `src/semantics/ossie_io.py`; `requirements.txt` (+ upstream package from git at the pinned commit, `subdirectory=python`) |
| Loader + validator: schema vs `_spec/`; 1:1 with catalog scope (sparse checkout of `table.yaml` at `main`); derived-dataset rules; `source` match; `temporal.date_field` names an `is_time` field | `src/semantics/{load,validate}.py` |
| Iceberg introspection helper (schema, snapshot id; REST catalog + `x-goog-user-project`, reimplementing catalog's `run_ingestion.py` L96–152 pattern — no cross-repo import) | `src/semantics/iceberg.py` |
| **Seed** (one-shot): copy the 9 hand-authored datasets' `description`, `cols.yaml` column descriptions and `temporal` from catalog into `datasets/{ns}/{table}/semantic.ossie.yaml` as starting text. No ABS seeding (regenerated from SDMX in Phase 5), no equality guarantees, no ongoing sync — fields/datatypes are completed by the drift job after the rebuild | `src/semantics/seed.py` (delete after use) |
| Assembler (per root domain from catalog `domain:`) + OM projector (§3.8) + publisher (`current.json` last; rollback input; dispatches catalog `jobs=curate`) | `src/semantics/{assemble,project,publish}.py` |
| CI: `validate.yml` on PR (schema, structure, catalog scope — no GCP creds; needs an App token for the catalog sparse checkout); `publish.yml` (push to `main` → test; dispatch → prod) | `.github/workflows/` |
| Tests: projection shape matches what catalog's loader expects (`description`, `columns[{name,description}]`, `temporal`), validator negatives, assembler membership | `tests/` |

**Exit:** seed merged; bundle + `projections/om.json` publish to test; projection shape tests green.

### Phase 3 — catalog becomes governance-only (one change, no transition)

| Task | Files (catalog) |
|---|---|
| `apply_semantic_projection(datasets, env)` — sole source of `description`/`column_defs`/`temporal`, failure modes per §3.8 | `ingestion/curation_loader.py`; call site in `ingestion/patch_metadata.py` `main` |
| Remove `cols.yaml` and `columns:` handling from the loader | `ingestion/curation_loader.py` |
| Dependency | `ingestion/requirements.txt` (+`google-cloud-storage`) |
| `jobs` input (`all`/`curate`): `taxonomy`/`ingest`/`knowledge` run only for `all`; `curate` runs with `if: always() && !failure() && !cancelled()` so it works whether or not upstream jobs ran; set `SEMANTICS_BUCKET=${PROJECT_ID}-semantics` | `.github/workflows/sync_catalog.yml` |
| Remove `description` + `temporal` and delete `cols.yaml` for all 9 hand-authored datasets (after the Phase 2 seed has merged in semantics) | `curation/datasets/*/` |
| Slim `generate.py` to governance-only (removals listed in §2 catalog table); keep `--check`/`--list-new`/`--dataset`; regenerate all ~117 `_generated` files (`table:` only) | `curation/utils/abs/generate.py`; `curation/datasets/abs/_generated/**` |
| Drift workflow comments/issue text → governance-only drift | `.github/workflows/abs-curation-and-domain-drift.yml` |
| Onboarding PR body: drop the `TODO(curation)` description instructions; say descriptions arrive via the semantics drift PR | `.github/workflows/onboard-new-abs-datasets.yml` |
| PR guard: fail on `description`/`temporal` in any `table.yaml`/`_generated`/`overrides` file, any `cols.yaml`, any `columns:` key | new `.github/workflows/validate-curation.yml` |
| Prompt becomes governance-only (description half moved to semantics in Phase 1) | `.github/prompts/curate-dataset.prompt.md` |
| Docs (§6); catalog ADR-0003/0008 status lines + banners | catalog docs |

Safe to merge before the rebuild: skip-on-empty means a sync against the current OM leaves existing descriptions untouched, and the rebuild (Phase 6) replaces that state anyway.

**Exit:** zero `description`/`temporal`/`cols.yaml`/`columns:` in catalog; `validate-curation.yml` green; a test `jobs=curate` sync applies a published projection to OM.

### Phase 4 — etl evidence flows

| Task | Files |
|---|---|
| `get_duckdb_con(db_path=None, temp_dir=None, memory_limit=None)` overrides (defaults unchanged) | `src/transformation/utils/gcs_duckdb.py` |
| `profile_dataset` flow | new `src/evidence/profile_dataset.py` |
| `run_probes` flow + sqlglot guard | new `src/evidence/run_probes.py`; `requirements.txt` (+`sqlglot`) |
| Deployments, no triggers, concurrency-1 queue | new `orchestration/evidence.yaml` |
| Dispatch-only workflow (`run-name: evidence ${{ inputs.run_id }}`; `environment` input selects the env's deploy SA/VM; input validation per §3.5) | new `.github/workflows/run-evidence.yml` |
| Docs | `documentation/evidence/README.md`; `.github/WORKFLOWS.md` |

Can run in parallel with Phases 2–3 (needs only Phase 1's bucket and grants).

**Exit:** profile of the 3 PBO tables lands in GCS while a silver run executes concurrently without lock/spill interference; a DDL probe is rejected.

### Phase 5 — generators + drift automation (semantics)

| Task | Notes |
|---|---|
| Drift detector | catalog-scoped table without a semantics file (new onboarding); Iceberg column without pass-through field (via Phase 2's introspection helper); removed column; changed type; SDMX `release_version` change |
| Field drafter | deterministic from schema + profile (`datatype`, `is_time` from type + format pattern). **Implemented 2026-10-27** as `semantics.fields` (semantics #10), run by `semantic-drift.yml` before profiling, non-ABS tables only: adds undescribed pass-through fields, drops removed ones (never `temporal.date_field`), fixes datatypes, creates a description-less file for a new table; a leading-underscore rename carries the old description |
| ABS generator | `src/semantics/generators/abs/`: `semantic.ossie.yaml`, `sdmx.yaml`, shared codelists (§3.7); 4 calls/dataset, codelists written once per run; `--check` mode; overrides merge for `fields`/`metrics` keyed by `name`; flags categorisation ≠ catalog `domain:` and configured DSD behind latest. Free to improve on catalog's old description formats — there is no existing state to stay compatible with (§3.9) |
| ABS enrichment (decided 2026-10-27) | All deterministic, from responses the generator already fetches; no new API calls. Field descriptions become *{concept name}. {concept description when present.} Codes from {codelist} (e.g. 2–3 example codes)* — concept scheme comes back with the DSD's `references=children` and is currently discarded. Dataset level: methodology URL from the dataflow `EXT_RESOURCE` annotation; category path as plain text ("Labour > Employment and unemployment"). Frequency stated only when availability lists exactly one `FREQ` code. Unit is **not** added: availability carries no `UNIT_MEASURE` codes and the dataflow description already says "Unit of measure: …" (amended 2026-10-27 at implementation). Example codes prefer ones present in availability. Codelists keep per-code ABS descriptions when present. One-off churn PR across all generated files is accepted. Profile-derived facts (cardinality, null rates) deliberately excluded from prose — extension candidate later |
| Non-SDMX description drafter | Gemini via a **drafter-only SA** (see guardrails); inputs: etl config + transformation file (App contents read), catalog `links.source` pages, profile; output JSON sentences each with `source_ids` **and a verbatim supporting quote** that must appear in the cited input; reject unsourced or unquoted sentences; numbers must appear in inputs; inputs delimited as data (SEC-004 pattern); appends the `Source:` credit line. Proposes only via the bot PR; never overwrites a hand-authored file. Ownership differs from ABS: non-SDMX files are hand-owned, the ABS generator owns `datasets/abs/**` |
| Source page fetching (decided 2026-10-27) | Gemini has **no** web/search tools. Our code fetches only URLs listed in the dataset's catalog `table.yaml` `links.source`: HTTPS only, host allowlist (`*.gov.au` + named exceptions), host re-checked on every redirect, private/link-local IPs rejected after DNS resolution, size/time/page caps. HTML and PDF converted to plain text; other types (xlsx etc.) skipped and listed in the PR body. Fetched text is kept as a run artifact for reproducibility |
| Relationship inference | many-to-one candidates from matching key names → `inclusion` + `uniqueness` probes. ABS cross-dataset links are **not** relationships: they're drill-across metric proposals (Phase 7b) keyed on identical codelist id + version (§3.7) — no dimension datasets or tables are created |
| Metric templates | PBO long format: per-`item` value on latest-vintage dataset, per-`uom`; ABS: one candidate per available `MEASURE` code with `additive_over: []`, widened only per §3.7's rule (count-like unit + passing reconciliation probe; `TIME_PERIOD` also needs a cited source); everything else fail-closed |
| `ai_context` drafter | synonyms/examples; pre-PR checks: no synonym collisions within a root domain, examples reference existing metrics |
| Temporal auto-maintenance | `earliest_data`/`most_recent_data` from profile min/max (replaces catalog's manual step); compared with SDMX availability time range — a gap means ingestion lag, shown in the PR |
| Bot PR | `semantic-drift.yml` (daily + dispatch; `environment` input / `vars.EVIDENCE_ENVIRONMENT` per §3.5.1; concurrency group); branch `bot/semantics`, force-push, `gh pr create`/`gh pr edit`; body = evidence environment, summary, probes, gaps, declined metrics with reasons |

**Exit:** dispatching the drift job on a test table with an added column yields a correct bot PR, no human authoring; the ABS generator produces valid files for every flow in `datasets.yml`.

**Status (2026-10-28, test):** drift detector, ABS generator (with enrichment), temporal module, field sync and `semantic-drift.yml` are merged in semantics (#4, #6, #8, #9, #10, #11) and ran end to end: 123 valid ABS files (category 123, frequency 102, methodology 73), 1 skipped. First bot PR (#5) was merged and the first bundle published; catalog `jobs=curate` dispatched (test OM is empty, so it patches nothing). Required fixes along the way: semantics SA lakehouse read (infra #64), generator carries `temporal` forward (#6), bot-PR tests independent of generated files (#9). Not started: non-SDMX drafter (+ drafter SA in infra), relationship inference, metric templates, `ai_context` drafter, availability time-range gap comparison; exit check in progress: the first dispatch after #10 pushed `bot/semantics` with the ATO `_biglake_ingestion_*` -> `biglake_ingestion_*` rename (4 added, 4 removed, descriptions carried) but edited merged PR #5 instead of opening a new one (`gh pr view` matches merged PRs; fixed in #11 to use open PRs only); a re-dispatch should open the real bot PR. Gotcha: the drift report lists `new_table` x1 and `table_absent` x5 for human attention.

**LLM drafting guardrails (decided 2026-10-25):** Vertex AI Gemini via the semantics SA (WIF, `aiplatform.user`), in the same project as the evidence environment.
- **Credential isolation (amended 2026-10-27, semantics SEC-001):** three jobs. (1) **fetch + parse** untrusted pages with **no credentials at all** (no WIF, no App token), plain text out as an artifact; (2) **draft** as a dedicated `biglake-sa-semantics-drafter-{env}` holding only `roles/aiplatform.user` — not the semantics SA, whose `storage.objectAdmin` on the semantics bucket could overwrite `bundles/`/`current.json` and bypass review; (3) **validate** (schema, cited sources, verbatim quotes, numeric grounding, output hardening) and only then create the branch/PR. Prompt-injected output or a parser exploit can't push, dispatch or publish.
- **Output hardening (semantics SEC-002):** generated text becomes bundle content that the Phase 7 planner reads into its own prompt, so a poisoned page could plant instructions that survive as a plausible, correctly-cited sentence. Plain text only (no markup, no links outside the host allowlist); reject imperative/second-person sentences; per-field and per-dataset length caps; human PR review is the backstop and can't be skipped for drafted text.
- **What leaves the project:** public pages, etl code and profiles of public data, to Vertex in the same project. Profiles stay at `sample_rows: 0`; etl files get a secret-pattern scan before being sent (semantics SEC-005).
- **Drift-scoped:** LLM calls only for datasets the drift detector flagged.
- **Caps:** 200 calls/run (initial; tune from usage) and a per-dataset token cap; hitting a cap leaves visible gaps in the PR, not a failure.
- **Reproducibility:** pinned model version, temperature 0; model id + prompt hash in the PR body.
- **Pre-req:** confirm `infra/modules/budget` covers Vertex spend in both projects — budgets alert but don't stop spend, so the caps are the real control.
- **Pre-req (infra):** the drafter SA + its WIF binding in `infra/modules/iam/` before the drafter is built; drop `aiplatform.user` from the semantics SA once nothing else uses it.
- **Security register:** [semantics `documentation/security/risks.md`](https://github.com/big-lake/semantics/blob/main/documentation/security/risks.md) SEC-001–005.

### Phase 6 — clean rebuild (§3.9)

Starts only after Phases 1–5 are merged and deployed. Test first, then prod. Every destructive step is confirmed with the user at execution time.

- [ ] Clear all datasets and their OM entities; clear knowledge stores as needed (§3.9 step 2).
- [ ] Re-run etl (ingest → bronze → silver) and knowledge RAG pipelines from scratch.
- [ ] catalog `sync_catalog.yml` `jobs=all`.
- [ ] semantics drift → bot PR → review/merge → publish → auto-dispatched catalog `jobs=curate`.
- [ ] Verify per §3.9 step 8; repeat for prod.

**Exit:** both environments rebuilt from scratch; every curated table in OM has its description from the semantics projection (gaps listed explicitly); end-to-end ABS onboarding works: `datasets.yml` entry → catalog governance PR → ingestion → semantics drift PR → publish → OM shows the description.

### Phase 7 — intelligence consumption

| Task | Files |
|---|---|
| `SEMANTIC_BUNDLE_URI` VM metadata → `.env` | `infra/modules/intelligence/{main.tf,startup.sh}` |
| Bundle loader: `current.json` → index; TTL cache (5 min); disabled when URI empty | new `app/services/semantic/bundle.py`; `app/config.py` |
| Grounding: schema block from bundle (fields, descriptions, datatypes, metrics, joins, synonyms); OM `get_table_schema()` only for tables not in the bundle | `tools/data_query.py` (`_format_schema_block`) |
| Planner (Gemini Flash, JSON-only, `_delimit`, soft timeout per ADR-0013). **Bundle text (descriptions, synonyms, examples) is delimited as data**, never trusted as instructions — drafted descriptions originate from web pages (semantics SEC-002) | new `app/services/semantic/planner.py` |
| Compiler (§3.6) | new `app/services/semantic/compiler.py` |
| Path: plan → compile → validate → `data_sql_ready`; fallback to raw path; `ENABLE_SEMANTIC_COMPILER` default off | `tools/data_query.py`, `orchestrator._data_path()` |
| **Eval oracle (decided 2026-10-25):** generated and `gold_sql` both executed through the api's `POST /query` (same executor, validator, views and type mapping as production — the public contract, no DuckDB in intelligence, no reach into api internals). Default target: local api (`python run.py`) on the **test** lakehouse; `--api-base-url` for deployed apis (throttled — Cloud Armor 429s). Gold is SQL run in the same window, not stored numbers, so ABS revisions don't break cases. Comparison: equal column + row counts, rows as a multiset after canonical sort, numeric tolerance; a result at the 10,000-row cap is **incomparable**, never a pass; an api validator rejection of compiled SQL is a **fail**. Trap cases assert properties, not values (no `AUS` mixed with states, no stock summed over years, latest vintage only). Runs locally like today's evals; CI optional later. Replace the harness's "SCOPE BOUNDARY" docstring | `evaluation/data_query_eval.py`, `evaluation/test_set.json` |
| Repo-local ADR: planner/compiler design | `intelligence/documentation/adr/00NN-…` |

**Gate:** baseline (current OM path) measured first; enable compiler only if result accuracy ≥ baseline and every trap case passes.

### Phase 7b — cross-dataset (drill-across) metrics

Starts only after the Phase 7 gate passes on single-dataset plans.

- [ ] Declared explicitly in semantics `relationships/<root_domain>.yaml` as model-scoped metrics with a BIGLAKE extension: participating certified datasets, shared code columns (identical codelist id + version), time-alignment rule, per-side additivity.
- [ ] Compiler: aggregate each certified table in its own CTE under its own safety rules, then join on the declared code columns — still no tables or views beyond silver.
- [ ] Generator proposes one only when codelists match **and** probes pass; prefers a publisher-derived measure where ABS already publishes the ratio (e.g. births measure `3`, crude birth rate). Time basis must be cited — e.g. births are calendar-year registrations (flow) while ERP is at 30 June (stock), so a naive year join is rejected.
- [ ] Eval trap cases for time-basis and edition mismatches.

### Phase 8 — ontology + mappings

- [ ] Shared concepts seeded deterministically from the SDMX concept URNs in `sdmx.yaml` (REGION, TIME_PERIOD, FREQ, MEASURE …) plus Publisher, Period, Indicator; region instances use ASGS linked-data URIs (`ASGS_LOCI_URI`, e.g. `http://linked.data.gov.au/dataset/asgsed3/AUS/AUS`) as IRIs; other IRIs where a public vocabulary exists.
- [ ] Mapping docs per root domain (generated, overridable); validate against `mapping.json` (upstream parser support for standalone mapping docs not yet available — validate with schema only).
- [ ] Cross-domain conceptual links surfaced to the planner as *candidates* — a shared concept is not a join; executable only when codelist id + version match and probes pass.

### Phase 9 — knowledge linking (separate ADR)

Enrichment tags documents/chunks with ontology concept IRIs; Neo4j `Concept` nodes + `INSTANCE_OF` edges; IRIs denormalised into pgvector rows at index time (org ADR-0002); intelligence retrieval filters/joins on concept. Requires a new data-SA read grant on `bundles/` — Phase 1 only grants the `evidence/` prefix.

---

## 5. Dependency order

```
P0 ─► P1 ─┬─► P2 ──┬─► P3 ──────┐
          │         └────────────┤
          └─► P4 ────────► P5 ──┴─► P6 (clean rebuild) ─► P8 ─► P9
          P2 ─► P7 (build) ── enable-gate after P6 ─► P7b
```

- P3 needs P2's seed merged (catalog deletes the text only after semantics holds it) and a published projection to test against.
- P5 needs P4's flows and P2's loader/validator/introspection.
- P6 needs P3 and P5 deployed: after the rebuild, catalog must already be governance-only and semantics must already be able to generate every dataset.
- P7 can be built once bundles exist (P2) but is only enabled after P6, when every dataset's semantics live in the bundle.

---

## 6. Documentation updates (per org standard)

| Where | Update |
|---|---|
| semantics `README.md`, `SETUP.md`, `.github/WORKFLOWS.md`, `.github/copilot-instructions.md` | New repo: layout, generators, extraction order, evidence flow, publish/rollback, App + WIF setup (human GitHub UI steps), "credit fetched sources" rule, ABS SDMX gotchas (constraint validity, stale annotation ids) |
| semantics `documentation/adr/` | Repo-local ADRs as needed (e.g. ABS text-format compatibility) — org ADR-0003 covers the split |
| catalog `README.md` | Curation sections become governance-only; "Writing curation descriptions" and "Column description coverage" replaced by a pointer to semantics; ADR-0008 section notes the split |
| catalog `ingestion/README.md` | "How `earliestData` and `latestData` are set" → from the semantics projection; projection overlay + failure modes |
| catalog `.github/copilot-instructions.md` | Project structure (no `cols.yaml`); description-crediting rule moves to semantics; curation is governance-only |
| catalog `.github/WORKFLOWS.md` | `sync_catalog` `jobs` input + semantics-dispatched curate runs; `validate-curation`; "Adding a new dataset" now spans catalog + semantics |
| catalog ADR-0003, ADR-0008 | Status → `amended by org ADR-0003` when Phase 3 merges (status line + banner only) |
| etl `documentation/evidence/README.md`, `.github/WORKFLOWS.md`, `README.md` gotchas | New flows, temp-dir isolation, `run-evidence.yml` callers |
| intelligence `README.md`, `DESIGN.md` | Semantic path, settings |
| infra `README.md`/`SETUP.md` | Bucket, semantics SA + WIF, grants, metadata var |
| `.github/ARCHITECTURE.md`, org `copilot-instructions.md` | `semantics` in the repo map and diagram |
| `.github/DEVELOPMENT.md` | `SEMANTIC_BUNDLE_URI` for local intelligence; semantics local setup |
| `biglake.code-workspace` | Add `semantics` folder |

---

## 7. Decisions (all closed 2026-10-25)

1. ~~**ADR status**~~ — **decided:** org ADR-0003 is `provisional`; Phase 0 exit triggers listed in its *Revisit when*; re-assessed after implementation. catalog ADR-0003/0008 stay `accepted` until Phase 3 merges. `.github/TODO.md` untouched for now (§9 rows not applied).
2. ~~**Trigger path**~~ — **decided:** B via GitHub App (§3.5); evidence environment is a workflow input, prod preferred, `test` until the prod lakehouse is ready (§3.5.1).
3. ~~**ABS conformed dimensions**~~ — **decided:** no dimension datasets or tables (neither generated nor etl-published). Compiler v1 is single-dataset; cross-dataset questions fall back. Drill-across metrics are explicit, reviewed and compile to CTEs over certified tables (Phase 7b). Driven by the traceability principle (§3.6). Revisit etl-published reference tables only if browsable reference data is wanted in the UI — and then as certified datasets in their own right.
4. ~~**LLM in CI**~~ — **decided:** Vertex AI via the semantics SA with the Phase 5 guardrails (isolated drafting job, drift-scoped, 200 calls/run cap, pinned model).
5. ~~**Prod promotion**~~ — **decided:** one `publish.yml` run identically per environment with an `environment` input (push to `main` → test; dispatch → prod), mirroring catalog's per-env workflows.
6. ~~**Eval execution**~~ — **decided:** through the api's `POST /query` (Phase 7 eval oracle row): production executor, gold-as-SQL, property-based traps, local api on test by default.
7. ~~**Upstream tooling**~~ — **decided:** adopt upstream's `python/` package (Pydantic v2 `OssieDocument`, duplicate-key rejection) and `validation/validate.py`, all from **one pinned upstream commit** shared with `_spec/`; all Ossie I/O behind `src/semantics/ossie_io.py`; read via Pydantic, write via our own deterministic YAML emitter with a round-trip check; BigLake rules, extensions, `sdmx.yaml` and ontology/mapping (schema-only) stay ours. Fallback to `jsonschema` + own loader if the Phase 0 spike fails.
8. ~~**Where SDMX facts live**~~ — **decided:** committed in semantics git (`sdmx.yaml` + shared codelist files, plain BigLake YAML) and published in the bundle — reviewable, reproducible from one commit, rolled back with the bundle. Rejected: evidence-only (expires, unreviewed) and fetch-at-publish (non-reproducible; ABS outages would block every publish).
9. ~~**Where the semantic layer lives**~~ — **decided:** a dedicated `big-lake/semantics` repo with one-way dependencies (ADR-0003 decision 2).
10. ~~**Description ownership**~~ — **decided:** descriptions and temporal coverage are owned by semantics; catalog projects them into OM from the **published bundle** (`projections/om.json`, §3.8) and stays the only OM writer; a semantics publish auto-dispatches catalog's curate-only sync for the same environment.
11. ~~**Existing data**~~ — **decided:** no migration and no compatibility code for state that doesn't match the target. All datasets are cleared and the etl + knowledge RAG pipelines re-run from scratch (§3.9, Phase 6). Only the 9 hand-authored datasets' description text is carried over, as a one-shot seed (Phase 2).

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| Ossie spec churn (`0.2.0.dev0`) | One pinned upstream commit for schemas + package + validator; upgrade = explicit migration PR; all Ossie I/O behind `ossie_io.py` |
| LLM-drafted metric is plausible but wrong | Fail-closed rule; reconciliation + execution probes; eval trap cases |
| Prompt injection via data values / fetched pages | Delimit as data, JSON-only, sanitise, numeric grounding check; drafting job holds no write credentials (Phase 5 guardrails) |
| Vertex cost runaway | Per-run call cap + per-dataset token cap; drift-scoped calls; budget alerts |
| Evidence flows starve silver runs | Concurrency-1 queue, 2GB limit, isolated db + temp dir |
| Bot PR fatigue (large ABS diffs) | One rolling branch; PR body summarises; generated files skimmable, overrides small |
| Semantics CI ↔ Prefect VM availability | Drift job retries next schedule; nothing on the user request path |
| Rebuild is destructive (all datasets, OM entities, knowledge stores) | §3.9: test before prod; explicit user confirmation at each destructive step; existing workflows only; hand-authored text preserved in git via the Phase 2 seed |
| OM has no descriptions between rebuild and first publish | Expected and visible; §3.9 steps 6–7 follow immediately; step 8 lists any gaps |
| Onboarding gap: table in OM before its description | Visible (OM shows no description, not a wrong one); daily drift run + auto-dispatched OM sync bound the lag; revisit trigger in ADR-0003 |
| GitHub App token misuse (Actions write on etl + catalog) | Input-validated target workflows, `main`-only, prod environment protection, never exposed to the LLM job |
| Projection unavailable when catalog syncs | Warn and apply nothing — OM keeps current text (skip-on-empty) |
| ABS Data API is beta (no availability guarantee) | Generator fails the run, never writes partial files; drift job retries next schedule; nothing on the request path |
| SDMX constraints are release-specific (births returned expired release A and current B) | Select by `validFrom`/`validTo`; fail if zero or several are current |
| ABS annotations are inconsistent (births' `DEFAULT` cites a dimension not in its DSD) | Annotations are hints; validate every referenced id before use |
| Large geography codelists bloat PRs | Shared, deduplicated codelist files; availability lists capped at 200 codes (§3.7); PR body summarises code-level changes |
| Bad bundle published | `current.json` re-point rollback (§3.3); consumers keep serving the last good bundle on compatibility failure |
| Evidence from test misrepresents prod (different tables, vintages) | Evidence environment recorded in every result and the PR body; flip `EVIDENCE_ENVIRONMENT` to `prod` once available; intelligence scopes plans to the serving env's OM |

---

## 9. Proposed `.github/TODO.md` entries (not applied — confirm first)

- Semantic layer (org ADR-0003): Phase 0 decisions + spike → Phase 1 create `big-lake/semantics` + infra.
- catalog: `sync_catalog.yml` is dispatch-only, so curation merges don't reach OM automatically (pre-existing gap; descriptions get auto-sync via semantics in Phase 3, governance-only merges still don't).
- catalog/infra: review `biglake-sa-catalog-{env}` `objectAdmin` on the lakehouse bucket (DataLake-connector era) — candidate catalog `SEC-NNN`.
