---
name: build-widget
description: Build a new embeddable widget in the beshir-widgets repo through a demesne pipeline and open a PR for review — a slow-tier orchestrator runs research (including finding data sources) → plan → implement (copying a sibling widget as a template or from scratch) → an authoritative in-sandbox build + validate gate → a headless browser-image loop that proves the widget renders and then iterates on its visual quality → review and fix, then a host-side landing that pushes a branch and opens a PR with a render screenshot. Apply when the user wants a small self-contained web visualization or demo turned into a deployable widget — e.g. "make a widget that visualizes X", "build an embeddable demo of this concept", "add a widget to beshir-widgets", "visualize this math/logic idea". Skip for tweaks to one existing widget's internals (a normal edit), for non-widget code, and when the demesne MCP or the beshir-widgets repo isn't reachable.
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, mcp__demesne__sandbox_agent, mcp__demesne__sandbox_research, mcp__demesne__sandbox_script, mcp__github__create_pull_request
---

Turn a concept into a new widget in `beshir-widgets` — an independent, iframe-embeddable Preact + Vite SPA deployed to its own `<slug>.widgets.beshir.org` subdomain on Cloudflare Workers. A slow-tier orchestrator copies the repo, runs research → plan → implement → an authoritative in-sandbox build+validate gate → a `browser`-image loop that first proves the widget renders and then runs visual-polish cycles → review and fix, commits on a branch, and delivers `/out/repo`. The host lands the branch, pushes it, and opens a PR — then hands you the PR link and the render screenshot for a quick review.

Repo: `jbeshir/beshir-widgets`, cloned at `/home/jbeshir/code/beshir-widgets`. Each `widgets/<slug>/` is one independent app; `function-plotter` is the reference template. Two repo docs are the source of truth for choices — read them in the pipeline, don't reinvent them:
- `LIBRARIES.md` — the curated, sized library menu and the "How To Choose" guidance.
- `DATA.md` — the per-widget data contract (`static` / `prebake` / `live`).

**Default stack:** **Preact core** runtime; **Observable Plot** for chart-like widgets. Aim for a genuinely polished result — prettiness-first within a **~100–150 KB gzip** budget. Drop to the lightweight tier (Preact + modular D3/SVG, ~19 KB) only as a deliberate choice when bytes matter. For non-chart widgets (simulation, geometry/graphs, animation), pick from `LIBRARIES.md`.

**Watch out:** Every widget must be **fully self-contained** — no runtime CDN/external `<script>`/`<link>`, everything bundled by Vite — because the E2E render runs at `egress=none` and the widget must work embedded anywhere. The orchestrator must `cp -a /workspace/repo /out/repo` itself (a `sandbox_script` child writes only to `/out/child/<name>`). `/workspace` is torn down on exit; only `/out` persists.

## The repo's per-widget convention

A widget directory carries: `package.json` + `package-lock.json`, `vite.config.ts` with **`base: './'`** (so the build loads over `file://` for the render and at the subdomain root), `index.html`, `src/`, `public/_headers` (removes `X-Frame-Options`/`Content-Security-Policy` so it embeds anywhere), `wrangler.jsonc` (`assets` Static-Assets block + a `routes` entry with `custom_domain: true`), `widget.json` (machine-readable metadata: slug/title/description/hostname/dataSources/embed snippet, plus an optional `data` block per `DATA.md`), and a `README.md`. The slug drives everything: dir basename = `widget.json.slug`, Worker name = `widget-<slug>`, hostname = `<slug>.widgets.beshir.org`. Deployment is declarative — Cloudflare Custom Domains auto-create DNS + TLS on `wrangler deploy`; no DNS scripting.

Two shared scripts already live in the repo and are reused by the gate, not rewritten:
- `scripts/validate-widgets.mjs` — asserts the slug/Worker/hostname/wrangler invariants and validates the optional `widget.json` `data` block.
- `scripts/render.cjs` — CommonJS Playwright harness; loads `file://<dist/index.html>`, waits for `#widget-ready`, screenshots to `/out`. It already carries the file:// + ES-module fixes (`--allow-file-access-from-files`, `waitForSelector` state `attached`). Every widget must render an element with `id="widget-ready"` once its first paint is done — that is the render gate's assertion.

## Data (per `DATA.md`)

