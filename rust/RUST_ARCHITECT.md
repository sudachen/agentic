# Rust Architecture Analysis Specifics

## 1. Module Boundaries and Crates
* Analyze `Cargo.toml`: identify subprojects in `[workspace]`, feature flags (`features`), and key dependencies.
* **Workspace inheritance:** distinguish workspace-level (`[workspace.dependencies]`, `version.workspace = true`) and crate-level dependencies. `[workspace.dependencies]` is the single source of truth for versions; record crate-level deviations as architectural decisions.
* Separate public API (`pub`, `pub(crate)`) from internal implementation in `lib.rs` / `main.rs` / `mod.rs`.
* Define layer boundaries: Domain Logic, I/O Infrastructure, FFI/C-bindings.

## 2. Conditional Compilation and Codegen
* **`#[cfg(...)]`**: Map `#[cfg(feature = "...")]` and `#[cfg(target_...)]` branches. Different configurations can form different architectures.
* **Feature Matrix:** identify which features pull in which dependencies and which are mutually exclusive. Check if `cargo check --all-features` passes (or document reasons why it does not).
* **`build.rs`**: Find build scripts. Document: code generation, protocol parsing (protobuf, SQL), C library linking, constant generation. `build.rs` is part of the architecture affecting compilation.

## 3. Types, Abstractions, and Memory Layout
* **Traits**: Find key traits — they define interfaces and extension points of the system.
* **Dispatch**: Record usage of dynamic (`dyn Trait`) and static (`<T: Trait>`) dispatch.
* **ADT (Enum/Struct)**: Identify domain data structures and `enum` states (Type-State and State Machine patterns).
* **Serde Boundaries:** `#[derive(Serialize, Deserialize)]` defines contracts at system boundaries (API, persistence). Separate serializable DTOs from internal domain types — these are points where data crosses process/layer boundaries.
* **Zero-Copy & Layout:** record usage of `bytes::Bytes`, `zerocopy`, `bytemuck`, custom allocators/arenas (`bumpalo`), alignment `#[repr(align(...))]`, and zero-copy parsing in hot paths.

## 4. Ownership, State, and Message Passing
* **Shared State**: Trace shared memory flows: `Arc<Mutex<T>>`, `Arc<RwLock<T>>`, `Rc<RefCell<T>>`, `parking_lot`.
* **Channels & Actor Pattern:**
  * Map message passing (`tokio::sync::mpsc`, `broadcast`, `watch`, `oneshot`, `crossbeam`).
  * Identify actor-like patterns (`ActorHandle` + `Receiver` loop) and actor frameworks (`actix`, `ractor`).
  * **Backpressure:** check for bounded vs unbounded channels and behavior under saturation (blocking, dropping, error).
* **Lifetimes & Pin:**
  * Identify places with explicit `'a` annotations to trace reference relationships between objects.
  * Find usage of `Pin`, `PhantomPinned`, self-referential structs. Document types that cannot be moved and why (async generators, embedded DMA buffers).

## 5. Concurrency and Async
* Identify the async runtime used (`tokio`, `async-std`, `smol`, `no_std`/custom execution).
* Find long-running background tasks (`tokio::spawn`, `select!`, `join!`).
* **Send/!Send Boundaries:** trace `Send` bounds for spawned tasks — this is an architectural constraint defining which types can be used in async contexts.
* **Async Traits:** document the approach — native `async fn` in traits, `async-trait` crate, or `trait_variant`.
* **Cancellation Safety:** verify cancellation patterns for async tasks. Pay attention to `tokio::select!` with `biased` — this affects processing priorities and potential resource leaks on future cancellation.

## 6. Data Layer, Transactions, and Storage
* **Connection Pooling:** identify connection pools for databases and external resources (`sqlx::Pool`, `deadpool`, `bb8`, `r2d2`).
* **Transaction Management:** document transaction management strategies (explicit `&mut Transaction` passing, Unit of Work, Repository trait).
* **Embedded Storage:** map access to local embedded KV storage engines (`rocksdb`, `sled`, `redb`, `heed`), locking schemes, and snapshots.

## 7. Errors, Macros, and Unsafe
* **Errors & Boundaries:**
  * Identify error handling strategy (`thiserror`, `anyhow`, custom `enum Error`).
  * Check error isolation: ensure internal infrastructure errors do not leak out through uncontrolled `#[from]` in public APIs.
* **Macros:** Identify `macro_rules!` and procedural macros (`proc_macro`) hiding code generation or DSLs.
* **Unsafe:** Find all `unsafe` blocks, document their purpose, safety invariants (`// SAFETY:`), and encapsulation boundaries.

## 8. Observability and Telemetry
* **Tracing & Context:** document usage of `tracing`, `tracing::instrument`, and span context propagation across thread/async task boundaries.
* **Metrics & Audit:** map metric collection points (`metrics`, `prometheus`) and critical business event logging.

## 9. FFI and ABI
* **`extern`**: Distinguish `extern "C"`, `extern "Rust"`, `extern "system"`. Record ABI at each FFI boundary.
* **`#[repr(C)]`**: Identify structs with `#[repr(C)]` — these are types crossing FFI boundaries. Verify correspondence with C-side definitions.
* **Callbacks & Panics:** Find callback functions via `extern "C" fn` — points where C code calls back into Rust. Document `catch_unwind` and panic-safety at these boundaries.

## 10. Test Organization
* **Unit Tests:** `#[cfg(test)]` modules inside `src/` — document which modules have inline unit tests.
* **Integration Tests:** `tests/` directory — identify scenarios testing against the crate's public API.
* **Strategy:** reflect whether tests cover architectural layers independently (unit) or across boundaries (integration). Highlight test coverage gaps in key components.

## 11. Analysis Tools
* `cargo tree -d` — Detect duplicate dependencies (incompatible versions of the same crate).
* `cargo modules structure --tree` / `cargo modules dependencies` — Module visibility graph and coupling.
* `cargo depgraph --workspace-only | dot -Tpng -o deps.png` — Visualize the workspace crate graph.
* `cargo expand <path::to::module>` — Inspect macro expansions and code generation.
* `cargo audit` — Known vulnerabilities in dependencies.
* `cargo bloat` / `cargo llvm-lines` — Binary size analysis and heavy generic monomorphization.
* `cargo clippy` — Architectural linter warnings (style, ownership, concurrency).

