# BigLake

An open data platform for Australian government datasets — queryable with SQL, searchable with AI.

BigLake ingests and transforms public government data (ATO, DSS, Parliamentary Budget Office, and others) through a structured ETL pipeline, indexes unstructured publications for AI-powered retrieval, and serves everything through a unified web interface.

## What it does

- **ETL pipelines** — ingest, clean, and transform structured datasets into analysis-ready parquet files
- **Data catalog** — browse datasets, column schemas, and lineage via OpenMetadata
- **SQL query interface** — run SQL directly over the data lake in your browser
- **AI assistant** — ask questions in plain language, get cited answers via RAG
- **Document editor** — compose and publish data-driven articles and analysis

## Architecture

```
External Sources (ATO, DSS, PBO, ...)
         │
         ├──────────────────────────┐
         ▼                          ▼
      [etl]                    [knowledge]
  structured data             unstructured docs
  (parquet → GCS)             (text → vector DB)
         │                          │
         ▼                          ▼
     [catalog]                [intelligence]
   (OpenMetadata)              (RAG / agents)
         │                          │
         └──────────┬───────────────┘
                    ▼
                  [api]
              (Flask BFF)
                    │
                    ▼
                  [ui]
```

## Repositories

| Repo | Role |
|---|---|
| [`infra`](https://github.com/big-lake/infra) | GCP infrastructure (Terraform) |
| [`etl`](https://github.com/big-lake/etl) | Data ingestion & transformation pipelines |
| [`catalog`](https://github.com/big-lake/catalog) | Data catalog config & curation (OpenMetadata) |
| [`api`](https://github.com/big-lake/api) | Flask BFF — unified API for the UI |
| [`ui`](https://github.com/big-lake/ui) | Frontend web application (Vue 3) |
| [`knowledge`](https://github.com/big-lake/knowledge) | AI knowledge base ingestion & processing |
| [`intelligence`](https://github.com/big-lake/intelligence) | RAG orchestration & agent logic |
| [`.github`](https://github.com/big-lake/.github) | Org-level docs & platform architecture |

## Getting started

To run the API and UI locally (no GCP required): see [DEVELOPMENT.md](.github/blob/main/DEVELOPMENT.md).

For deploying a new GCP environment, see the `SETUP.md` in each repo.

## Contributing

See [CONTRIBUTING.md](.github/blob/main/CONTRIBUTING.md). Contributions are welcome — bug reports, fixes, new data sources, and documentation improvements.

## Copyright

© 2026 BigLake contributors. All rights reserved.
Source is publicly viewable and contributions are welcome. Reproduction or redistribution requires explicit permission — see [COPYRIGHT](.github/blob/main/COPYRIGHT).
