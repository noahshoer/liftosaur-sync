# Security audit: rustyfit 0.10.3

- **Date:** 2026-10-14
- **Auditor:** Claude security sub-agent
- **Source audited:** the crates.io package `rustyfit-0.10.3`, as unpacked by `cargo fetch` at
  `~/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/rustyfit-0.10.3`.
  Its Cargo.lock checksum is `07509579836cbc3cca701fac09d305aeab7bcd17b7274762ed0246267d1f2c8a`. It matches the `.crate` file's SHA-256 and the crates.io API.
  The `.cargo_vcs_info.json` sha1 is `92915bfd09fc8314c4da0d8d488637093c1ddf7e` and `path_in_vcs` is `rustyfit`.
- **Verdict: ACCEPT WITH CONDITIONS**

## Summary

rustyfit 0.10.3 has a small supply-chain footprint. It has no build script, no proc macros, no FFI and no ambient I/O, and it is `#![no_std]`. Its only runtime dependency is `embedded-io` (owned by the rust-embedded HAL team, no `unsafe`, no build script). The published `src/` is byte-identical to the Git tag commit `92915bf`. It contains 9 `unsafe` blocks, all `Vec::from_raw_parts` reinterpretations between a `#[repr(transparent)]` newtype and its own primitive. All 9 are sound, and Miri confirms it. The encoder produced valid output on every input it accepted: 22 SDK and forum sample files and about 779,000 fuzz-mutated files decoded back to identical messages.

The decoder is **not** safe against hostile input:

1. It **panics** (remainder by zero) on a 27-byte file. This happens in every build profile, and the bug is still present on upstream `master`.
2. Zero-size field definitions give about **10 KB of heap per input byte**: a 100 KB file takes about 1 GB of heap, enough to OOM-kill the container on a Pi 4.
3. A **quadratic CPU** path in developer-field lookup takes about 4 s on a 16 KB file, and roughly hours on 1 MB if the measured n² scaling holds.

None of these needs a crafted CRC: the data CRC is only checked after the whole sequence has been decoded. All three are prevented by a streaming pre-scan plus `catch_unwind`, which I wrote and tested (Conditions of use). With those conditions in place the crate is acceptable for our use.

## Findings

