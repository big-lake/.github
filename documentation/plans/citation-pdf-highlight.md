# Implementation Plan — Citation → Highlighted Snippet in Source PDF

**Status:** Phases 1, 3 and 3b live on test (2026-11-01); prod not deployed.
Styling refinements pending. Phase 2 (knowledge positions, re-index) pending
(knowledge resume).
**Date:** 2026-11-01
**Repos touched:** intelligence, ui, api, infra, knowledge (phase 2 only)

## Objective

Clicking **View source** on a knowledge citation chip opens the in-app knowledge
source tab (`KnowledgeSourceTab`), with the PDF viewer showing the **cited
document**, scrolled to the **cited page**, with the **cited passage highlighted**.

## Current state (verified 2026-11-01)

```
intelligence  RetrievedChunk {document_key, page, text, …}
    └─ chunk_to_citation() → Citation {chunk_id, label, document_title,
                                       snippet≈300ch, source_url=None, collection_id}
api           SSE pass-through (no transformation)
ui            Workspace.onChunk copies named fields → cell.chunks
    └─ View source → openKnowledgeSource({collection_id, title})
        └─ KnowledgeSourceTab: always selects documents[0],
           iframe = /knowledge/open 302 → signed GCS URL → browser PDF viewer
```

Gaps, in order of user impact:

| # | Gap | Where |
|---|-----|-------|
| G1 | Citation drops `document_key` → tab opens the **wrong PDF** in multi-document collections | `intelligence/app/models/schemas.py` `chunk_to_citation()` |
| G2 | Citation drops `page` → viewer never scrolls | same |
| G3 | `View source` silently no-ops in ChatTab and PublishedNotebook (no `openKnowledgeSource` handler) | `ui/src/components/ChatTab.vue`, `ui/src/views/PublishedNotebook.vue` |
| G4 | Chunk `page` is the **section start** page, not the passage's page (text chunks; table chunks are exact) | `knowledge/src/ingestion/utils/section_segmenter.py` |
| G5 | Parent expansion swaps the matched ~150-token child for its ~768-token parent and discards the child → citation granularity is the parent | `intelligence/app/services/retrieval.py` `expand_to_parents()` |
| G6 | No geometry: parser bboxes are discarded at segmentation; pgvector has no region columns | knowledge chunking + `pgvector_writer.py` |
| G7 | Browser-native PDF viewer in a cross-origin iframe cannot be scripted → no highlight possible | `ui/src/components/KnowledgeSourceTab.vue` |
| G8 | Citation `snippet` capped at ~300 chars; api `Citation` schema documents the pre-ADR-0020 shape | intelligence schemas; `api/app/schemas/__init__.py` |

## Decisions (made 2026-11-01)

| # | Decision | Outcome |
|---|----------|---------|
| D1 | Add `pdfjs-dist` npm dependency to ui | **Approved** (explicit approval per ui repo rules). ADR in ui. |
| D2 | PDF byte delivery | **API signs, GCS serves.** API endpoint returns the signed URL as JSON (gatekeeper: session + bucket/path validation + TTL unchanged); browser fetches bytes directly from GCS under a narrow bucket CORS rule. Rejected: streaming through the API (2 sync gunicorn workers on e2-small + 120 s timeout + Cloud Armor 30 req/IP/min would be saturated by PDF transfers; range-forwarding code to own). ADR in api; infra change links to it. |
| D3 | Chunk location schema + re-index | **Approved.** Knowledge ADR; executed when knowledge work resumes. |

## Security design

The threat model additions are: (a) PDFs are **untrusted external files** now
parsed by our own JS instead of the browser's sandboxed viewer; (b) a signed
URL becomes readable by page JS; (c) a private bucket gains a CORS rule.

