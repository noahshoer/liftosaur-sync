# liftosaur-sync Implementation Guide

> **How this plan is used:** you write the code yourself, to stay sharp. Claude is planner and crate auditor, and can answer questions about the external APIs.
> - Each task says **what** to build: the interfaces, the behaviours to test, which APIs to read, and the gotchas.
> - It deliberately leaves out the **how** (no solution code).
> - Ask Claude about an external API, to review a task ("review task 3"), or to unstick you. Hints come first, code only if you ask for it.
> - Tick the checkboxes as you go.

**Goal:** A Home Assistant app (Rust, arm64, GHCR) that uploads every Garmin activity from Intervals.icu to Strava exactly once. For strength workouts, it first rewrites the FIT file's sets from the matching Liftosaur workout, so Strava shows the right exercises while keeping the watch's heart rate. It also writes the exercise lines into the Intervals.icu description.

**Architecture:**
- One synchronous Rust binary with two threads:
  - **Poll thread:** owns `State` and runs one sync pass every `poll_interval_seconds`, or sooner when woken through a channel.
  - **Web thread:** serves the ingress status page, the Strava connect form and `/trigger`. It talks to the poll thread only through an `mpsc` channel and a read-only `Arc<Mutex<State>>` snapshot.
- API clients sit behind traits, so the sync engine is tested with in-memory fakes.

**Tech stack (to be confirmed by the audits in Task 2):** Rust 2024 (MSRV 1.93), `rustyfit =0.10.3`, `embedded-io-adapters` 0.7, `ureq` 3, `tiny_http` 0.12 (see the audit note), `serde`/`serde_json`, `toml` 1, `chrono` 0.4, `flate2` 1, `base64` 0.23, `anyhow`, `log` + `env_logger`.

**Spec:** `docs/superpowers/specs/2026-10-13-liftosaur-sync-design.md`

## Global Constraints

- **The repo is public from day one.** No secret ever goes in git. Every API key, ID and secret is set in the HA app's Configuration tab and read from `/data/options.json`.
  - Local runs use `options.local.json`, which is gitignored.
  - CI needs no API secrets; only the publish workflow uses `GHCR_PAT`.
