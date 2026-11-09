# ADR-0006 — Dataset field bindings pin a vocabulary release; consumers validate before activating

- **Status:** proposed
- **Date:** 2026-11-10
- **Tags:** vocabulary, semantics, bundle, activation
- **Builds on:** [ADR-0003](0003-ossie-semantic-layer-source-of-truth.md), [ADR-0005](0005-vocabulary-repo-and-published-bundle.md)

## Context

`semantics` publishes the semantic bundle (dataset grounding); `vocabulary` publishes concept schemes. They release
independently. A binding that references a concept or code that a newer or older vocabulary release does not contain
would dangle. A shared concept also does not make a join or roll-up safe (plan correction 1), so a binding must say
what it asserts.

## Decision

1. **Binding shape.** A dataset field may carry a `BIGLAKE` custom extension (no Ossie ontology/mapping documents in v1):
   `vocabulary: {scheme: <scheme IRI>, concept?: <concept IRI>, representation: code | label | numeric}`. It states
   meaning, coding scheme and version, and representation only. Grain, cardinality and aggregation constraints stay in
   semantics query constraints and are out of scope here.
2. **Pin.** The semantic bundle records `vocabulary_bundle_id`; the vocabulary bundle is addressed by manifest
   (`current.json` pattern, as the semantic bundle). Semantics CI fails if any referenced IRI does not resolve in the
   pinned release.
3. **Activation.** Runtime consumers load a vocabulary release only after verifying every IRI referenced by the active
   semantic bundle resolves; on failure keep the last good pair and log. Intelligence `/health` reports both ids
   (`semantic_bundle_id` exists; `vocabulary_bundle_id` follows), so evaluation runs record what grounded them.
4. **Representation, not equivalence.** A binding to a publisher scheme never implies a correspondence to another
   publisher's scheme; cross-publisher links exist only where a source states them.
5. **First slice.** Bind the `abs.building_activity` dimensions (`MEASURE`, `SECTOR_OWN`, `BLD_WORK_TYPE`, `TYPE_BLDG`,
   `TSEST`, `PRICE_ADJ`) to the matching imported `CL_*` schemes. Existing codelist copies in `semantics/reference/`
   remain until a later ADR retires the duplication.

## Consequences

- Failure mode is stale but consistent, never dangling.
- Semantics gains a vocabulary dependency at CI time only; runtime loading stays in knowledge/intelligence.
- Bundle publication infrastructure (bucket, service account, workflow) must exist before pinning is enforceable.

## Assumptions

- Vocabulary releases are infrequent and small.
- The `BIGLAKE` extension mechanism already carries extra fields through the bundle unchanged.

## Revisit when

- Rollbacks from failed activation become frequent (loosen to per-scheme pins).
- A consumer needs binding-level join or roll-up semantics (needs its own ADR and eval class).
