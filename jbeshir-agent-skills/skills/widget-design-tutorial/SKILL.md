---
name: widget-design-tutorial
description: Write a UI-design and CSS tutorial for one widget in the beshir-widgets repo through a demesne pipeline and open a PR. A slow-tier orchestrator runs research → outline → per-section drafting (one subagent per section) → tie-together → a review-improve cycle, producing a markdown tutorial that explains the widget's visual design choices and how its CSS achieves them — how styles nest and cascade, how design tokens and theming work, and which CSS features carry the layout, type, colour, and responsive behaviour. Each tutorial builds on earlier ones in the track and is deliberately steered to cover CSS ground the previous tutorials have not. Apply when the user wants the design/styling side of a widget explained — "write the design tutorial for X", "explain the CSS choices in this widget", "how is this widget styled", "add a styling walkthrough". Skip for the React/JS build-through (use widget-frontend-tutorial) and for building the widget itself (use build-widget).
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, mcp__demesne__sandbox_agent, mcp__demesne__sandbox_research, mcp__demesne__sandbox_script, mcp__github__create_pull_request
---

Produce one **design and CSS tutorial** for a single widget in `beshir-widgets` through a demesne pipeline, and land it as a PR. The tutorial reads the widget's stylesheet and markup and teaches the *visual* design: the choices made (layout, spacing, typography, colour, dark mode, states), why they were made, and the CSS mechanisms that realise them — the cascade, nesting and how selectors interact, custom properties and theming, and the specific features each effect relies on. It is one entry in a growing curriculum whose defining rule is **cover new CSS ground**: each tutorial spends its depth on techniques the earlier tutorials have not already taught. A slow-tier orchestrator runs research → outline → per-section drafting → tie-together → review-improve in a sandbox, commits the tutorial on a branch, and delivers `/out/repo`; the host lands it and opens the PR.

Repo: `jbeshir/beshir-widgets`, cloned at `/home/jbeshir/code/beshir-widgets`. Each `widgets/<slug>/` carries its own `src/styles.css` and `index.html`/`src/App.tsx` markup. The widgets share a house style — a `:root` design-token block with a `prefers-color-scheme: dark` override, a centred card, focus rings, `prefers-reduced-motion` guards. `LIBRARIES.md` carries the responsive/iframe guidance (`ResizeObserver`, `prefers-color-scheme`, `prefers-reduced-motion`); the design choices themselves come from the widget's own CSS.

## Where tutorials live — the curriculum

Design tutorials live in `tutorials/design/` (the pipeline creates it on first run). One file per widget, numbered in reading order:

```
tutorials/design/
  README.md                  # ordered index: number · widget · CSS themes covered · link
  01-function-plotter.md
  02-japanese-verb-tower.md
  ...
```

The track is **progressive and non-repeating**. Tutorial N assumes the reader has read 1…N−1, and its job is to teach what they haven't seen:

- **Cover new CSS ground.** This is the central constraint. The outline stage reads the `## CSS techniques introduced` section of every existing design tutorial and assembles the covered-set; the new tutorial leads with the techniques this widget uses that are *not* in it. For a technique already taught, name it, link where it was introduced (`the token/dark-mode pattern from [01](./01-function-plotter.md)`), and don't re-explain.
- **The ledger is the tutorials themselves.** Every design tutorial ends with `## CSS techniques introduced` — the CSS features/techniques it taught for the first time (e.g. `:focus-visible` rings, grid `minmax`, `clamp()` type scale, container queries, `color-mix()`, stacking contexts, transitions). That list is the source of truth for the covered-set; don't duplicate it into the README.
- **If a widget's CSS is mostly familiar,** the outline finds the parts that aren't — an unusual selector, a layout technique, a colour or state treatment this widget needed — and builds around those. A short tutorial on one genuinely new technique beats a long re-tour of the house style.
- **Cross-link the sibling track.** Component structure, hooks, and JS belong to the frontend tutorial; link `../frontend/<same-number>-<slug>.md` for those and stay on styling here.

## Resolving the target widget (host prep)

The user names a widget by `<slug>` or by an open widget PR. Get the right code into the clone **before** launching the orchestrator (it copies the clone whole into `/workspace/repo`):

