# ARCHITECT.md

This document defines the workflow for creating architecture change proposals.
An architecture change proposal is a document that describes a planned change to the system architecture before any code is written.
The proposal lives in `docs/proposals/` until it is implemented or deleted, and uses a human-readable name.

## Proposal File Location

Active proposal files live in the `docs/proposals/` directory at the repository root.
Implemented proposals move to `docs/proposals/done/` (see Proposal Lifecycle).
Git tracks proposal files. Proposals are version-controlled documents, unlike local plan files in `project/`.
Create the `docs/proposals/` and `docs/proposals/done/` directories if they do not exist.
Write relative links in proposal files with the repository root as the base.

### Naming Convention

Name each proposal file with a short, human-readable identifier:

- **Format:** `docs/proposals/<short_readable_name>.md`
- **Rules:**
  - Use `snake_case` for the file name.
  - Keep the name under 40 characters.
  - Do not include dates or version numbers in the file name.
  - Example: `docs/proposals/rwlock_for_feed_state.md`

The no-dates rule applies to active proposals only. Archival into `docs/proposals/done/` adds a `<yymmdd>_` prefix (see Proposal Lifecycle).

---

## Writing Style Standard

> **Note:** All proposals produced under this workflow follow [STYLE.md](STYLE.md).

---

## Phase 1 — Intake and Clarification

The user provides the main idea of the architecture change in a prompt.
The agent must not create a proposal document until it resolves every blocking question.

### 1.1 Record the User Intent

Summarize the user's idea in one paragraph. Record:
- **What** the user wants to change.
- **Why** the user wants the change (the problem or motivation).
- **Where** in the system the change applies (which components, modules, or layers).

This summary feeds Sections 1 (Summary) and 2 (Motivation) of the proposal document.

### 1.2 Explore the Current Architecture

Check whether `docs/ARCHITECTURE.md` exists.
If `docs/ARCHITECTURE.md` is missing, run the Architecture Extraction Workflow in [REVERSE.md](REVERSE.md) first.
If it exists but is stale, apply the Incremental Updates procedure in [REVERSE.md](REVERSE.md).
Read the existing architecture documentation and relevant source code before asking the user any question.
Consult the stack-specific architecture guide at `agentic/<lang>/<LANG>_ARCHITECT.md` (such as [rust/RUST_ARCHITECT.md](rust/RUST_ARCHITECT.md)).
Read `agentic/<lang>/notes/` (if present) for accumulated, language-specific learnings: conventions, toolchain gotchas, and patterns that inform the proposal.
Use these guidelines to evaluate types, memory layout, concurrency, error boundaries, and invariants.
Answer discoverable questions through codebase exploration instead of asking the user.

### 1.3 Identify Blocking Questions

A blocking question is a gap that changes the proposal's scope, trade-offs, or implementation plan depending on its answer.
Classify each question:
- **Discoverable:** The codebase, existing documentation, or version control history contains the answer. Answer it through exploration. Do not ask the user.
- **Blocking:** Only the user can resolve the answer. Ask the user.

### 1.4 Ask the User

Present every blocking question to the user in a single, structured list.
Do not proceed to Phase 2 until the user resolves every blocking question.

---

## Phase 2 — Propose Changes and Iterate

The agent drafts the proposed change terms and presents them to the user for review.
This phase iterates until the user agrees and requests no further improvements.

### 2.1 Draft the Proposal Terms

Prepare a concise summary of the proposed architecture change. Apply domain constraints from the language-specific architecture guide. Include:
- **Scope:** What the change includes and what it excludes.
- **Affected components:** Which modules, layers, or external integrations the change touches.
- **Key decisions:** The primary architectural choices the proposal makes.
- **Trade-off summary:** The main benefits and costs, in brief.
- **High-level implementation approach:** How a future implementation would proceed, in broad strokes.

### 2.2 Present to the User

Present the draft terms to the user in a readable format.
Ask the user to confirm, reject, or request changes.

### 2.3 Iterate

If the user requests changes or rejects the terms:
1. Update the draft terms.
2. Present the revised terms to the user.
3. Repeat until the user agrees and states that no further improvements are needed.

Do not proceed to Phase 3 until the user explicitly agrees.

---

## Phase 3 — Create the Proposal Document

Once the user agrees to the terms and requests no further improvements, the agent creates the proposal file.

### 3.1 Choose the File Name

Derive a short, human-readable name from the proposal's subject.
Check `docs/proposals/` for name collisions. If the name is taken, append a distinguishing qualifier that narrows scope or aspect (for example, `rwlock_for_feed_state_orders.md`). Do not use dates or version numbers as qualifiers.

### 3.2 Write the Proposal File

Create the file at `docs/proposals/<short_readable_name>.md` using the Proposal Document Template below.
Set the initial status to `created` and the `Created` field to the current ISO 8601 date.
If a proposal file for this change already exists in `docs/proposals/`, update it in place instead of creating a new file.
The document must contain every section in the template.

### 3.3 Commit the Proposal

Stage and commit the proposal file:
```bash
git add docs/proposals/<short_readable_name>.md
git commit -m "Add architecture proposal: <short_readable_name>"
```

Do not push to remote unless the user asks.

---

## Proposal Document Template

Every proposal file must follow this template.

