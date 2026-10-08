# Agentic Workflow

A structured AI agent operating framework for multi-language system and embedded software development. It defines three disciplined, multi-phase workflows that constrain AI coding agents (e.g., Cascade, Devin, Cursor) into a predictable, high-quality engineering process — eliminating ad-hoc code generation, infinite fix loops, and shallow reviews.

The framework supports **Rust**, **Go**, **Python**, and **F#** through language-specific workflow files. Each language file provides the exact format, test, lint, and dependency-check commands for Phase 4 verification. Rust and F# also include a dedicated multi-pass code review workflow with specialist roles.

## What It Does

The agentic workflow provides three core processes:

### 1. Feature Implementation & Engineering (`WORKFLOW.md`)

A 6-phase pipeline (Phases 0–5) that governs how AI agents implement features, fix bugs, and refactor code:

- **Phase 0 — Preflight & Context Recovery:** Confirm the repository is under Git, record the branch, check the worktree for pre-existing changes, and recover an active plan if one exists.
- **Phase 1 — Triage & Handle Ambiguity:** Classify risk, identify blocking ambiguities, identify safe assumptions, answer discoverable questions through exploration, propose improvements, and ask the user to resolve blocking items only.
- **Phase 2 — Create a Single Plan File:** Create a `PLAN_<name>.md` file with feature summary, scope, architecture blueprint, testing strategy, ordered tasks, dependencies, and acceptance criteria.
- **Phase 3 — Execute Tasks, Write Triad Tests, and Track Inline:** Work through tasks one by one, writing triad tests (happy path, edge cases, negative cases) before implementation code, and update the plan inline with execution logs.
- **Phase 4 — Verify & Commit After Every Task:** Select a verification gate (Fast, Dependency, or Final), run language-specific format, test, lint, and dependency commands, and commit each task atomically. A circuit breaker halts after 2 failed implementation fix attempts to prevent infinite loops.
- **Phase 5 — Finalize and Archive Plan:** Run the Final gate on the full workspace, verify every acceptance criterion, and rename `PLAN_<name>.md` to `DONE_<name>.md`.

#### Change Tiers

The workflow classifies each request before Phase 1:
- **Minor change:** Touches one file, changes fewer than 50 lines, and matches no High-Risk Category. Uses a reduced plan with only Summary and Task List.
- **Standard change:** Everything else. Follows all phases with the complete plan structure.

A change that touches security, public API, database schema, concurrency, or deployment configuration always uses the Standard change plan, even when it touches one file and changes fewer than 50 lines.

#### Plan Lifecycle

Each plan has exactly one state: `active`, `blocked`, `paused`, `completed`, or `stale`. Only one plan may hold Status `active` per branch. A plan on another branch never blocks work on the current branch.

#### Verification Gates

Phase 4 selects one of three gates for each task:
- **Fast gate:** Formatter, tests, and lint for the affected package only.
- **Dependency gate:** Fast gate checks, plus checks for shared API, manifest, dependency, or cross-package impact.
- **Final gate:** Complete workspace or project checks, plus explicit verification of every acceptance criterion.

The language-specific workflow files define which change types select each gate.

### 2. Multi-Pass Role-Based Code Review (`<lang>/REVIEW.md`)

A 3-stage review process that avoids shallow single-pass reviews by assigning specialized expert roles to different aspects of a diff. Review workflows currently exist for Rust (`rust/REVIEW.md`) and F# (`fs/REVIEW.md`):

- **Stage 1 — Plan Generation:** Deconstruct the diff into domain-specific tasks and assign a role to each based on the language's trigger heuristics (e.g., in Rust, `unsafe` blocks route to the Unsafe Soundness Expert and `async` + `Mutex` to the Concurrency Expert).
- **Stage 2 — Incremental Execution:** Load one role's guidelines at a time (JIT), inspect code through that role's lens, and append findings to an intermediate review file. Optional verification via language-specific tools (e.g., `cargo miri` or `loom` for Rust) or hardware docs.
- **Stage 3 — Final Synthesis:** Deduplicate findings, apply severity escalation rules, and produce a structured final review report with a verdict (`APPROVE` / `NEEDS_CHANGES` / `BLOCK`).

### 3. Architecture Change Proposal (`ARCHITECT.md`)

A 3-phase workflow that governs how AI agents create architecture change proposals in `docs/proposals/`:

- **Phase 1 — Intake and Clarification:** The user provides the main idea of the architecture change. The agent explores the current architecture, answers discoverable questions through codebase exploration, and asks the user to resolve only blocking questions.
- **Phase 2 — Propose Changes and Iterate:** The agent drafts the proposed change terms (scope, affected components, key decisions, trade-off summary, implementation approach) and presents them to the user. The agent iterates until the user agrees and requests no further improvements.
- **Phase 3 — Create the Proposal Document:** The agent writes a proposal file at `docs/proposals/<short_readable_name>.md` using a structured template (Summary, Motivation, Current Architecture, Proposed Changes, Trade-offs, Implementation Plan, Testing Strategy, Open Questions) and commits it to Git.

#### Proposal Lifecycle

Each proposal has exactly one of two states: `created` or `implemented`. A proposal lives in `docs/proposals/` only until it is implemented or deleted; there is no registry file. A `created` proposal may trigger the Planning and Execution Workflow (`WORKFLOW.md`) for implementation. When the associated plan is finalized (`WORKFLOW.md` Phase 5), the proposal's `Implemented` date is recorded and the file moves to `docs/proposals/done/<yymmdd>_<name>.md` with Status `implemented`, so the archive sorts by finalization date.

### Review Roles

Specialist roles are defined per language in `<lang>/review_roles/`. Each language ships a role set tailored to its idioms:

- **Rust** (`rust/review_roles/`): Unsafe Soundness Expert, Concurrency Expert, Principal Systems Architect, Embedded Baremetal Expert, Cross-Cutting Integration Expert.
- **F#** (`fs/review_roles/`): Principal Type-Driven Architect, Concurrency Expert, Cross-Cutting Integration Expert, Interop & Unsafe Expert, Performance & Allocation Expert.

Each role file contains: persona definition, 5-pillar inspection checklist, 4 anti-patterns with code examples, 4+ best practices with code examples, and severity assessment criteria.

### Writing Style Standard

All plans, reviews, and documentation produced under the agentic workflow follow [STYLE.md](STYLE.md). The standard defines ASD-STE100-inspired rules: active voice, one idea per sentence, maximum 25 words per sentence, present tense, one term per concept, and no vague qualifiers. Technical nomenclature (package names, type names, function names, tool names) is exempt.

## How to Use

### Setup

1. Copy the `agentic/` directory into your repository root.
2. Add a routing rule in your `AGENTS.md` (or equivalent agent configuration) that points to these workflow files. Example:

```markdown
| User Intent / Command | Required Action | Workflow File to Read |
| :--- | :--- | :--- |
| Feature Implementation / Bug Fix / Refactoring | Read and execute the Planning & Execution Workflow | agentic/WORKFLOW.md |
| Code Review / PR Audit / Diff Inspection | Read and execute the Multi-Pass Role-Based Review Workflow | agentic/<lang>/REVIEW.md |
| Architecture Extraction / Reverse Engineering | Read and execute the Architecture Extraction Guide | agentic/REVERSE.md |
| Architecture Change Proposal | Read and execute the Architecture Change Proposal Workflow | agentic/ARCHITECT.md |
| Documentation / Explanation / Questions about existing code | Answer directly using codebase exploration tools. Do NOT invoke a workflow. | N/A |
```

Phase 4 of the feature workflow delegates to language-specific files:

| Language | Language-Specific Workflow File |
| :--- | :--- |
| Rust | `agentic/rust/RUST_WORKFLOW.md` |
| Go | `agentic/go/GO_WORKFLOW.md` |
| Python | `agentic/python/PYTHON_WORKFLOW.md` |
| F# | `agentic/fs/FS_WORKFLOW.md` |

If the project's language has no dedicated workflow file, the agent asks the user for the exact format, test, lint, and dependency-check commands before proceeding with Phase 4.

3. Ensure your project has a `project/` directory at the root for plan files.

### Windsurf Setup

