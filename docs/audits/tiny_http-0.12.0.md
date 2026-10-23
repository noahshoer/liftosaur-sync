# Security audit (light): tiny_http 0.12.0, and the HTTP server choice

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `tiny_http-0.12.0`, as unpacked by `cargo fetch` at
  `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tiny_http-0.12.0`.
  Its Cargo.lock checksum is `389915df6413a2e74fb181895f933386023c71110878cd0825588928e64cdc82`. It matches the `.crate` file's SHA-256 and the crates.io API.
  The `.cargo_vcs_info.json` sha1 is `212b1c45852fef2093dc1374875a9393c55eb4b9` ("Prepare for 0.12.0 release" on `tiny-http/tiny-http`), `path_in_vcs` is empty. I did not diff the package against the Git tree (light audit).
- **Scope:** light audit (metadata, deps, `unsafe`, build scripts, advisories, a targeted read of request parsing and body framing, and a few local PoCs against a throwaway harness on macOS). Alternatives are covered more briefly.
- **Verdict: REJECT as published.** It would be acceptable only with a small local patch (Option 1b below). **Recommended instead: a hand-rolled server on `std::net::TcpListener` + `httparse` (Option 2a).**

## Summary

tiny_http has a very small footprint: 4 crates besides itself (`ascii`, `chunked_transfer`, `httpdate`, `log`), with no build scripts, no proc macros and no `unsafe` in tiny_http itself. The default features are empty, so no TLS. It builds on 1.93. `cargo audit` would pass today.

The problems are in behaviour, and the project is effectively unmaintained. The last release was 2022-10-06 and the last commit 2023-05-16. There are 76 open issues, and a Content-Length fix posted in an issue thread in 2024 was never merged. Two CVEs were filed in 2026-07 and have no fix and no maintainer reply. None of the following is protected by our 403/401 checks, because they all happen in tiny_http before or after our handler runs:

1. **One small request kills the whole process.** The request is about 80 bytes with `Content-Length: 1000000000000000`. If the handler doesn't read the body (our 403/401 paths), dropping the request runs `vec![0; remaining]` and the allocation fails. That is an `abort`, which `catch_unwind` can't catch. It takes the poll thread down with it. Reproduced. With `Content-Length: 18446744073709551615` you get a "capacity overflow" panic in the handler thread instead.
2. **No limits and no timeouts.** Header line length, header count and idle time are all unbounded. Each connection gets its own thread, with no cap. Streaming one 512 MiB header line pushed the server to about 516 MiB RSS (reproduced). That is enough to OOM-kill a Pi 4.
3. **The accept loop gives up for good on an accept error.** Running out of file descriptors (`EMFILE`) stops it. In a run with `ulimit -n 64`, 31 idle connections were enough. After that `recv()` returns `Err` forever and the web UI is dead until restart. Reproduced.
4. **Lax framing.** Any `Transfer-Encoding` value means chunked (CVE-2026-66752). Lone LF/CR are passed through in header values (CVE-2026-66753). With duplicate `Content-Length` headers the first one wins (reproduced as a desync: the body was served as a second request).

Our exposure is low. The port is reachable only on HA's internal `hassio` Docker network: the Supervisor ingress proxy, HA core and other apps. It is not internet-exposed. So the realistic attacker is a compromised or buggy co-located app, or an authenticated HA user. Still, finding 1 is a crash of the whole sync process that a single packet can trigger, and the app has no way to defend against it. For a component this small there are better choices with zero or few new crates.

## Findings

