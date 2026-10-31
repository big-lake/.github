---
description: "Describe a UI look-and-feel change in plain words; the agent maps it to the shared tokens/classes that control it, explains what else it affects, and offers options before editing."
mode: agent
---

# Restyle the UI (plain-language)

I'm not a UI designer. I'll describe what I see or want in everyday words; you work out which shared styles control it and explain that back to me before changing anything.

What I want to change: ${input:change:Describe it in your own words (e.g. "the buttons feel too sharp", "make the top bar feel warmer", "the notebook cells look cramped", "I don't like the green")}

---

## Ground rules

- Read `ui/README.md` → "Styling & branding", `ui/src/styles/tokens.css`, `components.css`, `base.css` and `ui/brand/colours.md`.
- Plain CSS only, no new dependencies. Raw hex/rgba and `rem` font sizes live only in `tokens.css` (`npm run check:styles` enforces; `AgentTerminal.vue` is dormant).
- Work top-down and stop at the first layer that achieves the change: **style-guide knobs** (the broad rules in `tokens.css`: `--tint-*` hover/selected strength, `--disabled-opacity`, `--control-pad-*`/`--card-pad`, `--radius-*`, `--text-*`, `--duration-*`, `--shadow-*`) → **colour tokens** → **shared classes** → **per-component layout**. Prefer a knob over editing individual components: e.g. "hover feels too faint" is `--tint-subtle`, "too cramped" is `--control-pad-*`/`--card-pad`. If a component re-implements a button, input, card or chip locally, switch it to the shared class instead of editing the local copy.
- Styling only: no behaviour, markup-structure or API changes.
- Explain everything in plain language. Avoid design jargon, or define it in one short phrase the first time (e.g. "radius = how rounded corners are").

## Style-guide knobs (check these first)

These live in `ui/src/styles/tokens.css`. Read the file for current values and look for newer knobs it may have added. Map my wording to a knob where you can.

| If I say... | Knob | Effect |
|---|---|---|
| "hover is too faint / too strong", "selected state is hard to see", "outlines too subtle" | `--tint-subtle` (hover wash), `--tint-medium` (selected fill), `--tint-strong` (edges), `--tint-bold` (emphasis) | Strength of every translucent fill and border |
| "disabled things look too faded / not faded enough" | `--disabled-opacity` | All disabled controls |
| "cramped", "too spacious", "buttons/inputs feel small" | `--control-pad-y/x`, `--control-pad-y/x-sm` | Padding in `.btn`, `.field` |
| "cards/panels feel tight" | `--card-pad` | Card/panel padding |
| "too sharp", "too rounded" | `--radius-sm/md/lg/xl/pill` | Corner roundness everywhere |
| "text too small/large", "hierarchy unclear" | `--text-2xs` … `--text-3xl` | Font sizes everywhere |
| "feels flat", "too heavy/floaty", "modal backdrop too dark" | `--shadow-*`, `--overlay-bg` | Elevation and backdrop |
| "animations feel sluggish / snappy" | `--duration-fast`, `--duration-base`, `--ease` | Hover/focus transition speed |
| "lines too thin/thick" | `--border-width` | Borders on shared classes |
| "change the green / accent / feel" | `--brand-*`, `--accent`, `--highlight`, semantic colours | Colour (check light and dark) |

When proposing options, name the knob(s), the current value, and the suggested value. Say honestly when a knob only reaches the shared classes (e.g. `--control-pad-*` does not yet affect components with their own padding).

### Proposing a new knob

If what I describe isn't controlled by any existing knob but is repeated across components (e.g. "all the section headings", "the left sidebar width", "list row height"), don't hardcode it per component. Instead propose a new knob: its name (`--kebab-case`, grouped with the related ones in `tokens.css`), a default equal to today's most common value, which shared class or components would consume it, and how many places change. Add it only after I agree. Then migrate the consumers, extend `ui/scripts/check-styles.mjs` if a literal could creep back in, and add a row to the README's "Style guide knobs" table and to this table.

## Step 1 — Translate my description into what controls it

Do this before proposing edits, and report it to me concisely:

1. **Restate** what you think I mean in one sentence. If it's ambiguous (e.g. "the green" could be the brand colour, the top bar, or the chart colours), ask me one short question, offering the options.
2. **Find where it appears.** Search `ui/src` for the relevant classes/tokens and open the running app (`http://localhost:5173`, dev server is normally up) to look at the real thing. Take a screenshot if useful.
3. **Show a "what controls this" table** with these columns:
   | Thing I'll see change | Controlled by (token / shared class / component) | Where it's defined | Everything else it affects |
   Use the real token and class names. "Everything else it affects" is the important column: count roughly how many components/views use a token or class (`grep` for `var(--token)` / class usage) and name the notable ones in words ("every button", "all modals", "notebook cells", "charts").
4. **Say which layer is right** and why: a style-guide knob or colour token (changes the look everywhere), a shared class (changes every use of that component), or one component only. Flag the trade-off if I only want a local change but the cleanest fix is global, or vice versa. Prefer an existing knob; if none fits, propose a new one (see above).

## Step 2 — Offer options

Give 2–3 concrete options in plain words, each with the exact change (token values, class edits) and its blast radius. Include a "smallest change" and a "most consistent" option. Recommend one. For colour changes, state the effect in both light and dark themes and check text contrast stays readable.

Then **stop and ask me to pick** — don't edit until I do. Unless I say "just apply it", offer to preview the options first (Step 3).

## Step 3 — Try before committing (live preview, no file edits)

Use this whenever I want to "see" or compare options. Nothing is written to the repo during preview.

1. Open the running dev app (`http://localhost:5173`; start the `ui: dev (Vite :5173)` task if it isn't up) in the browser tool.
2. For each option, inject a temporary `<style id="style-trial">` into the page that overrides only the relevant custom properties, e.g. `:root { --radius-md: 12px; --accent: #2a6f7f; }` and a `:root[data-theme="dark"] { ... }` block if the dark theme needs different values. Replace the same `<style id="style-trial">` each time so options don't stack. A page reload resets everything.
3. Screenshot each option on the same screen(s) in light and dark, and show them side by side with a one-line description of each ("Option A: rounder, softer"). Include a screen that shows the thing I described plus one other place it will shift (e.g. a modal, the sidebar).
4. Let me tweak: if I say "a bit less round" or "warmer", adjust the injected values and re-screenshot. Keep a running list of the final values for each option.
5. Only when I pick one, go to Step 4 and make it permanent. If I just wanted to look around, finish by noting which option (if any) I preferred and the token values, without editing files.

If the change needs a shared class or component edit that tokens can't express (e.g. a different button shape), preview it by injecting the CSS rule the same way.

## Step 4 — Apply

- Edit in order: tokens → shared classes → components.
- New brand colours: update the `--brand-*` tokens **and** the table in `ui/brand/colours.md` together.
- If a needed value has no token, add one to `tokens.css` rather than hardcoding it.

## Step 5 — Verify

- `cd ui; npm run check:styles; npm run build`
- In the browser, check light and dark theme on: Workspace (notebook cells, sidebar, chat tab), a modal (e.g. Dataset info), Login, and one page-shell page (Mission or Security). Screenshot before/after.

## Step 6 — Report back in plain language

- What I'll now see differently, and where.
- Which tokens/classes were changed, and what else shifted as a side effect.
- Anything that still looks old because a component has its own local styling (list it; offer to migrate it onto the shared class).
- Update `ui/README.md` "Styling & branding" if tokens or shared classes were added, renamed or removed.
- Don't commit or push without my confirmation.
