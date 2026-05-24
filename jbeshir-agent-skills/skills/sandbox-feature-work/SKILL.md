---
name: sandbox-feature-work
description: Drive a non-trivial code change through a demesne-orchestrated pipeline — an Opus orchestrator that runs research → plan → orchestrator-chosen numbered implementation phases → automated validation → iterative code+completeness review and fix, then a host-side validate-fix backstop. Apply when the user wants a feature, refactor, or other substantial change built in a sandbox rather than edited directly on the host. Triggers include "run this through demesne", "plan a pipeline to…", "use a sandboxed orchestrator", "implement this in phases in the sandbox", and any request to build something larger than a quick edit where the demesne MCP is available. Skip for trivial one-line edits, pure investigation, or work in a repo demesne can't reach.
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, mcp__demesne__sandbox_agent, mcp__demesne__sandbox_research, mcp__demesne__sandbox_script
---

Drive a substantial code change through a demesne pipeline instead of editing on the host. You (the host session) author one orchestrator prompt, launch a single Opus `sandbox_agent`, and that orchestrator fans out the rest. The host's job afterwards is the real build/lint/test that the sandbox can't run, fed back as a fix loop until green.

This is the workflow validated in the demesne repo itself. Follow it; the constraints below are paid-for lessons, not style preferences.

## Pipeline shape

```
host (you)
 └─ Opus sandbox_agent  ── ORCHESTRATOR ─────────────────────────────┐
      copies repo whole (incl .git) /in/<repo> → /workspace/repo      │
      1. RESEARCH    sandbox_research (isolated, open egress)         │
      2. PLAN        orchestrator picks numbered phases + handoff     │
      3. IMPLEMENT   sandbox_agent phase01, phase02, …  (sonnet)      │
      4. VALIDATE    sandbox_script  image=go egress=none             │
      5. REVIEW      sandbox_agent review01, review02, … (≤3 rounds)  │
      6. FINALIZE    writes /out/CHANGES.md + DONE                    │
                                                                      │
 └─ host validate-fix loop  ◀─── reads /workspace + /out/CHANGES.md ──┘
      make validate / go vet -tags integration; feed failures into a
      fix pass until green; then commit.
```

The orchestrator runs the whole inner pipeline autonomously. You don't drive each step — you write one good prompt, launch it, then validate what lands in the shared `/workspace`.

### Step roles

