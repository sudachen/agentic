## Overview & Core Principles
This document defines the agentic workflow for conducting multi-pass, role-based code reviews in F#/.NET codebases.

> **Note:** All review documents produced under this workflow follow [STYLE.md](../STYLE.md).

The review process MUST NOT be executed in a single broad pass. Instead, it operates strictly in three sequential stages:
1. **Plan Generation** (`REVIEW_PLAN.md`) references [REVIEW.md](agentic/fs/REVIEW.md).
2. **Role-Based Incremental Execution** (`INTERMEDIATE_REVIEW.md`) references `REVIEW_PLAN.md` and [REVIEW.md](agentic/fs/REVIEW.md)
3. **Final Synthesis** (`FINAL_REVIEW.md`)

---

## Baseline Assumptions
- The target code **ALREADY passes** `dotnet build --warnaserror`, `fantomas`, and `fsharplint`.
- **DO NOT** report basic syntax issues, formatting, or micro-style lints.
- Act strictly as a high-level Principal Engineer, focusing on type-driven design, async correctness, interop safety, allocation discipline, and architecture.

---

## STAGE 1: Review Plan Generation (`REVIEW_PLAN.md`)

1. Inspect the provided `git diff` or changed files in the workspace.
2. Create a `REVIEW_PLAN.md` file in the workspace root.
3. Deconstruct the changes into targeted, domain-specific tasks.
4. **Assign a Specific Role** to each task based on the following heuristics:

| Diff Trigger / Modifiers | Assigned Role | Role Guideline File |
| :--- | :--- | :--- |
| `NativePtr`, `fixed`, `[<DllImport>]`, `Marshal`, `GCHandle`, `SafeHandle`, `unmanaged`, `nativeint`, `void*` | `[Role: Interop & Unsafe Expert]` | `agentic/fs/review_roles/interop.md` |
| `async`/`task` CEs, `MailboxProcessor`, `lock`, `Monitor`, `Interlocked`, `SemaphoreSlim`, `Channel`, `CancellationToken` | `[Role: Concurrency Expert]` | `agentic/fs/review_roles/concurrency.md` |
| Discriminated unions, records, `.fsi` signature files, modules, interfaces, `Result`/`Option` error modeling, public APIs | `[Role: Principal Type-Driven Architect]` | `agentic/fs/review_roles/architecture.md` |
| `Span`/`Memory`, `[<Struct>]`, `byref`/`inref`, `voption`, recursion depth, `seq`/`List`/`Array` in hot paths, boxing | `[Role: Performance & Allocation Expert]` | `agentic/fs/review_roles/performance.md` |
| `async`/`task` + `IDisposable`/interop in same scope, `Async.AwaitTask`/`StartAsTask`/`StartImmediate`, `SynchronizationContext`, locks composed with agents | `[Role: Cross-Cutting Integration Expert]` | `agentic/fs/review_roles/cross_cutting.md` |

5. Format every task with an uncompleted checkbox `[ ]`.

### Multi-Role Task Handling
If a single diff hunk triggers multiple role heuristics (e.g., a `fixed` block used inside a `task` CE, or a `MailboxProcessor` guarding shared mutable state), create **separate tasks** for each role on the same code region. Each role examines the same code through its own lens independently. This ensures no perspective is missed due to premature role convergence.

### Large Diff Batching
If the diff exceeds **20 files**, group related files into sub-tasks per module or functional area rather than creating one task per file. This keeps the plan focused and prevents the review from losing context across an oversized task list.

### Example `REVIEW_PLAN.md` Format:
```markdown
# Review Plan

- [ ] **[Role: Interop & Unsafe Expert]** Audit `fixed` pinning and `NativePtr` lifetime in `src/io/native_buffer.fs`
- [ ] **[Role: Concurrency Expert]** Check cancellation propagation and `SemaphoreSlim` usage in `src/net/worker.fs`
- [ ] **[Role: Principal Type-Driven Architect]** Evaluate discriminated-union state modeling in `src/domain/connection.fs`
- [ ] **[Role: Performance & Allocation Expert]** Verify tail recursion and `voption` usage in `src/parsing/scanner.fs`
- [ ] **[Role: Cross-Cutting Integration Expert]** Audit `IDisposable` lifetime across `let!` points in `src/net/channel.fs`
```

---

## STAGE 2: Incremental Role Execution (`INTERMEDIATE_REVIEW.md`)

Execute tasks from `REVIEW_PLAN.md` ONE BY ONE. Do not analyze multiple tasks simultaneously.

For each task:

**Step 2.1: Load Just-In-Time (JIT) Role Context**
Before inspecting any code, you MUST read the assigned role's guideline file using your file-reading tool:
- For [Role: Interop & Unsafe Expert] -> Read `agentic/fs/review_roles/interop.md`
- For [Role: Concurrency Expert] -> Read `agentic/fs/review_roles/concurrency.md`
- For [Role: Principal Type-Driven Architect] -> Read `agentic/fs/review_roles/architecture.md`
- For [Role: Performance & Allocation Expert] -> Read `agentic/fs/review_roles/performance.md`
- For [Role: Cross-Cutting Integration Expert] -> Read `agentic/fs/review_roles/cross_cutting.md`

