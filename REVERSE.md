# Project Architecture Extraction Guide (Architect Agent)

## Goal
Reverse-engineer the codebase (up to 100k LOC), identify key components, reconstruct data flow, and build logic diagrams.

## Input Conditions and Constraints
* **Input:** Path to repository root (git root). For a multi-language monorepo — analyze package-by-package / crate-by-crate followed by hierarchical assembly.
* **Exclude:** By default, skip `target/`, `node_modules/`, `dist/`, `.git/`, vendor directories. Extensible via argument.
* **Exceeding the limit (>100k LOC):**
  1. Break the analysis into strata (by packages/crates/subsystems).
  2. Perform an independent analysis of each stratum, saving intermediate notes to `docs/arch/<subsystem>.md`.
  3. Assemble a summary document `docs/ARCHITECTURE.md` referencing the detailed strata.

## Discovery Protocol (Discovery Phase)
Before generating diagrams and descriptions, perform a 4-step preliminary scan:
1. **Skeleton Scan:** Analyze project manifests (`Cargo.toml`, `go.mod`, `package.json`), build configurations, `README`, and directory tree to a depth of 2–3 levels.
2. **Entrypoint Tracing:** Identify all executable binaries, CLI commands, HTTP/gRPC/WebSocket routers, background task workers, and event listeners.
3. **Core Domain Types:** Find root data structures, interfaces (traits), aggregates, and business logic state enums.
4. **Integration Points:** Localize network clients, database drivers, message brokers, external SDKs, and FFI bindings.

## Analysis Algorithm

### 1. High-Level Design
* Extract entrypoints and assemble the subsystem tree.
* Analyze external dependencies, configuration files, and environment variables.
* Construct a high-level component diagram (Mermaid `graph TD` or `C4Context`).

### 2. External Integrations and System Boundaries
* Identify external services: databases, message brokers, HTTP/gRPC APIs, third-party SDKs.
* Highlight trust boundaries and authentication/authorization points.
* Record protocols and serialization formats at system boundaries (JSON, Protobuf, binary formats).

### 3. Data Flow & State
* Trace the path of key entities from input points (API/CLI/I-O) to storage or output.
* Identify global state, shared memory, and synchronization mechanisms.
* Reflect key entity lifecycles via a state diagram (Mermaid `stateDiagram-v2`).

### 4. Sequence Diagrams
* For complex scenarios and branching business logic, generate Mermaid `sequenceDiagram`.
* Reflect asynchronous calls, error handling, timeouts, and transaction boundaries.

### 5. Language-Specific Guidelines
Before decomposing modules, be sure to apply recommendations from the relevant file depending on the project stack:
* **Rust**: `agentic/rust/RUST_ARCHITECT.md`
* For languages without a dedicated file, use general heuristics: entrypoints → modules → data types → error handling → concurrency.
* File name format: `agentic/<lang>/<LANG>_ARCHITECT.md` (e.g., `agentic/ts/TS_ARCHITECT.md`).

### 6. Architectural Invariants
* Identify verifiable architectural invariants (e.g., "Domain layer does not depend on I/O", "Public API does not expose internal types").
* For each invariant, specify whether it can be covered by a test or linter, and attach the corresponding check.

## Mermaid Diagram Conventions
* **Size:** No more than 30 nodes per diagram. If exceeded — break down into subdiagrams with references.
* **Node Naming:** Short identifiers (`A`, `B`, `SVC`), display-label in square brackets: `A[Order Matcher]`.
* **Diagram Types:**
  * `graph TD` / `flowchart TD` — Control flows and components.
  * `sequenceDiagram` — Interaction scenarios.
  * `stateDiagram-v2` — State machines and domain entity state transitions.
  * `classDiagram` — Domain model and type hierarchies.
  * `C4Context` / `C4Container` — System context with external actors and containers.
* **C4 Diagram Syntax:** C4 diagrams use their own DSL, not flowchart syntax.
  * Define actors/systems with `Person(alias, "Label", "Description")`, `System(alias, "Label", "Description")`, `Container(alias, "Label", "Technology", "Description")`.
  * Define relationships with `Rel(source, target, "Label")` — **not** `source -->|label| target`. Flowchart arrow syntax (`-->`, `-->|...|`) causes lexical errors in C4 diagrams.
  * Do **not** use double-dashes (`--`) inside C4 edge labels or descriptions; they are interpreted as arrow syntax. Use `submit / cancel` instead of `--submit / --cancel`.

## Incremental Updates
* If `docs/ARCHITECTURE.md` already exists, compare current code state with git history since the last generation.
* Regenerate only affected sections and diagrams. Keep unchanged blocks as is.
* Update the version tag in the document header: `<!-- Generated: YYYY-MM-DD | Base commit: <sha> -->`.

## Final Document Template (`docs/ARCHITECTURE.md`)

```markdown
<!-- Generated: YYYY-MM-DD | Base commit: <sha> -->

# Project Architecture <Project Name>

## 1. Overview & Domain Model
Brief description of system purpose, primary use cases, and key domain entities.

\`\`\`mermaid
classDiagram
    %% Domain types and their relationships
\`\`\`

## 2. Component Architecture
High-level decomposition into modules/subsystems and areas of responsibility.

\`\`\`mermaid
flowchart TD
    %% Components and relationships between them
\`\`\`

## 3. External Integrations & Trust Boundaries
External services, interaction protocols, trust boundaries, and security.

\`\`\`mermaid
C4Context
    %% External systems and users
\`\`\`

## 4. Data Flow, Concurrency & State Management
Data flows, state management, synchronization mechanisms, and asynchronous pipelines.

\`\`\`mermaid
stateDiagram-v2
    %% Key entity lifecycle
\`\`\`

## 5. Key Sequence Diagrams
Diagrams for critical end-to-end scenarios.

\`\`\`mermaid
sequenceDiagram
    %% Operation execution scenario
\`\`\`

## 6. Architectural Invariants
| Invariant | Level | Verification Mechanism | Status |
| :--- | :--- | :--- | :--- |
| Invariant 1 | Compile-time / Lint / Test | `cargo test ...` | Enforced / Proposed |

## 7. Architectural Debt & Bottlenecks
- **[Debt/Risk Item]**: Problem description, affected files (`path/to/file:line`), potential impact, and remediation paths.
```

## Completion Criteria
The agent's work is considered complete when all items are checked:
- [ ] Preliminary Discovery Phase completed.
- [ ] All entrypoints are reflected in the component diagram.
- [ ] Each key scenario has a sequence diagram.
- [ ] External integrations and trust boundaries are documented.
- [ ] Architectural invariants are identified and verifiable.
- [ ] Architectural debt is documented with specific files/lines.

