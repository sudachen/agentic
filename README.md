# Agentic Workflow

A structured AI agent operating framework for Systems/Baremetal Rust projects. It defines two disciplined, multi-phase workflows that constrain AI coding agents (e.g., Cascade, Devin, Cursor) into a predictable, high-quality engineering process — eliminating ad-hoc code generation, infinite fix loops, and shallow reviews.

## What It Does

The agentic workflow provides two core processes:

### 1. Feature Implementation & Engineering (`WORKFLOW.md`)

A 5-phase pipeline that governs how AI agents implement features, fix bugs, and refactor code:

- **Phase 1 — Clarify:** Identify ambiguities, propose improvements, and ask the user before writing any code.
- **Phase 2 — Plan:** Create a single `PLAN_<name>.md` file with feature summary, scope, architecture blueprint, testing strategy, ordered tasks, dependencies, and acceptance criteria.
- **Phase 3 — Execute:** Work through tasks one by one, writing triad tests (happy path, edge cases, negative cases) and updating the plan inline with execution logs.
- **Phase 4 — Verify & Commit:** Run `cargo fmt`, `cargo test`, `cargo clippy -- -D warnings`, and `cargo check --workspace` after every task. A circuit breaker halts after 2 failed fix attempts to prevent infinite loops. Each task ends with an atomic git commit.
- **Phase 5 — Archive:** Rename `PLAN_<name>.md` to `DONE_<name>.md` when all tasks are complete and acceptance criteria are met.

### 2. Multi-Pass Role-Based Code Review (`rust/REVIEW.md`)

A 3-stage review process that avoids shallow single-pass reviews by assigning specialized expert roles to different aspects of a diff:

- **Stage 1 — Plan Generation:** Deconstruct the diff into domain-specific tasks and assign a role to each based on trigger heuristics (e.g., `unsafe` → Unsafe Soundness Expert, `async` + `Mutex` → Concurrency Expert).
- **Stage 2 — Incremental Execution:** Load one role's guidelines at a time (JIT), inspect code through that role's lens, and append findings to an intermediate review file. Optional verification via `cargo miri`, `loom`, or hardware docs.
- **Stage 3 — Final Synthesis:** Deduplicate findings, apply severity escalation rules, and produce a structured final review report with a verdict (`APPROVE` / `NEEDS_CHANGES` / `BLOCK`).

### Review Roles

Five specialist roles are defined in `rust/review_roles/`:

| Role | File | Focus |
|:---|:---|:---|
| Unsafe Soundness Expert | `unsafe_expert.md` | Pointer aliasing, UB, FFI safety, `// SAFETY:` documentation |
| Concurrency Expert | `concurrency.md` | Atomic ordering, lock hierarchies, async cancellation, `.await` lock safety |
| Principal Systems Architect | `architecture.md` | Type-State pattern, API ergonomics, boundary isolation, error hierarchy |
| Embedded Baremetal Expert | `embedded.md` | `no_std`, MMIO volatile access, ISR synchronization, DMA buffer ownership |
| Cross-Cutting Integration Expert | `cross_cutting.md` | `Send`/`Sync` correctness, unsafe+async interactions, compositional soundness |

Each role file contains: persona definition, 5-pillar inspection checklist, 4 anti-patterns with code examples, 4+ best practices with code examples, and severity assessment criteria.

## How to Use

### Setup

1. Copy the `agentic/` directory into your repository root.
2. Add a routing rule in your `AGENTS.md` (or equivalent agent configuration) that points to these workflow files. Example:

```markdown
| User Intent / Command | Required Action | Workflow File to Read |
| :--- | :--- | :--- |
| Feature Implementation / Bug Fix / Refactoring | Read and execute the Planning & Execution Workflow | agentic/WORKFLOW.md |
| Code Review / PR Audit / Diff Inspection | Read and execute the Multi-Pass Role-Based Review Workflow | agentic/rust/REVIEW.md |
```

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