| ID | Severity | Title | Location | Impact for our usage | Recommendation |
|---|---|---|---|---|---|
| T1 | **High** | Unread body with a huge `Content-Length` → allocation of that size on drop → process `abort` (or a capacity-overflow panic) | `src/util/equal_reader.rs:70` (`let mut buf = vec![0; remaining_to_read];` in `Drop`), reached from `src/request.rs:215` | An 80-byte request to any route aborts the whole binary, poll thread included, right after our 403/401 is sent. The abort can't be caught. HA restarts the app, and the attack can be repeated. Reproduced (`memory allocation of 1000000000000000 bytes failed`). Still present on `master`. Upstream issue #252 (2023) had a fix posted in its comments and never merged. App code can't avoid the drop. The only workaround, `mem::forget(request)`, leaks a socket and a thread per request, which turns this into T3. | Don't use 0.12.0 unpatched (Option 1b patches this line). |
| T2 | Medium | No limit on header line length or header count | `src/client.rs:80-102` (`read_next_line` pushes into an unbounded `Vec`), `:115-127` | Memory grows at line rate until the OOM killer acts. Measured: 512 MiB sent → about 516 MiB RSS. Open since 2015 (#56, PR #232 unmerged). The chunk-size line in `chunked_transfer` (`decoder.rs:64-84`) is unbounded too. | Patch (1b) or replace. |
| T3 | Medium | No read/write timeouts. One unbounded thread per connection. The accept loop exits on any `accept()` error | `src/lib.rs:288-345` (`Err(e) => { …; break; }`), `src/util/task_pool.rs:69-78` | Idle connections hold threads and fds forever (#143, open since 2018). When fds or threads run out the accept thread exits (#195, #220, #221), and every later `recv()` is `Err`. Reproduced: 31 idle sockets with `ulimit -n 64` → `recv error: Too many open files`. PRs #284/#285 (2025, unmerged) handle related `accept`/`peer_addr` failures. | If kept, treat `recv()`/`incoming_requests()` ending as fatal and exit so HA restarts the app. |
| T4 | Low (for us) | CVE-2026-66752: any `Transfer-Encoding` value means chunked, and `Content-Length` is dropped. No fix | `src/request.rs:143-153, 218-221` | Smuggling needs a front end that frames differently. HA ingress is the only proxy in front, and we serve one user. A non-chunked body with `TE: identity` gets no response and ties up a connection thread (T3). Not yet in RustSec (db at `640bf5a`, 2026-10-22). **When it lands, `cargo audit` in CI fails with no patched version to move to.** | If kept, add an ignore entry with justification to `deny.toml`/`audit.toml` when the advisory appears. |
| T5 | Low (for us) | CVE-2026-66753: lone CR/LF kept in request header values, and response header values not validated. No fix | `src/client.rs:92-100`, `src/common.rs:166-191`, `src/response.rs:99-104` | We never reflect request headers into response headers, and we compare the secret header exactly. Response headers are constants. | Never build a response header from request data. |
| T6 | Low | Duplicate `Content-Length`: the first one wins, no rejection | `src/request.rs:155-158` | Reproduced desync: `CL: 0` then `CL: 30` → the body was served as a second request, `GET /smuggled`. Same front-end caveat as T4. | Patch (1b) or replace. |
| T7 | Info | Panic on `peer_addr()` failure in the connection thread | `src/client.rs:139` (`*self.remote_addr.as_ref().unwrap()`) | It kills only that connection's pool thread, not ours. PR #285 unmerged. | None. |
| T8 | Info | Maintenance | crates.io, GitHub | Owners `tomaka`, `frewsxcv`, `bradfier`. 0.12.0 was released 2022-10-06. `master` has 5 commits after the tag (last 2023-05-16: native-tls, optional `log`, MSRV 1.57), and the last push was 2024-08. 76 open issues, 18 open PRs, two 2026 CVE issues with no reply. Forks on crates.io (`tiny_http_fork`, `tiny_http_dh`) have only a few hundred downloads and aren't a realistic way out. | Treat it as unmaintained. |
| T9 | Info | Supply-chain surface | see §1 | Small and clean: no build scripts, no proc macros, no `unsafe` in tiny_http/chunked_transfer/httpdate. `ascii` has 97 `unsafe` (str/byte casts, owners include SimonSapin). `log` is already in the app. RUSTSEC-2020-0031 (TE smuggling) and RUSTSEC-2023-0015 (`ascii`) are both patched in the resolved versions. License MIT OR Apache-2.0 (all deps compatible with Apache-2.0). | None. |

## 1. tiny_http: build-time code, deps, `unsafe`, features, MSRV

```
$ cargo tree -e normal,build --target aarch64-unknown-linux-gnu      # tiny_http = "=0.12.0"
tiny_http v0.12.0
├── ascii v1.1.0            (owners SimonSapin, tomprogrammer, tbu-, tormol; 2022-09)
├── chunked_transfer v1.5.0 (owner frewsxcv; 2023-11)
├── httpdate v1.0.3
└── log v0.4.34             (already a direct dep of the app)
```

| Crate | LOC (src) | `unsafe` (grep -w) | build.rs | rust-version |
|---|---|---|---|---|
| tiny_http 0.12.0 | 3,698 | 0 | no | unset (edition 2018; master targets 1.57) |
| ascii 1.1.0 | 4,187 | 97 | no | unset |
| chunked_transfer 1.5.0 | 512 | 0 | no | unset |
| httpdate 1.0.3 | 580 | 0 | no | 1.56 |
| log 0.4.34 | 5,754 | 8 | `build = false` | 1.71 |

- **Features:** `default = []`. `ssl`/`ssl-openssl` (openssl + zeroize) and `ssl-rustls` are off by default. **Keep them off**: ingress is plain HTTP on the internal network. On 0.12.0 `log` can't be turned off, which doesn't matter for us.
- **MSRV:** `cargo +1.93 check` passes.
- **Net new crates over the app's planned dependency set** (rustyfit, embedded-io-adapters, ureq 3, serde, serde_json, toml 1, chrono, flate2, base64 0.23, anyhow, log, env_logger; 69 crates on aarch64-linux): **4** (`tiny_http`, `ascii`, `chunked_transfer`, `httpdate`).

## 2. tiny_http: PoCs

Harness: `/tmp/audit-http-server/p-tiny`. A `Server::http` loop answers every request with `Response::empty(403)` and doesn't read the body, which is what our non-ingress and bad-secret paths would do. If `recv()` errors it exits. The client was a Python socket script. Run on macOS. On the Pi (Linux, `vm.overcommit_memory=0`), a single allocation larger than RAM+swap fails the same way.

| Case | Request | Result |
|---|---|---|
| T1 huge CL | `POST /trigger`, `Content-Length: 1000000000000000`, no body | 403 sent, then `memory allocation of 1000000000000000 bytes failed`, process aborted, port closed |
| T1 max CL | `Content-Length: 18446744073709551615` | 403 sent, then `panicked … capacity overflow` in the handler thread |
| T2 long header | `GET / HTTP/1.1\r\nX: ` + 512 MiB of `a` | RSS 263 MB after 1 s, ~529 MB after 2 s, flat afterwards |
| T3 fd exhaustion | 80 idle connects under `ulimit -n 64` | 31 accepted, then `recv error: Too many open files (os error 24)`. Server permanently dead |
| T4 TE identity | `Transfer-Encoding: identity` + `Content-Length: 5` | `body_length=None` (CL ignored) |
| T6 dup CL | `CL: 0`, `CL: 30`, body = `GET /smuggled …` | Two requests logged: `POST /` then `GET /smuggled` on the same connection |

## 3. Alternatives surveyed

Each was resolved for `aarch64-unknown-linux-gnu`. "New" means crates added on top of the app's planned 69. `unsafe` is `grep -w unsafe` over `src/`.

| Candidate | Version / last release | Maintainers, activity | New crates | `unsafe` / build.rs / proc macros | Limits and timeouts | Verdict |
|---|---|---|---|---|---|---|
| **rouille** | 3.6.2 / 2023-04-24 | `tomaka`; last commit 2025-06; 71 open issues | 84 total (default-features off) | pulls `multipart` 0.18, `buf_redux`, `safemem`, `twoway`, all flagged **unmaintained** in RustSec; serde_derive + ICU4X derive proc macros (via `url`/`idna`) | it *is* tiny_http 0.12 underneath, so T1–T6 carry over | **Reject** |
| **astra** | 0.4.0 / 2024-11-07 | `ibraheemdev` (author of `matchit`, used by axum) alone; last commit 2025-01-12; 225 stars | 12 (hyper 1, hyper-util, mio, tokio with `sync` only, …) | astra: 3, sound (zeroing a read buffer, standard `Pin::new_unchecked` in a `block_on`); hyper 61, mio 153; no build.rs | hyper's parser (100 headers, ~400 KB buffer, strict TE/CL); bounded worker pool. **No header-read or idle timeout** (no hyper `Timer` wired). | Workable technically, but stale with a single maintainer. It swaps one quiet project for another. Not recommended. |
| **touche** | 0.0.15 / 2025-02-08 | `reu` alone; 56 stars; idle since 2025-02 | 19 (http, headers, sha1, thiserror (proc macro), threadpool, …) | 0 in touche | httparse-based; `max_threads` 512, `read_timeout` **off by default** | Reject (0.0.x, tiny user base) |
| **may_minihttp** | 0.1.11 / 2024-09-22 | `Xudong-Huang`; active repo | 35 total | built on the `may` stackful-coroutine runtime: `may` 207 `unsafe` + build.rs, `generator` 134 `unsafe` (context-switch asm) + build.rs | benchmark-oriented | **Reject**: a whole coroutine runtime with a large `unsafe` surface, just to serve three routes |
| **std `TcpListener` + `httparse` (hand-rolled)** | httparse 1.10.1 / 2025-03-03 (repo active, last commit 2026-06) | `seanmonstar` (hyper); it's hyper's own parser, heavily fuzzed; ~815M downloads | **0**: httparse is already in the tree via `ureq-proto` | httparse: 75 `unsafe` (SIMD); build.rs only runs `rustc --version` to enable SIMD cfgs; no proc macros; no RustSec advisories ever | whatever you write: fixed header buffer, socket read/write timeouts, body cap | **Recommended** (see Decision) |
| **axum + tokio** | axum 0.8.9 / 2026-04-14, tokio 1.54.1 / 2026-10-21, hyper 1.12.0 / 2026-10-06 | tokio-rs org; very active | **28** (axum, axum-core, hyper, hyper-util, tokio, tokio-macros, mio, socket2, tower, futures-util, matchit, serde_urlencoded, …) with `default-features = false, features = ["http1","tokio","form"]` and tokio `rt`,`net` | tokio 1,066, hyper 61, hyper-util 17, mio 153; axum/axum-core 0; 1 new proc macro (tokio-macros; syn/quote already present via serde_derive); MSRV axum 1.80, tokio 1.85 (fine for 1.93) | hyper's parser; axum `DefaultBodyLimit` 2 MB; the accept loop survives `EMFILE` (logs, sleeps 1 s). **`axum::serve` sets no hyper `Timer`, so no header-read timeout** (`hyper` `header_read_timeout` defaults to 30 s but only takes effect with a timer) | Solid and maintained. The cost is new dependencies and async, not risk. |

RustSec cross-check (db commit `640bf5a`, 2026-10-22, against each lockfile): no advisory affects the resolved versions of any candidate. The only hits are rouille's unmaintained notices. tokio 1.54.0 (yanked) was hit by RUSTSEC-2026-0384, fixed in 1.54.1. If you choose tokio, keep it at `>= 1.54.1`.

### What going async would cost (Option 3)

- **Deps:** +28 crates, including 1 new proc macro. Total tree 69 → 97. Also a bigger re-audit diff on each Dependabot bump. tokio and hyper ship releases often (tokio had 5 in October 2026 alone).
- **Integration:** low friction. You don't need `spawn_blocking`. The poll loop stays a plain `std::thread`. The web thread builds a `tokio::runtime::Builder::new_current_thread().enable_io()` runtime and `block_on(axum::serve(...))`. Handlers call the same pure `handle(&Web, …) -> Reply`. They use `std::sync::mpsc::Sender::send` (doesn't block) and take short `std::sync::Mutex` locks on the snapshot, which is fine in async code as long as no lock is held across an `.await`. `ConnectInfo<SocketAddr>` gives the remote IP (requires `into_make_service_with_connect_info`).
- **Learning:** `async fn`, extractors, `Router`, `IntoResponse`, `Send` bounds in compiler errors. Moderate for three routes. Useful if you'll write async Rust later, but it's the only async code in an otherwise blocking app.
- **Still on you:** slowloris has no protection by default. tower's `TimeoutLayer` covers handlers, not header reads. Fixing that means running hyper-util by hand with a `TokioTimer`.

## 4. Decision for the user

Context: three routes (`GET /`, `POST /strava/code`, `POST /trigger`), tiny bodies (a pasted URL or code; `/trigger` has none), single-digit requests per hour, internal network only, and you're writing the code yourself.

**Option 1a: accept tiny_http 0.12.0, pinned, as is.** *Not acceptable.* T1 is a single-request crash of the whole process that your handler can't prevent. A pin protects against future releases, but there won't be any.

**Option 1b: tiny_http pinned plus a local patch** (`[patch.crates-io]` pointing at a vendored copy in the repo). Changes: (1) the `EqualReader` drop reads into a fixed-size buffer; (2) cap header line length and header count in `read_next_line`; (3) reject duplicate CL and any TE that isn't exactly `chunked`; (4) set socket read/write timeouts on accept; (5) keep accepting after `accept()` errors. That keeps the familiar API and 4 new crates. The cost is that you own a fork of about 3.7k lines, with a NOTICE/licence obligation for the vendored MIT/Apache code. You'd also need ignore entries for the CVEs in `cargo audit`/`cargo deny` once they land. More work than 2a for a worse result.

**Option 2a (recommended): hand-rolled on `std::net::TcpListener` + `httparse`.** Zero new crates: `httparse` is already in the tree via ureq, maintained by hyper's author, and fuzzed. It's about 150–250 lines that you write and test yourself, which fits how this plan works. The behaviours to build and test (the what, not the how):
- Handle one request per connection, and always send `Connection: close`. No keep-alive and no pipelining means smuggling and desync have nothing to work with.
- Call `set_read_timeout`/`set_write_timeout` on each accepted stream (a few seconds). Handle connections one after another on the web thread, or with a small fixed number of threads. At this traffic, one after another is fine.
- Read headers into a fixed buffer (8–16 KiB) until `httparse` returns `Complete`. If the buffer is full first, send 431. Use a fixed headers array (e.g. 32); `TooManyHeaders` gets 431.
- Reject any `Transfer-Encoding` (501 or 400). Reject duplicate `Content-Length`. Require digits only: Rust's `usize::from_str` accepts a leading `+`, which hyper had to fix (RUSTSEC-2021-0078). Cap the body (e.g. 8 KiB, otherwise 413) and read exactly that many bytes, with the timeout still in force.
- Keep handling `accept()` errors (log them, sleep briefly, carry on). Don't return from the loop.
- Keep the plan's pure `handle(...) -> Reply` and unit-test it as written. Add a few socket-level tests: oversized header → 431, `TE` → rejected, huge CL → 413 without allocating, slow client → times out, 403 from a non-ingress IP.

For: no new supply chain, full control, synchronous like the rest of the app. Against: you own the HTTP framing. The bullet list above is the whole of it, and it's small because the server only has to handle what HA ingress and `rest_command` send.

**Option 2b: astra 0.4.0.** hyper's well-tested parser behind a blocking API, +12 crates. It's quiet too (last commit 2025-01, one maintainer) and has no timeouts. Only worth it if you want hyper's parser without async and don't want to write the framing yourself.

**Option 3: axum 0.8 + tokio 1.54.** The most robust and best-maintained stack. +28 crates, an async island in the web thread, and some async learning. No `spawn_blocking` is needed, because the poll loop stays a std thread. Choose it if you'd rather depend on tokio-rs than own ~200 lines, or if you expect the web side to grow (more routes, JSON API, SSE status updates).

**Recommendation: Option 2a.** If you'd rather not own the HTTP framing, go with Option 3 over 2b, because maintenance matters more here than the dependency count. Whichever you choose, update the Tech-stack line in the plan and Task 12's "APIs" note. Task 12's ingress check should use the accepted stream's `peer_addr()` (or `ConnectInfo` under axum).

## 5. How to re-check

```sh
W=/tmp/audit-http-server
# metadata / checksums
curl -s https://crates.io/api/v1/crates/tiny_http | jq '.versions[:3][]|{num,created_at,checksum}'
shasum -a 256 ~/.cargo/registry/cache/index.crates.io-*/tiny_http-0.12.0.crate
# advisories (RustSec + GitHub)
git -C $W/untrusted/advisory-db pull && ls $W/untrusted/advisory-db/crates/{tiny_http,httparse,hyper,tokio,axum}
gh api '/advisories?ecosystem=rust&affects=tiny_http'
gh issue list -R tiny-http/tiny-http --state open --search CVE
# PoCs (tiny_http)
$W/p-tiny/target/release/p-tiny 2>srv.log & python3 -I $W/scripts/poc.py cl_huge   # cases: cl_huge cl_max longhdr idle te dupcl
# dependency deltas over the app baseline
cd $W/p-base && cargo tree -e normal,build --target aarch64-unknown-linux-gnu --prefix none | sort -u
```
