---
description: "Continue the UI styling overhaul on the current ui branch: study a reference site's styling, then roll out ui/brand/brand.md through the central tokens, major elements first down to hover states."
mode: agent
---

# UI overhaul (reference-site-driven restyle)

Continue the styling overhaul on the current `ui` branch. The reference site's look is the inspiration; `ui/brand/brand.md` is the brand spec we're rolling out; `ui/src/styles/` is the only place styling decisions live.

Reference site URL: ${input:referenceUrl:Public site to take styling cues from}

What to work on this session: ${input:focus:e.g. "top bar and navigation", "buttons and forms", "cards and panels", "hover and selected states", "dark mode", or "next steps"}

Colour instructions (optional): ${input:colours:Classes + colours to try, e.g. class="cell-focus" - lime green; or blank to let you choose from brand.md}

---

## Ground rules

- **Never write the reference site's name or URL into anything committed**: code, comments, CSS, docs, commit messages, branch names, PR text. Refer to it only as "the reference site". Check with `git grep -i` for its name before every commit.
- **Branch**: work on the current `ui` branch (expected `styling-overhaul`; check `git branch --show-current`). Never commit to `main`. Commit freely; **ask before push**.
- **Centralised and reusable**: colours, fonts, weights, radii, shadows, gaps and states are defined once in `ui/src/styles/tokens.css` (layer 1 `--brand-*`, layer 2 semantic, scales/knobs). Components consume tokens or shared classes in `components.css`/`base.css`; no hex/rgba literals or per-component duplicates. If a component re-implements a button, input, card or chip, move it onto the shared class.
- **Plain CSS, no new dependencies**, no framework. Follow the `ui` repo's `.github/copilot-instructions.md`; keep `.vue` files lean.
- **Styling only**: no behaviour, markup-structure or API changes.
- **Subtle, not loud.** The reference site is bold and loud; ours should be calmer. Borrow structure, rhythm and palette relationships, not its intensity.
- **No italics anywhere.** Don't add `font-style: italic`. (The editor's Italic formatting button is functional and stays.)
- **Balance like a graphic designer**: roughly 60% neutrals, 30% dark structure, 10% accent. One primary accent for interaction, a second accent used sparingly for small highlights. Check text contrast in light and dark.
- **Dark mode is kept**; always set and check both themes.
- **`ui/brand/colours.md` is legacy reference; don't edit it.** `ui/brand/brand.md` is the current spec.
- `AgentTerminal.vue` is dormant; ignore it.

## Known gotchas

- JS reads some tokens with `getComputedStyle(...).getPropertyValue(...)` and passes them to d3: `--text`, `--surface-on-dark`, `--text-tertiary` and `--chart-1..8`. These must resolve to plain colours (hex, or `var()` of a hex). Never use `color-mix()` for them.
- `:root[data-theme="dark"]` re-maps the semantic layer only; if you add a semantic token, define it for dark too (or confirm it inherits sensibly).
- `scoped` component styles override global rules, so removing a look usually means editing the component rule, not adding a global override. Don't use `!important`.
- The screenshot tool can fail; fall back to reading computed styles with Playwright (`getComputedStyle`) to verify colours. Parallel read results can be stale right after an edit; re-read the file if something looks wrong.
- `html`/`body` should never scroll; verify `document.documentElement.scrollHeight <= window.innerHeight` after layout-affecting changes.

## Step 1 — Orient (don't skip)

1. `git branch --show-current`, `git status -sb`, and the last few commits, so you know where the overhaul stands.
2. Read `ui/brand/brand.md`, `ui/src/styles/tokens.css`, `base.css`, `components.css`, and `ui/README.md` → "Styling & branding".
3. Summarise in 3-5 lines: what the current scheme is, what's already applied, and what the session's focus will change.

## Step 2 — Study the reference site (only what's needed for the focus)

Fetch the page HTML and linked stylesheets directly; the text-only page fetch doesn't return CSS. Example (PowerShell):

```powershell
$h = (Invoke-WebRequest -UseBasicParsing -Uri $url -UserAgent "Mozilla/5.0").Content
[regex]::Matches($h,'/_next/static/css/[^"]+\.css') | % Value | sort -Unique   # or <link rel="stylesheet"> hrefs
```

Download the stylesheets and extract, for the focus area: colours by frequency, `font-family`/weights, heading sizes, radii, shadows, button/nav/card rules, hover/focus rules and transitions. Also open the live page in the browser tool for the visual structure if useful.

Report what you found as a short list of **design observations** (e.g. "nav is flat with a thick accent underline on the active item", "buttons are squared with a scale-on-hover"), then say how each would translate into **our** tokens, calmer. Add genuinely new brand attributes (colours, fonts, styles) to `ui/brand/brand.md` with a name, hex and role; never invent a colour I haven't given you without saying so.

## Step 3 — Plan top-down

Work from the largest surfaces to the smallest, stopping at the first layer that achieves each change:

1. Page background, surfaces, text, borders
2. Top bar / navigation and tabs (active, hover)
3. Panels, cards, modals, toolbars
4. Controls: buttons, inputs, chips, checkboxes
5. Data views: tables, charts, focused/selected rows
6. Interaction states: hover washes (`--tint-*`, `--node-hover`), selected, focus ring, disabled
7. Dark mode pass over all of the above

For my colour instructions, map each class to a **semantic token** (not a literal on the class), find every other place using the same role, and change the token so they move together. If a class shares a token with something I didn't mention, tell me and offer to split the token.

## Step 4 — Preview, then apply

- For anything exploratory, inject a temporary `<style id="style-trial">` overriding custom properties in the running app (`http://localhost:5173`; start the `ui: dev (Vite :5173)` task if it isn't up) and screenshot light and dark. Nothing is written to files until I agree.
- When I've approved (or said "just apply it"), edit in order: tokens, shared classes, components. Keep `brand.md` in sync with `tokens.css` layer 1 and its "How the UI uses the palette" table.
- If a needed value has no token, add one in `tokens.css` and use it, rather than hardcoding.

## Step 5 — Verify

- `cd ui; npm run check:styles; npm run build`
- In the browser, in light and dark: Workspace (sidebar, tab bar, notebook cells, data table, chat tab), a modal, Login, and a page-shell page (Mission or Security). Screenshot, or fall back to computed-style checks.
- Check no page overflow and that charts still render with the new `--chart-*` values.
- `git grep -i` for the reference site's name in the diff (must find nothing).

## Step 6 — Report and commit

- Report briefly: what changed and where, which tokens moved, side effects, anything still looking old because of local component styles (offer to migrate it), and any contrast concerns.
- Update `ui/README.md` "Styling & branding" if tokens or shared classes were added, renamed or removed.
- Commit with a clear message on the current branch. **Ask me before pushing**, and confirm first if I'd rather keep the work uncommitted for comparison.
- If I ask to "revert back", restore only the style files (`brand/brand.md`, `src/styles/*`) to the last good commit with `git restore --source=<commit> -- <files>`; don't touch unrelated work.
