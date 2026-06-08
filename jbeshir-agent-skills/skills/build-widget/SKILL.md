---
name: build-widget
description: Build a new embeddable widget in the beshir-widgets repo through a demesne pipeline and open a PR for review — a slow-tier orchestrator runs research (including finding data sources) → plan → implement (copying a sibling widget as a template or from scratch) → an authoritative in-sandbox build + validate gate → a headless browser-image E2E render loop that proves the widget renders → review and fix, then a host-side landing that pushes a branch and opens a PR with a render screenshot. Apply when the user wants a small self-contained web visualization or demo turned into a deployable widget — e.g. "make a widget that visualizes X", "build an embeddable demo of this concept", "add a widget to beshir-widgets", "visualize this math/logic idea". Skip for tweaks to one existing widget's internals (a normal edit), for non-widget code, and when the demesne MCP or the beshir-widgets repo isn't reachable.
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, mcp__demesne__sandbox_agent, mcp__demesne__sandbox_research, mcp__demesne__sandbox_script, mcp__github__create_pull_request
---

Turn a concept into a new widget in `beshir-widgets` — an independent, iframe-embeddable React+Vite SPA deployed to its own `<slug>.widgets.beshir.org` subdomain on Cloudflare Workers. A slow-tier orchestrator copies the repo, runs research → plan → implement → an authoritative in-sandbox build+validate gate → a `browser`-image E2E render loop → review and fix, commits on a branch, and delivers `/out/repo`. The host lands the branch, pushes it, and opens a PR — then hands you the PR link and the render screenshot for a quick review.

Repo: `jbeshir/beshir-widgets`, cloned at `/home/jbeshir/code/beshir-widgets`. Each `widgets/<slug>/` is one independent app; `function-plotter` is the reference template.

**Watch out:** Every widget must be **fully self-contained** — no runtime CDN/external `<script>`/`<link>`, everything bundled by Vite — because the E2E render runs at `egress=none` and the widget must work embedded anywhere. The orchestrator must `cp -a /workspace/repo /out/repo` itself (a `sandbox_script` child writes only to `/out/child/<name>`). `/workspace` is torn down on exit; only `/out` persists.

## The repo's per-widget convention

A widget directory carries: `package.json` + `package-lock.json`, `vite.config.ts` with **`base: './'`** (so the build loads over `file://` for the render and at the subdomain root), `index.html`, `src/`, `public/_headers` (removes `X-Frame-Options`/`Content-Security-Policy` so it embeds anywhere), `wrangler.jsonc` (`assets` Static-Assets block + a `routes` entry with `custom_domain: true`), `widget.json` (machine-readable metadata: slug/title/description/hostname/dataSources/embed snippet), and a `README.md`. The slug drives everything: dir basename = `widget.json.slug`, Worker name = `widget-<slug>`, hostname = `<slug>.widgets.beshir.org`. Deployment is declarative — Cloudflare Custom Domains auto-create DNS + TLS on `wrangler deploy`; no DNS scripting.

Two shared scripts already live in the repo and are reused by the gate, not rewritten:
- `scripts/validate-widgets.mjs` — asserts the slug/Worker/hostname/wrangler/widget.json invariants.
- `scripts/render.cjs` — CommonJS Playwright harness; loads `file://<dist/index.html>`, waits for `#widget-ready`, screenshots to `/out`. It already carries the file:// + ES-module fixes (`--allow-file-access-from-files`, `waitForSelector` state `attached`). Every widget must render an element with `id="widget-ready"` once its first paint is done — that is the render gate's assertion.

## Procedure

1. **Host prep.** Ensure the local clone is current: `git -C /home/jbeshir/code/beshir-widgets switch main`, then `git -C … pull --ff-only`. Derive the new widget's `<slug>` (lowercase, hyphenated) from the concept.

2. **Launch the orchestrator** (slow tier, `agent=claude-code model=opus` when driving Claude) with `directories: ["/home/jbeshir/code/beshir-widgets"]`. It copies the repo whole (incl. `.git`) into `/workspace/repo` and runs steps 3–8 autonomously.

3. **Research** — a `sandbox_research` child (isolated, open egress). It nails the math/algorithm and **finds appropriate data sources** for the concept, or concludes the widget is self-contained. Because research can't see the repo, the orchestrator first reads one or two sibling widgets (e.g. `function-plotter`) and passes their salient shape into the research prompt. → `/out/FINDINGS.md`. If the concept needs live data, prefer bundling a build-time snapshot or making the fetch optional with a bundled fallback, so the offline render still passes.

4. **Plan** — the orchestrator decides copy-a-template (default: clone `widgets/function-plotter` and rework it) vs scratch, fixes the `<slug>`, and writes `/workspace/PLAN.md` with a handoff contract.

5. **Implement** — one or two medium-tier `sandbox_agent` children (`name=phase01`, …; lowercase DNS-1123 labels). They create `widgets/<slug>/` honouring the convention above, including the `#widget-ready` marker, `base: './'`, `public/_headers`, `widget.json`, and `wrangler.jsonc`. Phases share `/workspace` — run them sequentially. Each writes `/out/SUMMARY.md`.

