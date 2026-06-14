# BigLake Platform Architecture

## Overview

Big Lake is a data platform for ingesting, transforming, cataloguing, and serving Australian government datasets. It runs on Google Cloud Platform with a self-hosted orchestration layer.

## Repositories

```
biglake/
├── infra/           # GCP infrastructure (Terraform)
├── etl/             # Data ingestion & transformation pipelines
├── catalog/         # Data catalog config & curation (OpenMetadata)
├── api/             # Flask BFF — single API surface for the UI
├── ui/              # Frontend web application
├── knowledge/       # AI knowledge base ingestion & processing
├── intelligence/    # RAG orchestration & agent logic
└── .github/         # Org-level docs, architecture, Copilot instructions
```

## Repo Responsibilities

### infra

**Role:** Infrastructure-as-code for the entire platform.

- Terraform modules for GCP resources (VMs, networking, IAM, storage, secrets)
- Deploys all serving infrastructure: Prefect VM, OpenMetadata VM, vector DB, etc.
- Multi-environment support (test / prod) with separate state buckets
- GitHub Actions workflows for plan/apply

**Principle:** Infra deploys everything. No application logic lives here.

### etl

**Role:** Structured data ingestion and transformation through a medallion pipeline.

- Ingests Australian government datasets (ATO, DSS, Parliamentary Budget Office, etc.)
- Transforms through bronze → silver → gold layers using DuckDB
- Orchestrated by Prefect 3.x (self-hosted on GCP VM)
- Event-driven triggers chain stages: ingestion → bronze → silver
- Config-driven: one YAML file per dataset

**Tech:** Python, Prefect 3.x, DuckDB, GCS, openpyxl, Vertex AI Gemini (structural parsing only)

### catalog

**Role:** Data discovery and metadata management via OpenMetadata.

- OpenMetadata server config and Docker Compose deployment
- Iceberg REST connector configuration (reads from BigLake Lakehouse runtime catalog)
- Dataset curation: column schemas, domain tagging, tier, certification
- Ingestion scripts run from GHA

**Tech:** OpenMetadata 1.12, Docker Compose, MySQL, Elasticsearch

**Note:** The OM API is the interface. Flask (in `api`) orchestrates calls to it.

### api

**Role:** Backend-for-frontend (BFF) — single API surface for the UI.

- Flask application providing a unified API
- Orchestrates calls to: OpenMetadata API, intelligence services
- Queries Iceberg tables directly (DuckDB `iceberg_scan` views over the BigLake Lakehouse runtime catalog) to serve data to the UI
- SQLite for application data (users, settings, preferences)
- Shields the frontend from backend service topology

**Tech:** Flask (Python), SQLite

### ui

**Role:** Frontend web application for data exploration and dashboards.

- Consumes the Flask BFF exclusively (no direct backend calls)

### knowledge

**Role:** AI knowledge base — ingestion and processing of knowledge artifacts.

- PDF parsing (government publications, budget statements, etc.)
- Web scraping for definitions and reference data
- Chunking and embedding generation
- Writes processed artifacts to vector DB (deployed by `infra`)

**Principle:** Produces knowledge. Does not own serving infrastructure or RAG logic.

### intelligence

**Role:** RAG orchestration and agent logic.

- Retrieval-augmented generation workflows
- Prompt chains and agent orchestration
- Reads from vector DB, calls LLM APIs
- Decoupled from knowledge ingestion (consumes what `knowledge` produces)

**Principle:** Consumes knowledge, reasons over it. Separate from ingestion by design.

### .github

**Role:** Org-level documentation and configuration.

- Platform architecture documentation (this file)
- Org-wide Copilot instructions
- Shared CI/CD workflows (future)

## Design Principles

1. **Loose coupling** — Each repo has a single responsibility. No repo directly depends on another's internals.
2. **Infra owns deployment** — All serving infrastructure (VMs, databases, vector DBs) is provisioned from `infra`. Application repos define config, not infrastructure.
3. **Knowledge produces, intelligence consumes** — Knowledge ingestion and RAG logic are separate concerns in separate repos.
4. **BFF pattern** — The UI talks to one API (`api`), which orchestrates all backend services.
5. **Config-driven pipelines** — ETL pipelines are defined by YAML config, not code changes.
6. **AI for structure, SQL for logic** — In ETL, AI handles structural parsing (finding headers, data boundaries). All transformation logic is deterministic SQL in DuckDB.

