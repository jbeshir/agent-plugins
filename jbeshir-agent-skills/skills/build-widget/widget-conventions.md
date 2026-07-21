# Widget conventions

Use the repository's `LIBRARIES.md`, `DATA.md`, and `TESTING.md` as the source of truth. Inspect one or two current sibling widgets before planning; prefer `function-plotter` when it remains representative.

## Application contract

Create `widgets/<slug>/` as an independent Preact + Vite SPA deployed to `<slug>.widgets.beshir.org` on Cloudflare Workers. Keep these values consistent:

- directory basename and `widget.json.slug`: `<slug>`;
- Worker name: `widget-<slug>`;
- hostname: `<slug>.widgets.beshir.org`;
- Vite `base`: `'./'`;
- Wrangler Static Assets configuration and custom-domain route;
- iframe-permissive `public/_headers`;
- metadata and embed snippet in `widget.json`.

Include `package.json`, `package-lock.json`, `vite.config.ts`, `index.html`, `src/`, `public/_headers`, both favicon files, `wrangler.jsonc`, `widget.json`, and `README.md`. Include `journey.json` for every interactive widget.

## Stack selection

Use Preact core by default. Use Observable Plot for chart-like work. Select another library from `LIBRARIES.md` only when a required capability is missing from the defaults; record the missing capability and selection rationale in `PLAN.md`.

Treat roughly 400 KB gzip as the normal ceiling. Exceed it only when the plan identifies a necessary capability with no smaller suitable option and the widget provides an explicit loading state.

## Offline and data behavior

Make development, build, first-paint, and journey capture succeed with no network access.

- `static`: bundle all data.
- `prebake`: commit representative sample data and add a documented fetch-and-normalize command for host/CI use.
- `live`: commit representative sample data as the offline fallback and show loading/error states for production fetches.

Declare the mode and sources in `widget.json` according to `DATA.md`. Research must record algorithm assumptions, validation examples, source-selection criteria, URLs and licenses when applicable, and either a representative offline sample or the reason no external data is needed.

## Render and journey hooks

Add `#widget-ready` only when first paint is complete. Interactive widgets must also:

- set `data-widget-state` on `<html>` for every declared lifecycle state;
- expose stable `data-testid` attributes and accessible roles/names on controls;
- define `journey.json` using the repository action vocabulary;
- cover initial, populated, and applicable empty/loading/error states;
- use state markers rather than arbitrary settle delays.

Capture every declared state across the required viewport and color-scheme matrix. Freeze time/randomness and disable animation through the shared harness.

## Visual and accessibility floors

Provide a polished responsive card, light and dark themes, `ResizeObserver`-based width handling, and reduced-motion behavior. Use sans-serif UI/body text unless a display heading of at least 18 px intentionally uses serif. Require body text at least 16 px, secondary text at least 14 px, and no text below 12 px. Require WCAG AA contrast and APCA Lc 75 in both themes; use off-white on near-black rather than pure white on black.

For multiple floating layers, render them through one dedicated overlay root. Permit a non-portal implementation only when a platform/library constraint prevents portaling and `PLAN.md` records that constraint; then define a documented z-index token scale and audit every ancestor for stacking contexts. Keep user-invoked popovers/modals above persistent chrome. Add a mobile compound journey state with at least two layers visible together.

## Embedding

Detect iframe use and add `embedded` to `<html>`; allow `?embed=0` to disable the framed appearance. In embedded mode, make the body transparent and remove outer width, padding, border, radius, and shadow while preserving an opaque theme-correct widget surface.

Observe the root and post `{ type: 'resize', height }` to `window.parent`. Report natural untransformed height for scale-to-fit widgets. Document the host listener in the widget README.

## Favicon

Create a `32×32` SVG with a rounded rose-gradient tile and one legible gold/cream glyph related to the widget. Include a dark-scheme media rule and keep the glyph bold at 16 px. Generate the 16/32/48 ICO from the SVG's light variant through a checked-in repository script with pinned dependencies. If the repository lacks that helper, add one reusable script and its pinned dependencies during implementation. Do not embed an ad-hoc shell-quoted `node -e` recipe in the orchestration prompt.

Include ICO first and SVG second in `index.html`:

```html
<link rel="icon" href="./favicon.ico" sizes="any" />
<link rel="icon" type="image/svg+xml" href="./favicon.svg" />
```
