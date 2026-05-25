---
name: sandbox-quality-improvement
description: Bring an existing codebase that has been through prior rounds of work up to good standing via a demesne-orchestrated audit → fix → re-review loop. An Opus orchestrator runs the deterministic gate first, fans out specialized qualitative reviewers (correctness, security, design/maintainability, completeness, types) that audit the whole codebase citing file:line, synthesizes and tiers their findings, applies behaviour-preserving fixes in small phases, and iterates until the gate is green and no blocking findings remain. Apply when the user wants to harden, clean up, or quality-pass a whole repo or subsystem rather than ship a feature — "get this to good standing", "quality sweep", "clean up the codebase", "audit and fix quality issues", "pay down quality debt". Skip for tidying a single just-written diff (use quality-pass on the host instead) or for building new functionality (use sandbox-feature-work).
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, mcp__demesne__sandbox_agent, mcp__demesne__sandbox_research, mcp__demesne__sandbox_script
---

Take a codebase that has accreted across prior rounds of work and drive it to good standing through a demesne pipeline. Unlike `sandbox-feature-work`, there is no feature to build: the input is the whole repo, and the bar is the codebase's own intended design plus the project's quality standards. Unlike the host-side `quality-pass` skill, this audits the *entire* codebase (not one diff) through specialized reviewer agents and a multi-round fix loop.

You (the host session) author one orchestrator prompt, launch a single Opus `sandbox_agent`, and that orchestrator runs the audit-fix-review loop. The host then runs the real build/lint/test as a backstop and commits.

This design is grounded in published practice (Google eng-practices / Sadowski 2018, Bacchelli & Bird 2013, Self-Refine, the "LLMs can't self-correct without external signal" result, LLM-as-judge bias work). The principles below are load-bearing, not decoration.

## Core principles (why the pipeline is shaped this way)

1. **Deterministic gate first; reviewers never re-find what tooling catches.** Run formatter + linter + type-check + tests + any SAST to green *before* qualitative review, and explicitly forbid reviewers from flagging lint/format/type nits. Qualitative review is for design, correctness, missing tests, spec/intent fit, coupling — the ~75% of real findings tooling can't see.
2. **Separate the critic from the fixer.** The agent that fixes code is a poor judge of its own work (intrinsic self-correction degrades without an external signal). Reviewers run in distinct contexts/roles from fixers.
3. **Specialize reviewers by dimension, then synthesize.** Parallel dimension-specialized auditors beat one generalist; a single synthesizer merges and dedupes — reviewers do not debate each other.
4. **Tier every finding; "good standing with a documented backlog" is a valid terminal state.** Only *blocking* findings drive fix cycles. Non-blocking nits are recorded, not looped on.
5. **Behaviour preservation is the invariant.** This is quality improvement, not a feature change. The test suite passing every round is the behaviour-preservation safety net — that's why the gate must stay green after every fix phase.
6. **Require file:line evidence for every finding.** Sharpens the reviewer and exposes cosmetic/empty findings (a guard against the loop churning to game its own verdict).

## Pipeline shape

```
host (you)
 └─ Opus sandbox_agent  ── ORCHESTRATOR ─────────────────────────────┐
      copies repo whole (incl .git) /in/<repo> → /workspace/repo      │
      0. BASELINE   sandbox_script image=go: run the gate, capture    │
                    its findings; fix gate failures first             │
      1. AUDIT      parallel sandbox_agent reviewers, one per          │
                    dimension; each writes /out/AUDIT-<dim>.md with    │
                    file:line findings + blocking/non-blocking tier    │
      2. SYNTHESIZE orchestrator merges → prioritized /workspace/ISSUES.md │
      3. PLAN       group blocking findings into small numbered phases │
      4. FIX        sandbox_agent fix01, fix02 … behaviour-preserving  │
      5. GATE       sandbox_script image=go: re-run to green per phase │
      6. RE-REVIEW  sandbox_agent review01 … reads git diff, verdict   │
                    PASS / CHANGES_NEEDED (≤3 rounds)                  │
      7. FINALIZE   commits work on a branch; CHANGES.md + BACKLOG.md │
                                                                      │
 └─ host landing  ◀──── git fetch <workspace>/repo <branch> ─────────┘
      ff-merge into target; one cheap in-repo `make validate`
      re-check + `make test-integration`; integrate when asked.
```

### Step roles

