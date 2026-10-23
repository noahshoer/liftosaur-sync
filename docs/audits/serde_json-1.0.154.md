# Security audit (light): serde_json 1.0.154

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `serde_json-1.0.154`, unpacked by `cargo fetch` in a scratch project (`rust-version = "1.93"`) at `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/serde_json-1.0.154`.
  Its Cargo.lock checksum is `e7e9cc8b1b85264074fbcc02a88680c4096b1e47df8f739dceb03bf482f04bd6`, which matches the crates.io API. The `.cargo_vcs_info.json` sha1 is `20572a6d066958542d79cdd6cf1a7cd9f0c8e1f8`. `src/` and `build.rs` are byte-identical to the `v1.0.154` tag commit (2026-10-11).
- **Verdict: ACCEPT WITH CONDITIONS** (the conditions are about how we use it. The crate itself has no issues.)

## Summary

serde_json is dtolnay's JSON format crate for serde (1.48 billion downloads, 186 releases since 2015). With default features it adds three small runtime dependencies: `itoa`, `zmij` (both dtolnay) and `memchr` (BurntSushi). The build script only chooses a 32- or 64-bit arithmetic limb width. The `unsafe` code is a handful of reviewed UTF-8, pointer-offset and buffer-write sites, plus a `raw_value` module that only compiles with that feature. Against hostile input it does well: the default **recursion limit of 128** stopped every deep-nesting attempt, unknown fields are skipped **iteratively** (100,000-deep junk inside a typed struct parsed without a stack overflow), and out-of-range numbers, duplicate struct fields, lone surrogates and trailing data are all rejected with errors rather than panics. The real risk is **memory amplification** when parsing into `serde_json::Value` with no size cap. That is handled by the conditions below.

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| J1 | Medium | Memory is proportional to input, with ~16–32× amplification for `Value` | Measured: an 8 MB `[0,0,…]` (4 M elements) parsed into `Value` in 33 ms. `size_of::<Value>() == 32`, so that's ≥128 MB plus `Vec` growth. The probe process peaked at 337 MB RSS. A 64 MiB string parsed in 5 ms (1× copy). A malicious or broken API response could OOM the Pi. | **C1**, **C2** |
| J2 | Info | Recursion limit (128) is on by default | Depth 127 OK, depth 128 and 100,000 return `recursion limit exceeded`. A depth-127 `Value` parses and drops fine on a 256 KiB thread stack in a debug build. | **C3**: never enable `unbounded_depth` or call `disable_recursion_limit()`. |
| J3 | Info | Unknown fields are skipped without recursion | `Deserializer::ignore_value` (`de.rs:1102`) is a loop with an explicit `scratch` stack of enclosing brackets, using 1 byte per nesting level. `{"id":"x","junk":[[[…100000…]]]}` into a typed struct gives `Ok`. Nested unknown objects inside array elements also work (100,000 deep). | Prefer typed structs for remote responses, so unknown fields cost no allocation. |
| J4 | Info | Error behaviour is as expected | Duplicate struct field → `duplicate field 'id'`. `1e400` → `number out of range`. A 23-digit integer into `i64` → type error. `"\ud800"` → error. Trailing garbage → `trailing characters`. No panics. | None. For `Value`/maps, duplicate keys keep the **last** value silently, which is irrelevant for typed structs. |
| J5 | Info | `unsafe` | §2 | None. |
| J6 | Info | Three releases in two days (1.0.152–154, 2026-10-10/11) | Changelog: `-0` sign under `arbitrary_precision`, an f32 infinity comparison, the invalid-UTF-8 error position, `preserve_order` on `no_std`. These are bug fixes, by the long-standing owner. | Accept. |

## Conditions of use

