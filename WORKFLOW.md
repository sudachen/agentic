# WORKFLOW.md

This document defines the strict workflow for implementing features, bug fixes, and refactors in this project.
Every change must follow these phases in order to ensure quality, avoid infinite loops, and maintain a clean history.

## Plan File Location

Plan files live in the `project/` directory at the repository root.
Git must not track plan files. Plan files exist only on the local machine.
Users exclude `project/` through a global ignore file, such as `~/.config/git/ignore`.
Fresh-context sessions rely on these local files for context recovery.

## Change Tiers

Classify each request before Phase 1.

- **Minor change:** Touches one file and changes fewer than 50 lines. Uses a reduced plan with only Summary and Task List. Phases 3 and 4 still apply in full.
- **Standard change:** Everything else. Follows all phases with the complete plan structure.

---

## Writing Style Standard

> **Note:** All plans and documentation under this workflow follow [STYLE.md](STYLE.md).

---

## Phase 0 — Resume In-Progress Plans

Before starting any new work, list the `project/` directory for in-progress `PLAN_*.md` files.
If one exists, resume from its last incomplete task.
Only one plan may be in progress at a time. Do not create a new plan while another plan is active.

---

## Phase 1 — Clarify Ambiguities and Propose Improvements

Before writing any code, the agent must:

- **Identify ambiguities** in the feature request — unclear requirements, missing edge cases, undefined behavior, conflicting constraints.
- **Identify design issues** — potential architectural concerns, performance implications, security risks, or maintainability problems.
- **Propose improvements** — suggest better alternatives, simpler approaches, or critical consequences the user may not have considered.
- **Ask the user** to clarify each ambiguity or choose between proposed options.

Do **not** proceed to planning until all ambiguities are resolved.
Record every resolved clarification and decision in the plan's Feature Summary and Scope sections.
Fresh-context sessions rely on this record.

---

## Phase 2 — Create a Single Plan File

Create a single plan and tracking file in the `project/` directory at the repository root.

- **Naming convention:** `PLAN_<short_readable_name>.md`
  - Example: `PLAN_analytics_integration.md`
- **Workflow Reference:** The file MUST start with a blockquote linking back to this document:
  `> **Note:** This plan is executed according to the strict workflow rules defined in [WORKFLOW.md](agentic/WORKFLOW.md).`
- **Link base:** Write relative links in plan files with the repository root as the base.
- The plan file must contain the following sections:
  1. **Feature Summary:** One-paragraph description of what is being built.
  2. **Scope:** What is included and explicitly excluded. 
  3. **Architecture & Technical Blueprint:** High-level architectural decisions, data structures, type signatures, and module boundaries. Must provide enough detail for a fresh AI session to understand the overall design without prior chat context. 
  4. **Testing Strategy:** Required test coverage (Happy path, Edge cases, Failure modes, Property/Snapshot tests if applicable). 
  5. **Execution Tasks:** Ordered list of tasks with detailed specifications and tracking logs (see Phase 3). 
  6. **Dependencies:** Packages, modules, or external systems affected. 
  7. **Acceptance Criteria:** How to know the feature is complete (including passing test criteria). 
  8. **Discovered Gotchas & Constraints:** A running log of unexpected build/toolchain constraints, architectural rules, or unwritten project conventions discovered during execution.

After Phase 2, each task's Specification is fixed.
Amend it only in place, with a dated note describing the deviation.
Changes to the plan's scope require explicit user re-approval.

---

## Phase 3 — Execute Tasks, Write Triad Tests, and Track Inline

For each task in the `PLAN_<name>.md` file, execute the work and immediately update the file inline.

Execute tasks in list order.
Reorder only when tasks are independent, and record the reordering in the Execution Log.
Do not skip a task without explicit user direction.

Write the tests from the Triad Test Plan before writing the implementation code.

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
    - **Exposed APIs & Artifacts:** New public types, functions, modules, or interfaces added (serves as reference for subsequent tasks in fresh context).
    - **Verification:** Test and lint results.
```

When a task is completed, change `[ ]` to `[x]`, update the status, and document the changes.

Status rules:
- Only the circuit breaker may set `blocked`. A `blocked` task resumes when the user provides guidance.
- Only the user may direct a task to `skipped`.

### The Testing Triad Mandate

Every non-trivial code modification MUST include automated tests covering:
- **Happy Path:** Expected successful execution under standard conditions.
- **Edge Cases:** Boundary values, empty inputs, maximum capacity limits, unusual unicode, etc.
- **Negative Cases:** Invalid inputs, missing dependencies, I/O failures, expected Err or Panic states.

---

## Phase 4 — Verify & Commit After Every Task

Every task **must** end with verification. No task is considered complete until all checks pass.

The exact commands for formatting, tests, and lint are language-specific. Read the workflow file for the project's language before running Phase 4:

| Language | Language-Specific Workflow File |
| :--- | :--- |
| Rust | [rust/RUST_WORKFLOW.md](rust/RUST_WORKFLOW.md) |
| Go | [go/GO_WORKFLOW.md](go/GO_WORKFLOW.md) |
| Python | [python/PYTHON_WORKFLOW.md](python/PYTHON_WORKFLOW.md) |

If the project's language has no dedicated workflow file, ask the user for the exact format, test, lint, and dependency-check commands before proceeding with Phase 4.

### 4.1 Code Formatting
Run the formatter command defined in the language-specific workflow file.

### 4.2 Run Tests
Run tests for the **affected packages** — not the entire codebase unless the change is codebase-wide. Use the test command defined in the language-specific workflow file.

### 4.3 Run Lint
Run the linter on the **affected packages** only. Use the lint command defined in the language-specific workflow file.

### 4.4 Run Full Build/Dependency Check
Ensure local changes do not break downstream dependencies. Use the check command defined in the language-specific workflow file.

### 4.5 Fix Problems (Circuit Breaker)
- If any check fails, fix the issue before proceeding.
- **CIRCUIT BREAKER:** One fix attempt is one verify-fix-verify cycle on a single task. If test or lint failures are not resolved after **2 consecutive fix attempts** on the same task, the agent MUST stop. Mark the task as `blocked` in the plan, document the error in the "Gotchas" section, and ask the user for guidance to prevent infinite fix loops.
- Do not suppress linter warnings unless there is a justified reason documented in the plan file. See the language-specific workflow file for the exact suppression syntax to avoid.

### 4.6 Atomic Git Commit
Add newly created files to the git commit.
Never commit plan files. The `project/` directory is local-only and git does not track it:
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

Before archiving, run the full check suite on the entire workspace, not only the affected packages.
Verify every acceptance criterion explicitly.

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
│ Phase 0: Resume In-Progress Plans       │  ← Resume or start fresh
└────────────────────┬────────────────────┘
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
├── agentic/
│   ├── WORKFLOW.md                       ← this file
│   ├── STYLE.md                          ← writing style standard
│   ├── rust/
│   │   ├── RUST_WORKFLOW.md              ← Rust-specific Phase 4 commands
│   │   └── REVIEW.md                     ← code review workflow
│   ├── go/
│   │   └── GO_WORKFLOW.md                ← Go-specific Phase 4 commands
│   └── python/
│       └── PYTHON_WORKFLOW.md            ← Python-specific Phase 4 commands
├── project/
│   ├── PLAN_analytics_integration.md    ← active feature plan
│   └── DONE_dpi_rule_editor.md          ← completed, archived plan
└── ...
```
