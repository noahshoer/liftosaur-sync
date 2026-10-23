# Security audit (light): base64 0.23.1

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `base64-0.23.1`, unpacked by `cargo fetch` in a throwaway project under `/tmp/audit-misc`.
  Its Cargo.lock checksum is `ac07cdecf99051d9a5238b80f35af32cdeba5b336e55d957b318b50137e18da5`, which matches the SHA-256 of the cached `.crate` file. The `.cargo_vcs_info.json` sha1 is `069bf7067b949f5c0a92b6ceb82492920502f2c2`.
- **Verdict: ACCEPT WITH CONDITIONS** (one feature choice)

## Summary

**base64 0.23 exists, and 0.23.1 (2026-08-04) is the current release**, so the plan's version is correct. The crate has no dependencies, no build script, no proc macros and no FFI. New in 0.23.0 (2026-07-23) is a **default-on `simd-unsafe` feature** that adds AVX2 and NEON engines. Those engines are the crate's only `unsafe` code. With `default-features = false, features = ["std"]` the crate is `#![forbid(unsafe_code)]`. That is exactly how **ureq 3.5.0 already depends on it** (`base64 ^0.23.1`, `default-features = false`, `["std"]`), so declaring it the same way adds no new crate and no new code to the build. We only *encode* `API_KEY:<key>` for the Intervals.icu basic-auth header, which is trusted input.

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| B-F1 | Low | Default `simd-unsafe` feature adds hand-written SIMD `unsafe` | `src/engine/simd.rs` (807 lines, `#![allow(unsafe_code)]`) holds the AVX2 and NEON kernels and the runtime-detecting `Simd` engine. Every block has a `// SAFETY:` comment with explicit bounds guards. The `GeneralPurpose` engine (`STANDARD`) does **not** use SIMD (`engine/general_purpose/mod.rs:21-22`), so we would never execute this code. But on aarch64 (NEON is a baseline target feature) it compiles in, and Cargo feature unification would turn it on for ureq's copy as well. | Use `default-features = false, features = ["std"]`, which gives `#![forbid(unsafe_code)]` (`lib.rs:284`). |
| B-F2 | Info | RUSTSEC-2017-0004, an integer overflow in `encode_config_buf` | Patched `>= 0.5.2`. Not applicable. | None. |
| B-F3 | Info | Already a transitive dependency | ureq 3.5.0 → `base64 ^0.23.1` (non-optional). Declaring `0.23.1` keeps a single copy, which matters if `multiple-versions = "deny"`. | Keep the same minor version as ureq when bumping. |
| B-F4 | Info | Not constant-time | Documented in `general_purpose/mod.rs:24`. That only matters for decoding secrets an attacker can time, and we only encode our own key locally. | None. |

## Details

- **Features:** `default = ["std", "simd-unsafe"]`, `std = ["alloc"]`, `alloc = []`, `simd-unsafe = []`.
- **Recommended:** `base64 = { version = "0.23.1", default-features = false, features = ["std"] }`. Usage is `base64::engine::general_purpose::STANDARD.encode(..)` with `use base64::Engine as _;`. The plan's note that the `Engine` trait must be in scope is still correct. In the scratch build, `STANDARD.encode("API_KEY:abc123")` gives `QVBJX0tFWTphYmMxMjM=`.
- **Unsafe:** with the recommended features there is none: `#![cfg_attr(not(feature = "simd-unsafe"), forbid(unsafe_code))]` (`lib.rs:284`). With `simd-unsafe` on, `unsafe` stays confined to `engine/simd.rs` (`lib.rs:285` sets `deny(unsafe_code)` everywhere else).
- **Dependencies:** none (dev-dependencies only: clap, criterion, rand, rstest, strum).
- **Build-time code:** no `build.rs`, not a proc macro.
- **Licence:** `MIT OR Apache-2.0`. Allowed.
- **MSRV:** 1.71.0 (raised from 1.48 in 0.23.0). That's ≤ 1.93, and it builds with `cargo +1.93 build --locked`.
- **Maintenance:** owners `marshallpierce` and `alicemaz`, and every release is published by marshallpierce. Release history: 0.21.x (2023–24), 0.22.0 (2024-03), 0.22.1 (2024-04), then a two-year gap, then 0.23.0 (2026-07-23) and 0.23.1 (2026-08-04, a test-only fix for non-SIMD architectures). None are yanked. Repo `marshallpierce/rust-base64`: 740 stars, last push 2026-08-18. About 1.8 G downloads.
- **Advisories:** RustSec advisory-db (HEAD `640bf5a`, 2026-10-22) lists only RUSTSEC-2017-0004 (patched). GitHub security advisories: none.

## Conditions of use

1. Use `default-features = false, features = ["std"]`, so the crate is `forbid(unsafe_code)`, the same as ureq's copy.
2. Use `general_purpose::STANDARD` for basic auth. Never log the encoded header: it's the API key in reversible form.

## Re-audit on bump

```sh
sed -n '/\[features\]/,/^\[lib/p' $R/Cargo.toml     # is simd-unsafe still the only unsafe-bearing feature?
grep -n 'forbid(unsafe_code)' $R/src/lib.rs
cargo tree -i base64 --target all                   # one copy, shared with ureq
```