- Turn on GitHub secret scanning and push protection.
- Rust `edition = "2024"`, `rust-version = "1.93"` (rustyfit's MSRV). The local toolchain is 1.96.
- `rustyfit` is pinned `=0.10.3`, Dependabot ignores it, and it only gets bumped after reviewing the diff against its audit doc. Its `serde` feature stays off. Keep `panic = "unwind"` (never `abort`), because the decoder guard relies on `catch_unwind`. The audit and its conditions are in `docs/audits/rustyfit-0.10.3.md`.
- `Cargo.lock` is committed, and every build uses `--locked`. Run `cargo fmt` before committing; CI runs `fmt --check`, `clippy -D warnings`, `test`, `cargo-deny` and `cargo-audit`.
- Image: `ghcr.io/noahshoer/{arch}-liftosaur-sync`, `aarch64` only, built in CI on `ubuntu-24.04-arm` (free for public repos).
- Only Intervals.icu activities with `source == "GARMIN_CONNECT"` and start ≥ `sync_after` are processed.
- The Strava description is never set. On Intervals.icu, only the description of matched strength activities changes, and only between the `[liftosaur-sync]` and `[/liftosaur-sync]` lines.
- Option defaults: poll 150 s, notify after 30 min, give up after 180 min, `include_warmups` true, `dry_run` **true**.
- Matching: starts within 30 min **and**, when both durations are known, overlap ≥ 50 % of the shorter one. One workout matches at most one activity.
- State entries older than 60 days are pruned. Retry backoff is capped at 1 h.
- Private watch FITs for testing live in gitignored `tests/data/private/`. CI tests use synthetic FITs built in code.

## Review Focus

These are the five failure cases most likely to bite. Each has a test listed in the task that owns it.
1. **Restart after a Strava upload but before state was saved.** The re-upload comes back as "duplicate of activity N", which must be recorded as uploaded, and the description must still be written. (Task 11)
2. **A Liftosaur record that doesn't parse.** It's skipped, other workouts still match, nothing panics. (Tasks 5 and 11)
3. **A Garmin file that isn't a usable FIT** (GPX, or a FIT without a session). It's uploaded unchanged and never left pending. (Task 11)
4. **A corrupt `/data/state.json`.** The app starts, keeps a backup, and Strava's duplicate check stops double uploads. (Task 9)
5. **The UI reached from outside HA ingress** gets 403. `/trigger` with a wrong or missing secret gets 401. (Task 12)

## Gotchas found while researching (read once before starting)

- **rustyfit `std` use:** wrap readers and writers in `embedded_io_adapters::std::FromStd`. `Encoder::encode` needs `Write + Seek`, so encode into `FromStd::new(&mut Cursor<Vec<u8>>)`.
- **Decode with `Decoder::builder().expand_components(false).build()`** when you intend to re-encode.
- **Comparing messages:** compare them by `(num, fields, developer_fields)`, not as whole `Message`s. The `header` byte holds the local message number, and the encoder reassigns those.
- **The decoder isn't safe on hostile input** (audit F1–F3):
  - a 27-byte file panics it (a remainder by zero on invalid base types)
  - size-0 field definitions use about 10 KB of memory per input byte, an OOM that can't be caught
  - developer-field lookups take quadratic time

  So every file goes through the guard in Task 3 before `Decoder::decode`.
- **Building `set` messages:** `Message::from(mesgdef::Set)` is sound per the audit (its `unsafe` cast is a `repr(transparent)` newtype), but building the `Field`s yourself is still the clearer route. The field-number constants are on `mesgdef::Set` (`TIMESTAMP=254, DURATION=0, REPETITIONS=3, WEIGHT=4, SET_TYPE=5, START_TIME=6, CATEGORY=7, CATEGORY_SUBTYPE=8, MESSAGE_INDEX=10`). Use `is_expanded: false`, because `encode` drops expanded fields. Fields 7 and 8 must be `Value::VecUint16`, or `Set::from` ignores them (F11).
- **Encoder settings:** encode with `HeaderOption::Normal(15)` to keep the output close to the input size (F10). **Never** use `HeaderOption::Compressed`: it corrupts timestamps in 0.10.3 (F6).
- **Developer fields the encoder refuses:** `Encoder::encode` rejects developer fields that have no `developer_data_id`, even though the decoder accepts them (F4). Treat that as "can't rewrite this activity", not a panic.
- **Set field units:**
  - `duration` is ms (scale 1000)
  - `weight` is kg × 16
  - `category` and `category_subtype` are *arrays* of u16
  - `set_type` 0 = rest, 1 = active
  - `timestamp` = set end time
- **rustyfit prints FIT enum values missing from the profile as `TypeName(n)`.** That's how you can tell an unknown exercise name.
- **Liftosaur history records are only `{id, text}`.** The text is Liftohistory format (see the Liftosaur MCP `get_liftohistory_reference`). There are no per-set timestamps, and it's unknown whether the header timestamp is the start or the end of the workout; Task 4 settles that.
- **Intervals.icu:**
  - basic auth with username `API_KEY`
  - `GET /api/v1/athlete/{id}/activities?oldest=&newest=` (dates)
  - `GET /api/v1/activity/{id}/file` returns **gzipped** bytes
  - `PUT /api/v1/activity/{id}` with `{"description": …}`
  - `source` values include `GARMIN_CONNECT` and `STRAVA`
- **Strava:**
  - `POST /api/v3/uploads` is multipart (`file`, `data_type=fit`, `external_id`) and returns an upload id. Poll `GET /api/v3/uploads/{id}` until `activity_id` or `error` is set.
  - Duplicates show up as `error` containing "duplicate of activity N" (sometimes wrapped in a link).
  - Tokens last 6 h, and the refresh token can rotate on every refresh, so always store the newest one.
  - `localhost` is an allowed redirect domain. A bad code or refresh token gets HTTP 400.
- **ureq 3:**
  - Build an `Agent` with `http_status_as_error(false)` so 4xx/5xx come back as responses you can classify.
  - It has no multipart builder, so you'll write a small one.
  - `read_to_vec()` stops at 10 MB by default; use `body_mut().with_config().limit(..)`.
- **HA:**
  - Ingress requests come from `172.30.32.2`.
  - Notifications: `POST http://supervisor/core/api/services/notify/<name>` with `Bearer $SUPERVISOR_TOKEN`. This needs `homeassistant_api: true`, and `run.sh` must use `with-contenv` so the token is visible.
  - Options arrive in `/data/options.json`.

---

### Task 1: Public GitHub repo, scaffold and CI — *you*

**Build:**
- `gh repo create noahshoer/liftosaur-sync --public --source . --remote origin --push`
- In the repo settings, turn on secret scanning + push protection.
- `cargo init` as lib + bin (`liftosaur_sync` lib, `liftosaur-sync` bin) with edition and MSRV as above. Add `rustyfit = "=0.10.3"` and nothing else yet. Add the other crates in the task that first needs them, once Task 2 has cleared them.
- `.gitignore`: `target`, `options.local.json`, `state.local.json`, `share/`, `tests/data/private/`, `fuzz/target`, `fuzz/corpus`, `fuzz/artifacts`.
- `deny.toml`:
  - licenses: allow only permissive ones (MIT, Apache-2.0, BSD-2/3, ISC, Unicode-3.0, Zlib, CDLA-Permissive-2.0 for webpki roots)
  - `wildcards = "deny"`
  - unknown registries and git sources denied
- `.github/dependabot.yml`: cargo (ignore `rustyfit`) and github-actions, weekly.
- `.github/workflows/ci.yml`: fmt, clippy, test, `EmbarkStudios/cargo-deny-action`, `cargo audit`.

**Done when:** CI is green on GitHub and the README links the spec.

- [ ] Repo created (public), existing commits pushed, push protection on
- [ ] Crate scaffolded, CI green

### Task 2: Crate audits — *Claude*, you review

**Claude produces** `docs/audits/<crate>-<version>.md`:
- **rustyfit 0.10.3, full audit,** on the crates.io source, not the repo:
  - build scripts and proc macros
  - dependency tree
  - `no_std` and the lack of ambient I/O
  - every `unsafe` site (9 `Vec::from_raw_parts` casts in `From<mesgdef::X> for Message`), each checked for `repr(transparent)` soundness
  - maintainer and release history
  - license (BSD-3 + FIT SDK)
  - a `cargo-fuzz` decode→encode run in Docker (nightly), plus a fuzz target over your `fitfile::rewrite` once Task 3 exists
- **Light audits of every other direct dependency:** version, maintenance, license, `unsafe` and build-script surface, and transitive count.
  - **Already flagged: `tiny_http` 0.12.0, last released 2022-10.** It's only exposed through HA ingress plus an internal `/trigger`, so the risk is low. The audit will set out the choice between:
    - accept it, pinned
    - a maintained sync server
    - axum + tokio, which means learning async

**You:** read the audits and decide on the tiny_http question. Nothing in Task 3+ adds a crate that hasn't been audited.

- [ ] rustyfit audit + fuzz run reviewed
- [ ] Other crates audited, HTTP server choice made

### Task 3: FIT names, decode/encode, rewrite — *you*

**Files:** `src/fit_names.rs`, `src/fitfile.rs`, `src/bin/fit_tool.rs`, `tests/real_fit.rs`

**Interfaces** (later tasks rely on these names):
- `fit_names`:
  - `UNKNOWN_CATEGORY: u16` (= 65534)
  - `category_code(&str) -> Option<u16>`
  - `exercise_code(category: u16, &str) -> Option<u16>`
  - `resolve(category: &str, exercise: Option<&str>) -> Result<(u16, Option<u16>), String>`
- `fitfile`:
  - `FitError`
  - `decode(&[u8]) -> Result<FIT, FitError>`
  - `encode(&mut FIT) -> Result<Vec<u8>, FitError>`
  - `gunzip_if_needed(Vec<u8>) -> Result<Vec<u8>, FitError>`
  - `ActivityInfo { start: DateTime<Utc>, elapsed_s: u32, is_strength: bool, watch_sets: Vec<WatchSet> }`
  - `WatchSet { start: FitTime, duration_ms: u32, active: bool }`
  - `info(&FIT) -> Result<ActivityInfo, FitError>`
  - `PlannedSet { category: u16, exercise: Option<u16>, reps: u16, weight_kg: Option<f64>, duration_s: Option<u32> }`
  - `rewrite(original: &[u8], &[PlannedSet]) -> Result<Vec<u8>, FitError>`
- `#[cfg(test)] fitfile::testutil::synthetic_fit(strength: bool, watch_sets: &[(offset_s, dur_s, active)]) -> Vec<u8>`: a 30-min activity with HR records and a session

**Behaviour:**
- **Strength check:** `info` reports strength when the session is sport `TRAINING` + sub-sport `STRENGTH_TRAINING`.
- **What `rewrite` changes:** it removes all `set` and `exercise_title` messages and inserts new `set` messages where the first old one was (or before the first lap/session). Everything else stays untouched.
- **Timing:**
  - If the watch's active-set count equals the planned count, reuse the watch's start times and durations.
  - Otherwise spread the sets evenly across the activity, with rest sets in between.
- **Validation:** `rewrite` re-decodes its own output (which checks the CRC) and confirms:
  - non-set messages are unchanged by content
  - the session start and elapsed time are unchanged
  - the active-set count equals the planned count

  It errors if any check fails.
- **Refusals:** `rewrite` refuses non-strength files and an empty plan.
- **Decoder guard** (audit conditions C3–C5, C9). `decode` must never be called on a file that hasn't passed it.
  - **Size cap:** reject input over 4 MiB.
  - **Pre-scan:** before decoding, walk **every chained sequence** with `Decoder::stream` (`expand_components(false)`), not just the first one (F9). Don't stop at the first `Crc` event. Reject the file on:
    - any field definition with `size == 0` or `base_type.size() == 0`
    - any developer field definition with `size == 0`
    - more than 256 `field_description` messages
    - a `field_description` whose `fit_base_type_id` (field 2) isn't a `Uint8` with a non-zero `FitBaseType(b).size()`
    - more than 5,000,000 fields + developer fields in total
    - any decode error

    The stream decoder reuses one `Message`, so the pre-scan uses constant memory.
  - **Panics:** wrap decode, the `mesgdef::*::from` conversions, the rewrite and encode in `std::panic::catch_unwind(AssertUnwindSafe(..))`, and turn a caught panic into a `FitError`.
  - **Checksums:** keep `checksum(true)`, the default, but don't count on it for DoS protection. It's only checked after the whole file is parsed (F8).
  - **Cost after the pre-scan:** about 80 B of heap per input byte at most, so roughly 320 MiB at the 4 MiB cap.
  - **No time-box needed:** the pre-scan bounds the decoder's work. A true time-box would need an abandonable worker thread, which isn't worth it.
  - **Reference:** the auditor wrote a pre-scan in its scratch dir. It's deliberately not copied into the repo.

**Tests to write:**
- `info` reads the synthetic session
- records/HR are unchanged after `rewrite`
- planned reps, weight (×16) and category land in the sets
- the watch-timing reuse path
- the even-spread path stays inside the activity
- non-strength and empty plan are rejected
- garbage, empty and truncated input give `Err` without panicking
- gzip passthrough and inflate
- the guard rejects each of the audit's PoC shapes, built in the test with the encoder or as raw bytes: a field with an invalid base type (F1, the 27-byte panic), size-0 fields (F2), more than 256 field descriptions (F3), and a bad file hidden in a second chained sequence (F9). None of them may panic.
- the guard accepts every real FIT in `tests/data/private/`, so it has no false positives
- `tests/real_fit.rs` loops over `tests/data/private/*.fit[.gz]` if present

**`fit_tool` dev CLI:** `info <in.fit>` and `rewrite <in.fit> <sets.json> <out.fit>`, where sets.json is `[{category, exercise, reps, weight_kg}]` by FIT name.

**APIs:** rustyfit's `Decoder`/`Encoder`, `proto::{FIT, Message, Field, Value}` and `profile::{mesgdef, typedef}` (docs.rs/rustyfit/0.10.3). The typedefs are `u16`/`u8` newtypes whose `Display` gives the FIT name. Also `flate2::read::GzDecoder`.

**Hint:** there are ~50 per-category exercise-name typedefs (`BenchPressExerciseName`, …). One `match category { 0 => BenchPressExerciseName(sub).to_string(), … }` is the simple route; a `macro_rules!` is the fun route.

- [ ] fit_names + tests
- [ ] fitfile + tests, fit_tool, real_fit
- [ ] Review with Claude

### Task 4: Does Strava draw the muscle map? (gate) — *you + Claude*

1. Download one strength activity's original FIT from Intervals.icu into `tests/data/private/`, write that day's sets as `spike-sets.json`, then run `fit_tool rewrite`.
2. **Liftosaur timestamp:** compare `fit_tool info`'s start with that workout's Liftosaur header timestamp. That decides whether the header is the workout's start or end (used in Task 5).
3. Delete that one activity on Strava and upload the rewritten FIT at strava.com/upload/select. Check the muscle map, the exercises/reps/weights and the HR graph.
4. **Decide:** all good → continue. Otherwise, stop and revisit the design. The fallback is the JSON upload with an HR stream.

- [ ] Muscle map confirmed, timestamp semantics recorded here: ____

### Task 5: Liftohistory parser — *you*

**File:** `src/liftohistory.rs`

**Interfaces:**
- `DATE_IS_END: bool` (from Task 4)
- `ParseError`
- `Workout { id: u64, date: DateTime<Utc>, duration_s: Option<u32>, exercises: Vec<Exercise> }`, with `Workout::start()`
- `Exercise { name, equipment: Option<String>, notes: Vec<String>, completed: String /* raw */, warmup: Option<String> /* raw */, sets: Vec<LiftSet>, warmup_sets: Vec<LiftSet> }`, with `full_name()`
- `LiftSet { reps: u16, weight_kg: Option<f64>, duration_s: Option<u32> }`
- `parse(id, &str) -> Result<Workout, ParseError>`
- `parse_sets(&str) -> Vec<LiftSet>`

**Behaviour:**
- **Header:** the first non-`//` line is the header, `date / key: value / … / exercises: {`. Read the date (RFC 3339) and the optional `duration: Ns`.
- **Exercise lines:** sections are separated by ` / `.
  - The first section is `Name[, Equipment]`.
  - The first section without a prefix is "completed".
  - `warmup:` is the warm-ups.
  - `target:` is ignored.
- **Notes:** `//` lines attach to the next exercise.
- **Sets:**
  - `3x8 185lb @7` gives 3 sets; lb → kg uses × 0.45359237
  - `5+` means AMRAP, so 5 reps
  - `8|7` means 8, the larger side
  - `60s` and `40s|35s` are holds (take the larger); `?` means unknown
  - groups that don't parse are skipped
- **Failures:** a bad date is an `Err`. Odd exercise lines never panic.

**Tests to write:** the Liftohistory reference example, all of the set notations above, a missing duration, a bad date, garbage lines, notes, and `start()` with `DATE_IS_END`.

- [ ] Parser + tests, reviewed

### Task 6: Exercise mapping and set planning — *you*

**Files:** `data/exercises.toml`, `src/mapping.rs`

**Interfaces:**
- `Entry { liftosaur, equipment: Option<String>, category: String, exercise: Option<String> }` (serde)
- `FitExercise { category: u16, exercise: Option<u16> }`
- `Mapping::builtin()` and `Mapping::with_overrides(&[Entry]) -> Result<(Mapping, Vec<String> /* warnings */), String>`
- `Mapping::lookup(name, equipment) -> Option<FitExercise>`
- `Plan { sets: Vec<PlannedSet>, unmapped: BTreeSet<String> }`
- `plan_sets(&Workout, &Mapping, include_warmups) -> Plan`

**Behaviour:**
- **Table:** compiled in with `include_str!`.
- **Lookup:** keys are lowercased `(name, equipment)`, falling back to `(name, "")`.
- **Overrides:** they win. An invalid override becomes a warning, not a crash. An invalid **built-in** entry fails a test.
- **Unmapped exercises** get `UNKNOWN_CATEGORY` and are listed.
- **Warm-ups** come before each exercise's working sets when included.

**Table:**
- Build it from Liftosaur's exercise list (MCP `list_exercises`) and FIT names from `rustyfit-0.10.3/src/profile/typedef/*_exercise_name.rs`. Use liftosaur2garmin only to cross-check.
- Already confirmed to exist in rustyfit: `barbell_bench_press`, `dumbbell_bench_press`, `incline_barbell_bench_press`, `barbell_back_squat`, `barbell_front_squat`, `goblet_squat`, `leg_press`, `barbell_deadlift`, `romanian_deadlift`, `sumo_deadlift`, `overhead_barbell_press`, `dumbbell_shoulder_press`, `barbell_row`, `one_arm_bent_over_row`, `seated_cable_row`, `pull_up`, `chin_up`, `lat_pulldown`, `barbell_biceps_curl`, `dumbbell_biceps_curl`, `triceps_pressdown`, `lying_ez_bar_triceps_extension`, `body_weight_dip`, `dumbbell_lunge`, `dumbbell_bulgarian_split_squat`, `dumbbell_lateral_raise`, `dumbbell_flye`, `barbell_hip_thrust_with_bench`, `standing_calf_raise`, `seated_calf_raise`, `leg_curl`, `hanging_leg_raise`, `plank`, `push_up`.
- **Not present:** `hammer_curl`, `skull_crusher`, `lying_leg_curl`, `seated_leg_curl`.
- After Task 12 exists, its `unmapped` command lists the exercises from your own history that still need entries.

**Tests:** the built-in table resolves fully, case-insensitive lookup + fallback, overrides replace and invalid ones warn, `plan_sets` ordering, unmapped tracking, warm-ups toggled.

**APIs:** `toml` 1.x (`toml::from_str`).

- [ ] Mapping + starter table + tests, reviewed

### Task 7: Intervals.icu description block — *you*

**File:** `src/description.rs`

**Interfaces:**
- `START`/`END` marker consts
- `render_block(&Workout, include_warmups) -> String`
- `merge(existing: Option<&str>, block: &str) -> String`

**Behaviour:**
- **Block contents:** a marker line, then for each exercise with completed sets: its notes as `// …`, then `Full Name / <completed>[ / warmup: <warmup>]`, then the end marker. No `target:`.
- **Merge:**
  - empty or missing description → just the block
  - an existing block → replace only that block
  - otherwise → append after a blank line

**Tests:** render with and without warm-ups, skipping exercises with no completed sets, all three merge cases, text outside the markers preserved.

- [ ] Done, reviewed

### Task 8: Matcher — *you*

**File:** `src/matcher.rs`

**Interface:** `best_match<'a>(start: DateTime<Utc>, elapsed_s: u32, &'a [Workout], used: &HashSet<u64>) -> Option<&'a Workout>`

**Behaviour:** apply the matching rule from the Global Constraints. The largest overlap wins, then the closest start. With an unknown duration, match on start time only.

**Tests:** a clean overlap, a 31-min start difference, a 33 % overlap rejected, best of two, a used workout skipped, an unknown duration.

- [ ] Done, reviewed

### Task 9: Persisted state — *you*

**File:** `src/state.rs`

**Interfaces:**
- `Status { Pending, Notified, Uploaded, DryRun, Failed, Skipped }`, with `is_done()`
- `StravaTokens { access_token, refresh_token, expires_at: i64 }`
- `ActivityState { status, start, first_seen, matched_workout, strava_activity_id, description_written, description_failed, attempts, next_attempt, last_error }`, with `new(start, now)`
- `State { strava, strava_auth_problem, activities: BTreeMap<String, ActivityState>, unmapped, last_poll, last_poll_error }`, with:
  - `load_or_recover(&Path)`
  - `save(&Path)`: atomic, via a temp file and rename
  - `prune(now)`: removes done entries older than 60 d
  - `used_workouts(except_id) -> HashSet<u64>`

**Behaviour:**
- A missing file gives a default state.
- A corrupt file is renamed to `state.json.corrupt-<unix>` and the app starts fresh.
- Fields added later must not break old files, so use `#[serde(default)]`.

**Tests:** save/load round-trip, no leftover temp file, corrupt file backed up and reset (Review Focus 4), prune only drops old finished entries, `used_workouts` excludes itself.

- [ ] Done, reviewed

### Task 10: API clients — *you*

**Files:** `src/clients/{mod,http,liftosaur,intervals,strava,ha}.rs`, plus `#[cfg(test)] fake_server.rs`

**Interfaces (`clients::`):**
- `ApiError { Transient(String), Unauthorized(String), Permanent { status, body } }`
- `UploadOutcome { Created(u64), Duplicate(u64) }`
- `HistoryRecord { id, text }`
- `IcuActivity { id, start_date, start_date_local, kind /* "type" */, source, elapsed_time, file_type }`, with `start_utc()`
- Traits:
  - `Liftosaur::history(start: NaiveDate, end: NaiveDate)`
  - `Intervals::{list_activities, download_file, get_description, set_description}`
  - `Strava::{exchange_code, refresh, upload(token, bytes, data_type, external_id)}`
  - `Notifier::notify(&str)`: infallible, it logs its own errors
- Concrete clients take a `base` URL, so tests can point them at a local fake server.

**Behaviour:**
- **Error classes:** 401/403 → `Unauthorized`; 429/5xx/network → `Transient`; anything else → `Permanent`.
- **Liftosaur:** follow `hasMore`/`nextCursor`.
- **Intervals.icu:** gunzip downloads, and cap the body at 4 MiB both compressed and after gunzip (audit C5). A bigger file is `Permanent`. Parse `start_date` with or without the `Z`.
- **Strava uploads:** poll with a configurable delay (zero in tests) and a max poll count. Parse the duplicate id. A refresh that gets 400 → `Unauthorized`.
- **Notify:** a missing or blank service means log only.

**Tests:** run each client against a `tiny_http` fake that replays canned responses and records requests. Check auth headers, query strings, pagination, the multipart layout, duplicate parsing and status classification.

**APIs:** `ureq` 3 (`Agent::config_builder`, `.query`, `.header`, `send_form`, `send_json`, `body_mut().read_json()`, at docs.rs/ureq/3) and the `http` types it re-exports. `base64` 0.23 needs the `Engine` trait in scope. There's no multipart support, so follow RFC 7578.

- [ ] Clients + fakes + tests, reviewed

### Task 11: Sync engine — *you* (the biggest task; ask for a design walk-through first if you like)

**File:** `src/sync.rs`

**Interfaces:**
- `Settings { sync_after, notify_after, give_up_after, include_warmups, dry_run, dry_run_dir, retry_base }`
- `Syncer<'a> { settings, mapping, liftosaur: &dyn Liftosaur, intervals: &dyn Intervals, strava: &dyn Strava, notifier: &dyn Notifier }`
- `run_once(&self, &mut State, now, commit: &mut dyn FnMut(&State))`

**Behaviour:** the flow in the spec, plus:
- **Activity window:** list activities from max(`sync_after`, now − 7 d) to tomorrow.
- **Liftosaur history:** fetch it lazily, once per pass (now − 8 d … tomorrow). Skip records that don't parse.
- **Description-only retry:** an `Uploaded` activity with a match but no description gets only the description retried.
- **Strava `Unauthorized`** (including "not connected"): set `strava_auth_problem`, notify once, and pause uploads until it's cleared.
- **Liftosaur/Intervals `Unauthorized`:** a poll-level error. Record it in `last_poll_error` and notify once while it lasts.
- **`Transient`:** `attempts += 1` and `next_attempt = now + min(retry_base × 2^attempts, 1 h)`.
- **`Permanent`:** status `Failed` and one notification. If the upload already succeeded, set `description_failed` instead.
- **Rewrite failure:** upload the original and note why. This includes `encode` refusing developer fields (F4).
- **Guard failure:** a file the decoder guard rejects, or a caught decoder panic, is uploaded unchanged as a non-FIT would be. It's logged with the reason and never retried every poll.
- **Dry run:** write `<id>.<type>` and `<id>.description.txt` into `dry_run_dir`, set status `DryRun`, and never call Strava or `set_description`.
- **Tokens:** refresh when the access token has less than 5 min left, and store the new tokens.
- **Saving:** call `commit` after each activity.

**Tests (with in-memory fakes):**
- a run uploads unchanged
- a matched strength workout is rewritten (active-set count) and described
- an unmatched one: pending → one notification at 30 min → original uploaded at 180 min
- duplicate after a restart still writes the description (RF1)
- a description failure is retried without re-uploading
- Strava revoked: pauses, notifies once, resumes after reconnect
- transient backoff
- a permanent failure → `Failed` + one notification
- other sources and pre-`sync_after` activities are ignored
- GPX and garbage FITs upload unchanged (RF3)
- an unparseable history record is skipped (RF2)
- one workout matches only one of two activities
- dry run writes files and makes no Strava calls
- an expired token is refreshed and saved
- a bad Intervals.icu key: recorded, notified once

- [ ] Engine + tests, reviewed

### Task 12: Config, web UI, main loop, CLIs — *you*

**Files:** `src/config.rs`, `src/web.rs`, `src/main.rs`

**Interfaces:**
- **`config`:**
  - `Options`, with the fields in the spec's option table and serde defaults
  - `load(&Path)`: fails on empty required secrets, or when give_up < notify
  - `sync_after_utc()`: accepts `YYYY-MM-DD` or RFC 3339
  - `poll_interval()`: clamped to 60–3600 s
  - `settings(dry_run_dir)`
- **`web`:**
  - `Command { Trigger, StravaCode(String) }`
  - `Web { snapshot, tx, webhook_secret, authorize_url, warnings }`
  - a pure `handle(&Web, method, path, remote_ip, headers, body) -> Reply`, plus `serve(addr, Web)`
- **`main`:**
  - `serve` (default): reads `LIFTOSAUR_SYNC_OPTIONS`/`_STATE`/`_DRY_RUN_DIR`/`_ADDR`, with `/data/...` defaults
  - `unmapped`: lists unmapped exercises from the last year of history, by count
  - `rewrite-workout <in.fit> <out.fit>`: matches against Liftosaur and writes the FIT, then prints the description

**Behaviour:**
- **UI routes** (`GET /`, `POST /strava/code`) only answer `172.30.32.2`; anyone else gets 403.
- **`/trigger`** needs the header `X-Liftosaur-Sync-Secret`, which must equal `webhook_secret`. Otherwise it's 401.
- **Strava connect:** accept the pasted redirect URL or the bare code, percent-decoding the form body.
- **Escaping:** HTML-escape every dynamic value.
- **Poll loop:** `recv_timeout(poll_interval)`. On `StravaCode`, exchange the code, store the tokens or the error, then commit.

**Tests:** config defaults and validation; for `web`, status renders for ingress, non-ingress gets 403 (RF5), code extracted from a pasted URL, `/trigger` secret checks (RF5), HTML escaping.

**Local dry run:** with your own `options.local.json`, run it against the real services with `dry_run: true`, then fill `data/exercises.toml` using `unmapped`.

**APIs:** `tiny_http` 0.12, or whatever Task 2 chose. `Request::remote_addr` gives you the ingress-IP check.

- [ ] Done, local dry run OK, mapping filled from history, reviewed

### Task 13: HA app packaging + image publishing — *you*

**Files:** `repository.yaml`, `.dockerignore`, `liftosaur-sync/{config.yaml, Dockerfile, run.sh, DOCS.md, README.md, CHANGELOG.md}`, `.github/workflows/publish-ha-image.yml`. Model these on `../travelmap`.

**Key points:**
- **`config.yaml`:**
  - `slug: liftosaur_sync`, `arch: [aarch64]`
  - `ingress: true`, `ingress_port: 8099`
  - `homeassistant_api: true`, `map: [share:rw]`
  - secrets typed `password`, optional fields `str?`
  - a `custom_exercises` list schema
  - `image: ghcr.io/noahshoer/{arch}-liftosaur-sync`
  - **every API credential is an option, so nothing secret lives in the repo**
- **Dockerfile:** a `rust:1-alpine` builder, then `FROM ${BUILD_FROM}`; set the `LIFTOSAUR_SYNC_*` env vars. `run.sh` is `#!/usr/bin/with-contenv bashio` + `exec`.
- **Publish workflow:** like travelmap's (arm64 runner, `GHCR_PAT`, version from `config.yaml`), plus `cargo test --locked` before pushing.
- **DOCS.md:**
  - prerequisites (Liftosaur key, Intervals.icu ID + key, Strava API app with callback domain `localhost`)
  - install (repository + GHCR registry credential)
  - every option
  - connecting Strava
  - the switch-over checklist
  - optional Intervals.icu webhook → HA automation → `rest_command` to `/trigger`
  - troubleshooting

**Done when:** a local `docker build` works and the binary starts in the image, the publish workflow pushes the image, and `GHCR_PAT` is set as a repo secret (`gh secret set GHCR_PAT`).

- [ ] Done, reviewed

### Task 14: Install, dry run, switch over — *you, Claude assists via the HA ssh skill*

1. Add the repo + GHCR credential in HA, install, and configure with `dry_run: true` and `sync_after` = 7 days ago. Connect Strava.
2. Check `/share/liftosaur-sync/` output and the status page; inspect one FIT with `fit_tool info`.
3. **Switch over:**
   - disconnect Garmin → Strava (keep Intervals.icu)
   - set `sync_after` to now and `dry_run: false`, then restart
   - after the next workout: one Strava activity with HR + muscle map, and an Intervals.icu description with the exercises
4. Open the follow-up issue: "Consider an Intervals.icu OAuth app for ACTIVITY_UPLOADED webhooks".
5. Add `liftosaur-sync/` to the workspace `CLAUDE.md` repo table.

- [ ] Live
