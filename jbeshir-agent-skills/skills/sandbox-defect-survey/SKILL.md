---
name: sandbox-defect-survey
description: Survey a codebase for the kinds of defects common to vibecoded / AI-generated projects through a demesne-orchestrated pipeline — an Opus orchestrator runs an open-web research child that distils a sourced taxonomy of ~10 common AI-code flaw types, fans out one detection subagent per type that hunts evidence-backed instances in the repo (file:line, confirmed vs suspected), and synthesises per-type reports plus a prioritised executive summary carrying an improvement plan for each flaw type. Report-only — it surveys and proposes, it does not fix or land code. Apply when the user wants to know what categories of latent problems a codebase has, grounded in real commentary on how AI-generated code tends to fail — "defect survey", "research common flaws and find them in our code", "what's wrong with this vibecoded project", "audit for AI-code antipatterns", "what kinds of problems does this codebase have", "find the common AI-slop flaws in here". Skip for fixing-and-landing quality work (use sandbox-quality-improvement), building features (use sandbox-feature-work), and external product / competitive gap analysis (use sandbox-product-research).
---

Survey a codebase for the failure modes common to vibecoded / AI-generated code, grounded in what people actually write about those failures. You (the host session) author one orchestrator prompt, launch a single Opus `sandbox_agent`, and that orchestrator first researches a sourced taxonomy of ~10 flaw *types* on the open web, then fans out one detection subagent per type to hunt real instances in the repo, then synthesises per-type reports plus an executive summary. The deliverable is markdown in `/out`; there is **no code landing** — this skill surveys and proposes, it does not fix.

This is the report-only sibling of `sandbox-quality-improvement`. The difference is load-bearing: that skill audits along a **fixed** internal dimension list and runs an audit→fix→land loop; this one **derives its lenses from fresh external research** (so the catalogue tracks evolving commentary on how AI code fails) and stops at a report. Re-researching the taxonomy every run is the point — don't hardcode a flaw list. To act on what it finds, follow up with `sandbox-quality-improvement` (behaviour-preserving fixes) or `sandbox-feature-work` (a specific change).

The shape is grounded in empirical studies of AI-generated-code defects (Copilot/Codex security analyses, the ASPLOS Go-concurrency study, duplication corpora, slopsquatting reports) — the research child sources these live rather than relying on the model's priors.

## Pipeline shape

```
host (you)
 └─ Opus sandbox_agent  ── ORCHESTRATOR ─────────────────────────────┐
      reads /in/<repo> (read-only; detection children inherit it)     │
      1. RESEARCH   one sandbox_research child, open web, isolated     │
                    → sourced taxonomy of ~10 flaw TYPES              │
                      (/out/child/research01/TAXONOMY.md)             │
      2. FINALISE   orchestrator keeps ~10, drops/merges N/A types     │
                    → /workspace + /out/TAXONOMY.md (the detect spec)  │
      3. DETECT     N × sandbox_agent, one per flaw type, batches ≤4   │
                    each READS the repo for its one lens →             │
                    /out/child/detect-<slug>/REPORT.md                │
      4. COLLATE    orchestrator cp's each → /out/reports/NN-<slug>.md │
                    + writes /out/EXECUTIVE_SUMMARY.md                 │
                                                                       │
 └─ host reads /out/EXECUTIVE_SUMMARY.md + reports ◀───────────────────┘
      → survey artefact; verify the load-bearing findings; no landing
```

The orchestrator runs the inner pipeline autonomously. You write one good prompt, launch it, then read the reports.

### Step roles

1. **RESEARCH** — one `sandbox_research` child (`name=research01`, Sonnet, open web, **isolated** — no repo access, which is correct: it only needs the web). It surveys real practitioner and academic writing on common flaws in AI-generated codebases and distils a taxonomy of about **10 distinct flaw types** biased toward the target's domain (a Go backend, a CLI, a web app — say which in the prompt so it doesn't return frontend flaws for a server). For each type it writes: a short name, a 2–4 sentence description of the flaw and why AI code tends to exhibit it, concrete **detection signals** (what to grep/read for), and **≥1 cited source** (title + URL). Aim for breadth — distinct categories, not 10 variants of one. Output: `/out/child/research01/TAXONOMY.md`.

2. **FINALISE TAXONOMY** — the orchestrator reads the research output, keeps ~10 types, drops or merges any wholly inapplicable to the target (noting what it dropped and why), and writes the consolidated, numbered list to `/out/TAXONOMY.md` (each entry: name + description + detection signals + source). This is the spec the detection children work from.

