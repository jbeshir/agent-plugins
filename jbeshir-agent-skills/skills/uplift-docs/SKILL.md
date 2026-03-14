---
name: uplift-docs
description: Uplift non-code documentation for a project. Writes or updates a README with a service summary, key domain concepts, and mermaid diagrams (architecture, dependencies, data flow). Adds or updates an OpenAPI spec for services exposing APIs.
allowed-tools: Read, Glob, Grep, Bash, Write, Edit, Agent
---

You are performing a documentation uplift on this project. Your goal is to produce high-quality, accurate non-code documentation by deeply understanding the codebase first, then dispatching parallel subagents to write the documentation artifacts.

Work through the following phases in order.

## Phase 1: Codebase Analysis

Before writing anything, thoroughly explore the codebase to understand:

1. **Purpose & Role** — What does this service/software do? What problem does it solve? Who are its users or consumers?
2. **Key Domain Concepts** — What are the core entities, abstractions, and terminology someone needs to understand to work with this project?
3. **Architecture** — How is the codebase organized? What are the major components/modules and how do they interact?
4. **Dependencies** — What external services, databases, message queues, or third-party libraries does this project depend on? What depends on it?
5. **Data Flow** — How does data enter the system, get transformed, and leave? What are the key pipelines or request paths?
6. **API Surface** — Does this project expose any APIs (REST, gRPC, GraphQL, CLI)? If so, catalog the endpoints/commands.

Use the Explore agent liberally to build a thorough understanding. Read key files: entrypoints, configuration, route definitions, schema definitions, dependency manifests (package.json, go.mod, requirements.txt, Cargo.toml, etc.).

After this phase, write a detailed summary of your findings. You will pass this summary to the subagents in Phase 2.

## Phase 2: Parallel Documentation

Launch **two subagents in parallel** using the Agent tool. Each subagent independently plans, executes, and validates its own work. Pass your full codebase analysis summary from Phase 1 to both subagents so they have full context without needing to re-explore.

### Subagent A: README

Launch a general-purpose Agent with the following prompt (include your Phase 1 summary where indicated):

---

You are writing or updating the README.md for this project. Here is a detailed analysis of the codebase:

<codebase-analysis>
{YOUR PHASE 1 SUMMARY HERE}
</codebase-analysis>

Work through these steps:

**Step 1: Plan**

Based on the codebase analysis, plan the README structure. Decide:
- What the title and summary paragraph should convey
- Which domain concepts are essential to define
- What architecture diagram best communicates the system structure
- What dependency diagram to include
- What data flow diagram(s) to include (and whether sequence diagrams or flowcharts are more appropriate)
- Whether additional diagrams would meaningfully aid understanding (e.g. state diagrams, ER diagrams)
- What getting started instructions are needed
- Whether an API reference section is needed

If a README.md already exists, read it and decide what to preserve, rewrite, or remove.

**Step 2: Execute**

Write or update `README.md`. Include the following sections. Use clear, concise prose written for a new team member who needs to get oriented quickly.

1. **Title & Summary** — One-paragraph description of what the project is, what it does, and why it exists.

2. **Key Concepts** — A glossary or short descriptions of the core domain concepts. Define terms that appear throughout the codebase and that a reader needs to understand before diving in.

3. **Architecture Overview** — A mermaid diagram showing the major components/modules and their relationships, with a brief prose explanation.

4. **Dependency Diagram** — A mermaid diagram showing external dependencies (services, databases, queues, third-party APIs) and the direction of dependency.

5. **Data Flow** — A mermaid diagram showing how data moves through the system for the primary use case(s). Use sequence diagrams or flowcharts as appropriate.

6. **Getting Started** — How to install dependencies, configure, and run the project locally.

7. **API Reference** — If the project exposes an API, a brief summary of the available endpoints/commands with a pointer to the full OpenAPI spec if one exists.

Diagram guidelines:
- Keep diagrams focused and readable — no more than ~15 nodes per diagram. Split into multiple diagrams if needed.
- Use descriptive labels on nodes and edges.
- Choose the right diagram type: `graph TD` for hierarchies/architectures, `graph LR` for dependency flows, `sequenceDiagram` for request/data flows, `flowchart` for decision processes.
- Add additional diagrams beyond the required three if they would meaningfully aid understanding (e.g. state diagrams for lifecycle management, ER diagrams for data models).

**Step 3: Validate**

After writing the README:
1. Re-read the final README.md from disk.
2. Verify every mermaid diagram has valid syntax (correct node definitions, arrow syntax, proper quoting of labels with special characters).
3. Cross-check that the README accurately reflects the codebase — no references to components that don't exist, no missing major components.
4. Fix any issues found.

---

### Subagent B: OpenAPI Specification

Launch a general-purpose Agent with the following prompt (include your Phase 1 summary where indicated):

---

You are reviewing and writing the OpenAPI specification for this project. Here is a detailed analysis of the codebase:

<codebase-analysis>
{YOUR PHASE 1 SUMMARY HERE}
</codebase-analysis>

Work through these steps:

**Step 1: Plan**

Determine whether this project exposes an HTTP API (REST or similar). Check for:
- Route/handler definitions (e.g. Express routes, FastAPI endpoints, Go HTTP handlers, Spring controllers)
- An existing OpenAPI/Swagger spec file (`openapi.yaml`, `openapi.json`, `swagger.yaml`, `swagger.json`, or similar)

If the project does not expose an HTTP API, report that no API was detected and stop.

If it does, plan your approach:
- If no spec exists, you will create `openapi.yaml` in the project root conforming to OpenAPI 3.1.
- If a spec exists, read it and catalog what needs to be added, corrected, or improved.
- Read the actual route/handler code to build an accurate picture of every endpoint, its parameters, request bodies, response schemas, and error cases.

**Step 2: Execute**

Write or update the OpenAPI spec. It must include:
- `info` with title, description, and version
- All endpoints with accurate paths, methods, parameters, request bodies, and response schemas
- Meaningful `description` fields on operations, parameters, and schemas
- Proper `$ref` usage for shared schemas in `components/schemas`
- Accurate status codes and error response schemas
- Authentication/security schemes if applicable

**Step 3: Validate**

After writing the spec:
1. Re-read the final spec file from disk.
2. Verify it is valid YAML and conforms to OpenAPI 3.1 structure.
3. Cross-check every endpoint in the spec against the actual route/handler code — ensure nothing is missing or inaccurate.
4. Fix any issues found.

---

## Phase 3: Summary

After both subagents complete, provide a brief summary of what was created or updated, and flag any areas where you were uncertain or where the documentation would benefit from human review.