1. **RESEARCH** — `sandbox_research`, **isolated**: no `/in`, no `/workspace`, no repo. Open egress + Claude available. It studies the problem from the spec you embed in its prompt (and the open web) and writes `/out/FINDINGS.md`. Isolation is deliberate: research shouldn't touch or depend on the work tree. Anything the researcher must know about the codebase goes in its prompt, because it can't read the repo.
2. **PLAN** — the orchestrator itself, after reading FINDINGS, decides the numbered implementation phases and writes `/workspace/PLAN.md` including a **handoff contract**: what each phase receives, edits, and leaves for the next. Phases are chosen post-research, not pre-baked by you.
3. **IMPLEMENT** — one `sandbox_agent` per phase (`name=phase01`, `phase02`, …), Sonnet, editing the **shared** `/workspace/repo`. Each writes `/out/SUMMARY.md`. Phases share the work tree, so they run sequentially unless genuinely independent.
4. **VALIDATE** — `sandbox_script`, `image=go`, `egress=none`. Runs the project's **full** gate against `/workspace/repo`: `make validate` (formatter, golangci-lint, tests, build), not just `go build/vet/test`. If the image lacks golangci-lint (the `golang:1` image does), **install it; don't skip the check** — `go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest` works at `egress=none` through the module-proxy sidecar (match the major version to the repo's `.golangci.yml`), add `$(go env GOPATH)/bin` to `PATH`, then run the gate.
5. **REVIEW + ITERATE** — `sandbox_agent` (`name=review01`, …) reads `git -C /workspace/repo diff` and judges both code quality and completeness against the spec, writing `/out/REVIEW.md` ending in a verdict line `PASS` or `CHANGES_NEEDED`. On `CHANGES_NEEDED` the orchestrator spawns a fix phase, then re-reviews. **Cap at 3 rounds.**
6. **FINALIZE** — orchestrator writes `/out/CHANGES.md` (what changed + how it was validated + any caveats) and prints `DONE`.

## Launching the orchestrator

- **`directories: ["<abs path to repo>"]` is mandatory.** Forgetting it mounts nothing — the orchestrator wakes up with no repo, burns budget diagnosing the emptiness, and produces nothing. This is the single most expensive mistake; double-check it on every launch.
- Tell the orchestrator to **copy the repo whole, including `.git`**, from `/in/<repo>` into `/workspace/repo`. The `.git` enables `git diff` for reviewers and a future git-worktree-per-phase path.
- Model: **Opus** for the orchestrator; **Sonnet** for implementation/review phases (the orchestrator sets these when it spawns children).
- Child sandbox **names must be lowercase DNS-1123 labels**: lowercase letters, digits, interior hyphens only, ≤40 chars. `phase01`, `review02` — never `Phase_1` or `review.final`. Bad names produce invalid volume names and poison sibling spawns.

## Writing the orchestrator prompt

The orchestrator starts cold with only your prompt and the mounted repo. Treat the prompt as a complete briefing document, not an instruction. A good one contains:

1. **The goal and why**, in enough depth that the orchestrator can make judgment calls — not just a one-line task.
2. **The full feature spec** plus any **design decisions already seeded** by the user (so phases don't relitigate settled choices).
3. **The pipeline contract**: spell out the six steps above, the child naming rule, and that research is isolated.
4. **"Do NOT build, lint, or test yourself — validate via a `go` `sandbox_script` child."** The agent image has no Go toolchain; an orchestrator that tries to compile wastes turns failing. Validation is a child's job (and the host's).
5. **The VALIDATE command explicitly** — the full gate including lint (e.g. "install golangci-lint via `go install`, then run `make validate`"), not just `go build && go test`.
6. **Files to study** — point it at the key files/dirs so it doesn't waste a research budget rediscovering structure.
7. **The handoff contract requirement** — tell it to write `/workspace/PLAN.md` and have each phase honour it strictly.
8. **Already-ruled-out facts** — anything you've investigated and disproven. This stops the orchestrator re-treading dead ends (e.g. "the X timeout is not the cause; MCP_TOOL_TIMEOUT defaults to ~27h").
9. **Output contract**: `/out/FINDINGS.md`, per-phase `/out/SUMMARY.md`, `/out/REVIEW.md` with a `PASS`/`CHANGES_NEEDED` verdict line, final `/out/CHANGES.md` + `DONE`.

Terse prompts produce shallow pipelines. Over-specify the contract; under-specify the solution.

## Host-side validate-fix backstop

The sandbox validation is necessary but **not sufficient** — trust the host, not the pipeline's self-report.

1. When the orchestrator returns, read `/out/CHANGES.md` and inspect the diff in `/workspace/repo` (or wherever the shared workspace landed on the host).
2. Run the **real** validation on the host: `make validate`, plus integration vetting like `go vet -tags integration` where the project has it. Use the project's Makefile targets, not ad-hoc `go` commands.
3. **Read the validation log, do not trust an exit code.** A trailing `echo` (or any final command) can mask an earlier `make` failure with a 0 exit. The log is authoritative; the exit code is not.
4. Feed any failures back — either a host-side fix or another sandbox fix phase — and re-validate until green.
5. Manual quality pass (the `CLAUDE.md` checks) and only then commit, when the user asks.

## Constraints and known issues

- **Lint must run in the gate, not just on the host.** A `go build/vet/test`-only VALIDATE step misses gofmt/gosec/testifylint/errorlint/etc. nits golangci-lint catches. If the image lacks golangci-lint, the gate `go install`s it (through the module proxy, so `egress=none` is fine) and runs it — never skip a major check because the tool isn't preinstalled. The host backstop is a second line of defence, not where lint first runs.
- **Don't trust pipeline/background exit codes** (see backstop step 3). Read logs.
- **Research is isolated by design** — it has no repo access. If the researcher needs codebase facts, put them in its prompt.
- **Phases share `/workspace`** — they collide if run in parallel against the same tree. Keep them sequential unless independent. (Parallel phases via git-worktree-per-phase is a known future improvement enabled by keeping `.git`, not yet standard.)
- **Resilience**: nested child sandboxes are kept alive by keepalive progress notifications over the held-open MCP connection. If children start dying mid-run with partial transcripts and no `results.json`, suspect that mechanism regressed — investigate by reading the code path, don't guess.
- **Cost/window**: long agent runs survive Anthropic quota windows via the retry/resume wrapper (waits for the reported reset, resumes the same session). There is no per-pipeline budget gate or checkpoint-resume yet — a long pipeline can still be expensive; size the work accordingly.
- **Scripts and data work run in demesne, never on the host** (`sandbox_script` one-shot, `sandbox_agent` when it needs an agent). The host runs only routine repo ops (git, make) and the validate-fix backstop.
