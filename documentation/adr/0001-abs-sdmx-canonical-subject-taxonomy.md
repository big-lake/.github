# ADR-0001 — ABS SDMX category scheme is the canonical subject taxonomy: fetch live, consistently, per repo

- **Status:** accepted
- **Date:** 2026-09-12
- **Tags:** taxonomy, abs, sdmx, cross-repo, convention, security

## Context

Multiple BigLake repos independently align their subject grouping on the
Australian Bureau of Statistics (ABS) SDMX **category scheme**
(`GET /rest/categoryscheme/ABS`, public, unauthenticated):

| Repo | Uses the ABS category scheme for | Output shape |
|---|---|---|
| **catalog** | OpenMetadata domains (`curation_generators/abs/generate_domains.py`, ADR-0008) | 3-level tree, rich descriptions, `parentDomain` |
| **knowledge** | `_inbox/{category}/{subcategory}/` upload folders + config grouping (`setup/taxonomy/abs_taxonomy_source.py`) | 2-level flat `(category, subcategory)` pairs |
| **etl** | ABS ingestion config consolidation (ADR-0006, `src/ingestion/abs/`) | category → dataflow grouping via `categorisation/ABS` |

Each fetches the same upstream endpoint and applies the same subject-scheme
allow-list and exclusions — but the allow-list is currently **kept in sync by
convention** (knowledge's code literally comments *"Kept identical to
catalog's SUBJECT_SCHEMES"*). With three consumers now depending on it, that
copy-by-convention is fragile: if one repo's allow-list drifts, a dataset can
get an OpenMetadata domain (catalog) with no valid `_inbox/` upload path
(knowledge), or vice versa — a data-consistency bug, not just duplicated code.

### Alternatives considered

- **Centralise into a shared published artifact** — one publisher fetches ABS,
  applies the canonical allow-list, and writes a canonical
  `gs://…/reference/abs_categories.json` that all repos read. **Rejected.** The
  category scheme changes only ~once or twice a year, the data is public and
  unauthenticated, and a shared artifact introduces a new cross-repo trust
  boundary (whoever writes it controls every repo's taxonomy) plus infra to
  build and operate. Rule-of-three is met, but the cost/benefit doesn't justify
  shared infrastructure for slowly-changing public reference data. If
  divergence ever becomes painful in practice, this is the option to revisit.
- **Shared code package** imported by every repo — **rejected.** Creates a
  versioned code dependency between repos, which the platform's loose-coupling
  principle exists to prevent, and is heavy for ~50 lines that change rarely.

## Decision

Each repo **fetches `GET /rest/categoryscheme/ABS` live, when it needs the
taxonomy**, using the *same canonical convention* documented here. There is no
central artifact and no shared package — consistency is enforced by this ADR
being the single written spec that every repo's fetch conforms to.

The canonical convention:

1. **Endpoint:** `GET https://data.api.abs.gov.au/rest/categoryscheme/ABS`
   with `Accept: application/vnd.sdmx.structure+json`.
2. **Subject-scheme allow-list** (the top-level SDMX schemes we treat as
   subject taxonomy): `ECONOMY`, `ENVIRONMENT`, `HEALTH`, `INDUSTRY`,
   `LABOUR`, `PEOPLE`, `SNAPSHOTS`.
3. **Exclusions:** `DATA_BY_REGION` (geography/classification, not subject
   matter) and the Census schemes — never subject groupings a dataset or
   document would be filed under.
4. **Slug rule:** lowercase the ABS id, **and validate against
   `^[a-z0-9_]+$`** — skip/reject anything that doesn't match. ABS ids are
   already `UPPER_SNAKE_CASE` ASCII, so this is normally just a lowercase, but
   the validation is **mandatory**: these slugs become path segments and, in
   knowledge, GCS object names (see Security below).
5. **Attribution:** carry the CC BY 4.0 credit line
   (`Source: Australian Bureau of Statistics category scheme (…), ©
   Commonwealth of Australia, licensed under CC BY 4.0`).
6. **No committed generated artifact is required** — a live fetch is the
   canonical source. A repo *may* cache or commit a generated file for its own
   reasons (e.g. catalog commits `abs_domains_generated.yaml` for PR-reviewable
   OM-domain diffs), but the *fetch method above* is what's standardized.

Each repo then **shapes** the filtered result to its own needs (catalog's
3-level OM domains, knowledge's 2-level pairs, etl's dataflow grouping). Only
the *filtered source set and fetch method* are common; output shape is
deliberately repo-specific.

## Consequences

**Positive**
- Consistent subject taxonomy across the whole platform with **zero shared
  infrastructure** and no new cross-repo runtime dependency.
- One written spec (this ADR) that every repo's fetch conforms to, replacing
  the fragile copy-by-comment sync of the allow-list.
- No secrets involved anywhere — the ABS API is public and unauthenticated,
  so there is nothing to leak on this path.

**Negative / accepted**
- ~50 lines of fetch + allow-list logic are duplicated per repo. Accepted: the
  logic changes rarely, and this ADR is the reconciliation point if it drifts.
- Each repo independently hits ABS's API (including in CI, for knowledge's
  domain-validation check) — an external network dependency in those paths.
  Accepted for public, cache-friendly, rarely-changing data.

**Security** (see also each repo's `documentation/security/risks.md`)
- **Untrusted external data becomes path segments / GCS object names**
  (knowledge's `_inbox/{category}/{subcategory}/`). The mandatory `^[a-z0-9_]+$`
  slug validation (Decision §4) is the control — without it, a malformed or
  tampered ABS id could produce traversal-ish or junk object paths. Every
  repo's generator MUST enforce it.
- **Integrity rests on TLS to ABS** — ABS publishes no signature/checksum, so a
  successful MITM would be trusted. Control: never disable TLS verification
  (`verify=False`). Accepted (no stronger option upstream).
- Any repo that acts *destructively* on the fetched taxonomy (e.g. knowledge's
  `_inbox/` reconcile deleting stale placeholders) MUST guard against an empty
  or drastically-shrunk response (ABS outage / poisoning) rather than acting on
  it — fail safe, don't mass-delete.

## References

- catalog ADR-0008 — SDMX-extracted, generated-then-reviewed curation for ABS
- catalog `documentation/governance-taxonomy.md` — the ABS SDMX category tree
- etl ADR-0006 — ABS config consolidation by category
- knowledge `setup/taxonomy/abs_taxonomy_source.py` — the live-fetch implementation