## Data Flow

```
External Sources (APIs, PDFs, web)
        │
        ▼
   ┌─────────┐     ┌───────────┐
   │   etl   │     │ knowledge │
   │(structured)  │(unstructured)
   └────┬────┘     └─────┬─────┘
        │                │
        ▼                ▼
   ┌─────────┐     ┌──────────┐
   │   GCS   │     │ Vector DB│
   │(Iceberg)│     │          │
   └────┬────┘     └─────┬────┘
        │                │
        ├────────┐       │
        ▼        │       ▼
   ┌─────────┐   │  ┌──────────────┐
   │ catalog │   │  │ intelligence │
   │ (om:8585)   │  │ (RAG/agents) │
   └────┬────┘   │  └──────┬───────┘
        │        │         │
        └────────┼─────────┘
                 ▼
            ┌─────────┐
            │   api   │  (Flask BFF, :8000)
            │         │
            │ SQLite  │  (users, settings)
            └────┬────┘
                 │
        ┌────────▼────────┐
        │  Global HTTPS LB │  (api.biglake.au / app.biglake.au / om.biglake.au)
        │  Managed cert    │  Google-managed TLS, auto-renew
        └────────┬─────────┘
                 │
            ┌────▼────┐
            │   ui    │  (:80)
            └─────────┘
```

## Infrastructure (GCP)

| Service | Purpose | Provisioned By |
|---|---|---|
| GCS — raw bucket | Landing files + bronze Iceberg tables | `infra/modules/storage/` |
| GCS — lakehouse bucket | Silver + gold Iceberg tables; also the BigLake gcs-bucket catalog | `infra/modules/storage/` |
| Compute Engine | Prefect server, OpenMetadata server, api VM, ui VM | `infra/modules/{prefect,openmetadata,api,ui}/` |
| Global HTTPS LB | TLS termination + host-based routing for `api.biglake.au`, `app.biglake.au`, `om.biglake.au` | `infra/modules/lb/` |
| Cloud DNS | Managed zone `biglake-au-{env}`; A records for all four subdomains → LB IP | `infra/modules/dns/` |
| Secret Manager | API keys, HMAC keys, DB credentials, admin password | `infra/modules/iam/` |
| SQLite | Application data (users, settings) — co-located with `api` | `api/` (file-based, no infra needed) |
| Cloud SQL / Vector DB | Metadata store, vector embeddings | `infra/` (TBD for vector DB) |
| IAM | Service accounts with least-privilege roles; Workload Identity Federation for GHA | `infra/modules/iam/` |
| VPC | Private networking, firewall rules — all VMs internal-only after Phase 2 LB rollout | `infra/modules/network/` |

## Environments

Two GCP projects, both in `australia-southeast2`:

| | Test | Prod |
|---|---|---|
| **Project ID** | `big-lake-test-490405` | `big-lake-prod` |
| **Region / Zone** | `australia-southeast2` / `-b` | `australia-southeast2` / `-a` |
| **Raw bucket** (landing + bronze Iceberg) | `big-lake-test-490405-raw` | `big-lake-prod-raw` |
| **Lakehouse bucket** (silver + gold Iceberg, catalog) | `big-lake-test-490405-lakehouse` | `big-lake-prod-lakehouse` |
| **TF state bucket** | `biglake-tf-state-test` | `biglake-tf-state-prod` |
| **Deploy SA** | `terraform-deploy@big-lake-test-490405.iam.gserviceaccount.com` | `terraform-deploy@big-lake-prod.iam.gserviceaccount.com` |
| **Storage force_destroy** | `true` | `false` |

- **Environment variable:** `ENV` (`test` or `prod`) — used by all application repos
- **Terraform:** Applied exclusively via GitHub Actions using Workload Identity Federation (no local runs)
- **WIF pool:** `github-pool` per project, with a single org-wide OIDC provider (`github-provider`, condition: `startsWith('big-lake/')`) — access controlled at the service account level
