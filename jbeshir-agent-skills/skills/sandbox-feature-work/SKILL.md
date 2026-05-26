---
name: sandbox-feature-work
description: Drive a non-trivial code change through a demesne-orchestrated pipeline — an Opus orchestrator that runs research → plan → orchestrator-chosen numbered implementation phases → an authoritative in-sandbox validation gate → iterative code+completeness review and fix, then a minimal host-side landing (branch fetch + a cheap re-check + integration tests). Apply when the user wants a feature, refactor, or other substantial change built in a sandbox rather than edited directly on the host. Triggers include "run this through demesne", "plan a pipeline to…", "use a sandboxed orchestrator", "implement this in phases in the sandbox", and any request to build something larger than a quick edit where the demesne MCP is available. Skip for trivial one-line edits, pure investigation, or work in a repo demesne can't reach.
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, mcp__demesne__sandbox_agent, mcp__demesne__sandbox_research, mcp__demesne__sandbox_script
---

Drive a substantial code change through a demesne pipeline instead of editing on the host. You (the host session) author one orchestrator prompt, launch a single Opus `sandbox_agent`, and that orchestrator fans out the rest. The in-sandbox gate (full `make validate`) is the authoritative deterministic check; the host's job afterwards is deliberately small — fetch the orchestrator's branch into the real repo, run one cheap in-repo re-check, and the host-only integration tests. Minimising host-side work (and the approvals it triggers) is a primary goal of the pipeline.

This is the workflow validated in the demesne repo itself. Follow it; the constraints below are paid-for lessons, not style preferences.

## Pipeline shape

```
host (you)
 └─ Opus sandbox_agent  ── ORCHESTRATOR ─────────────────────────────┐
      copies repo whole (incl .git) /in/<repo> → /workspace/repo      │
      1. RESEARCH    sandbox_research (isolated, open egress)         │
      2. PLAN        orchestrator picks numbered phases + handoff     │
      3. IMPLEMENT   sandbox_agent phase01, phase02, …  (sonnet)      │
      4. VALIDATE    sandbox_script image=go: full `make validate`    │
                     — the authoritative deterministic gate           │
      5. REVIEW      sandbox_agent review01, review02, … (≤3 rounds)  │
      6. FINALIZE    commit on branch; cp → /out/repo; CHANGES + DONE│
                                                                      │
 └─ host landing  ◀──── git fetch <output_dir>/repo <branch> ────────┘
      ff-merge into target; one cheap in-repo `make validate`
      re-check + `make test-integration`; integrate when asked.
```

The orchestrator runs the whole inner pipeline autonomously. You don't drive each step — you write one good prompt, launch it, then validate what lands in the shared `/workspace`.

### Step roles

1. **RESEARCH** — `sandbox_research`, **isolated**: no `/in`, no `/workspace`, no repo. Open egress + Claude available. It studies the problem from the spec you embed in its prompt (and the open web) and writes `/out/FINDINGS.md`. Isolation is deliberate: research shouldn't touch or depend on the work tree. Anything the researcher must know about the codebase goes in its prompt, because it can't read the repo.
2. **PLAN** — the orchestrator itself, after reading FINDINGS, decides the numbered implementation phases and writes `/workspace/PLAN.md` including a **handoff contract**: what each phase receives, edits, and leaves for the next. Phases are chosen post-research, not pre-baked by you.
3. **IMPLEMENT** — one `sandbox_agent` per phase (`name=phase01`, `phase02`, …), Sonnet, editing the **shared** `/workspace/repo`. Each writes `/out/SUMMARY.md`. Phases share the work tree, so they run sequentially unless genuinely independent.
4. **VALIDATE** — `sandbox_script`, `image=go`, `egress=none`. This is the **authoritative deterministic gate**, not a dry run the host repeats. Runs the project's **full** gate against `/workspace/repo`: `make validate` (formatter, golangci-lint, tests, build), not just `go build/vet/test`. The `golang:1` image lacks golangci-lint, so the gate must provide it — **never skip the check**. Run it via `go tool golangci-lint` (go.mod `tool` directives, Go 1.24+): the version is pinned in `go.mod` and built/fetched through the module-proxy sidecar, so it works at `egress=none` and is identical to host and CI. Don't use `go install ...@latest` (it drifts — see landing note).
5. **REVIEW + ITERATE** — `sandbox_agent` (`name=review01`, …) reads `git -C /workspace/repo diff` and judges both code quality and completeness against the spec, writing `/out/REVIEW.md` ending in a verdict line `PASS` or `CHANGES_NEEDED`. On `CHANGES_NEEDED` the orchestrator spawns a fix phase, then re-reviews. **Cap at 3 rounds.**
6. **FINALIZE** — the orchestrator commits the validated work as one or more commits on a branch in `/workspace/repo`, branched from the mounted HEAD (e.g. `pipeline/<short-task>`). It then **copies the committed repo into its own `/out`** so the host can reach it: `cp -a /workspace/repo /out/repo`, run **by the orchestrator itself** (plain `cp`, no toolchain needed) — *not* via a `sandbox_script` child, whose `/out` is `/out/child/<name>` and would strand the repo there. `/workspace` is torn down when the orchestrator exits; only `/out` (the returned `output_dir`) is persisted to the host, so the branch must live in a repo under `/out`. Finally it writes `/out/CHANGES.md` (what changed, how it was validated, the **branch name**, any caveats) and prints `DONE`. Committing on a branch is what makes host landing a fetch rather than a patch-apply.

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
4. **"Do NOT build, lint, or test yourself — validate via a `go` `sandbox_script` child."** The agent image has no Go toolchain; an orchestrator that tries to compile wastes turns failing. Validation is a child's job, and that in-sandbox gate is authoritative — the host does not re-run it as a matter of course.
5. **The VALIDATE command explicitly** — the full gate including lint (e.g. "install golangci-lint via `go install`, then run `make validate`"), not just `go build && go test`.
6. **Files to study** — point it at the key files/dirs so it doesn't waste a research budget rediscovering structure.
7. **The handoff contract requirement** — tell it to write `/workspace/PLAN.md` and have each phase honour it strictly.
8. **Already-ruled-out facts** — anything you've investigated and disproven. This stops the orchestrator re-treading dead ends (e.g. "the X timeout is not the cause; MCP_TOOL_TIMEOUT defaults to ~27h").
9. **Output contract**: `/out/FINDINGS.md`, per-phase `/out/SUMMARY.md`, `/out/REVIEW.md` with a `PASS`/`CHANGES_NEEDED` verdict line, the validated work **committed on a branch** and **copied to `/out/repo`** by the orchestrator itself, and a final `/out/CHANGES.md` (naming that branch) + `DONE`.

