# Landing and output contract

## Parent output promotion

Treat each child output directory returned by the live tool call as unique. After terminal success:

1. Resolve the exact returned directory.
2. Validate all declared files, statuses, formats, and expected matrix cells there.
3. Copy the validated artifacts into the orchestrator's parent `/out` explicitly.
4. Record source directory, child name, terminal status, and digests in `GATES.md`.

Never read a root-looking child `/out` path as though it were the parent's `/out`. Never reuse a previous invocation's artifacts. Aggregate phase summaries deterministically, in phase order, into parent `/out/SUMMARY.md` while preserving uniquely named raw summaries under `/out/phase-summaries/`.

Before printing `DONE`, validate this packet:

```text
/out/
  repo/
  gallery/
  render/
  FINDINGS.md
  PLAN.md
  SUMMARY.md
  phase-summaries/
  UX-REVIEW.md
  REVIEW.md
  GATES.md
  CHANGES.md
```

Require `/out/repo` to contain `.git`, branch `widget/<slug>`, and exactly the intended commit above the recorded base. Require `REVIEW.md` to end in `PASS`, `GATES.md` to mark every required gate successful, and `CHANGES.md` to record branch, base commit, slug, hostname, stack, data mode, journey cells, UX rounds, fixes, and gallery path.

## Checkout-free host landing

1. Read and validate `CHANGES.md`, `GATES.md`, and `REVIEW.md`.
2. Fetch the sandbox branch into `refs/heads/widget/<slug>` without switching the host working tree.
3. Immediately run a checkout-free `git fetch origin main` and resolve the refreshed `origin/main`.
4. Require refreshed `origin/main` to be an ancestor of `widget/<slug>`.
5. If it is not, rebase in an isolated writable repository copy, rerun validation, render, journey, UX, and final review against the new base, then replace the branch only with the newly validated result.
6. Push `widget/<slug>` to origin.
7. Create a PR from `widget/<slug>` to `main` using any available authenticated GitHub capability. Summarize the widget, stack, data mode, gates, and future URL. If no PR-creation capability exists, return the pushed branch, compare URL, title, and prepared body for manual creation.

Do not commit directly to the default branch. Return the PR or manual-handoff URL, future widget URL, gate summary, and representative gallery images.
