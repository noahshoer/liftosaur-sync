# Security audit (light): chrono 0.4.45

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `chrono-0.4.45`, unpacked by `cargo fetch` in a throwaway project under `/tmp/audit-misc`.
  Its Cargo.lock checksum is `1aa79e62e7697b8e29b513a68abacf485adcd1fe8284a4316c5ae868e6633327`, which matches the SHA-256 of the cached `.crate` file.
  The `.cargo_vcs_info.json` sha1 is `170338250e836976a211e64728ec956e45e78a39`.
- **Verdict: ACCEPT WITH CONDITIONS** (the conditions are about which features to enable, not about defects)

## Summary

chrono 0.4.45 (2026-06-04) is the latest release in the 0.4 series. It is a mature, very widely used crate (about 905 M downloads) with active maintainers. With `default-features = false, features = ["std", "now"]` it pulls in only `num-traits` (and `autocfg` at build time). It has no proc macros, no FFI and no file or environment access, and only 2 `unsafe` sites get compiled, both trivial. The default features (`clock` and `wasmbind`) would add `iana-time-zone`, `windows-*`, `wasm-bindgen`/`js-sys` and several proc macros to Cargo.lock: 47 lockfile packages instead of 14. We don't need any of them, because the app works only in `DateTime<Utc>`, RFC 3339 and `NaiveDate`, and never uses the local time zone. RUSTSEC-2020-0159 (`localtime_r`) was fixed in 0.4.20, and the code it concerned isn't even compiled without `clock`.

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| C-F1 | Low | Default features pull in a large target-specific tree | `clock` adds `iana-time-zone` 0.1.65 with `android_system_properties`, `core-foundation-sys`, `iana-time-zone-haiku` (a `cc` build script), `js-sys`/`wasm-bindgen` (proc macros, syn 3) and `windows-core` (proc macros, syn 2). On aarch64 Linux only `iana-time-zone` itself compiles, but cargo-deny and cargo-audit scan the whole lockfile. It would also put two `syn` majors into the lock, which matters if `multiple-versions = "deny"`. | `default-features = false, features = ["std", "now"]`. Don't use `Local`. |
| C-F2 | Info | RUSTSEC-2020-0159, a potential segfault in `localtime_r` | Patched `>= 0.4.20`. 0.4.45 is unaffected. Since 0.4.20 chrono parses TZif itself (`offset/local/tz_info`) instead of calling libc. That module is `#[cfg(feature = "clock")]`, so our build doesn't compile it. | None. |
| C-F3 | Low | Panicking arithmetic APIs | `DateTime + TimeDelta` panics on overflow (`datetime/mod.rs:1535` `expect("DateTime + TimeDelta overflowed")`). `TimeDelta::seconds` panics if the value is out of range (`time_delta.rs:208-210`). `Utc::now()` panics if the clock is before 1970 (`offset/utc.rs:97`). API-supplied timestamps (Intervals.icu `elapsed_time`, Liftohistory `duration`) feed into this arithmetic. | Use `checked_add_signed`, `TimeDelta::try_seconds` and `DateTime::from_timestamp(..)` (which returns `Option`) wherever the operand comes from an API or the state file. |
| C-F4 | Info | `unsafe` | 11 non-comment `unsafe` lines in `src`. Only 2 compile with `std,now`: `NonZeroI32::new_unchecked` in `NaiveDate::from_yof` (`naive/date/mod.rs:1496`, guarded by debug-asserted invariants that every constructor upholds), and `unsafe impl Send for Date<Tz>` (`date.rs:468`, on the deprecated `Date` type, bounded on `Offset: Send`). The other 9 are in `offset/local/{windows,win_bindings,tz_data,tz_info}`, which need `clock`. | None. Don't use the deprecated `Date<Tz>`. |
| C-F5 | Info | Pi clock at boot | A Pi 4 has no RTC, so `Utc::now()` can be wrong until NTP syncs. HA OS syncs early, but this affects `first_seen` and the 30/180-min timers. | Use `std::time::Instant` for timeouts and backoff within a run. Don't trust `now` being before `sync_after`. |

