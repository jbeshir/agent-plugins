---
name: sandbox-prose-defect-survey
description: Survey a project's documentation, READMEs, code comments, and generated text for the flaw types common to AI-generated prose through a demesne-orchestrated pipeline — an Opus orchestrator runs an open-web research child that distils a sourced taxonomy of ~10 common AI-text flaw types (unsupported or hallucinated claims, verbosity and filler, terminology drift, structural bloat, baseline padding, example rot, broken cross-references, marketing fluff), fans out one detection subagent per type that hunts evidence-backed instances across every prose surface (file:line, confirmed vs suspected), and synthesises per-type reports plus a prioritised executive summary carrying an improvement plan for each flaw type. Report-only — it surveys and proposes, it does not rewrite or land changes. Apply when the user wants to know what categories of latent problems the project's writing has, grounded in real commentary on how AI-generated documentation and prose tend to fail — "prose defect survey", "docs defect survey", "find the AI-slop in our docs", "audit our documentation for AI-text tells", "what's wrong with this project's writing", "survey the docs and comments for common flaws". Skip for code-logic flaws (use sandbox-code-defect-survey), refreshing a README / diagrams / OpenAPI to match the current code (use uplift-docs), and external product / competitive gap analysis (use sandbox-product-research).
---

Survey a project's *prose* — documentation, READMEs, examples, code comments, and the text the code generates — for the failure modes common to AI-generated writing, grounded in what people actually write about those failures. You (the host session) author one orchestrator prompt, launch a single Opus `sandbox_agent`, and that orchestrator first researches a sourced taxonomy of ~10 prose-flaw *types* on the open web, then fans out one detection subagent per type to hunt real instances across the project's text surfaces, then synthesises per-type reports plus an executive summary. The deliverable is markdown in `/out`; there is **no landing** — this skill surveys and proposes, it does not rewrite.

This is the prose twin of `sandbox-code-defect-survey`: identical pipeline, different surface. Keep the split clean — **factual docs/code alignment belongs to the code survey** (catching a claim that contradicts the code takes reading the code to verify). This survey owns how the text is *written*: slop tells, verbosity, terminology drift, structure, example rot, padding, dead cross-references. When a detection child finds a claim that is outright *false against the code*, it should note it and hand it off to the code survey's docs/code-alignment lens rather than adjudicate it here. It also differs from `uplift-docs`, which *regenerates* a README / diagrams / OpenAPI to match the code — a production task, not a flaw survey.

The shape is grounded in critical writing on AI-generated text and "AI slop" (LLM writing-tell catalogues, documentation-quality studies, style-guide critiques of generated docs) — the research child sources these live rather than relying on the model's priors.

## Pipeline shape

```
host (you)
 └─ Opus sandbox_agent  ── ORCHESTRATOR ─────────────────────────────┐
      reads /in/<repo> (read-only; detection children inherit it)     │
      1. RESEARCH   one sandbox_research child, open web, isolated     │
                    → sourced taxonomy of ~10 PROSE flaw TYPES        │
                      (/out/child/research01/TAXONOMY.md)             │
      2. FINALISE   orchestrator keeps ~10, drops/merges N/A types     │
                    → /workspace + /out/TAXONOMY.md (the detect spec)  │
      3. DETECT     N × sandbox_agent, one per flaw type, batches ≤4   │
                    each READS the project's prose surfaces for its    │
                    one lens → /out/child/detect-<slug>/REPORT.md      │
      4. COLLATE    orchestrator cp's each → /out/reports/NN-<slug>.md │
                    + writes /out/EXECUTIVE_SUMMARY.md                 │
                                                                       │
 └─ host reads /out/EXECUTIVE_SUMMARY.md + reports ◀───────────────────┘
      → survey artefact; verify the load-bearing findings; no landing
```

The orchestrator runs the inner pipeline autonomously. You write one good prompt, launch it, then read the reports.

### Step roles

1. **RESEARCH** — one `sandbox_research` child (`name=research01`, Sonnet, open web, **isolated** — no repo access, which is correct: it only needs the web). It surveys real practitioner and critical writing on common flaws in AI-generated documentation and prose and distils a taxonomy of about **10 distinct flaw types**. For each: a short name, a 2–4 sentence description of the flaw and why AI-written prose tends to exhibit it, concrete **detection signals** (the phrasings, structures, and tells to read for), and **≥1 cited source**. Aim for breadth — distinct categories, not 10 shades of "too wordy". Output: `/out/child/research01/TAXONOMY.md`.

2. **FINALISE TAXONOMY** — the orchestrator reads the research output, keeps ~10 types, drops or merges any wholly inapplicable to this project's docs (noting what it dropped and why), and writes the consolidated, numbered list to `/out/TAXONOMY.md` (each entry: name + description + detection signals + source). This is the spec the detection children work from.