| ID | Severity | Title | Location | Impact for our usage | Recommendation |
|---|---|---|---|---|---|
| F1 | **High** | Decoder panics: `size % base_type.size()` where an invalid base type has `size() == 0` | `src/decoder/mod.rs:410` (normal field of an unknown message/field), `:475` (developer field whose `field_description.fit_base_type_id` is invalid) | Any malformed FIT (27 bytes is enough) panics `Decoder::decode`. Without `catch_unwind` this kills the poll loop. Because the same activity is fetched again on every poll, it becomes a persistent outage. Happens in debug and plain release builds. Not fixed on `master` as of 2026-10-14. | Wrap decode, inspection and encode in `std::panic::catch_unwind`, with `panic = "unwind"` (the default). Run the pre-scan (C3) to reject such inputs. Report the bug upstream. |
| F2 | **High** | Unbounded memory amplification from zero-size field definitions | `decoder/mod.rs:384-443` (fields of size 0 are zero-padded and still emitted), `:173` (`messages.push(self.mesg.clone())`) | One definition with 255 fields of `size 0`, followed by 1-byte data messages, makes 255 `Field`s (40 B each) per input byte. Measured: 20 KB gives about 195 MiB and 100 KB gives about 978 MiB. An OOM kill cannot be caught, so the container restart-loops on the same file. | Pre-scan (C3): reject `size == 0` field and developer-field definitions, and cap total fields. Cap input size (C4). |
| F3 | **Medium** | Quadratic CPU in developer-field lookup | `decoder/mod.rs:456` (linear `find` over `field_descriptions`, which grows by one per `field_description` message with no limit), `:489` | N empty `field_description` messages, then M data messages with 255 unmatched developer fields, cost M×255×N comparisons. Measured: 4.8 KB → 0.24 s, 16.8 KB → 3.85 s. This stalls the poll loop and can't be caught. | Pre-scan (C3): cap `field_description` messages (for example at 256) and reject zero-size developer fields. Put a wall-clock limit on the whole sync job. |
| F4 | Low | Encoder rejects files the decoder accepts: developer fields without a prior `developer_data_id` | `encoder/validator.rs:200-208` versus `decoder/mod.rs:487` ("we allow missing developer_data_id") | `Encoder::encode` returns `Err(MissingDeveloperDataId)` and that activity cannot be rewritten. This is an error, not a panic. Verified with a crafted file. | Treat encode errors as "skip this activity and log it". Optionally drop orphan developer fields before encoding. |
| F5 | Low | Silent data loss: developer fields with no matching `field_description` are dropped on decode | `decoder/mod.rs:456-461` (`continue`) | Re-encoded files lose those developer fields without any error. Verified with a crafted file. Normal Garmin and Connect IQ files always include descriptions. | Accept, and document it. |
| F6 | Low | `HeaderOption::Compressed` mis-compresses timestamps after a jump of 256–287 s | `encoder/mod.rs:302` (`as u8` truncation). Fixed upstream after the release in `aced7f2` ("fix: encoder compress timestamp overflow", 2026-09-15). | Silent timestamp corruption, but **only** with the Compressed header option. We use the default `Encoder::new()` (`Normal(0)`), so we are not affected. Taken from the upstream fix and its test, not reproduced here. | Never use `HeaderOption::Compressed` with 0.10.3. |
| F7 | Info | 9 `unsafe` `Vec::from_raw_parts` sites | `profile/mesgdef/*.rs` (see §4) | Sound. Miri found no UB, including under strict provenance. Avoiding `Message::from(mesgdef::Set)` is not required for safety. | None. Re-check on every bump (§Re-audit). |
| F8 | Info | CRC gives no DoS protection | `decoder/mod.rs:176, 589-603` (data CRC checked after the whole sequence); `:234` (a header CRC of 0 skips the check) | F1–F3 trigger before any CRC check. With `checksum(true)` (the default), a bad data CRC still returns `Err(ChecksumMismatch)` at the end. | Keep `checksum` on. Don't count on it for safety. |
| F9 | Info | `decode()` returns one FIT sequence; chained files need a loop | `decoder/mod.rs:144-183` | If we decode only once, trailing chained sequences are silently not re-encoded. Garmin activity files are not chained in practice. The pre-scan must cover every sequence the app decodes. My first pre-scan stopped at the first CRC, and the fuzzer bypassed it with chained inputs (§7). | Decode exactly one sequence and reject files with trailing bytes, or loop and pre-scan all sequences. |
| F10 | Info | The default `Normal(0)` header option inflates output | `encoder/mod.rs:358-370`, `lru.rs` | Output is valid but can grow: `large.fit` went from 2.77 MB to 6.29 MB and `triathlon_summary_last.fit` from 274 KB to 620 KB. | Use `Encoder::builder().header_option(HeaderOption::Normal(15)).build()` if size matters. |
| F11 | Info | `Encoder::encode` mutates its input `FIT` | `validator.rs:153-181` | It drops `is_expanded` fields, reorders fields, overwrites `mesg.header`, and fills in `file_header` size, CRC, `data_size` and `profile_version`. With `expand_components(false)` no field is expanded. Raw `set` fields we build must use `is_expanded: false`, or they are silently dropped. | Build inserted fields with `is_expanded: false` and the profile base type. Use `Value::VecUint16` for `set.category` and `set.category_subtype` (fields 7 and 8), because `Set::from` ignores any other variant (`set.rs:166-173`). |
| F12 | Info | Small decode normalisations | `proto.rs:332-566` | Invalid UTF-8 becomes U+FFFD (`from_utf8_lossy`). Short fields are zero-padded and over-long ones truncated to their base size. These are stable across our round trip: decode→encode→decode is identical on all samples and fuzz inputs. | Accept. |
| F13 | Info | Maintenance and provenance | crates.io, GitHub | Single owner, `muktihari`, who wrote all non-bot commits. Young pre-1.0 crate (created 2025-04-10) with 18 releases and fast churn. `master` has a breaking refactor (`35f933f`, Value uses boxed slices). 15 stars, 7,652 downloads. No RustSec advisories, no GitHub security advisories, no security issues. | Pin `=0.10.3` and re-audit every bump. |
| F14 | Info | FIT SDK licence on the generated profile | `LICENSE-FIT-SDK`; Cargo `license` says only `BSD-3-Clause` | See §9. There are obligations a public repo should note. This is not legal advice. | Don't vendor rustyfit's source, keep the repo's licence non-copyleft, ship both licence texts, and keep output spec-compliant. |

## 1. Build-time code

```
$ grep -E '^build|proc-macro|build-dependencies' Cargo.toml
build = false
$ ls build.rs            -> absent
$ grep -rn "proc_macro|include_bytes|include_str|env!|option_env" src   -> none
```

The package `Cargo.toml` (normalised) has `build = false`, no `[build-dependencies]`, and is not a proc-macro crate. `autobins`, `autotests`, `autoexamples` and `autobenches` are all `false`. `include = ["Cargo.toml","Cargo.lock","src/**/*","LICENSE*"]` (347 files, 4.5 MB). The profile code was generated by `fitgen/main.go` in the repo, but the generator is not shipped and does not run at build time. **No build-time code execution.**

## 2. Dependency tree

```
$ cargo tree -e normal,build        # project = rustyfit =0.10.3 + embedded-io-adapters 0.7 (std)
fetch v0.0.0
├── embedded-io-adapters v0.7.0
│   └── embedded-io v0.7.1
└── rustyfit v0.10.3
    └── embedded-io v0.7.1
```

| Crate | Version | crates.io owners | build.rs | `unsafe` | Notes |
|---|---|---|---|---|---|
| rustyfit | 0.10.3 | muktihari | no | 9 blocks (§4) | |
| embedded-io | 0.7.1 | Dirbaio, github:rust-embedded:hal | no (`build = false`) | none (grep found no `unsafe` in `src`) | `no_std` unless `std` |
| embedded-io-adapters | 0.7.0 | Dirbaio, github:rust-embedded:hal | no | none (only the word in two comments, `futures_03.rs:86` and `tokio_1.rs:109`) | we enable `std` only; the tokio and futures features are off |

About 1,456 lines of Rust across the two embedded-io crates.

