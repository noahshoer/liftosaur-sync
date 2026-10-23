# Security audit (light): ureq 3.5.0

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Scope:** light audit (Task 2), as a blocking HTTPS client for Intervals.icu, Strava and the HA Supervisor API. It includes the transitive TLS stack (rustls, ring, webpki-roots), which is where most of the build-script and `unsafe` surface is.
- **Source audited:** the crates.io package `ureq-3.5.0`, as unpacked by `cargo fetch` in a scratch project (`/tmp/audit-ureq`) at
  `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/ureq-3.5.0`.
  Its Cargo.lock checksum is `cde93ff05758a36a851a398a772b0b79a82fbe3c186a46f2696baaeaf84c966b`. It matches the `.crate` file's SHA-256 and the crates.io API.
  The `.cargo_vcs_info.json` sha1 is `1bd534e8aa0ba34b1ab33982c109de09ba54e1cb`. That is the commit the `3.5.0` Git tag points to, and the published `src/` is byte-identical to the tag (`diff -r`).
- **Feature set audited:** `default-features = false, features = ["rustls", "json"]`. The resolved graph is for `aarch64-unknown-linux-gnu`; `-musl` gives the same graph.
- **Verdict: ACCEPT WITH CONDITIONS**

## Summary

ureq is a mature, widely used crate: created in 2018, about 238 M downloads, and 117 releases. ureq itself and its protocol crate `ureq-proto` are `#![forbid(unsafe_code)]` and have no build script. The TLS stack is rustls 0.23.46 (also `forbid(unsafe_code)`) with **ring** 0.17.14 as the crypto provider. aws-lc-rs is not in the tree. ring is the one significant native surface: its build script compiles 17 C files plus pregenerated aarch64 assembly with `cc`. That is expected and well understood. With the audited features, the graph has **30 crates**, 3 of them build-time only. All 30 licences are on our `deny.toml` allowlist. As of advisory-db `640bf5a` (2026-10-22), **no RustSec advisory affects the resolved versions**. Several TLS advisories from this month (rustls ×4, rustls-webpki, webpki-roots) are fixed in exactly the versions ureq 3.5.0's lower bounds require. MSRV is 1.85, and a probe project built and ran on Rust 1.93.

