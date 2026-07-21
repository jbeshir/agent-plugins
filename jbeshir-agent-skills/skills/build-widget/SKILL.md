---
name: build-widget
description: Build new embeddable widgets in beshir-widgets through isolated research, implementation, offline validation, interaction capture, UX review, and pull-request delivery. Use when asked to create a new independently deployed visualization, simulation, explorable explanation, or web demo. Skip routine edits to existing widgets and non-widget work.
---

# Build a widget

Create one independent Preact + Vite application in the repository's widgets directory, verify it offline, review its rendered states, and land it through a branch and pull request.

Read these references before launching the pipeline:

- [widget-conventions.md](widget-conventions.md) for the widget, data, embedding, favicon, accessibility, and journey contracts.
- [pipeline.md](pipeline.md) for the ordered sandbox workflow, child contracts, and fail-closed gates.
- [landing.md](landing.md) for output promotion, branch freshness, push, and pull-request delivery.

## Preflight

1. Discover the `beshir-widgets` repository root from the current workspace or user-provided context. Do not assume a username, home directory, checkout location, or mounted basename.
2. Validate the expected repository authority documents, shared validation/render/journey scripts, and widgets directory before continuing.
3. Resolve the available capabilities at runtime:
   - isolated agent orchestration with shared workspace;
   - isolated research with open web access;
   - Node and browser script sandboxes;
   - shell and Git access;
   - pull-request creation through any available GitHub integration or CLI.
4. Stop before mutation if a required sandbox capability is unavailable. If pull-request creation alone is unavailable, allow the build to finish and provide an exact manual PR handoff instead.
5. Fetch the main branch from origin without switching the host working tree. Record the refreshed remote base commit and host Git author name/email.
6. Derive a lowercase hyphenated slug and reject it if the corresponding widget directory already exists at the refreshed base.

## Orchestrate

Launch one capable orchestrator with the discovered repository root mounted read-only. Do not select or require a named agent implementation, provider, credential type, or qualitative model. Use a user-requested model only when the live schema supports it; otherwise use the credential-aware default.

Require the orchestrator to:

1. Discover the single mounted repository directory at runtime and copy it, including Git metadata, to the standard writable Demesne workspace repository path.
2. Follow [pipeline.md](pipeline.md) in order and treat every validation, render, journey, UX, and final-review condition as fail-closed.
3. Apply [widget-conventions.md](widget-conventions.md) and the repository's own `LIBRARIES.md`, `DATA.md`, and `TESTING.md` as the authorities.
4. Capture every child call's terminal status and returned output directory. Validate and promote only artifacts from that exact invocation.
5. Branch from the recorded remote main base, author the commit with the recorded host identity, and produce the parent output packet defined in [landing.md](landing.md).
6. Print `DONE` only after every required parent artifact is validated and the final review verdict is `PASS`.

The orchestrator may delegate qualitative research, implementation, fixes, and reviews to any available compatible agent runtime. Prompts must describe capabilities and outputs, not a particular vendor, agent product, or model family. Use unique lowercase DNS-1123 child names whenever the nested tool contract requires names.

## Host landing

Treat the orchestrator's gates as authoritative; do not silently substitute host-side builds for missing evidence. Follow [landing.md](landing.md) checkout-free. Refresh the remote main ref immediately before checking ancestry. If the branch needs rebasing, perform the rebase in an isolated writable copy and rerun every gate affected by the new base before pushing.

Return the PR URL, future widget URL, gate summary, and representative gallery images. If PR tooling is unavailable, return the pushed branch plus an exact compare URL or manual PR command.
