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

- **Status** — `proposed` | `accepted` | `superseded by ADR-NNNN` | `deprecated`
- **Context** — what forced the decision; constraints; alternatives considered
- **Decision** — what we actually did
- **Consequences** — positive, negative, and follow-on work

Filenames: `NNNN-kebab-case-title.md`. `NNNN` is a zero-padded monotonic
counter; once assigned it never changes (so links stay stable). When
superseding an ADR, write a new one referencing it — don't rewrite history.

## Index

| # | Title | Status |
|---|---|---|
| [0001](0001-abs-sdmx-canonical-subject-taxonomy.md) | ABS SDMX category scheme is the canonical subject taxonomy — fetch live, consistently, per repo | accepted |