The conditions are about ureq's defaults, not defects. It has **no timeouts by default**. The body-size limit only applies to the `read_*` convenience methods (and the limit is exclusive). It reads proxy settings from the environment. It allows plain `http://` unless told otherwise. The plan's ureq gotchas are confirmed, except one: ureq *does* now have a multipart builder, but it is experimental and outside semver (U7).

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| U1 | **Medium** | No timeouts by default | `Timeouts::default()` is all `None` except `await_100` (1 s). A stalled Strava or Intervals connection would block the poll thread forever, with no error, retry or notification. | Set `timeout_global` on every Agent (for example 60–120 s). Optionally also set `timeout_connect` (around 10 s). |
| U2 | **Medium** | Body limits only apply to some read paths, and the limit is exclusive | `read_to_vec`, `read_to_string` and `read_json` cap at 10 MiB (`10 * 1024 * 1024`). But `with_config()` without `.limit()` is **unlimited** (`u64::MAX`), and so are `as_reader()` and `into_reader()`. Measured: with `limit(N)`, a body of exactly N bytes fails with `BodyExceedsLimit(N)` (with or without Content-Length). The real maximum is N−1 bytes. There is no preallocation from Content-Length, so a lying header can't force a large allocation. | Always call `.limit(..)` when using `with_config()`, and never read a response through an unlimited reader. Size the limit one byte above the largest body you accept, or just generously. |
| U3 | **Medium** (our code) | Gzip bomb on the Intervals.icu `/file` download | The ureq limit counts the *compressed* bytes, but our flate2 gunzip has no limit. A small response could inflate to gigabytes and OOM the Pi. This ties in with rustyfit audit C4 (input size cap). | Cap the *decompressed* size too (for example `Read::take` on the decoder, then reject if the cap was hit). |
| U4 | Low | `http_status_as_error` defaults to `true` | With the default, 4xx/5xx become `Error::StatusCode(u16)`, and the body (for example Strava's error JSON or the "duplicate of activity N" text) is lost. Confirms the plan's gotcha. | `http_status_as_error(false)` on every Agent, then classify `response.status()` yourself. Transport errors still arrive as `Err`. |
| U5 | Low | Proxy taken from the environment | `Config::default()` uses `Proxy::try_from_env()`, which reads `ALL_PROXY`, `HTTPS_PROXY`, `HTTP_PROXY` (and lowercase forms) and honours `NO_PROXY`. Anything that sets these in the container would route the API keys through that proxy. | Set `.proxy(None)` unless you want proxy support (user decision). |
| U6 | Low | `https_only` defaults to `false` | Without it, a typo or a redirect to `http://` would send credentials in clear text. The HA Supervisor endpoint (`http://supervisor/...`) is legitimately plain HTTP. | Use two Agents: an external one with `https_only(true)` for Intervals and Strava, and a Supervisor one without it. Verified: `https_only(true)` rejects `http://` with `Error::RequireHttpsOnly`. |
| U7 | Info | The plan says "no multipart builder". That has been outdated since 3.1.3. | `ureq::unversioned::multipart::{Form, Part}` exists behind the `multipart` feature. Its module docs say it **does not follow semver** and is experimental. It adds 4 crates (`getrandom 0.4`, `mime_guess`, `mime`, `unicase`), and `Form::new()` panics if `getrandom` fails. | Keep the `multipart` feature **off** and write the small RFC 7578 body yourself as planned. There are three fields (`file`, `data_type`, `external_id`), so it is a small job. Use a fixed or random boundary, and check that it doesn't appear in the payload. |
| U8 | Info | The `gzip` feature must stay off | With `gzip` on (it is a default feature), ureq sends `Accept-Encoding: gzip` and transparently decodes `Content-Encoding: gzip` responses. If Intervals serves the file that way, our own gunzip step would then get plain bytes and fail. With `gzip` off, ureq sends no `Accept-Encoding` and passes any encoding through untouched (`body/mod.rs:801-818`). | `default-features = false`. Optionally check for the gzip magic `1f 8b` before gunzipping. That makes the step robust either way, and lets non-gzipped originals go straight to the FIT and GPX handling. |
| U9 | Info | Redirects | Up to 10 are followed by default. `Authorization` is **dropped on every redirect** (`RedirectAuthHeaders::Never`), which is the safe default. | Keep the defaults. If an Intervals download returns 401 only after a redirect, look at `RedirectAuthHeaders::SameHost` before anything wider. |
| U10 | Info | Logging is redacted | At `debug`, ureq logs method, host and an allowlist of headers. `Authorization` and other headers are counted, not printed, and the path and query show as `/******`. The full path and query are logged only at `trace`. Bodies are never logged. | Don't run `ureq=trace` in production. Keep secrets out of URLs (Strava tokens in a header, the refresh in a form body). |
| U11 | Info | TLS stack: ring, not aws-lc-rs | ureq's `rustls` feature means `rustls-no-provider + _ring + rustls-webpki-roots`. ureq picks the provider in this order: explicit `TlsConfig`, the process default, then ring. ring 0.17.14 was released 2025-03 and hasn't had a release in 19 months, but its repo is active (last commit 2026-07-17). The RustSec "unmaintained" notice RUSTSEC-2025-0007 was **withdrawn**. aws-lc-rs would pull in the much larger AWS-LC C code base (cmake/cc build, optionally bindgen). | Keep ring, which is ureq's default. Don't enable rustls' `aws-lc-rs` feature anywhere in the tree. |
| U12 | Info | Bundled roots are frozen into the binary | `webpki-roots` (Mozilla's CA set, CDLA-Permissive-2.0) is compiled in. No `ca-certificates` is needed in the image, but CA distrusts only take effect after a rebuild. RUSTSEC-2026-0381 (emSign) landed 3 days ago and is fixed in 1.0.10. | Keep Dependabot (cargo) and CI `cargo audit` on, and rebuild and republish the image when TLS advisories hit. |
| U13 | Info | Very fresh release | 3.5.0 was published 2026-10-21, the same day as 3.4.3 (the "Update TLS dependencies for security fixes" bump). 3.5.0 includes two "technically breaking" changes: EINTR retry, and the reworked timeout budgets. | Fine with the Cargo.lock and `--locked` rule. Require at least 3.4.3 so the patched TLS floor is guaranteed. |
| U14 | Info | `read_json` limit | `read_json` uses the same 10 MiB cap. That is plenty for activity lists and upload status. | None. |

## 1. Maintenance and provenance

- Repo `algesten/ureq`: 2,215 stars, 42 open issues, not archived, last pushed 2026-10-22. No GitHub security advisories have been published for the repo.
- crates.io owners are `algesten` (Martin Algesten, the author, who published every 3.x release) and `jsha` (Jacob Hoffman-Andrews, a rustls maintainer). `ureq-proto` has the same two owners.
- The 3.x line started 2025-01 and has had steady releases: 3.4.0 on 2026-08-08, 3.4.1 on 09-06, 3.4.2 on 09-13, then 3.4.3 and 3.5.0 on 10-21. Nothing is yanked. The last 6 months have 37 commits from 16 authors, 21 of them from algesten.

## 2. Dependency tree (normal + build, aarch64-linux)

```
ureq 3.5.0 ─┬ base64 0.23.1, log 0.4.34, percent-encoding 2.3.2, utf8-zero 0.8.1
            ├ ureq-proto 0.6.5 ─ http 1.5.0 (bytes 1.12.1, itoa), httparse 1.10.1
            ├ serde 1.0.229 / serde_core, serde_json 1.0.154 (itoa, memchr, zmij)
            ├ rustls 0.23.46 ─┬ ring 0.17.14 ─ cfg-if, getrandom 0.2.17, libc 0.2.191, untrusted 0.9.0
            │                 │   [build] cc 1.9.0 ─ find-msvc-tools 0.1.15, shlex 2.0.1
            │                 ├ rustls-webpki 0.103.16, rustls-pki-types 1.15.1
            │                 └ once_cell, subtle 2.6.1, zeroize 1.9.1
            └ webpki-roots 1.0.10
```

This is 30 crates including ureq. Several overlap with crates the plan already uses: `serde`, `serde_json`, `log`, and `base64 0.23`, which is the same major as the plan's `base64 0.23`. Adding the `multipart` feature would bring it to 34 (U7). The default features would add `flate2` and its backend.

## 3. `unsafe`, build scripts, proc macros, FFI

These are textual counts of `unsafe` in `src/`, excluding comments.

| Crate | `unsafe` | Build script | Native code / FFI |
|---|---|---|---|
| ureq, ureq-proto | 0, `forbid(unsafe_code)` | none | none |
| rustls 0.23.46 | 0, `forbid(unsafe_code)` | 13 lines; only sets a nightly `read_buf` cfg | none |
| rustls-webpki 0.103.16, untrusted, webpki-roots | 0 (`forbid` in webpki-roots) | none | none |
| rustls-pki-types | 1 (`transmute [u16;8]→[u8;16]` for an IPv6 address, sound) | none | none |
| **ring 0.17.14** | ~228 | **yes, 935 lines.** It compiles 17 C files and the pregenerated `*-linux64.S` (aarch64) with `cc`. Perl/NASM are only used in git checkouts or with `RING_PREGENERATE_ASM`, and the shipped prebuilt `.o` files are Windows-only. No network access, and it writes only to `OUT_DIR`. | C/asm crypto (BoringSSL-derived) |
| libc, getrandom 0.2 | many (FFI bindings) | libc: a rustc-version probe | syscalls (`getrandom`) |
| http, bytes, httparse, memchr, serde_json, zmij, itoa, once_cell, zeroize, base64, … | ordinary, well-reviewed `unsafe` (SIMD, buffers) | httparse, serde, serde_core, serde_json, zmij: rustc-version probes or a generated `private.rs` in `OUT_DIR` | none |

There are no proc macros in this graph (`serde`'s `derive` is not enabled by ureq). **What the build needs:** ring's build needs a working C compiler for the target. A native `ubuntu-24.04-arm` runner or a `rust:` Docker image has one. No cmake, perl or nasm is needed.

## 4. Licences

All 30 match the `deny.toml` allowlist:
- `MIT OR Apache-2.0` (most crates)
- `Apache-2.0 OR ISC OR MIT` (rustls)
- `Apache-2.0 AND ISC` (ring, both allowed; ring also ships `LICENSE-BoringSSL` and `LICENSE-other-bits`)
- `ISC` (rustls-webpki, untrusted)
- `BSD-3-Clause` (subtle)
- `MIT` (bytes, zmij)
- `Unlicense OR MIT` (memchr, satisfied by MIT)
- `CDLA-Permissive-2.0` (webpki-roots)

ring 0.17.14's declared expression is accurate, so no cargo-deny `clarify` is needed. For Task 13, include ring's three licence files and webpki-roots' `LICENSE` in the image's third-party texts.

## 5. Advisories (RustSec advisory-db `640bf5a`, 2026-10-22)

`cargo audit` isn't installed here, so I matched the resolved versions against `crates/*` by hand.

| Advisory | Patched | Resolved |
|---|---|---|
| rustls RUSTSEC-2026-0285, -0346, -0347, -0348, -0349 | ≥ 0.23.45 / 0.23.46 | 0.23.46 |
| rustls-webpki RUSTSEC-2026-0049, -0098, -0099, -0104, -0382 | ≥ 0.103.16 for the newest | 0.103.16 |
| webpki-roots RUSTSEC-2026-0381 | ≥ 1.0.10 | 1.0.10 |
| ring RUSTSEC-2025-0009 | ≥ 0.17.12 | 0.17.14 |
| bytes RUSTSEC-2026-0007 | ≥ 1.11.1 | 1.12.1 |

The older advisories on base64, http, once_cell, shlex and untrusted only affect versions years older than the resolved ones. ring's -0007 was withdrawn, and -0010 only covers versions below 0.17. ureq 3.5.0's own minimums (`rustls >= 0.23.46`, `webpki-roots >= 1.0.10`) keep the patched floor. **No applicable advisories.**

## 6. MSRV

ureq and ureq-proto declare 1.85 (edition 2024), and that is the highest `rust-version` in the graph. A probe crate with `rust-version = "1.93"` built with `cargo +1.93` and made live HTTPS requests successfully.

## 7. API notes for our use (verified against the source and a live probe)

- **Agent:** `Agent::config_builder()` … `.build().into()`. The settings to choose:
  - `http_status_as_error(false)` (U4)
  - `timeout_global(Some(..))` (U1)
  - `https_only(true)` on the external Agent (U6)
  - `proxy(None)` (U5)
  - optionally `user_agent("liftosaur-sync/<ver>")`
- **Requests:** `.query(k, v)` percent-encodes the value, while `query_raw` doesn't. `.header(k, v)`, then `send_form(iter)` (`application/x-www-form-urlencoded`, for the Strava token exchange), `send_json(value)` (needs the `json` feature), `send(bytes)` for your own multipart body with a `Content-Type: multipart/form-data; boundary=…` header you set, and `call()` / `send_empty()` for bodiless requests.
- **Responses:** `response.status()`, `response.body_mut().read_json::<T>()` (10 MiB), `read_to_vec()` (10 MiB), or `body_mut().with_config().limit(n).read_to_vec()` for a custom cap (exclusive, U2). Body over the limit gives `Error::BodyExceedsLimit(n)`.
- **Connection pool:** a response body has to be read to the end before its connection goes back to the pool. A body dropped early just closes the connection, which is harmless.
- **Plan gotchas:**
  - `http_status_as_error(false)`: confirmed.
  - "no multipart builder": **outdated**, see U7. The advice to write your own still holds.
  - "`read_to_vec()` stops at 10 MB, use `body_mut().with_config().limit(..)`": confirmed. Be precise, though: it is 10 MiB, it is exclusive, and `with_config()` on its own is unlimited.

## Conditions of use

1. Features: `ureq = { version = "3.4.3"` or later `, default-features = false, features = ["rustls", "json"] }`. Don't enable `gzip`, `multipart`, `brotli`, `cookies`, `native-tls`, `platform-verifier`, `socks-proxy`, or any rustls `aws-lc-rs` feature without re-auditing.
2. Every Agent sets `http_status_as_error(false)` and a `timeout_global`.
3. The external-API Agent sets `https_only(true)`. Supervisor calls use a separate Agent.
4. Every response body read is bounded: the `read_*` defaults, or `with_config().limit(..)`. Never use an unlimited `as_reader()`/`into_reader()`/`with_config()`.
5. The Intervals.icu FIT download is bounded twice: compressed bytes by ureq, decompressed bytes by our gunzip (U3).
6. Decide on `proxy(None)` (recommended), and don't run `ureq=trace` logging in production.
7. Keep CI `cargo audit` and Dependabot (cargo) on, and rebuild the image for TLS-stack advisories (U12).

## Re-audit triggers

- A ureq minor bump. 3.x minors have been "technically breaking" on timeouts, EINTR and `RequestExt`.
- A change in the TLS provider or the feature set.
- A new ring major or minor release.
- Any RustSec advisory on a crate in §2.