0. **BASELINE** — `sandbox_script`, `image=go`, `egress=none`. Run the project's **full** gate (`make validate` — formatter, golangci-lint, tests, build) against `/workspace/repo` and capture the output. Run the lint check via the repo's own target; **never skip a major check because the tool isn't preinstalled.** The right mechanism is `go tool` (go.mod `tool` directives, Go 1.24+): `go tool golangci-lint run` builds the exact version pinned in `go.mod`, fetched through the module-proxy sidecar (so `egress=none` works) — identical in sandbox, host, and CI, no `@latest` drift. If the repo doesn't yet declare its tools that way, adopting `go tool` is itself a worthwhile completeness fix (see landing note). These are the tool-catchable issues; fix them in an early phase (or note them for the fix phases). Everything the gate flags is *off-limits* to the qualitative auditors.
1. **AUDIT** — parallel `sandbox_agent` reviewers (`name=audit-correctness`, `audit-security`, `audit-design`, `audit-completeness`, `audit-types`), Sonnet. Each reads the codebase (and `git -C /workspace/repo log` for context) and writes `/out/AUDIT-<dim>.md`: a list of findings, **each with file:line, a severity tier (blocking / non-blocking), and a one-line rationale**. They audit only their dimension and must not flag anything the BASELINE gate already covers. Suggested dimensions and what each owns:
   - **correctness** — swallowed/ignored errors, missing error handling at boundaries, races, resource/goroutine leaks, incorrect edge-case logic.
   - **security** — trust-boundary correctness, credential/secret handling, injection surfaces, over-broad permissions. (Tools catch ~50% of vulns — this layer covers the rest.)
   - **design / maintainability** — dead code, unused fields/functions, premature or unjustified abstraction, duplicate definitions, naming that encodes history, mutable accumulation where immutable would do, over-defensive optional fields.
   - **completeness / wiring** — half-implemented features, stray TODOs, fields plumbed but never read/displayed, data that connects to no output, and project parity checks (e.g. OpenAPI ↔ router, MCP client ↔ domain, tool defs ↔ manifest ↔ README), plus "is every new invariant wired into CI?".
   - **types** — string comparisons on errors instead of `errors.Is`/`errors.As`, type switches that should be methods, bare primitives that want named types, loose `any`/`interface{}` where the type is known.
