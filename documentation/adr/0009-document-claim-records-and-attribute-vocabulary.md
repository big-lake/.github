# ADR-0009 — Document claim records and the shared comparability attribute values (housing slice)

- **Status:** accepted (2026-11-12)

> 2026-11-13: the "operation from the scope step" assumption does not hold (the router emits only a class label); intelligence ADR-0035 picks the operation from claim status, row count and explicit "add up" wording.
- **Date:** 2026-11-12
- **Plan:** [vocabulary-unified-intelligence.md](../plans/vocabulary-unified-intelligence.md), "Scope and comparability" and Phase 2 step 3
- **Completes:** [ADR-0008](0008-dataset-scope-and-comparability-records.md) decision 6 (the document side) and fixes its undefined `equal`
- **Builds on:** knowledge ADR-0023 (`applies_to`, pinned snapshots, cascade-stale), knowledge ADR-0022 (corpus manifest)
- **Pairs with:** intelligence ADR-0033/0034 (consumer), api ADR-0028
- **Tags:** knowledge, semantics, comparability, mixed, contract

## Context

ADR-0008 gave the dataset side six attributes and a per-attribute rule (`compatible` / `different` /
`unknown`) but left four gaps:

1. **No document-side record.** The mixed conclusion (intelligence `mixed.py`) therefore reports all six
   attributes as `unknown` and is always partial. That is safe but caps every mixed answer at "side by side".
2. **"Both stated and equal" has no definition.** Dataset `value` and a document's wording are free text
   unless both sides draw from one closed set. Text matching would reintroduce similarity-based equivalence,
   which plan correction 9 forbids.
3. **Dataset values depend on the query, not the dataset.** Series, measure, geography, period and unit are
   properties of the filters a query applies, so a dataset-wide record cannot say what one returned row is.
4. **One rule for every operation is wrong.** The plan checks attributes "for the requested operation". A
   target and an actual differ in status and period by nature, so an all-attributes-compatible rule would
   forbid `MQ-0003` (pace against the 1.2 million target), which the acceptance set expects to be computed.

Walking the 11 mixed cases through the draft also showed two missing distinctions: tenure (`MH-0002`, social
and affordable homes versus ABS ownership sector) and a total-contains-parts relation (`MH-0003`).

## Decision

1. **Closed attribute values, owned here.** Each attribute's `value` is an object of facets drawn from the
   enums below. Equality is code equality, never text similarity. The housing-slice enums:

   | Attribute | Facets (closed enums unless noted) |
   |---|---|
   | `measure_definition` | `measure`: `dwellings_approved`, `dwellings_commenced`, `dwellings_completed`, `dwellings_under_construction`, `value_of_work_done`, `dwelling_investment_na`, `other` |
   | `dwelling_scope` | `build_type`: `new`, `alterations`, `new_and_alterations`; `dwelling_type`: `houses`, `other_residential`, `total_residential`; `sector` (ownership): `private`, `public`, `total`; `tenure`: `not_distinguished`, `social`, `affordable`, `social_and_affordable`, `market` |
   | `series_adjustment` | `series`: `original`, `seasonally_adjusted`, `trend` |
   | `observation_status_and_vintage` | `status`: `actual`, `target`, `forecast`; `published` (ISO date, informational) |
   | `period_and_units` | `period_type`: `quarter`, `financial_year`, `calendar_year`, `target_window`; `start`, `end` (ISO dates); `duration` (ISO 8601, e.g. `P5Y`, for target windows); `unit`: `count`, `aud_current`, `aud_chain_volume` |
   | `geography_and_grain` | `geography`: `AUS` or an ABS state code; `grain`: `total`, `component` |

   `other` never compares equal to anything. A new enum member needs a PR to this ADR's table, not a free
   string. Ownership `sector` is not tenure: ABS `public` sector is never `social` or `affordable`.

