---
name: widget-frontend-tutorial
description: Write a frontend (Preact/React) build-through tutorial for one widget in the beshir-widgets repo through a demesne pipeline and open a PR. A slow-tier orchestrator runs research → outline → per-section drafting (one subagent per section) → tie-together → a review-improve cycle, producing a markdown tutorial that walks the widget's code as a guide to building it yourself — framework and library choices, initialisation, hooks and state, domain logic, and how the pieces fit. Each tutorial builds on earlier ones in the track as assumed knowledge and only teaches the framework concepts the widget newly introduces. Apply when the user wants a code-walkthrough tutorial for a widget — "write the frontend tutorial for X", "tutorial-walk the React side of this widget", "explain how this widget is built", "add a build-through for the new widget". Skip for CSS/visual-design explanation (use widget-design-tutorial) and for building the widget itself (use build-widget).
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, mcp__demesne__sandbox_agent, mcp__demesne__sandbox_research, mcp__demesne__sandbox_script, mcp__github__create_pull_request
---

Produce one **frontend build-through tutorial** for a single widget in `beshir-widgets` through a demesne pipeline, and land it as a PR. The tutorial reads the widget's own code and teaches how to build that widget yourself from a Preact/React standpoint: why the framework and libraries were chosen, how the app initialises, how state and effects drive it, where the domain logic lives, and how the parts compose. It is a teaching document in a growing curriculum — it assumes everything earlier tutorials in the track taught and spends its depth on what *this* widget introduces. A slow-tier orchestrator runs research → outline → per-section drafting → tie-together → review-improve in a sandbox, commits the tutorial on a branch, and delivers `/out/repo`; the host lands the branch and opens the PR.

Repo: `jbeshir/beshir-widgets`, cloned at `/home/jbeshir/code/beshir-widgets`. Each `widgets/<slug>/` is one independent Preact + Vite SPA. The repo docs `README.md`, `LIBRARIES.md`, and `DATA.md` are the source of truth for the stack and the choices available — the orchestrator reads them so the tutorial's "why this library" matches the repo's actual menu rather than inventing rationale.

## Where tutorials live — the curriculum

Frontend tutorials live in `tutorials/frontend/` (the pipeline creates it on first run). One file per widget, numbered in reading order:

```
tutorials/frontend/
  README.md                  # ordered index: number · widget · one-line summary · link
  01-function-plotter.md
  02-japanese-verb-tower.md
  ...
```

The track is **progressive**. Tutorial N assumes the reader has read 1…N−1, so:

- **Don't re-teach what an earlier tutorial taught in depth.** When the widget reuses a covered concept (a `useState`/`useMemo` pattern, the imperative-library-in-`useEffect` integration, the `#widget-ready` marker, `ResizeObserver` sizing), name it, link the tutorial that introduced it (`see [02](./02-japanese-verb-tower.md)`), and move on. Spend the page on what is genuinely new here.
- **The ledger is the tutorials themselves.** Every frontend tutorial ends with a `## Concepts introduced` section — the framework/architecture techniques it taught *for the first time*. The outline stage reads that section in every existing frontend tutorial to assemble the known-set. It is the single source of truth for "already covered"; don't duplicate it into the README.
- **Cross-link, don't duplicate, the sibling track.** Styling and CSS are the design tutorial's job. Where layout or visual concerns surface, point to `../design/<same-number>-<slug>.md` instead of explaining CSS here.

## Resolving the target widget (host prep)

The user names a widget by `<slug>` or by an open widget PR. Make the **base ref** the orchestrator branches from available in the clone **before** launching it (it copies the clone whole into `/workspace/repo` and branches from the ref inside its copy) — **checkout-free, so the host working tree is never switched and concurrent tutorial/widget pipelines don't block each other**:

- **Merged / on `main`:** `git -C /home/jbeshir/code/beshir-widgets fetch origin main`. Base ref = `origin/main`.
- **An open widget PR:** check the PR is still open and **not already merged** (`gh pr view <n> --json state,headRefName,mergedAt`). If merged, treat as the merged case. If open, fetch its branch (no checkout): `git -C … fetch origin <headRef>`. Base ref = `origin/<headRef>` — the tutorial branches from and is pushed back onto it so they review and merge together.