```markdown
# Architecture Proposal: <Human-Readable Title>
<!-- Link base: write relative links with the repository root as the base. -->

> **Note:** This proposal follows the workflow defined in [ARCHITECT.md](agentic/ARCHITECT.md).

- **Author:** <name or agent session>
- **Created:** <ISO 8601 date>
- **Status:** created
- **Implemented:** <ISO 8601 date — set only when the proposal moves to `docs/proposals/done/`>

## 1. Summary

One-paragraph description of the proposed architecture change and the problem it solves.

## 2. Motivation

Explain the problem, limitation, or opportunity that drives this proposal.
Reference specific files, modules, or architectural debt from `docs/ARCHITECTURE.md` where applicable.

## 3. Current Architecture

Describe the relevant parts of the current system that the proposal changes.
Include only the components, data flows, and invariants the proposal affects.

\`\`\`mermaid
%% Current state diagram (optional — follow Mermaid conventions in REVERSE.md)
\`\`\`

## 4. Proposed Changes

Describe how the architecture must change.
Cover each of the following:
- **Structural changes:** New, removed, or reorganized modules, layers, or components.
- **Data and type changes:** New types, changed interfaces, or modified serde boundaries.
- **Concurrency and state changes:** New synchronization strategies, channels, or state ownership.
- **Integration changes:** New or modified external integrations, protocols, or trust boundaries.

\`\`\`mermaid
%% Proposed state diagram (optional — follow Mermaid conventions in REVERSE.md)
\`\`\`

## 5. Trade-offs

List the benefits and costs of the proposed change.

### Benefits
- **[Benefit]:** Description and impact.

### Costs and Risks
- **[Cost or Risk]:** Description, impact, and mitigation.

### Alternatives Considered
- **[Alternative]:** Description and reason for rejection.

## 6. Implementation Plan

Describe how the change can be implemented.
Break the work into ordered phases or tasks. For each:
- **Goal:** What the phase accomplishes.
- **Files to touch:** Which files the phase creates or modifies.
- **Approach:** The key technical steps.

Reference the Planning and Execution Workflow (`agentic/WORKFLOW.md`) as the process for executing the implementation.
Architecture proposals typically trigger High-Risk Categories as defined in `agentic/WORKFLOW.md` (concurrency or shared state, public API or interface, database schema or migration).
These require the Standard change plan tier in `project/PLAN_<name>.md`.
Tasks listed here become execution tasks in `PLAN_<name>.md`.
Note: `project/` plan files are local-only and not tracked by Git. Reference them by name, not by link.

## 7. Testing Strategy

Describe how the change can be tested. Cover:
- **Unit tests:** Which modules or functions need new tests.
- **Integration tests:** Which cross-component or API boundary tests the change requires.
- **Invariant checks:** Which architectural invariants the change introduces, modifies, or removes.
- **Migration or rollback tests:** Apply when the change alters data layout, storage, or public API.

These invariant and test checks become Acceptance Criteria in `project/PLAN_<name>.md` (local-only file; reference by name, not by link).

## 8. Open Questions

List questions that remain unresolved at the time of writing.
Mark each as `blocking` or `non-blocking`.
```

---

## Proposal Lifecycle

Each proposal has exactly one of two states:

- **created:** The proposal exists in `docs/proposals/` and awaits implementation. A proposal remains in this state only until it is implemented or deleted.
- **implemented:** The associated plan in `project/` finished under `agentic/WORKFLOW.md` Phase 5. The proposal file moves to `docs/proposals/done/<yymmdd>_<name>.md`, where `<yymmdd>` is the implementation date, so the `done/` directory sorts by finalization date. Its Status is set to `implemented` and its `Implemented` field records the completion date.

```text
created ──(plan finished)──> implemented   [file moves to docs/proposals/done/<yymmdd>_<name>.md]
created ──(user discards)──> deleted       [file removed from docs/proposals/]
```

While a proposal is `created`, amend it in place at the user's request — including when a newer idea replaces it. Do not keep replaced or rejected proposals; delete them.
An `implemented` proposal is a historical record. Do not amend it; write a new proposal instead.

When the user triggers the Planning and Execution Workflow to implement a proposal:
- The plan file in `project/PLAN_<name>.md` must reference the proposal file in its Feature Summary.
- Transfer tasks from Section 6 and invariant checks from Section 7 into the plan file.
- `agentic/WORKFLOW.md` Phase 5 moves the proposal to `docs/proposals/done/` when the plan is finalized.

---

## Workflow Summary

```text
User provides architecture change idea
      │
      ▼
┌─────────────────────────────────────────┐
│ Phase 1: Intake and Clarification       │  ← Explore codebase, check ARCHITECTURE.md,
└────────────────────┬────────────────────┘    consult stack-specific guide, ask blocking items
                     ▼
┌─────────────────────────────────────────┐ ◄──────────┐
│ Phase 2: Propose Changes and Iterate    │            │ Loop until
│          (Draft terms, user reviews)    │ ───────────┘ user agrees
└────────────────────┬────────────────────┘
                     ▼
┌─────────────────────────────────────────┐
│ Phase 3: Create Proposal Document       │  ← Write file in docs/proposals/
│          in docs/proposals/             │    and commit to Git
└─────────────────────────────────────────┘
```

---

## Completion Criteria

The agent's work is complete when all items are checked:

- [ ] The agent confirmed `docs/ARCHITECTURE.md` exists or created it via `agentic/REVERSE.md`.
- [ ] The agent consulted the language-specific architecture guide (`agentic/<lang>/<LANG>_ARCHITECT.md`) and the notes in `agentic/<lang>/notes/` (when present).
- [ ] Every blocking question is resolved.
- [ ] The user agreed to the proposal terms and requested no further improvements.
- [ ] The proposal file exists at `docs/proposals/<short_readable_name>.md` with Status `created` and a `Created` date.
- [ ] The proposal file contains every section from the Proposal Document Template.
- [ ] The proposal conforms to [STYLE.md](STYLE.md).
- [ ] The proposal file is committed to Git.
