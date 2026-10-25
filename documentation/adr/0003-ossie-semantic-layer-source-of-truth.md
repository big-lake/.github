# ADR-0003 — Apache Ossie, in a dedicated `semantics` repo, is the source of truth for semantic content; catalog keeps governance only

- **Status:** provisional (2026-10-25) — to be revisited after implementation; see *Revisit when*
- **Date:** 2026-10-25
- **Tags:** semantics, ontology, catalog, intelligence, openmetadata, cross-repo, curation, new-repo
- **Amends (on acceptance):** catalog ADR-0003 (curation in YAML — descriptions and `temporal` leave catalog), catalog ADR-0008 (SDMX-generated curation — ABS descriptions are generated in `semantics`)
- **Implementation plan:** [ossie-semantic-layer.md](../plans/ossie-semantic-layer.md)

## Context

BigLake joins structured data (Iceberg) and unstructured knowledge
(pgvector/Neo4j) but has no **semantic layer or ontology** — no
machine-readable account of what data means, how to calculate from it, or how
datasets relate. Semantic content today is prose only: `table.yaml`
`description` (mixed in with governance fields) and `cols.yaml`, written to
OpenMetadata (OM) by `patch_metadata.py`. intelligence's `data_query` fetches
that prose from OM per request and the LLM writes SQL from it — no metrics,
joins or synonyms exist, so every question re-derives business logic.

OM is the governance and discovery plane (catalog ADR-0005, org ADR-0002).
Its Glossary and Metric entities are documentary: no join graph, nothing that
compiles to SQL.

[Apache Ossie](https://ossie.apache.org/) (incubating; formerly Open Semantic
Interchange) is a vendor-neutral YAML spec with a **core semantic model**
(datasets, fields, metrics, relationships, `ai_context`; multi-dialect SQL
expressions) and an **ontology** layer (`0.2.0.dev0`: concepts, n-ary
relationships with verbalizations, rules, RDF/OWL IRIs, mapping documents).

Alternatives considered:

1. **Curation YAML stays the source; Ossie generated from it.** Two files keep
   describing the same table. Rejected as the long-term shape.
2. **Ossie holds everything**, governance fields in `custom_extensions`.
   Rejected: those payloads are JSON strings (unreviewable in diffs), and Ossie
   has no owner/tier/certification/display-name concepts.
3. **Semantic layer inside OM.** Documentary, can't compile SQL, and puts OM
   on the intelligence request path (contrary to org ADR-0002). Rejected.
4. **Other formats** (MetricFlow, Cube, LookML, warehouse semantic views).
   Engine-specific or no ontology; Ossie converters keep them reachable.
5. **Ossie files inside the catalog repo.** Rejected: the semantic layer is a
   cross-cutting product consumed by intelligence, knowledge and potentially
   BI tools — not OM configuration. A separate repo gives it its own
   reviewers, release cadence and bot-PR stream, and keeps the new
   privileges (Vertex, evidence dispatch, semantics bucket) off catalog's
   identity.

## Decision

1. **Split ownership by kind of content, with no field owned twice:**

   | Owned by **`semantics`** (Ossie) | Owned by **catalog `table.yaml`** (governance, scope, presentation) |
   |---|---|
   | Dataset description | `namespace`, `table_name` (the scope allowlist, unchanged) |
   | Field (column) descriptions, `datatype`, `is_time` | `display_name` |
   | Temporal coverage (`date_field`, earliest / most-recent data) — evidence-derived | `domain`, `tier`, `certification`, `owner`, `tags` |
   | Metrics (dataset- and model-scoped) | `logo`, `links` |
   | Relationships (joins) | |
   | `ai_context` (instructions, synonyms, example questions) | |
   | Ontology concepts, relationships, rules; ontology ↔ model mappings | |

   `cols.yaml`, `table.yaml`'s `description` and `temporal` leave catalog in
   one change. *Why descriptions go to semantics:* they are both catalog
   documentation and the LLM's primary grounding; keeping them beside
   synonyms and `ai_context` avoids a dual source. *Why `temporal` moves:*
   coverage dates are facts about the data, maintained by hand today —
   semantics derives them from evidence. **Pilot exit-check:** if Ossie's
   verbosity makes description diffs materially harder to review than
   `cols.yaml`, fall back to "catalog keeps descriptions; semantics copies
   them in at build time" before migrating past the pilot.

