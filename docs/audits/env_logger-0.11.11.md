# Light audit: env_logger 0.11.11

- **Date:** 2026-10-22
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io packages `env_logger-0.11.11` and `env_filter-2.0.0`, plus `jiff-0.2.38` and `jiff-core-0.1.1` for the recommended feature set. All were unpacked by `cargo fetch` into `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/` in a throwaway project under `/tmp`. All checksums match the crates.io API:

  | Crate | Cargo.lock checksum | VCS sha1 |
  |---|---|---|
  | env_logger 0.11.11 | `de671bd27a75a797dc9ae289ba1e77276e75e2026408aab65185384e2d5cd3f6` | `b4d3f2b8dd3f1c3362f07da8f6f4a30c701358cf` |
  | env_filter 2.0.0 | `900d271a03799a1ee8d1ca9b19893b48ca674a9284fefcfb85f05e74ed314217` | `cc2b2efcd7454be82ca49f8ac165b3fbc3095ae3` |
  | jiff 0.2.38 | `b2b005715dcbeb0089a3c0dab99f2ff1cc3b2525323552703d648585d342a383` | `4ce3e20ecd3930b6f18cee5b9e4f0270287ea2d9` |
  | jiff-core 0.1.1 | `5e52fe76043ccecc9005d2305ebaadf7d7fc0cc89ca6baa10a94d6bc68c7128c` | `7e8133168382ab8d65a795126c2c9624f61e9e89` |

- **Use:** the `log` backend, writing to stderr, which becomes the HA app log.
- **Verdict: ACCEPT WITH CONDITIONS.** The conditions are to use a minimal feature set and follow the log-hygiene rules.

## Summary

env_logger itself is small and has **no `unsafe`, no build script and no proc macros**. It is maintained by epage (rust-cli). Its default features pull in `regex` (filter regexes), `anstream` (colour) and `jiff` (timestamps): **14 extra crates** on `aarch64-unknown-linux-gnu`. None of them is needed except timestamps. With `default-features = false, features = ["humantime"]` the tree is env_logger + env_filter + jiff + jiff-core (+ log). All four are first-party ecosystem crates with no build script and no proc macro on our target.

env_logger writes log arguments **verbatim**. It does no redaction and doesn't escape control characters. Keeping secrets out of logs is our job (C3–C6).

## Findings

