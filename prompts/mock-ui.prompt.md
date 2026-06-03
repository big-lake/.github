---
description: "Add a mocked UI feature — hardcoded data, no real API call. Reads BFF_API_REQUIREMENTS.md to locate the feature, proposes an API shape, and updates the tracker."
mode: agent
---

# Mock a UI feature

Use this when you want to build and validate UI for a feature **before the API endpoint exists**. The mock lets you iterate on layout and interaction cheaply; the proposed API shape becomes the spec for the real endpoint later.

Feature to mock: ${input:featureDescription:Short description (e.g. "Suggested prompt chips above the prompt bar")}

---

## Step 1 — Read BFF_API_REQUIREMENTS.md in full

Open `api/BFF_API_REQUIREMENTS.md` and read the entire file before writing a single line of code.

You need to establish:

- **Is this feature already tracked?** Find the matching section (or confirm it isn't there).
- **What is its current UI Status?** (`Not built`, `Mocked`, `Partial`, `Wired`, `Blocked`)
- **Does a proposed API shape already exist?** If so, honour it — use the same field names in the mock so the real cutover is a search-and-replace, not a rename.
- **What Open Work rows reference this feature?** Note them so you can update them after the mock lands.
- **What neighbouring features are already mocked or wired?** Understand the surrounding context so the new mock fits the existing pattern.

If the feature is already **Wired** (real API call in place), stop and tell the user: this prompt is for mocking unbuilt features. Use `/api+ui-change` to change existing wired behaviour.

---

## Step 2 — Propose a plan before writing code

Based on the BFF doc and your understanding of the UI codebase, write a short plan covering:

1. **Where does the mock live?** Which `.vue` file(s) will change? Is it a new component extracted from an existing view, or inline?
2. **What hardcoded data will the mock use?** Show the shape. Use realistic field names that match (or will become) the real API response. If a proposed shape is already in BFF_API_REQUIREMENTS.md, use it exactly.
3. **What interaction does the mock need to support?** Click handlers, loading states, empty states.
4. **What section of BFF_API_REQUIREMENTS.md will be added or updated?**

Ask the user to confirm the plan before proceeding.

---

## Step 3 — Implement the mock

Follow the UI conventions exactly:

- **Vue 3 Composition API, `<script setup>`** — no Options API, no TypeScript.
- **Hardcoded data inline** — use a `const MOCK_<NAME> = [...]` constant at the top of the component (not in `client.js`, which is for real API calls only). Name it clearly so it's easy to find and replace later.
- **No real API call** — do not add a `fetch` or import anything from `client.js` for this feature.
- **No new npm dependencies** — plain CSS, scoped styles only.
- **Keep `.vue` files under ~300 lines** — extract a focused child component into `components/` only when there's a clear boundary. Do not create helpers or abstractions for one-time use.
- **Use existing design tokens** (`--highlight`, `--text-tertiary`, `--border`, etc. from `styles.css`) — do not hardcode colours.
- **Loading / empty states** — add at least a minimal `v-if` for the empty case, even if the mock never hits it. This makes the real cutover safer.

---

## Step 4 — Review the code

Do **not** use Playwright or any browser automation to verify. The user will test the UI themselves.

Instead, review the code you've written and confirm:

- The mock renders without obvious errors (no missing imports, no undefined references).
- Layout logic matches the described feature.
- Interaction (clicks, toggles, expand/collapse) is wired up correctly in the template.
- The mock data shape uses the field names that match or will become the real API response.

---

## Step 5 — Update BFF_API_REQUIREMENTS.md

Open `api/BFF_API_REQUIREMENTS.md` and update it to reflect what was built.

### If the feature is already tracked (status was `Not built`):

- Change `**UI Status:**` to `**Mocked**`.
- If there was no API endpoint block, add one now with the **proposed** request/response shape (use the mock's field names).
- Add or expand the `**Notes:**` block:
  - Name the constant(s) that hold the hardcoded data (e.g. `PROMPT_CHIPS` in `Dashboard.vue`).
  - State what needs to happen to wire it: "Replace `MOCK_<NAME>` with `await client.getFoo()` once `GET /foo` is implemented."
- Update the relevant row(s) in the **Section 8 — Open Work** table: status stays `Not started` for the API work, but add a note like `UI mocked as of <date>`  in the Notes column.

### If the feature is not yet tracked (new feature):

Add a new subsection in the most appropriate section (1–6). Use the standard template:

```
### N.X <Feature name>

**UI Feature:** <one-sentence description of the UI element>
**UI Status:** Mocked
**Purpose:** <what problem this solves for the user>
**API Endpoint:** `<METHOD> /proposed/path` — not yet built.

**Proposed request:**
\`\`\`json
{ ... }
\`\`\`

**Proposed response:**
\`\`\`json
{ ... }
\`\`\`

**Wiring status:** Not started — `MOCK_<NAME>` is hardcoded in `<File>.vue`. Replace with `<clientFn>()` once the endpoint exists.

**Notes:** <anything worth knowing — mock constant name, edge cases, dependency on another feature>
```

Also add a row to the **Section 8 — Open Work** table:

```
| <next letter> | <Area> | <brief item description> | Not started | UI mocked in `<File>.vue` (`MOCK_<NAME>`) |
```

---

## Checklist before commit

- [ ] `BFF_API_REQUIREMENTS.md` read in full before starting
- [ ] Mock data uses realistic field names that match (or will become) the real API shape
- [ ] Hardcoded data is a named `MOCK_<NAME>` constant — easy to grep and replace
- [ ] No `fetch`, no `client.js` import for the new feature
- [ ] No new npm dependencies
- [ ] No hardcoded colours — design tokens only
- [ ] `BFF_API_REQUIREMENTS.md` updated: status set to `Mocked`, proposed API shape documented, Open Work table updated
