# Agentic Workflow

A structured AI agent operating framework for multi-language system and embedded software development. It defines two disciplined, multi-phase workflows that constrain AI coding agents (e.g., Cascade, Devin, Cursor) into a predictable, high-quality engineering process — eliminating ad-hoc code generation, infinite fix loops, and shallow reviews.

The framework supports **Rust**, **Go**, and **Python** through language-specific workflow files. Each language file provides the exact format, test, lint, and dependency-check commands for Phase 4 verification. Rust has the most complete coverage, including a dedicated multi-pass code review workflow with specialist roles.

## What It Does

The agentic workflow provides two core processes:

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

### 2. Multi-Pass Role-Based Code Review (`rust/REVIEW.md`)

A 3-stage review process that avoids shallow single-pass reviews by assigning specialized expert roles to different aspects of a diff:

- **Stage 1 — Plan Generation:** Deconstruct the diff into domain-specific tasks and assign a role to each based on trigger heuristics (e.g., `unsafe` → Unsafe Soundness Expert, `async` + `Mutex` → Concurrency Expert).
- **Stage 2 — Incremental Execution:** Load one role's guidelines at a time (JIT), inspect code through that role's lens, and append findings to an intermediate review file. Optional verification via `cargo miri`, `loom`, or hardware docs.
- **Stage 3 — Final Synthesis:** Deduplicate findings, apply severity escalation rules, and produce a structured final review report with a verdict (`APPROVE` / `NEEDS_CHANGES` / `BLOCK`).

### Review Roles

The review workflow is **Rust-specific**. Five specialist roles are defined in `rust/review_roles/`:

| Role | File | Focus |
|:---|:---|:---|
| Unsafe Soundness Expert | `unsafe_expert.md` | Pointer aliasing, UB, FFI safety, `// SAFETY:` documentation |
| Concurrency Expert | `concurrency.md` | Atomic ordering, lock hierarchies, async cancellation, `.await` lock safety |
| Principal Systems Architect | `architecture.md` | Type-State pattern, API ergonomics, boundary isolation, error hierarchy |
| Embedded Baremetal Expert | `embedded.md` | `no_std`, MMIO volatile access, ISR synchronization, DMA buffer ownership |
| Cross-Cutting Integration Expert | `cross_cutting.md` | `Send`/`Sync` correctness, unsafe+async interactions, compositional soundness |

Each role file contains: persona definition, 5-pillar inspection checklist, 4 anti-patterns with code examples, 4+ best practices with code examples, and severity assessment criteria.

### Writing Style Standard

All plans, reviews, and documentation produced under the agentic workflow follow [STYLE.md](STYLE.md). The standard defines ASD-STE100-inspired rules: active voice, one idea per sentence, maximum 25 words per sentence, present tense, one term per concept, and no vague qualifiers. Technical nomenclature (crate names, type names, function names, tool names) is exempt.

## How to Use

### Setup

1. Copy the `agentic/` directory into your repository root.
2. Add a routing rule in your `AGENTS.md` (or equivalent agent configuration) that points to these workflow files. Example:

```markdown
| User Intent / Command | Required Action | Workflow File to Read |
| :--- | :--- | :--- |
| Feature Implementation / Bug Fix / Refactoring | Read and execute the Planning & Execution Workflow | agentic/WORKFLOW.md |
| Code Review / PR Audit / Diff Inspection | Read and execute the Multi-Pass Role-Based Review Workflow | agentic/rust/REVIEW.md |
| Documentation / Explanation / Questions about existing code | Answer directly using codebase exploration tools. Do NOT invoke a workflow. | N/A |
```

Phase 4 of the feature workflow delegates to language-specific files:

| Language | Language-Specific Workflow File |
| :--- | :--- |
| Rust | `agentic/rust/RUST_WORKFLOW.md` |
| Go | `agentic/go/GO_WORKFLOW.md` |
| Python | `agentic/python/PYTHON_WORKFLOW.md` |

If the project's language has no dedicated workflow file, the agent asks the user for the exact format, test, lint, and dependency-check commands before proceeding with Phase 4.

3. Ensure your project has a `project/` directory at the root for plan files.

### Windsurf Setup

