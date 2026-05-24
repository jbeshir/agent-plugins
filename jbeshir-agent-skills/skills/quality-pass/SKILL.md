---
name: quality-pass
description: Tidy a completed change before commit or PR so the diff leaves the surrounding code better than it found it. Apply when a change has just been finished and the user wants a focused pass that runs automated checks, performs simplifications and dead-code removal the change enables, tightens typing, and applies the manual quality checks from CLAUDE.md. Triggers include "do a quality pass", "tidy this up", "polish", "quality check", "boy scout", "leave it better than we found it", and pre-commit / pre-PR cleanup. Skip if the change is being intentionally discarded, or if reviewing someone else's PR (use review or security-review instead).
allowed-tools: Read, Glob, Grep, Bash, Write, Edit
---

Tidy the pending changes in this project. The bar is the boy scout rule applied to a diff: this branch leaves the surrounding code tidier than it found it.

Specifically:

1. The change is clean against the project's automated tooling.
2. Any simplifications or refactorings the change *enables* have been performed.
3. Anything that became dead, unused, or redundant as a result of the change is gone.
4. Types are as tight as they reasonably can be — without contortions.
5. The manual checks from `CLAUDE.md` (root, project, and any nested ones) have been applied.

Work through the phases below in order.

## Guardrails

These apply throughout every phase:

- **Behaviour preservation.** Refactors and simplifications must not alter observable behaviour. If a change would, it isn't a quality fix — flag it and stop.
- **Confidence floor.** If you're not certain an issue is real, don't flag it. False flags cost more than missed nits.
- **Skip false positives without arguing.** If a flagged issue turns out not to apply on closer reading, drop it and move on. Don't try to justify it.
- **Don't re-find what tooling already catches.** Phase 2 runs the formatter and linter. Phase 3 onwards is for issues those tools can't see — don't burn effort on lint-catchable findings.

## Phase 1: Identify the change scope

1. `git status` and `git diff` for uncommitted changes.
2. Identify the trunk branch (try `main`, then `master`) and `git diff <trunk>...HEAD` for committed changes ahead of trunk.
3. Read the combined diff end-to-end — you need to understand the *intent* of the change before you can tell what it enables.
4. Read the root `CLAUDE.md` and any project-level `CLAUDE.md` in the working directory; they define the validation commands and project-specific manual checks for this repo.

If the diff is empty, report that and stop.

## Phase 2: Automated validation

Run the validation commands defined in `CLAUDE.md` for this project. Common patterns:

- Go projects: `make lint`, `make test-short`, `make build`. API projects also: `make lint-openapi`.
- If a single canonical script exists (`make check`, `scripts/validate.sh`, the CI entry point), run it — it's authoritative.
- Otherwise: formatter, then linter / static analysis (including type-check for typed languages), then tests.

If anything fails, fix the underlying issue. Don't silence a lint or skip a test to make it pass.

## Phase 3: Manual quality checks

Read the diff again, looking for these issues. Fix what you find.

1. **Unnecessarily optional fields** — Did the change add a pointer / `Optional` / `| None` / `?` to something that's always provided? Make it required.
2. **Duplicate code** — Did the change add a constant, type, or helper that already exists elsewhere? Consolidate.
3. **Incomplete or disconnected logic**:
   - Legacy code the change has obsoleted but not deleted.
   - Closures capturing values that won't be re-evaluated when the underlying state changes.
   - Fields in structs/types/interfaces never read or never written.
   - Functions defined but never called.
   - Fields plumbed through but never displayed; UI fields not backed by data.
   - Half-implemented features the change started but didn't wire up.
   - `TODO`/`FIXME` comments — resolve, or be explicit about why they're left.
4. **Redundant fields or parameters**:
   - Fields/parameters whose value is derivable from other fields/parameters.
   - Methods with no callers (or only test callers).
   - Re-exports of types callers can import directly.
5. **Poor use of types**:
   - String comparisons on errors instead of typed errors with `errors.Is`/`errors.As` (Go) or exception classes (Python).
   - Type switches that branch to call different methods — usually a method on the type itself is cleaner.
6. **Mutable accumulation where immutable would do** — Functions that mutate a receiver/object instead of returning a result.
7. **Naming that encodes history** — No `RecommendEnhanced` because the old `Recommend` was removed. Names should make sense from the current code alone; let git remember the history.
8. **Comments narrating WHAT the code does** — Delete. Keep only comments that explain WHY: a non-obvious constraint, a workaround for a specific bug, an invariant, behaviour that would surprise a reader. Well-named identifiers already say what.
9. **Validation script coverage** — A new invariant or check should be wired into CI.