| Control | Detail |
|---|---|
| pdf.js hardening | Pin current `pdfjs-dist`; `isEvalSupported: false` (CVE-2024-4367 class); worker bundled locally by Vite — never a CDN; canvas rendering only; no XFA; keep the pin fresh (Dependabot). |
| No HTML injection path | Highlights are absolutely-positioned empty `<div>`s computed from **numeric** page/bbox values (`Number()`-coerced). Chunk text is never rendered with `v-html`. Text-search fallback only reads pdf.js `textContent` for string matching — output is positioning, not markup. |
| Single-fetch bytes | UI fetches the PDF **once** as an ArrayBuffer and passes `data` to pdf.js. No range requests → no mid-read URL expiry, no request flood, one GCS GET per document view. |
| Signed URL in JSON | No privilege regression vs today's 302 (same-origin JS can already read the redirect target). JSON response sent with `Cache-Control: no-store`. TTL stays 10 min. Signed URL never logged (existing `gcs_knowledge.py` logs bucket/blob only — keep). |
| Endpoint validation reused | `get_knowledge_signed_url()` unchanged: server-side `GCS_KNOWLEDGE_BUCKET` only, path-traversal regex, `@session_required` (anonymous sessions allowed — same policy as `/chat`, covers public PublishedNotebook readers). |
| Bucket CORS is access-neutral | Bucket stays private; CORS only lets JS read responses it was already authorised (by signature) to fetch. Rule: `GET, HEAD` only; exact per-env origins (prod: `https://biglake.au`, `https://app.biglake.au`; test adds `http://localhost:5173`); `response_header = [Content-Type, Content-Length, Content-Range, Accept-Ranges, Cache-Control]`; `max_age_seconds = 3600`. Pattern: existing `notebooks_public` CORS block in `infra/modules/storage/main.tf`. |
| Cloud Armor | One `/knowledge/sign` call per document open — fits the 30 req/IP/min global rule, no carve-out. PDF bytes go straight to GCS and never traverse the LB (also avoids the known 429-masquerading-as-CORS failure mode). |
| No SSRF surface | Bucket fixed server-side; client supplies only the object path, validated as today. |

Residual risks to record in `api/documentation/security/risks.md` if accepted:
signed URLs are bearer tokens for 10 minutes (unchanged from today); pdf.js
parser bugs are now in our page context rather than the browser's PDF sandbox
(mitigated above, not eliminated).

---

## Phase 1 — Right document, right page (no re-index) — ✅ shipped 2026-11-01

User-visible: View source opens the correct PDF at the (section-start) page.

> **As built (differs from the steps below):** `snippet` cap is 1500 chars, not
> uncapped (saved-notebook size). Citation ADR is **api ADR-0027** — api ADR-0020
> is NL→SQL, not citations. `/knowledge/sign` returns `{url, expires_in}`.
> Follow-up fix: `GET /knowledge/sources?ids=` now also accepts collection
> **names** (`k-…`) — citations carry the name, the sidebar carries the OM UUID;
> before the fix the citation tab loaded no documents (api PR #4).

### intelligence
1. `Citation` gains `document_key: str`, `page: int | None`, `kind: str | None`;
   `snippet` becomes the **full chunk text** (rename decision: keep field name
   `snippet`, drop the 300-char cap — the UI menu already clamps visually).
   `chunk_to_citation()` populates them. SSE shape is additive → no version bump,
   but update the api ADR-0020 contract (amendment ADR).
2. No retrieval changes.

### api
3. Fix the stale `Citation` Marshmallow schema (`app/schemas/__init__.py` ~L780)
   to the real ADR-0020 shape + new fields. Contract-first loop: schema → route
   docstrings → `python scripts/update_openapi_baseline.py` → Newman.
4. Add `GET /knowledge/sign?source_url=<gs_url>` → `200 {"url": "...", "expires_at": "..."}`,
   `Cache-Control: no-store`. Same validation/service as `/knowledge/open`
   (which stays, as the non-JS fallback). Newman tests: happy path, 400 wrong
   bucket, 400 traversal, 401 no session.
5. ADRs: amend ADR-0020 (citation fields); new ADR for D2 (signed-URL JSON vs
   API proxy vs public bucket — with Assumptions: VM size/worker model,
   Cloud Armor global limit; Revisit when: API moves off sync workers, or
   byte-level audit/watermarking is required).

### ui
6. `Workspace.onChunk` copies the new fields (`document_key`, `page`, `kind`);
   saved notebooks without them must degrade to today's behaviour (guard for
   `undefined`).
