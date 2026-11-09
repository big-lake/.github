# Implementation Plan — Data Answers: Context Table, Answer Highlighting, Narration

**Status:** phases 1–3 built and deployed to test 2026-11-08 (intelligence ADR-0032); prod not deployed. Official eval 2026-11-09 (run 42541560625, intelligence `a85ffea`, api `394a499`): 79/87 correct on every repeat, 0 invalid, 0 regressions vs the 73/81 baseline.
**Prod gate:** eval gate met (above); widening on vs off was not isolated. Before prod, deploy with `ENABLE_CONTEXT_WIDENING=false` and `ENABLE_DATA_NARRATION=false` and flip them after, or accept the defaults (both on).
**Date:** 2026-11-08
**Repos touched:** intelligence, api, ui (evaluation later, semantics none)

## Objective

A data answer today is a bare table — often a single cell (`12894.0`). Make it:

1. **Highlighted** — the cell(s) that answer the question are visually marked
   in the UI's `data-table`, whether that's one cell, a run of cells, or a
   whole column.
2. **Contextualised** — the table shows nearby rows/columns (other years,
   sibling measures) so the answer sits in context, without ever changing the
   answer itself.
3. **Narrated** — a short text commentary above the table ("In 2024, there
   were **12,894** male Aboriginal and Torres Strait Islander births…"),
   generated **from the real executed rows** (ADR-0010 phase 2b, option (a)),
   arriving after the table so it never delays it.

## Current state (verified 2026-11-08)

```
intelligence  data_query.py → {"sql", "title", "nl_query"}        (LLM or semantic compiler)
    └─ orchestrator emits internal `data_sql_ready`
api           routes/agent.py `_agent_stream()` intercepts it,
    └─ execute_query(sql, limit=1000) → yields `data` event
       {columns, columnTypes, columnInfo, rows, totalRows, sql, executedSql,
        executionTimeMs, title, nlQuery, artifactId?, order?}   (AgentDataEvent)
ui            ChatTab.vue entry role 'data' → CellData.vue → DataTable.vue
              (no highlight hook, no prose next to tables;
               notebook insert copies the payload verbatim — ChatTab.vue ~L942)
```

- No post-execution text exists anywhere. ADR-0010 phase 2b deferred narration
  **because intelligence never sees executed rows** — the BFF runs SQL after
  intelligence's stream has moved on. Option (a) (BFF calls back into
  intelligence with the rows) was pre-identified and is what we build here.
- The semantic bundle already carries what context-widening needs:
  `dimension.is_time`, `observed_codes`, `label_of`, `role: unit`
  (see `semantics/datasets/*/semantic.ossie.yaml`).

## Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | **SQL marks its own answer rows** via a boolean `meta_focus` column; model also returns `focus_columns`. api strips `meta_focus` from the payload and emits `focus: {rows, columns, shape}` (row indices; `rows: null` = all rows). | DuckDB evaluates the predicate — no type/code mismatches from re-evaluating a filter in JS; UI logic is one rule: highlight cell if row flagged AND column in `focusColumns`. |
| D2 | **Widening is guarded by an exact/context SQL pair.** Intelligence returns today's narrow `sql` plus an optional `context_sql`. api runs both; the wide table is shown only if the exact values all appear in the wide table's focus cells. Otherwise fall back to the exact table (with a trace noting it). | Widening can only ever add context, never change the answer. Deterministic check, no LLM. Costs one extra ~0.2 s DuckDB query. |
| D3 | **Narration is option (a) of ADR-0010 phase 2b**: new intelligence `POST /agent/narrate`; api calls it *after* yielding the `data` event, then yields a `narration` event. | LLM stays in intelligence (api's query surface stays SQL-only); table latency unchanged; narration adds ~1–2 s to the tail of the turn only. |
| D4 | **Narration is one whole event, not a token stream**, and is dropped (not shown) if its figures fail verification or the call times out (~8 s). | Can't retract streamed text after a failed check. Table always stands alone; a lost narration is cosmetic. |
| D5 | **Verification is deterministic, in intelligence**: before the LLM call, compute summary facts (first/last/min/max/pct-change) in plain code; after, require every number in the text to match a focus-cell value or a computed fact (within rounding). Returns `verified: bool`; api drops unverified text. | Removes the confident-but-wrong failure mode ADR-0010 exists to prevent, without a second LLM judge. |
| D6 | Flags: `ENABLE_CONTEXT_WIDENING` (intelligence) and `ENABLE_DATA_NARRATION` (api) — planned default off until eval comparison passes (shipped default ON, see "As built"). | Each phase independently revertible; eval A/B before default-on. |

### Not building (no spaceships)

- **No per-figure provenance array** (`figures: [{text, row, column}]`) in v1 —
  hover-a-number-highlights-its-cell is a nice-to-have; `text` + `verified` is
  enough. Revisit only if users ask.
- **No streamed narration tokens**, no retries, no second narration attempt on
  verification failure — silence is the failure mode.
- **No compiler-path widening in v1.** The semantic compiler currently handles
  single-dataset plans and fell back on the motivating example anyway. v1
  widens via the raw LLM prompt + D2 guard; compiler widening is a listed
  follow-on.
- **No new infra, models, or services.** Same Gemini Flash tier as
  `data_query`; narrate is one more route on the existing intelligence app.
- **No UI redesign** — one CSS class on `<td>`, one prose block above the table.

## Protocol changes

### `data_sql_ready` (intelligence → api, internal — never reaches the UI)

```jsonc
{
  "sql": "...",                       // unchanged: narrow, exact answer
  "title": "...", "nl_query": "...",  // unchanged
  "focus_columns": ["Male births"],   // NEW optional
  "context_sql": "SELECT ..., (time_period='2024') AS meta_focus ...", // NEW optional (phase 2)
  "answer_shape": "single_value",     // NEW optional: single_value | series | other
  "artifact_id": "d1", "order": 0     // unchanged
}
```

### `data` event (api → ui) — additive only

```jsonc
{
  // ...existing AgentDataEvent fields...
  "focus": {                 // NEW optional — absent = no highlight (old behaviour)
    "rows": [5],             // row indices into rows[]; null = all rows
    "columns": ["Male births"],
    "shape": "single_value", // single_value | series | other
    "source": "context"      // "context" (widened, verified) | "exact" (fallback/no widening)
  }
}
```

### `narration` event (api → ui) — NEW, 0 or 1 per data artifact, after `data`

```jsonc
{
  "text": "In 2024, **12,894** male Aboriginal and Torres Strait Islander births were registered in Australia.",
  "artifactId": "d1",        // present when the data event carried one
  "verified": true           // always true on the wire — api drops unverified
}
```

### `POST /agent/narrate` (api → intelligence, internal, BFF-only per ADR-0004)

Request: `{trace_id, nl_query, title, answer_shape, columns, column_info,
focus_rows (≤50, focus cells only), computed_facts_allowed: true}`.
Response: `{text, verified}` (or 4xx/timeout → api emits nothing).

## Phases

Each phase ships and is useful alone. Order: plumbing → widening → narration.

---

### Phase 1 — Focus plumbing + UI highlighting (no widening)

User-visible: trend/comparison questions (which already return multi-row
tables) get their answer column/cells highlighted. Single-value answers
highlight their one cell.

**intelligence**
1. `data_query.py` `_SYSTEM_PROMPT`: instruct the model to also return
   `"focus_columns"` (the column(s) holding the asked-for values) and, when the
   result contains rows beyond the exact answer, to select a boolean
   `meta_focus` marking the answer rows. Response JSON gains optional
   `focus_columns` + `answer_shape`. Passthrough in `ToolResult.payload` and
   orchestrator's `data_sql_ready` emit.
2. Semantic-compiler path: `focus_columns` = the plan's metric output columns
   (deterministic); no `meta_focus` (exact result ⇒ all rows are the answer).
3. SQL validator: allow/ignore the `meta_focus` output column in the
   column-membership check.

**api** (`routes/agent.py`, `schemas/__init__.py`)
4. In the `data_sql_ready` interception: after `execute_query`, pop
   `meta_focus` from columns/rows; build
   `focus = {rows: [...indices] | null, columns, shape, source: "exact"}`
   only when `focus_columns` was supplied and names real result columns
   (defensive: intersect with actual columns, else omit `focus` entirely).
5. `AgentDataEvent`: optional `focus` nested schema. Contract-first loop:
   schema + route docstring → `python scripts/update_openapi_baseline.py` →
   Newman request asserting a `focus`-bearing stream → commit baseline.

**ui**
6. `DataTable.vue`: new optional prop `focus`; `<td>` gets class
   `cell-focus` when `(focus.rows === null || focus.rows.includes(ri)) &&
   focus.columns.includes(col)`. One CSS class in scoped styles (background
   tint + slightly stronger text — respect existing design tokens).
7. `CellData.vue` / `ChatTab.vue`: pass `focus` through; include `focus` in the
   insert-to-notebook payload (ChatTab.vue ~L942) and notebook save path
   (payload is already saved verbatim, so this is prop plumbing only).
   Old notebooks have no `focus` → no highlight → unchanged.

**Fallback behaviour:** model omits `focus_columns` (or names a non-existent
column) → no `focus` field → UI renders exactly as today. Nothing hard-fails.

---

### Phase 2 — Context widening (flag: `ENABLE_CONTEXT_WIDENING`, intelligence)

User-visible: single-value answers come back as a small table (e.g. last 6
years × male/female/all births) with the one answer cell highlighted.

1. **Prompt** (`data_query.py`): when the flag is on, ask the model for an
   additional `context_sql` following a fixed written policy:
   - answer filters (the question's own constraints) become the `meta_focus`
     predicate; scope filters (region etc.) stay in `WHERE`;
   - widen along **at most 2 axes**: time (recent periods ending at the
     answer's period) and ONE small sibling dimension pivoted to ≤5 columns;
   - **same unit only** — never mix counts with medians/percentages;
   - target ≤50 rows; if the exact answer is already ≥50 rows, return no
     `context_sql`.
   Prefer label columns (`*_name`) over code columns for the pivot headers —
   the Ossie `label_of` fields in the schema block make this possible.
2. **Guard in api**: when `context_sql` is present, run the exact `sql` first
   (cheap), then `context_sql`. Verify: every value in the exact result
   appears among the wide table's focus cells. Pass → emit wide table with
   `focus.source: "context"`; fail (mismatch, SQL error, or >50 rows) → emit
   the exact table as today with `focus.source: "exact"` plus a trace event
   noting the fallback. The exact `sql` (not `context_sql`) remains the `sql`
   field shown to users either way — it is the auditable answer.
3. **Eval before default-on**: run the e2e suite (and evaluation repo batch)
   with the flag on vs off; widening must not move answer accuracy. The
   `focus.source` field gives a measurable widening success rate for free.

---

### Phase 3 — Narration (flag: `ENABLE_DATA_NARRATION`, api) + ADR

User-visible: a sentence or two of commentary appears above the data table
~1–2 s after it renders.

1. **ADR first** (intelligence `0031-…`): records option (a) as the resolution
   of ADR-0010 phase 2b; amend ADR-0010's status line to point at it.
   Assumptions: BFF stays the SQL executor; narrate input is capped focus rows.
   Revisit when: intelligence ever gains result access another way, or
   narration needs more than focus rows.
2. **intelligence `POST /agent/narrate`** (new route beside `/agent/describe`):
   - compute facts in plain code from `focus_rows`: first, last, min, max,
     absolute + percent change (series); the single value (single_value);
   - one Gemini Flash call: question, title, column descriptions/units, focus
     rows (≤50), computed facts. Output: 1–3 sentences, markdown bold on
     figures, no tables/links/headings;
   - verify: extract every number from the text; each must match a focus-cell
     value or computed fact within rounding (reuse the `/agent/describe`
     output-sanitisation approach for markup/length);
   - return `{text, verified}`; soft-timeout per ADR-0013.
3. **api**: after yielding the `data` event, if the flag is on and `focus` was
   emitted, call narrate (timeout ~8 s, no retry) and yield `narration`
   (only when `verified`). Always still yield `done` after. On any
   error/timeout: a trace event ("Commentary skipped"), no `narration`.
   Add `AgentNarrationEvent` schema + OpenAPI baseline + Newman.
4. **ui**: `ChatTab.vue`/`CellData.vue` show a muted "Summarising…" line above
   the table once `data` arrives, replaced by the narration text (rendered via
   the existing sanitised `renderMarkdown()` path — never raw HTML) or removed
   on `done` without narration. Keyed by `artifactId` for combined turns.
   Notebook insert/save includes `narration` text with the cell; a cell saved
   before the narration arrives gets patched when the event lands (same
   pattern as other late-arriving artifact updates in Workspace.vue).

## Latency budget (per data turn, ~26 s today end-to-end)

| Addition | Cost | When user sees the table |
|---|---|---|
| Phase 1 focus | ~0 (same single query) | unchanged |
| Phase 2 widening | +1 DuckDB query ≈ 0.2 s | +0.2 s |
| Phase 3 narration | +1 Flash call ≈ 1–2 s | **unchanged** — table emitted first; only `done` is later |

## Security notes

- `meta_focus` is stripped server-side in api; it never reaches the UI or
  saved notebooks (also keeps the `meta_` namespace out of user-visible data).
- Narration text is model output rendered in the UI: must flow through the
  existing sanitised markdown renderer; intelligence re-checks for
  links/markup like `/agent/describe` does. No HTML, no URLs in output.
- `/agent/narrate` sits behind the existing BFF-only network posture
  (ADR-0004); its input is result rows the BFF already holds — no new data
  exposure. Cap `focus_rows` at 50 and total request size defensively.
- `context_sql` goes through the exact same read-only validation +
  LIMIT-wrapping in `execute_query` as every other query — no new SQL surface.

## Documentation to update alongside code

- intelligence: ADR-0031 (phase 3), README agent-surface event list,
  `data_query.py` docstring.
- api: route docstring + `AgentDataEvent`/`AgentNarrationEvent` schemas,
  OpenAPI baseline, Newman collection, BFF_API_REQUIREMENTS.md event table.
- ui: README/DESIGN notes for `cell-focus` styling token and narration block.
- This plan's Status line as phases ship.

## As built (2026-11-08) — deviations from the plan above

- **Flags default ON**, not off: `ENABLE_CONTEXT_WIDENING` (intelligence) and `ENABLE_DATA_NARRATION`
  (api, env read per request) so test could be exercised immediately. Kill-switch = set to `false` and
  redeploy. The eval A/B before prod is still owed.
- **No `meta_focus` in the phase 1 prompt**; it arrives with `context_sql` (phase 2 prompt).
- **Verification of the context table** compares the sorted answer-column values of the exact result
  against the `meta_focus` rows of the wide result (same column aliases required) — see
  `api/app/services/answer_focus.py::use_context`.
- **Narration request carries the answering rows with all columns** (≤50), not "focus cells only", so
  the model sees labels such as the year. Numbers are verified against every numeric value in those
  rows plus computed facts; `answer_shape` is passed through.
- **No "Summarising…" placeholder**: a persisted pending flag would stick in saved chats/notebooks, so
  the text simply appears above the table when it arrives. Notebook cells store `narration`.
- ADR is intelligence **ADR-0032** (0031 was taken).
- Newman covers none of this (needs a live LLM turn).
- Narrate timeout in the BFF is 10 s (plan said ~8 s).
- `use_context` returns an explicit `widened` flag; `focus.source` no longer infers it from a
  `meta_focus` column in the result (api `394a499`).
- **Post-test findings (2026-11-09, OPP-044):** the same trend question routed `multi_hop` twice and
  `data` once, and one data run added `obs_status IS NULL` (blank `obs_status` is `''` in etl output)
  and returned 0 rows. Fixed with `temperature=0` on `classify_query`/`data_query`, a router example
  and a prompt rule against status-column filters (intelligence `61cec7b`). Known limits: the
  decline-to-data fallback is still `factoid`-only; etl writes `''` rather than NULL.
- **Calculated answers (2026-11-09, intelligence `45539ab`):** the context-table prompt now also widens a percentage/ratio answer (single value or series) with its input columns plus at most one comparable value, computed from the displayed inputs so the existing exact-vs-wide check covers the inputs transitively; the comparable value is unverified. Calculated figures are `ROUND(..., 2)` in both queries, and narration may round decimals to 1-2 places. No api/ui change; the answer stays identifiable via `focus` (no separate exact array). Mixed-unit tables make the chart toggle unhelpful.
- Known, accepted: narration verification accepts a figure matching any numeric cell in the answer
  rows (not only focus cells); narrate runs inline in the SSE loop, so in combined turns it can delay
  later events by up to its timeout.

## Open questions (decide before the phase that needs them)

1. Phase 2: widen on the semantic-compiler path too, or accept the raw-path
   prompt only? (Compiler widening is deterministic and strictly better, but
   the compiler currently loses most multi-dataset turns to fallback anyway.)
2. Phase 3: should chat-surface (`/chat/stream`) data answers also narrate, or
   agent surface only? (Plan assumes agent surface only — chat currently has
   its own synthesis prose.)
3. Evaluation repo: adopt `focus` as the extraction target for data-case
   scoring (replaces guessing which cell is the answer) — coordinate once
   phase 1 is on test.
