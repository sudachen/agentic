# REQUEST.md — Upstream Feature Request Workflow

This document defines a 4-phase workflow for writing feature requests against external dependencies. The workflow applies to Rust crates, Go modules, and Python packages. The output is a self-contained document that an upstream maintainer can act on without access to this repository.

Use this workflow when the user reports a problem in an external dependency, asks for a missing API or feature in an external dependency, or complains about an API sequence that requires too many steps.

## Request Classes

Classify the request before drafting. The class determines which template sections carry the most weight.

| Class | Trigger | Emphasis |
| :--- | :--- | :--- |
| **Defect** | Observed behavior contradicts documented or intended behavior | Reproduction, mechanism, expected vs actual |
| **Missing API** | Needed data or control exists internally but is not public | Proposed signature, semver impact |
| **Missing feature** | The dependency lacks a capability entirely | Use case, ranked options, alternatives evaluated |
| **API ergonomics** | A common operation requires a long or fragile call sequence | Before/after comparison, convenience wrapper sketch |

A defect request still proposes solutions. An ergonomics request still needs a reproduction that shows the awkward sequence. Do not file a defect as a feature request or vice versa.

## Phase 1 — Confirm the Issue Is Upstream

Do all steps before drafting. Skip no step.

1. **Pin the dependency version.** Record the exact version and the manifest line that selects it.
   - Rust: `cargo tree -i <crate>` for the resolved version; `Cargo.toml` for the requirement.
   - Go: `go list -m <module>`; `go.mod` for the requirement.
   - Python: `pip show <package>`; `pyproject.toml` or `requirements.txt` for the requirement.

2. **Read the upstream source.** Prefer the locally resolved source over web documentation.
   - Rust: `~/.cargo/registry/src/<registry>/<crate>-<version>/`.
   - Go: `$(go env GOMODCACHE)/<module>@<version>/`.
   - Python: the installed `site-packages/` tree.
   - Check the upstream main branch too. The issue may already be fixed after the pinned version.

3. **Rule out downstream misuse.** Read the dependency's documentation and examples. Verify the call sequence, initialization order, and configuration against upstream intent. Record what we checked.

4. **Search the upstream tracker.** Look for existing issues, pull requests, and discussions that match. If one exists, the request becomes a comment or a reproduction on that issue instead of a new document.

5. **Record the current workaround.** Note exactly what this project does today to live with the issue. Include file and symbol references. The request document must state the workaround so the maintainer can judge severity.

Output of Phase 1: the request class, the pinned version, a confirmed upstream root cause, and a list of related upstream issues.

## Phase 2 — Build a Minimal Reproduction

The reproduction must run without this repository.

- **Self-contained.** One file or one minimal project. No private crates, modules, or test fixtures.
- **Deterministic.** State inputs, timing assumptions, and required environment (e.g., two containers on one network).
- **Runnable against the pinned version.** Pin the dependency in the reproduction's own manifest.
- **Short.** Show the failure in the fewest lines possible. A defect reproduction ideally prints the wrong behavior and exits.

Language conventions:

| Language | Preferred form | Run command to document |
| :--- | :--- | :--- |
| Rust | `cargo new` bin crate, or a `#[test]` in a gist | `cargo run` / `cargo test` |
| Go | `main_test.go` or single `main.go`; a Go Playground link when no OS access is needed | `go test` / `go run` |
| Python | One script plus a pinned `requirements.txt` | `python repro.py` |

When a live reproduction is impossible (hardware, network topology), document the observed failure instead: logs, counters, packet captures, and a timeline. Mark the section **Observed behavior** and state why a standalone repro is not feasible.

## Phase 3 — Write the Request Document

Create the document at `docs/upstream/<package>-<slug>.md`, where `<slug>` is a short readable name. Example: `docs/upstream/light-telemetry-liveness-gate.md`.

### Header fields

```markdown
# Feature Request: <one-line title>

- **Author:** <name or agent session>
- **Date:** YYYY-MM-DD
- **Status:** draft
- **Upstream:** `<package>` (<repo URL>, version `<x.y.z>`)
- **Request class:** defect | missing-api | missing-feature | ergonomics
- **Downstream workaround:** <file:symbol or "none">
```

Status lifecycle: `draft → sent → accepted | rejected | implemented | superseded`. Update the field when the state changes. Record the upstream issue URL after `sent`.

### Required sections

Number sections. Keep this order. Omit a section only when it cannot apply; write why.