Once set up, the Cascade agent in Windsurf will automatically read `.windsurfrules` as global rules and route feature requests to `agentic/WORKFLOW.md` and review requests to `agentic/rust/REVIEW.md`. You can also use the `/feature` and `/review` slash commands to trigger the workflows explicitly.

### Triggering the Feature Workflow

Tell your AI agent to implement a feature, fix a bug, or refactor code. The agent will:
1. Read `agentic/WORKFLOW.md`.
2. Ask clarifying questions (Phase 1).
3. Create a `project/PLAN_<name>.md` file (Phase 2).
4. Execute tasks one by one with verification and commits (Phases 3–4).
5. Archive the plan to `project/DONE_<name>.md` (Phase 5).

### Triggering the Review Workflow

Tell your AI agent to review code, audit a PR, or inspect a diff. The agent will:
1. Read `agentic/rust/REVIEW.md`.
2. Generate a `REVIEW_PLAN.md` with role-assigned tasks (Stage 1).
3. Execute each task by loading the role's guidelines, inspecting code, and appending findings to `INTERMEDIATE_REVIEW.md` (Stage 2).
4. Synthesize a `FINAL_REVIEW.md` with deduplicated findings, severity escalation, and a final verdict (Stage 3).

### Customizing Roles

Each role file in `rust/review_roles/` is self-contained. To customize:
- **Add a new role:** Create a new `.md` file following the same structure (Persona, Checklist, Anti-Patterns, Best Practices, Severity Criteria), then add a routing row in `REVIEW.md`'s Stage 1 table and a JIT loading instruction in Stage 2.
- **Modify an existing role:** Edit the corresponding `.md` file directly. All code examples use concrete types (not bare generics) to ensure correct markdown rendering.

## Generating Agent Rules for Your Project

The agent rules file is the entry point that routes your AI agent to the correct workflow. Templates are provided with placeholders for project-specific values:

- **Generic agents:** `agentic/rust/AGENTS.md` → copy to repository root as `AGENTS.md`
- **Windsurf (Cascade):** `agentic/rust/AGENTS.md` → copy to repository root as `.windsurfrules`

Copy the appropriate template to your repository root:
```bash
# For generic AI agents:
cp agentic/rust/AGENTS.md ./AGENTS.md

# For Windsurf:
cp agentic/rust/AGENTS.md ./.windsurfrules
```

### Customizing the Template

The template covers the common case for Rust systems projects. You may need to adjust:
- **Routing table:** Add or remove rows if you have additional workflows (e.g., deployment, migration).
- **Code Quality section:** If your project uses different linters or formatters, update the commands accordingly.
- **Verification Quick Reference:** Replace the cargo commands with your project's build/test commands.
- **Architecture preferences:** Adjust the Type-State and trait-based guidance to match your project's conventions.

## Directory Structure

```text
agentic/
├── WORKFLOW.md                   ← Feature implementation workflow (5 phases)
├── README.md                     ← This file
└── rust/
    ├── REVIEW.md                 ← Multi-pass role-based review workflow (3 stages)
    ├── AGENTS.md                 ← Template for project-root AGENTS.md (generic agents)
    └── review_roles/
        ├── unsafe_expert.md      ← Unsafe Soundness Expert role
        ├── concurrency.md        ← Concurrency Expert role
        ├── architecture.md       ← Principal Systems Architect role
        ├── embedded.md           ← Embedded Baremetal Expert role
        └── cross_cutting.md      ← Cross-Cutting Integration Expert role
```

## Requirements

- The workflow is designed for **Rust** projects using `cargo` for build, test, and lint.
- The review workflow assumes the target code already passes `cargo clippy -- -D warnings` and `rustfmt`.
- The feature workflow requires `cargo`, `git`, and a `project/` directory at the repository root.

## Metadata

- **Author:** Alexey Sudachan
- **License:** MIT
- **Original GitHub Repo:** https://github.com/sudachen/agentic
