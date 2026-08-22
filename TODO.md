# BigLake Platform — Org-Wide TODO

Last updated 2026-08-22. This is the single source of truth for cross-repo planning. Repo-local TODOs should only cover repo-internal task tracking; anything that crosses repo boundaries lives here.

**Intelligence RAG-quality work has its own live register — don't duplicate it here.** `intelligence/evaluation/results/OPPORTUNITIES.md` (open/in-progress rows) + `OPPORTUNITIES-ARCHIVE.md` (closed rows) is the authoritative, actively-maintained backlog for retrieval/routing/generation/eval quality opportunities, produced by the `/intelligence-scan` + `/intelligence-improve` loop. This file only summarises the handful of rows that are cross-repo or currently blocked on a human decision — see "Intelligence quality backlog" below.

---

## Current state snapshot

| Repo | Status |
|---|---|
| **infra** | Test env up. Litestream SQLite backup sidecar live (ADR-0010). Lakehouse Iceberg catalogs (`-raw`, `-lakehouse`) provisioned and in full use. Prod parity not yet verified. |
| **etl** | Bronze/silver/gold flows running for treasury + ATO + PBO sources via Prefect. Full Iceberg cutover complete (bronze+silver+gold, `meta_` column convention) — no more flat-parquet dual-write. |
| **catalog** | OpenMetadata deployed, Iceberg connector live, curation YAMLs (`logo`/`links`/`latestData`/`cols.yaml`) cover all 5 curated datasets. Known issue: stale Elasticsearch entries from the decommissioned DataLake connector (OPP-028, mitigated not resolved). |
| **api** | Stage A + Stage B (notebooks) both fully shipped: real intelligence calls (S2S ID-token auth), chat + notebook + article persistence in SQLite w/ Litestream backup, real OM catalog adapter, DuckDB `/query` with Iceberg REST attach, sqlglot read-only enforcement, cookie-based passwordless auth, message feedback endpoint. Known perf issue: ~23s cold-start latency, root cause not fully isolated (see `api-startup-latency` repo memory). |
| **ui** | Live end-to-end against `api` for catalog, knowledge, query, chat tabs, notebooks (CRUD + publish + share), agent. Remaining mocked surfaces: Sponsorship/Stripe checkout, metadata-only "donate to unlock" flow, dataset lineage view (API exists, no UI), knowledge-document PDF preview. |
| **knowledge** | Production through indexing (Flows 1–4) plus a `search_summaries` retrieval tier (OPP-027, done). `search_questions` tier designed but not built (OPP-029, open). Eval harness built; blocked on a human curating the golden set (OPP-005). |
| **intelligence** | Chat + agent surfaces live: hybrid retrieval + Cohere rerank + Gemini synthesis, real `thought` SSE events, gated query decomposition (multi-artifact chat responses, phase 2a), class-based Pro/Flash model routing, per-turn LLM budget, classifier confidence from logprobs. See `evaluation/results/OPPORTUNITIES.md` for the live quality backlog. |

---

## Stage A — Thin end-to-end slice — ✅ COMPLETE

All items shipped and deployed. `ui → api → intelligence → answer-in-browser` and `ui → api → OpenMetadata` are both live in `test`.

- [x] **api/intelligence_client.py → real intelligence call.** HTTP call to intelligence `/chat` + `/agent/send`, GCP ID-token S2S auth both sides ([ADR-0003](../api/documentation/adr/0003-s2s-auth-gcp-id-tokens.md)), citations + trace passed through.
- [x] **Chat sessions in SQLite** ([ADR-0004](../api/documentation/adr/0004-chat-sessions-in-sqlite.md), [ADR-0010](../api/documentation/adr/0010-sqlite-litestream-then-cloudsql.md), api ADR-0023). Full `chats` table (transcript JSON, mirrors notebooks), `GET/POST/PATCH/DELETE /chats`, `ChatBrowserModal.vue` resume flow.
- [x] **Litestream backup of SQLite** ([ADR-0010](../api/documentation/adr/0010-sqlite-litestream-then-cloudsql.md)). `infra/modules/api/startup.sh` runs Litestream as a systemd sidecar, restore-on-boot, writes to the dedicated `{project_id}-api-data` bucket (moved out of the public analytics bucket per `infra/documentation/security/risks.md` SEC-003).
- [x] **api/catalog_client.py → real OM API call.** Real OM adapter, paging, `_om_table_to_api()` mapping ([ADR-0008](../api/documentation/adr/0008-catalog-shape-from-om.md)).
- [x] **Catalog curation.** `catalog/curation/{dataset}/{table.yaml,cols.yaml}` covers `individual_income_and_taxation`, `cash_flow_aggregates`, `expenses_by_function`, `fiscal_accrual_aggregates`, `historical_debt`. Custom properties (`logo`, `links`, `latestData`, `earliestData`) synced via `patch_metadata.py`.
- [x] **api/query_service.py → real GCS+DuckDB.** DuckDB pool with Iceberg REST catalog ATTACH (falls back to `iceberg_scan`), sqlglot AST-level read-only enforcement ([ADR-0006](../api/documentation/adr/0006-duckdb-connection-and-om-views.md), [ADR-0007](../api/documentation/adr/0007-sql-read-only-enforcement.md), [ADR-0009](../api/documentation/adr/0009-query-response-shape.md)).
- [x] **Session persistence.** SQLite `users` table, werkzeug pbkdf2 hashing, cookie-based passwordless auth (magic-link, OAuth, passkey — see api ADR-0014/0015/0017).
- [x] **UI mock removal + session persistence.** `USE_LIVE_CATALOG` / `USE_LIVE_KNOWLEDGE` / `USE_LIVE_AGENT` all `true` in [ui/src/config.js](../ui/src/config.js). Auth persists via the session cookie; `getMe()` on init.
- [x] **Chat error handling + citations rendering.** Citations, trace timeline, `thought` SSE bubbles, clarify/sources cards all live and browser-verified.

