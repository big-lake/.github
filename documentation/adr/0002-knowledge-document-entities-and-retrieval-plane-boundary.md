# ADR-0002 — Knowledge document entities in OpenMetadata; OM as control plane, not retrieval plane

- **Status:** accepted
- **Date:** 2026-09-19
- **Tags:** knowledge, catalog, intelligence, openmetadata, cross-repo, retrieval, governance

## Context

`knowledge` documents are currently represented in OpenMetadata only as a
JSON array (`extension.documents`) on a single collection `Container`
entity (`catalog/ingestion/patch_knowledge.py`). An individual document
(e.g. a PDF) has no OM entity of its own — it cannot be found, linked to,
owned, tagged, or governed independently of its collection.

Separately, the platform is building toward agentic RAG over this corpus
(`intelligence/DESIGN.md`). The retrieval plane (pgvector, Neo4j, GCS
manifests) is a different system from the governance plane (OpenMetadata),
with different consumers, different change cadence, and different
availability requirements. Conflating them — e.g. by making retrieval
depend on synchronous OM lookups, or by making OM the source of chunk-level
data — would create a fragile coupling neither side needs.

The desired human-facing browsing model, confirmed with the user, is:

```
Domain / Subdomain (ABS SDMX category / subcategory)
  └─ Publisher
       └─ Collection
            └─ Document
```

OpenMetadata 1.12's `Container` entity natively supports parent/child
hierarchy, unstructured files (single-file `fullPath` entries), `domains`
(multi-valued relationship), `owners`, `tags`, and `extension` custom
properties — confirmed against the upstream schema
(`entity/data/container.json`, `api/data/createContainer.json`,
`1.12.0-release` tag) and the Storage Services documentation (unstructured
PDFs are supported as child containers of a manifest entry).

## Decision

1. **Documents become first-class OM `Container` entities**, one per
   `document_key`, parented under a Collection `Container`, which is in
   turn parented under a Publisher `Container`, under the existing
   `knowledge` StorageService:

   ```
   StorageService: knowledge
   └─ Container: {publisher_id}            (NEW)
      └─ Container: {collection_id}        (existing, now a true parent)
         └─ Container: {document_key}      (NEW)
   ```

   `extension.documents` JSON on the collection is a **transitional**
   representation only, removed once document entities are live and every
   consumer reads native children instead.

2. **Domain/Subdomain is an OM `domains` relationship, not a Container
   parent.** It is attached to Collection and Document containers (and may
   be multi-valued — a document can be relevant to more than one Domain).
   This avoids duplicating a global Publisher under every Domain it
   happens to publish into, and lets re-categorisation change a relationship
   without touching entity identity or parentage.

3. **OpenMetadata is the governed control plane, not the agent's primary
   retrieval engine.** Concretely:
   - `knowledge`'s git registry remains the source of truth for what is
     *approved* (`document_key`, `collection_id`, publisher, dates, hash).
   - OpenMetadata is the source of truth for discoverability, hierarchy,
     ownership, classification, and lifecycle — a human- and agent-facing
     governed entity graph.
   - `knowledge`'s pipeline (parse/chunk/enrich/embed) produces the
     retrieval plane (pgvector, Neo4j, manifests) — optimised for query,
     not for governance browsing.
   - `intelligence` reads the retrieval plane for routine queries and does
     **not** call OM synchronously per candidate result; OM identity
     (`document_key`, `collection_id`, publisher, Domain, OM reference) is
     denormalised into retrieval-plane rows at index time instead.

4. **Identity contract (locked, applies across all 5 repos):**

   | Representation | Stable identity | Owner |
   |---|---|---|
   | Publisher | canonical `publisher_id` (e.g. `abs`) | catalog publisher registry |
   | Collection | globally-unique `collection_id` | knowledge registry |
   | Document | globally-unique `document_key` | knowledge registry |
   | OM Container UUID/FQN | platform reference only, never canonical | catalog (generated) |

   OM must be rebuildable from the registry + governance overlay without
   breaking any join key used by GCS, pgvector, Neo4j, citations, or
   evaluation data — those all key on `document_key`/`collection_id`, never
   an OM UUID.

5. **Entity markers required before document entities are created:**
   every managed Container gets `managedBy: biglake-knowledge-registry` and
   `knowledgeAssetType: publisher | collection | document` custom
   properties, so pruning/reconciliation can distinguish managed hierarchy
   levels instead of assuming every Container under the service is a
   collection (the current `prune_orphaned_containers()` behaviour, which
   would otherwise delete new Publisher/Document containers as "orphans").

### Alternatives considered

- **Keep documents as JSON only, never as OM entities.** Rejected per
  explicit product requirement — a document must be a real, linkable,
  ownable, governable entity, not a string in a sidecar array.
- **Make OM the retrieval index itself** (e.g. store chunk text/embeddings
  as OM entities or extensions). Rejected — OM has no vector search, no
  BM25, no graph traversal; forcing retrieval through OM would be slower
  and would couple agent latency to OM availability. OM's job is
  governance and discovery, not query-time retrieval.
- **Make Domain a physical Container parent** (`Domain > Publisher >
  Collection > Document`). Rejected — a publisher/document can span
  multiple Domains; a physical single-parent tree cannot represent that
  without duplication. `domains` (the native multi-valued relationship) is
  the correct mechanism.

## Consequences

- **Positive:** every approved document gets a discoverable, ownable,
  taggable OM entity — closes the concrete gap that motivated this ADR
  (`Household Income and Wealth.pdf` was invisible as its own entity).
- **Positive:** clean separation of concerns — OM outages or slow queries
  never block `intelligence`'s retrieval path; retrieval-plane schema
  changes never require an OM migration.
- **Positive:** re-categorising a document (new Domain, new Collection) is
  a metadata/relationship edit, not an identity change, consistent with the
  taxonomy-independent storage identity already established in `knowledge`
  ADR-0015.
- **Negative / follow-up:** `catalog/ingestion/patch_knowledge.py` and its
  pruning logic must be rewritten for the 3-level hierarchy before any
  document entity is created, or the existing prune step will delete them.
  `knowledge`'s registry needs an explicit `collection_id` per document
  (already possible via folder-colocated `collection.yaml`; this ADR does
  not mandate exactly how it becomes explicit, only that folder position
  alone must not remain the sole source of truth long-term).
- **Negative / follow-up:** `intelligence`'s manifest/retrieval schemas
  currently use `source` to mean publisher, have no `collection_id`, and
  derive citation titles from `document_key` rather than a governed title.
  These must be extended (additively) once document entities exist — this
  ADR does not require it before document entities can be created, only
  before agents can present fully governed citations.
- **Negative / follow-up:** `catalog/documentation/governance-taxonomy.md`
  and ADR-0009 (knowledge Container governance) describe a single-Container-
  per-collection model; ADR-0009 is amended (not superseded) by this ADR —
  its Domain/owner/tags governance mechanism is reused unchanged for
  Publisher and Document containers.

## Related

- `knowledge` ADR-0015 (taxonomy-independent storage identity — `document_key`)
- `knowledge` ADR-0017 (domain-oriented addressing — `domain` is organisational only)
- `knowledge` ADR-0018 / ADR-0019 (`collection` block / one-document-per-file registry shape)
- `catalog` ADR-0007 (three-axis categorisation: Publisher/Domain/Medallion)
- `catalog` ADR-0009 (knowledge Container governance and onboarding) — amended by this ADR
- `.github` ADR-0001 (ABS SDMX canonical subject taxonomy — same cross-repo-convention pattern)
