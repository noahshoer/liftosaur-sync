# Light audit: anyhow 1.0.104

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `anyhow-1.0.104`, unpacked by `cargo fetch` into `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/anyhow-1.0.104`, in a throwaway project under `/tmp`.
  Its Cargo.lock checksum is `330a5ed07fa54e4702c9d6c4174f74427fc0ef6e214bbd677ae50a5099946470`, and it matches the crates.io API. The `.cargo_vcs_info.json` sha1 is `1dbe1862aae650423e3361fbd20b7d17c5109cc3`.
- **Use:** error type and `context()` in the binary. `main` returns `anyhow::Result<()>`.
- **Verdict: ACCEPT WITH CONDITIONS**

## Summary

anyhow is dtolnay's standard error crate. It has **zero dependencies**, no proc macros, and a build script that only probes the compiler (see §1). It uses `unsafe` heavily to type-erase errors behind a thin pointer, and one unsoundness advisory from 2026 is fixed in this version. The main project risk is not in the crate. It is in how we use it: `{:#}`, `{:?}` and a `main` that returns `Err` print **every** context string and source error. If any of those include a secret, it ends up in the HA log. The conditions below cover that.

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| A1 | Low | RUSTSEC-2026-0190: `Error::downcast_mut` unsound after `.context()` (UB under Stacked Borrows) | Fixed in 1.0.103. 1.0.104 is not affected. | Require `anyhow = "1.0.103"` or later in `Cargo.toml` (C1). |
| A2 | Medium (usage) | Error chains print everything | `{:#}` gives `outer: inner`, and `{:?}` (also what `main` prints on `Err`) gives the context plus a `Caused by:` list of every source's `Display`. HTTP client errors can include the request URL, and `context(format!(..))` can include anything. | C2–C4. |
| A3 | Info | Backtrace capture | On stable, `Error` captures a `std::backtrace::Backtrace` when `RUST_BACKTRACE` or `RUST_LIB_BACKTRACE` is set, and `{:?}` prints it. A backtrace holds paths and symbols, not data. HA does not set these variables. | None. Don't set `RUST_LIB_BACKTRACE` in the image (it costs CPU on every error). |
| A4 | Info | `unsafe` surface | 104 `unsafe` tokens, almost all in `error.rs` (79: a hand-written vtable plus an `Own`/`Ref`/`Mut` thin-pointer scheme in `ptr.rs`) and `fmt.rs`. These are well-trodden: more than 1 billion downloads, and Miri runs upstream. A1 shows bugs are possible. | Accept. Don't call `downcast_mut` on hot paths we can't test (we have no reason to). |
| A5 | Info | Build script | Runs `rustc` twice at build time (§1). No network access, writes only to `OUT_DIR`. | Accept. |

## 1. Build-time code

`build = "build.rs"`. No `[build-dependencies]`. Not a proc-macro crate. What `build.rs` does:

1. With the `std` feature on (the default), it runs `$RUSTC` (through `RUSTC_WRAPPER`/`RUSTC_WORKSPACE_WRAPPER` if set) on `src/nightly.rs` with `--emit=dep-info,metadata --cap-lints=allow`, and `--target` / `CARGO_ENCODED_RUSTFLAGS` when they are set. The output goes to `$OUT_DIR/probe`, which it deletes afterwards. If that compiles (nightly, or `RUSTC_BOOTSTRAP`), it sets `cfg(error_generic_member_access)`. On our stable toolchain the probe fails, so the cfg is not set.
2. Runs `$RUSTC --version` and sets compatibility cfgs for compilers older than 1.81 and 1.85. None of them apply to us.
3. Prints `rerun-if-changed=src/nightly.rs`, `rerun-if-env-changed=RUSTC_BOOTSTRAP` and `rustc-check-cfg` lines.

It does no network or file I/O outside `OUT_DIR`, and reads no environment variables except Cargo's own and `RUSTC_BOOTSTRAP`/`RUSTC_STAGE`. That is harmless. Note that a `RUSTC_BOOTSTRAP=1` in CI would quietly change which code anyhow compiles.

The package also ships a `rust-toolchain.toml` (`components = ["rust-src"]`). It only matters if you build inside anyhow's own directory, so it has no effect on us.

## 2. Dependencies, features, licence, MSRV

- **Runtime dependencies:** none. Transitive count: **0**.
- **Features:** `default = ["std"]`; `backtrace = []` is a legacy no-op on modern Rust. **Keep the defaults** (`std` gives `std::error::Error` interop and backtraces).
- **Licence:** `MIT OR Apache-2.0`. Both are on the `deny.toml` allowlist.
- **MSRV:** `rust-version = "1.68"`, well below our 1.93.

## 3. Maintenance

- Single owner `dtolnay`, who publishes every release.
- Created 2019-10-05, with 106 versions. Recent releases: 1.0.101 (2026-02-05), 1.0.102 (2026-02-20), 1.0.103 (2026-06-25, the A1 fix), 1.0.104 (2026-07-18).
- About 1.08 billion downloads. Actively maintained.
- The single-owner bus factor is accepted across the ecosystem for dtolnay crates.

## 4. RustSec

The advisory-db clone of 2026-10-22 lists one advisory: RUSTSEC-2026-0190 (informational: unsound, patched `>= 1.0.103`). Nothing affects 1.0.104.

## Conditions of use

- **C1** Declare `anyhow = "1.0.103"` (or `"1.0.104"`). Never lower it. `cargo-deny`/`cargo-audit` in CI will flag a regression.
- **C2** Never put a secret in a `context()` / `with_context()` / `anyhow!` / `bail!` message. That means: API keys, Strava access and refresh tokens, `client_secret`, the OAuth `code` or the pasted redirect URL that holds it, `webhook_secret`, `SUPERVISOR_TOKEN`, and the `Authorization` header. Use IDs (activity ID, HTTP status, endpoint name) instead.
- **C3** Keep secrets out of URLs. Send them in headers or form bodies: basic auth for Intervals, `Bearer` for Strava and the Supervisor, and the token exchange as a form body. HTTP client errors may then print a URL safely. Check this when the `ureq` audit lands: find out whether ureq's `Error` `Display` includes the URL or query, and whether it ever includes headers.
- **C4** When wrapping an HTTP error body (`ApiError::Permanent { body }`), truncate it (for example to 500 bytes) before it goes into an error or log. Never wrap the body of a **successful** token response.
