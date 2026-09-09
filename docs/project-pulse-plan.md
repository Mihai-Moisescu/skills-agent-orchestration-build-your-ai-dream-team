---
# Project Pulse Dashboard — Implementation Plan

## 1. Overview

**Project Pulse** is a lightweight, static, client-side web dashboard that Mona will use to track the health and status of her active projects at a glance. It renders a grid of **project cards** driven by a JSON data file, with each card summarizing a project's name, owner, current status, recent activity, and priority. Status and priority are communicated visually through **badges** and deliberate **priority treatment** (e.g., visual emphasis for high-priority items). The layout is **responsive**, adapting from a single-column stack on narrow viewports to a multi-column grid on wider screens.

Design goals:
- **Zero build tooling** — plain HTML/CSS/JS served statically from `app/`.
- **Data-driven** — cards are rendered from `app/project-data.json` at runtime; no hardcoded content in HTML.
- **Deterministic hooks** — stable CSS classes (`.dashboard`, `.project-card`, status/priority modifier classes) to keep design and code loosely coupled.
- **Accessible and responsive** — semantic HTML, ARIA where appropriate, sufficient color contrast, keyboard-friendly, works on mobile and desktop.
- **Locally previewable** — a `.vscode/launch.json` configuration named **"Run Project Pulse Dashboard"** serves the `app/` directory over a local HTTP server and opens `http://localhost:<port>/index.html` in a browser.

The work is executed by three specialist agents coordinated by the Orchestrator: **Planner** (this document), **Designer** (UI/UX, visual language, CSS), and **Coder** (markup structure, data schema, rendering JS, launch configuration). See `docs/agent-team.md`.

---

## 2. File Assignments

| File | Owner | Purpose |
| --- | --- | --- |
| `app/index.html` | **Coder** (structure/hooks) with **Designer** review for semantics & a11y | Root document; defines the semantic skeleton and the deterministic CSS hooks the Designer will style. |
| `app/styles.css` | **Designer** | All visual design: layout grid, card styling, status badges, priority treatment, responsive breakpoints, typography, color system. |
| `app/project-data.json` | **Coder** | The data source: top-level `"projects"` key with an array of project objects. Schema is fixed and agreed up front. |
| `.vscode/launch.json` | **Coder** | Strict-JSON (no comments) VS Code launch configuration named `Run Project Pulse Dashboard` that serves `app/` and opens `index.html` in a browser. |

### 2.1 `app/index.html` (Coder-owned)

Must contain:
- HTML5 doctype, `<html lang="en">`, `<meta charset>`, `<meta name="viewport" content="width=device-width, initial-scale=1">`.
- `<title>Project Pulse</title>` and a linked `<link rel="stylesheet" href="styles.css">`.
- A semantic header (`<header>`) with the dashboard title (e.g., "Project Pulse") and an optional short subtitle.
- A **main dashboard container** exposing the `.dashboard` class (deterministic hook required by validation), implemented as `<main class="dashboard" aria-label="Project dashboard">` or equivalent.
- An empty **container element** into which project cards will be injected by JS (e.g., `<section class="dashboard__grid" id="project-list" aria-live="polite">`).
- A **card template** — either an inline `<template id="project-card-template">` describing the `.project-card` markup, OR a documented DOM-construction contract in the render script. Prefer `<template>` for clarity and determinism.
- A `<footer>` (optional) with last-refreshed timestamp or attribution.
- A `<script src="app.js" defer>` (or inline `<script type="module">` inside `index.html`) that fetches `project-data.json` and renders cards. **Note:** No separate JS file is required by validation; the Coder may inline the render script inside `index.html` to keep the app to the four assigned files. This decision is the Coder's, but must be made before rendering logic begins. **Recommended: inline script inside `index.html`** to avoid introducing an untracked fifth file.
- The literal substrings `dashboard`, `.dashboard`, `project-card`, and `index.html` will end up covered across `index.html` and `styles.css` — the Coder must ensure `project-card` (as the card class) appears in the markup/template.

### 2.2 `app/styles.css` (Designer-owned)

Must contain (at minimum):
- A `.dashboard` selector defining the outer layout container (padding, max-width, responsive grid host).
- A `.project-card` selector defining the card visual: **must include `border-radius` and `box-shadow`** (deterministic validation requirement) plus padding, background, typography scale, spacing.
- Status badge styles (e.g., `.status-badge`, `.status-badge--on-track`, `.status-badge--at-risk`, `.status-badge--blocked`, `.status-badge--complete`) with distinct, WCAG-AA-contrast color pairs.
- Priority treatment (e.g., `.project-card--priority-high`, `--priority-medium`, `--priority-low`) using a mix of accent border, badge, or subtle background — not color alone (a11y).
- A responsive grid: single column on small viewports, two columns at a medium breakpoint (~640–768px), three+ columns at wide (~1024px+). Use CSS Grid with `repeat(auto-fill, minmax(...))` or explicit `@media` queries.
- A restrained, cohesive color palette + typography stack (system fonts are fine).
- `:focus-visible` styles for interactive elements; respect `prefers-reduced-motion`.

