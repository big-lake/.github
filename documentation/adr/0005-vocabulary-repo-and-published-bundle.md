# ADR-0005 — A `vocabulary` repo publishes source-backed concept schemes as a versioned bundle (amends ADR-0001)

- **Status:** accepted
- **Date:** 2026-11-10
- **Amends:** [ADR-0001](0001-abs-sdmx-canonical-subject-taxonomy.md)
- **Tags:** vocabulary, skos, xkos, cross-repo, provenance
- **Plan:** [vocabulary-unified-intelligence.md](../plans/vocabulary-unified-intelligence.md) (decisions 1, 2, 4, 7; Phase 1)

## Context

Cross-publisher answers need publisher-defined meaning (what "completed", "seasonally adjusted" or "public sector"
means, and which classification a code belongs to) attached to datasets and documents with provenance. Today that
meaning is scattered: 191 SDMX codelist copies in `semantics/reference/`, descriptions in Ossie files, and prose in
`knowledge` chunks. The Phase 0 baseline ([evaluation run 20261109-222123-1](https://github.com/big-lake/evaluation/actions/runs/42676098866))
shows mixed cases at 0/10, with scope 6/28 and comparability 3/28 passing: scope and definition evidence is missing
or unusable, not just unretrieved.

ADR-0001 rejected a shared published artifact for the **subject taxonomy**: it changes once or twice a year, is
public, and a shared artifact adds a trust boundary and infrastructure. That reasoning holds for the taxonomy. It does
not cover this case: vocabulary content is *authored or extracted* (not mirrored), carries per-statement provenance,
is consumed by two independently released bundles (semantics, knowledge) that must agree on versions, and has an
automated-extraction trust problem (no subject-matter experts) that needs validators in one place.

### Alternatives considered

- **Module inside `semantics`.** Rejected: semantics owns dataset field bindings and executable constraints and is
  on hold pending evaluation; documents and classifications span datasets, and vocabulary releases must not wait on
  Ossie releases.
- **Put it in `knowledge`.** Rejected: knowledge ingests and indexes evidence; it must not become the authority that
  decides what a term means (plan, ownership boundaries).
- **Keep per-repo copies (ADR-0001 style).** Rejected for definitions and classifications: no single place to run
  provenance validators, and drift between semantics and knowledge copies would produce dangling references.
- **Use an OpenMetadata Glossary.** Rejected as a runtime dependency: catalog is a governance projection, not a
  runtime lookup (plan non-plans; revisit as a projection later).

## Decision

1. A new **`vocabulary`** repo owns publisher-scoped concept schemes, definitions, classifications and explicit
   correspondences, and publishes them as a **versioned bundle** (manifest-addressed objects in GCS, same
   `current.json` pattern as the semantic bundle).
2. **Model:** SKOS/XKOS semantics in YAML with an explicit committed mapping and a CI round-trip through `rdflib`
   (plan correction 6). No RDF store or reasoner at runtime.
3. **Publisher scoping.** One scheme per publisher artefact and version. Similar labels in different publishers stay
   distinct; `exactMatch` is never produced by similarity (plan correction 9).
4. **Provenance tiers** (plan, Intake): A = code-only import of machine-readable standards, `authoritative`;
   B = source-quoted definitions, `extracted`, dropped unless the quote verifies verbatim against a pinned snapshot;
   C = LLM-suggested links, `suggested`, retrieval hints only. The first slice uses A and B only.
5. **Licence is mandatory** on every scheme; content under an unknown or restrictive licence is metadata and link
   only.
6. **Compatibility.** `semantics` bindings pin a `vocabulary` release; consumers validate that every referenced IRI
   resolves before activating a release and keep the last good one on failure (plan correction 7).
7. **ADR-0001 scope narrowed, not reversed.** The ABS category scheme remains fetched live per repo for subject
   taxonomy. Vocabulary does not become a subject-taxonomy source, and consumers of ADR-0001 migrate only by their
   own later choice.
8. **IRI minting.** SDMX-sourced concepts use their SDMX URNs
   (`urn:sdmx:org.sdmx.infomodel.codelist.Code=ABS:CL_X(1.0.0).CODE`); no invention. Publishers without linked-data
   identifiers (page-defined glossary terms) use
   `https://vocab.biglake.au/{publisher}/{scheme-slug}@{version}/{concept-slug}` with the source URL recorded
   separately. Slugs match `^[a-z0-9][a-z0-9-]*$`. The host is an identifier namespace only, not a promise to serve
   it. Answers plan open question 1.
9. **Delivery mechanics.** The repo is created by a human or approved GHA; the bundle bucket, service account and
   workflows come from `infra/` Terraform and a GHA deploy, never direct CLI.

## Consequences

**Positive**
- One place for provenance validators and the standards round-trip check.
- semantics and knowledge release independently without dangling references.
- A defined path from a failing scope/comparability eval case to the definition evidence it lacked.

**Negative / accepted**
- A new repo, a bundle pipeline and a new cross-repo trust boundary (whoever writes the bundle controls every
  consumer's definitions). Mitigated by tiers, verbatim-quote validators and PR audit trail.
- No subject-matter reviewers: validators catch structure and provenance, not statistical judgement (standing limit).

**Security**
- Extracted text that reaches prompts uses the same SEC-002 wrapping as bundle text. Snapshot fetching is https-only
  and host allow-listed; slugs are validated. Findings go to `vocabulary/documentation/security/risks.md`.

**Follow-on**
- Repo map and `ARCHITECTURE.md` updated at repo creation; `semantics` and `knowledge` contract ADRs follow (see plan
  Phase 1).

## Assumptions

- Vocabulary content for the first slice is small (tens of concepts); volume is controlled by per-source caps.
- The knowledge Postgres instance can host the loaded lexicon later; no new database in Phase 1.
- Publishers' licences (ABS CC BY 4.0, Budget Papers CC BY 4.0) permit republishing short quoted definitions.

## Revisit when

- A consumer needs vocabulary content that cannot be validated by quote or by machine-readable standard.
- Vocabulary releases are more than ~quarterly or exceed ~10k concepts (reconsider bundle size and storage).
- ADR-0001 consumers want to migrate subject taxonomy onto the bundle.
- The bundle activation check produces repeated "dangling reference" rollbacks (reconsider pinning granularity).
