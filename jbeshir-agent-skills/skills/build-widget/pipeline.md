# Pipeline

Run these stages in order in one writable orchestrator workspace. Use deterministic script sandboxes for commands and compatible qualitative agents for research, implementation, fixes, and review. Do not require a named agent product, provider, or model.

## 1. Research

Inspect representative siblings and the three repository authority documents. Give an isolated open-web research child only the necessary repository shape and concept. Require `FINDINGS.md` containing:

- algorithm and assumptions;
- validation examples;
- data-source selection criteria, URLs, and licenses when applicable;
- offline sample data or a reason none is needed;
- a fetch/normalize recipe for prebake mode.

After terminal success, use the child's returned output directory to validate and promote `FINDINGS.md` into the parent output and `/workspace`.

## 2. Plan

Write `/workspace/PLAN.md`. Fix the slug, template-or-scratch choice, stack, data mode, required states, journey matrix, accessibility constraints, and acceptance criteria. Document every permitted exception from the defaults.

## 3. Implement

Run sequential implementation phases against `/workspace/repo`. Use unique child names. Require each child to write a uniquely named summary, such as `phase01-summary.md`. After terminal success, promote each summary from that child's returned output directory into `/workspace/phase-summaries/`.

Implement the complete contract in [widget-conventions.md](widget-conventions.md). Never let parallel phases mutate the shared repository.

## 4. Validate

Run a Node script child with package-manager egress. In the widget directory:

1. Run the repository's checked-in favicon generation command.
2. Run `npm install`.
3. Run `npm run build`.
4. At repository root, run `node scripts/validate-widgets.mjs`.

Require exit code zero. Validate the expected lockfile, `dist/`, and favicon outputs. On failure, run one fix phase and repeat the entire validation stage, up to three attempts. If the third attempt fails, write failed-gate evidence and stop without creating a landable commit or repository output.

## 5. First paint

Run a browser script child with no egress:

```sh
node /workspace/repo/scripts/render.cjs /workspace/repo/widgets/<slug>/dist/index.html widget-ready /out
```

Require exit code zero, `render-ok`, and a nonblank `screenshot.png` in that invocation's returned output directory. Promote validated files into `/workspace/render/` and parent `/out/render/`. Fix, revalidate, and rerender up to three attempts. Fail closed after the third failure.

## 6. Journey capture

For interactive widgets, run a browser script child with no egress:

```sh
node /workspace/repo/scripts/journey.cjs /workspace/repo/widgets/<slug>/dist/index.html /workspace/repo/widgets/<slug>/journey.json /out
```

Require exit code zero, `journey-ok`, parseable `journey-report.json`, and the complete expected PNG matrix in the child's returned output directory. Validate that reports contain no console/page errors, missed markers, or horizontal overflow. Promote exactly those artifacts into `/workspace/repo/.journey-out/`, `/workspace/gallery/`, and parent `/out/gallery/`.

A purely static widget may use one state or omit this stage only when `PLAN.md` records why no interaction exists. Fix, revalidate, and recapture up to three attempts. Fail closed after the third failure.

## 7. UX review

Dispatch five qualitative reviewers concurrently after the final gallery is promoted. Give each the validated gallery and journey report and one lens:

- visual design: typography, spacing, hierarchy, polish, clipping, and layer order;
- responsive: cross-viewport overflow, cramping, touch targets, text floors, and layer order;
- theme: computed contrast and theme-specific defects;
- interaction flow: discoverability, feedback, and intentional empty/error states;
- accessibility: accessibility tree, names, focus, contrast, and control semantics.

Require structured findings with severity `blocking`, `should`, or `nice`, plus state, viewport, scheme, evidence, issue, and repair. Validate every reviewer status and report. Retry an infrastructure failure once; treat missing coverage as a failed gate rather than a clean review.

Synthesize and deduplicate findings into `UX-REVIEW.md`. Fix all `blocking` and `should` findings, then rerun validation, journey capture, and affected reviews. Cap the loop at three rounds. Proceed only when the final synthesis contains no `blocking` or `should` findings; otherwise stop without a landable commit or repository output.

## 8. Final review and commit

Run a final qualitative review of the diff, requested concept, repository conventions, validated gallery, and gate records. Require the report to end with exactly `PASS` or `CHANGES_NEEDED`.

For `CHANGES_NEEDED`, fix and rerun affected gates and review, up to three rounds. Proceed only with `PASS`. If the final verdict is not `PASS`, stop without committing or producing `/out/repo`.

With all gates clean, create `widget/<slug>` from the recorded `origin/main`, set the recorded host author identity, and create one intentional commit. The orchestrator—not a child—copies `/workspace/repo` to parent `/out/repo` and completes the output packet in [landing.md](landing.md).