Terse prompts produce shallow pipelines. Over-specify the contract; under-specify the solution.

## Host-side landing (keep it minimal)

The in-sandbox `make validate` is the authoritative deterministic gate. The host does **not** re-run the full build/lint/test loop or apply patches — it lands the orchestrator's branch and does one cheap confirmation. Keeping this small is the point: every host command is a potential approval prompt.

1. Read `/out/CHANGES.md` for the branch name and a summary.
2. **Land via branch fetch from `output_dir`.** The orchestrator left the committed repo at `<output_dir>/repo` (`output_dir` is the host path the run returned). `/workspace` is gone — `/out` is the only persisted location — so fetch from there and fast-forward your target branch:
   - `git -C <repo> fetch <output_dir>/repo <branch>`
   - `git -C <repo> merge --ff-only FETCH_HEAD` — the branch descends from the HEAD the sandbox copied, so this is a clean fast-forward. This preserves the sandbox's commits; no patch-apply, no re-copy.
   - **Fallback:** if `<output_dir>/repo` is absent (an orchestrator that copied via a child instead of itself), the repo is under `<output_dir>/child/<name>/repo` — find the one whose `git log` carries the branch and fetch from there.
3. **One cheap in-repo re-check.** Run `make validate` once in the real repo as a backstop, in-repo via the project's own target (not ad-hoc `go` commands). With tool versions pinned (below) this is a formality that passes first time; a failure means a version/environment gap to fix, not routine churn. **Read the log, not the exit code** — a trailing command can mask an earlier `make` failure with a 0 exit.
4. **Integration tests.** Run the project's host-only integration target (`make test-integration`) — the one check the sandbox can't do (it needs real podman + `.env`).
5. Manual quality pass (the `CLAUDE.md` checks); integrate/commit when the user asks.

**Use `go tool` to pin tooling so the sandbox gate is trustworthy.** The host re-check is only cheap if the sandbox and host run the *same* linter and toolchain. A repo that installs golangci-lint via `go install ...@latest` lets the sandbox and host resolve different versions that disagree on findings — a real failure mode: gosec/`nolint` verdicts differ between golangci-lint v2.11.x and v2.12.x, so a change green in-sandbox fails the host re-check on a line neither author touched. The fix is `go tool` (Go 1.24+): declare golangci-lint and the other dev tools as `tool` directives in `go.mod` (`go get -tool ...`) and invoke them as `go tool golangci-lint run` from the Makefile. The version is then pinned in the module and resolved identically in the sandbox, on the host, and in CI — no `@latest`, no separate install step, no drift. A repo still on `@latest` should adopt `go tool` as a first-class fix; only then does "sandbox authoritative" actually hold.

## Constraints and known issues

- **Lint must run in the gate, not just on the host.** A `go build/vet/test`-only VALIDATE step misses gofmt/gosec/testifylint/errorlint/etc. nits golangci-lint catches. The `golang:1` image lacks golangci-lint, so run it via `go tool golangci-lint` (pinned in `go.mod`, fetched through the module proxy, so `egress=none` is fine) — never skip a major check because the tool isn't preinstalled, and never `@latest` it (drift). The in-sandbox gate is authoritative; the host re-check is a second line of defence, not where lint first runs.
- **Don't trust pipeline/background exit codes** (see backstop step 3). Read logs.
- **Research is isolated by design** — it has no repo access. If the researcher needs codebase facts, put them in its prompt.
- **Phases share `/workspace`** — they collide if run in parallel against the same tree. Keep them sequential unless independent. (Parallel phases via git-worktree-per-phase is a known future improvement enabled by keeping `.git`, not yet standard.)
- **Resilience**: nested child sandboxes are kept alive by keepalive progress notifications over the held-open MCP connection. If children start dying mid-run with partial transcripts and no `results.json`, suspect that mechanism regressed — investigate by reading the code path, don't guess.
- **Cost/window**: long agent runs survive Anthropic quota windows via the retry/resume wrapper (waits for the reported reset, resumes the same session). There is no per-pipeline budget gate or checkpoint-resume yet — a long pipeline can still be expensive; size the work accordingly.
- **Scripts and data work run in demesne, never on the host** (`sandbox_script` one-shot, `sandbox_agent` when it needs an agent). The host runs only the minimal landing: a `git fetch` + ff-merge, one cheap `make validate` re-check, and the host-only integration tests.