### 2.3 `app/project-data.json` (Coder-owned, schema agreed with Planner/Designer first)

Must contain:
- A top-level object with a `"projects"` key (validation requirement) whose value is an array.
- Each project object must include these fields (validation requirement): `name`, `owner`, `status`, `recentActivity`, `priority`.
- Suggested value domains (agreed up front so Designer can style them deterministically):
  - `status`: one of `"On Track"`, `"At Risk"`, `"Blocked"`, `"Complete"`.
  - `priority`: one of `"High"`, `"Medium"`, `"Low"`.
  - `recentActivity`: short human-readable string (e.g., `"PR #42 merged 2h ago"`).
- 4–8 seed projects covering every status and priority combination the Designer needs to demonstrate visual variety.
- Strict JSON, UTF-8, no comments, no trailing commas.

### 2.4 `.vscode/launch.json` (Coder-owned)

Must contain:
- Strict JSON, **no comments** (validation requirement — `python3 -m json.tool` must parse it).
- `"version": "0.2.0"`.
- A `configurations` array containing at least one configuration whose `"name"` is exactly **`Run Project Pulse Dashboard`** (validation requirement).
- The configuration must **serve from the `app` directory** and open **`http://localhost:<port>/index.html`** (validation requires the literal string pattern `http://localhost:%s/index.html` context and `index.html` as launch target).
- Recommended shape: a compound of a `preLaunchTask` (or `serverReadyAction`-style pattern) that runs a static server rooted in `app/`, plus a `chrome`/`msedge`/`node`-terminal type entry that opens the URL. A common minimal-dependency approach: a `"type": "node-terminal"` configuration whose `"command"` runs `python3 -m http.server` from `cwd: ${workspaceFolder}/app` with a `serverReadyAction` that opens `http://localhost:<port>/index.html`.
- Any port is acceptable, but it must appear in the URL that VS Code opens.

---

## 3. Designer Responsibilities

The Designer owns everything a user sees and how they perceive project state.

1. **Information architecture per card.** Decide the visual hierarchy inside `.project-card`: project `name` (primary, largest), `owner` (secondary), `status` (badge, high-visibility), `priority` (badge or accent treatment), `recentActivity` (tertiary, muted). Confirm this hierarchy with the Coder before he freezes the card template markup.
2. **CSS hooks (deterministic).** Style the required `.dashboard` and `.project-card` classes plus status/priority modifier classes. Do **not** rename these hooks — they are validated and consumed by the Coder's render logic.
3. **Status badge treatment.** Distinct visual badge per status value with WCAG AA contrast (≥ 4.5:1 for text). Include a non-color signal (icon, shape, or label) so state is not conveyed by color alone. Badges must be legible at small sizes.
4. **Priority treatment.** Apply a clear visual differentiation for High vs. Medium vs. Low — e.g., accent left border, priority pill, or subtle background tint. High priority must draw the eye first without being alarming.
5. **Responsive layout.** Define the grid behavior with at least these breakpoints: mobile (<640px, single column), tablet (640–1023px, two columns), desktop (≥1024px, three or more columns). Cards must not overflow, and text must wrap gracefully.
6. **Polished visual details.** Include `border-radius` and `box-shadow` on `.project-card` (required by validation), consistent spacing scale, restrained palette, readable typography.
7. **Accessibility.**
   - Ensure sufficient color contrast across all badges and text.
   - Provide `:focus-visible` styles for any interactive elements.
   - Support `prefers-reduced-motion`.
   - Recommend/coordinate ARIA on containers (in review of `index.html`).
8. **Empty / loading / error states.** Provide styles for a "no projects" empty state and a rendering-error state that the Coder can toggle via a class.

---

## 4. Coder Responsibilities

The Coder owns structure, data, behavior, and tooling.

1. **`app/index.html` skeleton and hooks.**
   - Semantic landmarks (`header`, `main.dashboard`, `footer`).
   - The `.dashboard` container and a card-list container element.
   - A `<template id="project-card-template">` describing card markup with the `.project-card` class and the child elements the Designer has agreed to (name, owner, status badge, priority indicator, recent activity).
   - Link to `styles.css`; embed the render script (see below).