2. **A dedicated `semantics` repo, with one-way dependencies.**

   ```
   catalog git (scope, domain, governance) ──┐
   etl git (ABS dsd_version)  ───────────────┼─► semantics (reads via sparse checkout at main)
                                            │        │ publish (environment input)
                                            │        ▼
                                            │   gs://{project}-semantics/bundles/…
                                            │        ├─► catalog: projects into OM (decision 4)
                                            │        ├─► intelligence: runtime grounding + compiler
                                            │        └─► knowledge: ontology (decision 9)
   ```

   Semantics never reads anything it influences: catalog consumes a
   **published contract**, not semantics' internals. "Deploying" semantics
   means publishing validated files to a per-environment bucket — Ossie has
   no runtime — by one workflow run identically for test and prod with an
   `environment` input. Semantics CI runs as a new
   `biglake-sa-semantics-{env}`; catalog gains only read on published
   bundles.

3. **Authored per dataset, published per domain.**
   - Each physical dataset has one Ossie file in `semantics`, keyed by
     identity, not taxonomy: `datasets/{namespace}/{table}/semantic.ossie.yaml`
     (a valid single-dataset Ossie model: dataset, fields, dataset-scoped
     metrics). Keying by `{namespace}/{table}` keeps files stable when
     catalog reclassifies a domain.
   - ABS datasets also get a generated **SDMX structural extract**
     (`sdmx.yaml`) for publisher facts Ossie has no construct for —
     measure codes, attribute attachment (e.g. unit depends on measure),
     availability constraints, categorisation — referencing shared, versioned
     codelist files (hierarchy, ordering, geography URIs) under
     `reference/abs/`. Plain YAML rather than `custom_extensions` JSON
     strings, so diffs stay reviewable; published in the bundle. Generated
     ABS files are never hand-edited; corrections go in a sparse
     `semantic.override.yaml` beside them.
   - Cross-dataset relationships and model-scoped metrics live in
     `relationships/{root_domain}.yaml`; ontology and mappings in
     `{ontology,mappings}/`; pinned upstream schemas in `_spec/`.
   - The publish step **assembles one Ossie model per root OM domain** (Ossie
     relationships exist only within a model). Membership is derived from
     catalog `table.yaml`'s `domain:` — the single writer; the ABS SDMX
     categories → OM domains chain (org ADR-0001) is unchanged. Cross-domain
     alignment is via shared ontology concepts.
   - `source` = `{namespace}.{table_name}`; expressions use `ANSI_SQL`
     limited to what DuckDB runs unchanged.

4. **OM keeps receiving descriptions — still only through catalog.** Each
   bundle includes an Ossie-free projection (`projections/om.json`: table
   and column descriptions, temporal coverage, keyed by
   `{namespace}.{table}`). catalog's `patch_metadata.py` reads the
   environment's current projection and remains the **only OM writer**;
   catalog never parses Ossie. A semantics publish dispatches catalog's
   OM sync for the same environment, so OM never lags the bundle.

5. **intelligence reads semantics from a published bundle, not OM.** Semantics
   CI publishes a validated, versioned bundle to GCS; intelligence holds it in
   memory. The LLM still interprets the question, but emits a **semantic query
   plan** (named metrics, dimensions, filters, time range) that a
   deterministic compiler turns into SQL — it never re-derives a formula or
   invents a join. Questions outside defined metrics fall back to today's
   raw-SQL path, grounded by Ossie instead of OM prose. Compiled SQL leaves via
   the existing `data_sql_ready` event, so the api executes it unchanged.
   **Compiled SQL reads only certified silver tables, by name.** The semantic
   layer never creates or depends on tables or views after silver; derived
   datasets and cross-dataset metrics are inline CTEs over certified tables.
   Direct traceability to certified datasets outranks query efficiency.

