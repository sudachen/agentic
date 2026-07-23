# WORKFLOW.md

This document defines the strict workflow for implementing new features in this project.
Every new feature must follow these phases in order to ensure quality, avoid infinite loops, and maintain a clean history.

---

## Phase 1 — Clarify Ambiguities and Propose Improvements

Before writing any code, the agent must:

- **Identify ambiguities** in the feature request — unclear requirements, missing edge cases, undefined behavior, conflicting constraints.
- **Identify design issues** — potential architectural concerns, performance implications, security risks, or maintainability problems.
- **Propose improvements** — suggest better alternatives, simpler approaches, or critical consequences the user may not have considered.
- **Ask the user** to clarify each ambiguity or choose between proposed options.

Do **not** proceed to planning until all ambiguities are resolved.

---

## Phase 2 — Create a Single Plan File

Create a single plan and tracking file in the `project/` directory at the repository root.

- **Naming convention:** `PLAN_<short_readable_name>.md`
  - Example: `PLAN_analytics_integration.md`
- **Workflow Reference:** The file MUST start with a blockquote linking back to this document:
  `> **Note:** This plan is executed according to the strict workflow rules defined in [WORKFLOW.md](agentic/WORKFLOW.md).`
- The plan file must contain the following sections:
  1. **Feature Summary:** One-paragraph description of what is being built.
  2. **Scope:** What is included and explicitly excluded. 
  3. **Architecture & Technical Blueprint:** High-level architectural decisions, data structures, type signatures, and module boundaries. Must provide enough detail for a fresh AI session to understand the overall design without prior chat context. 
  4. **Testing Strategy:** Required test coverage (Happy path, Edge cases, Failure modes, Property/Snapshot tests if applicable). 
  5. **Execution Tasks:** Ordered list of tasks with detailed specifications and tracking logs (see Phase 3). 
  6. **Dependencies:** Crates, modules, or external systems affected. 
  7. **Acceptance Criteria:** How to know the feature is complete (including passing test criteria). 
  8. **Discovered Gotchas & Constraints:** A running log of unexpected compiler constraints, architectural rules, or unwritten project conventions discovered during execution.

---

## Phase 3 — Execute Tasks, Write Triad Tests, and Track Inline

For each task in the `PLAN_<name>.md` file, execute the work and immediately update the file inline.

To ensure resilience against context window resets ("fresh context problem"), every task MUST be split into a Specification (defined during Phase 2) and an Execution Log (filled during Phases 3–4).

Each item in the "Execution Tasks" section must use this format:

```markdown
- [ ] **Task <N>: <Short description>**
  - **Specification (Defined in Phase 2):**
    - **Goal:** Clear definition of what needs to be accomplished and why.
    - **Files to touch:** List of files created or modified.
    - **Technical Details & Contracts:** Data structures, API signatures, and core algorithms.
    - **Triad Test Plan:** Specific happy path, edge case, and negative test scenarios to write.
  - **Execution Log (Updated in Phases 3-4):**
    - **Status:** `pending` | `in_progress` | `completed` | `blocked` | `skipped`
    - **Date:** ISO 8601 timestamp of completion.
    - **Changes & Decisions:** Key technical decisions, trade-offs, and deviations from specification.
    - **Exposed APIs & Artifacts:** New public structs, functions, modules, or traits added (serves as reference for subsequent tasks in fresh context).
    - **Verification:** Test and Clippy results.
```

When a task is completed, change `[ ]` to `[x]`, update the status, and document the changes.

### The Testing Triad Mandate

Every non-trivial code modification MUST include automated tests covering:
- **Happy Path:** Expected successful execution under standard conditions.
- **Edge Cases:** Boundary values, empty inputs, maximum capacity limits, unusual unicode, etc.
- **Negative Cases:** Invalid inputs, missing dependencies, I/O failures, expected Err or Panic states.

---

## Phase 4 — Verify & Commit After Every Task

Every task **must** end with verification. No task is considered complete until all checks pass.

### 4.1 Code Formatting
Ensure the code is properly formatted:
```bash
cargo fmt --all
```

### 4.2 Run Tests (Targets, Features & Docs)
Run tests only for the **affected crates** — not the entire workspace unless the change is workspace-wide:
```bash
cargo test -p <crate_name>
```

### 4.3 Run Clippy
Run clippy on the **affected crates** only:
```bash
cargo clippy -p <crate_name> -- -D warnings
```

### 4.4 Run Workspace Check
Ensure local crate changes do not break downstream dependencies:
```bash
cargo check --workspace
```

### 4.5 Fix Problems (Circuit Breaker)
- If any check fails, fix the issue before proceeding.
- **CIRCUIT BREAKER:** If test or clippy failures are not resolved after **2 fix attempts**, the agent MUST stop. Mark the task as `blocked` in the plan, document the error in the "Gotchas" section, and ask the user for guidance to prevent infinite fix loops.
- Do not suppress warnings with `#[allow(...)]` unless there is a justified reason documented in the plan file.

### 4.6 Atomic Git Commit
Add newly created files to the git commit (except files in the project directory):
```bash
git add <new_files>
```
Once all verification checks pass for a task, create a git commit immediately:
```bash
git commit -am "<task summary>"
```

Continue to the next Task in the plan.

---

## Phase 5 — Finalize and Archive Plan

Once all execution tasks are marked as completed `[x]` and all acceptance criteria are met, the plan is considered finished.

Rename the plan file: Change the prefix from `PLAN_` to `DONE_` to archive it:
```bash
mv project/PLAN_<name>.md project/DONE_<name>.md
```

---

## Workflow Summary

```text
Feature Request
      │
      ▼
┌─────────────────────────────────────────┐
│ Phase 1: Clarify Ambiguities            │  ← Ask user before writing plan
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐
│ Phase 2: Create PLAN_<name>.md          │  ← Defines Tasks, Scope & Tests
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐ ◄──────────┐
│ Phase 3: Execute Task N & Triad Tests   │            │
└────────────────────┬────────────────────┘            │ Loop for
                     ▼                                 │ each Task
┌─────────────────────────────────────────┐            │
│ Phase 4: Verify & Commit Task N         │ ───────────┘
│          (Circuit Breaker: 2 attempts)  │
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐
│ Phase 5: Finalize & Archive Plan        │  ← Rename to DONE_<name>.md
└─────────────────────────────────────────┘
```

---

## Directory Structure

```text
k4dpi/
├── agentic /
│   └── WORKFLOW.md                       ← this file
├── project/
│   ├── PLAN_analytics_integration.md    ← active feature plan
│   └── DONE_dpi_rule_editor.md          ← completed, archived plan
└── ...
```