- **Merged / on `main`:** `git -C /home/jbeshir/code/beshir-widgets switch main`, then `git -C … pull --ff-only`. The mounted HEAD is `main`.
- **An open widget PR:** check the PR is still open and **not already merged** (`gh pr view <n> --json state,headRefName,mergedAt`). If merged, treat as the merged case. If open, check out its branch so `src/styles.css` and the tutorial's base are that branch (`git -C … fetch origin <headRef>`, `git -C … switch <headRef>`); the tutorial appends to it so they review and merge together.

Confirm `widgets/<slug>/src/styles.css` exists before going further.

## Pipeline stages

The orchestrator runs these in `/workspace/repo` (the copied clone):

1. **Research** — a `sandbox_research` child (isolated, open egress). Research **can't see the repo**, so the orchestrator first reads `src/styles.css` and its markup and passes the salient shape into the prompt: the layout mechanism, the token/theming approach, the selectors and states in play, and the CSS techniques the outline expects to teach. The child returns accurate, current background for those features — semantics, browser support, accessibility and contrast guidance, idiomatic usage — so the teaching rests on facts, not recall. → `/out/FINDINGS.md`.

2. **Outline** — a medium-tier `sandbox_agent` (`name=outline`). It reads `styles.css` and its markup, `FINDINGS.md`, and the `## CSS techniques introduced` section of every existing design tutorial (the covered-set). It diffs the widget's techniques against the covered-set and writes `/workspace/OUTLINE.md`: the section list led by the **new** techniques, each section's teaching goal, the `styles.css:<line>` it anchors to, and whether it introduces a technique or just references an earlier tutorial. → also copied to `/out/OUTLINE.md`.

3. **Per-section drafting** — **one `sandbox_agent` child per outline section** (`name=sec01`, `sec02`, …; lowercase DNS-1123 labels). Each reads only its section's slice of `OUTLINE.md` plus the CSS and markup it anchors to, and writes that section's markdown to `/workspace/drafts/<NN>-<key>.md`. Because each writes a disjoint draft file and only reads the repo, the section children run concurrently — dispatch each with `background: true` (collect its `job_id`) and poll with `sandbox_wait`, keeping ≤8 in flight; blocking children are issued one per turn and run sequentially, so background dispatch is what runs the sections concurrently. Each drafter gives the *why* (the visual/usability goal) and the *how* (the CSS feature, and how the cascade/nesting/specificity make it land), quotes small real excerpts with `styles.css:<line>` anchors, and for a referenced (already-covered) technique writes the short cross-link rather than re-teaching it.

4. **Tie-together** — a medium-tier `sandbox_agent` (`name=stitch`). It reads all section drafts in order and composes the single tutorial at `tutorials/design/<NN>-<slug>.md`: a "The look" intro, the stitched sections with smooth transitions and one consistent voice, dedup, the cross-links, and the closing `## CSS techniques introduced` ledger (precise — nothing an earlier tutorial already owns). It updates `tutorials/design/README.md` with the new row and the CSS themes this entry covers.

