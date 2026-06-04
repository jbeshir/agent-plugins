---
name: sandbox-docs-quality
description: Audit a project's documentation against the code as ground truth through a demesne-orchestrated pipeline — an Opus orchestrator fans out four fixed lenses (completeness vs code, accuracy vs code, placement / information architecture, cross-doc consistency), each cross-referencing the docs against the source, and synthesises per-lens reports plus an executive summary carrying an information-architecture recommendation. Catches docs that understate a capability the code supports, contradict the code, bury operational requirements in the wrong place, or state facts inconsistently across pages. Report-only — it surveys and proposes; it does not rewrite or land changes. Apply when the user wants to know whether the docs correctly, completely, and coherently describe what the code actually does — "docs quality pass", "audit the docs against the code", "are the docs complete and accurate", "do the docs describe the full capability", "find docs that understate / misplace / contradict the code", "check documentation completeness and information architecture". Skip for writing-quality / AI-slop in prose (use sandbox-prose-defect-survey), regenerating a README / diagrams / OpenAPI to match the code (use uplift-docs), code-logic flaws (use sandbox-code-defect-survey), and fixing-and-landing (this pass is report-only).
---

Audit whether a project's documentation **correctly, completely, and coherently describes what the code actually does**, by cross-referencing every doc claim against the source. You (the host session) author one orchestrator prompt, launch a single Opus `sandbox_agent`, and that orchestrator fans out four fixed lenses over the docs + code, then synthesises per-lens reports plus an executive summary. The deliverable is markdown in `/out`; there is **no landing** — this pass surveys and proposes, it does not rewrite.

This is the *code-grounded correctness* member of the docs survey family, distinct from its siblings:
- vs **`sandbox-prose-defect-survey`** — that judges how the text is *written* (slop tells, verbosity, terminology drift); this judges whether the text is *true of, and complete against, the code*. Run both for full docs coverage.
- vs **`uplift-docs`** — that *regenerates* a README / diagrams / OpenAPI to match the code (a production task); this *audits* the existing docs for gaps (report-only).
- vs **`sandbox-code-defect-survey`** — that audits code logic; its docs/code-alignment lens is a side-check, whereas this is a dedicated, thorough docs audit.

Unlike the defect surveys, this pass needs **no web research** — the lenses are fixed and the ground truth is the repo itself. The whole pass is one fan-out plus a synthesis.

## Pipeline shape

```
host (you)
 └─ Opus sandbox_agent  ── ORCHESTRATOR ─────────────────────────────┐
      reads /in/<repo> (read-only; lens children inherit it)          │
      1. FAN OUT    4 × sandbox_agent, one per lens, concurrent —     │
                    each cross-references DOCS vs CODE ground truth   │
                    → /out/child/detect-<lens>/REPORT.md              │
      2. COLLATE    orchestrator cp's each → /out/reports/NN-<lens>.md│
                    + writes /out/EXECUTIVE_SUMMARY.md (with an IA    │
                    recommendation)                                    │
                                                                       │
 └─ host reads /out/EXECUTIVE_SUMMARY.md + reports ◀───────────────────┘
      → audit artefact; verify the load-bearing findings; no landing
```

There is no research phase — the orchestrator goes straight to the four lenses.

### The four lenses (one `sandbox_agent` each, Sonnet, run concurrently)

Each child reads BOTH the docs AND the relevant code, and reports findings with **file:line for both the doc and the code ground-truth it checked against**, marking confirmed vs suspected. "Clean on this axis" is a valid result — no manufactured findings. Each writes `/out/REPORT.md`: `# <lens>` → `## Summary` → `## Findings` (each: doc location + code ground-truth + the gap + severity + proposed fix) → `## Improvement plan`.

