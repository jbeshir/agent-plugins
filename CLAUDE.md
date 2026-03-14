# Global CLAUDE.md

## Command Execution

- Issue shell commands separately rather than chaining with `&&` or `;`. This makes failures easier to identify and tool calls easier to review.
- Avoid multiline commands where possible.

## Code Standards

- Names should make sense from the current code alone — don't encode history (e.g. no `RecommendEnhanced` because the old `Recommend` was removed). Let git remember the history.
- Prefer immutable data flows over mutable state. Functions should return results rather than accumulating state on objects/receivers.

## Manual Quality Checks

After automated validation passes, review code for these issues:

1. **Unnecessarily Optional Fields**: Simplify dependencies by assuming we always provide them rather than defensively allowing nil/optional.

2. **Duplicate Code**: Check for repeated definitions (constants, types, helper functions) that should be consolidated.

3. **Incomplete or Disconnected Logic**: Look for:
   - Legacy mechanisms or functions left in instead of being removed
   - Closures closing over data that may be outdated
   - Fields in types/interfaces that are never read or written
   - Functions that are defined but never called
   - Data properties that don't connect to any UI or output
   - Features partially implemented but not wired up
   - TODO comments indicating unfinished work

4. **Redundant Fields or Parameters**: Look for:
   - Fields/parameters that contain information present in or inferrable from other fields
   - Methods that aren't used or are only used in tests
   - Re-exporting of types available elsewhere

5. **Poor Use of Types**: Look for:
   - String comparisons on errors instead of using typed errors and `errors.Is`/`errors.As`
   - Type switches to determine functionality when a method on the type would be cleaner

6. **Validation Script Coverage**: Ensure any new validations are added to CI.

## Go

These checks apply where relevant — not every Go project will have all of these, but check for them when the project does.

- **OpenAPI Spec Completeness**: When modifying API endpoints, verify that the OpenAPI spec includes all endpoints from the router, has correct request/response schemas, documents all parameters and request bodies, and lists all possible HTTP status codes.
- **MCP Client Parity**: When modifying domain structs or API endpoints, verify that any MCP client stays in sync — client struct fields match domain fields, and new endpoints are exposed as MCP tools.
- **MCP Tool Definitions in Sync**: When adding or changing tool parameters, descriptions, or behaviour, check all locations are consistent: tool registration code, `manifest.json`, and `README.md`.
- **Datasource Constructors**: Single `New` constructor taking `(Config, *http.Client)` — no `NewWithURL`/`NewWithClient` variants. `Config` struct holds optional overrides; zero value uses built-in defaults.
- **URL Handling**: Store `baseURL` in config, compute full URLs inline in methods — don't precompute URL functions or store derived URL fields.

## Diagrams

Use the mermaid MCP server to generate and validate diagrams for READMEs and documentation where appropriate.

- **In READMEs**: Use fenced mermaid code blocks (` ```mermaid `) so GitHub renders them natively. Do not embed pre-rendered images. Still use the mermaid MCP server to validate the diagram renders correctly, and review and iterate on the result.
- **In other documentation**: Use the mermaid MCP server to render diagram images.
- **Always validate**: Generate every diagram with the mermaid MCP server to confirm it renders correctly before committing. Review the output and iterate if needed.
- **Keep diagrams simple**: Aim for <12 nodes. Consolidate related concepts into single nodes rather than exploding detail.
- **No custom colors**: Do not use `style` directives to set fill/stroke colors. Mermaid's default theme adapts to light and dark mode; hardcoded colors break in one or the other.
- **Design for GitHub width**: GitHub's rendered markdown is ~888px wide. Use `LR` (left-to-right) layout sparingly — prefer `TD` (top-down) for wider graphs. Keep node labels short. Test that the diagram doesn't require horizontal scrolling.

## Python

- When running Python commands, use the project's `venv/` if present.