2. **Data schema authoring (`app/project-data.json`).**
   - Emit strict JSON with the top-level `"projects"` array.
   - Every project object contains `name`, `owner`, `status`, `recentActivity`, `priority`.
   - Provide seed data that exercises every status and priority combination so the Designer can visually verify all badge variants.
3. **Rendering logic (inline `<script>` in `index.html`).**
   - `fetch('./project-data.json')` on `DOMContentLoaded`.
   - Parse and validate presence of `projects` array; if missing/empty, render the empty state.
   - For each project, clone the `<template>`, populate text nodes, and apply modifier classes derived deterministically from `status` and `priority` values (e.g., `status-badge--at-risk`, `project-card--priority-high`). Slug/normalize by lowercasing and replacing spaces with dashes.
   - Handle fetch/parse errors: render the error state, log to console with a clear message.
   - No frameworks, no bundlers, no external network requests.
4. **`.vscode/launch.json`.**
   - Strict JSON with no comments. Must parse with `python3 -m json.tool`.
   - A configuration named exactly `Run Project Pulse Dashboard`.
   - Serves from the `app/` directory (e.g., via `python3 -m http.server` with `cwd: ${workspaceFolder}/app`).
   - Uses a `serverReadyAction` (or equivalent) to open `http://localhost:<port>/index.html` in the default browser once the server is ready.
5. **No git actions.** Do not stage, commit, or push — git operations remain with the learner.

---

## 5. Dependencies (Ordering Graph)

```
[A] Data schema agreement                (Planner + Coder + Designer, this doc)
        │
        ├──────────────────────────────┐
        ▼                              ▼
[B] app/index.html structure       [C] app/project-data.json
    & CSS hooks (Coder)                (Coder)
        │                              │
        │  (hooks published)           │  (schema published)
        ▼                              ▼
[D] app/styles.css (Designer)      [E] Render script inside
                                       index.html (Coder)
        │                              │
        └──────────────┬───────────────┘
                       ▼
              [F] Visual + data integration check
                       │
                       ▼
              [G] .vscode/launch.json (Coder)
                       │
                       ▼
              [H] Validation (see §7)
```

Explanation:
- **A → B, C**: The status/priority value domains and card field list must be fixed in this plan before anyone writes markup or JSON, otherwise Designer's modifier classes and Coder's render logic will drift.
- **B → D**: `styles.css` targets the deterministic hooks declared in `index.html`. The Designer needs the final class names before finalizing rules.
- **C → E**: The render script consumes the JSON schema; the schema must be locked first.
- **D + E → F**: Visual correctness of populated cards can only be judged once both the styles and rendered DOM exist.
- **F → G**: `launch.json` isn't strictly blocked by the app content, but it's most useful (and most fairly validatable) once there's something to preview. It **can** be produced earlier in parallel if the Coder prefers (see §6).
- **G → H**: Final validation includes launching via the debug configuration.

---

## 6. Parallel vs. Sequential Work

### Can run in parallel
- **`app/project-data.json` (Coder) and `app/styles.css` scaffolding (Designer)** once step A (schema + hook agreement) is complete. They touch different files and depend only on the agreed contract.
- **`.vscode/launch.json` (Coder)** can be produced in parallel with `styles.css` (Designer) and even in parallel with rendering-script authoring, because it depends only on the directory layout (`app/index.html` exists) and the port choice — not on the app's internal behavior. Recommended to start it as soon as `app/index.html` exists as a stub.
- **Designer accessibility review of `index.html`** can happen in parallel with the Coder writing the render script, provided the semantic skeleton is stable.

### Must run sequentially
- **Schema/hook agreement (A) before everything else.** Renaming hooks or fields later invalidates both Designer's CSS and Coder's render logic.
- **`app/index.html` skeleton (B) before `app/styles.css` finalization (D).** Designer needs the final DOM shape to write and test selectors.
- **`app/project-data.json` (C) before the render script (E).** The render script's field access and modifier-class derivation depend on the exact keys and value strings.
- **Both D and E before integration verification (F).** You can't verify populated, styled cards without both.
- **F before final validation (H).** Launching a broken app via `launch.json` proves nothing.

### Coordination notes
- The Orchestrator should freeze the status vocabulary (`On Track`, `At Risk`, `Blocked`, `Complete`) and priority vocabulary (`High`, `Medium`, `Low`) at handoff so Designer's modifier class names and Coder's slugging logic agree exactly.
- If the Designer wants to change a class name mid-flight, it must be echoed back into `index.html` and the render script in the same phase.

