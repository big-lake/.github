# BigLake Partner Data Access Guide

This guide explains how an authorised partner connects to BigLake's Iceberg tables using their own compute engine. No custom SDK or proprietary tooling is required — all connection patterns use standard Iceberg REST catalog interfaces.

## Overview

BigLake uses a two-plane architecture:

```
Control plane (PRIVATE)
  BigLake Lakehouse runtime catalog (Iceberg REST)
  https://biglake.googleapis.com/iceberg/v1/restcatalog
  ↓ resolves table metadata → data file locations

Data plane (direct GCS reads)
  gs://{project}-lakehouse/{namespace}/{table}/data/
  gs://{project}-raw/{namespace}/{table}/data/
```

You bring your own compute (pyiceberg, Spark, Trino, DuckDB) and point it at the REST catalog. Your engine fetches metadata from the catalog, then reads data files directly from GCS — data never flows through the BigLake API.

## Prerequisites

You will receive from BigLake:

| Item | Description |
|---|---|
| GCP service account key or OAuth2 client credentials | For authenticating to the catalog REST endpoint |
| GCP project ID | For quota attribution (`x-goog-user-project` header) |
| Catalog warehouse URI | `gs://{project}-lakehouse` (silver/gold data) and/or `gs://{project}-raw` (bronze data) |
| Namespace and table names | e.g. `australian_taxation_office.individual_income_and_taxation` |

---

## Connection recipes

### Python — pyiceberg

```python
import google.auth
import google.auth.transport.requests
from pyiceberg.catalog import load_catalog

# --- Auth ---
# Assumes GOOGLE_APPLICATION_CREDENTIALS is set to your SA key file,
# or you are running on a GCE VM with the SA attached.
credentials, project_id = google.auth.default(
    scopes=["https://www.googleapis.com/auth/cloud-platform"]
)
credentials.refresh(google.auth.transport.requests.Request())
token = credentials.token

# --- Catalog ---
catalog = load_catalog(
    "biglake",
    **{
        "type": "rest",
        "uri": "https://biglake.googleapis.com/iceberg/v1/restcatalog",
        "warehouse": "gs://{project}-lakehouse",   # replace {project}
        "token": token,
        "header.x-goog-user-project": project_id, # required for quota attribution
    },
)

# --- Read a table ---
table = catalog.load_table("australian_taxation_office.individual_income_and_taxation")
df = table.scan().to_pandas()
print(df.head())
```

> **Note:** `header.x-goog-user-project` is a BigLake-specific requirement — without it you will get a 403 even with a valid token.

### Python — pyiceberg with time travel

```python
# Read a specific snapshot
snapshot_id = 1234567890   # get from catalog.load_table(...).snapshots()
scan = table.scan(snapshot_id=snapshot_id)
df = scan.to_pandas()
```

### Python — pandas/pyarrow via DuckDB

```python
import duckdb
import google.auth
import google.auth.transport.requests

credentials, project_id = google.auth.default(
    scopes=["https://www.googleapis.com/auth/cloud-platform"]
)
credentials.refresh(google.auth.transport.requests.Request())
token = credentials.token

con = duckdb.connect()
con.install_extension("iceberg")
con.load_extension("iceberg")

# Get the metadata file location from the catalog (one-time lookup)
# Replace with your actual metadata file path (from catalog /tables/{tbl} response)
metadata_uri = "gs://{project}-lakehouse/australian_taxation_office/individual_income_and_taxation/metadata/00001-{uuid}.metadata.json"

df = con.execute(f"SELECT * FROM iceberg_scan('{metadata_uri}') LIMIT 100").df()
```

> **Tip:** To discover the current metadata file path without pyiceberg, call the catalog REST directly:
> ```
> GET https://biglake.googleapis.com/iceberg/v1/restcatalog/v1/{prefix}/namespaces/australian_taxation_office/tables/individual_income_and_taxation
> Authorization: Bearer {token}
> x-goog-user-project: {project}
> ```
> The response contains `metadata-location`.

### Apache Spark