## Stage B — Notebooks persistence — ✅ COMPLETE

- [x] Schema designed and shipped (SQLite `notebooks` table, `cells_json` verbatim storage).
- [x] `GET/POST /notebooks`, `GET/PATCH/DELETE /notebooks/{id}`, `POST /notebooks/{id}/publish`, `GET /notebooks/published`, `GET /notebooks/public/{id}` — all implemented in [api/app/routes/notebooks.py](../api/app/routes/notebooks.py).
- [x] `client.js` fully wired (`createNotebook`/`getNotebooks`/`getNotebook`/`updateNotebook`/`deleteNotebook`/`publishNotebook`); `NotebookBrowserModal.vue`, `NotebookPickerModal.vue`, dataset/knowledge source tabs all live.
- [x] XSS sanitisation for published text cells — `api/app/services/html_sanitize.py` (nh3-based), citation-chip attributes allowlisted.
- [x] Cell share endpoint/flow — no new API needed; `cell.id` survives autosave + publish verbatim, `Workspace.vue`/`PublishedNotebook.vue` build deep links (`#cell={id}`) with Web Share API + copy-link fallback.

## Stage C — Prod deploy — status unverified, likely still pending

Not evidenced anywhere in recent work — treat as still open until confirmed otherwise.

- [ ] Verify `infra` prod env is provisioned to parity with test (Prefect VM, OM VM, intelligence VM, knowledge_db, storage, IAM).
- [ ] Populate prod Secret Manager values via GCP Console UI (Cohere API key, pgvector DSN, OM admin, etc.). (human)
- [ ] Run all `etl` pipelines against prod to seed gold-layer datasets in `gs://{prod_project}-lakehouse`.
- [ ] Run all `knowledge` pipelines against prod to populate prod pgvector + Neo4j.
- [ ] Deploy `catalog` (OM ingestion + curation) to prod.
- [ ] Deploy `api` + `ui` + `intelligence` to prod via GHA.
- [ ] Smoke-test end-to-end in prod: login → catalog browse → query → chat.
- [ ] See `environment-copy-plan` repo memory for the (currently undecided) question of copying expensive-to-recompute artefacts (embeddings, enriched chunks) test→prod vs. re-running pipelines from scratch. No code written yet — needs a decision before Stage C if re-running everything from raw landing files in prod is too slow/costly.

---

## Intelligence quality backlog (summary only — see `OPPORTUNITIES.md` for full detail)

The rows below are cross-repo or explicitly blocked on a human decision — everything else lives purely in `intelligence/evaluation/results/OPPORTUNITIES.md`.