7. `CellKnowledge.viewSource()` emits `{collection_id, document_key, page, title}`.
8. `Workspace.handleOpenKnowledgeSource` threads a `focus` object into the tab
   data; `KnowledgeSourceTab` accepts a `focus` prop: select the document whose
   `id === focus.document_key` (fall back to `documents[0]`), append `#page=N`
   to the iframe URL, scroll the preview section into view. Make focus survive
   the detail-fetch watcher and re-clicks while the tab is already open
   (changing `focus` must re-trigger even if same doc).
9. Fix G3: ChatTab and PublishedNotebook handle `@open-knowledge-source`
   (ChatTab forwards to Workspace's tab opener; PublishedNotebook opens the
   fallback `/knowledge/open` URL in a new tab if no tab system exists there).
10. Docs: update `api/BFF_API_REQUIREMENTS.md` §6.8 and the chunk-map field list.

**Exit criteria:** multi-document collection citation opens the right PDF at
`#page=N` in Workspace, ChatTab, PublishedNotebook; Newman + OpenAPI baseline
green; old saved notebooks unchanged.

## Phase 2 — Accurate locations (knowledge; requires re-index)

Scheduled for when knowledge work resumes. User-visible only via Phase 3.

### knowledge
1. `section_segmenter.py`: keep per-paragraph provenance — `content` becomes
   `[{text, page, bbox}]` (schema version bump on `parsed/`); Docling parser
   mirrors the same shape.
2. `chunker.py`: each chunk records `page_start`, `page_end`, and
   `regions: [{page, bbox}]` (per contributing paragraph/sentence window;
   tables emit their single region). Bump `CHUNK_RECORD_SCHEMA_VERSION`.
3. `pgvector_writer.py`: additive migration — `page_end INT`,
   `regions JSONB` on `knowledge_chunks` (+ propositions where derivable).
4. Re-parse → re-chunk → re-index. **Cost check first:** confirm whether the
   chunk-version bump forces re-enrichment (LLM spend) or whether
   `context_prefix` can be carried over keyed on chunk text hash.
5. Knowledge ADR: chunk geometry schema (Assumptions: pdf.js-style top-left
   point coordinates; Revisit when: parser change or non-PDF sources).

### intelligence
6. `expand_to_parents()`: keep the matched child on the expanded result —
   `RetrievedChunk` gains `matched_child_id`, `matched_child_regions`;
   `Citation` gains `regions` (parent) + `match_regions` (child). Page from the
   child, not the parent.

**Exit criteria:** citations carry exact page + region lists for text and table
chunks; propositions/summaries degrade to page/document gracefully.

## Phase 3 — In-app viewer with highlight — ✅ shipped 2026-11-01

