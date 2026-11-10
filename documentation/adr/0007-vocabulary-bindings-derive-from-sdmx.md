# ADR-0007 — Vocabulary bindings are derived from `sdmx.yaml`; `vocabulary.yaml` pins the release (amends ADR-0006 decision 1)

- **Status:** accepted
- **Date:** 2026-11-10
- **Amends:** [ADR-0006](0006-dataset-bindings-pin-vocabulary-release.md)
- **Tags:** vocabulary, semantics, bindings

## Context

ADR-0006 decision 1 proposed a per-field `BIGLAKE` extension (`vocabulary: {scheme, concept, representation}`) on Ossie
fields. Implementing it showed the binding already exists: each dimension in a table's `sdmx.yaml` declares its
`codelist`, and the shared codelist files carry the SDMX URN that is the vocabulary scheme id (org ADR-0005 decision 8).
Ossie field files are also generated and regenerated from the lakehouse schema, so a hand-added per-field extension would
be a second place to keep the same fact and would risk being overwritten.

## Decision

1. **No per-field vocabulary extension in v1.** A binding is the dimension's `codelist` in `sdmx.yaml`, resolved to the
   codelist's URN. It states meaning and coding scheme only, as before (ADR-0006 decision 4).
2. **`semantics/vocabulary.yaml`** pins `vocabulary_bundle` and lists which dimensions per table are bound (opt-in, so the
   ~190 codelist copies are not all required in the vocabulary). The semantic bundle index records the pin and the resolved
   URNs under `vocabulary`.
3. **Verification** (`python -m semantics.vocabulary <index.json>`): the supplied vocabulary release must equal the pin and
   contain every bound scheme. Wiring it into CI waits for the vocabulary bucket and a read grant, so until then it runs
   locally and in unit tests via `VOCABULARY_INDEX`.
4. ADR-0006 decisions 2-5 stand. Per-concept bindings (`concept`) and non-code `representation` remain for a later ADR
   with a consumer.

## Consequences

- One source of truth for codelist names; nothing to regenerate or drift.
- The bound-dimension list is hand-maintained; a dimension missing from it is simply not covered.
- Non-SDMX datasets cannot be bound this way yet.

## Assumptions

- Vocabulary scheme ids for SDMX codelists are the codelist URNs.
- The first consumers need coding-scheme provenance, not per-code semantics beyond the scheme.

## Revisit when

- A non-SDMX dataset needs a binding (needs a field-level or dataset-level declaration).
- A consumer needs concept-level binding (a single code mapped to a concept), as opposed to scheme-level.