6. **Validate** (authoritative gate) — a `sandbox_script` child, `name=validate`, `image=node`, `egress=package-managers`. In `widgets/<slug>/`: `npm install` (generates `package-lock.json` — keep it) then `npm run build` (produces `dist/`); then at repo root `node scripts/validate-widgets.mjs`. All must exit 0. On failure, spawn a fix phase and re-validate.

7. **E2E render** — a `sandbox_script` child, `name=render`, `image=browser`, `egress=none`. The build from step 6 left `dist/` in the shared `/workspace/repo`. Run `node /workspace/repo/scripts/render.cjs /workspace/repo/widgets/<slug>/dist/index.html widget-ready /out`. It must exit 0 and write `render-ok` + a non-blank `screenshot.png`. The `browser` image builds on first use (slow first call). If it fails (e.g. `#widget-ready` never appears, or the page is blank), spawn a fix phase, re-validate, re-render. Cap render+fix at 3 rounds.

8. **Review + finalize** — a medium-tier `sandbox_agent` (`name=review01`) reads `git -C /workspace/repo diff`, judges quality + completeness vs the concept, and writes `/out/REVIEW.md` ending in `PASS` or `CHANGES_NEEDED` (≤3 rounds; fix and re-validate/re-render between rounds). Then: set git identity to `Pipeline <pipeline@local>`, branch `widget/<slug>` from the mounted HEAD, commit the widget (single commit). The orchestrator itself runs `cp -a /workspace/repo /out/repo`, copies the render `screenshot.png` to `/out/screenshot.png`, and writes `/out/CHANGES.md` (branch, base commit, slug, hostname, what the widget does, the gate summary, the screenshot path). Print `DONE`.

## Launching the orchestrator

- **`directories: ["/home/jbeshir/code/beshir-widgets"]` is mandatory.** Without it the orchestrator wakes with no repo.
- Tier: **slow** orchestrator; **medium** implement/review phases; the `validate`/`render` children are `sandbox_script` (no tier — `image=node` / `image=browser`).
- Tell it explicitly: **do NOT build, validate, or render yourself** — the agent image has no Node toolchain or browser; those are `sandbox_script` children.
- Restate the grounding facts (Workers Static Assets, `custom_domain: true` auto-DNS, iframe `_headers`, `base: './'`, self-contained/`egress=none`, the `#widget-ready` marker) and the per-widget convention, so phases don't relitigate them.

## Writing the orchestrator prompt

Brief it as a complete document: the concept and what a good demo of it looks like; the per-widget convention and the two shared scripts to reuse; the seven pipeline steps with the child-naming rule and that research is isolated; the explicit VALIDATE (`image=node`, `npm install` + `npm run build` + `node scripts/validate-widgets.mjs`) and RENDER (`image=browser` `egress=none`, `scripts/render.cjs … widget-ready /out`) commands; the self-contained / offline-render constraint; and the output contract (`/out/FINDINGS.md`, per-phase `/out/SUMMARY.md`, `/out/REVIEW.md` with a verdict line, `/out/repo` with branch `widget/<slug>`, `/out/CHANGES.md`, `/out/screenshot.png`, `DONE`).

## Host-side landing + PR

The in-sandbox build+validate+render gate is authoritative; the host does not re-run it.

1. Read `/out/CHANGES.md` for the branch (`widget/<slug>`), base commit, and hostname.
2. Fetch and create the branch locally:
   - `git -C /home/jbeshir/code/beshir-widgets fetch <output_dir>/repo widget/<slug>`
   - `git -C … switch -c widget/<slug> FETCH_HEAD` (fallback: if `<output_dir>/repo` is absent, fetch from `<output_dir>/child/<name>/repo`).
3. Re-author the commit with the host identity: `git -C … commit --amend --reset-author --no-edit`; confirm with `git log -1 --format='%an <%ae>'`.
4. Push: `git -C … push -u origin widget/<slug>`.
5. Open a PR with `mcp__github__create_pull_request` (base `main`, head `widget/<slug>`); the body summarises the widget, the gate results, and the future URL `https://<slug>.widgets.beshir.org`. `check-widgets.yml` runs on the PR.
6. Give the user the **PR URL** and attach `/out/screenshot.png` (SendUserFile) for the quick review. Merging the PR triggers `deploy-widgets.yml` → live at `<slug>.widgets.beshir.org` (once the repo's one-time Cloudflare secrets are set).

## Output contract

```
/out/
  repo/           # full .git repo; host fetches widget/<slug> from here
  screenshot.png  # the E2E render, surfaced to the user for review
  FINDINGS.md     # research: algorithm + data sources (or self-contained)
  SUMMARY.md      # per implement phase
  REVIEW.md       # verdict: PASS or CHANGES_NEEDED
  CHANGES.md      # branch, base commit, slug, hostname, gate summary, screenshot path
```
