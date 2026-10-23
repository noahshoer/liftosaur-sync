# Security audit (light): flate2 1.1.10

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `flate2-1.1.10`, unpacked by `cargo fetch` in a throwaway project under `/tmp/audit-misc`.
  Its Cargo.lock checksum is `6e634e2e0ebac1ee034020da1ca582e17ffe4e0f5e985823721e168928136dcb`, which matches the SHA-256 of the cached `.crate` file. The `.cargo_vcs_info.json` sha1 is `ed93d4fc60eaf876c6aded741bf992d524551930`.
  Its backend `miniz_oxide-0.9.1` has checksum `b63fbc4a50860e98e7b2aa7804ded1db5cbc3aff9193adaff57a6931bf7c4b4c` (vcs `4e58239`).
- **Verdict: ACCEPT WITH CONDITIONS** (an output cap is mandatory, see the Conditions)

## Summary

flate2 1.1.10 (2026-08-28) is the latest 1.x release. It is maintained under the `rust-lang` GitHub org. Its **default backend is `miniz_oxide`, which is pure Rust and `#![forbid(unsafe_code)]`**. There is no C code, no `cc` build script and no FFI. We keep that default: the alternatives buy speed we don't need for files of at most 4 MiB, at the cost of a lot of `unsafe` (zlib-rs) or C and a build script (libz-sys). The gzip header parser is bounded (64 KiB per header field), and corrupt, truncated and bad-CRC streams all return `Err`.

The real risk is the one you'd expect: **a decompression bomb**. A 4 GiB run of zeros compresses to 4,171,638 bytes (1030:1), which is *under* the 4 MiB compressed cap. Decompressed without a cap, a 1 GiB bomb (1 MB on the wire) took 1.1 GB RSS. With an output cap of `take(4 MiB + 1)` the same input stopped after 4,194,305 bytes in under 1 ms. A second finding concerns ureq: its own `gzip` feature decompresses `Content-Encoding: gzip` **after** its body `limit()` has been applied, so that limit doesn't protect against a bomb (F-F2).

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| F-F1 | **Medium** | Decompression bomb: output is unbounded unless the caller caps it | Measured on aarch64 (Apple silicon, release build): 4 GiB of zeros → 4,171,638 B gzip, fits under the 4 MiB input cap. Uncapped `read_to_end` of a 1 GiB bomb: max RSS 1,115,275,264 B. That's an OOM kill on a Pi 4 container, and it can't be caught. Capped `GzDecoder::new(r).take(CAP + 1).read_to_end(..)` on the 4 GiB bomb read 4,194,305 B in 0.96 ms, with 19.8 MB RSS for the whole process including generating the bomb. | Always read the decoder through `.take(MAX_FIT + 1)`. If you get more than `MAX_FIT` bytes, treat it as `Permanent` "too large". Also keep the compressed-side cap of 4 MiB. |
| F-F2 | **Medium** (cross-crate, ureq 3.5.0) | ureq's transparent gzip runs *after* its `limit()` | `ureq-3.5.0/src/body/mod.rs:521` wraps the **raw** body in `LimitReader`, then `:801-807` puts `MultiGzDecoder` *on top* of it when `Content-Encoding: gzip` is set. `limit(4 MiB)` (and the 10 MB default of `read_to_vec`/`read_json`) therefore caps the *compressed* bytes, and the decoded output is unbounded. ureq's default features are `["rustls", "gzip"]`, and with `gzip` on it sends `Accept-Encoding: gzip`. If Intervals.icu (or a MITM-free but misbehaving upstream) serves the file with `Content-Encoding: gzip`, (a) the body is already gunzipped, so our own `GzDecoder` sees FIT bytes and fails with "invalid gzip header", and (b) the size cap doesn't hold. | This matches `docs/audits/ureq-3.5.0.md` U3/U8, which already require `default-features = false` (no `gzip`). With that, flate2's only consumer is our code, and all gunzipping happens under F-F1's cap. If `gzip` is ever turned back on, put `.take(MAX + 1)` on the **decoded** reader, not just `limit()`. |
| F-F3 | Low | `GzDecoder` reads only the first gzip member and silently ignores trailing bytes | Verified: two concatenated members → `GzDecoder` returns only the first. First member plus `"GARBAGE"` → `Ok` with the garbage ignored. `MultiGzDecoder` returns both members, but fails with `UnexpectedEof` on trailing garbage. | Use `GzDecoder` (a single member is what Intervals.icu serves). It doesn't need to be stricter: the FIT pre-scan (rustyfit audit C4) validates the result. |
| F-F4 | Info | Non-gzip input | A plain FIT returns `Err(InvalidInput, "invalid gzip header")`, and empty input returns `Err(UnexpectedEof)`. | Sniff the magic bytes `1f 8b`. Gunzip only if they're present, otherwise treat the body as raw FIT, still under the 4 MiB cap. This also covers F-F2(a). |
| F-F5 | Info | Error and integrity handling | A truncated stream gives `Err(UnexpectedEof)`. A flipped CRC32 or ISIZE trailer gives `Err(InvalidInput, "corrupt gzip stream does not have a matching checksum")`. A 200 KB FNAME without a NUL gives `Err("gzip header field too long")` (`gz/mod.rs:23` `MAX_HEADER_BUF = 65535`; FEXTRA is limited to `u16` `xlen`). The CRC is only checked at EOF, so it isn't checked when the cap stops the read. That's fine, because we reject the file in that case anyway. | Map every decode error to `Permanent`. |
| F-F6 | Info | Backend selection is global (feature unification) | If any crate in the graph enables `zlib`, `zlib-ng`, `zlib-rs` or `cloudflare_zlib` on flate2, `any_zlib`/`any_c_zlib` takes precedence over miniz_oxide (`ffi/mod.rs:63-77`) and would bring in C or a lot of `unsafe`. With ureq's `gzip` off, only our code depends on flate2. | Check `cargo tree -e features -i flate2` whenever dependencies change. Expect only `default`, `rust_backend`, `miniz_oxide` and `runtime_detection`. |
| F-F7 | Info | `zlib-rs` 0.6.8 shows up in Cargo.lock but isn't compiled | flate2's `runtime_detection = ["zlib-rs?/std", ..]` is a weak-dependency feature, and Cargo records weak optional dependencies in the lockfile. `cargo tree -i zlib-rs --target all` prints nothing. cargo-deny and cargo-audit will still scan it. Its licence is `Zlib`, which is allowed. | None. Don't be surprised by it. |