- **OPP-005** (human-curation blocker, `knowledge`) — golden-set eval harness (`recall_at_k`/`mrr`/`hit_rate`, candidate generation, retrieval scoring) is fully built, but `golden_sets/treasury/budget_release_papers/v1.yaml` still has `questions: []`. Needs a human to trigger `generate_eval_candidates` (Prefect, real Vertex AI cost) and curate ~50–100 questions. Nothing else in the eval pipeline can progress until this lands.
- **OPP-007** (`in-progress`, `intelligence` + `api`) — feedback capture (`POST /messages/{id}/feedback`, `trace_id` persistence) and a scoped production-trace sampling slice (`TRACE_SAMPLE_RATE` knob, log-based capture) are shipped. Durable/queryable trace storage (GCS bucket + IAM) is written (intelligence ADR-0022, infra Terraform) and **committed + pushed but not yet applied** — needs a human to run the `infra` `deploy.yml` GHA workflow (`workflow_dispatch`) before `TRACE_SAMPLE_BUCKET` resolves to anything real. Drift metric itself still unbuilt.
- **OPP-023** (`open`, needs human decision, `intelligence`+`ui`) — `clarify.request` SSE event has a dead/no-trigger contract. Proposed ADR-0021 (implement vs. deprecate) written but not accepted — needs a human to pick a direction; not bulk-workable.
- **OPP-028** (`open`, `catalog`, high-risk) — stale OpenMetadata/Elasticsearch entries from the decommissioned DataLake connector still occasionally surface in search (mitigated in `intelligence/app/services/om_client.py` via `_prefer_canonical_tables`, not fixed at the source). Real fix (`catalog/README.md` "Purging the decommissioned biglake-datalake service" runbook) is a live, irreversible OM/ES admin operation — deliberately left for a human to run, not automated (accidental-data-loss guardrail).
- **OPP-029** (`open`, `knowledge`) — `search_questions` (hypothetical-questions) retrieval tier has a first design sketch (ADR-0014) but 3 open design questions (parent vs. child chunk scope, dedup strategy, artefact shape) — needs a dedicated design session, not a bulk pass.

---

## Cross-repo infra & platform backlog

- [ ] **Move Prefect server off SQLite onto Postgres.** `biglake-prefect-{env}` runs Prefect's default SQLite backend; concurrent deploys (etl + knowledge share the VM) can saturate its async session pool and wedge the server (`httpx.ReadTimeout` on every API call, incl. `/api/health`). A `concurrency` guard in `etl/.github/workflows/deploy-prefect.yml` mitigates the triggering symptom but doesn't remove the underlying write-contention class. See `prefect-vm-deploy` repo memory for full incident history. (`infra`)
- [ ] **`etl` bronze/silver DuckDB single-writer lock contention.** `get_duckdb_con()` (`etl/src/transformation/utils/gcs_duckdb.py`) opens one shared local DuckDB file (`/opt/biglake/data/bronze.duckdb`) for every bronze/silver flow; DuckDB allows only one connection at a time, so two flows triggered close together (e.g. onboarding several new datasets and running their ingests back-to-back) race for the file lock and the loser fails with `IOException: Could not set lock on file ...`. First hit 2026-08 onboarding `declared_public_hospitals` + `federal_electoral_divisions` together. Current mitigation is manual retry from the Prefect UI once the winning run finishes. Proposed durable fix (needs a decision, not yet built): a work-pool-level concurrency limit of 1 on bronze/silver deployments, or moving bronze/silver DuckDB state off a single shared local file. See `prefect-vm-deploy` repo memory for the incident writeup and `etl/README.md` "Known gotcha" section. (`etl`)
- [ ] **API cold-start latency (~23s first boot, ~6s first request).** Partially diagnosed — Iceberg REST ATTACH + per-table metadata fetches already optimised; remaining culprit not isolated (possibly `INSTALL/LOAD iceberg` hitting the DuckDB extension CDN on first load). See `api-startup-latency` repo memory for the investigation checklist. (`api`)
- [ ] Read-only pgvector role for `intelligence` (new secret in Secret Manager; populate via GCP Console UI). (`infra`)
- [ ] Cost guards / alerts on Vertex + Cohere spend. (`infra`)
- [ ] Public-access proof for lakehouse data (anonymous pyiceberg/DuckDB client reads via public GCS + metadata location) — blocked on an org-admin policy override (`allowedPolicyMemberDomains` blocks `allUsers` at org scope). Deferred; test env doesn't need it, tracked as prod follow-up. (`infra`, human/org-admin)
- [ ] Snapshot-expiry flow first real run TBD — `iceberg_maintenance.py` is deployed but hasn't yet pruned a real 30-day-old snapshot. (`etl`)
- [ ] Partner federation recipe (pyiceberg/Spark/DuckDB → REST catalog connection doc) — deferred until a real partner materialises; no RBAC/credential-vending build until then. (`infra`, `.github` docs)

## UI backlog (not yet started)