2. **SYNTHESIZE** — the orchestrator merges the per-dimension audits into one deduped, prioritized `/workspace/ISSUES.md`, blocking findings first. It does *not* let reviewers argue; it adjudicates overlaps itself.
3. **PLAN** — group the blocking findings into small numbered remediation phases (keep each phase to a reviewable size, ideally a few hundred lines of change). Write `/workspace/PLAN.md` with a handoff contract per phase.
4. **FIX** — one `sandbox_agent` per phase (`name=fix01` …), Sonnet, editing the shared `/workspace/repo`. **Behaviour-preserving**: a fix that changes observable behaviour is out of scope — it gets flagged in `BACKLOG.md`, not applied. Each writes `/out/SUMMARY.md`.
5. **GATE** — re-run the deterministic gate (step 0's script) after each fix phase or batch. Must be green before review. Green tests are the proof behaviour was preserved.
6. **RE-REVIEW + ITERATE** — `sandbox_agent` (`name=review01` …) reads `git -C /workspace/repo diff`, judges whether the applied fixes are correct, complete, and didn't introduce new issues, and writes `/out/REVIEW.md` ending in a verdict line `PASS` or `CHANGES_NEEDED`. On `CHANGES_NEEDED` (blocking only) spawn another fix phase and re-review. **Cap at 3 rounds.** If a round yields only non-blocking findings, that's a PASS — push them to the backlog, don't loop.
7. **FINALIZE** — the orchestrator commits the validated work as one or more commits on a branch in `/workspace/repo`, branched from the mounted HEAD (e.g. `pipeline/<short-task>`), writes `/out/CHANGES.md` (what changed, how it was validated, the **branch name**, behaviour-preservation note) and `/out/BACKLOG.md` (non-blocking findings + anything deferred as behaviour-changing), then prints `DONE`. Committing on a branch is what makes host landing a fetch rather than a patch-apply.

## Definition of "good standing" (terminal state)

The pipeline is complete when **all three** hold:
- the deterministic gate is green;
- no *blocking* findings remain across any dimension;
- remaining non-blocking findings are captured in `BACKLOG.md`.

Perfection is not the bar (Google: approve once it "definitely improves overall code health"). A clean gate + zero blocking findings + a documented backlog *is* good standing.

## Writing the orchestrator prompt

The orchestrator starts cold with only your prompt and the mounted repo. Brief it fully:

1. **The goal**: bring `/workspace/repo` to good standing — audit, fix blocking quality issues, leave a backlog. Not a feature change.
2. **The authoritative rubric**: tell it to read the root and project `CLAUDE.md` files in the repo — those define the project's manual quality checks and parity requirements, and are the rubric the auditors score against. Tell it intended behaviour is defined by the tests + public API, and that **behaviour must be preserved** (green tests are the proof).
3. **The pipeline contract**: the eight steps above, the dimension list, the child-naming rule, and the deterministic-gate-first / reviewers-don't-re-find-tool-issues rule.
4. **"Do NOT build, lint, or test yourself — run the gate via a `go` `sandbox_script` child."** The agent image has no Go toolchain.
5. **The exact gate command** (e.g. "run `make validate`"), including the lint target — not just `go build && go test`.
6. **Severity discipline**: every finding needs file:line + tier; only blocking findings drive fix/re-review cycles; "good standing with a backlog" is the goal, not zero nits.
7. **Output contract**: `/out/AUDIT-<dim>.md`, `/workspace/ISSUES.md`, `/workspace/PLAN.md`, per-phase `/out/SUMMARY.md`, `/out/REVIEW.md` with a `PASS`/`CHANGES_NEEDED` line, the validated work **committed on a branch** in `/workspace/repo`, and final `/out/CHANGES.md` (naming that branch) + `/out/BACKLOG.md` + `DONE`.

## Pipeline mechanics (shared with sandbox-feature-work)

These are the same hard constraints as the feature pipeline — see that skill for the full treatment:

- **`directories: ["<abs path to repo>"]` is mandatory** on the orchestrator launch. Forgetting it mounts no repo and burns budget on nothing.
- Orchestrator on **Opus**; auditors / fixers / reviewers on **Sonnet**.
- Tell it to **copy the repo whole, including `.git`** (`/in/<repo>` → `/workspace/repo`) so reviewers get `git diff`/`git log`.
- **Child names must be lowercase DNS-1123 labels** (lowercase letters, digits, interior hyphens, ≤40 chars): `audit-design`, `fix01`, `review02`.
- **Minimal host landing, and read logs not exit codes.** The in-sandbox `make validate` is the authoritative gate; the host does not re-run the full loop. When the orchestrator returns, read `/out/CHANGES.md` for the branch name, **land via branch fetch** (`git -C <repo> fetch <workspace>/repo <branch>` then `git merge --ff-only FETCH_HEAD`), run **one cheap in-repo `make validate`** re-check plus the host-only `make test-integration` (read the log — a trailing command can mask a `make` failure with exit 0), then integrate/commit when the user asks. See sandbox-feature-work's "Host-side landing" for the full treatment and the `go tool` pinning rationale.

## Constraints and known issues

- **Behaviour-preservation is non-negotiable.** Any "improvement" that changes observable behaviour is a feature decision, not a quality fix — route it to the backlog for explicit user sign-off, don't slip it into a fix phase.
- **Reviewers must cite file:line and stay in their lane.** Without evidence, findings drift into opinion and the loop can churn to game its own verdict. Forbid re-finding gate-catchable nits.
- **Lint must run in the gate** — a `go build/test`-only gate misses gofmt/gosec/testifylint/etc. The `golang:1` image lacks golangci-lint, so run it via `go tool golangci-lint` (pinned in `go.mod`, fetched through the module proxy at `egress=none`); never skip a major check because the tool isn't preinstalled, and never use `@latest` (it drifts between sandbox and host). The in-sandbox gate is authoritative; the host re-check is a second line of defence, not the place lint first runs.
- **Fix phases share `/workspace`** — keep them sequential against the same tree unless genuinely independent.
- **Don't over-iterate.** Gains concentrate in rounds 1–2; the 3-round cap plus the "only non-blocking left ⇒ PASS" rule are the convergence guards. Persistent findings at the cap signal a judgment-call gap, not a reason to keep spinning.
- **Scripts and data work run in demesne, never on the host.** The host runs only the minimal landing: a `git fetch` + ff-merge, one cheap `make validate` re-check, and the host-only integration tests.
