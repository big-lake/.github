# Cross-publisher unified intelligence — plan

**Status: Phase 0 complete (2026-11-10); Phase 1 approved by the user and in progress.** Coverage verified, anchor moved to housing construction (see [First acceptance slice](#first-acceptance-slice)), mixed case kind + per-kind site shipped (evaluation ADR-0003), and the official baseline is public (see [Phase 0 result](#phase-0-result-official-baseline-2026-11-10)). No bucket, SA or deploy workflow exists yet (Phase 1 bundle publication is code-only so far; see [Phase 1 progress](#phase-1-progress)). Each phase needs a user go-ahead; the ADRs named per phase are written when that phase starts, not up front.

- **Why:** answer cross-publisher questions using compatible, traceable evidence from structured datasets and unstructured documents. Vocabulary supports discovery and interpretation; building a concept layer is not the end goal. The first functional milestone is one complete mixed answer, without a graph database, an agent framework, or an LLM-invented ontology.
- **Shaped by:** iterative design reviews and the user's cross-publisher priority (2026-11-09). The corrections are binding and recorded in [Design corrections](#design-corrections-binding).
- **Standing constraints:** `.github/TODO.md` is not edited without user confirmation; infra changes only via Terraform + GHA; prod/destructive actions need explicit user OK; eval-first — a phase ships only when its named eval gate moves.

## Decisions (user, 2026-11-09)

| # | Decision |
|---|---|
| 1 | New **`vocabulary` repo** — not a module inside `semantics`. Needs an org ADR superseding/amending org ADR-0001's "no central reference artifact" decision (see Phase 1). |
| 2 | **SKOS/XKOS** vocabulary model; **no Ossie ontology/mapping documents in v1** (no consumer; field bindings via existing `BIGLAKE` custom extensions instead). |
| 3 | **Retire Neo4j** and the triple extraction output (parallel track, own ADRs in knowledge + infra). |
| 4 | **Full automation, review-only humans.** We have no subject-matter experts — recorded as a standing limitation, mitigated by provenance rules, not by pretending review is curation. |
| 5 | External resource intake reuses the **knowledge inbox front door** (PR/GCS only, no UI upload). Quality over volume — enforced by validators, not convention. |
| 6 | Runtime storage reuses the **existing knowledge_db Postgres** (new schema); no new database instance. knowledge remains the single writer. |
| 7 | Prove cross-publisher retrieval and grounded mixed synthesis first. Start with ABS datasets and Treasury documents; introduce only the vocabulary needed for the slice. Publisher boundaries and missing concept mappings must not block retrieval. |

## Current state (repository review, 2026-11-09)

- Ossie ontology/mapping schemas sit unused in `semantics/_spec/`; the bundle publishes `ontology: null, mappings: []`. Plan phases 8–9 were demoted for having no consumer — this plan is the consumer-naming exercise done properly.
- knowledge extracts entities + free-form-predicate triples per chunk (one Gemini call also producing context prefixes and propositions — `configuration/pipeline.yaml` `enrich:`), with GCS outputs. Neo4j writing defaults off and no reviewed config enables it; infrastructure defines a container, but its deployed state, contents and actual memory use have not been verified. No intelligence graph consumer was found.
- intelligence: router (7 classes) → three tools (`knowledge_retrieval`, `data_discovery`, `data_query`); Ossie bundle grounds the data path only. The combined path writes knowledge prose **before** SQL exists and never sees executed rows; the BFF intercepts `data_sql_ready` and executes. `narrate` (intelligence ADR-0031/0032) is the only place a model sees real rows — rows only, no documents.
- OM has no Glossary; no dataset↔document links. Eval: 133 cases, all `kind: data`; definition/knowledge/mixed ability is unmeasured.
- Assets ready to reuse: 191 SDMX codelists with hierarchies, `generators/abs.py` already parses ConceptSchemes + `ASGS_LOCI_URI`, evaluation's page-snapshot pattern, knowledge's sha256-pinned inbox→promote flow.

## Design corrections (binding)

Earlier drafts of this design made claims the reviews struck down. They are recorded here so no phase quietly resurrects them.

1. **A shared concept does not make a join or roll-up safe.** Field bindings state *meaning + coding scheme/version + representation*. Scope and comparability checks are required in the first mixed slice; executable joins/roll-ups additionally require grain, cardinality and aggregation constraints owned by `semantics` and remain outside this plan. "One annotation unlocks joins" was wrong.
2. **A verbatim quote proves text exists, not that an inferred link is true.** Tier C (LLM-proposed links, § Intake) is never authoritative: suggestions improve retrieval ranking only, are never cited as definitions, never become dataset-wide caveats, and carry `status: suggested` forever unless a source states the link explicitly.
3. **Unified retrieval ≠ one undifferentiated index.** One search *interface* over typed candidate pools with bounded per-type retrieval; document chunks must not crowd out dataset/definition cards, and concept links are ranking boosts, never mandatory filters that hide unlinked content.
4. **Resolution returns candidates, not answers.** "Geelong" legitimately matches multiple ASGS codes; `resolve_value` returns ranked candidates scoped to the active dataset's schemes and the caller (planner or clarify flow) decides.
5. **Topical similarity is not applicability.** A methodology paper *mentioning* CPI ≠ *changing* a CPI series. Doc→dataset links carry a typed relation (`mentions` | `defines` | `changes`) with effective dates; only explicitly stated `defines`/`changes` links produce caveats.
6. **"YAML with SKOS-ish field names" is not standards compliance.** The schema defines an explicit mapping to SKOS/XKOS terms, keeps language tags and IRIs, and CI round-trips a sample export through `rdflib`. No RDF store or reasoner at runtime.
7. **Independent bundles need compatible activation.** semantics pins the vocabulary release its bindings reference; the runtime loader validates references before activating and keeps the last good release on failure. "Record bundle IDs in traces" alone was insufficient.
8. **Ossie ontology docs are not trivially derivable later.** If they're ever wanted, that's a real piece of work needing its own ADR — the earlier "generate them in an afternoon" claim is retracted.
9. **Framework alignment is not concept equivalence.** A Treasury statement that it follows an ABS accounting framework is evidence about its basis, not permission to generate `exactMatch` links or assume identical measures. Preserve the statement and scope; do not infer equivalence from it.
10. **Unification need not produce one combined number.** Compatible evidence can support a joint conclusion; incompatible figures must be separated and explained. Never sum transfers and recipient spending, equate estimates with actuals, or substitute Commonwealth spending for all-government spending without explicit justification.

## First acceptance slice

Anchor question: **"Is Australia building enough new homes to meet its housing target?"** The target is a user-facing answer, not a preselected dataset query or a glossary lookup. It needs both publishers: the target and its framing come from Treasury, the activity from ABS.

**Anchor change (user, 2026-11-09).** The original anchor, "How much does Australia spend on public healthcare?", failed Phase 0 coverage: the deployed lakehouse has no ABS Government Finance Statistics table and no government health expenditure series (`abs.ana_sfd`'s health item is household consumption), and PBO `expenses_by_function` is modelled in `semantics` but not deployed to test. Only Commonwealth budget estimates (2025–26 Budget Papers) were retrievable. Healthcare questions stay in the suite as expected coverage-gap cases; making them answerable is a bounded onboarding decision (ABS GFS, PBO deployment, or AIHW) for later.

**Coverage verified in Phase 0 (test, 2026-11-09; intelligence `a85ffea`, api `394a499`, knowledge index `v1`):**

- **ABS** `abs.building_activity` (ABS Building Activity): dwelling units commenced/completed/under construction and value of work, by building type (houses, other residential, total residential, non-residential), sector (public/private/total), work type (new/alterations/total), original/seasonally adjusted/trend, Australia + states, quarterly 1955-Q1 to 2026-Q1, one ingestion vintage. Anomaly to confirm: "Dwelling units completed" × "Total Work" × Total Sectors seasonally adjusted/trend rows hold ~24.7 million (looks like a value series mislabelled as a count) — a data bug or a trap a correct answer must avoid.
- **Treasury** Budget Papers 2025–26 (retrievable): BP1 p.25 ($33 billion housing plan, National Housing Accord target of 1.2 million homes over five years, "Nearly 45,000 new homes were completed in the first quarter of the Accord"); BP1 p.49 Table 2.2 dwelling investment forecasts; BP1 p.50 construction capacity; BP2 p.87 / BP3 p.72 prefabricated and modular housing measures; BP1 pp.323–326 and Treasury PBS p.278 Housing Australia Future Fund and 30,000 social and affordable dwellings.
- Cross-check available: ABS new residential completions, total sectors, 2024-Q3 are 45,930 original / 44,773 seasonally adjusted / 45,477 trend against the Budget's "nearly 45,000" for the Accord's first quarter.

If later cases need coverage beyond this, record a bounded onboarding prerequisite or an expected coverage-gap answer; do not silently substitute a nearby measure.

| Case | Expected behaviour |
|---|---|
| Unqualified anchor question (free-form discovery, no injected dataset) | Discover the ABS table and the Accord passages, then clarify material ambiguity (commenced vs completed, period, series) or explicitly state a supported scope; do not silently pick a measure. |
| Completions over a specified period against the Accord target | Execute validated SQL for the stated measure and period, compute the comparison with the target's pace deterministically, and cite the target, its start date and its definition. |
| Treasury forecast or target versus ABS actuals | Distinguish targets, budget forecasts and observed actuals; say which period each covers. |
| Compare an ABS result with a Treasury claim (e.g. the Accord's first-quarter completions) | Use both sources substantively; identify agreement and explain residual differences (series adjustment, revisions since publication) rather than merely attaching two citations. |
| Incompatible measures, double counting, or missing coverage | Refuse to compare dwelling investment (national accounts, $) with dwelling counts, or social/affordable commitments with public-sector builds; never sum totals with their components; report periods after the latest ingested quarter as a coverage gap. |

Each answerable mixed case must require both a structured result and documentary evidence. Include absent concept links, an irrelevant same-topic passage, and an outdated release as negative tests. Published sources establish the gold facts; the extraction model's interpretation is not its own scoring oracle. Source-backed automated case generation is allowed, but cases without independently checkable expectations do not become release gates.

### Evaluation and the public site

All eval kinds are tracked in one place: the existing public site at [big-lake.github.io/evaluation](https://big-lake.github.io/evaluation/), uplifted rather than replaced. The publishing model is unchanged: case YAML merges to evaluation `main` by PR (generated cases via `bot/gold-cases`, hand-written via a feature branch); only official CI runs land on the `results` branch; `publish-site.yml` rebuilds the static site from `results` with whitelisted fields only. No backend, database or login.

Gaps closed in Phase 0, before the first mixed cases merge (otherwise the baseline produces results nothing can display):

1. **Mixed case kind.** `cases.py` accepts only `data` (`DQ-`/`DH-`/`DX-`) and `knowledge` (`KQ-`). Add `mixed` (`MQ-`; `MH-` for hand-written coverage-gap/negative cases) whose gold *composes* the existing shapes rather than inventing one: the numeric part reuses data gold (publisher figure, verbatim quote, gold SQL that must reproduce it via `/query`), the documentary part reuses knowledge gold (documents that must be cited, facts the answer must state), plus `expected_scope` and `expected_behaviour` (`answer | clarify | side_by_side | coverage_gap`). Healthcare cases enter as expected coverage-gap cases.
2. **Site shows only data cases.** `site.py` loads `kinds=("data",)`. Extend it to load every kind, with a section/filter per kind and **separate headline scores per kind** — never one blended score, which would let a mixed gain hide a data regression. Extend the whitelist with the mixed-case scores (scope verdict, comparability decision, citation check); still no system prompts, full traces, hostnames or SA identities.
3. **Licensing of published quotes.** Each case source records its licence; the site builder enforces it mechanically — republishable licence shows the quote, otherwise only the URL and page/section locator. Verify per resource (e.g. each Budget Paper), never assume an agency-wide licence.
4. **Knowledge-eval open decisions (evaluation plan §5.2).** Mixed cases force them: gold quotes come from the publisher's public copy (not the knowledge bucket's parsed text), and questions are phrased from companion documents or paraphrased, not from the answering passage. Settle them in the same evaluation ADR; this also unblocks the on-hold `KQ-` knowledge cases at little extra cost.

Stays off the public site: local experiments, intelligence's internal eval register (`intelligence/evaluation/results/`), held-out cases if ever introduced, and anything the whitelist already excludes. Only official runs are published — that rule is what makes the site credible.

### Scope and comparability

Maintain a small evidence record, not a new ontology or general rules engine. Dataset attributes live in `semantics`; document attributes are attached to the relevant passage/release in `knowledge`. Each attribute carries provenance and may be unknown. Intelligence checks them for the requested operation before SQL planning and again before final synthesis.

| Attribute | Required distinction for this slice |
|---|---|
| Measure definition | Commenced, completed or under construction; dwelling units versus value of work; building activity versus national-accounts dwelling investment; the Accord's definition of a counted home |
| Scope of dwellings | New versus alterations; houses, other residential or total residential; public, private or total sector; social/affordable commitments versus public-sector construction |
| Series adjustment | Original, seasonally adjusted or trend; which series a published claim used; annual totals from original series |
| Observation status and vintage | Actual, target or forecast; publication/revision date versus the period measured |
| Period and units | Quarter versus financial year; target period (five years from 1 July 2024); counts versus dollars, current price versus chain volume |
| Geography and grain | Australia versus state; totals versus components (no summing a total with its parts) |

An unknown required attribute blocks a claim of comparability, not retrieval. It triggers at most one targeted evidence retrieval, then clarification, a qualified side-by-side answer or abstention. LLM extraction can locate evidence but cannot establish missing compatibility. Automatic cross-dataset joins, roll-ups and weighted conversions remain outside this slice.

### Phase 0 result (official baseline, 2026-11-10)

Evaluation run `20261109-222123-1` ([CI run 42676098866](https://github.com/big-lake/evaluation/actions/runs/42676098866)), official, identity stable; intelligence `a85ffea`, api `394a499`, knowledge index `v1`. Per-kind headline scores are published separately.

| Kind | Result |
|---|---|
| data | 79/87 pass (90.8%), 0 invalid — unchanged |
| mixed (10 cases: 4 `MQ-`, 6 `MH-`) | **0/10 pass** (1 flaky, 7 fail, 2 incomparable from judge 429s) |

Mixed checks (passed of scored): numeric 0/8, scope 6/28, citation 5/22, comparability 3/28. By behaviour: side_by_side 4 fail; answer 1 fail; clarify 1 fail; coverage_gap 4 (1 flaky `MH-0002`, 3 fail). This is the number Phases 1–2 must move.

Findings carried forward:

- `abs.building_activity` "dwelling units completed" in dollars (M7 × Total Work × Total Sectors, SA/Trend) is an upstream ABS quirk (present in the ABS Data API), not an etl bug; encoded as a trap in `MQ-0003`.
- Vintages differ: the Mar-2026 ABS page matches the lakehouse exactly; Sep-2024 and Jun-2026 pages differ by revision. Gold pins the lakehouse vintage. `MH-0004` retires when `abs.building_activity` is re-ingested past 2026-Q1.
- Gap against the Phase 0 gate: `semantic_bundles` is empty in run records because `/health` does not report the semantic bundle id. Phase 1 adds it (intelligence change, own commit) before any vocabulary pinning relies on it.
- Still missing from the suite: an irrelevant-same-topic-passage negative case (added in Phase 1).
- The Langfuse mirror covers data runs only (relevant to the Phase 2 trace decision).

### Re-baseline with MH-0007 and fleet identity (official run 42772124301, 2026-11-10)

| Kind | Result |
|---|---|
| data | 78/87 pass, 2 flaky, 7 fail (was 79/87; within repeat variance, same intelligence 85ffea) |
| mixed (11 cases) | 1/11 pass: MH-0002 now passes (was flaky); MH-0007 (the same-topic negative) fails as expected; all side_by_side, nswer and clarify cases still fail |

Mixed checks (passed of scored): numeric 0/9, scope 6/33, citation 3/27, comparability 4/33; no incomparable repeats this time (judge 429s did not recur). semantic_bundles now populated from traces (20261108T052050Z-68a32f2); the /health field arrives after an intelligence deploy.

## Phase 1 progress

Started 2026-11-10. All Phase 1 commits are pushed (`big-lake/vocabulary` created by the user); no infra yet.

| Item | State |
|---|---|
| Org ADR-0005 (vocabulary repo + bundle; amends ADR-0001; answers open question 1 on IRI minting) | accepted, pushed |
| `vocabulary` repo: schema, `_spec/mapping-to-skos.md`, tier A SDMX import (6 `building_activity` codelists), tier B glossaries (ABS methodology 19 terms, Treasury BP1 2 terms), verbatim-quote validators, `rdflib` round-trip, SSRF-guarded snapshots, CI workflow, risks register | done, 12 tests pass |
| `/health` reports `semantic_bundle_id` (intelligence `37ca7dc`) and eval records it (evaluation `73e937b`) | pushed; takes effect after an intelligence deploy |
| ABS commenced / completed / under construction / dwelling / dwelling-unit counting definitions | sourced verbatim from the Building Activity methodology page's Glossary section. Every ABS release has such a methodology page, so a deterministic Glossary parser is the Phase 4 generalisation |
| Knowledge snapshot/applicability contract (knowledge ADR-0023) and binding pin contract (org ADR-0006) | written, `proposed`, awaiting user review |
| Bundle publication (GCS bucket, SA, workflow in `infra/` + GHA) | code done: vocabulary `bbf325f` (builder, atomic `current.json` publish, `--rollback-to`, `publish.yml` gated by `PUBLISH_ENABLED`, SETUP.md, 17 tests). Infra written on branch `feat/vocabulary-bundle-bucket` (`ff75746`, **local, not pushed**; fmt + validate pass): versioned `{project}-vocabulary` bucket, `biglake-sa-vocabulary-{env}` with `bundles/`-conditioned objectAdmin, environment-scoped WIF binding. **Nothing deployed**; needs the infra PR/GHA deploy, then the GitHub environments/secrets in SETUP.md. No consumer read grants yet |
| Irrelevant-same-topic-passage negative case | done: `MH-0007` (hand-written, coverage-gap: no annual Accord target in the Budget Papers), pushed in evaluation `80fdd40`. It joins the mixed suite, so the mixed baseline above (10 cases) predates it and is re-baselined on the next official run |
| Document-only retrieval baseline (before vocabulary boosts) | suite and runner pushed (evaluation `d95d2ce`; `KQ-0001..0004`). First dispatch [42696729271](https://github.com/big-lake/evaluation/actions/runs/42696729271) hit the overnight fleet shutdown and was correctly refused by the publisher. Second dispatch [42772124301](https://github.com/big-lake/evaluation/actions/runs/42772124301) published data and mixed but the **knowledge runner crashed** (`KeyError: trap_violations`, a bug in the new runner that unit tests missed; fixed with a regression test in evaluation `b16a356`). Crash fix confirmed by a local, non-official one-repeat run against the deployed `a85ffea`: 3/4 (`KQ-0001/2/4` pass; `KQ-0003` "size of the housing plan" fails because retrieval returned BP1 p.170, PBS and BP4 pages and missed the BP1 p.25 passage, so the system declined: a genuine document-retrieval gap, not a bad case). **Official baseline deliberately deferred** to the run that follows the intelligence deploy, so it also records the `/health` bundle id |
| Eval resilience (from the plan review): outage fail-fast and judge retry | done: evaluation `cdd110f`. Runners abort without writing a record after 12 consecutive infrastructure failures; the judge retries 429/503 up to 4 times. Documented in the evaluation README |

## Architecture

This is the target component map, not the build order. The first mixed slice reuses existing dataset discovery and document retrieval; the complete lexicon, discovery-card index and entity linking below are not prerequisites.

```
 SOURCES        ABS SDMX API      glossaries/standards (HTML)     PDFs/papers      other publishers
                     |                      |                          |            (same front door)
                     |                      +-----------+--------------+------------------+
                     |                                  v
                     |                 knowledge INBOX (PR/GCS, sha256, licence, kind)
                     |                                  |
                     |              +-------------------+--------------------+
                     |              | kind: vocabulary / both                | kind: knowledge
                     v              v                                       v
 vocabulary    harvest (tier A) + extract (tier B) [+ suggest (tier C)]   parse -> chunk -> enrich
 repo          SKOS/XKOS YAML in git, validators, PR audit trail           (entity mentions -> vocab IRIs;
                     |                                                      triples/Neo4j RETIRED)
                     v                                                            |
               vocabulary bundle -> GCS  (pinned by semantics)                    |
                     |                                                            |
       +-------------+--------------------+                                      |
       v                                  v                                      v
 semantics: field bindings         knowledge indexer loads vocab -> Postgres (vocabulary schema,
 (BIGLAKE ext: concept +           lexicon, concept links) + typed discovery cards -> pgvector
  scheme@version)                                 |
       |                                          v
       v                        intelligence tools: search (typed pools) | define_term |
 semantic bundle ------------->  resolve_value (candidates) | get_caveats | data_query
 (grounding + compiler)                           |
                                                  v
                      discover across publishers -> establish scope -> retrieve methodology ->
                      plan validated SQL -> BFF executes -> returns capped rows + provenance ->
                      check comparability -> synthesise rows + passages with claim-level citations
                                                  |
                      evaluation: definition / knowledge / mixed / caveat cases gate every phase
```

### Ownership boundaries (what each repo must not become)

| Repo | Owns | Must not become |
|---|---|---|
| `vocabulary` | Publisher-scoped concept schemes, definitions, classifications, explicit correspondences; the bundle | A hand-curated universal ontology; a subject-taxonomy monopoly (ADR-0001 consumers migrate only by their own choice, later) |
| `semantics` | Dataset field bindings, metrics, executable query constraints | A duplicate glossary |
| `knowledge` | Intake, snapshots, chunks, evidence indexes, the Postgres loader | An authority that promotes LLM guesses into facts |
| `catalog` | Governance + browseable projections (OM Glossary later) | A runtime dependency for definition lookups |
| `intelligence` + `api` | Evidence selection, validated execution, grounded synthesis | An unrestricted agent loop |

**Repo descriptions stay in sync:** each phase that changes a repo's role updates that repo's description — GitHub repo "About" text, `README.md` opening, and `.github/copilot-instructions.md` — to state both what the repo *is* and what it *is not* (per this table), plus the org repo map in `.github/copilot-instructions.md` and `ARCHITECTURE.md`. The `vocabulary` repo gets this at creation (Phase 1); `knowledge`, `intelligence`, `api` and `semantics` get theirs when the relevant phase lands; the Neo4j/graph mention is removed from descriptions in Phase R. This is part of each phase's definition of done, not a cleanup pass at the end.

## Vocabulary document model

YAML files in `vocabulary/` git, one scheme per file tree, mirroring SKOS/XKOS semantics with an explicit term mapping committed as `_spec/mapping-to-skos.md` + JSON Schema:

- **Scheme** (`skos:ConceptScheme`, XKOS classification fields where applicable): `id` (IRI — ABS URL where one exists, e.g. ASGS LOCI URIs), `publisher`, `version`, `licence`, `source` (snapshot ref + fetched date), `levels`/`notationPattern` for classifications.
- **Concept**: `prefLabel`, `altLabel[]`, `hiddenLabel[]`, `definition` (+ mandatory `source.url` + `source.quote` for tier B), `scopeNote`/`inclusionNote`/`exclusionNote`, `notation` (the code), `broader`, `validity` (`dcterms:valid` dates), `changeNote[]`, `status: authoritative | extracted | suggested`.
- **Correspondence** (`xkos:Correspondence`): only from published correspondence tables (tier A) or explicit source statements. Cross-publisher `exactMatch` is **never** emitted by similarity — `exactMatch` is transitive in SKOS and a wrong one poisons every scheme it touches; similarity produces `status: suggested` `relatedMatch` hints at most.
- Versioning: scheme files are `{scheme}@{version}`, matching the existing codelist convention; new versions are new files; `xkos:supersedes` links them; nothing is overwritten.

Publisher scoping preserves provenance and differing definitions: "Inflation (RBA)" and "Inflation (ABS)" remain distinct unless evidence establishes a specific relationship. It is not a retrieval boundary. Search all eligible publishers; prefer definitions applicable to the measure, not publisher identity alone, and attribute alternatives. Framework alignment alone does not establish equivalence. The model never forces non-ABS sources into ABS shapes.

## Intake (front door)

Knowledge inbox extended, PR/GCS only:

- Registry YAML gains `kind: knowledge | vocabulary | both` and a **mandatory `licence`** field. Open licence → full ingestion; unknown/restrictive → metadata + link only, no stored text (the eval site is public).
- `kind: vocabulary` resources route to a reserved `_inbox/_vocabulary/{publisher}/` segment — the ABS *subject* taxonomy is wrong for classifications that span subjects. `kind: both` (e.g. an ABS standard) registers once, feeds both consumers.
- **HTML snapshots are the one new ingestion capability** (promote-flow extension; borrow evaluation's snapshot pattern). Linked URLs are fetched once, stored immutably with sha256; every extracted statement references the snapshot. Scheduled re-fetch diffs produce a drift PR; a changed snapshot marks both the knowledge artefacts (existing cascade-stale) and the vocabulary extraction (hash comparison in vocabulary's drift run — no eventing) stale.
- Fetching is SSRF-guarded (https only, no private ranges, size/content-type caps); ids obey the ADR-0001 slug rule; extracted text gets the SEC-002 prompt wrapping like bundle text.

**Extraction tiers** (quality-over-volume is enforced here):

| Tier | Input | Method | Output status |
|---|---|---|---|
| A | SKOS/RDF, SDMX, classification CSV/XLSX with a declared column mapping | Code only (`rdflib`/`openpyxl`); auto-merge when validators pass | `authoritative` |
| B | Glossary pages, standard appendices | Structural parse (LLM fallback for messy markup); **definition must match a verbatim quote in the snapshot or the term is dropped** | `extracted` |
| C | Definitional sentences in prose | LLM detection + quote check | `suggested` — retrieval hints only, never cited, no approval path to authoritative |

Validators gating merge: quote-verified definitions, licence present, stable unique concept IDs, language-tagged label validity, per-source term caps and slug/IRI validity. Repeated labels across distinct concepts are allowed and must preserve ambiguity. Each new source PR includes source-grounded evaluation fixtures exercising its terms and a rejection/ambiguity case; generators may produce them, but expected facts must be independently checkable against publisher evidence, not accepted on an LLM's assertion.

Rejected proposals (tier B/C) are recorded (overrides-file pattern) so generators never re-propose them.

## Runtime

- **Postgres (existing knowledge_db instance, new `vocabulary` schema):** concept/label/hierarchy tables, a **lexicon** table (label + synonym + notation + scheme, trigram/FTS indexed) for value resolution, and doc→dataset/concept link tables. Loaded by the **knowledge indexer** from the vocabulary bundle; intelligence stays read-only. Reuse existing storage first and measure capacity. Prefer manifest-addressed object reads: no bucket-list permission is needed unless an actual list operation is required. Prefix-scoped object grants cannot authorize such a list operation.
- **Typed discovery cards** in the existing pgvector: dataset cards (from the semantic bundle) and concept/definition cards (from vocabulary), each a distinct `kind` searched as **separate candidate pools** with bounded top-k per type, merged after rerank. Document chunk retrieval is unchanged.
- **Capabilities introduced only as needed:** `define_term` returns attributed, scope-relevant definitions across publishers; `resolve_value` returns ranked code candidates scoped to dataset schemes; methodology retrieval uses explicit applicable links where available and ordinary hybrid retrieval otherwise. Missing links never suppress evidence. Implement these inside the existing services rather than requiring three new public endpoints. Dataset cards and lexicon expansion follow measured discovery/resolution failures; they are not prerequisites for the first mixed answer.
- **Bundle activation:** semantics' bindings pin `vocabulary_bundle_id`; the indexer validates every referenced IRI resolves before switching the active release; traces record both bundle ids.

## Grounded mixed synthesis (the actual unification)

Today's combined path emits prose and a SQL artifact that never meet. Closing that gap is the first functional milestone (Phase 2), not the last integration step:

discover datasets and documents across publishers → establish scope → retrieve applicable methodology/definitions → generate/validate SQL → **BFF executes (execution stays in the BFF, ADR-0004 boundary intact)** → capped rows + provenance return to intelligence → check comparability → one synthesis grounded in rows **and** evidence.

Reuse the post-execution `narrate` call pattern, not its rows-only verification unchanged. A request-scoped evidence envelope carries the question/scope, source and snapshot IDs, cited passages, SQL/query ID, capped rows with units/periods and truncation status, and bundle versions. Preserve provenance for each numerical and documentary claim. Number membership alone does not verify that a figure refers to the right measure, period or population. Comparisons and arithmetic require explicit compatible inputs and deterministic computation, never model-invented calculations.

The BFF sends trusted execution results to intelligence in a second bounded call; no result round-trip through user-editable client state. Evidence and results remain under the request's authorization scope. Define SSE ordering and duplicate/cancellation behaviour in the contract before implementation. Do not stream a final mixed conclusion before execution succeeds. On SQL or evidence failure, return a clearly labelled partial answer or coverage gap rather than a fabricated complete answer. The first slice allows at most one targeted follow-up retrieval, with explicit time/token/row limits; no open-ended planning loop or new persistent job service.

This changes mixed-path orchestration while retaining the existing router and tools. Existing single-source paths remain available behind a feature flag. The bounded general agent loop stays a non-plan until the static workflow demonstrably caps out on eval.

### Latency (mixed path)

The mixed path adds sequential steps (methodology before planning, execution, a second synthesis call), so mixed answers will have a later final conclusion than today's stream-immediately combined path. Measures to keep perceived wait minimal, built into Phase 2 rather than retrofitted:

- **Parallelise independent steps:** dataset discovery and document retrieval run concurrently; SQL generation may start once scope is established, overlapping with evidence retrieval where inputs allow.
- **Progressive streaming:** emit intermediate results as they become available — sources found, scope/clarification, the executed data table — before the final synthesis. Only the *conclusion* waits for execution + comparability; the user is never watching a blank stream.
- **Bounded work:** at most one targeted follow-up retrieval, with explicit time/token/row limits (above). Comparability checks are code, not model calls.
- **No regression elsewhere:** data-only and document-only paths are untouched; the mixed workflow sits behind its feature flag.
- **Measured, not assumed:** the Phase 2 latency/cost budget covers both time-to-first-useful-output and time-to-final-answer, set before implementation and reported per eval run.

## Phases

Each phase: user go-ahead → ADRs → build → eval gate → stop. One change-class per eval run. Later phases may be re-ordered or dropped on evidence; none starts implicitly.

| Phase | Scope | ADRs to write | Eval gate |
|---|---|---|---|
| **0 — coverage and acceptance baseline — DONE 2026-11-10** | ~~Verify ABS/Treasury coverage~~ (done: healthcare recorded as a coverage gap, anchor moved to housing construction). Resolve the `building_activity` completions anomaly (data bug vs trap). Add the `mixed` case kind + validator, uplift the site to all kinds with per-kind scores and licence-enforced quotes ([Evaluation and the public site](#evaluation-and-the-public-site)). Then a small mixed/negative suite, including free-form discovery without injected datasets and healthcare coverage-gap cases. Fix expected scope, facts and citations from independent source evidence. | evaluation ADR (required — existing kinds cannot represent mixed cases): mixed kind + gold shape + scoring, per-kind site sections, licence rule for published quotes, and the §5.2 knowledge-eval decisions | Reproducible official baseline visible on the public site per kind; answerable versus coverage-gap cases distinguished; numeric, scope, comparability and citation checks defined before feature work; existing data-case pages unchanged |
| **1 — minimum evidence and vocabulary** | Onboard only the ABS/Treasury releases, methodology and definitions needed by the slice through PR/GCS. Bootstrap `vocabulary` with SKOS/XKOS schema, a narrow SDMX import, source-backed definitions and versioned publication. Add HTML snapshots if those sources require them. Keep existing codelist copies; no bulk migration. | org ADR for vocabulary publication amending/superseding ADR-0001's central-artifact decision; vocabulary model and knowledge snapshot/applicability contracts; name retired duplication per the Ossie plan's perimeter rule | Required source facts and definitions available with provenance; compatible bindings validate; standards export checked. Document-only retrieval baseline measured before vocabulary boosts |
| **2 — first cross-publisher mixed answer** | Use existing discovery/retrieval, scoped vocabulary bindings and comparability records; retrieve methodology before planning. Implement the BFF execution-return contract and joint synthesis behind a flag. Add only discovery/resolution changes demonstrated necessary by Phase 0. No pre-linked concept requirement. | intelligence + api ADRs for bounded mixed workflow and execution-return contract; semantics binding/constraint contract | Housing construction mixed cases use executed data and relevant Treasury evidence; ambiguous cases clarify/state scope; incompatible figures stay separate; zero unsupported equivalences, double-counting or unsupported caveats in negative cases; current data-suite regression gate met; latency budget (time-to-first-useful-output and time-to-final-answer) and cost budget met |
| **3 — measured discovery and resolution improvements** | Expand typed dataset/definition candidate pools and the Postgres lexicon only for observed failures. Add candidate-based resolution and source-backed applicable links; run ablations with boosts off/on. | intelligence retrieval/resolution ADR and infra schema permissions as needed | Free-form recall and wrong-code rates improve on a failure class, with no hiding of unlinked documents or mixed/data regressions; ambiguous labels are not forced to match a gold interpretation |
| **4 — generalise intake and coverage** | Complete reusable `kind`/licence routing, glossary extraction and drift/hash invalidation beyond the pilot sources. Prove a second domain or publisher combination without changing the core schema/workflow. Quality over volume; full automation, no SME curation dependency. | Remaining knowledge intake and vocabulary adapter decisions, only where the pilot contract needs extension | New cross-publisher cases and extraction negatives pass; resource updates preserve provenance and last-good compatible activation; existing suites remain stable |
| **R — Neo4j retirement** (parallel, independent) | Remove unused triple output and graph writer/config dependencies after auditing consumers; the enrichment call serving other outputs remains. Verify deployed state/data before container removal via Terraform. Entity linking is separate and not a retirement prerequisite. | knowledge + infra retirement decisions | Existing retrieval unaffected; runtime/container absence verified after approved deployment; retain data or obtain explicit deletion consent |

For every behavioural phase, use the latest reproducible deployed baseline with service commits and bundle IDs, not a frozen 73-pass subset. The historical Ossie result is 73/81 pass^3 ([run 42437018692](https://github.com/big-lake/evaluation/actions/runs/42437018692)); it is context, not the current release gate. Score discovery recall, scope correctness, code selection, compatibility decisions, claim-level citations, supported abstention, latency and cost separately. Compare the existing baseline, document ingestion alone, vocabulary-assisted retrieval/resolution, and joint synthesis one change-class at a time. Set budgets before running each phase; report repeat variability and the visible-case limitation rather than tuning indefinitely.

**Non-plans** (each needs a new ADR with a named consumer + failing eval class): automatic cross-dataset joins/roll-ups (7b), bounded general agent loop, Ossie ontology/mapping documents, OM Glossary projection, ADR-0001 taxonomy-consumer migration, tier C at scale, exhaustive entity linking, UI upload channel. Neither broad harvesting nor a new graph/runtime service is a prerequisite for the first mixed answer.

## Risks & limitations

- **No SMEs** (standing): review catches structural errors, not statistical ones. Mitigations: provenance rules above + eval as backstop + publisher scoping limiting blast radius of a bad definition to its scheme.
- **Licensing:** verify permissions per resource, including third-party material, rather than assuming an agency-wide licence. Unknown/restrictive resources remain metadata-only under the intake policy; attribution and public-eval publication rules apply separately.
- **Injection/SSRF/paths:** SEC-002 wrapping for all vocabulary text entering prompts; fetch guards; slug validation. Register rows in each repo's `documentation/security/risks.md` at build time.
- **Volume crowding:** per-source caps + status-aware ranking (validator-enforced).
- **Two-bundle drift:** pinning + pre-activation validation (correction 7). Failure mode is "stale but consistent", never "dangling".
- **Capacity unverified:** reuse shared Postgres initially; measure index size, query latency and ingestion contention during the pilot. Do not promise capacity from code counts or nominal Neo4j memory settings alone.
- **False reconciliation:** a shared label, framework or publisher cannot establish compatible measures (e.g. Accord "homes" vs ABS dwelling completions, dwelling investment $ vs counts). Preserve unknowns and differences; a supported side-by-side explanation is a successful mixed answer.
- **Public quote publication:** the eval site republishes gold quotes; licence enforcement in the site builder (not convention) prevents publishing text we may not redistribute.

## Open questions (decide at the owning phase)

1. Phase 1: exact IRI minting rules for publishers without linked-data URIs (ABS glossary terms have page anchors; RBA may not).
2. Phase 2: reuse existing `clarify.request` for material scope/code ambiguity where possible; decide the exact UI contract with a browser check before adding a new card type.
3. Phase 2: capped-rows/evidence return endpoint, SSE ordering, authorization, truncation and partial-failure behaviour; settle before implementation, keeping SQL execution in the BFF.
4. ~~Phase 0: which ABS tables and Treasury releases cover the healthcare slice?~~ Answered 2026-11-09: none for all-government health spending, so the anchor moved to housing construction (`abs.building_activity` + Budget Papers 2025–26). Making healthcare answerable (ABS GFS, PBO deployment, or AIHW) remains an explicit bounded scope decision.
5. Phase R timing: independent of vocabulary delivery; inspect deployed state and retained data before scheduling removal.