5. **Review-improve cycle** — a medium-tier `sandbox_agent` (`name=review01`) reads the finished tutorial against `styles.css` and judges: every excerpt is real and correctly anchored (no fabrication, no CSS feature credited that the widget doesn't use), the new-ground discipline holds (covered techniques are linked not re-taught), it explains *interaction* (cascade/specificity/nesting/inheritance) not just lists declarations, prose is lean, and the ledger is precise. It writes `/out/REVIEW.md` ending in `PASS` or `CHANGES_NEEDED`. On `CHANGES_NEEDED`, re-run tie-together (or specific drafters) and re-review; cap at 3 rounds.

Then the orchestrator commits on a branch (see Launching), runs `cp -a /workspace/repo /out/repo`, writes `/out/CHANGES.md`, and prints `DONE`. `/workspace` is torn down on exit; only `/out` persists, so the orchestrator must do the `cp -a` itself (a `sandbox_script` child writes only to `/out/child/<name>`).

**Child artefacts route through shared `/workspace`, not child `/out`.** A child's `output_path` resolves inside *that child's own* filesystem, so a child writing to `/out/FINDINGS.md` actually lands it at `/out/child/<name>/.../FINDINGS.md` — nested and easy to lose. Have every child write into shared `/workspace` (e.g. the research child to `/workspace/FINDINGS.md`, drafters to `/workspace/drafts/`, outline to `/workspace/OUTLINE.md`); the **orchestrator** then copies the top-level `/out/*.md` artefacts (`FINDINGS.md`, `OUTLINE.md`, `REVIEW.md`, `CHANGES.md`) up itself. The tutorial files the children write under `/workspace/repo/tutorials/` are already in the right place and ride along in the `cp -a`.

## Launching the orchestrator

- **`directories: ["/home/jbeshir/code/beshir-widgets"]` is mandatory.** Without it the orchestrator wakes with no repo. It copies the clone (incl. `.git`) into `/workspace/repo` and runs the stages there.
- Tier: **slow** orchestrator (`agent=claude-code model=opus` when driving Claude); **medium** outline/section/stitch/review children; research is `sandbox_research` (isolated, open egress).
- This pipeline writes **markdown only** — nothing to build or render. Don't add a build gate. Reference accuracy is enforced in the review stage by checking excerpts against `/workspace/repo` source.
- Brief the orchestrator as a complete document: the widget slug and its visual intent; the curriculum and the `## CSS techniques introduced` ledger (read prior tutorials' ledgers to get the covered-set; **cover new CSS ground**, link the rest); the tutorial's target arc (below); the five stages with the child-naming rule, that research is isolated and repo-blind, and **one section child per outline section**; and the output contract (`/out/FINDINGS.md`, `/out/OUTLINE.md`, `/out/REVIEW.md` with a verdict line, `/out/repo` with branch `tutorial/design-<slug>`, `/out/CHANGES.md`, `DONE`). Set git identity to `Pipeline <pipeline@local>`, branch `tutorial/design-<slug>` from the mounted HEAD, single commit.

### Target arc for the tutorial

Adapt to the widget and lead with its new ground; the arc to draw from:

1. **The look** — what the widget should feel like and the design intent; link the live URL (`https://<slug>.widgets.beshir.org`).
2. **How the styles are organised** — the shape of `styles.css` (tokens → base → components → states → media) and *why that order* matters to the cascade; teach once for the track, link thereafter.
3. **Theming with custom properties** — `--token` on `:root` plus the `prefers-color-scheme: dark` override producing light/dark from one rule set, and how `var()` keeps component rules theme-agnostic; introduce once, link when reused.
4. **The new techniques this widget brings** — the heart: each technique this widget introduces that the covered-set lacks (layout — flex/grid, `minmax`, `gap`, `clamp`; a selector or cascade interaction worth dwelling on; colour handling; state styling; motion). For each: the goal, the CSS feature, and how nesting/specificity/inheritance make it work.
5. **States and accessibility** — focus rings, error/disabled treatment, contrast in both themes, `prefers-reduced-motion`; reference the shared patterns, teach only this widget's additions.
6. **Responsive behaviour** — how it fits an arbitrary iframe (fluid widths, media/container queries) and where JS (`ResizeObserver`) hands sizing back to CSS (cross-link the frontend tutorial for the JS half).
7. **`## CSS techniques introduced`** — the CSS features/techniques this tutorial taught for the first time; feeds the next tutorial's covered-set.

## Output contract

```
/out/
  repo/         # full .git repo; host fetches the branch from here
  FINDINGS.md   # research background for the CSS features taught
  OUTLINE.md    # the agreed section outline, led by new techniques
  REVIEW.md     # verdict: PASS or CHANGES_NEEDED
  CHANGES.md    # branch, base commit, slug, track, tutorial path, sections, new techniques, summary
```

## Host-side landing + PR

The in-sandbox review is authoritative; the host does not re-run the pipeline.

1. Read `/out/CHANGES.md` for the branch (`tutorial/design-<slug>`), base commit, and tutorial path.
2. Fetch and create the branch locally: `git -C /home/jbeshir/code/beshir-widgets fetch <output_dir>/repo tutorial/design-<slug>` then `git -C … switch -c tutorial/design-<slug> FETCH_HEAD` (fallback: `<output_dir>/child/<name>/repo`).
3. Re-author the commit with the host identity: `git -C … commit --amend --reset-author --no-edit`.
4. Push: `git -C … push -u origin tutorial/design-<slug>`.
5. **Merged widget:** open a PR to `main` with `mcp__github__create_pull_request` — body: which widget, the live URL, the CSS techniques it newly introduces. **Open widget PR:** push the tutorial commit onto the widget's branch (the mounted HEAD was that branch) so it joins the existing PR — re-confirm the PR is open and unmerged before pushing.
6. Give the user the PR URL.
