# ADR-0008 — Dataset scope and comparability records for the mixed workflow (housing slice)

- **Status:** proposed (2026-11-11; not accepted until the user agrees)
- **Date:** 2026-11-11
- **Plan:** [vocabulary-unified-intelligence.md](../plans/vocabulary-unified-intelligence.md), "Scope and comparability" and Phase 2
- **Pairs with:** intelligence ADR-0033 (the workflow that consumes it), api ADR-0028
- **Builds on:** [ADR-0003](0003-ossie-semantic-layer-source-of-truth.md), [ADR-0007](0007-vocabulary-bindings-derive-from-sdmx.md)
- **Tags:** semantics, comparability, mixed, bundle

## Context

The mixed workflow (intelligence ADR-0033) must decide, in code, whether a dataset figure and a
document figure measure the same thing. A shared label, publisher or framework is not evidence of
that (plan correction 9). The plan names six attributes that matter for the housing slice but no
repo yet owns the dataset side. `semantic.ossie.yaml` is regenerated from the lakehouse schema
(ADR-0007), so a hand-added block inside it risks being overwritten.

## Decision

1. **One hand-authored file per dataset**, `semantics/datasets/{publisher}/{dataset}/comparability.yaml`,
   not part of the generated Ossie file. v1 covers `abs.building_activity` only.
2. **Shape:** a map from the six plan attributes (measure definition, scope of dwellings, series
   adjustment, observation status and vintage, period and units, geography and grain) to a record
   `{value, status: stated | unknown, source: {url, quote}}`. `stated` needs a verbatim quote from a
   pinned snapshot (the same rule as vocabulary tier B); `unknown` is allowed and never inferred.
   Series adjustment and units are properties of a query's filters, so the file records the
   selectable values and which each `measure_code` supports.
3. **Validation:** a `python -m semantics.comparability` check in the existing CI fails on a `stated`
   record without a quote that appears in the pinned snapshot, on an unknown attribute name, and on
   a dataset with no record.
4. **Publication:** the bundle publishes the records under `comparability/` and the bundle index
   records their hash. `/health` already reports `semantic_bundle_id`, so a run records which
   records were live.
5. **Comparison rule (consumed by intelligence, code only):** for each required attribute the check
   compares the dataset record with the document claim's attribute. Result per attribute:
   `compatible` (both stated and equal), `different` (both stated and unequal) or `unknown` (either
   side unknown). Any `different` forces a side-by-side answer stating the difference; any `unknown`
   on a required attribute blocks a claim of comparability and triggers at most one targeted
   retrieval (plan). No tolerance, conversion or roll-up is applied.
6. **Document side is out of scope here.** The matching claim record (attribute values extracted from
   the cited passage with a verbatim quote) is a knowledge-repo contract and a separate ADR. Until it
   exists the check can only report `unknown` for the document side, which is the safe result.

## Alternatives considered

- **Per-field `BIGLAKE` extensions in the Ossie file** — overwritten on regeneration (ADR-0007).
- **LLM-judged comparability at synthesis time** — cannot establish missing compatibility, and the
  eval's comparability check is the thing it would be grading itself on.
- **Put the records in `vocabulary`** — the attributes describe a dataset's series, not a concept;
  vocabulary stays definition-only (org ADR-0005).
- **Cover all datasets now** — no consumer, no failing eval class beyond housing.

## Consequences

- A new hand-authored artefact to maintain for each dataset that joins the mixed slice; quote
  validation keeps it honest but not complete.
- Comparability can only improve to `compatible` once the knowledge-side claim record exists; until
  then mixed answers are qualified side-by-side answers, which the plan counts as a success.

## Assumptions

- Verbatim quotes are available on the ABS methodology snapshot already in knowledge for the six attributes.
- The six attributes cover the housing slice; they are not a general ontology.

## Revisit when

- A second dataset enters the mixed slice (check whether the shape generalises).
- The knowledge-side claim record is accepted (tighten the document-side comparison).
- Ossie gains first-class comparability fields (retire this file).