## Details

- **Features** (`Cargo.toml`): `default = [clock, std, oldtime, wasmbind]`; `clock = [winapi, iana-time-zone, now]`; `now = [std]`; `std = [alloc]`; `serde` is optional (`serde` 1.0.99+, `default-features = false`). `oldtime` is an empty compatibility feature.
- **Recommended:** `chrono = { version = "0.4.45", default-features = false, features = ["std", "now"] }`.
  - `now` gives `Utc::now()`, which uses `SystemTime` (`offset/utc.rs:95-99`).
  - `std` gives `std::error::Error` impls, which `anyhow` needs.
  - Parsing (`DateTime::parse_from_rfc3339`, `NaiveDateTime::parse_from_str(..).and_utc()` for `start_date` without a `Z`) and `NaiveDate` need no extra features. I checked both in the scratch build.
  - **Add `serde`** only if `State` stores `DateTime<Utc>` directly rather than as `i64`/RFC 3339 strings. It adds no new crates, because `serde` is already in the tree through `serde_json`/`toml`.
- **Dependency tree** (`cargo tree -e normal,build --target all`, recommended features): `chrono 0.4.45 → num-traits 0.2.19 [build: autocfg 1.5.1]`.
  - `num-traits` (owner cuviper, 2024-05-03) has a `build.rs` that uses `autocfg` to probe `f64::total_cmp`. It only runs `rustc` on a probe expression. 1 non-comment `unsafe` line.
  - `autocfg` (no `unsafe`) compiles probe snippets with `$RUSTC`.
- **Build-time code:** chrono has no `build.rs` and isn't a proc macro. Everything that runs at build time comes from `num-traits`/`autocfg`, as above.
- **Ambient capability:** with `std,now` the only OS access is `SystemTime::now()`. The `File::open`/`env::var` calls (`offset/local/tz_data.rs`, Android/OHOS only) and `/etc/localtime` reading (`tz_info`) are behind `clock`.
- **Licence:** `MIT OR Apache-2.0` (`num-traits`, `autocfg`: `MIT OR Apache-2.0`). All are on the deny.toml allowlist.
- **MSRV:** chrono 1.62, num-traits 1.60, autocfg 1.0, all ≤ 1.93. The scratch project builds with `cargo +1.93 build --locked`.
- **Maintenance:** crates.io owners `lifthrasiir` and `github:chronotope:release-manager`. Recent releases are published by `djc`, and earlier ones by `pitdicker`. Cadence: 0.4.41 (2025-04), 0.4.42 (2025-09), 0.4.43 (2026-01), 0.4.44 (2026-02), 0.4.45 (2026-06). None of the recent releases is yanked. Repo `chronotope/chrono`: 3.9 k stars, not archived, last push 2026-09-07.
- **Advisories:** RustSec advisory-db (HEAD `640bf5a`, 2026-10-22, 1,329 advisories) lists only RUSTSEC-2020-0159 (patched). There's also RUSTSEC-2022-0049 for `iana-time-zone` (unsound on macOS/iOS, patched `>= 0.1.45`), but we don't pull that crate. GitHub security advisories for the repo: none.

## Conditions of use

1. Use `default-features = false, features = ["std", "now"]`, adding `"serde"` only if needed. Never enable `clock`, `wasmbind`, `unstable-locales` or `rkyv*`.
2. Use only `Utc`/`FixedOffset`. Never `Local` (it isn't compiled without `clock` anyway).
3. Use checked arithmetic on values that come from APIs or `state.json` (C-F3).

## Re-audit on bump

```sh
cargo tree -e normal,build --target all -p chrono     # expect only num-traits (+autocfg)
grep -rnw unsafe $R/src | grep -v '//'                # expect the same sites; check which are outside offset/local
grep -n 'RUSTSEC' -r advisory-db/crates/chrono
```