3. **DETECT** — one `sandbox_agent` per finalised flaw type (Sonnet; `name=detect-<slug>`). They are independent and read-only, so run them **in batches of at most 4 concurrent** (not all 10 at once — resource pressure). Each child gets the repo map and its one flaw type, and must: **investigate by reading the code, not grep-then-guess** (a grep hit is a lead, not a finding); report only evidence-backed instances, each with **file:line**, a short excerpt, why it's an instance, and **severity**; mark **confirmed vs suspected**; say **"clean on this axis"** plainly when the code is clean (no padding); and write an **improvement plan** tied to its findings. Output: `/out/child/detect-<slug>/REPORT.md`, structured `# <type>` → `## Summary` (N confirmed / M suspected / top severity) → `## Findings` → `## Improvement plan`.

4. **COLLATE** — the orchestrator copies each detection report into `/out/reports/<NN>-<slug>.md` with plain `cp` (**itself**, not via a child — a child's `/out` is `/out/child/<name>` and would strand the file), then writes `/out/EXECUTIVE_SUMMARY.md`: method + the types investigated (and any dropped, with why); a findings **table** (type | confirmed | suspected | top severity | one-line headline); **cross-cutting themes** (patterns spanning ≥2 types); a candid **codebase-health read** (where it's strong, where the real weaknesses are — fair, acknowledging any prior cleanup); and a **prioritised remediation list**, each item pointing at the report that details it. Then prints `DONE`.

## Writing the orchestrator prompt

Brief it as a complete document, not a one-liner:

1. **The target and its domain** — what the codebase is and what kind of system (so the research biases the taxonomy correctly), plus a **repo map** (key dirs/packages) so detection children don't burn budget rediscovering structure.
2. **The pipeline contract** — the four steps above, the child-naming rule, that research is isolated, and the batches-of-≤4 detection rule.
3. **The taxonomy bar** — ~10 distinct types, grounded in **cited** commentary (not the model's priors), domain-appropriate; drop/merge inapplicable ones.
4. **Detection discipline** — read-not-grep-guess; evidence with file:line; confirmed vs suspected; **"clean on this axis" is a valid, valuable result**; **no manufactured findings**; an improvement plan per type tied to the specific findings.
5. **Prior-state note** — if the repo has already had quality passes, say so: agents should expect (and honestly report) few or zero instances on some axes. A near-empty report on a hardened axis is signal, not failure.
6. **Output contract** — the files below; report-only, no edits/builds/commits/branch.

## Output contract

```
/out/
  EXECUTIVE_SUMMARY.md            # The headline deliverable
  TAXONOMY.md                     # The ~10 finalised flaw types + sources
  reports/
    NN-<slug>.md                  # One detailed report per flaw type
```

## Launching the orchestrator

- **`directories: ["<abs path to repo>"]` is mandatory** — the detection children inherit this mount and read the repo from `/in/<repo>`. Forgetting it leaves them with nothing to survey.
- **Opus** for the orchestrator; **Sonnet** for the research child and the detection children.
- Detection children run **in batches of ≤4 concurrent**; say this explicitly — the default failure mode is spawning all ten at once.
- Cost: rough range **$10–20** for a ~10-type survey (one research child + ~10 detection children + synthesis). It is the most expensive of the sandbox skills; only justified when a broad survey is wanted rather than a targeted look.

## Constraints and known pitfalls

- **Report-only — no landing.** No edits, no fix loop, no `make validate`, no commit, no branch. The deliverable is the reports. Acting on them is a separate, explicit step (`sandbox-quality-improvement` to fix-and-land; `sandbox-feature-work` for a specific change).
- **Verify the load-bearing findings on the host before they drive changes.** Detection children are Sonnet; a confirmed high-severity, security, or concurrency finding should be re-read against the actual code by the host session before it becomes a fix — false positives happen, and a quick read of the cited file:line settles it. This host-side verification is part of using the skill well, not optional polish.
- **No manufactured findings.** A hardened codebase should produce small or empty lists on several axes. Forbid padding; "what I checked and found clean" is the correct output there, and the empty lists are themselves evidence of prior cleanup working.
- **Re-research the taxonomy every run.** Don't let the orchestrator reuse a canned flaw list — the freshly sourced taxonomy is what differentiates this from a fixed-dimension audit and keeps it current with evolving commentary.
- **Read-not-grep-guess in detection.** A grep/signal hit locates a candidate; the finding requires reading the surrounding code to confirm it. Enforce this in the child prompts.
- **Don't over-spawn.** ~10 types is the target; one detection agent per type. More types means more cost for diminishing distinctness — cap around 10–12.
- **Collate via the orchestrator's own `/out`.** The per-type reports and summary must be copied into the orchestrator's `/out` by the orchestrator itself; a `sandbox_script` child's `/out` is `/out/child/<name>` and would strand them.
- **Scripts and data work run in demesne, never on the host.** The host's only role is reading the returned `/out`, verifying the top findings, and deciding what to route into a fix pipeline.