Windsurf (Codeium's IDE with the Cascade AI agent) supports global rules via a `.windsurfrules` file at the repository root and slash-command shortcuts via `.windsurf/workflows/`. To set up a Windsurf project to use the agentic workflow:

1. **Copy the `agentic/` directory** into your repository root (if not already present):
   ```bash
   cp -r agentic/ /path/to/your-repo/agentic/
   ```

2. **Generate the rules file** from the Windsurf template:
   ```bash
   cp agentic/rust/AGENTS.md /path/to/your-repo/.windsurfrules
   ```
 
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
   Read and execute the workflow defined in `agentic/rust/REVIEW.md`.
   ```
   These enable `/feature` and `/review` slash commands in the Windsurf chat panel.

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

Once set up, the Cascade agent in Windsurf will automatically read `.windsurfrules` as global rules and route feature requests to `agentic/WORKFLOW.md` and review requests to `agentic/rust/REVIEW.md`. The feature workflow auto-detects the project language and loads the appropriate language-specific workflow file for Phase 4. The review workflow is Rust-specific. You can also use the `/feature` and `/review` slash commands to trigger the workflows explicitly.

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
1. Read `agentic/rust/REVIEW.md`.
2. Generate a `REVIEW_PLAN.md` with role-assigned tasks (Stage 1).
3. Execute each task by loading the role's guidelines, inspecting code, and appending findings to `INTERMEDIATE_REVIEW.md` (Stage 2).
4. Synthesize a `FINAL_REVIEW.md` with deduplicated findings, severity escalation, and a final verdict (Stage 3).

### Customizing Roles

The review roles are **Rust-specific**. Go and Python review roles do not exist yet. Each role file in `rust/review_roles/` is self-contained. To customize:
- **Add a new role:** Create a new `.md` file following the same structure (Persona, Checklist, Anti-Patterns, Best Practices, Severity Criteria), then add a routing row in `REVIEW.md`'s Stage 1 table and a JIT loading instruction in Stage 2.
- **Modify an existing role:** Edit the corresponding `.md` file directly. All code examples use concrete types (not bare generics) to ensure correct markdown rendering.

## Generating Agent Rules for Your Project

The agent rules file is the entry point that routes your AI agent to the correct workflow. A template is provided for Rust projects:

- **Generic agents:** `agentic/rust/AGENTS.md` → copy to repository root as `AGENTS.md`
- **Windsurf (Cascade):** `agentic/rust/AGENTS.md` → copy to repository root as `.windsurfrules`

Copy the template to your repository root:
```bash
# For generic AI agents:
cp agentic/rust/AGENTS.md ./AGENTS.md

# For Windsurf:
cp agentic/rust/AGENTS.md ./.windsurfrules
```

### Customizing the Template

The template covers the common case for Rust systems projects. For Go or Python projects, adjust the following sections:
- **Routing table:** Add or remove rows if you have additional workflows (e.g., deployment, migration).
- **Code Quality section:** Replace the Rust commands (`cargo clippy`, `cargo fmt`) with the appropriate language commands (`go vet`, `golangci-lint` for Go; `ruff`, `mypy` for Python).
- **Architecture preferences:** Replace the Type-State and trait-based guidance with patterns appropriate for your language.
- **Suppression rules:** Replace `#[allow(...)]` with the language-specific suppression syntax (`//nolint` for Go; `# noqa` or `# type: ignore` for Python).

For Rust projects, you may also need to adjust:
- **Verification Quick Reference:** Replace the cargo commands with your project's build/test commands.
- **Architecture preferences:** Adjust the Type-State and trait-based guidance to match your project's conventions.

## Directory Structure

```text
agentic/
├── WORKFLOW.md                   ← Feature implementation workflow (Phases 0–5)
├── STYLE.md                      ← Writing style standard (ASD-STE100-inspired)
├── README.md                     ← This file
├── rust/
│   ├── RUST_WORKFLOW.md          ← Rust-specific Phase 4 commands
│   ├── REVIEW.md                 ← Multi-pass role-based review workflow (3 stages)
│   ├── AGENTS.md                 ← Template for project-root agent rules
│   └── review_roles/
│       ├── unsafe_expert.md      ← Unsafe Soundness Expert role
│       ├── concurrency.md        ← Concurrency Expert role
│       ├── architecture.md       ← Principal Systems Architect role
│       ├── embedded.md           ← Embedded Baremetal Expert role
│       └── cross_cutting.md      ← Cross-Cutting Integration Expert role
├── go/
│   └── GO_WORKFLOW.md            ← Go-specific Phase 4 commands
└── python/
    └── PYTHON_WORKFLOW.md        ← Python-specific Phase 4 commands
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

- The review workflow is **Rust-specific** and requires `cargo`, `clippy`, and `rustfmt`.
- The review workflow assumes the target code already passes `cargo clippy -- -D warnings` and `rustfmt`.

## Metadata

- **Author:** Alexey Sudachan
- **License:** MIT
- **Original GitHub Repo:** https://github.com/sudachen/agentic