| ID | Severity | Title | Impact for our usage | Recommendation |
|---|---|---|---|---|
| E1 | Low | Default features pull in 14 unneeded crates | `regex`, `regex-automata`, `regex-syntax`, `aho-corasick`, `memchr`, `anstream`, `anstyle`, `anstyle-parse`, `anstyle-query`, `colorchoice`, `is_terminal_polyfill`, `utf8parse`, plus jiff/jiff-core, which we keep. All are reputable, but it's attack surface and build time for nothing. | C1: `default-features = false, features = ["humantime"]`. |
| E2 | Medium (usage) | No redaction: anything formatted is logged | `log::debug!("{config:?}")` with `#[derive(Debug)]` on the config, token or request structs prints every secret. HA logs get pasted into GitHub issues and forums, and the repo is public. | C3, C4. |
| E3 | Low | `RUST_LOG=debug`/`trace` turns on dependency logging | `parse_default_env`/`from_env` let an environment variable enable debug and trace output for **all** crates. HTTP and TLS stacks log request details at those levels (check this in the ureq audit). | C5: build the filter from our own `log_level` option and cap dependency crates. |
| E4 | Low | Log-line forging and ANSI passthrough | Measured: a `\n` in an argument becomes a new line, which the default format indents by 4 spaces, so a forged `[... ERROR ...]` line is visibly offset. An `ESC` (`\x1b[31m`) is written raw. Activity names and API error bodies are remote-controlled strings. | C6: log external strings with `{:?}`, which escapes `\n` and `\u{1b}`, or truncate them. |
| E5 | Info | Write errors are ignored | `logger.rs:667` does `let _ = ...print(...)`, so a broken stderr doesn't panic the app. | None. |
| E6 | Info | RustSec | env_logger, env_filter, jiff and jiff-core: none. Default-only crates: `regex` RUSTSEC-2022-0013 (patched `>=1.5.5`, we'd get 1.13.1) and `anstream` RUSTSEC-2024-0404 (informational unsound, fixed long before 1.0.0). Neither applies, and C1 removes both crates anyway. | None. |

## 1. Build-time code and `unsafe`

| Crate | build.rs | proc macro | `unsafe` (grep of `src`) | Notes |
|---|---|---|---|---|
| env_logger 0.11.11 | no (`build = false`) | no | **0** | ~2,950 lines |
| env_filter 2.0.0 | no | no | **0** | ~1,300 lines |
| jiff 0.2.38 | no | no on our target | 79 tokens in total. On Linux they're in `fmt/buffer.rs` and `fmt/mod.rs` (a `MaybeUninit` formatting buffer) and `tz/timezone.rs` (a tagged-pointer `TimeZone` repr). `tz/system/{android,windows,wasm}` aren't compiled for us. | BurntSushi. Fuzzed and Miri-tested upstream. The `jiff-static` proc macro sits behind the dead target `cfg(any())` and the `static-tz` feature, so it is never built for us. |
| jiff-core 0.1.1 | no | no | 2 (`Date::new_unchecked` is an `unsafe fn`, plus one block in `util/mod.rs`) | BurntSushi |

Ambient behaviour: env_logger reads `RUST_LOG` / `RUST_LOG_STYLE` only via `env_logger::init()`, `Builder::from_env`/`from_default_env` or `parse_default_env`. `Builder::new()` (and `Builder::init()`) read nothing. It writes to stderr by default and does no file or network I/O. With `humantime` it calls `jiff::Timestamp::now()` and formats the time in **UTC** (`2026-10-22T23:55:32Z`). That needs no tz database and reads no `/usr/share/zoneinfo`, because jiff's `tz-system`/`tzdb` features are off.

## 2. Feature sets and transitive count

Measured with `cargo tree -e normal,build --target aarch64-unknown-linux-gnu`, together with anyhow and log (musl gives the same counts):

| env_logger features | Crates (excl. our own) | Added vs minimal |
|---|---|---|
| default (`auto-color`, `humantime`, `regex`) | **18** | regex ×3, aho-corasick, memchr, anstream ×4, anstyle, colorchoice, is_terminal_polyfill, utf8parse, jiff, jiff-core |
| `default-features = false` | **4** (anyhow, log, env_logger, env_filter) | — |
| **`default-features = false, features = ["humantime"]` (recommended)** | **6** | jiff, jiff-core |
| `["humantime", "color"]` | 12 | + anstream, anstyle, anstyle-parse, colorchoice, is_terminal_polyfill, utf8parse |

What you lose without the defaults:
- **`regex`:** the `RUST_LOG=target/pattern` filter becomes a plain substring match. That's fine for us.
- **`auto-color`/`color`:** no ANSI colours. The HA log viewer doesn't need them, and with stderr not being a TTY, auto-colour would switch them off anyway.

Timestamps (`humantime`): keep them unless the user decides HA's log viewer already timestamps every line well enough. The zero-crate alternative is `default-features = false` plus a custom `Builder::format` that uses `chrono`, which is already a dependency. That trades 2 well-maintained crates for code we would have to write.

Licences: env_logger and env_filter are `MIT OR Apache-2.0`. jiff and jiff-core are `Unlicense OR MIT`: Unlicense isn't on the allowlist, but `cargo-deny` accepts an `OR` expression when any alternative is allowed, so MIT is enough. The default-only crates are `MIT OR Apache-2.0`, or `Unlicense OR MIT` for memchr and aho-corasick. **All of them pass the `deny.toml` allowlist.**

MSRV: env_logger and env_filter declare 1.71, and jiff and jiff-core declare 1.70. Every crate in the default tree is at or below 1.71, and all are below our 1.93. The recommended set was built and run on the local stable toolchain (1.99.1). The local `1.93` toolchain has no `rustc` installed, so an actual 1.93 build wasn't run. The declared versions are well clear of it.

## 3. Maintenance

- **env_logger:**
  - Owners: `huonw`, `sfackler`, `sebasmagri`, the `github:rust-lang-nursery:log-owners` team, and the `github:env-logger-rs:publishers` team. `epage` publishes the releases.
  - Created 2015-01-27, with 59 versions. Recent releases: 0.11.8 (2025-04-01), 0.11.9 (2026-02-11), 0.11.10 (2026-03-23), 0.11.11 (2026-06-25).
  - About 598 million downloads.
- **env_filter:** owners are `epage` and `github:rust-cli:maintainers`. Created 2024-01-19 (it was split out of env_logger). 2.0.0 was released 2026-06-25, the same day as env_logger 0.11.11. About 224 million downloads.
- **jiff:** single owner `BurntSushi` (author of regex and ripgrep). Created 2024-02-17, with 70 versions. 0.2.38 was released 2026-10-06. 0.2.36 was yanked and replaced the same day by 0.2.37. About 239 million downloads.

## Conditions of use

- **C1** Declare `env_logger = { version = "0.11.11", default-features = false, features = ["humantime"] }`. Don't enable `kv`/`unstable-kv`. Re-check this audit if a 0.12 brings new defaults.
- **C2** Initialise the logger once at startup. Write to stderr (the default) or stdout. Both go to the HA app log.
- **C3** Never `#[derive(Debug)]` on a type that holds a secret: options/config, `StravaTokens`, `State` (it contains `StravaTokens`), the HTTP clients that hold credentials, and `Web { webhook_secret }`. Either write a manual `Debug` that prints `"<redacted>"`, or wrap secrets in a newtype whose `Debug` and `Display` both redact. Never log a whole request, `Authorization` header, form body, options file or `state.json`.
- **C4** Never log the OAuth `code` or the pasted Strava redirect URL. Log that a code was received, not the code. Don't log raw token-endpoint responses. Error bodies from other endpoints may be logged truncated (see the anyhow audit, C4).
- **C5** Build the filter in code from a `log_level` option, for example `Builder::new().filter_level(Info).filter_module("liftosaur_sync", <option>)`. Cap the HTTP/TLS crates (`ureq`, `rustls`, `tiny_http`) at `warn`, so "debug" never turns on dependency request logging. If `RUST_LOG` is honoured at all, document it as a developer-only setting.
- **C6** Log remote strings (activity names, error bodies, workout names) with `{:?}` or truncate them, so newlines and ANSI escapes are escaped.
- **C7** Add a test that formats the options, token and state types with `{:?}` and asserts that the secret values don't appear. This guards C3 against a future `derive`.