---

## 7. Edge Cases to Handle

1. **Empty `projects` array** — render a friendly empty state, not a blank page.
2. **Missing or malformed `project-data.json`** — catch fetch/parse errors; render an error state; log a clear message; do not throw uncaught exceptions (no console errors under normal operation).
3. **Unknown `status` or `priority` value** — fall back to a neutral badge/priority class rather than crashing or producing an unstyled element.
4. **Long project names or activity strings** — must wrap or truncate gracefully; no horizontal overflow of the card or grid.
5. **Very small viewports (~320px)** — single-column layout must remain readable and tap targets ≥ 44px.
6. **High-contrast / reduced-motion / dark UA preferences** — respect `prefers-reduced-motion`; ensure focus states are visible even in forced-colors mode.
7. **Color-only status signaling** — must not be the sole channel; badges must have text labels (and ideally an icon or shape).
8. **`file://` vs `http://` loading** — `fetch('./project-data.json')` will fail under `file://` in some browsers due to CORS. The `launch.json` configuration solves this by serving over `http://localhost`; document this in the plan so learners don't just double-click `index.html` and get confused.
9. **Port already in use** — the Coder should pick a reasonable default port (e.g., 8000) but the `serverReadyAction` regex should capture whatever port the server reports so a fallback still opens the correct URL.
10. **Strict JSON discipline** — no comments in `.vscode/launch.json` or `project-data.json` (validation runs `python3 -m json.tool`).

---

## 8. Validation Expectations

Validation is performed in three tiers.

### 8.1 Automated (already wired in the repo)
- `python3 -m json.tool .vscode/launch.json` must succeed.
- The launch configuration name must be exactly `Run Project Pulse Dashboard`.
- `index.html` must contain the substrings `dashboard`, `project-card`, and `index.html` (launch target).
- `styles.css` must contain `.dashboard`, `.project-card`, `border-radius`, and `box-shadow`.
- `project-data.json` must contain a top-level `projects` key and the fields `name`, `owner`, `status`, `recentActivity`, `priority`.
- These are enforced by `.github/workflows/3-step.yml` via `skills/action-keyphrase-checker@v2`.

### 8.2 Manual functional
- Run **VS Code → Run and Debug → "Run Project Pulse Dashboard"**. Expect a local static server to start rooted in `app/` and a browser tab to open at `http://localhost:<port>/index.html`.
- Confirm the page renders one `.project-card` per entry in `app/project-data.json`.
- Confirm each card displays `name`, `owner`, `status` (as a badge), `priority` (as a treatment/badge), and `recentActivity`.
- Modify `project-data.json` (add/remove a project, change a status) and reload — the UI updates without code changes.
- Open browser DevTools → Console: expect **zero errors and zero warnings** in normal operation.

### 8.3 Responsive & accessibility
- Resize the viewport from ~320px to ~1440px:
  - Below ~640px: single column, no horizontal scroll.
  - ~640–1023px: two columns.
  - ≥1024px: three or more columns.
- Tab through the page: focus indicators must be visible.
- Run an axe/Lighthouse a11y audit: no critical violations; contrast ≥ AA on all badges and body text.
- Verify status is distinguishable without color (icon/label/shape).
- Verify `prefers-reduced-motion` disables any non-essential transitions.

---

## 9. Open Questions

1. **Should the render script live inside `index.html` (inline `<script>`) or in a separate `app/app.js`?**
   Recommendation: **inline**, because only four files are named in the assignment and the validation workflow. If the Orchestrator wants a separate JS file, it should be declared explicitly before the Coder starts step E.
2. **Which static server should `launch.json` use?**
   Recommendation: `python3 -m http.server` (already available in the devcontainer), with `cwd: ${workspaceFolder}/app` and a `serverReadyAction` regex that captures the port. Alternative: a Node `http-server`, but that adds a dependency.
3. **Should the dashboard support any interactivity (filter/sort by status or priority)?**
   Not required by validation. Recommend deferring; if added later, the Designer will need to spec filter-control styling and the Coder will need to add controls to `index.html` and handlers to the render script.
4. **Timestamp / "last updated" display?**
   Not required. If desired, the Coder can display `Date.now()` in the footer; no data-file changes needed.
5. **Should we seed a specific number of projects, and should Mona's real project names be used?**
   Recommend 4–8 fictional but plausible projects covering every status × priority combination the Designer needs to demonstrate. Confirm with Mona if real names should be used.
6. **Preferred default port for the local server?**
   Recommend `8000`. Confirm no conflict with other Codespace tasks.
---