> **As built:** infra knowledge-bucket CORS (infra #71); ui `PdfViewer.vue`
> (canvas-only, `pdfjs-dist` 6.4.299, ui ADR-0001) with iframe fallback.
> Highlight = snippet word-anchor match on the cited page + 5 pages ahead;
> table chunks page-only. Gotcha: the UI web server serves `.mjs` as
> `application/octet-stream`, so the worker must be imported with Vite's
> `?worker` (bundled as `.js`), not `?url`.
>
> **User feedback (2026-11-01):** highlights too broad (half pages / whole
> tables); wants single table-row highlight; the canvas viewer is much worse
> than the native one → wants a toggle back to it. Addressed in **Phase 3b**.

### infra
1. CORS block on the knowledge bucket (per Security design above), test + prod.
   Deployed via GHA Terraform as usual; links to the api D2 ADR in a comment.

### ui
2. New `PdfViewer.vue` (pdf.js, hardened per Security design): renders pages to
   canvas, lazy page rendering, scroll-to-page, highlight overlay layer.
   Highlight resolution ladder:
   1. `match_regions` (phase 2 data) → strong highlight; `regions` → light.
   2. Else fuzzy text match of the citation snippet via pdf.js `textContent`
      (normalise whitespace/hyphenation; skip for `kind=table_row` and
      propositions/summaries).
   3. Else scroll to `page` only.
3. `KnowledgeSourceTab` swaps the iframe for `PdfViewer` (keep iframe fallback
   behind a prop/flag for one release in case of regressions).
4. `client.js`: `signKnowledgeUrl(sourceUrl)` → calls `/knowledge/sign`; viewer
   fetches the ArrayBuffer itself.
5. ui ADR: viewer choice (pdf.js vs native iframe vs server-side page images)
   with the hardening settings as Assumptions.
6. ui README/DESIGN note + bundle-size note (pdf.js is ~400 KB gzipped; lazy-load
   the component with `defineAsyncComponent` so Workspace initial load is unaffected).

**Exit criteria:** clicking View source lands on the highlighted passage for
phase-2-indexed chunks; pre-phase-2 chunks still get document + page; donation-
locked (`metadata_only`) sources keep showing the banner, no PDF fetch attempted.

## Phase 3b — Better viewer, precise highlights, table rows (no re-index) — ✅ shipped 2026-11-01

> **As built:** ui ADR-0002 (viewer components, toggle, `.pdfv`-scoped viewer
> CSS via a PostCSS plugin in `vite.config.js`) and ui ADR-0003 (highlight
> selection). Matching lives in `ui/src/pdfMatch.js`; intelligence emits
> `match_texts`; api passes it through. Differences from the design below: the
> viewer toggle has no page sync and no persistence (user steer); PDFViewer
> 6.x has no `convertToViewportRectangle`, so overlays project with
> `convertToViewportPoint`.

Decided 2026-11-01 from user feedback. Absorbs the old optional Phase 4
(sentence-level evidence). Four steps, shipped in this order; each is
independently useful.

### Why highlights are broad today (root cause)

Retrieval matches a ~150-token **child**, then `expand_to_parents()` swaps in the
~768-token **parent** for synthesis and drops the child. The citation is the
parent (snippet capped at 1500 chars), the LLM cites a whole chunk with `[cN]`,
and the viewer highlights the whole snippet span → roughly half a page. Regions
(Phase 2) would make the boxes *exact* but not *narrower*; narrowing needs a
smaller unit of evidence (steps 3b.3–3b.4).

### 3b.1 — Viewer toggle + quick wins (ui)

- `KnowledgeSourceTab` preview header gets a two-state toggle: **Highlighted**
  (pdf.js) / **Original** (native iframe via `/knowledge/open#page=N`, the
  Phase 1 path — code already exists).
- **Default: Original.** Switches to Highlighted only when the user toggles, or
  when the tab is opened/refocused from a citation (so "View source" still
  lands on the highlight). No persistence — no `localStorage`, no remembering
  (user steer 2026-11-01).
- **No page sync between viewers** (user steer 2026-11-01 — dropped; the native
  frame can't be read anyway). Each viewer opens at the focus page when a
  citation drove the open, else page 1.
- Only the **active** viewer is mounted (`v-if`) — pdf.js code and PDF bytes
  are not fetched while Original is showing, so the default sidebar-browse path
  costs the same as before this feature.
- If pdf.js fails (CORS/sign/parse), auto-switch to Original and disable the
  toggle with a tooltip (today's fallback, made visible).
- Quick wins: preview fills the tab (`min(80vh, available)`) instead of a fixed
  700px; **Open in new tab** button (`/knowledge/open#page=N`).

### 3b.2 — Replace the canvas renderer with pdf.js viewer components (ui)

Use the components Firefox's viewer is built from — already in the installed
package (`pdfjs-dist/web/pdf_viewer.mjs` + `pdf_viewer.css`), so **no new
dependency and no vendored files**. Rejected: vendoring the full prebuilt
`viewer.html` app (~2 MB of copied release files, larger page-context surface).

- Wiring: `EventBus` + `PDFLinkService` + `PDFFindController` + `PDFViewer`
  in an absolutely-positioned container (pdf.js requirement). Gains: selectable
  text layer, find, internal links, proper zoom/scroll, lazy page rendering.
- Small own toolbar (same visual language as the tab): page `N / total` input +
  prev/next, zoom −/+/fit-width (`currentScaleValue = 'page-width'`), find box
  (dispatches the `find` event, highlight-all), Open in new tab, viewer toggle.
- `pagechanging` event → `currentPage` (feeds the toolbar page indicator and
  Open-in-new-tab deep link).
- **Highlight overlay:** rects computed once in **PDF user space** from
  `getTextContent()` item transforms; projected per render with
  `pageView.viewport.convertToViewportRectangle()` on `pagerendered`, so they
  stay aligned under zoom. Overlay divs live inside the page's `.page` element
  above the text layer, `pointer-events: none` (text stays selectable).
  **Styling is fully ours** — the overlays are app-owned divs (design tokens,
  dark mode, strong vs light tiers, rounded corners/fade-in as desired); pdf.js
  draws none of it. The find feature's match colour is pdf.js CSS and is
  overridden to match.
- **Performance:** pages are virtualised by `PDFViewer` (only visible pages
  render); text extraction for matching runs on the worker thread and only for
  the target page + 5-page lookahead; sentence/row scoring is plain string and
  number comparison (single-digit ms per citation click, no ongoing cost).
- Scroll to target: `scrollPageIntoView({pageNumber, destArray: [null,
  {name: 'XYZ'}, x, y, null]})` so the highlight lands near the top.
- `pdf_viewer.css` is global — imported only inside the lazy component, scoped
  under our container class where it collides with app styles.
- **Hardening (additions to the Security design):** `enableScripting: false`,
  `annotationMode: ENABLE` (links only, no form fields), `externalLinkTarget:
  BLANK`, `externalLinkRel: 'noopener noreferrer nofollow'`; pdf.js's URL
  validation limits link schemes to http(s)/mailto/tel. Text layer writes DOM
  text nodes (no HTML parsing). Existing `isEvalSupported: false`, local worker,
  single-fetch bytes all unchanged.
- ADR: new **ui ADR-0002** (viewer components + toggle) amends ui ADR-0001
  (ADR-0001's body is not rewritten; its Status becomes "amended by ADR-0002").

### 3b.3 — Precise highlights: matched child (A) + claim sentence (B)

**A. intelligence — keep what retrieval actually matched**
- `expand_to_parents()` collects, per parent, the matched children in rank
  order (cap 3) instead of discarding them. `RetrievedChunk` gains
  `matched_texts: list[str]`. Chunks with no parent (propositions, standalone)
  use their own text.
- `Citation` + SSE `chunk` event gain `match_texts: list[str] | None` (each
  ≤ 800 chars). `snippet` stays the parent (1500 cap) — used only as the last
  fallback now.
- Add a focused test for the child collection if intelligence's test layout
  allows; otherwise verify via the SSE payload on test.

**B. ui — align the claim to one sentence (deterministic, no LLM)**
- On chip click, `CellKnowledge` derives the **claim**: the answer text from the
  previous sentence boundary up to the chip (chip after a full stop → the
  preceding sentence; grouped chips `[c1,c2]` share one claim). Sentence split
  must not break on decimals (`$940.5 billion`). Capped at 500 chars.
- Candidate sentences: split `match_texts` (then `snippet` if none match).
- Score = number overlap (normalised: strip `$`, commas, `%`; `(1.2)` → `-1.2`;
  `billion`/`bn`/`b` unit words kept as tokens), weighted heavily, + content-word
  containment (stopwords removed). Best sentence wins if it clears a threshold.

**Highlight tiers (replaces today's single broad highlight)**

| Tier | Shows | Fallback |
|---|---|---|
| Strong | Best claim sentence | Matched child text if no sentence clears threshold |
| Light | Matched child text(s) | Parent snippet only when there are no `match_texts` (old payloads) |
| — | Page only | No text match on target page + 5 ahead |

The parent is no longer shaded when better evidence exists — that's what
removes the half-page highlight. Precision floor: pdf.js text runs are usually
a whole line, so "one sentence" renders as the 1–3 lines it spans.

### 3b.4 — Single table-row highlight (ui; uses 3b.3-A data)

With A, a table citation carries its matched **row-group** markdown
(`kind = table_row` children, a few rows each). Picking one row:

1. Parse the row-group markdown into rows of cells (skip header + separator).
2. Target row = the row whose cells best overlap the claim's numbers/words
   (B's scorer); else every row in the group is light-highlighted.
3. On the cited page (+ lookahead), tokenise pdf.js text items into normalised
   cell tokens; split multi-number runs ("940.5 951.2") by character-proportion
   x-offsets.
4. Cluster matched tokens by baseline y (tolerance ≈ ½ line height). Accept the
   cluster with the most distinct cells matched, **requiring ≥ 2 cells** (row
   label or a second number) so repeated numbers elsewhere can't false-match.
5. Strong highlight = a band spanning the accepted line from the leftmost to
   the rightmost item on that line; other rows of the group light.

Fallbacks: tables split across pages, rotated or scanned tables → row-group
light highlight, else page only. Tables the parser didn't detect arrive as text
chunks and are handled by 3b.3. Phase 2 can later store exact row boxes from
the table extractor and replace steps 3–5.

### Contract changes (3b, all additive)

| Layer | Change |
|---|---|
| intelligence | `RetrievedChunk.matched_texts`, `Citation.match_texts`, SSE `chunk.match_texts` |
| api | `Citation` schema + OpenAPI baseline; BFF §6.8 note. No route changes. |
| ui | All three `onChunk` handlers copy `match_texts` (BUG-0001 trap); focus becomes `{document_key, page, snippet, kind, match_texts, claim, nonce}`; old saved notebooks lack both → today's behaviour |

Security: `claim` (LLM output) and `match_texts` (document text) are untrusted;
both are used **only** for string matching, length-capped, never rendered as
HTML.

### Exit criteria / manual test matrix (test env)

- "how much debt is australia in?" → strong highlight on the debt sentence
  (1–3 lines), not half a page.
- A budget-table citation → one row band.
- Toggle defaults to Original; switches to Highlighted on toggle or when
  arriving via a citation; no persistence across opens.
- Zoom in/out keeps highlights aligned; find works; text is selectable;
  external links open in a new tab.
- Old saved notebook citation: still opens the right page (old behaviour).
- pdf.js failure (block CORS in devtools) → Original with a disabled toggle.

## Rollout order & dependencies

```
Phase 1 (intelligence → api → ui)        — ✅ shipped
Phase 3 (infra → ui)                     — ✅ shipped
Phase 3b.1 toggle + quick wins (ui)      — next
Phase 3b.2 viewer components (ui)        — then
Phase 3b.3 A (intelligence → api) → B + tiers (ui)
Phase 3b.4 table rows (ui)               — after 3b.3 A is live
Phase 2 (knowledge + intelligence)       — when knowledge resumes; re-index;
                                            exact boxes, true pages, row boxes
```

Phase 2 is now a robustness upgrade (exact pages for long sections,
multi-column layouts, text-match misses, exact table rows), not a prerequisite
for useful highlighting.

## Documentation checklist (org standard)

- [x] api: ADR-0027 (citation fields + D2); OpenAPI baseline;
      BFF_API_REQUIREMENTS §6.8; Newman collection
- [x] ui: viewer ADR-0001; README
- [x] ui: ADR-0002 (viewer components + toggle, amends 0001), ADR-0003
      (highlight selection); README
- [x] intelligence: DESIGN.md citation section (`match_texts`, 3b.3-A)
- [x] api: `Citation.match_texts` in schema + OpenAPI baseline; BFF §6.8 (3b.3-A)
- [ ] knowledge: chunk-geometry ADR; pipeline README (parsed/chunk schema bumps)
- [x] infra: comment on CORS block linking api ADR-0027
- [ ] api `documentation/security/risks.md`: accepted-risk rows (signed-URL
      bearer exposure window; pdf.js parse surface)
- [ ] org `.github/TODO.md`: propose cross-repo row (confirm with user first)