Apply project-specific checks from the local `CLAUDE.md` too. Common ones in this codebase:

- **OpenAPI parity** — When endpoints change, the spec must list every endpoint from the router with correct schemas, parameters, and status codes.
- **MCP client/tool parity** — When domain structs or endpoints change, MCP client structs and tool registrations stay in sync. When tool parameters or descriptions change, all of `tools.go`, `manifest.json`, and `README.md` stay consistent.
- **Datasource constructors** — Single `New(Config, *http.Client)` constructor; no `NewWithURL`/`NewWithClient` variants. `Config` zero-value uses defaults; `baseURL` lives in config and full URLs are computed inline.
- **Environment variables** — Any env var the code reads is documented in the server's README and present in `.env.dist`.
- **`server.json` descriptions** — ≤100 characters (registry rejects longer ones).

## Phase 4: Refactoring opportunities the change enables

The highest-value part of the pass. For each modified or removed concept, ask:

1. **Did anything become unused?** Removed call sites can leave functions with no remaining callers; renamed fields can leave types with no users. Grep the codebase for old names and remove orphans. If only tests reference something, it's probably dead.
2. **Did any abstraction become unjustified?** Two implementations collapsed to one → the shared interface may be unnecessary. Generic container instantiated with only one type → specialise it. Config option with only one valid value → hardcode it.
3. **Did any branching become dead?** A removed code path may eliminate `switch` arms, `if` branches, or whole conditionals. Constants used only by the deleted branch go too.
4. **Did adjacent code stop making sense?** Callers often do redundant work — validating before a function that already validates, unwrapping then rewrapping, transforming into a shape the new code no longer needs. Tidy the immediate neighbourhood.
5. **Did a workaround become unnecessary?** Search for comments referencing old behaviour ("workaround for X", "until X is fixed", "TODO: remove once Y"). If the change resolves the underlying issue, the workaround goes.
6. **Did indirection lose its purpose?** A wrapper that used to dispatch but now just forwards is dead weight; a param-object that used to carry many fields but now carries one can be inlined.

## Phase 5: Typing review

Look at every type the diff touched and ask whether it could be tighter without contortions.

1. **Enums instead of strings** — A string with a fixed set of valid values becomes an enum / `StrEnum` / typed constant / string-literal union.
2. **Named types instead of bare primitives** — A `string` that's always a `UserID` and never anything else gets a type (Go: `type UserID string`; Python: `NewType('UserID', str)`; TypeScript: a branded type). Especially valuable when distinct IDs flow through the same code path and could be swapped.
3. **Discriminated unions instead of mutually-exclusive optionals** — In TypeScript / Python (`Union` of dataclasses with a literal discriminator) / Rust, prefer a tagged union to a struct with two optionals where exactly one must be set. *Skip this for Go* — emulating sum types isn't idiomatic and the encoding contorts the rest of the code.
4. **Parse, don't validate** — Validation that returns `error`/`bool` and hands back the same untyped input is weaker than a parser that returns a typed value. Convert "validate then use the original" into "parse into a typed value then use that."
5. **Enums instead of boolean parameters** — `DoThing(true, false)` is unreadable; `DoThing(WithRetry, WithoutCache)` is clear. Especially when a call passes two or more booleans.
6. **Tighten container types** — `map[string]any` / `Dict[str, Any]` / `Record<string, unknown>` that only ever holds known keys becomes a struct/dataclass/object type. Fixed-length slices become tuples.
7. **Remove `any`/`interface{}`/`unknown` where the actual type is known** — Often a leftover that no longer needs to be loose.

Apply the "without contortions" filter strictly. Don't introduce generics, builders, phantom types, or new patterns just to tighten one field. If tightening requires inventing a pattern no other code in the project uses, drop it.

## Phase 6: Re-run automated validation

After Phases 3–5, re-run the formatter, linter, and tests. Quality fixes routinely introduce regressions.

## Phase 7: Report

A tight summary, in three labelled sections. Omit any section that's empty.

**Fixed**
- One bullet per fix, grouped by phase (Phase 3 / 4 / 5). One line each.

**Deliberately skipped**
- Things you considered and chose not to change. One line for the finding, one line for the reason. This is the most useful section — it shows what was looked at and *why* it was left alone.

**Out of scope — flag for follow-up**
- Issues you noticed that the current change doesn't enable a clean fix for. Brief, so the user can decide whether to schedule a follow-up.

The diff speaks for itself; the report is for what the diff doesn't surface.
