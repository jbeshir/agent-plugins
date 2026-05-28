---
name: sandbox-product-research
description: Drive a product / competitive research pass through a demesne-orchestrated pipeline — an Opus orchestrator profiles the target, scans for comparables on the open web, fans out per-avenue subagents in parallel, then a compiler agent synthesises a prioritised executive report with per-avenue appendices identifying feature, documentation, and publishing gaps. Apply when the user wants to understand how a product compares to alternatives and what polished, complete, or competitive looks like for it. Triggers include "product research", "competitive research", "gap analysis", "identify our gaps vs competitors", "what's missing vs competitors", "what does a polished/published version of X look like", "competitive landscape research". Skip for code work (use sandbox-feature-work), internal-codebase audits (use sandbox-quality-improvement), and single open-web questions (call `sandbox_research` directly).
---

Drive a product / competitive research pass through a demesne pipeline instead of asking a single LLM. You (the host session) author one orchestrator prompt, launch a single Opus `sandbox_agent`, and that orchestrator fans out parallel isolated open-web subagents — one per research avenue — then a compiler agent synthesises a prioritised executive report with per-avenue appendices. The deliverable is markdown in `/out`; there is no code landing.

The shape is grounded in Anthropic's published multi-agent research patterns: lead-agent decomposition, parallel isolated subagents (no echo chamber), dedicated synthesis with citation discipline, explicit effort-scaling rules to bound cost.

## Pipeline shape

```
host (you)
 └─ Opus sandbox_agent  ── ORCHESTRATOR ─────────────────────────────┐
      1. INTAKE       profile the target → /workspace/target.md      │
                      (reads /in/<repo> if mounted; prose otherwise) │
      2. COMPARABLES  one sandbox_research child, open web           │
                      → /workspace/comparables.md (5–8 + rationale)  │
      3. DECOMPOSE    orchestrator writes /workspace/avenues.json    │
                      hybrid: per-dimension + a verification avenue  │
      4. AVENUES      N × sandbox_research IN PARALLEL (isolated)    │
                      → /out/child/<avenue>/finding.md + sources     │
      5. COMPILE      Opus sandbox_agent (compiler), reads all       │
                      → report.md + appendices/ in its /out          │
      6. DELIVER      orchestrator cp's compiler's output into its   │
                      OWN /out: report.md + appendices/ + metadata   │
                                                                      │
 └─ host reads /out/report.md ◀──────────────────────────────────────┘
      → research artefact; no code/branch landing
```

The orchestrator runs the inner pipeline autonomously. You write one good prompt, launch it, then read the report.

### Step roles

1. **INTAKE** — orchestrator writes a one-page target profile: what the product is, who uses it, what it competes on, the user's initial gap hypotheses. If `/in/<repo>` is mounted it reads the repo; otherwise it works from the prompt prose. Output: `/workspace/target.md`.

2. **COMPARABLES** — one `sandbox_research` child (isolated, open web, Sonnet default) sources 5–8 plausible comparables — adjacent categories, ecosystem peers, "what would a buyer compare us to" — with one paragraph of rationale each. Output: `/workspace/comparables.md`. The orchestrator may add/remove based on the user's prompt before proceeding.

3. **DECOMPOSE** — orchestrator writes `/workspace/avenues.json`. **Hybrid dimension-first rule:** one avenue per dimension covering all comparables — features, documentation, publishing, community/ecosystem, governance/sustainability — plus one **verification avenue** that cross-checks whether the comparables list missed obvious peers. Target 5–8 avenues; never more than 10. Each avenue: `{name, scope, comparables, dimension, research_question, tool_call_budget, output_format}`.

4. **AVENUES** — N × `sandbox_research` children dispatched **in parallel** (a single tool-call message containing all the spawns), each isolated with open web. Each subagent's prompt embeds the avenue object and requires: ≥3 *distinct* source types (GitHub, package registry, docs site, community forum, primary website, etc.), a bounded tool-call budget (10–25), and per-claim citation discipline — URL + access date + a quoted excerpt for every capability claim. Output: `/out/child/<avenue>/finding.md` (per-comparable table + gap summary).

5. **COMPILE** — one Opus `sandbox_agent` (the compiler) reads the target profile, comparables, and all avenue findings (via `/in/previous-jobs/<avenue>/finding.md`) and synthesises `report.md` plus `appendices/appendix-<n>-<avenue>.md` (each finding copied verbatim) into its `/out`. Every claim in the main report must cite the appendix it came from. Conflicting subagent findings are flagged (not silently resolved), with a recommended verification action.

6. **DELIVER** — the orchestrator copies the compiler's report and appendices into its OWN `/out` with plain `cp` (not via a child — a child's `/out` is `/out/child/<name>` and would strand the deliverable). It also writes `/out/metadata.json` (target, comparables list, avenue names, models used, run date).