1. **completeness-vs-code** — does every doc reflect the FULL current capability per the code? All tool params documented? Every supported provider/agent/model/mode/option named (the classic miss: a tool described as supporting one backend when the code registers two)? All env vars / config / output fields covered? Flag understated, narrowed, or partial descriptions; propose the completing text.
2. **accuracy-vs-code** — is what IS documented correct? Wrong defaults, stale counts, removed/renamed features still described, behaviours that changed, example output that no longer matches. Cite doc line + code line. (Pure doc/code contradiction.)
3. **placement / information architecture** — is each fact in the right doc TYPE and discoverable (Diátaxis: tutorial = first-run learning; how-to = task; reference = lookup; explanation = concepts)? Operational/host REQUIREMENTS buried in a tutorial, reference detail in a how-to, requirements not discoverable from where an operator looks. **Explicitly assess where operational requirements / prerequisites / config live and whether that's coherent; propose a concrete home.**
4. **consistency** — is the same fact / term / requirement stated consistently across docs and cross-referenced everywhere relevant (not single-homed when it should be linked from several places, nor duplicated with divergent wording)? Flag divergences and orphaned-but-should-be-linked facts.

## Writing the orchestrator prompt

Brief it as a complete document:

1. **The target and its docs** — what the project is, the doc surfaces (the Diátaxis tree, READMEs, `manifest.json`/tool metadata, schemas), AND the **code ground-truth map**: where the authoritative facts live — tool registration + descriptions + output formatting, the supported providers/agents/models, env/config loader, egress/mode enums, image allowlist. The lenses cross-reference docs *against these*, so name them.
2. **The four lenses** above, each cross-referencing docs vs code, with **file:line for both sides**.
3. **The two canonical archetypes to catch** (seed them so nothing's missed, then find similar): (a) *capability narrowing* — a tool/feature described as a subset of what the code supports; (b) *operational-requirement misplacement* — a host prerequisite buried in one tutorial step and not discoverable.
4. **Detection discipline** — read-not-skim; cite both doc and code; confirmed vs suspected; "clean on this axis" is valid; no manufactured findings.
5. **The synthesis must include an information-architecture recommendation** — propose where operational requirements / configuration should consistently live (e.g. a dedicated reference page) and how the tutorial/README/how-tos should point to it, so the class of placement issue doesn't recur.
6. **Output contract** — report-only, no edits.

## Output contract

```
/out/
  EXECUTIVE_SUMMARY.md            # findings table + the two archetypes confirmed + IA recommendation + prioritised fix list
  reports/
    NN-<lens>.md                  # one per lens (completeness, accuracy, placement, consistency)
```

The executive summary should carry: a findings table (lens | confirmed | suspected | top severity | headline); the canonical archetypes confirmed with their full extent; cross-cutting themes; a concrete **information-architecture recommendation**; and a prioritised fix list pointing at the reports.

## Launching the orchestrator

- **`directories: ["<abs path to repo>"]` is mandatory** — the lens children inherit this mount and read both docs and code from `/in/<repo>`.
- **Opus** orchestrator; **Sonnet** lens children. Only four — run them concurrently (no batching needed).
- Cost: rough range **$5–10** — cheaper than the research-grounded surveys (no research child; four lenses + synthesis).

## Constraints and known pitfalls

- **Report-only — no landing.** No edits, no commit, no branch. To act on the findings, follow up: a fix pipeline (sandbox-feature-work / sandbox-quality-improvement) or hand-edit; the IA restructure is usually the highest-leverage follow-up.
- **Cross-reference the CODE, not just the docs against each other.** The lens's value is checking doc claims against ground truth — the orchestrator prompt must point the children at the authoritative code surfaces, or they'll only catch internal inconsistency and miss capability/accuracy gaps.
- **Verify the load-bearing findings on the host before they drive changes.** Lens children are Sonnet; a confirmed high-severity finding (especially one in a *live* wire surface like a served tool description, or one that becomes a code edit) should be re-read against the cited file:line by the host session before it's fixed.
- **The information-architecture recommendation is a required output, not optional polish.** The most valuable result is usually "operational requirements have no coherent home → create one and point everything at it," and that only lands if the synthesis is told to produce it.
- **No manufactured findings.** Well-maintained docs should produce small lists on several lenses; "checked X against Y, consistent" is the correct output there.
- **Scripts and data work run in demesne, never on the host.** The host's only role is reading the returned `/out`, verifying the top findings, and deciding what to route into a fix pass.