## Details

- **Features:** `default = [rust_backend, runtime_detection]`. `rust_backend = [miniz_oxide, any_impl]`, where `miniz_oxide` turns on `dep:miniz_oxide` (with `simd`) and `dep:crc32fast`. The C backends are `zlib`, `zlib-default`, `zlib-ng`, `zlib-ng-compat` and `cloudflare_zlib`, and the pure-Rust but `unsafe`-heavy one is `zlib-rs`.
- **Backend choice:** **miniz_oxide** (the default) is pure Rust with `forbid(unsafe_code)` and is the most widely deployed backend (about 1 G downloads). `zlib-rs` (Trifecta Tech, about 195 M downloads, actively fuzzed) is faster but full of `unsafe`. C zlib needs `libz-sys`, a `cc` build script and FFI. Throughput doesn't matter for files of at most 4 MiB, so **stay on the default** and **don't** enable the other backends.
- **Recommended:** `flate2 = "1.1.10"`, with default features.
- **Dependency tree** (`cargo tree -e normal,build --target all`):
  ```
  flate2 v1.1.10
  ├── crc32fast v1.5.3 ── cfg-if v1.0.5
  └── miniz_oxide v0.9.1
      ├── adler2 v2.0.1
      └── simd-adler32 v0.3.10
  ```
  That's 5 transitive crates.

  | Crate | Owner(s) | Licence | build.rs | `unsafe` |
  |---|---|---|---|---|
  | flate2 1.1.10 | Byron, joshtriplett, rust-lang-owner | MIT OR Apache-2.0 | no | 36 lines. With rust_backend only 3 sites are live (see below). The rest is `ffi/c.rs` and zlib-gated paths in `mem.rs`. |
  | miniz_oxide 0.9.1 | oyvindln, Frommi | MIT OR Zlib OR Apache-2.0 | no | `#![forbid(unsafe_code)]` |
  | crc32fast 1.5.3 (2026-10-20) | srijs | MIT OR Apache-2.0 | **yes**: runs `$RUSTC --version` to emit `cfg`s for stable ARM CRC32 and VPCLMULQDQ intrinsics | aarch64 `__crc32*` kernels behind runtime `is_aarch64_feature_detected!("crc")`. The 1.5.2→1.5.3 diff only refactors state passing; the kernels are unchanged. |
  | adler2 2.0.1 | oyvindln | 0BSD OR MIT OR Apache-2.0 | no | `#![forbid(unsafe_code)]` |
  | simd-adler32 0.3.10 | mcountryman | MIT | no | NEON kernel (`imp/neon.rs`, 1 block, gated on `target_feature = "neon"`). It's only used for zlib-format Adler-32, not gzip, so it's compiled but not used on our path. |
  | cfg-if 1.0.5 | — | MIT OR Apache-2.0 | no | none |

