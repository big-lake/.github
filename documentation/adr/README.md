# Architecture Decision Records — Org-Wide

This directory holds **cross-repo** Architecture Decision Records — decisions
whose scope spans multiple BigLake repos and so are not owned by any single
one. Repo-local decisions still live in that repo's own `documentation/adr/`
(see `catalog/`, `knowledge/`, `etl/`, `api/`, `intelligence/`, `infra/`).

Use an org-level ADR when the decision is a **convention every repo follows**
(e.g. how all repos consume a shared external taxonomy), not something one
repo decides for itself.

## Format

Lightweight [MADR](https://adr.github.io/madr/)-style, matching the per-repo
convention:

- **Status** — see the table below
- **Context** — what forced the decision; constraints; alternatives considered
- **Decision** — what we actually did
- **Consequences** — positive, negative, and follow-on work
- **Assumptions** — conditions the decision depends on (scale, cost, a vendor
  limitation, a missing feature). If one stops being true, the decision is
  open to challenge.
- **Revisit when** — concrete triggers for a re-look (e.g. "DAU > 10k",
  "DuckDB ships native Iceberg writes")

`Assumptions` and `Revisit when` are required for new ADRs. Older ADRs
without them are still valid — add them the first time the ADR is touched
or challenged.

| Status | Meaning |
|---|---|
| `proposed` | Under discussion; not yet acted on |
| `provisional` | In effect, but expected to change — `Revisit when` lists the triggers |
| `accepted` | In effect; the best answer *given its Assumptions* |
| `amended by ADR-NNNN` | Core decision stands; part of it was changed by a later ADR — read both |
| `under review` | Formally challenged — don't build new work on it until resolved |
| `superseded by ADR-NNNN` | Replaced entirely |
| `deprecated` | No longer applies; nothing replaced it |

Filenames: `NNNN-kebab-case-title.md`. `NNNN` is a zero-padded monotonic
counter; once assigned it never changes (so links stay stable).

## Challenging an ADR

ADRs record *why we decided something under the conditions at the time* —
they are not rules. Any ADR, `accepted` included, can be challenged:

1. Check its Assumptions / Revisit when (or, for older ADRs, its Context)
   against today's reality.
2. If they no longer hold, or a better option has emerged, say so — set the
   status to `under review` while it's discussed, then write a new ADR whose
   header lists `**Amends:**` or `**Supersedes:**` the old one.
3. Don't rewrite an old ADR's body — the history is the point. The only edits
   allowed to an old ADR are its **Status** line and an optional short banner
   pointing at the newer ADR.

## Index

| # | Title | Status |
|---|---|---|
| [0001](0001-abs-sdmx-canonical-subject-taxonomy.md) | ABS SDMX category scheme is the canonical subject taxonomy — fetch live, consistently, per repo | accepted |
| [0002](0002-knowledge-document-entities-and-retrieval-plane-boundary.md) | Knowledge document entities in OpenMetadata; OM as control plane, not retrieval plane | accepted |