3. **DETECT** — one `sandbox_agent` per finalised flaw type (Sonnet; `name=detect-<slug>`). They are independent and read-only, so run them **in batches of at most 4 concurrent**. Each child gets the **prose-surface map** (below) and its one flaw type, and must: **read the text, not skim it**; report only evidence-backed instances, each with **file:line**, a **short verbatim quote** of the offending prose, why it's an instance, and **severity**; mark **confirmed vs suspected**; say **"clean on this axis"** plainly when the writing is fine (no padding); and write an **improvement plan** tied to its findings (usually rewrite / cut / consolidate / cross-link). Output: `/out/child/detect-<slug>/REPORT.md`, structured `# <type>` → `## Summary` (N confirmed / M suspected / top severity) → `## Findings` → `## Improvement plan`.

   **Prose surfaces to cover** (name them in the child prompts so nothing is missed): the docs tree (`*.md`, tutorials, how-tos, reference, explanation), all `README`s including `examples/`, **source-code comments and doc-comments**, and **the text the code generates** — help output, error messages, and any builder that emits prose (e.g. a generated `CLAUDE.md` / agent-context file). For generated text, the flaw lives in the *generator/template*, so the finding must point at the producing code, not just a sample of its output.

4. **COLLATE** — the orchestrator copies each detection report into `/out/reports/<NN>-<slug>.md` with plain `cp` (**itself**, not via a child — a child's `/out` is `/out/child/<name>` and would strand the file), then writes `/out/EXECUTIVE_SUMMARY.md`: method + the types investigated (and any dropped, with why); a findings **table** (type | confirmed | suspected | top severity | one-line headline); **cross-cutting themes** (tells that recur across surfaces); a candid **prose-health read** (where the writing is strong, where the real weaknesses are — fair, acknowledging any prior docs work); and a **prioritised remediation list**, each item pointing at the report that details it. Then prints `DONE`.

## Writing the orchestrator prompt

Brief it as a complete document, not a one-liner:

1. **The target and its docs** — what the project is and the shape of its documentation (Diátaxis tree? a single README? generated help?), plus the **prose-surface map** above so detection children know every place text lives.
2. **The pipeline contract** — the four steps, the child-naming rule, that research is isolated, and the batches-of-≤4 detection rule.
3. **The taxonomy bar** — ~10 distinct prose-flaw types, grounded in **cited** commentary (not the model's priors); drop/merge inapplicable ones.
4. **Detection discipline** — read-don't-skim; quote the offending text with file:line; confirmed vs suspected; **"clean on this axis" is a valid, valuable result**; **no manufactured findings**; an improvement plan per type tied to the specific quotes.
5. **The boundary with the code survey** — a claim that is *false against the code* is a docs/code-alignment finding owned by `sandbox-code-defect-survey`; detection children should flag-and-hand-off such cases, not adjudicate factual accuracy here. This survey judges the *writing*, not its truth against the implementation.
6. **Prior-state note** — if the docs have had prior simplification or review passes, say so: agents should expect (and honestly report) few or zero instances on some axes.
7. **Output contract** — the files below; report-only, no edits/rewrites/commits/branch.

## Output contract

```
/out/
  EXECUTIVE_SUMMARY.md            # The headline deliverable
  TAXONOMY.md                     # The ~10 finalised prose-flaw types + sources
  reports/
    NN-<slug>.md                  # One detailed report per flaw type
```

## Launching the orchestrator

- **`directories: ["<abs path to repo>"]` is mandatory** — the detection children inherit this mount and read the project's text from `/in/<repo>`. Forgetting it leaves them with nothing to survey.
- **Opus** for the orchestrator; **Sonnet** for the research child and the detection children.
- Detection children run **in batches of ≤4 concurrent**; say this explicitly — the default failure mode is spawning all ten at once.
- Cost: rough range **$8–18** for a ~10-type survey (one research child + ~10 detection children + synthesis). Prose surfaces are usually smaller than the codebase, so this tends to run a little cheaper than the code survey.

## Constraints and known pitfalls

- **Report-only — no landing.** No edits, no rewrites, no commit, no branch. The deliverable is the reports. Acting on them is a separate, explicit step (rewrite by hand, or route specific changes through `sandbox-feature-work`; regenerate stale READMEs/diagrams with `uplift-docs`).
- **Don't poach the code survey's lane.** Factual docs/code contradictions belong to `sandbox-code-defect-survey`. Here, flag-and-hand-off; judge the writing, not its truth against the code.
- **Verify the load-bearing findings on the host before they drive rewrites.** Detection children are Sonnet, and prose judgement is more subjective than code; a confirmed high-severity finding should be re-read in context by the host session before it becomes a rewrite — what reads as "filler" to one lens may be load-bearing nuance.
- **No manufactured findings.** Well-edited docs should produce small or empty lists on several axes. Forbid padding; "what I checked and found clean" is the correct output there.
- **Re-research the taxonomy every run.** Don't let the orchestrator reuse a canned tell-list — the freshly sourced taxonomy keeps it current with evolving critique of AI-written prose.
- **Quote, don't paraphrase.** Every finding carries a short verbatim quote + file:line. A paraphrased "this section is wordy" with no quote is opinion, not a finding, and lets the loop churn cosmetically.
- **Cover the generated text, not just the static docs.** The biggest blind spot is prose emitted by code (help, errors, generated context files); a finding there points at the generator. Name these surfaces explicitly or they get skipped.
- **Don't over-spawn.** ~10 types is the target; one detection agent per type; cap around 10–12.
- **Collate via the orchestrator's own `/out`.** The per-type reports and summary must be copied into the orchestrator's `/out` by the orchestrator itself; a `sandbox_script` child's `/out` is `/out/child/<name>` and would strand them.
- **Scripts and data work run in demesne, never on the host.** The host's only role is reading the returned `/out`, verifying the top findings, and deciding what to rewrite.