- **Live `unsafe` in flate2 (rust_backend):**
  - `ffi/mod.rs:7-12` `initialize_buffer` zero-fills a `[MaybeUninit<u8>]` and then casts it to `[u8]`. Sound.
  - `mem.rs:388` and `:529` (`compress_vec` and `decompress_vec`) call `write_to_spare_capacity_of_vec` (`:647`). That function does `set_len(cap.min(len + bytes_written))`: the length is clamped to capacity, and because the spare capacity was zero-filled, every byte it exposes is initialised. Sound.
  - Our path (`read::GzDecoder` → `Decompress::decompress`) goes through `initialize_buffer` only.
- **Build-time code:** only crc32fast's `build.rs` (the rustc version probe).
- **Licences:** all are on the allowlist (`MIT`/`Apache-2.0`/`Zlib`/`0BSD OR ...`). simd-adler32 is MIT only.
- **MSRV:** flate2 1.67, crc32fast 1.63, cfg-if 1.32. miniz_oxide, adler2 and simd-adler32 don't declare one. All build with `cargo +1.93 build --locked`.
- **Maintenance:** flate2 releases have been frequent (1.1.3–1.1.10 between 2025-10 and 2026-08). 1.1.6 and 1.1.7 (2025-12-05) were **yanked** for unintentional API breakage (issues #515 and #517), not for security. Repo `rust-lang/flate2-rs`: 1.1 k stars, last push 2026-10-18. Releases go through protected-branch PRs managed in the rust-lang team repo (`MAINTENANCE.md`).
- **Advisories:** RustSec advisory-db (HEAD `640bf5a`, 2026-10-22) has no entries for flate2, miniz_oxide, crc32fast, adler2, simd-adler32 or cfg-if. GitHub security advisories for `rust-lang/flate2-rs` and `Frommi/miniz_oxide`: none.
- **Test harness:** the scratch project's `src/main.rs` (in `/tmp/audit-misc/proj`) runs the bomb, member, trailing-data, truncation, CRC and header cases above.

## Conditions of use

1. **Cap the decompressed output** at `MAX_FIT` (4 MiB, rustyfit audit C5) with `.take(MAX_FIT + 1)` and reject anything longer. Keep the 4 MiB cap on compressed input as well.
2. Gunzip only when the body starts with `1f 8b`. Otherwise pass the bytes through as raw FIT, under the same cap.
3. Use default features (miniz_oxide). Don't enable any zlib, zlib-ng or zlib-rs feature. Check `cargo tree -e features -i flate2` when dependencies change.
4. **Keep ureq's `gzip` feature off**, as the ureq audit's condition 1 requires (F-F2). Remember that ureq's `limit(n)` is exclusive (ureq U2): use `limit(MAX_FIT + 1)` for the compressed side.
5. Treat any gunzip error as `Permanent` for that activity.

## Re-audit on bump

```sh
cargo tree -e normal,build --target all -i flate2; cargo tree -e features -i flate2
grep -n 'forbid(unsafe_code)' $(ls -d ~/.cargo/registry/src/*/miniz_oxide-*/src/lib.rs)
grep -rnw unsafe flate2-*/src | grep -v '//'          # expect ffi/c.rs, ffi/mod.rs, mem.rs
grep -n MAX_HEADER_BUF flate2-*/src/gz/mod.rs
# re-run /tmp/audit-misc/proj (bomb + edge cases) against the new version
```
