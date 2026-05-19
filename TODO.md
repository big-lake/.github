# BigLake Platform — Org-Wide TODO

Last updated 2026-05-19 (Stage A api-side complete). This is the single source of truth for cross-repo planning. Repo-local TODOs should only cover repo-internal task tracking; anything that crosses repo boundaries lives here.

---

## Current state snapshot

| Repo | Status |
|---|---|
| **infra** | Test env up. Intelligence VM deployed (`e2-medium`). Read-only pgvector role for `intelligence` not yet provisioned. Articles persistence layer not designed. |
| **etl** | Bronze/silver/gold flows running for treasury + ATO + DSS sources via Prefect. |
| **catalog** | OpenMetadata deployed. Curation status of gold-layer datasets in test env unverified — needs check before `api` wires to it. |
| **api** | Stage A complete (server side). Real intelligence call with ID-token s2s auth, chat sessions persisted to SQLite (WAL + migrations), real OM catalog adapter, DuckDB `/query` with sqlglot read-only enforcement, SQLite-backed users + werkzeug password hashing. Pending: Litestream backup, OM custom-property curation in `catalog`. |
| **ui** | Fully built against mocks. 20 open gaps in [`api/API_REQUIREMENTS.md`](../api/API_REQUIREMENTS.md). |
| **knowledge** | Production through indexing (Flows 1–4). P0 eval harness built, awaiting curated golden set. P1–P5 pending. |
| **intelligence** | Phase 1 walking skeleton live (`/health`, `/retrieve`, `/chat`, hybrid + Cohere rerank v3.5 + Gemini synthesis). No eval baseline. Phases 2–6 on roadmap. |

---

## Roadmap — prioritised

**Strategy: ship to prod first, improve later.** Get a working end-to-end product (thin slice → articles → prod deploy) before investing in RAG quality work. The eval framework is built and ready, but we'll only start using it once there's a real user-facing product in prod that we want to measurably improve.

### Stage A — Thin end-to-end slice (do first)

Goal: a working `ui → api → intelligence → answer-in-browser` and `ui → api → OpenMetadata` path. Validates contracts before more surface area is added.

> Service-layer design choices captured in [`api/documentation/adr/`](../api/documentation/adr/) (ADRs 0001–0009).