6. **Machines author, humans approve.** Generation, the branch and the PR are
   automated (plain `git` + `gh`); a human reviews and merges; **no
   auto-merge**. Corrections go through overrides and always win. Content
   is drafted in layers, each filling only what the previous left open:
   1. **Existing hand-authored catalog text** — the 9 hand-authored
      datasets' descriptions are copied once into semantics as starting
      text (reviewed human writing worth keeping). OM is not read back as a
      source: it's a projection, so that would add a loop and no
      information, and OM holds none of the SDMX structure below.
   2. **Publisher structural metadata** — for ABS, SDMX read
      deterministically: descriptions verbatim (catalog ADR-0008), concept
      and codelist identities, measure codes as metric candidates, unit
      attachment, code hierarchies, availability, categorisation, ASGS
      linked-data URIs.
   3. **Data evidence** — Iceberg schema for fields; profiles and probes
      (decision 7).
   4. **LLM proposals** — non-SDMX descriptions summarised from cited inputs
      only; synonyms, metrics where no publisher structure exists, ontology
      mappings.

   Publisher structure supplies identities and constraints, not conclusions:
   a shared codelist is a join *candidate*, a hierarchy shows which rows
   overlap but not that a measure is additive, and availability does not
   prove our snapshot is complete — each still needs layer-3 evidence.
   Disagreements between layers are flagged in the PR, never silently
   resolved. This refines ADR-0008's "no LLM originates a fact":
   descriptions stay sourced; metrics, synonyms and mappings are *modelling
   proposals*, allowed because each is mechanically validated against real
   data before review. Gaps with no source merge as visible gaps. LLM drafting
   uses Vertex AI via the semantics service account, in a CI job holding no
   repository write or cross-repo dispatch credentials.

