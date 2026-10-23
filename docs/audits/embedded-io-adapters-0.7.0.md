# Security audit (light): embedded-io-adapters 0.7.0

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `embedded-io-adapters-0.7.0`, unpacked by `cargo fetch` in a throwaway project under `/tmp/audit-misc`.
  Its Cargo.lock checksum is `c900a1c087f1f7d17cdc84a1290df91521cd90933efa76d68e568385d889f2f4`, which matches the SHA-256 of the cached `.crate` file. The `.cargo_vcs_info.json` sha1 is `a926c2a65948a4f6aeddcd5d7f8f1049c1c54c95`, and `path_in_vcs` is `embedded-io-adapters`.
  Its dependency `embedded-io-0.7.1` has checksum `9eb1aa714776b75c7e67e1da744b81a129b3ff919c8712b5e1b32252c1f07cc7`.
- **Verdict: ACCEPT**

## Summary

embedded-io-adapters 0.7.0 (2025-09-30) is the latest release. It is a thin shim from the rust-embedded HAL team, and with `features = ["std"]` its only dependency is `embedded-io` 0.7.1. The rustyfit audit already covered both crates (`docs/audits/rustyfit-0.10.3.md` §2: no build script, no `unsafe`, about 1,456 lines together, no advisories). This light audit re-checks that the package hasn't changed and reviews the `std` adapter we actually use. `FromStd<T>` is a newtype that forwards `read`, `read_exact`, `write`, `write_all`, `flush` and `seek*` straight to the wrapped `std::io` type, with `Error = std::io::Error`. It has no buffering, no allocation and no `unsafe`.

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| E-F1 | Info | `FromStd::write` turns `Ok(0)` on a non-empty buffer into `WriteZero` | `src/std.rs` (`impl embedded_io::Write for FromStd<T>`). `Cursor<Vec<u8>>` never returns `Ok(0)`, so this has no effect on us. | None. |
| E-F2 | Info | No size limit of its own | The adapter adds nothing on top of the wrapped reader or writer. Bounds come from our 4 MiB input cap (rustyfit C5, flate2 audit) and the encoder's output, which is about the input size with `Normal(15)` (rustyfit F10). | Decode from an already-capped `Cursor<&[u8]>`/`&[u8]`, never straight from a network reader. |
| E-F3 | Info | Optional `tokio-1` / `futures-03` features would pull in async runtimes | Both are off unless enabled. | Enable `std` only. |

## Details

- **Features:** `std = ["embedded-io/std"]`. `tokio-1` and `futures-03` add `tokio`, `futures` and `embedded-io-async`. There are no default features.
- **Recommended:** `embedded-io-adapters = { version = "0.7.0", features = ["std"] }`. Use `FromStd::new(&mut Cursor<Vec<u8>>)` for `Encoder::encode` (which needs `Write + Seek`, as the plan's Gotchas say), and `FromStd::new(Cursor::new(&bytes[..]))` or `FromStd::new(&bytes[..])` for decoding.
- **Dependency tree** (`cargo tree -e normal,build --target all`): `embedded-io-adapters 0.7.0 → embedded-io 0.7.1`. That is the same `embedded-io` version rustyfit 0.10.3 uses, so there is one copy.
- **Unsafe:** none. `grep -rnw unsafe src` finds only two comments (`futures_03.rs:86`, `tokio_1.rs:109`), and those files aren't compiled with `std` only. `embedded-io` 0.7.1: none.
- **Build-time code:** `build = false`, no proc macros, no FFI. The only I/O is through the wrapped type.
- **Size:** `src/` is 464 lines (`std.rs`, `fmt.rs`, `lib.rs`, plus the two async adapters).
- **Licence:** `MIT OR Apache-2.0` (both crates). Allowed.
- **MSRV:** 1.81 (both crates). That's ≤ 1.93, and it builds with `cargo +1.93 build --locked`.
- **Maintenance:** crates.io owners `Dirbaio` and `github:rust-embedded:hal`. Releases: 0.5.0 (2023-08), 0.6.0 (2023-10), 0.6.1 (2023-11), 0.6.2 (2025-08), 0.7.0 (2025-09-30), all published by Dirbaio. The repo is `rust-embedded/embedded-hal` (a workspace): 2.7 k stars, last push 2026-05-26. A slow release cadence is expected for a finished adapter crate.
- **Advisories:** RustSec advisory-db (HEAD `640bf5a`, 2026-10-22) has no entries for embedded-io or embedded-io-adapters. GitHub security advisories for `rust-embedded/embedded-hal`: none.

## Conditions of use

None beyond enabling `std` only. Bump it together with rustyfit, because the two must agree on the `embedded-io` major version. The rustyfit re-audit script already fetches both.