- [x] **api/intelligence_client.py → real intelligence call.** Done. HTTP call to `intelligence /chat`, GCP ID-token auth on both sides per [ADR-0003](../api/documentation/adr/0003-s2s-auth-gcp-id-tokens.md), citations + trace passed through. `INTELLIGENCE_URL` + `INTELLIGENCE_AUDIENCE` wired in `infra/modules/{api,intelligence}/startup.sh`. (`api`, `intelligence`)
- [x] **Chat sessions in SQLite** per [ADR-0004](../api/documentation/adr/0004-chat-sessions-in-sqlite.md) + [ADR-0010](../api/documentation/adr/0010-sqlite-litestream-then-cloudsql.md). Done server-side: `app/db/` repo layer, migration runner, `0001_init.sql` (users, conversations, messages), WAL + foreign keys, `/chat` returns `conversation_id` + `message_id`, accepts `conversation_id` to continue. UI side still pending — see Stage A UI item. (`api`)
- [ ] **Litestream backup of SQLite** per [ADR-0010](../api/documentation/adr/0010-sqlite-litestream-then-cloudsql.md). Add Litestream install + systemd unit to `infra/modules/api/startup.sh`. Restore-on-boot before api starts. Add `gs://{bucket}/api/sqlite-backups/{env}/` prefix with lifecycle rules. Add Litestream staleness check to api `/health`. (`infra`, `api`)
- [x] **api/catalog_client.py → real OM API call.** Done. Real OM adapter with paging cursor, `_om_table_to_api()` mapping per [ADR-0008](../api/documentation/adr/0008-catalog-shape-from-om.md): `source` from `DataSource.<X>` tag, `logo` / `links` / `mostRecentData` from `extension` custom properties, lineage flattened from `/lineage/table/{id}`. Falls back to safe defaults if properties are missing. (`api`)
- [ ] **Catalog prerequisite** — extend `catalog/curation/` YAML schema with `logo`, `links`, `most_recent_data`. Update ingestion to register these as OM custom properties on table entities. Define custom property types on `Table` entity (one-time, per env). Tag each table with a `DataSource.<X>` tag. Curate `individual_income_and_taxation`, `social_security_recipients`, `fiscal_budget`. Tracked as gap #24 in `api/BFF_API_REQUIREMENTS.md`. (`catalog`)
- [x] **api/query_service.py → real GCS+DuckDB.** Done. Per [ADR-0006](../api/documentation/adr/0006-duckdb-connection-and-om-views.md), [ADR-0007](../api/documentation/adr/0007-sql-read-only-enforcement.md), [ADR-0009](../api/documentation/adr/0009-query-response-shape.md). One DuckDB connection per gunicorn worker, sqlglot AST-level read-only enforcement (rejects multi-statement, non-SELECT, file-read funcs outside `gs://{bucket}/`), DuckDB session locked down (`enable_external_access=false`, `lock_configuration=true`, `statement_timeout`), two-query `totalRows` with opt-out, `columnTypes` mapping, NaN/Inf + date-safe JSON provider. OM view registration is a no-op until catalog curation lands (gap #25). (`api`)
- [x] **Session persistence (server side).** Done. Users moved from in-memory dict to SQLite `users` table, werkzeug pbkdf2 hashing, admin seeded at boot from `ADMIN_PASSWORD` secret, `/me` returns the current user. UI-side persistence (localStorage + `getMe()` on init) is the remaining work — see Stage A UI item below. Resolves the server half of API gap #1. (`api`)
- [ ] **UI mock removal + session persistence.** Wire Dashboard `executeQuery()` to real API; wire prompt bar `send()` to `/chat`; persist auth token (localStorage) + call `getMe()` on app init; store `conversation_id` between chat turns. Remove `mockQueryResult()` and `mockCatalog`. Resolves API gaps #1 (UI half), #2, #3, #8, #21. (`ui`)
- [ ] **Chat error handling + citations rendering.** API returns `{error, kind}` on failure and `citations[]` + `trace` on success. UI needs to render errors gracefully, render citations as references, and optionally show a collapsible "reasoning" panel for `trace`. Resolves API gaps #5, #22. (`ui`)

### Stage B — Articles persistence

Goal: unlock the UI's core value prop (drafting + publishing data articles). Pure `api`+`ui` work, doesn't touch RAG.

- [ ] Design schema (SQLite in `api` is the default per architecture). (`api`)
- [ ] Implement `POST /articles`, `GET /articles`, `GET /articles/{id}`, `PATCH /articles/{id}`, `DELETE /articles/{id}`, `POST /articles/{id}/publish`. (`api`)
- [ ] Add `createArticle`, `getArticles`, `getArticle`, `updateArticle`, `deleteArticle`, `publishArticle` to `ui/src/api/client.js`. Replace in-memory state. Resolves API gaps #7, #14, #19, #20. (`ui`)
- [ ] XSS sanitization for text cell HTML on load (server-side or DOMPurify). Resolves API gap #15. (`api` or `ui`)
- [ ] Cell share endpoint design + implementation. Resolves API gap #12. (`api`, `ui`)

### Stage C — Prod deploy

Goal: get the whole platform running in `prod` (currently only `test` is live end-to-end).

- [ ] Verify `infra` prod env is provisioned to parity with test (Prefect VM, OM VM, intelligence VM, knowledge_db, storage, IAM). (`infra`)
- [ ] Populate prod Secret Manager values via GCP Console UI (Cohere API key, pgvector DSN, OM admin, etc.). (`infra`, human)
- [ ] Run all `etl` pipelines against prod to seed gold-layer datasets in `gs://big-lake-prod-analytics-data`. (`etl`)
- [ ] Run all `knowledge` pipelines against prod to populate prod pgvector + Neo4j. (`knowledge`)
- [ ] Deploy `catalog` (OM ingestion + curation) to prod. (`catalog`)
- [ ] Deploy `api` + `ui` + `intelligence` to prod via GHA. (`api`, `ui`, `intelligence`)
- [ ] Smoke-test end-to-end in prod: login → catalog browse → query → chat. 

---

## Improvement backlog (post-prod)

Everything below waits until the platform is live in prod. Once it is, the eval harness in `knowledge/evaluation/` becomes the gate for all RAG quality work.

### Stage D — Eval baseline

Unlocks measurable iteration on intelligence Phases 2+ and knowledge P1+. Required by `intelligence/DESIGN.md` ("no retrieval/prompting/routing change ships without an eval delta").

- [ ] Run `generate_eval_candidates` deployment for `treasury`/`budget_release_papers` against prod intelligence. (`knowledge`)
- [ ] Curate ~50–100 questions into `knowledge/evaluation/golden_sets/treasury/budget_release_papers/v1.yaml`. (`knowledge`, human curation)
- [ ] Run `evaluate_retrieval` deployment to establish baseline Recall@k, MRR, HitRate sliced by `query_class`. (`knowledge`)
- [ ] Record baseline numbers in `intelligence/documentation/` and link from this TODO.

### Stage E — Intelligence Phase 2–4

Each step gated by an eval delta against the Stage D baseline.

- [ ] **Phase 2 — Query rewriting + small-to-big.** Multi-query expansion (Gemini Flash), parent-chunk expansion via `parent_id`. Wire RAGAS evals. (`intelligence`)
- [ ] **Phase 3 — Multi-tier retrieval.** Add `search_propositions`, `search_questions`, `search_summaries` tools as `knowledge` publishes them in the manifest. (`intelligence`)
- [ ] **Phase 4 — Routing.** Lightweight classifier (Gemini Flash few-shot) labelling queries `factoid | broad | relational | multi_hop | comparison` → tool selection. (`intelligence`)
- [ ] **Cost ceilings.** Hard caps on agent-loop iterations, LLM token budget, rerank calls. (`intelligence`)

### Stage F — Knowledge P1–P3

Each gated by the same eval harness. P1 is the cheapest quality win.

- [ ] **P1 — Hypothetical question generation.** Extend `enrichment/utils/schema.py` with `hypothetical_questions: list[str]`, update prompt, add `knowledge_questions` table (or `kind='hypothetical_question'` rows), bump `ENRICH_VERSION`, add to `pipeline.yaml`. (`knowledge`)
- [ ] **P2 — Document + section summaries (RAPTOR-lite).** One Gemini Flash call per section + per document. Store as `kind='summary'` rows with `summary_level` (`section` | `document`). Add `chunk_kind` enum migration. Bump `ENRICH_VERSION` and `INDEX_VERSION`. Surface in manifest. (`knowledge`)
- [ ] **P3 — Finalise retrieval-surface manifest.** Define JSON schema in `knowledge/shared/manifest_schema.py`. Fields: `index_version`, `embedding_model`, `embedding_dim`, pgvector table list with column types and filter dimensions, Neo4j node/edge schema, summary tier availability, hypothetical question tier availability. Validate on write. Document and link from `intelligence/`. (`knowledge`)

### Stage G — Intelligence Phase 5–6

- [ ] **Phase 5 — Graph traversal.** `traverse_graph` Neo4j Cypher tool + relational routing path. Add Neo4j read-only auth to `intelligence` SA in `infra/modules/iam/intelligence.tf`. (`intelligence`, `infra`)
- [ ] **Phase 6 — Agent loop + CRAG.** Multi-hop / comparison handling, corrective retrieval, faithfulness gate (second LLM call). (`intelligence`)

### Stage H — Infra hardening

- [ ] Read-only pgvector role for `intelligence` (new secret in Secret Manager; populate via GCP Console UI). (`infra`)
- [ ] Cohere API key secret (already created? — verify). (`infra`)
- [ ] Cost guards / alerts on Vertex + Cohere spend. (`infra`)
- [ ] Internal service auth wiring (see Stage A item). (`infra`)

### Stage I — Cross-repo nice-to-haves

- [ ] **Lineage UI.** `/catalog/lineage` exists in client.js but no UI. Build a simple lineage view. Resolves API gap #6. (`api`, `ui`)
- [ ] **Knowledge documents in OpenMetadata.** Only after defining the user story (unified discovery? lineage from structured datasets to source PDFs?). Likely a new OM connector registering documents as catalog assets. Currently unmotivated. (`catalog`, `knowledge`)
- [ ] **Server-side execTime** in `/query` response. Resolves API gap #17. (`api`)
- [ ] **Dashboard.vue extraction.** Tab bar + prompt bar components. Resolves API gap #10. (`ui`)

### Stage J — Optional / parked

- [ ] **knowledge P4 — Semantic chunk boundaries.** Embedding-similarity boundary detector. Opt-in via `chunking.semantic_boundaries.enabled`. Don't build until eval shows token-count splits are the bottleneck. (`knowledge`)
- [ ] **knowledge P5 — VLM parser tier.** Keep dispatcher pluggable in `ingestion/utils/pdf_tasks.py`; reserve `parser.vlm.provider` config. Don't migrate until eval flags PyMuPDF+Docling as a specific-source bottleneck. (`knowledge`)
- [ ] **HyDE** in intelligence — only behind per-source config flag with eval evidence. (`intelligence`)
- [ ] **Self-RAG** — revisit only if eval shows classifier is the bottleneck. (`intelligence`)

### Won't do

- **ColBERT / late-interaction index.** Lowest ROI given scale and cost; revisit only if reranking proves insufficient downstream.
- **Agentic logic inside `knowledge`.** By design. Ingestion stays deterministic and reprocessable. CRAG / Self-RAG / routing belong entirely in `intelligence`.
- **Web-search fallback in CRAG.** Closed-data assumption.

---

## Future-proofing principles (don't compromise)

1. **Version every artefact.** `knowledge/shared/provenance.py` — swapping a parser or embedder must trigger clean reprocessing.
2. **Abstract model choices.** Embedding model, reranker, LLMs all loaded from config + Secret Manager. No hardcoded model names.
3. **Reserve enums early.** `chunk_kind` enum from day one so future indexes don't need schema migrations.
4. **The manifest is the contract.** New retrieval surfaces appear as new manifest entries — `intelligence` never reaches into `knowledge` internals.
5. **Eval-first.** No retrieval/prompting/routing change ships without an eval delta.
6. **Cost ceilings per request.** Hard limits on agent iterations, token budget, rerank calls. Cost is a first-class config concern.
7. **Loose coupling.** Repos communicate via well-defined interfaces (APIs, GCS paths, manifest, vector DB). No reaching into internals.

---

## Notes

- Repo-local TODOs (small bugs, refactors that don't cross repo boundaries) stay in the respective repo. Anything touching ≥2 repos or affecting the platform roadmap belongs here.
- When closing an item, prefer striking it and linking to the PR/commit rather than deleting, so the history is recoverable.