2. **Dataset side resolved per returned row, by code.** ADR-0008's `comparability.yaml` adopts the same
   facets and adds:
   - `facet_columns`: a map from result columns (or a tuple of columns, when one alone does not decide a
     facet) to facet values, each mapping with a verbatim source quote. Example: `tsest_code` `20` →
     `series: seasonally_adjusted`. Units need a tuple: for `measure_code M7` some `price_adj_code` and
     building-type combinations hold dollar values, not counts (the `abs.building_activity` known gotcha).
   - `dataset_facets`: facets true of every row (e.g. `tenure: not_distinguished`), each with a quote.
   - `parts`: stated total-contains-parts relations, e.g. `total_residential` = `houses` + `other_residential`.

   Intelligence resolves each attribute of a returned row only from columns present in the trusted
   execution return. A facet whose column is absent, or whose combination is unmapped, is `unknown`. The
   model never asserts a dataset-side value, and SQL text is not parsed for this.

3. **Document claim record.** One sidecar per document, `{document_key}.claims.yaml` next to the document
   YAML (knowledge ADR-0019 layout):

   ```yaml
   document_key: budget_paper_1_2024_25
   document_sha256: "<sha of the pinned source>"
   claims:
     - id: c-0001                       # stable within the document, never reused
       quote: "<verbatim text containing the figure>"
       figure: {text: "1.2 million", value: 1200000, unit: dwellings}   # as stated, no conversion
       attributes:                      # all six keys required
         measure_definition:
           status: stated               # stated | unknown
           value: {measure: other}
           source: {document_key: ..., quote: "<verbatim>"}   # or {url, sha256, quote} for a pinned snapshot
         series_adjustment: {status: unknown}
         # ...remaining four
       review: {status: approved, by: <github handle>}
   ```

   A `stated` attribute needs its own verbatim `source.quote`, which may sit on another page or in a pinned
   methodology snapshot. `unknown` is the default and is never inferred.

4. **Authoring.** An LLM may propose claims and quotes in a scaffolding PR, as the inbox scaffolder does
   today, but a record counts only after a human review sets `review.status: approved`. The model locates
   evidence; it never establishes a value (plan correction 5). The slice starts with the few claims the
   housing acceptance cases need, not a corpus-wide extraction.

5. **Validation (knowledge and semantics CI).** Fail on: a `quote` (claim or attribute) that is not a whitespace-normalised
   substring of the parsed text at `document_sha256` or the pinned snapshot; a value outside the enums;
   a `stated` attribute without a source; a missing attribute key; an unapproved record in the published
   set; a `facet_columns` mapping without a quote. A changed `document_sha256` marks the sidecar stale
   through the existing cascade-stale path and excludes it until re-reviewed.

6. **Linking to retrieved passages.** Chunk ids change on re-chunking, so a claim attaches to a retrieved
   passage only when `claim.quote` is contained in that passage's text, checked in code at query time.
   No stored chunk reference.

7. **Publication.** `embed_and_index` writes approved claims to `claims/v1/claims.json` beside the corpus
   manifest (knowledge ADR-0022) and records its hash in the manifest. Intelligence loads it the way it
   loads the manifest and adds each passage's matching claims to the evidence envelope (still under the
   32 KiB cap). `/health` reports the claims hash so a run records which claims were live.