```python
# spark-defaults.conf or SparkSession builder
spark = (
    SparkSession.builder.appName("biglake-partner")
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
    .config("spark.sql.catalog.biglake", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.biglake.catalog-impl", "org.apache.iceberg.rest.RESTCatalog")
    .config("spark.sql.catalog.biglake.uri", "https://biglake.googleapis.com/iceberg/v1/restcatalog")
    .config("spark.sql.catalog.biglake.warehouse", "gs://{project}-lakehouse")
    .config("spark.sql.catalog.biglake.token", token)          # short-lived OAuth2 token
    .config("spark.sql.catalog.biglake.header.x-goog-user-project", project_id)
    .getOrCreate()
)

df = spark.table("biglake.australian_taxation_office.individual_income_and_taxation")
df.show()
```

### Trino / Presto

```properties
# etc/catalog/biglake.properties
connector.name=iceberg
iceberg.catalog.type=rest
iceberg.rest-catalog.uri=https://biglake.googleapis.com/iceberg/v1/restcatalog
iceberg.rest-catalog.warehouse=gs://{project}-lakehouse
iceberg.rest-catalog.security=OAUTH2
iceberg.rest-catalog.oauth2.token={your_oauth2_token}
iceberg.rest-catalog.http-header.x-goog-user-project={project}
```

```sql
SELECT * FROM biglake.australian_taxation_office.individual_income_and_taxation LIMIT 100;
```

---

## Available tables

| Namespace | Table | Layer | Description |
|---|---|---|---|
| `australian_taxation_office` | `individual_income_and_taxation` | silver | ATO individual income and tax statistics by postcode and financial year |
| `parliamentary_budget_office` | `cash_flow_aggregates` | silver | PBO historical cash flow aggregates |
| `parliamentary_budget_office` | `fiscal_accrual_aggregates` | silver | PBO fiscal/accrual aggregates |
| `parliamentary_budget_office` | `expenses_by_function` | silver | PBO expenses by function and sub-function |
| `parliamentary_budget_office` | `revenue_by_head_accrual` | silver | PBO revenue by head (accrual) |
| `parliamentary_budget_office` | `receipts_by_head_cash` | silver | PBO receipts by head (cash) |
| `parliamentary_budget_office` | `debt_net_interest_and_net_worth` | silver | PBO debt, net interest, and net worth |
| `parliamentary_budget_office` | `historical_cash_flow` | silver | PBO historical cash flow (since 1953-54) |
| `parliamentary_budget_office` | `historical_debt` | silver | PBO historical debt |
| `parliamentary_budget_office` | `institutional_sectors` | silver | PBO institutional sectors |
| `parliamentary_budget_office` | `medium_term_projections` | silver | PBO medium-term projections |
| `parliamentary_budget_office` | `general_government_financial_statements` | silver | PBO general government financial statements |

> **Bronze tables** (raw extraction, pre-transformation) are also available in the `-raw` catalog. Use `warehouse: gs://{project}-raw` and the same namespace/table names. Contact BigLake if you need bronze-level access.

---

## Discovering table properties

Each Iceberg table carries standard properties identifying its position in the medallion:

```python
table = catalog.load_table("australian_taxation_office.individual_income_and_taxation")
print(table.properties)
# {
#   "biglake.layer": "silver",
#   "biglake.source": "individual_income_and_taxation",
#   "biglake.topic": "taxation",
#   "write.metadata.metrics.default": "full",
#   ...
# }
```

---

## Auth notes

- **Token lifetime:** Google OAuth2 tokens expire after 1 hour. For long-running jobs, refresh before catalog instantiation or implement token refresh in your job's checkpoint logic.
- **GCS data reads:** Data files are read directly by your compute engine using the same GCP credentials. The SA needs `roles/storage.objectViewer` on the lakehouse bucket.
- **Catalog access:** The SA needs `roles/biglake.user` (read-only catalog access) on the GCP project.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `403 USER_PROJECT_DENIED` | Missing `x-goog-user-project` header | Pass `header.x-goog-user-project: {project_id}` to `load_catalog` |
| `401 ACCESS_TOKEN_TYPE_UNSUPPORTED` | Passing a service account key directly (not a bearer token) | Use `google.auth.default()` + `credentials.refresh()` to get a bearer token |
| `404 NOT_FOUND` on `/v1/config` | Wrong `warehouse` value | Must be exactly `gs://{bucket}` — no trailing path |
| `400 INVALID_ARGUMENT` on namespace | Multi-level namespace (e.g. `fiscal.parliamentary_budget_office`) | BigLake only supports single-level namespaces — use `parliamentary_budget_office` directly |
| `AccessDeniedException` on GCS data files | SA missing storage role | Ensure `roles/storage.objectViewer` on the lakehouse bucket |
