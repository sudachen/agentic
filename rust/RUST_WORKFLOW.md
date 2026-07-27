# RUST_WORKFLOW.md

> **Note:** This file provides the Rust-specific commands for Phase 4 of [WORKFLOW.md](../WORKFLOW.md). Read `WORKFLOW.md` first.

## 4.1 Code Formatting
Ensure the code is properly formatted:
```bash
cargo fmt --all
```

## 4.2 Run Tests (Targets, Features & Docs)
Run tests only for the **affected crates** — not the entire workspace unless the change is workspace-wide:
```bash
cargo test -p <crate_name>
```

## 4.3 Run Lint
Run clippy on the **affected crates** only:
```bash
cargo clippy -p <crate_name> -- -D warnings
```

## 4.4 Run Workspace Check
Ensure local crate changes do not break downstream dependencies:
```bash
cargo check --workspace
```

## 4.5 Suppression Rule
Do not suppress warnings with `#[allow(...)]` unless there is a justified reason documented in the plan file.
