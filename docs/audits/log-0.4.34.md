# Light audit: log 0.4.34

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `log-0.4.34`, unpacked by `cargo fetch` into `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/log-0.4.34`.
  Its Cargo.lock checksum is `f9f8bd3e56ce4dfc153cf470fffbfa98c7620958b312ca5c3a4b8d5181fd13c6`, and it matches the crates.io API. The `.cargo_vcs_info.json` sha1 is `8034743dd9d7f7583bd9a670271483d176130911`.
- **Use:** the logging facade (`info!`, `warn!`, ...). env_logger is the backend.
- **Verdict: ACCEPT**

## Summary

`log` is the rust-lang facade crate. With default features it has **no dependencies, no build script and no proc macros**. Its `unsafe` is limited to the global-logger static, which is guarded by an atomic state machine. It has no advisories. On its own it has no secret-handling behaviour. It forwards whatever the format arguments render (see the env_logger audit for log hygiene).

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| L1 | Info | `unsafe` surface | `lib.rs:1529` writes `static mut LOGGER` once, inside a `compare_exchange` UNINITIALIZED→INITIALIZING; `:1629` reads it after an `Acquire` load sees INITIALIZED; `:1438` transmutes the max-level atomic back into `LevelFilter`; `:452` is `unsafe impl Sync` only for targets without atomics (not ours). The `*_racy` functions are `unsafe fn`s we never call. Sound. | Accept. Initialise the logger once at startup with `env_logger::Builder::...init()`. |
| L2 | Info | Optional `kv*` features pull in `value-bag`, `serde_core` and `sval` | Not enabled by us or by env_logger with our feature set. | Leave `kv` off. |

## Details

- **Build-time code:** `build = false`, no `[build-dependencies]`, not a proc macro.
- **Dependencies:** none with default features (`value-bag`, `serde_core`, `sval` and `sval_ref` are all optional). Transitive count: **0**. env_logger turns on `log/std`, which adds no crates.
- **Features:** use the defaults (none). Don't enable `kv`/`kv_*`. The `max_level_*` / `release_max_level_*` features are **global** (they apply to every crate in the build) and aren't needed. Filter at runtime instead.
- **Licence:** `MIT OR Apache-2.0`, which the allowlist accepts.
- **MSRV:** `rust-version = "1.71.0"`, below 1.93.
- **Maintenance:**
  - Owners: `KodrAus`, `sfackler`, `huonw`, `rust-lang-owner`, and the `github:rust-lang-nursery:log-owners` team. KodrAus publishes the releases.
  - Created 2014-12-13, with 64 versions. Recent releases: 0.4.31 (2026-06-02), 0.4.32 (2026-06-04), 0.4.33 (2026-06-20), 0.4.34 (2026-08-22).
  - About 1.41 billion downloads.
- **RustSec:** no advisories for `log` in the advisory-db clone of 2026-10-22.
- **Ambient capability:** none. The crate does no I/O and reads no environment variables.

## Conditions of use

None specific to this crate. The log-hygiene rules are in `env_logger-0.11.11.md` (C3–C6).
