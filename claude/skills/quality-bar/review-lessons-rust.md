# Review lessons: Rust backend

Part of the review-lessons catalog (see "Catalog and caps" in section 8 of `SKILL.md`). Tag = highest severity seen: [H] High, [M] Medium, [L] Low.

## 1. Rust specifics

- [L] Per-item async blocks return `Result` and the consumer classifies once; erasing errors into `Option` prevents policy.
- [L] Prefer the workspace's existing primitives (`LazyLock`, `DashMap`) over `OnceLock`/`Mutex<HashMap>` with poison branches.
- [L] Env-var mutating tests restore via a Drop guard.
- [L] Result types carrying flags callers must read are `#[must_use]` or must be destructured.
- [L] Bind captured state as references before `async move` in `FnMut` closures (E0507); review fix snippets must compile.
- [L] `QueryBuilder::push_values` over hand-formatted placeholder lists.
- [L] Dev-only derives under `[dev-dependencies]` with `cfg_attr(test, ...)`, workspace-inherited.
- [L] Non-test helpers that unwrap get the crate-level allow, or return `Result`.
- [L] `impl IntoIterator<Item = &str>` over `&[&str]`; returned counters over `&mut` budget out-params; `..config` struct update over `.clone()`; `if` for one-arm matches; merged identical arms.
