## Master AI Router & Operating Guidelines

### Core Operating Principle
You are an intelligent engineering agent working on this codebase. You MUST NOT execute tasks haphazardly.
Before starting any work, identify the intent of the user request and read the dedicated workflow document in the agentic/ folder.

### Context Recovery
Before starting any new task, check `project/` for existing `PLAN_*.md` files that may be in progress. If found, resume from the last incomplete task rather than starting over. Completed plans are archived as `DONE_*.md`.

### Routing Table

| User Intent / Command | Required Action | Workflow File to Read |
| :--- | :--- | :--- |
| Feature Implementation / Bug Fix / Refactoring / Code Generation | Read and execute the Planning & Execution Workflow | `agentic/AGENTIC.md` |
| Code Review / PR Audit / Diff Inspection | Read and execute the Multi-Pass Role-Based Review Workflow | `agentic/rust/REVIEW.md` |
| Documentation / Explanation / Questions about existing code | Answer directly using codebase exploration tools. Do NOT invoke a workflow. | N/A |

When a workflow is assigned, follow its phases strictly. Do NOT skip phases even for "small" changes.

---

## Project Context

- **Language:** {{LANGUAGE}} (edition {{EDITION}})
- **Workspace:** {{WORKSPACE_DESCRIPTION}}
- **Purpose:** {{PROJECT_PURPOSE}}
- **Key dependencies:** {{KEY_DEPENDENCIES}}
- **Target:** {{TARGET_TRIPLE}} ({{TARGET_NOTES}})
- **Release profile:** {{RELEASE_PROFILE}}
- **Plan files:** Active plans in `project/PLAN_*.md`, archived in `project/DONE_*.md`
- **Agentic workflows:** See `agentic/README.md` for full documentation

---

## Baseline Project Constraints

### Code Quality
- **Clippy:** Code MUST pass `cargo clippy -- -D warnings` before completing any task.
- **Formatting:** Code MUST pass `cargo fmt --all`.
- **Workspace check:** `cargo check --workspace` must pass (downstream dependency safety).
- **No suppression:** Do NOT suppress warnings with `#[allow(...)]` unless there is a justified reason documented in the plan file.

### Architecture
- Prefer compile-time state invariants (Type-State Pattern) over runtime state flags.
- Prefer zero-cost abstractions and trait-based boundary isolation.

### Testing
- Every non-trivial code modification MUST include automated tests covering the Testing Triad:
  - **Happy Path:** Expected successful execution under standard conditions.
  - **Edge Cases:** Boundary values, empty inputs, maximum capacity limits.
  - **Negative Cases:** Invalid inputs, missing dependencies, expected `Err` or panic states.
- Do NOT delete or weaken existing tests without explicit user direction.

### Commit Discipline
- Create atomic git commits after every verified task: `git commit -am "<task summary>"`.
- Do NOT push to remote unless explicitly asked.
- Do NOT amend or rebase existing commits unless explicitly asked.

---

## Verification Quick Reference

```bash
cargo fmt --all
cargo test -p {{CRATE_NAME}}
cargo clippy -p {{CRATE_NAME}} -- -D warnings
cargo check --workspace
```

---

## Prohibited Actions

- **Do NOT** write code, review code, or create files before reading the assigned workflow document.
- **Do NOT** create files outside the task scope (no helper scripts, no random test files).
- **Do NOT** add comments, docstrings, or documentation unless explicitly asked by the user.
- **Do NOT** run destructive commands (`rm -rf`, `git push --force`, `git reset --hard`, etc.) without explicit user approval.
- **Do NOT** skip workflow phases even for "small" or "trivial" changes.
- **Do NOT** delete or weaken existing tests without explicit user direction.
- **Do NOT** suppress compiler warnings with `#[allow(...)]` without documented justification.
- **Do NOT** proceed past 2 failed fix attempts (circuit breaker — stop and ask the user).