- [ ] **Lineage UI.** `GET /catalog/lineage` + `getLineage()` exist but unused — no consumer view built. Deliberately descoped 2026-08-10 pending a defined user story; not removed. (`ui`)
- [ ] **Metadata-only "donate to unlock" affordance.** `Workspace.vue` mocks `availability.state = metadata_only` handling via `/mock/datasets.json` / `/mock/knowledge.json`; needs live BFF fields once the curation pipeline can distinguish fully-processed vs. metadata-only sources. (`catalog`, `api`, `ui`)
- [ ] **Knowledge-document PDF preview (§2.9).** Investigated 2026-08-10, blocked on ingestion-config work, not just UI wiring: no `sourceUrl` field anywhere in the curation layer; only one collection has real ingested PDFs that line up with curation; a bucket-name mismatch was found and needs fixing (`GCS_KNOWLEDGE_BUCKET` wiring in both envs). Full findings in `api/BFF_API_REQUIREMENTS.md` §2.9. (`knowledge`, `catalog`, `infra`, `api`, `ui`)
- [ ] **Sponsorship + Stripe checkout.** `Sponsorship.vue` is fully mocked (`MOCK_SPONSORS`, `MOCK_DONATIONS`); needs `GET /sponsorship/sponsors`, `GET /sponsorship/donations`, `POST /sponsorship/checkout` (BFF-managed, browser only ever sees `client_secret`), `POST /sponsorship/webhook` (Stripe signature verification). Not started. (`api`, `ui`, `infra` for secrets)
- [ ] **Live citation `source_url` for real chat citations.** `chunk_to_citation()` hardcodes `source_url=None` ("None until Phase 3+") — "View source" only works on mock citation data today. Needs the indexing pipeline to persist a real per-chunk GCS path through retrieval → `Citation`. Independent of, and bigger than, the PDF-preview item above. (`knowledge`, `intelligence`, `api`)

## Knowledge/Data-copy tooling (discussion only, not started)

- [ ] Decide whether to build cross-environment artefact copying (test⇄prod) for the lakehouse + vector DB, to avoid re-paying for expensive processing (embeddings, docling parsing, LLM enrichment). Full trade-off analysis (reproducibility risk vs. cost) in `environment-copy-plan` repo memory — recommendation is prod→test only if built at all, with an ADR before any test→prod direction. No code exists yet. (`knowledge`, `etl`, `infra`)

---

## Won't do

- **ColBERT / late-interaction index.** Lowest ROI given scale and cost; revisit only if reranking proves insufficient downstream.
- **Agentic logic inside `knowledge`.** By design. Ingestion stays deterministic and reprocessable. CRAG / Self-RAG / routing belong entirely in `intelligence`.
- **Web-search fallback in CRAG.** Closed-data assumption.
- **BigQuery-backed (multi-bucket) Iceberg catalog / self-hosted Polaris.** Ruled out for now — the GCP-native gcs-bucket catalog (1 bucket : 1 catalog) meets current needs; revisit only if multi-bucket sharing becomes a real requirement (see `lakehouse-iceberg-plan` repo memory).

---

## Future-proofing principles (don't compromise)

1. **Version every artefact.** `knowledge/shared/provenance.py` — swapping a parser or embedder must trigger clean reprocessing.
2. **Abstract model choices.** Embedding model, reranker, LLMs all loaded from config + Secret Manager. No hardcoded model names.
3. **Reserve enums early.** `chunk_kind` enum from day one so future indexes don't need schema migrations.
4. **The manifest is the contract.** New retrieval surfaces appear as new manifest entries — `intelligence` never reaches into `knowledge` internals.
5. **Eval-first.** No retrieval/prompting/routing change ships without an eval delta — enforced via `intelligence/evaluation/results/OPPORTUNITIES.md`.
6. **Cost ceilings per request.** Hard limits on agent iterations, token budget, rerank calls. Cost is a first-class config concern (see intelligence ADR-0019).
7. **Loose coupling.** Repos communicate via well-defined interfaces (APIs, GCS paths, manifest, vector DB). No reaching into internals.

---

## Notes

- Repo-local TODOs (small bugs, refactors that don't cross repo boundaries) stay in the respective repo. Anything touching ≥2 repos or affecting the platform roadmap belongs here.
- Intelligence retrieval/routing/generation/eval quality opportunities live exclusively in `intelligence/evaluation/results/OPPORTUNITIES.md` (+ `OPPORTUNITIES-ARCHIVE.md` for closed rows) — only cross-repo or human-blocked rows are summarised here, and only to point back at the register, not to duplicate its detail.
- Session-scoped implementation history (what changed, why, gotchas hit) lives in this assistant's repo memory (`wire-up-status.md`, `intelligence-improve-state.md`, `lakehouse-iceberg-plan.md`, etc.) — useful for resuming work, but not a substitute for this file or the register.
- When closing an item, prefer striking it and linking to the PR/commit rather than deleting, so the history is recoverable.
