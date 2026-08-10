# Copilot Instructions — BigLake Platform (Org-Wide)

These instructions apply across all BigLake repositories. Individual repos may have their own `.github/copilot-instructions.md` with repo-specific conventions.

## Platform Context

BigLake is a GCP-hosted data platform for Australian government datasets. See `ARCHITECTURE.md` in this repo for the full architecture diagram and repo responsibilities.

## Repository Map

| Repo | Role | Key Tech |
|---|---|---|
| **infra** | GCP infrastructure (Terraform) | Terraform, GCP |
| **etl** | Structured data pipelines (medallion) | Python, Prefect 3.x, DuckDB, GCS |
| **catalog** | Data catalog config & curation | OpenMetadata, Docker Compose |
| **api** | Flask BFF — unified API for the UI | Flask, Python |
| **ui** | Frontend web application | TBD |
| **knowledge** | Knowledge artifact ingestion & processing | Python, PDF parsing, embeddings |
| **intelligence** | RAG orchestration & agent logic | Python, LLM APIs, vector DB client |
| **.github** | Org-level docs & Copilot config | Markdown |

## Design Principles

1. **Loose coupling** — Repos do not depend on each other's internals. Communication is through well-defined interfaces (APIs, GCS paths, vector DB).
2. **Infra owns all deployment** — VMs, databases, vector DBs, networking are all provisioned from `infra/`. Application repos define config and logic, not infrastructure.
3. **Knowledge produces, intelligence consumes** — `knowledge` ingests and processes artifacts. `intelligence` retrieves and reasons over them. They are separate repos by design.
4. **BFF pattern** — The `ui` talks only to `api`. The `api` orchestrates all backend services (OpenMetadata API, GCS datalake queries, intelligence).
5. **Config-driven** — Pipelines and datasets are defined by YAML configuration, not hardcoded logic.
6. **AI for structure, SQL for logic** — In ETL, AI handles structural parsing (headers, boundaries). All transformation is deterministic DuckDB SQL.

## Cross-Repo Conventions

### GCP

- **Project:** Referenced via `GOOGLE_CLOUD_PROJECT` env var
- **Environment:** `ENV` env var (`test` or `prod`)
- **Storage:** GCS bucket via `GCS_BUCKET` env var
- **Secrets:** Google Secret Manager — never hardcode credentials
- **IAM:** Least-privilege service accounts, defined in `infra/modules/iam/`

### Python

- Python 3.8+ across all repos
- Use `yaml.safe_load()` for config (never `yaml.load()`)
- Use virtual environments (`.venv/`) per repo
- Security: follow OWASP Top 10 — no SQL injection, no hardcoded secrets, sanitize inputs

### Documentation

- **README.md** — single home for each documented area (pipeline, service, CI workflows). No more split `humans.md`/`robots.md` pair — that convention is retired org-wide.
- **documentation/adr/** — decisions (why X over Y, alternatives, consequences)
- **setup.md** — All manual steps to deploy the repo to a new GCP environment. Keep updated when adding secrets, variables, or infrastructure dependencies
- **DEVELOPMENT.md** (in `.github` repo) — Cross-repo local dev guide. Update it when any service's local setup changes (new env var, new prerequisite, changed port)
- In-file comments explain *why*, not *what*
- See `etl/.github/instructions/documentation.instructions.md` for the full documentation standard (applies org-wide)

### Security findings

- **Org-wide policy:** [`.github/SECURITY.md`](SECURITY.md) — disclosure + reporting.
- **Per-repo risk register:** `{repo}/documentation/security/risks.md` — single source of truth for security status. One row per finding (`SEC-NNN`), severity, status, owner.
- **Review reports:** `{repo}/documentation/security/reviews/YYYY-MM-DD-{topic}.md` — point-in-time snapshots. Each finding they produce becomes a row in `risks.md`.
- **ADRs vs risks:** decisions (e.g. "accept WebAuthn UV=preferred") → ADR. Findings (e.g. "rate limit not enforced") → `risks.md`.
- Do NOT duplicate security findings into API-contract gap tables (e.g. `BFF_API_REQUIREMENTS.md`) — link to a `SEC-NNN` row instead.
- When fixing a finding, change its status in `risks.md` in the **same commit** as the code fix and reference the commit SHA in the "Resolved" row.

### Naming

- Datasets and variables: `snake_case`
- GCS paths: `{subject}/{layer}/{source}/{dataset}/`
- Metadata columns: prefixed with `meta_` (e.g. `meta_source_file`, `meta_ingestion_epoch_utc`). The `_` prefix is reserved by BigLake/BigQuery for pseudo-columns and is rejected by the Lakehouse runtime catalog (etl ADR-0002).

## When Working Across Repos

- If a change in one repo affects another (e.g. new GCS path in `etl` needs a connector update in `catalog`), note the cross-repo dependency explicitly.
- Infrastructure changes (new VM, new database, new bucket) always go in `infra/`, never in application repos.
- API contracts between `api` and `ui` should be documented in `api/`.
- Vector DB schema changes affect both `knowledge` (writes) and `intelligence` (reads) — coordinate accordingly.