1. **Summary.** One paragraph: the problem, the impact, and the ask. A maintainer must understand the request from this paragraph alone.
2. **Environment.** Dependency version, toolchain version, platform, and the dependency-graph path that pulls the package in.
3. **Background.** The concrete context where the problem appeared. Include observed symptoms with evidence (logs, counters, timelines).
4. **Problem.** The upstream mechanism that produces the failure. Cite exact upstream `path::symbol` references. Include a timeline or sequence diagram when the failure is temporal.
5. **Minimal reproduction.** The Phase 2 artifact, inline or linked, plus its run command and observed output.
6. **Proposed solutions.** Two to four ranked options labeled A, B, C, D. For each option:
   - a code sketch in the upstream language, sized to show the API surface, not the full implementation;
   - cost and benefit in one or two sentences each;
   - semver impact: additive, breaking, or internal-only.
   For missing-API requests, option A is the proposed public signature.
7. **Current workaround.** What this project does today. Include downstream `file:symbol` references and the workaround's cost.
8. **Recommendation.** Pick one option. Justify the pick in two to four sentences. State acceptable fallbacks.
9. **Contribution offer.** State whether we can send a pull request, test a patch, or only report. Maintainers prioritize requests with contribution offers.
10. **References.** Upstream files cited, related upstream issues/PRs, and downstream consumer files.

### Writing rules

- Follow `agentic/STYLE.md`. The document leaves this repository; style still applies.
- Every upstream claim needs a `path::symbol` or `file:line` citation. No ungrounded assertions.
- Rank options by preference and say so explicitly ("in order of preference").
- Do not demand a specific implementation when several work. State the problem precisely; propose options; let the maintainer choose.
- Keep code sketches under 30 lines. Sketches show shape, not completeness.
- Do not paste downstream proprietary code into the document. Reference it by path only.

## Phase 4 — Review and Deliver

### Pre-send checklist

Run every check before marking the document `sent`.

- [ ] Version pinned in header matches the reproduction's manifest.
- [ ] Reproduction runs clean against the pinned version on a fresh checkout.
- [ ] Every upstream symbol citation resolves in the pinned version's source.
- [ ] Upstream tracker was searched; related issues are linked in References.
- [ ] Options are ranked and each states its semver impact.
- [ ] The Summary paragraph stands alone.
- [ ] Contribution offer is explicit.
- [ ] No private code, paths that leak internal structure beyond file names, or secrets appear in sections that will be pasted upstream.

### Delivery

- Paste sections 1–9 into the upstream issue tracker or discussion. Keep References; maintainers value precise citations.
- For a defect with an obvious one-line fix, the document may accompany a pull request instead. Say so in the Contribution offer.
- Update `Status` to `sent` and add the upstream issue URL to the header.
- When upstream lands a fix: set `Status` to `implemented`, note the released version, and create a follow-up task to remove the downstream workaround.

## Template

Copy this skeleton for Phase 3:

```markdown
# Feature Request: <title>

- **Author:**
- **Date:**
- **Status:** draft
- **Upstream:** `<package>` (<repo>, version `<x.y.z>`)
- **Request class:**
- **Downstream workaround:**

## 1. Summary

<problem + impact + ask, one paragraph>

## 2. Environment

- Package: `<name> <version>` (via `<manifest line>`)
- Toolchain: <rustc / go / python version>
- Platform: <os/arch>

## 3. Background

<context and observed symptoms with evidence>

## 4. Problem

### 4.1 Mechanism

<upstream path::symbol citations, behavior walkthrough>

### 4.2 <optional: timeline or secondary issues>

## 5. Minimal Reproduction

```<lang>
<self-contained code>
```

Run: `<command>`
Observed: `<output>`

## 6. Proposed Solutions

### Option A — <name> (preferred)

```<lang>
<API sketch>
```

- Cost:
- Benefit:
- Semver impact: additive | breaking | internal

### Option B — <name>

...

## 7. Current Workaround

<downstream file:symbol, what it does, its cost>

## 8. Recommendation

<pick + justification + acceptable fallback>

## 9. Contribution Offer

<PR offer / testing offer / report only>

## 10. References

- <upstream file:symbol>
- <related upstream issue/PR>
- <downstream consumer file>
```

## Anti-Patterns

- **No pinned version.** "Latest" rots. Always pin.
- **Repro that needs this repo.** A maintainer cannot run it. Keep it standalone.
- **Proposing a duplicate API.** Check upstream first; a missing-looking API often exists under another name.
- **Solution without problem statement.** Describe the mechanism before proposing fixes.
- **One option only.** Ranked options show the problem was analyzed, not just the preferred fix.
- **Silent severity.** State the workaround and its cost so the maintainer can prioritize.
- **Never updating status.** A stale `draft` that was already sent misleads the next reader.
