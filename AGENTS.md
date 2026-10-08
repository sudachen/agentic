## Master AI Router & Operating Guidelines

### Core Operating Principle
You are an intelligent engineering agent working on this codebase. You MUST NOT execute tasks haphazardly.
Before starting any work, identify the intent of the user request and read the dedicated workflow document in the agentic/ folder.

### Context Recovery
Before starting any new task, check `project/` for existing `PLAN_*.md` files that may be in progress. If found, resume from the last incomplete task rather than starting over. Completed plans are archived as `DONE_*.md`.

### Routing Table

| User Intent / Command | Required Action | Workflow File to Read |
| :--- | :--- | :--- |
| Feature Implementation / Bug Fix / Refactoring / Code Generation | Read and execute the Planning & Execution Workflow | `agentic/WORKFLOW.md` |
| Code Review / PR Audit / Diff Inspection | Read and execute the Multi-Pass Role-Based Review Workflow | `agentic/{{lang}}/REVIEW.md` |
| Architecture Extraction / Reverse Engineering / System Map | Read and execute the Architecture Extraction Guide | `agentic/REVERSE.md` |
| Architecture Change Proposal | Read and execute the Architecture Change Proposal Workflow | `agentic/ARCHITECT.md` |
| Documentation / Explanation / Questions about existing code | Answer directly using codebase exploration tools. Do NOT invoke a workflow. | N/A |

When a workflow is assigned, follow its phases strictly. Do NOT skip phases even for "small" changes.

---

## Baseline Project Constraints

### Code Quality
- **Lint:** Code MUST pass `{{lint_command}}` before completing any task.
- **Formatting:** Code MUST pass `{{format_command}}`.
- **Full check:** `{{check_command}}` must pass (downstream dependency safety).
- **No suppression:** Do NOT suppress warnings with `{{suppression_syntax}}` unless there is a justified reason documented in the plan file.

### Architecture
- Prefer compile-time state invariants over runtime state flags where the language supports them.
- Prefer boundary isolation through explicit interfaces or module boundaries.
- {{architecture_preferences}}

### Testing
- Every non-trivial code modification MUST include automated tests covering the Testing Triad:
  - **Happy Path:** Expected successful execution under standard conditions.
  - **Edge Cases:** Boundary values, empty inputs, maximum capacity limits.
  - **Negative Cases:** Invalid inputs, missing dependencies, expected error or panic states.
- Do NOT delete or weaken existing tests without explicit user direction.

### Commit Discipline
- Create atomic git commits after every verified task, staging only the task's declared files and following the staging and validation rules in `agentic/WORKFLOW.md` Phase 4.6. Do NOT use `git commit -am`, which stages unrelated changes.
- Do NOT push to remote unless explicitly asked.
- Do NOT amend or rebase existing commits unless explicitly asked.

---

## Prohibited Actions

- **Do NOT** write code, review code, or create files before reading the assigned workflow document.
- **Do NOT** create files outside the task scope (no helper scripts, no random test files).
- **Do NOT** add comments, docstrings, or documentation unless explicitly asked by the user.
- **Do NOT** run destructive commands (`rm -rf`, `git push --force`, `git reset --hard`, etc.) without explicit user approval.
- **Do NOT** skip workflow phases even for "small" or "trivial" changes.
- **Do NOT** delete or weaken existing tests without explicit user direction.
- **Do NOT** suppress warnings with `{{suppression_syntax}}` without documented justification.
- **Do NOT** proceed past 2 failed fix attempts (circuit breaker — stop and ask the user).

## Metadata

- **Author:** Alexey Sudachan
- **License:** MIT
- **Original GitHub Repo:** https://github.com/sudachen/agentic
