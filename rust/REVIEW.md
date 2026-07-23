## Overview & Core Principles
This document defines the agentic workflow for conducting multi-pass, role-based code reviews in Systems/Baremetal Rust.

The review process MUST NOT be executed in a single broad pass. Instead, it operates strictly in three sequential stages:
1. **Plan Generation** (`REVIEW_PLAN.md`) references [REVIEW.md](agentic/rust/REVIEW.md).
2. **Role-Based Incremental Execution** (`INTERMEDIATE_REVIEW.md`) references `REVIEW_PLAN.md` and [REVIEW.md](agentic/rust/REVIEW.md)
3. **Final Synthesis** (`FINAL_REVIEW.md`) 

---

## Baseline Assumptions
- The target code **ALREADY passes** `cargo clippy -- -D warnings` and `rustfmt`.
- **DO NOT** report basic syntax issues, formatting, or micro-style lints.
- Act strictly as a high-level Principal Systems Engineer, focusing on memory safety, unsoundness, concurrency hazards, system constraints, and architecture.

---

## STAGE 1: Review Plan Generation (`REVIEW_PLAN.md`)

1. Inspect the provided `git diff` or changed files in the workspace.
2. Create a `REVIEW_PLAN.md` file in the workspace root.
3. Deconstruct the changes into targeted, domain-specific tasks.
4. **Assign a Specific Role** to each task based on the following heuristics:

| Diff Trigger / Modifiers | Assigned Role | Role Guideline File |
| :--- | :--- | :--- |
| `unsafe`, raw pointers (`*const`, `*mut`), `transmute` | `[Role: Unsafe Soundness Expert]` | `agentic/rust/review_roles/unsafe_expert.md` |
| `Atomic*`, `Mutex`, `RwLock`, `async` / `tokio` | `[Role: Concurrency Expert]` | `agentic/rust/review_roles/concurrency.md` |
| Traits, public APIs, Type-State pattern, layering | `[Role: Principal Systems Architect]` | `agentic/rust/review_roles/architecture.md` |
| `no_std`, `volatile`, MMIO, `#[interrupt]`, DMA, `cortex_m`, PAC | `[Role: Embedded Baremetal Expert]` | `agentic/rust/review_roles/embedded.md` |
| `unsafe` + `async` in same scope, manual `Send`/`Sync` impl, raw pointers in async types, lock-free across sync/async | `[Role: Cross-Cutting Integration Expert]` | `agentic/rust/review_roles/cross_cutting.md` |

5. Format every task with an uncompleted checkbox `[ ]`.

### Multi-Role Task Handling
If a single diff hunk triggers multiple role heuristics (e.g., an `unsafe` block inside an `async` function with atomics), create **separate tasks** for each role on the same code region. Each role examines the same code through its own lens independently. This ensures no perspective is missed due to premature role convergence.

### Large Diff Batching
If the diff exceeds **20 files**, group related files into sub-tasks per module or functional area rather than creating one task per file. This keeps the plan focused and prevents the review from losing context across an oversized task list.

### Example `REVIEW_PLAN.md` Format:
```markdown
# Review Plan

- [ ] **[Role: Unsafe Soundness Expert]** Audit raw pointer casting in `src/ffi/buffer.rs`
- [ ] **[Role: Concurrency Expert]** Check atomic ordering and cancellation safety in `src/net/worker.rs`
- [ ] **[Role: Principal Systems Architect]** Evaluate Type-State pattern ergonomics in `src/connection.rs`
- [ ] **[Role: Embedded Baremetal Expert]** Verify volatile access and ISR synchronization in `src/hal/uart.rs`
- [ ] **[Role: Cross-Cutting Integration Expert]** Audit unsafe+async interaction and Send/Sync correctness in `src/net/raw_channel.rs`
```

---

## STAGE 2: Incremental Role Execution (`INTERMEDIATE_REVIEW.md`)

Execute tasks from `REVIEW_PLAN.md` ONE BY ONE. Do not analyze multiple tasks simultaneously.

For each task:

**Step 2.1: Load Just-In-Time (JIT) Role Context**
Before inspecting any code, you MUST read the assigned role's guideline file using your file-reading tool:
- For [Role: Unsafe Soundness Expert] -> Read `agentic/rust/review_roles/unsafe_expert.md`
- For [Role: Concurrency Expert] -> Read `agentic/rust/review_roles/concurrency.md`
- For [Role: Principal Systems Architect] -> Read `agentic/rust/review_roles/architecture.md`
- For [Role: Embedded Baremetal Expert] -> Read `agentic/rust/review_roles/embedded.md`
- For [Role: Cross-Cutting Integration Expert] -> Read `agentic/rust/review_roles/cross_cutting.md`

Adopt the mindset, checklists, best practices, and anti-patterns defined in that file.

**Step 2.2: Inspect Code & Append Findings**
Inspect the codebase exclusively through the persona of the loaded role. 
Append all findings to INTERMEDIATE_REVIEW.md using the template below:
```markdown
### [Role Name] Task Title

#### 🚨 Issues Found
- **[Severity: CRITICAL | WARNING | NITPICK]** `path/to/file.rs:line_number`
  - **Summary:** Concise description of the problem.
  - **Risk:** Technical consequence (e.g., Undefined Behavior, Data Race, Stack Overflow, API Misuse).
  - **Suggested Fix:** Concrete idiomatic Rust code or refactoring pattern.

#### ✨ Good Practices
- Highlight exceptional safety guarantees, zero-cost abstractions, or clean code structures.
```

**Step 2.3: Update Plan**
Mark the completed task as [x] in `REVIEW_PLAN.md`.

**Step 2.4: Optional Verification**
For findings that involve `unsafe` or concurrency hazards, recommend running targeted verification tools to confirm or refute the concern:
- For `unsafe`-related findings: `cargo miri -p <crate_name>` on the affected test modules.
- For concurrency-related findings: `cargo test -p <crate_name> --release` (concurrency bugs often only manifest under release optimizations) and `loom` tests if applicable.
- For embedded-related findings: verify against the target PAC documentation and hardware reference manual.
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
- Group critical vulnerabilities, memory unsafety, or major architectural bugs here.

## ⚠️ Important Recommendations
- Structural improvements, non-critical concurrency issues, and ergonomics.

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
└── rust/
    ├── review_roles/
    │   ├── unsafe_expert.md
    │   ├── embedded.md
    │   ├── concurrency.md
    │   ├── architecture.md
    │   └── cross_cutting.md
    └── REVIEW.md