## Writing the orchestrator prompt

Brief it as a complete document, not a one-liner:

1. **The target** — what the product is, the user it serves, the user's initial gap hypotheses, and any comparables the user wants treated as fixed. Mount `/in/<repo>` if there is one.
2. **The pipeline contract** — the six steps above; emphasise the parallel fan-out in step 4 (one tool-call message spawning all avenues simultaneously) and the deliver-via-own-`/out` step.
3. **The decomposition rule** — hybrid dimension-first with a verification avenue; 5–8 avenues, never >10. Embed the avenue-object schema.
4. **Frameworks to apply where relevant** (not mechanically):
   - **[Diátaxis](https://diataxis.fr/)** for documentation — tutorial / how-to / reference / explanation.
   - **Kano model** for feature classification — must-have / performance / delighter.
   - **Jobs-to-be-done** to broaden the competitor set ("any tool that accomplishes this job competes").
   - **[OSSF Scorecard](https://scorecard.dev/)**, **GitHub community standards**, **[opensource.guide](https://opensource.guide/)** for OSS publishing/community checklists.
5. **Prioritisation** — every recommendation in `report.md` must include an **ICE score** (Impact × Confidence × Ease, each 1–10), a one-line effort estimate, and a suggested owner role. Escalate to RICE if real usage data is available; use MoSCoW to categorise within ICE rankings.
6. **Subagent prompts** — the orchestrator generates each avenue subagent's prompt embedding the avenue object plus: the ≥3-distinct-source-types rule, the tool-call budget, the structured-output requirement, and an explicit "do not infer capabilities from general reputation; cite URL + quoted excerpt or omit the claim" instruction.
7. **Compiler prompt** — the compiler must produce the report in the five-section structure below. Forbid claims in the main report that aren't cited to an appendix.

## Output contract

```
/out/
  report.md                       # The deliverable
  appendices/
    appendix-a-<avenue-name>.md   # One per avenue
  metadata.json                   # target, comparables, avenues, models, run date
```

`report.md` sections in order: **TL;DR / Executive Summary** (≤500 words; the 3–5 prioritised recommendations as bullets; written last, placed first), **Methodology** (which comparables, why each, dimensions assessed, known limitations), **Cross-Cutting Findings** (patterns across ≥2 avenues; not a list of facts), **Prioritised Recommendations** (ICE-scored table + per-item narrative with owner + effort), **Appendix Index** (link to each appendix).

## Launching the orchestrator

- **`directories: ["<abs path>"]`** if the target is a repo on disk and the orchestrator should read it for intake. Optional.
- **Opus** for the orchestrator and compiler. Subagents default to Sonnet.
- The orchestrator spawns the avenue subagents in **one tool-call message** (parallel fan-out) — say this explicitly in the prompt; the default failure mode is sequential dispatch.
- Cost: rough range $5–20 for a 5–8 avenue pass — multi-agent systems use ~15× the tokens of a single-turn call, but the parallelism cuts wall-clock time accordingly. Only justified when the task value exceeds that.

## Constraints and known pitfalls

Paid-for lessons from Anthropic's multi-agent research write-up and standard product-research practice:

- **Don't over-spawn.** Effort scaling: 1 subagent for fact-finding, 2–4 for direct comparisons, 5–10 for a full landscape. Early prototypes that skipped this spawned 50 subagents for trivial queries. Cap at 10; typical operating range is 5–7.
- **Avoid echo-chamber sourcing.** Multiple subagents querying the same terms converge on the same corpus. Every avenue prompt must require ≥3 *distinct* source types. The compiler should flag if >50% of sources across appendices come from one domain.
- **Don't accept hallucinated capability claims.** Every claim about a comparable's feature / docs / release engineering / etc. must cite a specific URL + quoted excerpt. The compiler **flags** uncited claims rather than promoting them to recommendations. "I think Competitor X has good docs" is not a finding.
- **Don't treat the step-2 comparable list as final.** Include the verification avenue, and document in the methodology section which comparables were excluded and why — a reader needs to see the analysis boundary.
- **Don't ship recommendations without owner + effort + score.** A report listing 20 gaps with no prioritisation or ownership is a document, not a decision. Every recommendation needs an ICE score, a rough effort estimate, and a suggested owner role; without them the report sits unread.
- **Sourcing freshness matters.** For fast-moving categories (MCP ecosystem, LLM tooling, the AI dev-tools space generally) require subagents to note the publication date of each source and flag anything older than ~12 months.

## Host-side landing

There is no code landing — the deliverable is `/out/report.md`. Read it, share it, decide on the recommendations. Optionally copy `/out/` into the repo or a docs store for permanent reference. No `make validate`, no branch fetch, no commit — this skill produces a research artefact, not a code change.