Capture the host commit identity to hand the orchestrator (`git -C … config user.name` / `user.email`), so the in-sandbox commit is authored correctly and the host needs no re-author. Confirm the widget exists on the base ref: `git -C … ls-tree --name-only <base-ref> widgets/<slug>/`. The orchestrator derives the file number (next free in `tutorials/frontend/`) from its copy.

## Pipeline stages

The orchestrator runs these in `/workspace/repo` (the copied clone):

1. **Research** — a `sandbox_research` child (isolated, open egress). Research **can't see the repo**, so the orchestrator first reads the widget's source and `LIBRARIES.md`/`DATA.md` and passes the salient shape into the prompt: the framework, the key library, the hooks and domain modules in play, and the concepts the outline expects to teach. The child returns accurate, current background for those concepts — correct Preact/React hook semantics, the chosen library's idioms and integration pattern, current best practice — so the teaching rests on facts, not recall. → `/out/FINDINGS.md`.

2. **Outline** — a medium-tier `sandbox_agent` (`name=outline`). It reads the widget source, `FINDINGS.md`, and the `## Concepts introduced` section of every existing frontend tutorial (the known-set), then writes a section-by-section outline to `/workspace/OUTLINE.md`: each section's title, what it teaches, which code it anchors to (`file:line`), whether it introduces a new concept or just references an earlier tutorial, and a one-line teaching goal. The outline is the per-section drafters' contract. → also copied to `/out/OUTLINE.md`.

3. **Per-section drafting** — **one `sandbox_agent` child per outline section** (`name=sec01`, `sec02`, …; lowercase DNS-1123 labels). Each reads only its section's slice of `OUTLINE.md` plus the widget code it anchors to, and writes that section's markdown to `/workspace/drafts/<NN>-<key>.md`. Because each writes a disjoint draft file and only reads the repo, the section children run concurrently — dispatch each with `background: true` (collect its `job_id`) and poll with `sandbox_wait`, keeping ≤8 in flight; blocking children are issued one per turn and run sequentially, so background dispatch is what runs the sections concurrently. Each drafter quotes real code in small excerpts with `file:line` anchors and explains *why*, not just *what*; for a referenced (already-covered) concept it writes the short cross-link the outline specified rather than re-teaching.

4. **Tie-together** — a medium-tier `sandbox_agent` (`name=stitch`). It reads all section drafts in order and composes the single tutorial at `tutorials/frontend/<NN>-<slug>.md`: a "What you'll build" intro, the stitched sections with smooth transitions and one consistent voice, dedup of anything two drafters both said, the cross-links, and the closing `## Concepts introduced` ledger (accurate — nothing an earlier tutorial already owns). It updates `tutorials/frontend/README.md` with the new row.

5. **Review-improve cycle** — a medium-tier `sandbox_agent` (`name=review01`) reads the finished tutorial against the widget source and judges: every code excerpt is real and correctly anchored (no fabrication), the new-ground discipline holds (covered concepts are linked not re-taught), the teaching genuinely lets a reader rebuild the widget, prose is lean, and the ledger is accurate. It writes `/out/REVIEW.md` ending in `PASS` or `CHANGES_NEEDED`. On `CHANGES_NEEDED`, re-run tie-together (or the specific drafters) to fix and re-review; cap at 3 rounds.

Then the orchestrator commits the tutorial on a branch (see Launching), runs `cp -a /workspace/repo /out/repo`, writes `/out/CHANGES.md`, and prints `DONE`. `/workspace` is torn down on exit; only `/out` persists, so the orchestrator must do the `cp -a` itself (a `sandbox_script` child writes only to `/out/child/<name>`).

**Child artefacts route through shared `/workspace`, not child `/out`.** A child's `output_path` resolves inside *that child's own* filesystem, so a child writing to `/out/FINDINGS.md` actually lands it at `/out/child/<name>/.../FINDINGS.md` — nested and easy to lose. Have every child write into shared `/workspace` (e.g. the research child to `/workspace/FINDINGS.md`, drafters to `/workspace/drafts/`, outline to `/workspace/OUTLINE.md`); the **orchestrator** then copies the top-level `/out/*.md` artefacts (`FINDINGS.md`, `OUTLINE.md`, `REVIEW.md`, `CHANGES.md`) up itself. The tutorial files the children write under `/workspace/repo/tutorials/` are already in the right place and ride along in the `cp -a`.

## Launching the orchestrator