**What `serde` adds** (`cargo tree -e normal,build --features rustyfit/serde`; we won't enable it): serde 1.0.229 (build.rs), serde_core 1.0.229 (build.rs), serde_derive 1.0.229 (**proc macro**), proc-macro2 1.0.107 (build.rs), quote 1.0.47 (build.rs), syn 3.0.9, unicode-ident 1.0.27. All are owned by dtolnay (serde and serde_derive also by github:serde-rs:publish). The feature would bring in build-time code execution, and none of it is needed. **Keep `serde` off**, and keep `default-features` as is: the crate has no default features.

## 3. Ambient capability

```
src/lib.rs:618  #![cfg_attr(not(test), no_std)]
src/lib.rs:620  extern crate alloc;
$ grep -rn 'std::' src | grep -v 'core::\|alloc::'   -> only doc comments and #[cfg(test)] modules
   (decoder/mod.rs:925, encoder/mod.rs:717, semconv.rs:22 are inside `mod tests`)
$ grep -rn 'extern "C"\|#\[link\|extern crate std' src   -> none
```

Outside tests the crate is `no_std` and uses only `core`, `alloc` and `embedded_io::{Read, Write, Seek}`. All I/O goes through the reader and writer the caller passes in. There is no fs, net, process, env or FFI. **Confirmed.**

## 4. `unsafe`

```
$ grep -rn unsafe src
src/profile/mesgdef/aviation_attitude.rs:427 / :454, capabilities.rs:125, device_settings.rs:309,
dive_alarm.rs:267, dive_apnea_alarm.rs:267, monitoring_info.rs:205, segment_file.rs:182, set.rs:248
$ grep -rnE 'transmute|get_unchecked|from_utf8_unchecked|MaybeUninit|\.offset\(|\.add\(|ptr::|static mut|unsafe impl|as \*const|as \*mut|set_len' src
   -> only the 9 matching `into_raw_parts()` lines
```

Every site has the same shape, inside `impl From<mesgdef::X> for Message`:

```rust
let (ptr, len, capacity) = m.<field>.into_raw_parts();
unsafe { Vec::from_raw_parts(ptr.cast::<uN>(), len, capacity) }
```

| # | Site | Source element type | Type definition | Cast to | `repr(transparent)` over same primitive? | Same Vec's `into_raw_parts`? | Verdict |
|---|---|---|---|---|---|---|---|
| 1 | aviation_attitude.rs:427 | `Vec<AttitudeStage>` | `typedef/attitude_stage.rs:12-14` `struct AttitudeStage(pub u8)` | `u8` | yes | yes (`m.stage`, moved) | sound |
| 2 | aviation_attitude.rs:454 | `Vec<AttitudeValidity>` | `attitude_validity.rs:12-14` `(pub u16)` | `u16` | yes | yes (`m.validity`) | sound |
| 3 | capabilities.rs:125 | `Vec<SportBits0>` | `sport_bits_0.rs:12-14` `(pub u8)` | `u8` | yes | yes (`m.sports`) | sound |
| 4 | device_settings.rs:309 | `Vec<TimeMode>` | `time_mode.rs:12-14` `(pub u8)` | `u8` | yes | yes (`m.time_mode`) | sound |
| 5 | dive_alarm.rs:267 | `Vec<SubSport>` | `sub_sport.rs:12-14` `(pub u8)` | `u8` | yes | yes (`m.dive_types`) | sound |
| 6 | dive_apnea_alarm.rs:267 | `Vec<SubSport>` | as above | `u8` | yes | yes (`m.dive_types`) | sound |
| 7 | monitoring_info.rs:205 | `Vec<ActivityType>` | `activity_type.rs:12-14` `(pub u8)` | `u8` | yes | yes (`m.activity_type`) | sound |
| 8 | segment_file.rs:182 | `Vec<SegmentLeaderboardType>` | `segment_leaderboard_type.rs:12-14` `(pub u8)` | `u8` | yes | yes (`m.leader_type`) | sound |
| 9 | set.rs:248 | `Vec<ExerciseCategory>` | `exercise_category.rs:12-14` `(pub u16)` | `u16` | yes | yes (`m.category`) | sound |

**Allocator and layout invariants.** `#[repr(transparent)]` over a single primitive gives identical size and alignment, so `Layout::array::<T>(cap) == Layout::array::<uN>(cap)`. The pointer came from the global allocator (the `Vec` default `A = Global`) and is handed back with the same capacity. `len <= cap`, and the first `len` elements are initialised, valid `uN` values. The original `Vec` was consumed by `into_raw_parts`, so there is no double free. When the field is empty the call is skipped (`if !m.<field>.is_empty()`), but a dangling or zero-capacity pointer would also be fine. The newtypes derive only `Copy` and `Clone` and have no `Drop`. The reverse direction (`From<&Message>`) uses safe `iter().map().collect` (for example `set.rs:166-173`).

**Miri.** `tests/unsafe_sites.rs` in the scratch `miri` crate exercises all 9 sites, including a `Vec` with `capacity (7) != len (2)`, and checks the resulting values:

```
$ cargo +nightly miri test                                                     -> test result: ok. 1 passed
$ MIRIFLAGS="-Zmiri-strict-provenance -Zmiri-symbolic-alignment-check" cargo +nightly miri test -> ok. 1 passed
(miri 0.1.0 (822182db4c 2026-10-13))
```

The only fragility is maintenance: the generator would have to change a typedef's representation without updating the cast. Re-check on bumps.

## 5. Panic and DoS surface in the decoder

The decoder state is fixed-size: `buf: [u8; 256]`, 16 `MessageDefinition`s, and `Accumulator` with `[AccuValue; 7]`.

**Indexing and slicing on input-controlled sizes. All are in bounds:**

- `self.buf[..size]`: `size: u8` is at most 255, and the buffer is 256 bytes (`mod.rs:386, 448`). Definition chunks are capped at `min(255)` (`:282, :315`).
- `mesg_definitions[header & 0x0F]` and `[(header & 0x60) >> 5]` (`:257, :357`).
- `reslice_buffer` is only called when `size < base_size`, and `base_size` is at most 8 (`:632`).
- `from_parts` slices `buf[..2|4|8]` only after reslicing guarantees `len >= base_size` (`proto.rs:368-562`).
- `KNOWN_NUMS[field.num as usize >> 6]`: `u8 >> 6` is at most 3, and the array is `[u64; 4]` in every mesgdef.
- `Accumulator.values[self.len]` (`accumulator.rs`): the keys are `(mesg_num, field_num)` pairs taken from the static profile. I enumerated every `accumulate: true` in `lookup.rs` (20 occurrences) and found exactly 7 distinct keys: RECORD 5/18/19/28/29 and HR 9/10, which equals `TOTAL_ACCUMULATE = 7`. Cannot overflow.
- `Bits::pull` and `accumulate` shifts: component `bits` come from the static profile (1, 3, 4, 5, 8, 12, 14, 16), so `1 << bits` and `<< (64 - bits)` never reach 64.

**`unwrap`, `expect` and `unreachable`.** In non-test code these are only `try_into().unwrap()` on fixed-length slices (`mod.rs:244`, `proto.rs:368-562`), which cannot fail. There are no `unreachable!`, `panic!`, `todo!` or `expect` calls. `cargo clippy -W clippy::unwrap_used` reports 17 hits, all of this kind.

**Division (F1, the real panic).** `FitBaseType::size()` returns 0 when `base_type & 0x1F >= 17` (`fit_base_type.rs:40-46`). For a field with no profile entry the decoder evaluates `size > base_type.size() && size % base_type.size() == 0` (`mod.rs:410`), which divides by zero whenever `size >= 1`. The same happens for developer fields (`:475`), where the base type comes from a `field_description` message in the file. Rust always checks division by zero, so this panics in every profile. Verified in a plain `--release` build with `overflow-checks=false` and `debug-assertions=false`:

```
input (27 bytes): 0e 20 00 00 0b 00 00 00 2e 46 49 54 00 00 | 40 00 00 00 ff 01 00 01 11 | 00 ab | 4b 59
   header(14, crc 0) | definition: local 0, mesg 0xFF00, 1 field {num 0, size 1, base_type 0x11} | data | CRC
thread 'main' panicked at .../rustyfit-0.10.3/src/decoder/mod.rs:410:57:
attempt to calculate the remainder with a divisor of zero

poc_divzero_devfield.fit (47 bytes): field_description{dev_idx 0, field 0, fit_base_type_id 0x11},
   then a record definition with 1 developer field {num 0, size 1, idx 0} and one data message
thread 'main' panicked at .../decoder/mod.rs:475:49: attempt to calculate the remainder with a divisor of zero
```

On upstream `master` (`cc12f08`) the same expressions are still at `decoder/mod.rs:410` and `:475`.

**Integer overflow** (checked in a release build with debug assertions and overflow checks):

- `cur: u32 += size`: overflowing it would need more than 4 GiB of input.
- `Accumulator::v.value += masked delta`: the starting values are profile `UINT32`/`UINT8` fields, so they are at most 2³²; overflow would need more than 2³² messages.
- Compressed timestamps use `wrapping_add` and `wrapping_sub` (`:371`).
- Encoder `data_size: u32`: would need more than 4 GiB of output.

None of these is reachable with a size-capped input. Float to int conversions use `as`, which saturates.

**Allocation sized from the input.**

- There is no allocation taken directly from `data_size` or from a length field. `data_size` only bounds the loop, and every iteration consumes at least 1 byte.
- Per-definition `reserve_exact(255)` happens at most 16 times at a time.
- `Vec` sizes in `from_parts` are at most 255 bytes.
- The amplification is structural instead (F2). Each data message clones `mesg` (`:173`), and a field of size 0 still produces a `Field`, zero-padded via `reslice_buffer`. `size_of::<Message>() = 56`, `size_of::<Field>() = 40`, `size_of::<Value>() = 32`.

```
$ amplify 20000   -> input 20787 bytes -> 20000 messages, 5100000 fields, ~195 MiB heap (lower bound), 189 ms
$ amplify 100000  -> input 100787 bytes -> 100000 messages, 25500000 fields, ~978 MiB heap (lower bound), 1.10 s
```

**Recursion.** `expand_components` recurses only along the static profile's component graph, which is acyclic. Each component's `Bits` buffer is fixed-size (240 bits), and the loop over `fields.len()` is evaluated once, so it is not affected by fields that get pushed during expansion. Bounded.

**Loops.** Every loop in `decode` consumes input or iterates over a structure of at most 255 entries, except the linear `field_descriptions.iter().find` in `:456`. That vector grows by one per `field_description` message for the whole sequence, which gives F3:

```
$ quadratic 2000 2000  -> input 4794 bytes  in 235.66 ms (1.02e9 comparisons)
$ quadratic 8000 8000  -> input 16794 bytes in 3.85 s    (1.63e10 comparisons)
```

The 1 MB figure ("hours") is extrapolated from this n² scaling, not measured.

**Work per input byte.**

- Without hostile structure, each byte yields at most one field decode, a profile lookup (a const `match`), and one CRC update.
- Hostile structure removes that bound. Zero-size fields give up to 255 `Field`s per byte, and developer-field lookup costs up to 255 × N comparisons per byte, with N unbounded.
- With component expansion on, the extra work per field is bounded by the static profile.

**Chained FIT.** `decode()` resets state and returns `Ok(None)` only on clean EOF at a header boundary. A garbage tail gives `Err(NotFITFile)` or `Err(UnexpectedEof)`.

**CRCs.** A 14-byte header's CRC is checked, unless it is 0, before any data is read. A 12-byte header is CRC'd together with the data. The data CRC is compared only after the whole sequence has been parsed (F8).

## 6. Encoder validation

`MessageValidator::validate_message` (`encoder/validator.rs:130-242`):

- Protocol V1 rejects developer fields and base types above `BYTE`.
- `is_expanded` fields are **skipped and removed**: they are compacted out with `swap` and `truncate`, so they are never written.
- Each remaining field must pass `validate_field`. `Value::is_align(base_type)` must match the field's base type, so `Value::Invalid` and wrong-typed values give `Err(ValueTypeInvalid)`. Strings must be valid UTF-8. `value.size()` must be at most 255. At most 255 fields.
- Developer fields need a prior `developer_data_id` (tracked in a 256-bit set) and a prior `field_description`, and their value must match the described base type.
- It rejects messages with no fields at all (`EmptyMessageData`).
- It does **not** check that values are in range or "valid" in the FIT sense. A `0xFFFF` sentinel is written as is. Writes are into a 1,537-byte buffer whose bounds are guaranteed by those limits.

**Round trip** (`poc/src/bin/roundtrip.rs`): decode with `expand_components(false)`, then `Encoder::new().encode`, then decode again. Messages are compared with `{:?}` of `num`, `fields` and `developer_fields`; the header byte is excluded.

```
Activity.fit:          94096 B -> 3611 msgs; re-encoded 94096 B; msgs_equal=true bytes_stable=true
ActivityDevFields.fit: 94199 B -> 3613 msgs; re-encoded 94199 B; msgs_equal=true bytes_stable=true
large.fit:           2769422 B -> 87778 msgs; re-encoded 6288995 B; msgs_equal=true bytes_stable=true
triathlon_summary_last.fit: 273654 B -> 12136 msgs; re-encoded 619515 B; msgs_equal=true
... all 22 SDK/forum samples + big_activity.fit + 4 generated files: msgs_equal=true, count_equal=true,
    validator_changed=false, bytes_stable=true (encode(decode(encode(x))) == encode(x))
```

The fuzz oracle (§7, target a) checks the same property after our transformation (drop `set`/`exercise_title`, insert raw `set`s). It never failed across 778,785 successful encodes (runs 1–3). Encoder failures seen are all `Err` returns, never panics: F4, `Value::Invalid` from an unknown base type, and empty messages.

## 7. Fuzzing

**Deviation from the brief: libFuzzer did not run, so there are no edge or feature coverage numbers.**

- Docker Desktop would not start on this host. Its backend log reports `remove ...\Docker\run\sailor-ingest.sock: The file cannot be accessed by the system`. Fixing that means removing stale socket files in Docker's state directory, which needs the user's approval.
- The WSL Debian distro has no compiler and no package access.
- Native Windows `cargo +nightly fuzz build` (cargo-fuzz 0.13.2, nightly 2026-10-13) fails because MSVC `cl.exe` can't compile libFuzzer (`fatal error C1083: Cannot open compiler generated file: '': Invalid argument`).

The cargo-fuzz harness is written and ready to run under Linux or Docker, at `scratchpad/rustyfit-audit/fuzz/`. Its targets are `decode_reencode` (a) and `decode_mesgdef` (b), built with `profile.release` `debug-assertions = true` and `overflow-checks = true`.

Instead, I wrote a mutation fuzzer, `poc/src/bin/mutfuzz.rs`. It has the same two targets and oracle, and runs with `catch_unwind` and panic-location deduplication. It is built with release, debug assertions and overflow checks. It has no coverage feedback; it relies on structure-aware mutations:

- bit flips and "interesting" bytes (`0x11`, `0x1f`, `0x40`, `0x86`, …)
- forging definition headers
- chunk duplication and splicing between seeds
- truncation
- repairing `data_size` and the data CRC 75% of the time
- inputs capped at 64 KiB

It also has a corpus pool of up to 4,000 inputs. Per-input time is recorded, and any input slower than 1 s is saved.

**Corpus:** 26 seeds: 4 generated with the crate's encoder (normal, compressed-timestamp, big-endian and chained; they include developer data, `set`, `exercise_title`, `session`, `record`, and SINT64/FLOAT64/STRING fields), the 19 official SDK FITs and 2 Garmin-forum FITs from the repo's `tests/data`, and the repo's `examples/no_std_decode/src/sample.fit`. The published package contains no `.fit` files.

| Run | Target | Duration | Execs | Inputs decoding ≥1 message | Encodes OK (oracle checked) | Distinct crashes | Slowest input |
|---|---|---|---|---|---|---|---|
| 1 | (a) decode(no-expand) → transform → encode → decode | 660 s | 2,666,936 | 263,690 | 204,889, 0 oracle failures | 2: `mod.rs:410` (140,059 hits, min 32 B), `mod.rs:475` (1,012 hits, min 118 B) | 428 ms |
| 1 | (b) decode(default) → Session/Set/Record::from, plus `Message::from(Set)` | 660 s | 1,977,368 | 180,888 | n/a | the same 2 (213,040 and 702 hits; min 31 B and 118 B) | 332 ms |
| 2 | (a) with the C3 pre-scan first (first version: stopped at the first CRC) | 610 s | 4,253,589 | 353,534 | 320,725, 0 oracle failures | 2. The **pre-scan was bypassed** by chained inputs (674 B and 806 B; a second sequence after byte 360). | 159 ms |
| 2 | (b) with that pre-scan | 610 s | 6,579,901 | 560,830 | n/a | 1 (`mod.rs:410`, 601 B, the same chained bypass) | 38 ms |
| 3 | (a) with the fixed pre-scan (scans all sequences) | 610 s | 3,197,139 (2,921,185 rejected by pre-scan) | 274,396 | 253,171, 0 oracle failures | **0** | 122 ms |
| 3 | (b) with the fixed pre-scan | 610 s | 7,381,051 (6,763,696 rejected by pre-scan) | 612,633 | n/a | **0** | 47 ms |

No OOMs or timeouts occurred under the 64 KiB input cap; the slowest input was 428 ms. With the fixed pre-scan in front (run 3), about 1.4 M non-rejected inputs (by subtraction) produced **no panics**. A caveat for run 3: the pre-scan rejects about 92% of mutants, because zero-size fields are common mutations. F2 and F3 were found by code review and confirmed with constructed PoCs (§5), not by the fuzzer. Fuzzer-found reproducers are in `crashes_a/`, `crashes_b/` and `crashes_m*/`. The hand-minimised reproducers are in `scratchpad/rustyfit-audit/pocs/`:

- `poc_divzero_field.fit` (27 B)
- `poc_divzero_devfield.fit` (47 B)
- `poc_amplify_100k.fit`
- `poc_quadratic_100k.fit`

## 8. The crate's own tests and tooling

The package keeps its `#[cfg(test)]` unit tests but not `tests/` or `tests/data/`. I ran the tests on a copy of the package:

```
$ cargo test --locked --lib          (copy of the package)
test decoder::tests::test_stream ... FAILED   (File::open("tests/data/large.fit") -> not shipped)
test result: FAILED. 25 passed; 1 failed
$ cargo test --release --tests       (repo at 92915bf)
unittests: ok. 26 passed;  tests/roundtrip.rs: ok. 3 passed;  serde_json_roundtrip: 0 tests (feature off)
```

Clippy:

```
$ cargo clippy --locked --lib                         -> 0 warnings
$ cargo clippy --lib -- -W clippy::pedantic           -> style only (doc backticks, must_use, cast lints)
$ cargo clippy --lib -- -W clippy::arithmetic_side_effects -W clippy::indexing_slicing -W clippy::unwrap_used
     -W clippy::integer_division -W clippy::undocumented_unsafe_blocks
   481 arithmetic, 257 indexing, 62 slicing, 17 unwrap, 11 integer_division, 9 unsafe-without-SAFETY-comment
```

I reviewed the restriction-lint hits in `decoder/`, `encoder/`, `proto.rs` and `crc16.rs` by hand (§5). Most hits are float scaling in generated mesgdef code. The 9 `unsafe` blocks have no `// SAFETY:` comments.

Note: `cargo clippy --all-targets` on Windows failed to link dev-dependency build scripts (criterion's crossbeam-utils and proc-macro2) because the scratch path was too long (`LNK1104`). That is environmental. The crate has no fuzz targets and no Miri CI of its own.

## 9. Maintenance and provenance

- **Owners:** crates.io `/api/v1/crates/rustyfit/owners` lists one owner, `muktihari` (Hikmatulloh Hari Mukti). The repo has 136 commits by Hikmatulloh Hari Mukti plus 5 by "Mukti" (the same email), and 3 from dependabot.
- **Releases** (crates.io): 0.1.0 on 2025-04-10 (crate created 2025-04-10T07:44Z), then 0.2.0–0.2.2 in 2025. 0.3.0 came on 2026-01-31, then 0.4.0, 0.4.1, 0.5.0, then 0.6.0 (2026-05-18), 0.6.1, 0.7.0, 0.8.0, 0.8.1, 0.9.0, 0.10.0 (2026-06-23), 0.10.1, 0.10.2 and **0.10.3 (2026-09-11)**. All were published by muktihari and none are yanked. That is 11 releases in four months, several with breaking changes.
- **Release commit:** the release commit and tag are unsigned (`git log %G?` = `E`; `v0.10.3` is a lightweight tag). The package's `src/` is **byte-identical** to `rustyfit/src` at `92915bf` (339/339 files have matching git blob hashes, computed with `git hash-object --no-filters` against `git ls-tree`).
- **Other package files:** the package's `LICENSE-*` files equal the repo-root files, which `rustyfit/LICENSE-*` link to. `Cargo.toml.orig` equals `rustyfit/Cargo.toml`. The package's `Cargo.lock` is a regenerated subset of the workspace lock (it lacks the `no_std_decode` example's `talc`, `spinning_top`, `lock_api` and so on).
- **Upstream since the release:** 11 commits on `master` after 0.10.3. These include `aced7f2` (F6 fix) and `35f933f` (breaking: Value uses boxed slices). F1 is unchanged.
- **Advisories:** RustSec advisory-db (1,314 advisories) has no entry for rustyfit or embedded-io. GitHub `security-advisories` returns `[]`. Issues: #79 "road to heapless" (open), #77 and #119 closed; nothing security-related.
- **Repo:** github.com/muktihari/rustyfit, 15 stars, 1 fork, not archived, last pushed 2026-10-14. 7,652 total downloads.
- **Licence:**
  - The crate is BSD-3-Clause (2025, Hikmatulloh Hari Mukti): keep the copyright notice and licence text in source and binary distributions, and no endorsement.
  - The generated profile (`src/profile/**`, from Garmin's Profile.xlsx) is shipped with `LICENSE-FIT-SDK`, Garmin's *FIT Protocol License Agreement*. Its headers say "governed by a BSD-style license", which looks inconsistent with that.
  - My summary of the Garmin licence, which is not legal advice:
    1. The grant is non-exclusive, royalty-free and non-transferable, "for Licensee's internal business purposes, including to use the FIT protocol in any software created by Licensee".
    2. §2c: don't redistribute or "make available" the Licensed Technology itself except as allowed.
    3. §2d: don't distribute it or its derivatives under a licence that would force Garmin's IP to be disclosed in source form. In other words, no copyleft relicensing of the generated profile.
    4. §2a: don't remove Garmin's notices.
    5. §2g: no safety-critical use.
    6. §2h and §3: comply with and keep interoperability with the FIT protocol. That means we must emit spec-compliant files.
    7. §5: any Modifications are licensed back to Garmin.
    8. §9: Garmin may terminate on 30 days' notice.
    9. §2f: no benchmarking or competitive analysis of the SDK.
  - **What this means for us:**
    - Depending on the crate from crates.io (not vendoring it) avoids redistributing SDK source from our repo.
    - Our Docker image will contain compiled profile tables. Whether that counts as "making available" is a question for the project's licence review; it is common practice in the FIT ecosystem.
    - Don't license the repo under GPL/AGPL in a way that would cover the profile.
    - Include both licence texts with the image.
    - Keep output files valid FIT.

## 10. cargo-audit / cargo-deny

Project: `rustyfit = "=0.10.3"` plus `embedded-io-adapters = { version = "0.7", features = ["std"] }` (Cargo.lock has 4 packages).

```
$ cargo audit --version   -> cargo-audit-audit 0.22.2
$ cargo audit -d <advisory-db>
      Loaded 1314 security advisories
    Scanning Cargo.lock for vulnerabilities (4 crate dependencies)
(exit 0, no vulnerabilities, no warnings)

$ cargo deny --version    -> cargo-deny 0.20.2
$ cargo deny check        (deny.toml: yanked=deny, multiple-versions=deny, wildcards=deny, unknown registry/git=deny,
                           licenses allow BSD-3-Clause/MIT/Apache-2.0, private crates ignored)
advisories ok, bans ok, licenses ok, sources ok
$ cargo deny list
Apache-2.0 (2): embedded-io@0.7.1, embedded-io-adapters@0.7.0
BSD-3-Clause (1): rustyfit@0.10.3
MIT (2): embedded-io@0.7.1, embedded-io-adapters@0.7.0
```

cargo-deny can't see the FIT SDK licence, because rustyfit's `license` field declares only `BSD-3-Clause`.

## Conditions of use

1. **Pin** `rustyfit = "=0.10.3"` and commit `Cargo.lock`. CI should run `cargo deny check` with `yanked = "deny"`.
2. **Never enable** the `serde` feature: it adds a proc macro and build scripts.
3. **Wrap** decode, `mesgdef::*::from`, our transformation and encode in `std::panic::catch_unwind(AssertUnwindSafe(..))`, and keep `panic = "unwind"` (do not set `panic = "abort"` in the release profile). A caught panic should mark that activity as failed, log it, and stop it from being retried every poll.
4. **Pre-scan** every downloaded file before `Decoder::decode`, using `Decoder::stream` with `expand_components(false)`. The reference implementation is `scratchpad/rustyfit-audit/poc/src/lib.rs::prescan`. It found no false positives on the 27 sample files and blocks all 4 PoCs. Reject the file if any of these hold:
   - a field definition has `size == 0` or `base_type.size() == 0`
   - a developer field definition has `size == 0`
   - more than 256 `field_description` messages
   - a `field_description` without a valid `fit_base_type_id` (field 2 must be `Uint8` with `FitBaseType(b).size() != 0`)
   - more than 5,000,000 total fields and developer fields
   - any decode error

   **Scan every chained sequence**: don't stop at the first `Crc` event, or decode only one sequence (F9). The stream decoder reuses one `Message`, so the pre-scan uses constant memory.
5. **Cap input size** at 4 MiB, checked against `Content-Length` and while streaming the body. Strength-training activities are far smaller. After C4 the worst case is about 80 B of heap per input byte, around 320 MiB.
6. **Time-box** each sync job, for example 30 s of wall clock, and treat a timeout as a failed activity.
7. Use only `HeaderOption::Normal(_)`; **never** `HeaderOption::Compressed` (F6). `Normal(15)` keeps output size close to the input (F10).
8. Build inserted `set` fields with `is_expanded: false`, profile base types, and `Value::VecUint16` for fields 7 and 8 (F11). Handle `Encoder::encode` errors (F4) by skipping the upload, not by panicking.
9. Keep `checksum(true)` (the default) when decoding.
10. Avoiding `Message::from(mesgdef::Set)` is fine but not necessary for memory safety (§4).
11. Report F1 (and optionally F2 and F3) upstream, and re-audit when a fixed version ships.

## How to re-audit on a version bump

```sh
NEW=0.10.4   # target version
W=<scratch>/rustyfit-$NEW && mkdir -p $W/fetch/src && cd $W/fetch
printf '[package]\nname="fetch"\nversion="0.0.0"\nedition="2021"\npublish=false\n[dependencies]\nrustyfit="=%s"\nembedded-io-adapters={version="0.7",features=["std"]}\n' $NEW > Cargo.toml
echo 'fn main(){}' > src/main.rs && cargo fetch && grep -A3 'name = "rustyfit"' Cargo.lock      # checksum
R=$(ls -d ~/.cargo/registry/src/*/rustyfit-$NEW); cat $R/.cargo_vcs_info.json                  # release sha

# 1. package vs git (byte-exact)
git clone https://github.com/muktihari/rustyfit $W/repo && SHA=$(python -c "import json;print(json.load(open('$R/.cargo_vcs_info.json'))['git']['sha1'])")
git -C $W/repo checkout $SHA
git -C $W/repo ls-tree -r HEAD rustyfit/src | awk '{print $3" "$4}' | sed 's| rustyfit/| |' | sort -k2 > $W/git.txt
(cd $R && find src -type f | sort | while read f; do echo "$(git hash-object --no-filters $f) $f"; done) > $W/pkg.txt
diff $W/git.txt $W/pkg.txt && echo IDENTICAL
git -C $W/repo diff --stat v0.10.3 $SHA -- rustyfit/src        # review the delta, focus on decoder/, encoder/, proto.rs

# 2-3. build-time code, deps, ambient capability
grep -nE '^build|proc-macro|\[build-dependencies' $R/Cargo.toml; ls $R/build.rs
cargo tree -e normal,build; cargo tree -e normal,build --features rustyfit/serde
grep -n 'no_std' $R/src/lib.rs; grep -rn 'std::\|extern "C"\|#\[link' $R/src | grep -v '//'

# 4. unsafe inventory (expect the same 9 sites; re-verify the repr(transparent) typedefs)
grep -rn unsafe $R/src
grep -rnE 'transmute|get_unchecked|from_utf8_unchecked|MaybeUninit|static mut|unsafe impl|set_len|\.add\(|\.offset\(' $R/src
cd <scratch>/rustyfit-audit/miri && sed -i "s/=0.10.3/=$NEW/" Cargo.toml && cargo +nightly miri test

# 5. panic / DoS regression: PoCs must not panic, prescan must still reject them
cd <scratch>/rustyfit-audit/poc && sed -i "s/=0.10.3/=$NEW/" Cargo.toml && cargo build --release
for b in divzero devdivzero; do ./target/release/$b; done
./target/release/amplify 20000; ./target/release/quadratic 8000 8000
./target/release/prescan ../pocs/*.fit <repo>/rustyfit/tests/data/*/*.fit
grep -n '% base_type.size()' $R/src/decoder/mod.rs                 # F1 still present?

# 6. round trip on the sample corpus
./target/release/roundtrip <repo>/rustyfit/tests/data/*.fit <repo>/rustyfit/tests/data/*/*.fit

# 7. fuzz (Linux/Docker, preferred: gives coverage)
docker run --rm -v <scratch>/rustyfit-audit:/w -w /w rustlang/rust:nightly sh -c \
  'cargo install cargo-fuzz && sed -i "s/=0.10.3/=NEW/" fuzz/Cargo.toml && \
   for t in decode_reencode decode_mesgdef; do mkdir -p fuzz/corpus/$t && cp fuzz/seeds/* fuzz/corpus/$t/; \
   cargo fuzz run --fuzz-dir fuzz -a -O $t fuzz/corpus/$t -- -max_total_time=600 -rss_limit_mb=2048 -timeout=10 -max_len=65536 \
     -fork=4 -ignore_crashes=1; done'
#    fallback (any OS): ./target/release/mutfuzz a ../fuzz/seeds 600 ../crashes_a ; MITIGATE=1 ... (with prescan)

# 8. tests + clippy
cd $W/repo/rustyfit && cargo test --release --tests && cargo clippy --lib -- -W clippy::undocumented_unsafe_blocks

# 9-10. provenance + advisories
curl -s https://crates.io/api/v1/crates/rustyfit/owners; curl -s https://crates.io/api/v1/crates/rustyfit | jq '.versions[:5]'
gh api repos/muktihari/rustyfit/security-advisories; gh issue list -R muktihari/rustyfit --state all
cd $W/fetch && cargo audit && cargo deny check
```
