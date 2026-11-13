# Cross-publisher unified intelligence — plan

**Status: Phase 0 done (2026-11-10); Phase 1 done (2026-11-10); Phase 2 in progress (approved 2026-11-11). See [Current status](#current-status-2026-11-14).** Each phase needs a user go-ahead; the ADRs named per phase are written when that phase starts, not up front.

- **Why:** answer cross-publisher questions using compatible, traceable evidence from structured datasets and unstructured documents. Vocabulary supports discovery and interpretation; building a concept layer is not the end goal. The first functional milestone is one complete mixed answer, without a graph database, an agent framework, or an LLM-invented ontology.
- **Shaped by:** iterative design reviews and the user's cross-publisher priority (2026-11-09). The corrections are binding and recorded in [Design corrections](#design-corrections-binding).
- **Standing constraints:** `.github/TODO.md` is not edited without user confirmation; infra changes only via Terraform + GHA; prod/destructive actions need explicit user OK; eval-first — a phase ships only when its named eval gate moves; nothing merges to or deploys `semantics`, `intelligence`, `api` or `knowledge` while an official eval run is in progress (a semantics merge redeploys the bundle and voids the run's identity, as in run 42891794665).

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
| 8 | (2026-11-10) **One registry shape** in `knowledge/configuration/{domain}/` for every resource: the existing per-document YAML plus `source_url`, `licence`, `kind: knowledge \| vocabulary \| all` and optional `applies_to`. |
| 9 | (2026-11-10) **Online documents are registered by URL, not uploaded:** an optional fetch step in the existing `open-inbox-pr.yml` writes the snapshot into `_inbox/`, and the existing scaffolder pins its sha256 in the registry PR. Downloaded files still go to `_inbox/` by upload. |
| 10 | (2026-11-10) **HTML is kept as fetched** (no print-to-PDF): the page's own markup drives deterministic glossary extraction. |
| 11 | (2026-11-10) **Tier B definitions are produced by an extractor**, not hand-authored. The Building Activity methodology is the first end-to-end case, registered as `kind: all`. |

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
| data | 78/87 pass, 2 flaky, 7 fail (was 79/87; within repeat variance, same intelligence a85ffea) |
| mixed (11 cases) | 1/11 pass: MH-0002 now passes (was flaky); MH-0007 (the same-topic negative) fails as expected; all side_by_side, answer and clarify cases still fail |

Mixed checks (passed of scored): numeric 0/9, scope 6/33, citation 3/27, comparability 4/33; no incomparable repeats this time (judge 429s did not recur). semantic_bundles now populated from traces (20261108T052050Z-68a32f2); the /health field arrives after an intelligence deploy.

## Current status (2026-11-14)

The single place to resume from. Superseded resume snapshots are archived (see [Archive](#archive)).

- **Phases:** 0 and 1 done; 2 in progress (about 70%); 3, 4 and R not started. Phase 4's trigger has fired.
- **User value so far:** definitions from the ABS Building Activity methodology and the BP1 housing passages are
  answerable with citations (knowledge 5/5, run 42908854992). No user can yet receive a mixed answer that joins
  ABS rows with Treasury evidence: the last official mixed score is 1/11 (run 42958997358, old combined path).
- **Deployed to test:** intelligence `a54a017` (mixed route on, mixed workflow off), api `ba076ec`
  (`ENABLE_MIXED_CONCLUSION` off), semantic bundle `20261113T082957Z-96038f5`, claims `76b5a4fb...`, data
  through June 2026. The UI serves `54f00de`; ui#5 (`1e67aee`, mixed-conclusion renderer) is merged but not
  deployed.
- **Running:** flag-off BFF official baseline, evaluation run 43495860904. Merge/deploy freeze on semantics,
  intelligence, api and knowledge until it ends. Read mixed with `MH-0004` excluded (stale).
- **Budgets (intelligence ADR-0034):** first useful output p50 <= 30 s; final answer p50 <= 60 s, p90 <= 90 s;
  conclusion call <= 25 s without retry; mixed tokens <= 2x the flag-off baseline. Missing usage is unknown,
  never zero.
- **Next, in order:** (1) read the baseline; (2) retire `MH-0004` and deploy the UI (`1e67aee`) to test;
  (3) paired-flag trial: one infra PR enabling both flags on test, deploy plus VM restart, one official BFF run,
  browser-check the conclusion render, revert both flags in the same session if any gate fails; (4) Phase 4a.
- **Biggest risk to the Why:** the compatible-answer path only works for documents with approved claims (five,
  one Budget Paper). Phase 4b addresses it.

## Phase 1 progress

### Phase 1 result (official run 42897354117, 2026-11-10)

All records official, identity stable; intelligence `37ca7dc`, api `394a499`, semantic bundle `20261110T202449Z-0b557f2` (pins vocabulary `voc-6fc8fa64a77e`).

| Kind | Result |
|---|---|
| data | 79/87 pass, 1 flaky, 7 fail (within repeat variance of 78–80) |
| knowledge (first official baseline) | 3/5 pass: `KQ-0002`, `KQ-0004`, `KQ-0005` 3/3; `KQ-0001` flaky 2/3; `KQ-0003` 0/3 (BP1 p.25 not retrieved; Phase 2 entry item) |
| mixed (11 cases) | 0/11 pass, 1 flaky (`MH-0002` 2/3, was 3/3), 10 fail. Checks: numeric 0/9, scope 7/33, citation 4/27, comparability 3/33 (was 0/9, 6/33, 3/27, 4/33) |

Reading: the methodology is retrievable and answerable as a document (Phase 1's evidence goal); the mixed baseline is unchanged within variance, as expected, because nothing in Phase 1 reaches the mixed path (no runtime consumer of the vocabulary; joint synthesis is Phase 2). The Phase 1 gate items are met except the retrieval miss carried as the Phase 2 entry item.

Phase 1 item detail is under [Phase 1 progress detail](#phase-1-progress-detail).

## Phase 2 progress

### Accepted review correction (2026-11-13)

Approved by the user: "approved, lets do it". Phase 2's scope, budgets and safety
gate remain unchanged. The review's deployment correction above stands; code
release and CI success are not runtime activation or a passed acceptance gate.

1. Close the gate-integrity gaps before a measured run: record and compare
  `claims_sha256` (including public run records), expose `transport: bff` in
  official CI, select operations from the request with one unambiguous claim,
  and refuse sums without explicit disjoint parts. Intelligence ADR-0036 amends
  ADR-0035; the original ADR body is preserved.
2. Resolve MQ-0003 without rewriting a source-backed claim to match gold. The
  2026-11-13 read-only test query confirms Total Work / All Buildings = 43,816
  (the independently published figure), while New / Total Residential = 43,666.
  MQ-0003 keeps its numeric gold and becomes an explicit side-by-side scope
  case. A source-backed New Residential pace case is still required to meet
  the Phase 2 compatible-pace gate; the probe alone does not establish gold.
  (Done: MQ-0005, sourced from the ABS Data API; see the pace gate decision below.)
3. Prepare test caller identity through Terraform's existing intelligence
  `extra_env` map, preserving mixed routing and keeping `ENABLE_MIXED_WORKFLOW=false`.
  API's conclusion flag stays false. Deployments and trial activation remain
  separate from offline validation. No prod change or new service is needed.
4. Review the five BP1 claims individually, publish approved claims through
  knowledge `embed_and_index`, and verify deployed commits, semantic bundle
  and `claims_sha256` before recording a baseline. Correction approval is not
  blanket approval of the draft attributes.
5. Measure an official BFF flag-off baseline, then the approved paired-flag
  trial with housing, negatives, regression, latency and token measurements.
  Direct-intelligence runs remain diagnostics, not the BFF gate. Preserve the
  no-merge/deploy freeze during official evals. Only consider normal enablement
  after every existing Phase 2 gate passes.

**Local implementation checkpoint (2026-11-13):** corrections in intelligence,
semantics, evaluation, infra and this plan were validated offline before merge: evaluation 127 passed;
intelligence mixed 28 passed; semantics 87 tests, one existing skip; test Terraform `validate` and `fmt -check`
passed; all 149 cases validated.

#### Deployment and activation status (2026-11-13, evening)

No official eval run was in progress at any merge or deploy.

- **Merged to `main`:** semantics#53, intelligence#26 (ADR-0036), evaluation#7 and #8, knowledge#11 and #12,
  infra#80 (reviewed, merged and Terraform-deployed by the user; Terraform Deploy succeeded 09:24 UTC), .github#3
  and #4.
- **Deployed to test:** API (Deploy API run 43393951296), intelligence (runs 43393958741 and 43399770487),
  knowledge flows (run 43397639068). The UI was not redeployed: test still serves `54f00de` (Deploy run 2026-11-10),
  so ui#5's mixed-conclusion renderer (`1e67aee`) is merged but not deployed (review 2026-11-14). Intelligence `/health`: commit
  `a54a017` clean (= `main`), semantic bundle `20261113T082957Z-96038f5`, `claims_sha256` `76b5a4fb...`,
  `enable_mixed_route` true, `enable_mixed_workflow` false.
- **Claims:** c-0001..c-0005 approved by `jezchaves` 2026-11-13 and published to `claims/v1/claims.json`. The first
  `embed_and_index` publish failed: unquoted commas inside YAML flow-mapping quotes split them into stray keys and
  broke `json.dumps(sort_keys=True)`; fixed by quoting (knowledge#12, with a serialisation regression test) and
  republished. `embed_and_index` only reaches the publish step when a document is selected, so a claims-only
  change needs `only_keys` plus `force`.
- **Data vintage:** `abs.building_activity` was re-ingested through the June 2026 release (ingest, bronze, silver
  via the Prefect REST API). The API pins its DuckDB views per worker, so it showed the old snapshot until the
  Deploy API restart. MQ-0001/0002/0003 gold was repointed to the June 2026 vintage (45,499, 45,499, 44,592; the
  March release showed 45,410 and 43,816). `PUBLISHER_HOSTS` now allows `data.api.abs.gov.au`.
- **Pace case:** MQ-0005 (New / Total Residential completions, Jun-26 = 47,010, from the ABS Data API under the
  site-wide CC BY 4.0, accepted by the owner) is `verified`, but its expected behaviour is `side_by_side`:
  c-0001 leaves the target's counting measure and start date unknown by design, so no pace ratio can be
  asserted. The Phase 2 "compatible pace" gate therefore has a verified source-backed case but still no case
  that demonstrates a computed pace.
- **Pace gate decision (owner accepted the recommendation, 2026-11-14):** the gate is met by correct behaviour,
  not by forcing a number. For Phase 2, MQ-0005 passes only as a qualified side-by-side that states the
  target's measure is unknown and asserts no ratio; any computed pace on it fails. A computed-pace case is added
  only when a source document itself states how the target is counted (for example a Housing Accord publication
  naming ABS new dwelling completions), captured as a new reviewed claim. No existing claim is edited to make
  arithmetic possible.
- **Intelligence VM** was stopped/started by the owner on 2026-11-14 after the Terraform deploy; `/health`
  afterwards: `a54a017` clean, bundle `20261113T082957Z-96038f5`, `claims_sha256` `76b5a4fb...`, mixed route
  on, mixed workflow off (`MIXED_CONCLUSION_CALLER_SA` is not exposed by `/health`).
- **Flag-off BFF official baseline:** dispatched 2026-11-14 (evaluation run 43495860904, all suites, mixed
  `transport: bff`, 3 repeats). No merge or deploy to semantics, intelligence, api or knowledge until it ends.
- **MH-0004 is stale (found 2026-11-14):** it is a coverage-gap case premised on the table ending at 2026-Q1;
  after the re-ingest the June 2026 figure is answerable, so its expected behaviour is wrong in the running
  baseline. Its result in run 43495860904 is not a valid signal. After the run, retire MH-0004 (status
  `retired`, never reuse the id) rather than rewriting it; a period-beyond-data gap case can be re-created later
  against whatever quarter is then unpublished in the lakehouse. The Phase 4 methodology-release item (Jun-2026
  methodology as a new document with `supersedes`) is now due as well.
- **Next:** read the baseline (latency p50/p90, conclusion-call time, token totals as the 2x reference), then the
  paired-flag trial: one PR enabling `ENABLE_MIXED_WORKFLOW` (intelligence, Terraform `extra_env`) and
  `ENABLE_MIXED_CONCLUSION` (api) on test only, deploy plus VM restart, one official BFF run, compare against the
  budgets, and revert both flags in the same session if any gate fails. Both flags remain off until then.
  The `.github/TODO.md` Phase 2 bullet was updated with the owner's confirmation (.github#4).

Mechanics of the shipped transport (unchanged from 2026-11-12): intelligence retrieves evidence before SQL
planning, relays a bounded (32 KiB) envelope to the BFF, and accepts only the configured BFF identity at
`/agent/mixed/conclude`. The BFF executes SQL, emits the exact table, returns at most 50 rows for one checked
conclusion, suppresses duplicate mixed artifacts and preserves the table on callback failure. The 25 s
wall-clock deadline covers ID-token minting; the intelligence model call is capped at 20 s; Python cannot kill
in-flight sockets or Vertex calls, so abandoned work runs to its own timeout. Token usage is reported per
stream and summed by the BFF and eval; any unreadable call makes the total null, never zero. The eval's
`--transport bff` path uses cookie-backed source resends and the BFF's executed rows; internal SQL-ready events
do not count as first output. The direct-intelligence transport remains the default diagnostic.

**Phase 2 arithmetic (review 2026-11-14):** after the pace gate decision no Phase 2 case requires a computed
cross-publisher comparison, so the gate does not exercise compatible arithmetic. That is accepted for Phase 2
and deferred: a computed case is added when a reviewed claim states a compatible measure (Phase 4b makes such
claims cheap to produce).

### Phase 2 log (2026-11-11, historical)

Approved by the user 2026-11-11.

**Historical resume point (end of session 2026-11-11; superseded by [Current status](#current-status-2026-11-14)).** State: no eval run in progress; no open PRs; all repos on `main` and pushed (local `infra` is parked on the merged `feat/vocabulary-bundle-bucket` branch and `catalog` is 1 commit behind; both are only stale checkouts). Deployed on test: intelligence `996c4c3` (`ENABLE_MIXED_ROUTE=true`), api `394a499`, semantic bundle `20261110T202449Z-0b557f2`. Latest official baselines: run [42958997358](https://github.com/big-lake/evaluation/actions/runs/42958997358): data 78/87 (2 flaky), knowledge 5/5, mixed 1/11 (see the "Next action" row below for the numbers and latency). Decisions waiting on the user: (1) accept or change the ADR-0033 proposed budgets (first output p50 ≤ 30 s, final p50 ≤ 60 s and p90 ≤ 90 s, conclusion call ≤ 25 s, ≤ 2x tokens) before the workflow build starts; (2) whether the stale infra comment "revert after" on `ENABLE_MIXED_ROUTE` is fixed in the next infra change (the flag stays on). Next steps in order: fix the budgets in ADR-0033 -> build the mixed workflow and BFF conclude call behind `ENABLE_MIXED_WORKFLOW` / `ENABLE_MIXED_CONCLUSION` (intelligence + api, one change-class each per official run) -> `comparability.yaml` -> claim-record ADR only if needed. Parked, not in this phase: OPP-050 (`MH-0003` / `MH-0004` table-selection declines, same class as OPP-043); the four ABS collection `TODO(curation)` placeholders (one catalog PR); the Phase 4 Jun-2026 methodology release case. Tooling: `run-eval.yml` has per-suite jobs (sequential by design, for comparable latency) with targeted `data_cases` / `mixed_cases` / `knowledge_cases` inputs that are never published; the offline router check is `evaluation/build/classify_offline.py` (git-ignored; run with `PYTHONPATH=src`, needs ADC). The IAP tunnel to `localhost:8001` goes stale: if `/health` times out, kill the old gcloud python process and re-run the tunnel task.

| Item | State |
|---|---|
| Entry item: BP1 p.25 retrieval miss | Diagnosed with `/retrieve` probes on test: the passage is indexed (BP1 parent `p_0038`, child `c_00093`, page 25) but (1) the BM25 leg returned nothing for natural-language questions because `plainto_tsquery` ANDs every lexeme, so retrieval was dense-only; (2) 21% of BP1 child chunks are degenerate fragments (axis labels, page headers, empty table grids) that outrank real passages on embeddings. Fix 1 = intelligence OPP-047 (`19e23ae`, deployed to test): p.25 now ranks #1 for the KQ-0001 question and #7 of top-8 for KQ-0003 (absent before). **Done 2026-11-11:** official run [42908854992](https://github.com/big-lake/evaluation/actions/runs/42908854992) (intelligence `19e23ae`, api `394a499`, bundle `20261110T202449Z-0b557f2`, all three records official and identity-stable), fix 1 the only change-class: knowledge 3/5 → **5/5** (`KQ-0001`, `KQ-0003` 3/3); data 79/87 → 80/87 (per-case diff not examined); mixed unchanged at 0/11 (numeric 0/9, scope 8/33, citation 5/27, comparability 5/33), as expected since the mixed path is untouched. Fix 2 = OPP-048 (knowledge re-chunk + re-index) is deferred: no measured miss remains; reopen only if p.25 retrieval turns flaky under mixed load. OPP-047 is archived as done |
| intelligence + api ADRs (bounded mixed workflow, execution-return contract) | **accepted by the user 2026-11-11** (trigger since decided and shipped, see the next row; the ADR-0033 status line still says it is open): intelligence [ADR-0033](https://github.com/big-lake/intelligence/blob/main/documentation/adr/0033-bounded-mixed-workflow-and-joint-synthesis.md), api [ADR-0028](https://github.com/big-lake/api/blob/main/documentation/adr/0028-mixed-turn-execution-return-contract.md), and org [ADR-0008](../adr/0008-dataset-scope-and-comparability-records.md) (dataset comparability records, housing slice). Amended 2026-11-11 for two findings: (1) the evidence envelope is stateless, relayed by the BFF, because intelligence runs `--workers 2`; (2) **the `broad` trigger does not fire on the housing slice**: a routing probe of six mixed cases gave `data` x3 and `factoid` x3, none `broad`, and the three `data` cases declined with no document retrieval (five cases not probed: the token expired). Trigger decided 2026-11-11 (user accepted the recommendation): a `mixed` router class behind `ENABLE_MIXED_ROUTE`, as its own change-class. Full probe (11 cases x3, stable): none `broad`; 6 `data` (5 decline, MQ-0001 SQL only), 5 `factoid` (documents only). Router PR [intelligence#23](https://github.com/big-lake/intelligence/pull/23) merged (next row). Measured first-output latency on today's single-path turns: p50 27.8 s (n=17), so the ADR-0033 proposed budget is now 30 s, not 20 s. Latency/cost budgets are proposed in ADR-0033 (baseline: data p50 5.0 s, knowledge p50 23.0 s, mixed p50 35.1 s / p90 75.3 s). The budgets are still "proposed" in ADR-0033, not fixed. Workflow build not started. Time-to-first-output is recorded since evaluation `c4fd7be`. Still missing: the knowledge-side claim-record ADR that ADR-0008 depends on |
| Catalog entry for `k-abs-building-activity-methodology` | merged 2026-11-11 ([catalog#11](https://github.com/big-lake/catalog/pull/11), `02aba12`); Sync to OpenMetadata dispatched for test (run 42918842484). Scaffold matches the other ABS collections (owner/tags/logo/links `TODO(curation)` left as in those; fill all four in one later pass). Licence verified 2026-11-11 against abs.gov.au/website-privacy-copyright-and-disclaimer: CC BY 4.0 for site material, excluding logos, trade marks, microdata and third-party content, none labelled on the page; registry comment updated (knowledge `3648127`). The page also shows "This is not the latest release" (Jun-2026 exists): the Phase 4 release-versioning case is live |
| `mixed` router class on test (change-class 2 of Phase 2) | **Done 2026-11-11:** [intelligence#23](https://github.com/big-lake/intelligence/pull/23) merged (`7c59ec2`), deployed, `ENABLE_MIXED_ROUTE=true` on test via `extra_env` (infra `e4c4e9f`); **decided 2026-11-11 to keep it on** (the infra comment "revert after" is stale and is fixed with the next infra change). Official run [42949194242](https://github.com/big-lake/evaluation/actions/runs/42949194242) (flag the only change-class; all three records official and identity-stable): knowledge 5/5 (unchanged); data 78 pass / 2 flaky / 7 fail of 87 vs 80/0/7 (same 7 fail; new flaky `DQ-0033` and `DQ-0069`, both SQL-generation 20 s LLM timeouts plus one `tsest_code` typo; the record does not show the route taken, so a routing cause is not ruled out; not yet re-run to confirm); mixed 0/11 as expected because a `mixed` turn still runs the old combined path (numeric 0/9, scope 11/33, citation 12/25, comparability 4/31); `MH-0003`, `MH-0004` and `MQ-0004` end in `error` with no SQL and no citations, and `MH-0001` asks to clarify where an answer is expected. Latency baselines (ms): data p50 4,961 / p90 16,161 (no first-output); mixed p50 32,571 / p90 86,692, first output p50 23,914 / p90 35,085; knowledge p50 34,699 (p90 491,101 is one outlier of 15), first output p50 13,491 / p90 28,327 |
| **Next action:** router prompt tuning for `MH-0003`, `MH-0004`, `MQ-0004` | **Done 2026-11-11** ([intelligence#24](https://github.com/big-lake/intelligence/pull/24) merged `996c4c3` and deployed to test, `/health` verified; OPP-049). Official run [42958997358](https://github.com/big-lake/evaluation/actions/runs/42958997358) (all three records official and identity-stable, intelligence `996c4c3`, bundle `20261110T202449Z-0b557f2`), router prompt the only change-class vs run 42949194242: knowledge 5/5 (unchanged); data 78 pass / 2 flaky / 7 fail (same 7 fail as before; flaky `DQ-0033`, a 20 s LLM timeout again, and `DQ-0050`, a "Goods credits" schema decline; within the 78 to 80 seen across runs, so the gate holds); mixed **1/11** (was 0/11: `MH-0002` 3/3) with 2 flaky (`MH-0007`, `MQ-0004`), checks numeric 0/9, scope 16/33 (was 11), citation 20/25 (was 12), comparability 9/31 (was 4). `MH-0004` and `MQ-0004` now route `mixed` and no longer error, but the `mixed` class still runs the old combined path so neither produces SQL (`MH-0004` fails scope 3/3; `MQ-0004` clarified once, which is the expected behaviour). Latency (ms): data p50 4,859 / p90 14,933; mixed p50 35,173 / p90 67,978, first output p50 23,332 / p90 30,227; knowledge p50 20,716 / p90 43,129, first output p50 12,444 / p90 24,269. The flag stays on. The merge/deploy freeze is lifted. Reordered 2026-11-11 (plan review, accepted by the user): `comparability.yaml` has no consumer until the mixed workflow exists, so it moves after the workflow. **Finding:** `MH-0003` is not a routing miss. It is a pure data question the router correctly sends to `data`, and it declines because discovery/SQL picks the wrong table ("dwelling stock, prices, transfers"); `MH-0004` also declines on "Building Approvals, not Completions". Filed as OPP-050 (same class as OPP-043), outside this change-class. The router change targets `MH-0004` and `MQ-0004` only (prompt-only, flag-gated): offline check over 6 runs of all 149 eval questions: both route `mixed` 6/6 (were `data`), no data or knowledge question is ever `mixed`, `MH-0002` moves `factoid` -> `mixed` 5/6 (watch it). Local harness: `evaluation/build/classify_offline.py` (git-ignored). Next: the ADR-0033 budgets and the workflow build (below); `MH-0003` and the `MH-0004` table-selection decline wait on OPP-050. Order after that: (1) a targeted re-run of `DQ-0033` / `DQ-0069` was done 2026-11-11 (run 42955363211, 3/3 each, so the data flakiness looks like noise; not published); (2) fix the ADR-0033 budgets, then build the mixed workflow and BFF conclude call behind `ENABLE_MIXED_WORKFLOW` / `ENABLE_MIXED_CONCLUSION`, with the comparability check returning `unknown` (a qualified side-by-side answer); (3) `semantics/datasets/abs/building_activity/comparability.yaml` per org [ADR-0008](../adr/0008-dataset-scope-and-comparability-records.md), its validator and CI, once the workflow can consume it; (4) the knowledge-side claim-record ADR (`proposed`), only if step 3's results show it is needed |

## Phase 1 progress detail

Started 2026-11-10. All Phase 1 commits are pushed (`big-lake/vocabulary` created by the user); infra#78 deployed to test 2026-11-10.

| Item | State |
|---|---|
| Org ADR-0005 (vocabulary repo + bundle; amends ADR-0001; answers open question 1 on IRI minting) | accepted, pushed |
| `vocabulary` repo: schema, `_spec/mapping-to-skos.md`, tier A SDMX import (6 `building_activity` codelists), tier B glossaries (ABS methodology 19 terms, Treasury BP1 2 terms), verbatim-quote validators, `rdflib` round-trip, SSRF-guarded snapshots, CI workflow, risks register | schema, validators, tier A import and bundle done, 17 tests pass (incl. bundle/publish). **The tier B terms were transcribed by hand** and only quote-checked: they are seeds/fixtures, not the deliverable. No tier B extractor exists yet (see [Course correction](#course-correction-2026-11-10-automated-source-intake-and-extraction)) |
| `/health` reports `semantic_bundle_id` (intelligence `37ca7dc`) and eval records it (evaluation `73e937b`) | deployed to test 2026-11-10 (Deploy Intelligence, 19:57 UTC); official run records show intelligence `37ca7dc` and the bundle id |
| ABS commenced / completed / under construction / dwelling / dwelling-unit counting definitions | present verbatim in the Building Activity methodology page's Glossary section (hand-transcribed into the seed scheme). Automated extraction of this Glossary is Phase 1 work, not Phase 4. Other ABS methodology pages are not assumed to share its layout; each is checked when onboarded |
| Retired duplication (perimeter rule) | named: the six tier A codelists in `vocabulary` copy `semantics/reference/abs/codelists`. Accepted for Phase 1; it retires when semantics reads its codelists from the vocabulary bundle (Phase 3+, with the first runtime consumer) |
| Slice sources onboarded into **knowledge** (retrievable at answer time), incl. the ABS Building Activity methodology page | **onboarded 2026-11-10**: registered by URL as `kind: all`, fetched as HTML (knowledge#9, `433db53`; collection `k-abs-building-activity-methodology`); `KQ-0004` and `KQ-0005` pass 3/3 in official run 42891794665. No `catalog/curation/knowledge/k-abs-building-activity-methodology/` entry exists yet. History: the print-to-PDF upload route (knowledge ADR-0024) is withdrawn: the methodology is registered by URL as `kind: all` and fetched as HTML through the corrected intake ([Course correction](#course-correction-2026-11-10-automated-source-intake-and-extraction)); the rendered PDF in Downloads is not used. Blocked on knowledge#8 merging (the dispatch inputs only exist on `main`); exact steps in [As built](#as-built-2026-11-11). After onboarding, update `KQ-0004` (expected decline today) and add a `KQ-` case for "what counts as completed". `KQ-0003`'s retrieval miss is separate |
| Knowledge snapshot/applicability contract (knowledge ADR-0023) and binding pin contract (org ADR-0006, amended by ADR-0007) | accepted 2026-11-10. ADR-0024 (agent-written print-to-PDF deferral, marked accepted without user agreement) is **superseded by knowledge ADR-0025** (in knowledge#8), which restores HTML fetching via the existing inbox automation and amends ADR-0023 decisions 1 (`all` replaces `both`), 2 (no reserved `_vocabulary` segment) and 3a (one capture shared with vocabulary). Decision 4 (`applies_to`) is still unimplemented. ADR-0007 is a course correction found while implementing: bindings derive from each table's `sdmx.yaml` codelist instead of a new per-field extension (one source of truth; Ossie files are regenerated) |
| Bundle publication (GCS bucket, SA, workflow in `infra/` + GHA) | code done: vocabulary `bbf325f` (builder, atomic `current.json` publish, `--rollback-to`, `publish.yml` gated by `PUBLISH_ENABLED`, SETUP.md, 17 tests). Infra PR [big-lake/infra#78](https://github.com/big-lake/infra/pull/78) merged whole (`9eedd10`) and deployed to test 2026-11-10; no bundle published (vocabulary `Publish` runs are skipped, `PUBLISH_ENABLED` unset). **Publication stays off until Phase 3 names a runtime consumer** (the knowledge indexer loader); the semantics pin check builds from `vocabulary_ref` and needs no bucket. Since `8a0a67f` (2026-11-11) it also grants the vocabulary SA a `documents/`-conditioned read on the knowledge bucket for `extract.yml`, so its SA + WIF are needed now; the bundle bucket itself still has no consumer (decision pending: merge whole or split). Original content (`ff75746`; fmt + validate pass): versioned `{project}-vocabulary` bucket, `biglake-sa-vocabulary-{env}` with `bundles/`-conditioned objectAdmin, environment-scoped WIF binding. Deployed to test; the vocabulary GitHub environment `test` exists. No consumer read grants yet |
| Semantics vocabulary binding and pin (the Phase 1 "compatible bindings validate" gate) | merged 2026-11-10: [big-lake/semantics#51](https://github.com/big-lake/semantics/pull/51) (`0b557f2`): `vocabulary.yaml` pins `voc-6fc8fa64a77e` (was `voc-bb185d206700`) with `vocabulary_ref` `d56790d`, and binds the six `abs.building_activity` dimensions; `python -m semantics.vocabulary` verifies against a vocabulary index (passes against the real one); the bundle index records the pin. 72 tests pass. CI step written (`9210e6a`) but gated by `VOCABULARY_CHECK`: builds the vocabulary bundle from a `vocabulary` checkout (no bucket needed; `python -m vocabulary build`). `VOCABULARY_CHECK=true` is set, and the check builds from the pinned `vocabulary_ref` rather than vocabulary `main` (review decision 6 in [As built](#as-built-2026-11-11)). The bundle id needed two fixes to be the same on Windows and Linux (vocabulary `336c26c`, `d56790d`) |
| Irrelevant-same-topic-passage negative case | done: `MH-0007` (hand-written, coverage-gap: no annual Accord target in the Budget Papers), pushed in evaluation `80fdd40`. It joins the mixed suite, so the mixed baseline above (10 cases) predates it and is re-baselined on the next official run |
| Document-only retrieval baseline (before vocabulary boosts) | suite and runner pushed (evaluation `d95d2ce`; `KQ-0001..0004`). First dispatch [42696729271](https://github.com/big-lake/evaluation/actions/runs/42696729271) hit the overnight fleet shutdown and was correctly refused by the publisher. Second dispatch [42772124301](https://github.com/big-lake/evaluation/actions/runs/42772124301) published data and mixed but the **knowledge runner crashed** (`KeyError: trap_violations`, a bug in the new runner that unit tests missed; fixed with a regression test in evaluation `b16a356`). Crash fix confirmed by a local, non-official one-repeat run against the deployed `a85ffea`: 3/4 (`KQ-0001/2/4` pass; `KQ-0003` "size of the housing plan" fails because retrieval returned BP1 p.170, PBS and BP4 pages and missed the BP1 p.25 passage, so the system declined: a genuine document-retrieval gap, not a bad case). **Official baseline deliberately deferred** to the run that follows the intelligence deploy, so it also records the `/health` bundle id. **Official baseline 2026-11-10** ([run 42891794665](https://github.com/big-lake/evaluation/actions/runs/42891794665), intelligence `37ca7dc`, semantic bundle `20261110T202449Z-0b557f2`): 3/5 pass, 1 flaky (`KQ-0001`, one repeat declined), 1 fail (`KQ-0003`, 0/3: BP1 is cited but the $33 billion passage is not retrieved) |
| Eval resilience (from the plan review): outage fail-fast and judge retry | done: evaluation `cdd110f`. Runners abort without writing a record after 12 consecutive infrastructure failures; the judge retries 429/503 up to 4 times. Documented in the evaluation README |
| Automated source intake + tier B extraction (the corrected Phase 1 core) | **merged, deployed to test and run once end to end (2026-11-10 UTC):** [knowledge#8](https://github.com/big-lake/knowledge/pull/8), [vocabulary#1](https://github.com/big-lake/vocabulary/pull/1), [infra#78](https://github.com/big-lake/infra/pull/78). Details, review decisions and the remaining sequence in [As built](#as-built-2026-11-11) |

### Course correction (2026-11-10): automated source intake and extraction

**What drifted.** The Phase 1 goal for definitions is an automated, repeatable loop that turns publisher methodology and glossary resources into source-backed vocabulary supporting the ABS↔Treasury comparison. What was built instead:

- The 19 ABS and 2 Treasury tier B definitions were transcribed by hand, then quote-checked. Quote validation proves provenance, not automation; nothing discovers or extracts them.
- knowledge ADR-0024 replaced the approved HTML snapshot capability (ADR-0023 decision 3) with hand-printing web pages to PDF and uploading them, and was marked accepted without user agreement.
- Uploading the methodology into knowledge was then presented as the next required step. Ordinary RAG ingestion makes the page searchable; it produces no vocabulary, and vocabulary extraction does not depend on it.

**Corrected approach (user, 2026-11-10).** One front door, reusing the existing inbox automation rather than a parallel pipeline:

```
 downloaded file:  upload to _inbox/{domain}/ ---------------------------------------+
 online document:  dispatch open-inbox-pr.yml (url, domain, licence, kind)           |
                     -> fetch step: allow-listed https fetch -> _inbox/{domain}/     |
                        (+ source_url, fetched_at metadata)                          |
                                                                                     v
   existing scaffolder: sha256, validation, guessed metadata, source_url/licence/kind -> registry PR
                                                                                     v
                              human reviews + merges (the approval gate)
                                                                                     v
   existing promote: sha256 re-check -> documents/{document_key}/raw/
          |                                                   |
   kind: knowledge | all                          kind: vocabulary | all
   existing chain: parse -> chunk ->               tier B extractor on the same pinned snapshot
   enrich -> embed_and_index                       -> quote-verified scheme PR on the vocabulary repo
```

- **One registry shape** in `knowledge/configuration/{domain}/`: the existing per-document YAML plus `source_url` (alternative to an uploaded `source_object`), mandatory `licence`, `kind` (default `knowledge`, so existing entries are unchanged) and optional `applies_to`. The Building Activity methodology registers as `all` (statistical commentary plus Glossary); Treasury BP1 becomes `all` (it sources the Treasury terms).
- **Online documents are registered by URL.** The fetch is an optional first step of `open-inbox-pr.yml`, not a separate workflow: the inbox SA (`biglake-sa-inbox-{env}`) already has `_inbox/`-only write, and the scaffolder sees the new object in the same run. Cron runs without a URL behave as today; scheduled re-fetch of registered URLs for drift later sits on the same cron and opens a drift PR, never a silent overwrite. Nobody writes a sha256 by hand: the scaffolder hashes the fetched bytes into the PR, so the reviewer approves exact content.
- **HTML is kept as fetched.** These are the publisher's bytes, and the page's own glossary markup is what makes deterministic extraction possible. URLs pointing to PDFs (e.g. Budget Papers) use the existing PDF parsers unchanged. Pages needing JavaScript or a login fall back to manual upload.
- **Every kind lives and lands in the same place.** `knowledge`, `vocabulary` and `all` resources alike are registered in `knowledge/configuration/{domain}/` (the canonical registry), go through the same scaffold PR and promote, and are stored at `documents/{document_key}/raw/{source_file}` in the knowledge bucket. Domain is purely organisational, so a classification spanning subjects just goes in its best-fit domain. ADR-0023's reserved `_inbox/_vocabulary/{publisher}/` segment is dropped. `kind` only decides which consumers run after promote: the RAG chain skips `kind: vocabulary`.
- **knowledge owns registration and capture; vocabulary owns extraction and validation** against that same pinned capture, proposing scheme changes as a PR on the vocabulary repo. Humans review diffs and exceptions; they do not transcribe.
- **Hand-off from knowledge to the vocabulary extractor** (pull, manifest-addressed, no new eventing):
  1. A vocabulary `extract.yml` workflow (cron + `workflow_dispatch`) checks out the knowledge registry read-only with a GitHub App token (both repos are private; same pattern as semantics' `catalog-checkout`).
  2. It selects entries with `kind: vocabulary | all` and an open licence whose `sha256` no vocabulary scheme records yet as its `source.sha256`.
  3. For each, it GETs `documents/{document_key}/raw/{source_file}` from the knowledge bucket (path computed from the registry, so no list call) and checks the bytes against the registry's pinned sha256 before extracting.
  4. It commits the snapshot to `vocabulary/snapshots/` (as the seed schemes already do) plus the generated scheme, and opens or updates a PR with plain `git` + `gh` (same pattern as `open-inbox-pr.yml`). Vocabulary CI then verifies every quote against the committed snapshot with no GCS access, as today.
  5. A new registry sha256 (a drift PR merged in knowledge) is picked up on the next run and produces a diff PR; nothing is overwritten.
- **Permissions:** read-only object GET on `documents/` in the knowledge bucket, plus Contents read on the knowledge repo. Concretely: the `biglake-sa-vocabulary-{env}` SA from infra#78 gains a `roles/storage.objectViewer` binding on the knowledge bucket conditioned to the `documents/` prefix (GET only, which a prefix condition can scope; no list needed), and the vocabulary GitHub App installation gets Contents read on `knowledge`. Licence-restricted resources are never stored, so they are never extracted. infra#78's bundle bucket stays held; its SA + WIF binding become needed for extraction, so the PR is split or reduced accordingly.
- **Repeatable means configuration, not code,** for a supported structure. A genuinely different structure needs a small adapter with fixture tests. Each new page's structure is checked when onboarded rather than assumed.

**Refactors of existing code:**

| Existing piece | Change |
|---|---|
| `knowledge/.github/workflows/open-inbox-pr.yml` | Optional `url`/`domain`/`licence`/`kind` dispatch inputs; fetch step before the scaffold step |
| new `knowledge/src/shared/fetch_to_inbox.py` | https only, host allow-list (`www.abs.gov.au`, `archive.budget.gov.au` initially), size/content-type caps, redirects re-checked; writes object + metadata to `_inbox/`. A security boundary (the runner holds the inbox SA's credentials): knowledge `risks.md` row |
| `knowledge/src/shared/scaffold_inbox_registry.py` | Accept HTML as well as PDF; copy `source_url`/`licence`/`kind` from fetch metadata |
| `knowledge/src/shared/config_loader.py` | Optional `source_url`, `fetched_at`, `licence`, `kind`, `vocabulary` (`applies_to` not yet) |
| `knowledge` RAG stages (`ingest_and_parse`, `chunk`, `enrich`, `embed_and_index`) | Select candidates with `DomainConfig.rag_document_keys()`, which skips `kind: vocabulary` (done at stage selection rather than in `find_files`, because each stage selects documents independently) |
| `knowledge/src/ingestion/promote.py` | HTML branch in the validation gate (today rejects anything without `%PDF-`) |
| `knowledge/src/ingestion/utils/pdf_tasks.py` `_dispatch_parser` | HTML parser alongside PyMuPDF/Docling |
| `vocabulary` | Tier B Glossary extractor + `extract.yml` (hand-off above), opening a scheme PR; existing validators unchanged |
| `infra` (infra#78) | Keep `biglake-sa-vocabulary-{env}` + WIF; add `documents/`-conditioned objectViewer on the knowledge bucket; bundle bucket stays held |

**Consequences.** The hand-authored tier B schemes remain seeds/fixtures until the extractor regenerates them; the gate requires extractor output. A new knowledge ADR supersedes ADR-0024 and amends ADR-0023 decisions 1 (value `all` replaces `both`), 2 (no reserved `_vocabulary` segment) and 3a (one shared capture). Build order: ADR → fetch step + scaffolder/loader fields → promote HTML gate + HTML parser + RAG-stage kind filter → onboard the methodology (`all`) → infra grant + App access → extractor + `extract.yml` scheme PR → eval (`KQ-0004` update, new definition case).

### As built (2026-11-11)

**Status update (verified 2026-11-11):** everything in the table below is merged: knowledge#8 (`71cf242`), vocabulary#1 (`2a75e5b`), infra#78 (`9eedd10`, Terraform Deploy to test succeeded). `EXTRACT_ENABLED=true` is set. The first real extraction is merged as [vocabulary#2](https://github.com/big-lake/vocabulary/pull/2) (`43506aa`): one 37-term scheme, `schemes/abs/building-activity-methodology@mar-2026.yaml`, from the knowledge-promoted capture; the hand ABS seed and the Treasury BP1 scheme are removed. The table describes the PRs as they were reviewed.

| Repo / PR | Contents | Tests |
|---|---|---|
| [knowledge#8](https://github.com/big-lake/knowledge/pull/8) `feat/url-intake` | ADR-0025 (supersedes 0024; amends 0023). `open-inbox-pr.yml` dispatch inputs `url`/`domain`/`licence`/`kind`/`publisher` (passed via `env`) + fetch step; `src/shared/fetch_to_inbox.py` (https, allow-list `www.abs.gov.au` + `archive.budget.gov.au`, public addresses only, manual redirects re-checked, HTML/PDF only, 50 MB cap; writes `_inbox/{domain}/[{publisher}/]{last-two-url-segments}.{html,pdf}` with GCS object metadata `source_url`/`fetched_at`/`licence`/`kind`/`document_name`); `src/shared/html_snapshot.py` (gate shared by scaffolder + promote); `src/ingestion/utils/html_parsing.py` (stdlib; sections titled by h2/h3 path, h4–h6 terms prefixed onto their paragraph, no tables); `parsers.html` in `configuration/pipeline.yaml`; scaffolder writes `source_url`/`fetched_at`/`licence`/`kind`/`parser: html`; `config_loader` validates the fields (licence required when `kind` ≠ knowledge or `source_url` set); `rag_document_keys()` in all four RAG stages; SEC-001 in new `documentation/security/risks.md`; README "Online documents" + WORKFLOWS.md. CI needed `httpx` added to `validate-test-release.yml` (`209b457`) | 81 pass; CI green |
| [vocabulary#1](https://github.com/big-lake/vocabulary/pull/1) `feat/tier-b-extract` | `python -m vocabulary extract --registry <knowledge configuration/> (--bucket B \| --raw-dir D) [--only keys] [--force] [--summary-out f]`; `extract.ADAPTERS` (data; `heading-glossary` = `<h2>Glossary</h2>` section, `<h4>` per term); first paragraph → `definition`, rest → `scopeNote`, verbatim; skips non-open licences (`OPEN_LICENCES`) and bytes already extracted (`source.sha256`); PDFs and unknown structures reported, not guessed; registry `vocabulary: {publisher, scheme, version}` names the scheme (fallback: document_key slug + `date_issued` or sha prefix). Schema gains `source.sha256`/`source.document_key`; `validate` checks the snapshot `# raw-sha256` header. `extract.yml` (daily 22:30 UTC + dispatch, App token, WIF, PR on `extract/vocabulary`) gated by `EXTRACT_ENABLED`. SEC-004 | 23 pass; CI green |
| [infra#78](https://github.com/big-lake/infra/pull/78) `feat/vocabulary-bundle-bucket` | `8a0a67f`: `roles/storage.objectViewer` for `biglake-sa-vocabulary-{env}` on the knowledge bucket conditioned to `documents/` | fmt + validate pass (test, prod) |

Dry run (local, not committed): the extractor on the live ABS Building Activity methodology page produced 37 Glossary terms; every quote passes the existing validator. The real scheme must come from the knowledge-promoted capture.

**Review decisions for the user** (settled 2026-11-10; as applied in the repos: 1 the four non-Glossary seed terms are dropped; 2 the `vocabulary` block was added by hand in knowledge#9; 3 infra#78 merged whole; 4 licence backfill deferred; 5 the BP1 scheme is dropped from the bundle; 6 `vocabulary_ref` recorded beside the pin and checked out by CI):

1. The extracted scheme replaces the 19-term hand seed. Four seed terms (ownership, seasonal adjustment, trend estimate, revisions) come from non-Glossary methodology sections and drop out unless an adapter covers those sections.
2. The scaffolder does not write the `vocabulary` block: in the registry PR add `vocabulary: {publisher: abs, scheme: building-activity-methodology, version: mar-2026}` to keep the existing scheme id.
3. infra#78: merge whole (bundle bucket still unused) or split out the SA, WIF and knowledge read.
4. Licence backfill for existing upload-only registry entries (ADR-0025 requires it only on the new paths for now).
5. **Treasury BP1 terms vs the Phase 1 gate** (plan review 2026-11-11). The gate allows no hand-authored definitions, but the 2 BP1 terms stay hand-transcribed (no PDF glossary adapter). Either scope the gate to the ABS methodology scheme and mark the BP1 scheme a seed until a PDF adapter exists (Phase 4), or drop it from the bundle until then. Either choice changes the bundle, and so the semantics pin.
6. **The semantics pin check compares against a moving target** (plan review 2026-11-11). semantics CI (`9210e6a`) builds the bundle from vocabulary `main` and fails unless its id equals the pin; the id is a digest of every scheme file, so any vocabulary scheme merge (the extraction PR first) fails semantics CI until re-pinned. That is not the "stale but consistent" behaviour of correction 7. Options: record the vocabulary commit beside the pin and check out that ref, or read the pinned bundle from the bundle bucket once published (ties to decision 3). vocabulary#1 changes no scheme files, so merging it is safe either way.

**Remaining sequence** (each push/merge/deploy/dispatch/secret step needs the user). **Status 2026-11-11:** steps 0–9 are done. Step 9: the cases are pushed (`KQ-0004` rewritten, `KQ-0005` added; evaluation `7518c2d`). Official run 42891794665's mixed record was refused (semantics#51's Publish changed the bundle mid-suite); re-run [42897354117](https://github.com/big-lake/evaluation/actions/runs/42897354117) published all three records as official (see [Phase 1 result](#phase-1-result-official-run-42897354117-2026-11-10)).

0. Settle the review decisions: 3 before step 2, 2 before step 5, 1 and 5 before step 6, 6 before step 7.
1. Merge knowledge#8 (its release deploys knowledge to the Prefect VM; the dispatch inputs exist only on `main`).
2. Merge infra#78 (whole or split, decision 3) and run the infra Terraform deploy (test).
3. GitHub: vocabulary environment `test` with `WIF_PROVIDER`, `WIF_SERVICE_ACCOUNT`, `PROJECT_ID`; a GitHub App with Contents read on `knowledge` installed for vocabulary, secrets `VOCABULARY_APP_ID` + `VOCABULARY_APP_PRIVATE_KEY`; variable `EXTRACT_ENABLED=true` (vocabulary SETUP.md). Secrets via the GitHub UI.
4. Merge vocabulary#1 (no scheme changes, so the bundle id and the semantics pin are unaffected).
5. Dispatch knowledge "Open PR with _inbox documents" with `url=https://www.abs.gov.au/methodologies/building-activity-australia-methodology/mar-2026`, `domain=people/housing`, `licence=CC-BY-4.0`, `kind=all`, `publisher=abs`; review the scaffold PR (the `vocabulary` block per decision 2, `date_issued`, and a new collection for this publication alone, never grouped by domain or batch), merge; promote + RAG chain run automatically.
6. Dispatch vocabulary "Extract vocabulary" (or wait for the cron); review and merge the extraction PR; delete the superseded hand snapshot `snapshots/abs/building-activity-australia-methodology-mar-2026.txt`; apply decision 5 to the BP1 scheme.
7. Re-pin semantics: implement decision 6, rebuild the bundle from vocabulary `main`, set `vocabulary_bundle` (in semantics#51 if still open, otherwise a follow-up PR) and re-run the verifier; then grant the semantics App read on `vocabulary` and set `VOCABULARY_CHECK=true`.
8. Deploy intelligence to test (`37ca7dc` is on `main` but undeployed) via its normal GHA deploy, so `/health` reports `semantic_bundle_id`; this also closes the Phase 0 carried-forward gap. Confirm with the eval `/health` check before step 9.
9. Eval: update `KQ-0004` (no longer an expected decline), add a "what counts as completed" `KQ-` case, then the deferred official knowledge baseline. That run's single change-class is the methodology onboarding (no vocabulary boosts exist yet); record `KQ-0003` (BP1 p.25 retrieval) explicitly in it.

Not built yet: scheduled drift re-fetch of registered URLs; `applies_to`; release versioning (`supersedes` on extracted schemes, release selection at retrieval; Phase 4); PDF glossary adapter (the Treasury BP1 scheme was removed in vocabulary#2); ABS methodology sections outside the Glossary.

## Architecture

This is the target component map, not the build order. The first mixed slice reuses existing dataset discovery and document retrieval; the complete lexicon, discovery-card index and entity linking below are not prerequisites.

```
 SOURCES        ABS SDMX API      glossaries/standards (HTML)     PDFs/papers      other publishers
                     |                      |                          |            (same front door)
                     |                      +-----------+--------------+------------------+
                     |                                  v
                     |     knowledge INBOX (upload, or URL fetched by open-inbox-pr.yml; scaffold PR pins sha256, licence, kind)
                     |                                  |
                     |              +-------------------+--------------------+
                     |              | kind: vocabulary / all                | kind: knowledge
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

Knowledge inbox extended, PR/GCS only, reusing the existing automation (`open-inbox-pr.yml` → `scaffold_inbox_registry.py` → registry PR → `promote-documents.yml` → `promote.py` → Prefect chain). Detail and build order: [Course correction](#course-correction-2026-11-10-automated-source-intake-and-extraction).

- One registry YAML shape in `knowledge/configuration/{domain}/` gains `source_url`, `kind: knowledge | vocabulary | all` (default `knowledge`), a **mandatory `licence`** and optional `applies_to`. Open licence → full ingestion; unknown/restrictive → metadata + link only, no stored text (the eval site is public).
- Downloaded files are uploaded to `_inbox/{domain}/`; online documents are registered by URL and fetched into `_inbox/{domain}/` by an optional step in `open-inbox-pr.yml`. Either way the existing scaffolder pins sha256 in the PR. `kind: all` (e.g. a methodology page with commentary and a glossary) registers once and feeds both consumers.
- **HTML snapshots are kept as fetched** (promote gate + HTML parser extension). Every extracted statement references the snapshot. Scheduled re-fetch on the `open-inbox-pr.yml` cron produces a drift PR; a changed snapshot marks the knowledge artefacts (existing cascade-stale) and the vocabulary extraction stale. knowledge and vocabulary use the same pinned capture (amends ADR-0023 decision 3a).
- Fetching is SSRF-guarded (https only, host allow-list, no private ranges, size/content-type caps, redirects re-checked); ids obey the ADR-0001 slug rule; extracted text gets the SEC-002 prompt wrapping like bundle text.

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
| **1 — minimum evidence and vocabulary** | Onboard only the ABS/Treasury releases, methodology and definitions needed by the slice through the existing inbox automation, extended with URL fetch and kept-as-fetched HTML snapshots. Bootstrap `vocabulary` with SKOS/XKOS schema, a narrow SDMX import, an automated tier B extractor (first case: the Building Activity methodology Glossary, `kind: all`) and versioned publication. Keep existing codelist copies; no bulk migration. | org ADR for vocabulary publication amending/superseding ADR-0001's central-artifact decision; vocabulary model and knowledge snapshot/applicability contracts; knowledge ADR superseding ADR-0024 (URL fetch in `open-inbox-pr.yml`, HTML kept, one shared capture); name retired duplication per the Ossie plan's perimeter rule | Required source facts and definitions available **to the answering system** (retrievable, not only stored) with provenance; the methodology's tier B scheme **generated by the extractor from the pinned snapshot** and merged via a reviewed PR (no hand-authored definitions in the gate; the Treasury BP1 scheme is dropped from the bundle until a PDF glossary adapter exists, Phase 4); compatible bindings validate against a pin that survives vocabulary merges; standards export checked. Document-only retrieval baseline measured before vocabulary boosts |
| **2 — first cross-publisher mixed answer** | Entry item, before the mixed workflow: fix the BP1 p.25 retrieval miss (`KQ-0003` 0/3, `KQ-0001` flaky in the Phase 1 knowledge baseline) as its own change-class, measured on the knowledge suite. Use existing discovery/retrieval, scoped vocabulary bindings and comparability records; retrieve methodology before planning. Implement the BFF execution-return contract and joint synthesis behind a flag. Add only discovery/resolution changes demonstrated necessary by Phase 0. No pre-linked concept requirement. | intelligence + api ADRs for bounded mixed workflow and execution-return contract; semantics binding/constraint contract | Housing construction mixed cases use executed data and relevant Treasury evidence; ambiguous cases clarify/state scope; incompatible figures stay separate; zero unsupported equivalences, double-counting or unsupported caveats in negative cases; current data-suite regression gate met; latency budget (time-to-first-useful-output and time-to-final-answer) and cost budget met |
| **3 — measured discovery and resolution improvements** | Expand typed dataset/definition candidate pools and the Postgres lexicon only for observed failures. Add candidate-based resolution and source-backed applicable links; run ablations with boosts off/on. | intelligence retrieval/resolution ADR and infra schema permissions as needed | Free-form recall and wrong-code rates improve on a failure class, with no hiding of unlinked documents or mixed/data regressions; ambiguous labels are not forced to match a gold interpretation |
| **4a — current methodology release (pulled forward, review 2026-11-14)** | Register the Building Activity methodology Jun-2026 release through the existing URL intake as a knowledge document in the same collection, so answers cite the methodology matching the data vintage. No vocabulary work: `supersedes`, `applies_to` and release selection stay in Phase 4. | None (existing knowledge ADR-0025 intake) | The Jun-2026 methodology is retrievable; knowledge suite stays 5/5; one `KQ-` case cites the Jun-2026 release |
| **4b — automated claim drafting (added, review 2026-11-14)** | Draft document claim records automatically from registered documents, quote-verified against the pinned snapshot like tier B, opened as a reviewed PR; humans approve, never transcribe. The compatible-answer path only works for documents with approved claims, so this is what scales the Why past one Budget Paper. Starts after Phase 2's gate. | knowledge ADR for claim drafting (amends org ADR-0009 only if the claim contract changes) | Claims drafted for a second document without hand authoring; every quote verifies; at least one new mixed case uses them; no unsupported equivalences in negatives |
| **4 — generalise intake and coverage** | Extend the Phase 1 intake and extraction loop beyond the pilot sources: scheduled drift re-fetch, further page structures (new adapters only where structure differs), wider host allow-list. **Release versions of the same publication** (e.g. Building Activity methodology Jun-2026 after Mar-2026, a new URL rather than drift): register the new release as a new document in the same collection; the extractor writes the new scheme version with `supersedes` set to the previous one; `applies_to` (ADR-0023 decision 4) records the data period each release describes; retrieval prefers the release matching the queried data vintage and an outdated-release case becomes a negative test. Trigger: the next `abs.building_activity` ingestion past 2026-Q1 (Mar-2026 matches the data vintage until then; that ingestion also retires `MH-0004`). **Triggered 2026-11-13:** the table now holds the June 2026 release, so the Mar-2026 methodology no longer matches the data vintage and `MH-0004` must retire (see the deployment status). Prove a second domain or publisher combination without changing the core schema/workflow. Quality over volume; full automation, no SME curation dependency. | Remaining knowledge intake and vocabulary adapter decisions, only where the pilot contract needs extension | New cross-publisher cases and extraction negatives pass; resource updates preserve provenance and last-good compatible activation; existing suites remain stable |
| **R — Neo4j retirement** (parallel, independent) | Remove unused triple output and graph writer/config dependencies after auditing consumers; the enrichment call serving other outputs remains. Verify deployed state/data before container removal via Terraform. Entity linking is separate and not a retirement prerequisite. | knowledge + infra retirement decisions | Existing retrieval unaffected; runtime/container absence verified after approved deployment; retain data or obtain explicit deletion consent |

For every behavioural phase, use the latest reproducible deployed baseline with service commits and bundle IDs, not a frozen 73-pass subset. The historical Ossie result is 73/81 pass^3 ([run 42437018692](https://github.com/big-lake/evaluation/actions/runs/42437018692)); it is context, not the current release gate. Score discovery recall, scope correctness, code selection, compatibility decisions, claim-level citations, supported abstention, latency and cost separately. Compare the existing baseline, document ingestion alone, vocabulary-assisted retrieval/resolution, and joint synthesis one change-class at a time. Set budgets before running each phase; report repeat variability and the visible-case limitation rather than tuning indefinitely.

**Vocabulary hold (review 2026-11-14):** the vocabulary bundle has no runtime consumer, and user-visible definition answers came from ordinary knowledge retrieval of the same methodology page. No further vocabulary work (new adapters, bundle publication, lexicon, `supersedes`) starts until a measured Phase 3 failure class names it as the fix. Existing CI, extraction and the semantics pin keep running.

**Non-plans** (each needs a new ADR with a named consumer + failing eval class): automatic cross-dataset joins/roll-ups (7b), bounded general agent loop, Ossie ontology/mapping documents, OM Glossary projection, ADR-0001 taxonomy-consumer migration, tier C at scale, exhaustive entity linking, UI upload channel. Neither broad harvesting nor a new graph/runtime service is a prerequisite for the first mixed answer.

## Risks & limitations

- **No SMEs** (standing): review catches structural errors, not statistical ones. Mitigations: provenance rules above + eval as backstop + publisher scoping limiting blast radius of a bad definition to its scheme.
- **Licensing:** verify permissions per resource, including third-party material, rather than assuming an agency-wide licence. Unknown/restrictive resources remain metadata-only under the intake policy; attribution and public-eval publication rules apply separately.
- **Injection/SSRF/paths:** SEC-002 wrapping for all vocabulary text entering prompts; fetch guards; slug validation. Register rows in each repo's `documentation/security/risks.md` at build time.
- **Volume crowding:** per-source caps + status-aware ranking (validator-enforced).
- **Two-bundle drift:** pinning + pre-activation validation (correction 7). Failure mode is "stale but consistent", never "dangling".
- **Capacity unverified:** reuse shared Postgres initially; measure index size, query latency and ingestion contention during the pilot. Do not promise capacity from code counts or nominal Neo4j memory settings alone.
- **False reconciliation:** a shared label, framework or publisher cannot establish compatible measures (e.g. Accord "homes" vs ABS dwelling completions, dwelling investment $ vs counts). Preserve unknowns and differences; a supported side-by-side explanation is a successful mixed answer.
- **Anchor passage retrieval:** `KQ-0003` shows retrieval missing BP1 p.25 (the Accord target, the $33 billion plan and "nearly 45,000" completions), the passage most mixed cases must cite. It is filed as a document-retrieval gap, but the Phase 2 gate depends on it: measured in the Phase 1 knowledge baseline (`KQ-0003` 0/3, `KQ-0001` 2/3) and now the Phase 2 entry item. Resolved: the knowledge suite reached 5/5 (run 42908854992).
- **Public quote publication:** the eval site republishes gold quotes; licence enforcement in the site builder (not convention) prevents publishing text we may not redistribute.

## Open questions (decide at the owning phase)

1. Phase 1: exact IRI minting rules for publishers without linked-data URIs (ABS glossary terms have page anchors; RBA may not).
2. Phase 2: reuse existing `clarify.request` for material scope/code ambiguity where possible; decide the exact UI contract with a browser check before adding a new card type.
3. ~~Phase 2: capped-rows/evidence return endpoint, SSE ordering, authorization, truncation and partial-failure behaviour.~~ Answered 2026-11-11 by api ADR-0028 (accepted); SQL execution stays in the BFF.
4. ~~Phase 0: which ABS tables and Treasury releases cover the healthcare slice?~~ Answered 2026-11-09: none for all-government health spending, so the anchor moved to housing construction (`abs.building_activity` + Budget Papers 2025–26). Making healthcare answerable (ABS GFS, PBO deployment, or AIHW) remains an explicit bounded scope decision.
5. Phase R timing: independent of vocabulary delivery; inspect deployed state and retained data before scheduling removal.
6. ~~Phase 1: registry field name.~~ Decided 2026-11-10: `kind`, values `knowledge` (default) | `vocabulary` | `all`. The vocabulary scheme files' own `kind` (`glossary`/`codelist`) is a different file type and does not clash in practice.
7. ~~Phase 1: hand-off from knowledge's snapshot to the vocabulary extractor, and its IAM.~~ Designed 2026-11-10 in [Course correction](#course-correction-2026-11-10-automated-source-intake-and-extraction): `extract.yml` pulls registry-addressed `documents/` objects and commits the snapshot with the scheme PR.
8. ~~Phase 1: where vocabulary-only resources live.~~ Decided 2026-11-10: same registry folder and GCS layout as every other kind; `kind` only selects consumers.

## Archive

Superseded resume snapshots were removed on 2026-11-14 to keep this plan readable: the 2026-11-13 merge-time checkpoint (and its superseded next actions) and the accepted review correction of 2026-11-12 (its budgets now live in [Current status](#current-status-2026-11-14) and intelligence ADR-0034). Full text: [the plan as of `1fcbd91`](https://github.com/big-lake/.github/blob/1fcbd91/documentation/plans/vocabulary-unified-intelligence.md).