- **`directories: ["/home/jbeshir/code/beshir-widgets"]` is mandatory.** Without it the orchestrator wakes with no repo. It copies the clone (incl. `.git`) into `/workspace/repo` and runs the stages there.
- Tier: **slow** orchestrator (`agent=claude-code model=opus` when driving Claude); **medium** outline/section/stitch/review children; research is `sandbox_research` (isolated, open egress).
- This pipeline writes **markdown only** — there is nothing to build, validate, or render. Don't add a build gate. Reference accuracy is enforced in the review stage by checking excerpts against `/workspace/repo` source.
- Brief the orchestrator as a complete document: the widget slug and what it does; the per-track curriculum and the `## Concepts introduced` ledger (read prior tutorials' ledgers to get the known-set; teach only new ground, link the rest); the tutorial's target arc (below); the five stages with the child-naming rule, that research is isolated and repo-blind, and that there is **one section child per outline section**; and the output contract (`/out/FINDINGS.md`, `/out/OUTLINE.md`, `/out/REVIEW.md` with a verdict line, `/out/repo` with branch `tutorial/frontend-<slug>`, `/out/CHANGES.md`, `DONE`). Pass it the **base ref** (`origin/main`, or `origin/<headRef>` for an open widget PR) and the **host commit identity** (`user.name`/`user.email`). It must check out the base ref in its copy to read the widget source, set git identity to the host identity, branch `tutorial/frontend-<slug>` from that base ref (**not** the mounted working-tree HEAD), and make a single commit authored as the host.

### Target arc for the tutorial

Adapt to the widget; lead with its new ground:

1. **What you'll build** — what the widget does and what the reader will be able to build; link the live URL (`https://<slug>.widgets.beshir.org`) and the design tutorial for the look.
2. **The stack and why** — Preact core (and what it buys over React, only if not yet covered) and this widget's key library choice, grounded in `LIBRARIES.md`'s menu and "How To Choose"; link if the library was introduced earlier.
3. **Initialisation** — `index.html` → `main.tsx` mount → `App`; any URL/state hydration on load; the `#widget-ready` marker and why the render gate needs it.
4. **State and data flow** — the `useState` sources of truth, what's *derived* via `useMemo` (and why derive rather than store), how a user action propagates to a re-render.
5. **Domain logic** — the non-UI modules (parsing/evaluation/the algorithm), explained well enough to reimplement, and why they live outside the component.
6. **Effects and imperative integration** — `useEffect` mounting an imperative library into a ref'd host, dependency arrays, cleanup, and `ResizeObserver` for responsive sizing in the iframe; introduce once thoroughly, link thereafter.
7. **Putting it together** — a short "build it yourself" recap: the order to write the files, and the few decisions that matter.
8. **`## Concepts introduced`** — the techniques this tutorial taught for the first time; feeds the next tutorial's known-set.

## Output contract

```
/out/
  repo/         # full .git repo; host fetches the branch from here
  FINDINGS.md   # research background for the concepts taught
  OUTLINE.md    # the agreed section outline
  REVIEW.md     # verdict: PASS or CHANGES_NEEDED
  CHANGES.md    # branch, base commit, slug, track, tutorial path, sections, new concepts, summary
```

## Host-side landing + PR

The in-sandbox review is authoritative; the host does not re-run the pipeline. The landing is **checkout-free** — it only creates a branch ref and pushes, never switching the working tree — so it can't collide with another pipeline's in-flight work.

1. Read `/out/CHANGES.md` for the branch (`tutorial/frontend-<slug>`), base commit, and tutorial path.
2. Create the branch ref locally **without touching the working tree**: `git -C /home/jbeshir/code/beshir-widgets fetch <output_dir>/repo "+tutorial/frontend-<slug>:refs/heads/tutorial/frontend-<slug>"` (fallback: `<output_dir>/child/<name>/repo` if `<output_dir>/repo` is absent). The commit is already authored as the host identity, so there is no re-author step.
3. **Merged widget:** push the branch and open a PR — `git -C … push -u origin tutorial/frontend-<slug>`, then `mcp__github__create_pull_request` to `main` (body: which widget, the live URL, the concepts it newly introduces). **Open widget PR:** re-confirm the PR is open and unmerged, then fast-forward the tutorial commit onto the widget's branch so it joins the existing PR — `git -C … push origin "tutorial/frontend-<slug>:<headRef>"` (no local switch).
4. Give the user the PR URL.