8. **Per-attribute result (code only, replaces ADR-0008 decision 5's "equal").** `compatible` when both
   sides are `stated`, every facet is equal and none is `other`; `different` when both are `stated` and
   some facet differs; otherwise `unknown`. For `period_and_units` the compared facets are `period_type`,
   `start`, `end` and `unit`; `published` is shown and never compared.

9. **Operations.** Code first picks the operation the question asks for; each states which attributes
   *block* (must be `compatible`) and which may be *named* (an `unknown` is allowed but must be stated in
   the answer with what each side says). A `different` on any blocking or named attribute falls back to
   `side_by_side`, naming the difference.

   | Operation | Blocking | Named if unknown | Computation by code |
   |---|---|---|---|
   | `side_by_side` (always allowed) | — | — | none; each figure with its own stated attributes; no agreement claim |
   | `level_compare` | measure, dwelling scope, geography and grain, period and units | series adjustment, status and vintage | difference and ratio of the two figures |
   | `pace_vs_target` | geography and grain, `unit`, `build_type` | measure, rest of dwelling scope, series adjustment | pace = target ÷ number of the row's periods in the target window; ratio of actual to pace |
   | `sum` | every attribute except the component facet | — | sum, refused when a `parts` relation says one summand contains another |

   `pace_vs_target` also requires the dataset row's status `actual` and the claim's status `target` with a
   stated `target_window` `duration`. When the window `start` is stated, the row's period must fall inside
   it; when it is not, that is named. Its even-spread pace is labelled as an assumption, not a published
   figure. No conversion, roll-up or weighting is ever applied.
   An `unknown` blocking attribute permits one targeted retrieval (plan), then clarification, a qualified
   `side_by_side` answer or abstention.

   Housing check: `MQ-0003` → `pace_vs_target` (BP1 states "over five years" but no start date, so the
   window start and the target's measure are both named as unstated, as its gold answer does); `MH-0001` → `different` measure, status and unit → `side_by_side`; `MH-0002` →
   `different` tenure; `MH-0003` → `sum` refused by `parts`; `MQ-0001`/`MQ-0002` → `side_by_side` unless
   the Budget claim's dwelling scope is stated with a quote, then `level_compare`.

10. **Conclusion output.** The `conclusion` event reports the operation and, per attribute,
    `{attribute, dataset, document, result, named}` with the claim id used (api `AgentConclusionEvent`).
    The model writes no figure: computed values come from code and are quoted verbatim. `partial` is
    false only when the chosen operation's blocking attributes are all `compatible`, every named unknown is
    stated in the text, and every numeric statement is code-verified.

## Alternatives considered

- **Free-text values compared by an LLM or embeddings** — the equivalence-from-similarity failure the plan
  forbids, and it would grade itself in the eval's comparability check.
- **Claims stored in the vector index as chunk metadata** — tied to chunk ids that change on re-chunking.
- **Corpus-wide automatic extraction** — no reviewer, so no `stated` guarantee; also no consumer beyond housing.
- **Enums owned by `vocabulary`** — it stays definition-only (org ADR-0005); these describe series, not concepts.
- **Dataset side alone, document side permanently `unknown`** — the current state; caps every answer at partial.
- **One rule for all operations (all six `compatible` before any arithmetic)** — the first draft of this ADR;
  it forbids target-versus-actual pace, which by definition differs in status and period.
- **Model chooses the operation and fills unknown values from the question** — the model would be
  establishing scope; operations are chosen by code and unknowns stay unknown or come from a clarify answer.
- **Parse the generated SQL for dataset facets** — brittle and duplicates the executor; the returned
  columns are the trusted record of what ran.

## Consequences

- Reviewer effort per claim; quote validation keeps records honest but not complete.
- A closed enum will misclassify some real definitions as `other`, which yields `unknown`. That is the safe
  failure; the cost is fewer `compatible` answers, not wrong ones.
- Dataset `comparability.yaml` and the claim sidecars share one enum table, so enum changes touch two repos.
- The SQL planner must keep the facet code columns (e.g. `measure_code`, `tsest_code`, `region_code`) in the
  result, or every dataset facet resolves `unknown`. That is a data-query prompt/contract change.
- The operation set is closed; a question needing anything else gets `side_by_side`.
- The housing eval's comparability check improves only on cases that have an approved claim. Side-by-side and
  coverage-gap behaviour is unchanged by design.

## Assumptions

- The housing acceptance cases need on the order of ten claims, with source quotes available verbatim in
  the pinned documents and the ABS methodology snapshot (same assumption as ADR-0008).
- Parsed document text is stable for a given `document_sha256`.
- Knowledge ADR-0022's manifest loader in intelligence can load a second artefact next to the manifest
  (to confirm before build; otherwise load claims from a sibling GCS object by hash).
- Choosing the operation can be done by code from the question's scope step and the claim's status
  (`target` → `pace_vs_target`, several rows of one series → `sum`); if it cannot, the default is
  `side_by_side`. To confirm against the router's existing scope output before build.
- BP1 2025-26 states the target's duration verbatim ("homes over five years", checked in the eval snapshot)
  but not its start date; a start date needs another pinned source before window checks can pass.

## Revisit when

- A second dataset or publisher enters the mixed slice (do the enums generalise, or is `other` dominant).
- More than about thirty claims are needed (consider tooling beyond hand review).
- `other` accounts for most observed `unknown` results in eval traces.
- An acceptance case needs an operation outside the table.
- Ossie gains first-class comparability fields (merge the dataset side into it).