Windsurf (Codeium's IDE with the Cascade AI agent) supports global rules via a `.windsurfrules` file at the repository root and slash-command shortcuts via `.windsurf/workflows/`. To set up a Windsurf project to use the agentic workflow:

1. **Copy the `agentic/` directory** into your repository root (if not already present):
   ```bash
   cp -r agentic/ /path/to/your-repo/agentic/
   ```

2. **Generate the rules file** from the provided template:
   ```bash
   cp agentic/AGENTS.md /path/to/your-repo/.windsurfrules
   ```
   The template is language-agnostic; fill its `{{placeholder}}` markers with your project's values as described in [Customizing the Template](#customizing-the-template).
 
3. **Verify all placeholders are filled**:
   ```bash
   grep '{{' /path/to/your-repo/.windsurfrules && echo "Unfilled placeholders found!" || echo "All placeholders filled."
   ```

4. **Create slash-command shortcuts** in `.windsurf/workflows/` for quick access to the agentic workflows:
   ```bash
   mkdir -p /path/to/your-repo/.windsurf/workflows
   ```
   Create `.windsurf/workflows/feature.md`:
   ```markdown
   ---
   description: Implement a feature, fix a bug, or refactor code
   ---
   Read and execute the workflow defined in `agentic/WORKFLOW.md`.
   ```
   Create `.windsurf/workflows/review.md`:
   ```markdown
   ---
   description: Review code, audit a PR, or inspect a diff
   ---
   Read and execute the workflow defined in `agentic/<lang>/REVIEW.md`.
   ```
   Replace `<lang>` with your project's language directory (`rust`, `fs`, ...). These enable `/feature` and `/review` slash commands in the Windsurf chat panel.

5. **Create the `project/` directory** for plan files:
   ```bash
   mkdir -p /path/to/your-repo/project
   ```

6. **Commit everything** to your repository:
   ```bash
   cd /path/to/your-repo
   git add agentic/ .windsurfrules .windsurf/ project/
   git commit -m "Add agentic workflow with Windsurf integration"
   ```

Once set up, the Cascade agent in Windsurf will automatically read `.windsurfrules` as global rules and route feature requests to `agentic/WORKFLOW.md` and review requests to `agentic/<lang>/REVIEW.md`. The feature workflow auto-detects the project language and loads the appropriate language-specific workflow file for Phase 4. Review workflows are available for the languages that provide a `<lang>/REVIEW.md` file (currently Rust and F#). You can also use the `/feature` and `/review` slash commands to trigger the workflows explicitly.

### Triggering the Feature Workflow

Tell your AI agent to implement a feature, fix a bug, or refactor code. The agent will:
1. Read `agentic/WORKFLOW.md`.
2. Run preflight checks and recover an active plan if one exists (Phase 0).
3. Triage the request, handle ambiguities, and ask the user to resolve blocking items only (Phase 1).
4. Create a `project/PLAN_<name>.md` file (Phase 2).
5. Execute tasks one by one, write triad tests before implementation, and update the plan inline (Phase 3).
6. Select a verification gate, run language-specific commands, and commit each task atomically (Phase 4).
7. Run the Final gate, verify acceptance criteria, and archive the plan to `project/DONE_<name>.md` (Phase 5).

### Triggering the Review Workflow

Tell your AI agent to review code, audit a PR, or inspect a diff. The agent will:
1. Read `agentic/<lang>/REVIEW.md` for the project's language.
2. Generate a `REVIEW_PLAN.md` with role-assigned tasks (Stage 1).
3. Execute each task by loading the role's guidelines, inspecting code, and appending findings to `INTERMEDIATE_REVIEW.md` (Stage 2).
4. Synthesize a `FINAL_REVIEW.md` with deduplicated findings, severity escalation, and a final verdict (Stage 3).

### Triggering the Architecture Proposal Workflow

Tell your AI agent the main idea of an architecture change you want to make. The agent will:
1. Read `agentic/ARCHITECT.md`.
2. Explore the current architecture and ask you to resolve only blocking questions (Phase 1).
3. Draft the proposed change terms and present them for your review (Phase 2).
4. Iterate on the terms until you agree and request no further improvements.
5. Create a proposal document at `docs/proposals/<short_readable_name>.md` and commit it to Git (Phase 3).

### Customizing Roles

Review roles are defined per language in `<lang>/review_roles/`. Roles currently exist for Rust and F#; Go and Python review roles do not exist yet. Each role file is self-contained. To customize:
- **Add a new role:** Create a new `.md` file following the same structure (Persona, Checklist, Anti-Patterns, Best Practices, Severity Criteria), then add a routing row in that language's `REVIEW.md` Stage 1 table and a JIT loading instruction in Stage 2.
- **Modify an existing role:** Edit the corresponding `.md` file directly. All code examples use concrete types (not bare generics) to ensure correct markdown rendering.
- **Add roles for a new language:** Create a `<lang>/review_roles/` directory plus a `<lang>/REVIEW.md` file modeled on an existing one (e.g., `rust/REVIEW.md`).

## Generating Agent Rules for Your Project

The agent rules file is the entry point that routes your AI agent to the correct workflow. A language-agnostic template is provided at `agentic/AGENTS.md`:

- **Generic agents:** `agentic/AGENTS.md` → copy to repository root as `AGENTS.md`
- **Windsurf (Cascade):** `agentic/AGENTS.md` → copy to repository root as `.windsurfrules`

Copy the template to your repository root:
```bash
# For generic AI agents:
cp agentic/AGENTS.md ./AGENTS.md

# For Windsurf:
cp agentic/AGENTS.md ./.windsurfrules
```

### Customizing the Template

The template is language-agnostic. Replace every `{{placeholder}}` marker before use:

| Placeholder | Description | Examples |
|:---|:---|:---|
| `{{lang}}` | Language directory for the review workflow | `rust`, `fs` |
| `{{lint_command}}` | Lint command that must pass with warnings denied | `cargo clippy -- -D warnings`, `golangci-lint run`, `ruff check`, `dotnet fsharplint` |
| `{{format_command}}` | Format command | `cargo fmt --all`, `gofmt -w .`, `ruff format`, `fantomas .` |
| `{{check_command}}` | Full project check command | `cargo check --workspace`, `go build ./...`, `mypy .`, `dotnet build` |
| `{{suppression_syntax}}` | Warning-suppression syntax | `#[allow(...)]`, `//nolint`, `# noqa` / `# type: ignore`, `#nowarn` |
| `{{architecture_preferences}}` | Project architecture conventions | Type-State pattern, trait-based isolation |

If your project's language has no `agentic/<lang>/REVIEW.md` file, remove the review row from the routing table or point it at the language of your choice. Add or remove routing rows if you have additional workflows (e.g., deployment, migration).

## Directory Structure

```text
agentic/
├── WORKFLOW.md                   ← Feature implementation workflow (Phases 0–5)
├── ARCHITECT.md                  ← Architecture change proposal workflow (Phases 1–3)
├── REVERSE.md                    ← Architecture extraction / reverse engineering guide
├── STYLE.md                      ← Writing style standard (ASD-STE100-inspired)
├── README.md                     ← This file
├── AGENTS.md                     ← Language-agnostic template for project-root agent rules
├── LICENSE                       ← MIT license
├── rust/
│   ├── RUST_WORKFLOW.md          ← Rust-specific Phase 4 commands
│   ├── RUST_ARCHITECT.md         ← Rust-specific architecture extraction heuristics
│   ├── REVIEW.md                 ← Multi-pass role-based review workflow (3 stages)
│   └── review_roles/
│       ├── unsafe_expert.md      ← Unsafe Soundness Expert role
│       ├── concurrency.md        ← Concurrency Expert role
│       ├── architecture.md       ← Principal Systems Architect role
│       ├── embedded.md           ← Embedded Baremetal Expert role
│       └── cross_cutting.md      ← Cross-Cutting Integration Expert role
├── go/
│   └── GO_WORKFLOW.md            ← Go-specific Phase 4 commands
├── python/
│   └── PYTHON_WORKFLOW.md        ← Python-specific Phase 4 commands
└── fs/
    ├── FS_WORKFLOW.md            ← F#-specific Phase 4 commands
    ├── FS_ARCHITECT.md           ← F#/.NET architecture analysis specifics
    ├── REVIEW.md                 ← F# role-based review workflow
    └── review_roles/
        ├── architecture.md       ← Principal Type-Driven Architect role
        ├── concurrency.md        ← Concurrency Expert role
        ├── cross_cutting.md      ← Cross-Cutting Integration Expert role
        ├── interop.md            ← Interop & Unsafe Expert role
        └── performance.md        ← Performance & Allocation Expert role
```

> **Note:** The `agentic/` directory most likely has its own nested Git repository. It is therefore not tracked by the project's Git index. Run Git commands inside `agentic/` itself when working on workflow or style files.

## Requirements

### Feature Workflow

- Requires `git` and a `project/` directory at the repository root.
- Language-specific toolchain:
  - **Rust:** `cargo`, `rustc`, `clippy`
  - **Go:** `go`, `gofmt`, `golangci-lint` (optional but recommended)
  - **Python:** `python`, `ruff`, `mypy`, `pytest`
- If the project's language has no dedicated workflow file, the agent asks the user for the exact commands before proceeding with Phase 4.

### Review Workflow

- The review workflow exists for **Rust** (`rust/REVIEW.md`, requires `cargo`, `clippy`, `rustfmt`) and **F#** (`fs/REVIEW.md`).
- The Rust review workflow assumes the target code already passes `cargo clippy -- -D warnings` and `rustfmt`.

### Architecture Proposal Workflow

- Requires `git` and a `docs/proposals/` directory at the repository root.
- Requires `docs/ARCHITECTURE.md` as the reference for the current architecture. If absent, run the Architecture Extraction Workflow (`agentic/REVERSE.md`) first.

## Metadata

- **Author:** Alexey Sudachan
- **License:** MIT
- **Original GitHub Repo:** https://github.com/sudachen/agentic