- **C1. Cap response size before parsing.** Read HTTP bodies with a byte limit (for example `ureq`'s body `limit(...)` / its read_json default. Confirm the exact default in the ureq audit) of a few MB for Intervals.icu, Strava and Liftosaur JSON. Cap `/data/options.json` and `state.json` reads too (for example 1 MB via `Read::take`), because a corrupted state file then fails cleanly into the Task 9 recovery path.
- **C2. Deserialise remote responses into typed structs, not `serde_json::Value`.** Use `Option<T>`/`#[serde(default)]` for fields that might be missing, and don't use `#[serde(flatten)]`/`#[serde(untagged)]` on these types (see serde audit S5). Don't add `deny_unknown_fields` to **remote** types: the APIs add fields over time. It *is* fine on `Options`, if you want typos in user config caught.
- **C3. Features:** default only (`std`). Don't enable:
  - `arbitrary_precision`: it changes `Number` into a string-backed type, is a long-standing source of `flatten`/`untagged` bugs, and we have no >f64 numbers.
  - `unbounded_depth`: it removes the recursion limit.
  - `raw_value`: not needed, and it adds the `transmute` sites.
  - `preserve_order`: not needed, and it pulls in `indexmap` + `hashbrown` + `foldhash`.
  - `float_roundtrip`: not needed. It's harmless, but slower.
- **C4.** Use `from_slice`/`from_str` on the size-capped buffer, or `from_reader` on a `Read::take`-limited reader. Don't call `from_reader` on an unbounded stream.

## 1. Build script and dependencies

`build.rs` (31 lines): emits `rustc-check-cfg` and `cfg(fast_arithmetic = "64")` on aarch64, x86_64 and other 64-bit targets, otherwise `"32"`. It reads `CARGO_CFG_TARGET_*` only. It runs no processes and does no file I/O. Not a proc macro.

```
serde_json v1.0.154 (default, std)
├── itoa v1.0.18
├── memchr v2.8.3 (alloc, std)
├── serde_core v1.0.229 (alloc, result, std)
└── zmij v1.0.23
```

(`serde` itself shows up in serde_json's manifest only under `[target."cfg(any())"]`, a version-coordination trick, so it is never compiled there.)

| Crate | Owners | Licence | MSRV | build.rs | `unsafe` |
|---|---|---|---|---|---|
| serde_json 1.0.154 | dtolnay, github:serde-rs:publish | MIT OR Apache-2.0 | 1.71 | yes (above) | §2 |
| itoa 1.0.18 | dtolnay | MIT OR Apache-2.0 | 1.68 | no | 13 (integer → decimal into a fixed stack buffer; a long-standing crate) |
| zmij 1.0.23 | dtolnay | MIT | 1.71 | yes: `rustc --version` probe and `OPT_LEVEL` → cfgs only | 70 (float → shortest decimal, the successor to `ryu`, created 2025-12) |
| memchr 2.8.3 | BurntSushi | Unlicense OR MIT (MIT on our allowlist) | 1.61 | no | 340 (SIMD byte search, NEON on aarch64; one of the most widely used crates in the ecosystem, also used by `regex`) |
| serde_core 1.0.229 | see the serde audit | | | | |

All MSRVs are ≤ 1.93. All licences are allowed: cargo-deny accepts an `OR` expression when one branch is allowed. No package contains binary files.

## 2. `unsafe` in serde_json (16 grep hits: 9 sites in default features, 4 behind `raw_value`, 3 in comments or docs)

- `read.rs:500`: `chunk.as_ptr().offset_from(self.slice.as_ptr())`. The chunk comes from `rest.chunks_exact` of `self.slice`, so both pointers are in the same allocation. Sound.
- `read.rs:749`: `from_utf8_unchecked` in `SliceRead`/`StrRead` after the bytes are validated, or the input was already a `&str`.
- `read.rs:1042`: writes an encoded UTF-8 char (≤ 4 bytes) after `scratch.reserve(4)`, then `set_len`. The SAFETY comment holds: `encoded_len ≤ 4`, and `n < 0x80` returns early.
- `ser.rs:2106/2127/2250/2269`, `value/mod.rs:231`: `from_utf8_unchecked`/`String::from_utf8_unchecked` on serializer output. The serializer only emits `str` input plus ASCII escapes. `ser.rs:2121` is `unreachable_unchecked` in an escape-table match whose arms cover every table value.
- `raw.rs:123/127/131/228`: `transmute`s between `str` and `#[repr(transparent)] RawValue`, compiled **only** with `raw_value` (`lib.rs:440`). Excluded by C3.

## 3. Advisories

RustSec advisory-db (`640bf5a`, 2026-10-22) has nothing for serde_json, itoa, zmij or memchr. GitHub security advisories for `serde-rs/json`: none. (Unrelated crates `json` and `json5` do have advisories. Don't confuse them.)

## Recommended dependency line

```toml
serde_json = "1.0.154"
```

## Re-audit

On a bump, check the `build.rs` diff, any new default features, and new `unsafe` (`grep -rn unsafe src`), and confirm the recursion limit is still 128 (`de.rs`, `remaining_depth`).