Adopt the mindset, checklists, best practices, and anti-patterns defined in that file.

**Step 2.2: Inspect Code & Append Findings**
Inspect the codebase exclusively through the persona of the loaded role.
Append all findings to INTERMEDIATE_REVIEW.md using the template below:
```markdown
### [Role Name] Task Title

#### 🚨 Issues Found
- **[Severity: CRITICAL | WARNING | NITPICK]** `path/to/file.fs:line_number`
  - **Summary:** Concise description of the problem.
  - **Risk:** Technical consequence (e.g., Access Violation, Deadlock, Stack Overflow, Invalid Domain State, Allocation Hot Spot).
  - **Suggested Fix:** Concrete idiomatic F# code or refactoring pattern.

#### ✨ Good Practices
- Highlight exceptional type-level safety guarantees, elegant domain modeling, or clean resource management.
```

**Step 2.3: Update Plan**
Mark the completed task as [x] in `REVIEW_PLAN.md`.

**Step 2.4: Optional Verification**
For findings that involve concurrency hazards, interop, or performance regressions, recommend running targeted verification tools to confirm or refute the concern:
- For concurrency-related findings: `dotnet test -c Release` (races and `task`-scheduling bugs often only manifest under release optimizations) and repeated runs of the affected test modules.
- For interop-related findings: run the affected tests under `dotnet test` and, where applicable, stress runs under a different GC mode (`DOTNET_GCHeapHardLimit` / Server GC) to expose pinning and finalization bugs.
- For performance-related findings: `BenchmarkDotNet` micro-benchmarks on the affected hot path, and `dotnet-counters`/`dotnet-trace` for allocation and contention evidence.
Verification results should be noted in the `INTERMEDIATE_REVIEW.md` entry for that task.

---

## STAGE 3: Final Synthesis (`FINAL_REVIEW.md`)

Once all tasks in `REVIEW_PLAN.md` are marked [x]:
1. Read the entire `INTERMEDIATE_REVIEW.md`.
2. Deduplicate overlapping issues discovered across different passes.
3. Apply **Severity Escalation Rules** (see below).
4. Synthesize all findings into a structured `FINAL_REVIEW.md` using the format below.

### Severity Escalation Rules
When synthesizing findings from multiple roles:
- If **two or more roles** independently flag `WARNING`-level issues on the **same code region**, escalate to `CRITICAL` in the final report.
- If a `WARNING` from one role would **enable or worsen** a condition that another role flagged as `CRITICAL`, escalate the `WARNING` to `CRITICAL`.
- Document the escalation rationale in the final report so the developer understands why severity increased.

### Review Metadata Header
Both `INTERMEDIATE_REVIEW.md` and `FINAL_REVIEW.md` MUST begin with a metadata header for traceability:
```markdown
<!-- Review Metadata -->
<!-- Date: YYYY-MM-DDTHH:MM:SSZ -->
<!-- Commit: <git_commit_hash> -->
<!-- Branch: <git_branch> -->
<!-- Files Changed: <count> -->
<!-- Roles Invoked: <comma-separated list> -->
```
```markdown
# Final Code Review Report

<!-- Review Metadata -->
<!-- Date: YYYY-MM-DDTHH:MM:SSZ -->
<!-- Commit: <git_commit_hash> -->
<!-- Branch: <git_branch> -->
<!-- Files Changed: <count> -->
<!-- Roles Invoked: <comma-separated list> -->

## Executive Summary
- **Overall Verdict:** `APPROVE` | `NEEDS_CHANGES` | `BLOCK`
- High-level summary of code quality and key risks in 2-3 sentences.

## 🚨 Blockers & Critical Issues (Must Fix)
- Group critical hazards: pinning violations, deadlocks, representable invalid domain states, or data-loss paths.

## ⚠️ Important Recommendations
- Structural improvements, non-critical concurrency issues, and API/ergonomics fixes.

## ✨ Highlights & Good Practices
- Positive engineering callouts for the developer.

## 🔍 Nitpicks & Minor Notes (Optional)
- Non-blocking suggestions, comments, or micro-optimizations.
```
5. Provide a brief summary of the final verdict in the agent chat and point the user to `FINAL_REVIEW.md`.
6. Optionally clean up `REVIEW_PLAN.md` and `INTERMEDIATE_REVIEW.md` after synthesis, or keep them as an audit trail. Default: keep.

---

### Recommended Folder Structure
To make this workflow operational, organize your repository's documentation directory as follows:

```text
agentic/
└── fs/
    ├── review_roles/
    │   ├── interop.md
    │   ├── concurrency.md
    │   ├── architecture.md
    │   ├── performance.md
    │   └── cross_cutting.md
    └── REVIEW.md
