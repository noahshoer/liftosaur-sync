# Security audit (light): serde 1.0.229 + serde_derive 1.0.229

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io packages, unpacked by `cargo fetch` in a scratch project (`edition = "2024"`, `rust-version = "1.93"`, `serde = { version = "1", features = ["derive"] }`) under `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/`.

  | Crate | Cargo.lock checksum (SHA-256, matches the crates.io API) | `.cargo_vcs_info.json` sha1 |
  |---|---|---|
  | serde 1.0.229 | `4148590afebada386688f18773da617792bf2ef03ffc1e4cbd2b1d45b023e0ba` | `7fc3b4c30c94f73a96ebd1553f2b090d928fc3a8` (`serde`) |
  | serde_core 1.0.229 | `67dca2c9c51e58a4791a4b1ed58308b39c64224d349a935ab5039aa360942a48` | same commit (`serde_core`) |
  | serde_derive 1.0.229 | `e7a5d71263a5a7d47b41f6b3f06ba276f10cc18b0931f1799f710578e2309348` | same commit (`serde_derive`) |

  The published `src/` of all three, and serde's `build.rs`, are byte-identical to the `v1.0.229` tag commit `7fc3b4c` ("Release 1.0.229", 2026-07-18).
- **Verdict: ACCEPT**

## Summary

serde is the de-facto Rust serialisation framework, maintained by dtolnay since 2014 (1.57 billion downloads). 1.0.229 splits the trait definitions into `serde_core` (as since 1.0.220) and moves `serde_derive` to syn 3. The runtime crates have 2 trivially sound `unsafe` blocks each. Both build scripts only probe `rustc --version` and write a one-line module into `OUT_DIR`. `serde_derive` is a proc macro (build-time code execution by definition) built from source: **no precompiled binary ships in the package**. That was the 2023 problem, and it was reverted in 1.0.184. No RustSec advisories. Every licence is on the allowlist.

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| S1 | Info | `serde_derive` is a proc macro (build-time code execution) | Runs at compile time with the build user's privileges, both in CI and locally. Source is identical to the tag, and the package has no binaries (§1). | Accept. Keep `Cargo.lock` + `--locked`, so a bump is a reviewed PR. |
| S2 | Info | 2023 precompiled-binary episode | Not present in 1.0.229 (§1). | None. The `.crate` contents are worth a quick look on major bumps. |
| S3 | Info | Build scripts in `serde` and `serde_core` | Read-only `rustc --version` probe, `cargo:rustc-cfg` lines, and a generated `private.rs` in `OUT_DIR`. No network and no writes outside `OUT_DIR`. | Accept. |
| S4 | Info | Proc-macro toolchain uses **syn 3** | If another dependency still uses syn 2, both get compiled. That costs build time only, and cargo-deny `multiple-versions` may warn. | Accept. Allow the duplicate in `deny.toml` if needed. |
| S5 | Low | `#[serde(flatten)]` / `#[serde(untagged)]` buffer the whole subtree into an internal `Content` tree | With hostile or huge remote JSON this costs extra allocation and gives poorer errors. It is also known to misbehave with `serde_json/arbitrary_precision`. | Don't use `flatten`/`untagged` on remote-response types. Use plain structs with `Option`/`#[serde(default)]`. |

## 1. Build-time code

- `serde/build.rs` and `serde_core/build.rs`: they run `$RUSTC --version`, parse the minor version, and emit `rustc-check-cfg`/`rustc-cfg` flags (`no_core_net`, `no_diagnostic_namespace`, `no_serde_derive` below 1.71, and similar). They also write `OUT_DIR/private.rs`, a versioned `__private229` re-export. With our 1.93+ toolchain none of the `no_*` cfgs fire. There is no other I/O.
- `serde_derive`: `proc-macro = true`, `build = false`. Its deps are `proc-macro2` 1.0.107 (build.rs: a version probe plus a compile probe of `src/probe/*.rs` into `OUT_DIR`, deleted afterwards), `quote` 1.0.47 (build.rs: version probe), `syn` 3.0.9 (no build.rs) and `unicode-ident` 1.0.27. All are owned by dtolnay.
- **The precompiled-binary episode.** From serde_derive 1.0.172 (2023-07) to 1.0.183, the crate shipped a precompiled `serde_derive-x86_64-unknown-linux-gnu` macro binary that was used by default on that target. The community objected (it couldn't be audited or reproduced, and distros wouldn't take it). It was removed in **1.0.184 (2023-08-19)**, and every later release builds from source. Checked for 1.0.229: `find serde_derive-1.0.229 -type f` lists 37 files, all Rust source, `Cargo.*`, licences and READMEs. `file` reports no non-text file in serde, serde_core, serde_derive, proc-macro2, quote or syn. `src/` matches the git tag. No `include_bytes!` or `Command::new` exists in `serde_derive/src`.

## 2. Dependency tree

```
serde v1.0.229                 (features: default, derive, std)
├── serde_core v1.0.229        (alloc, result, std)
└── serde_derive v1.0.229 (proc-macro)
    ├── proc-macro2 v1.0.107 ── unicode-ident v1.0.27
    ├── quote v1.0.47
    └── syn v3.0.9
```

| Crate | Licence | MSRV | build.rs | `unsafe` (grep, `src/`) |
|---|---|---|---|---|
| serde | MIT OR Apache-2.0 | 1.56 | yes (§1) | 2, both `str::from_utf8_unchecked`, and both sound. `format.rs:16` is `Buf::as_str`, and `Buf`'s `fmt::Write` only ever copies whole `&str`s in. `ser/impls.rs:821` is the human-readable `Ipv4Addr` serialiser, whose buffer holds only ASCII digits and dots. |
| serde_core | MIT OR Apache-2.0 | 1.56 | yes (§1) | the same 2 sites |
| serde_derive | MIT OR Apache-2.0 | 1.71 | no | 0 |
| proc-macro2 / quote / syn / unicode-ident | MIT OR Apache-2.0 (unicode-ident: `AND Unicode-3.0`) | 1.71 | proc-macro2, quote | build-time only, not in the binary |

All licences are on the `deny.toml` allowlist. All MSRVs are at or below our 1.93. **Owners:** `dtolnay` and `github:serde-rs:publish` for serde, serde_core and serde_derive. All recent releases were published by dtolnay. Release cadence: 1.0.224–1.0.228 in 2025-09, then 1.0.229 on 2026-07-18 (syn 3).

## 3. Advisories

RustSec advisory-db (cloned 2026-10-22, `640bf5a`) has no entries for `serde`, `serde_core`, `serde_derive`, `proc-macro2`, `quote`, `syn` or `unicode-ident`. GitHub security advisories for `serde-rs/serde`: none.

## 4. Recommended dependency line

```toml
serde = { version = "1.0.229", features = ["derive"] }
```

Leave out `rc` (it serialises `Rc`/`Arc` by value, which we don't need), `unstable`, and `serde_derive/deserialize_in_place`. Hostile-input behaviour belongs to the format crate: see `serde_json-1.0.154.md` and `toml-1.1.8.md`.

## Re-audit

On a bump, check: whether the `.crate` has any non-source file, the build-script diff, and whether the owners changed.