Most visualization widgets are **`static`** (self-contained, no external data). If the concept needs real data, the widget always ships a committed **`sample`** dataset so dev and the offline E2E render work with no egress, and declares a `data` mode in `widget.json`:
- **`prebake`** — the host/CI fetches + normalizes real data into the bundle at deploy time (open-egress lane); the build/validate/render lanes only ever see `sample`. The pipeline produces the `sample` and a `prebake` command; the host runs `prebake` when wiring deploy.
- **`live`** — the deployed Worker fetches at runtime; `sample` is the offline/E2E fallback.

The agent build/validate/render lanes can never reach the open web — only the host/CI or the deployed Worker can. Build and render against `sample`.

## Procedure

1. **Host prep.** Ensure the local clone is current: `git -C /home/jbeshir/code/beshir-widgets switch main`, then `git -C … pull --ff-only`. Derive the new widget's `<slug>` (lowercase, hyphenated) from the concept.

2. **Launch the orchestrator** (slow tier, `agent=claude-code model=opus` when driving Claude) with `directories: ["/home/jbeshir/code/beshir-widgets"]`. It copies the repo whole (incl. `.git`) into `/workspace/repo` and runs steps 3–9 autonomously.

3. **Research** — a `sandbox_research` child (isolated, open egress). It nails the math/algorithm and **finds appropriate data sources** for the concept, or concludes the widget is self-contained. Because research can't see the repo, the orchestrator first reads one or two sibling widgets (e.g. `function-plotter`) and `LIBRARIES.md`/`DATA.md`, and passes their salient shape into the research prompt. → `/out/FINDINGS.md`. For a data widget, the research output must include a representative **`sample` dataset** (so dev/E2E stay offline) and, for `prebake`, the fetch+normalize recipe per `DATA.md`.

4. **Plan** — the orchestrator decides copy-a-template (default: clone `widgets/function-plotter` and rework it) vs scratch, picks the **stack** (Preact core; Observable Plot for charts; otherwise the right tool from `LIBRARIES.md`) and the **data mode** (`static`/`prebake`/`live`), fixes the `<slug>`, and writes `/workspace/PLAN.md` with a handoff contract.

5. **Implement** — one or two medium-tier `sandbox_agent` children (`name=phase01`, …; lowercase DNS-1123 labels). They create `widgets/<slug>/` honouring the convention above — Preact core, the chosen viz library, the `#widget-ready` marker, `base: './'`, `public/_headers`, `widget.json` (incl. the `data` block + committed `sample` for data widgets), and `wrangler.jsonc`. Build it for polish from the start: a clean card, `prefers-color-scheme` dark mode, `ResizeObserver` width, `prefers-reduced-motion`. Phases share `/workspace` — run them sequentially. Each writes `/out/SUMMARY.md`.

6. **Validate** (authoritative gate) — a `sandbox_script` child, `name=validate`, `image=node`, `egress=package-managers`. In `widgets/<slug>/`: `npm install` (generates `package-lock.json` — keep it) then `npm run build` (produces `dist/`); then at repo root `node scripts/validate-widgets.mjs`. All must exit 0. On failure, spawn a fix phase and re-validate.

7. **E2E render — correctness** — a `sandbox_script` child, `name=render`, `image=browser`, `egress=none`. The build from step 6 left `dist/` in the shared `/workspace/repo`. Run `node /workspace/repo/scripts/render.cjs /workspace/repo/widgets/<slug>/dist/index.html widget-ready /out`. It must exit 0 and write `render-ok` + a non-blank `screenshot.png`. The `browser` image builds on first use (slow first call). If it fails (e.g. `#widget-ready` never appears, or the page is blank), spawn a fix phase, re-validate, re-render. Cap render+fix at 3 rounds.

