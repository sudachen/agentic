# RUST_WORKFLOW.md

> **Note:** This file provides the Rust-specific commands for Phase 4 of [WORKFLOW.md](../WORKFLOW.md). Read `WORKFLOW.md` first.

## 4.0 Map the Change to a Gate

- **Fast gate:** The change touches one crate and adds or changes no public item, `Cargo.toml` entry, or non-default feature.
- **Dependency gate:** The change adds or changes a public item, a feature flag, a `Cargo.toml` dependency or version, or a workspace-level file.
- **Final gate:** The change is workspace-wide, or the task is the plan's last task.

## 4.1 Code Formatting
Ensure the code is properly formatted for every affected crate:
```bash
cargo fmt --all
```

## 4.2 Run Tests (Targets, Features & Docs)
Under the Fast gate, run tests for the **affected crate** only:
```bash
cargo test -p <crate_name>
```
Under the Dependency gate, also run tests with every feature combination the crate defines, and run documentation tests:
```bash
cargo test -p <crate_name> --all-features
cargo test -p <crate_name> --doc
```
Under the Final gate, run the full workspace test suite:
```bash
cargo test --workspace --all-features
```

## 4.3 Run Lint
Run Clippy on the **affected crates**. Under the Dependency gate, run it with every feature combination:
```bash
cargo clippy -p <crate_name> -- -D warnings
cargo clippy -p <crate_name> --all-features -- -D warnings
```

## 4.4 Run Dependency and Workspace Checks
Under the Dependency gate, check that dependency changes resolve cleanly:
```bash
cargo update -p <crate_name> --dry-run
```
Under the Dependency gate and the Final gate, ensure local crate changes do not break downstream dependencies:
```bash
cargo check --workspace --all-features
```

## 4.5 Toolchain Recording
Record the `rustc` and Clippy versions in the plan's Discovered Gotchas & Constraints section when a task's result depends on a specific toolchain version:
```bash
rustc --version
cargo clippy --version
```
Treat a mismatch between the recorded toolchain and the toolchain installed for verification as an Environment failure, not an Implementation failure.

## 4.6 Final Workspace Check
Before Phase 5 archives the plan, run the complete workspace check regardless of which crates the plan touched:
```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-features
cargo check --workspace
```

## 4.7 Suppression Rule
Do not suppress warnings with `#[allow(...)]` unless there is a justified reason documented in the plan file.
