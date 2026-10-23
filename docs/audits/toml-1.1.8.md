# Security audit (light): toml 1.1.8+spec-1.1.0

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io packages, unpacked by `cargo fetch` in a scratch project (`rust-version = "1.93"`) under `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/`.

  | Crate | Cargo.lock checksum (SHA-256, matches the crates.io API) | vcs sha1 |
  |---|---|---|
  | toml 1.1.8+spec-1.1.0 | `20489e00e4d8741d6be680764cc12e270655e375a20d1011e844a9c3379e678d` | `ceac6e4412420d5a5c19e4f7293ad005cf1a31b2` (`crates/toml`) |
  | toml_parser 1.1.5+spec-1.1.0 | `baa693a8032d7e1cada7d0041e96126df243179ff061456783ac7f12bda4744c` | `ceac6e4` (`crates/toml_parser`) |
  | toml_datetime 1.1.2+spec-1.1.0 | `2b86d767906c6c42421dcba507eb9d203e779497710a47782a224bb871653053` | `f72038c` |
  | serde_spanned 1.1.2 | `4e7523beb55eece201a2356bee0bbca0d1ab466c14c07703b2e0ee6d42cb0c2c` | `f72038c` |
  | toml_writer 1.1.3+spec-1.1.0 (default only) | `06bdbd8cfc056b8d2e2e85f29b56a3bdbecb527cef81eb39e3e7b98af4652770` | `f72038c` |
  | winnow 1.0.4 | `23b97319f7b8343df12cc98938e5c3eb436064524c8d2b4e30a1d3a36eecdf81` | `7539ec0` |

  The published `src/` of `toml` and `toml_parser` is byte-identical to the `toml-rs/toml` commit `ceac6e4` ("chore: Release", 2026-10-08).
- **Verdict: ACCEPT**

## Summary

**Where we use it:** only `toml::from_str` on `data/exercises.toml`, which is compiled in with `include_str!` (Task 6). That's a trusted, repo-controlled input, checked by a test that "the built-in table resolves fully". User overrides (`custom_exercises`) come from `/data/options.json`, so they go through serde_json, not toml. No remote or user-supplied data reaches the TOML parser, so the hostile-input exposure is close to zero.

The crate family is maintained by Ed Page (epage, also a Cargo team member) under `github:toml-rs:maintainers`. It has 1.04 billion downloads, and Cargo itself uses it. There are no build scripts and no proc macros. `toml`, `toml_datetime`, `serde_spanned` and `toml_writer` are `#![forbid(unsafe_code)]`. `toml_parser` is `forbid(unsafe_code)` unless its opt-in `unsafe` feature is on, and `toml` does not turn it on. The only reachable `unsafe` is in `winnow`'s stream impls (§2). No RustSec advisories.

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| T1 | Info | Recursion limit of 80 (unless the `unbounded` feature is on) | Measured: arrays and inline tables nested 80 deep parse; 81 and 100,000 return `TOML parse error`; a 100,000-segment dotted key returns `recursion limit`. No stack overflow, no panic. | Never enable `unbounded`. |
| T2 | Low | Unknown keys are ignored silently by default | Measured: `bogus=1` inside an `[[exercise]]` entry gives `Ok`. A typo such as `catgeory=` would be silently dropped. With `#[serde(deny_unknown_fields)]` it becomes `unknown field 'catgeory', expected …` at line 4. | Put `#[serde(deny_unknown_fields)]` on the TOML-only table types (for example the top-level wrapper). If `Entry` is also used for JSON overrides, a typo there becomes a warning (Task 6 behaviour), which is fine too. |
| T3 | Info | Duplicate keys are rejected | `a=1\na=2` gives a parse error, and 1.1.7 fixed the missing duplicate check when deserialising into `Table`. | None. |
| T4 | Info | Fast-moving but stable 1.x line | 122 releases. Recent ones are small fixes (1.1.5–1.1.8 in 2026-09/10). The changelog shows earlier stack-overflow fixes (many `=`/`+`/`-`, very large files), which the recursion guard addresses. | Accept. Dependabot bumps are fine. |

## 1. Build-time code and features

No `build.rs` in `toml`, `toml_parser`, `toml_datetime`, `serde_spanned`, `toml_writer` or `winnow` (`build = false`). No proc macros. `file` finds no binary files in any of the packages.

The default features are `std, serde, parse, display`. `display` only adds `toml_writer`, which serialises, and we never write TOML. **Recommended (parse-only), verified to build and pass the probes:**

```toml
toml = { version = "1.1.8", default-features = false, features = ["parse", "serde", "std"] }
```

That gives:

```
toml v1.1.8+spec-1.1.0 (parse, serde, std)
├── serde_core v1.0.229
├── serde_spanned v1.1.2 ── serde_core
├── toml_datetime v1.1.2+spec-1.1.0 ── serde_core
├── toml_parser v1.1.5+spec-1.1.0 ── winnow v1.0.4
└── winnow v1.0.4
```

Don't enable these:
- `unbounded`: removes the recursion limit.
- `preserve_order` / `fast_hash`: pull in `indexmap`/`foldhash`.
- `debug`: pulls in `anstream`/`anstyle` and prints the parse trace to stderr.

Note: `Cargo.lock` still lists `indexmap`, `hashbrown` and `equivalent`, because the lockfile records toml's optional `indexmap` dependency. `cargo tree` confirms they aren't compiled. cargo-audit/cargo-deny will still scan them (hashbrown 0.17.1 is outside RUSTSEC-2024-0402's affected range `>=0.15.0, <0.15.1`).

## 2. `unsafe`

| Crate | Licence | MSRV | `unsafe` |
|---|---|---|---|
| toml | MIT OR Apache-2.0 | 1.85 | `#![forbid(unsafe_code)]` (`lib.rs:150`) |
| toml_parser | MIT OR Apache-2.0 | 1.85 | `#![cfg_attr(not(feature = "unsafe"), forbid(unsafe_code))]` (`lib.rs:18`). Its 131 grep hits are all paired `#[cfg(feature = "unsafe")]` / `#[cfg(not(...))]` unchecked-slicing alternatives. Not compiled for us. |
| toml_datetime, serde_spanned, toml_writer | MIT OR Apache-2.0 | 1.85 | `#![forbid(unsafe_code)]` |
| winnow | MIT | 1.65 | 47 hits in `src/stream/*`: implementations of the `unsafe fn next_slice_unchecked`/`peek_slice_unchecked` stream methods (`get_unchecked` on a caller-validated offset), plus `#[repr(transparent)]` `BStr`/`Bytes` transmutes. Callers must opt in to the unchecked methods, and `toml_parser` can't, because of `forbid`. |

All licences are on the allowlist. All MSRVs are ≤ 1.93. Owners: toml, toml_parser and the others are `epage` + `github:toml-rs:maintainers`. winnow is `epage` alone. Everything was published by epage.

## 3. Advisories

RustSec advisory-db (`640bf5a`, 2026-10-22) has no entries for toml, toml_parser, toml_datetime, serde_spanned, toml_writer or winnow. GitHub security advisories for `toml-rs/toml`: none.

## Re-audit

On a bump, check that `forbid(unsafe_code)` is still present, that `toml` still doesn't enable `toml_parser/unsafe`, and that `LIMIT` is still in place (`src/de/parser/mod.rs`, currently 80).