8. **Visual-enhancement cycles** — once it renders correctly, **do not stop at "it works."** Run **at least one, ideally two or three** aesthetic-polish passes. Each pass: the orchestrator stages the latest render (`cp` the render child's `screenshot.png` into `/workspace`) and **looks at it** (Read the PNG — it renders visually), then spawns a medium-tier `sandbox_agent` (`name=polish01`, `polish02`, …) that is also shown the screenshot and improves the *visual quality* — layout, spacing, typography, colour, axis/legend/grid treatment, light/dark balance, responsive fit, empty/error states — then re-build (`validate`) and re-render and compare screenshots. Judge "is it genuinely attractive and clear?", not just "did it render." Stop when a pass yields no worthwhile improvement, or after 3 passes. This is a first-class gate: a correct-but-ugly widget is not done.

9. **Review + finalize** — a medium-tier `sandbox_agent` (`name=review01`) reads `git -C /workspace/repo diff`, judges quality, completeness vs the concept, **and visual polish**, and writes `/out/REVIEW.md` ending in `PASS` or `CHANGES_NEEDED` (≤3 rounds; fix and re-validate/re-render/re-polish between rounds). Then: set git identity to `Pipeline <pipeline@local>`, branch `widget/<slug>` from the mounted HEAD, commit the widget (single commit). The orchestrator itself runs `cp -a /workspace/repo /out/repo`, copies the final render `screenshot.png` to `/out/screenshot.png`, and writes `/out/CHANGES.md` (branch, base commit, slug, hostname, stack + data mode, what the widget does, the gate summary, how many polish passes ran, the screenshot path). Print `DONE`.

## Launching the orchestrator

- **`directories: ["/home/jbeshir/code/beshir-widgets"]` is mandatory.** Without it the orchestrator wakes with no repo.
- Tier: **slow** orchestrator; **medium** implement/polish/review phases; the `validate`/`render` children are `sandbox_script` (no tier — `image=node` / `image=browser`).
- Tell it explicitly: **do NOT build, validate, or render yourself** — the agent image has no Node toolchain or browser; those are `sandbox_script` children.
- Restate the grounding facts (Workers Static Assets, `custom_domain: true` auto-DNS, iframe `_headers`, `base: './'`, self-contained/`egress=none`, the `#widget-ready` marker, the default stack) and that `LIBRARIES.md`/`DATA.md` are the authority for stack and data choices.

## Writing the orchestrator prompt

Brief it as a complete document: the concept and **what a good, attractive demo of it looks like**; the per-widget convention, the default stack, and the two shared scripts to reuse; the nine pipeline steps with the child-naming rule and that research is isolated; the explicit VALIDATE (`image=node`, `npm install` + `npm run build` + `node scripts/validate-widgets.mjs`) and RENDER (`image=browser` `egress=none`, `scripts/render.cjs … widget-ready /out`) commands; the **visual-enhancement loop (≥1, ideally 2–3 passes) where the screenshot is actually viewed and the result iterated for prettiness**; the data contract (committed `sample`, mode in `widget.json`); the self-contained / offline-render constraint; and the output contract (`/out/FINDINGS.md`, per-phase `/out/SUMMARY.md`, `/out/REVIEW.md` with a verdict line, `/out/repo` with branch `widget/<slug>`, `/out/CHANGES.md`, `/out/screenshot.png`, `DONE`).

## Host-side landing + PR

The in-sandbox build+validate+render+polish gate is authoritative; the host does not re-run it.

1. Read `/out/CHANGES.md` for the branch (`widget/<slug>`), base commit, and hostname.
2. Fetch and create the branch locally:
   - `git -C /home/jbeshir/code/beshir-widgets fetch <output_dir>/repo widget/<slug>`
   - `git -C … switch -c widget/<slug> FETCH_HEAD` (fallback: if `<output_dir>/repo` is absent, fetch from `<output_dir>/child/<name>/repo`).
3. Re-author the commit with the host identity: `git -C … commit --amend --reset-author --no-edit`; confirm with `git log -1 --format='%an <%ae>'`.
4. Push: `git -C … push -u origin widget/<slug>`.
5. Open a PR with `mcp__github__create_pull_request` (base `main`, head `widget/<slug>`); the body summarises the widget, the stack + data mode, the gate results, and the future URL `https://<slug>.widgets.beshir.org`. `check-widgets.yml` runs on the PR.
6. Give the user the **PR URL** and attach `/out/screenshot.png` (SendUserFile) for the quick review. Merging the PR triggers `deploy-widgets.yml` → live at `<slug>.widgets.beshir.org` (once the repo's one-time Cloudflare secrets are set). For a `prebake` widget, run its `prebake` command (host/CI, open egress) before the first deploy so real data lands in the bundle.

## Output contract

```
/out/
  repo/           # full .git repo; host fetches widget/<slug> from here
  screenshot.png  # the final polished E2E render, surfaced to the user for review
  FINDINGS.md     # research: algorithm + data sources (or self-contained); sample data for data widgets
  SUMMARY.md      # per implement/polish phase
  REVIEW.md       # verdict: PASS or CHANGES_NEEDED
  CHANGES.md      # branch, base commit, slug, hostname, stack + data mode, gate summary, polish passes, screenshot path
```
