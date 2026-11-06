# Go-to-prod plan

- **Status (2026-11-07):** prod is completely undeployed. Nothing below has started.
- **Held-out eval set:** on hold, including for prod (user, 2026-11-07). Any prod claim rests on the visible-case eval only; say so wherever results are quoted.

## Semantic layer (from [ossie-semantic-layer.md](ossie-semantic-layer.md))

- **infra:** apply the semantics pieces to prod (`{project}-semantics` bucket, `biglake-sa-semantics-prod`, drafter SA if used, catalog/data/intelligence `bundles/` and `evidence/` grants). Only test has been applied (infra `0db5a24`, #64-#66, #72/#75/#76).
- **GitHub:** `prod` environments with protection rules in semantics, etl and catalog (currently none exist); WIF secrets + `PROJECT_ID` per env; the GitHub App is already org-installed.
- **semantics:** first prod publish (`publish.yml` dispatch to prod), then catalog `jobs=curate` against prod OM.
- **intelligence:** set `SEMANTIC_BUNDLE_URI` and choose the flags (`ENABLE_SEMANTIC_GROUNDING`, `ENABLE_SEMANTIC_COMPILER`) via infra `extra_env`; test runs both ON.
- **ADR-0028** (intelligence) is `provisional`; decide its status before enabling the compiler in prod.

## Data (etl)

- etl BUG-0002 (landing prefix collision) and BUG-0003 (FULL_SNAPSHOT stacked snapshots) are fixed in code, so a fresh prod ingest is clean by construction. Still verify after the first prod load: every table has one `meta_flow_ref`, and every FULL_SNAPSHOT table has one `biglake_ingestion_datetime_utc` batch (the profile run reports both).
- INCREMENTAL tables: BUG-0003's INCREMENTAL exposure (two landing files for the same period in one batch) is not fixed yet. Fix before the prod backfill, or backfill each INCREMENTAL dataset one landing file at a time.

## Order

1. infra prod apply (Terraform via GHA) → 2. GitHub prod environments + protection → 3. etl prod deploy + first ingest → profile check → 4. catalog prod sync → 5. semantics prod publish → 6. intelligence prod `extra_env` → 7. eval against prod (visible cases).