7. **Evidence and verification run on demand in etl-owned Prefect flows.**
   The generator triggers `profile_dataset` (deterministic DuckDB stats plus
   capped samples) and `run_probes` (read-only probes: candidate joins, metric
   SQL, reconciliation checks) against silver in a selectable environment
   (prod preferred; test and prod lakehouses can differ); results land in
   GCS. Semantics starts them by dispatching an etl workflow with a GitHub App
   token — semantics CI gains no VM access. Outputs
   are generic — etl knows nothing of Ossie.
   - **Metrics fail closed:** no cited source for aggregation behaviour (stock
     vs flow, additivity, unit) → no metric, gap shown in the PR.
   - **Data values are untrusted LLM input:** intelligence's SEC-004 pattern
     (delimit as data, JSON-only output, sanitise).
   - Rejected: the GHA runner querying the lakehouse directly (relearns the
     Prefect VM's DuckDB tuning; CI data-plane access) and etl pushing a
     manifest per silver write (another synced artifact that can go stale;
     contrary to catalog ADR-0008's fetch-it-yourself precedent).

8. **CI guardrails.** Semantics: schema validation against `_spec/`; 1:1
   catalog `table.yaml` ↔ physical Ossie dataset (a missing file triggers a
   bot PR, not a failure — catalog onboards first; derived query-sourced
   datasets are exempt and never projected to OM); one pass-through field per
   Iceberg column. Catalog: `table.yaml` with `description` or
   `temporal`, or any `cols.yaml`, fails CI.

9. **Unstructured knowledge is a follow-on ADR.** Ossie mappings target
   tabular data only. Concept tagging of documents/chunks belongs in the
   knowledge enrichment stage, denormalised into pgvector/Neo4j per org
   ADR-0002; the shared ontology is where the two sides join.

10. **No migration — clean rebuild.** The initial implementation does not
    migrate existing data or carry compatibility code for state that doesn't
    match the target. Once the new code is deployed, every environment's
    datasets (and their OM entities) are cleared and the etl and knowledge
    RAG pipelines re-run from scratch; catalog switches to governance-only in
    one change, and semantics generates every dataset against the fresh
    tables. Only source text in git carries forward (the hand-authored
    descriptions in decision 6, layer 1). Destructive steps are confirmed
    by a human at execution time, test before prod.

## Consequences

**Positive**
- Meaning (descriptions, metrics, joins, synonyms) is versioned, reviewed and
  executable — written once, used by OM, intelligence and any Ossie-converted
  BI tool.
- "LLM chooses, compiler computes" is more accurate and auditable than
  LLM-written SQL.
- OM stops standing in for a semantic layer.
- New and changed sources self-propose their semantics; humans only approve.
- Clear repo roles: catalog answers "what is governed and how is it
  classified"; semantics answers "what does it mean and how do you
  calculate with it". OM and intelligence see the same bundle version per
  environment.
- Temporal coverage stops being a manual step.

**Negative**
- **Unstable spec** (`0.2.0.dev0`, recent breaking changes; no upstream
  parser for standalone mapping documents). Pinning contains it; upgrades are
  manual migrations.
- **We build the compiler, the assembler and the generators** — Ossie has no
  reference engine.
- **Verbosity:** a pass-through Ossie field is wordier than a `cols.yaml`
  entry. Generators carry it.
- **Semantics CI depends on the Prefect VM** for evidence and verification (the
  ingestion pipeline already does).
- **A new repo** to stand up: WIF binding, service account, GitHub App
  installation, branch protection.
- **Onboarding spans two repos.** A new table appears in OM (catalog) before
  its description does (semantics bot PR → merge → publish → OM sync); the
  gap is visible, not silent.
- **Clean rebuild required** (decision 10): every environment's datasets,
  OM entities and knowledge stores are cleared and rebuilt from scratch —
  a planned, destructive, human-confirmed operation.
- **ABS is read by two repos** — catalog for display name and methodology
  link, semantics for descriptions and structure — per ADR-0008's
  fetch-it-yourself precedent.
- Cross-domain joins aren't expressible as Ossie relationships until upstream
  supports dataset reuse across models.
- **More generated ABS content** (SDMX extract + shared codelists) to review;
  PR bodies summarise code-level changes.

## Assumptions

- Ossie stays the leading vendor-neutral semantic spec and its ontology layer
  survives incubation broadly as-is.
- Catalogued datasets stay in the low hundreds (today ~126: ~117 ABS + 9
  hand-authored), mostly small statistical tables, so on-demand profiling is
  cheap.
- DuckDB stays the query engine.
- The ABS Data API (beta, no availability guarantee) keeps serving SDMX 2.1
  structural metadata, including constraints and categorisations.
- OM stays the governance and discovery plane.

## Revisit when

- **Phase 0 exit (provisional → accepted or amended):**
  - the pilot exit-check (decision 1) finds Ossie description diffs
    materially harder to review than `cols.yaml`;
  - the upstream-tooling spike fails (Ossie Python package / `validate.py`
    unusable, or emitter round-trip not clean) — fall back to `jsonschema`.
- **After implementation:** re-assess the whole decision and move to
  `accepted` (or amend).
- Ossie `0.2.0` (or later) is released — re-pin; re-check the ownership split
  against new fields (display name, governance hooks, units).
- Upstream ships dataset reuse across models, or a reference compiler/query
  engine.
- Ossie gains constructs for units, code hierarchies or availability — move
  those facts out of the SDMX extract.
- OM ships native Ossie import/export or an executable metric layer.
- The intelligence eval gate shows no accuracy gain from the compiler path.
- On-demand profiling becomes a Prefect VM bottleneck — reconsider
  push-based profiles.
- The Ossie project stalls (no release within 12 months) or leaves
  incubation unsuccessfully.
- Two-repo onboarding friction proves material (e.g. descriptions routinely
  lag OM by days) — reconsider co-locating.
