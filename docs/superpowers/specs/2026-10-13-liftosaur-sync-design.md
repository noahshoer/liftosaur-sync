# liftosaur-sync: Design

Date: 2026-10-13
Status: Approved

## Goal

Every activity from my Garmin watch shows up on Strava exactly once. Strength
workouts carry the exercises I logged in Liftosaur, so Strava draws the
correct muscle map, while keeping the watch's heart rate and timing. It runs
unattended as a Home Assistant app on my Raspberry Pi 4.

### Success criteria

- One Strava activity per Garmin activity. No hidden duplicates, no manual cleanup.
- Strength activities on Strava show the Liftosaur exercises and the full watch heart-rate data.
- Non-strength activities (runs, rides, …) reach Strava unchanged.
- Activities appear on Strava within a few minutes of reaching Intervals.icu.
- If a strength workout has no Liftosaur log, I get an HA notification, and the activity still reaches Strava eventually.
- On Intervals.icu, a matched strength activity's description lists each exercise's sets, reps and weight, close to how Liftosaur writes the workout.

### Non-goals (v1)

- Changing anything on Intervals.icu other than the description of matched strength activities.
- Listing muscle groups anywhere (Strava's muscle map comes from the FIT exercises).
- Changing the Strava activity's description.
- Updating the activity in Garmin Connect. There's no public API for it, only the private mobile-app API (impersonating the Android app's login), so Garmin keeps the watch's own set detection.
- Uploading through Strava's JSON format (no Liftosaur → Strava exercise mapping).
- Backfilling activities from before the switch-over.
- Supporting more than one athlete.

## Background and constraints

These findings shaped the design.

- **Liftosaur API** (`https://www.liftosaur.com/api/v1`, Bearer `lftsk_…`, premium only). History can be filtered by `startDate`/`endDate`. There are no webhooks and no "updated since" filter. History records are Liftohistory text: one line per exercise with completed sets, warm-ups and targets (e.g. `Bench Press, Barbell / 3x8 185lb @7, 1x6 185lb @9 / warmup: 1x10 95lb / target: 3x8-12 185lb @8 90s`), plus the workout's start time and duration. They contain **no per-set timestamps**.
- **Strava API.** Exercise and set data only enters Strava through an **upload** (FIT, or the JSON weight-training format), and every upload creates a new activity. `UpdatableActivity` (name, description, sport_type, hide_from_home, commute, trainer, gear_id) can't add exercises. The public API can't delete activities. Uploads that duplicate an existing activity are rejected (`duplicate of activity <id>`).
- **Garmin → Strava** pushes the watch activity as soon as it syncs. Edits made later in Garmin Connect (tested: changing exercise sets) are **not** sent on to Strava.
- **Therefore:** Garmin → Strava gets **disconnected**, and this app becomes the only thing uploading to Strava. Garmin → Intervals.icu stays connected.
- **Intervals.icu API.** `GET /api/v1/activity/{id}/file` returns the original uploaded file (gzipped FIT). A personal API key is enough for reading and for updating an activity's description (`PUT /api/v1/activity/{id}`). Webhooks (`ACTIVITY_UPLOADED`) need a registered OAuth app, and app creation isn't self-service. Webhooks are plain POSTs with a shared `secret` and no confirmation step, so a plain HA webhook automation can receive them.
- **Public repo:** the GitHub repo is public (which also gives free arm64 CI runners). No credentials ever go in the repo: every API key, ID and secret is entered in the HA app's Configuration tab during setup.
- **Runtime:** HA OS on a Raspberry Pi 4 with Nabu Casa. The image is built in CI and published to GHCR, like `travelmap`. Nothing is compiled on the Pi.

## Architecture

```
Garmin watch ──► Garmin Connect ──► Intervals.icu ◄──── poll every 2–3 min ────┐
                                         │  (later: ACTIVITY_UPLOADED webhook   │
                                         │   → HA webhook automation → /trigger)│
                                         ▼                                      │
                         ┌──────────── liftosaur-sync (HA app, Rust) ───────────┘
                         │  1. list new Garmin-sourced activities
                         │  2. download original FIT
                         │  3. strength? ── no ──► upload FIT unchanged ──► Strava
                         │        │ yes
                         │  4. match Liftosaur workout by time overlap ◄── Liftosaur API
                         │  5. rewrite sets in FIT ──► validate ──► upload ──► Strava
                         │  6. write the sets into the description ──► Intervals.icu
                         │  7. record result in /data/state.json
                         └── HA notifications via Supervisor API
```

### Pieces

- **The app.** A single Rust binary in an arm64 image on GHCR, installed from an HA app repository (`repository.yaml` at the repo root, app in `liftosaur-sync/`, same layout as `travelmap`).
- **Ingress web page** (inside HA):
  - "Connect Strava": the one-time OAuth authorization (scopes `activity:write,activity:read_all`). You paste the redirect URL or the code (`redirect_uri=http://localhost/exchange_token`), so no public callback is needed.
  - Status: recent activities and their status, pending items, unmapped exercises, last errors.
- **`/trigger` endpoint.** Starts a poll right away. It lets the Intervals.icu webhook be added later without code changes.
- **Optional later:** an HA webhook automation (via Nabu Casa) that receives Intervals.icu webhooks and calls `/trigger`.

### Flow for each activity

1. **Find new activities.** List Intervals.icu activities newer than `sync_after` that came from Garmin and aren't yet in state.
2. **Download** the original FIT.
3. **Non-strength activities:** upload the FIT to Strava unchanged and record the result.
4. **Strength activities** (FIT sport `training` with sub-sport `strength_training`, or Intervals.icu type `WeightTraining`): look for the Liftosaur workout whose time overlaps the activity.
   - **Match found:** rewrite the FIT (see below), validate it, upload it to Strava, update the Intervals.icu description (see below), record the result. Strava and Intervals.icu are tracked separately in state, so a failure on one doesn't redo the other.
   - **No match:** mark it `pending` and retry on each poll.
     - At `notify_after_minutes` (default 30): send one HA notification, status becomes `notified`.
     - At `give_up_after_minutes` (default 180): upload the original FIT unchanged.
5. **Strava duplicate error:** record the activity as `uploaded` with the existing Strava activity ID. Never retry it.

### Matching

- **Overlap rule:** a Liftosaur workout matches when its time range overlaps the watch activity by ≥ 50% of the shorter of the two, and their start times are within 30 minutes of each other.
- **Several candidates:** the one with the largest overlap wins.
- **One match per workout:** a Liftosaur workout can match at most one activity, tracked in state.

## Exercise mapping

- **Table:** a data file in the repo, compiled into the binary. It maps a **Liftosaur exercise plus equipment** (e.g. `Bench Press, Barbell`) to a **Garmin FIT `exercise_category` plus exercise name**.
- **How it's built:** from Liftosaur's exercise list and the Garmin FIT SDK's exercise list.
  - liftosaur2garmin (`github.com/Zettt/liftosaur2garmin`) is a **reference only**, for checking our entries. No code or tables are copied.
- **Overrides:** the `custom_exercises` option adds entries or overrides built-in ones, for Liftosaur custom exercises or corrections, without a release.
- **Unmapped exercises:** written with FIT category `unknown` (65534) and no exercise. Listed on the status page.

## Intervals.icu description

Written only for strength activities matched to a Liftosaur workout. Example:

```
Bench Press, Barbell / 3x8 185lb @7, 1x6 185lb @9 / warmup: 1x10 95lb
Overhead Press, Barbell / 3x10 95lb
Pull Up / 3x8|7 0lb
Wall Sit / 2x1 0kg 60s, 1x1 0kg 52s
```

- **Exercise lines:** copied from the Liftohistory line as-is (same names, units, RPE, AMRAP `+`, left|right reps, hold times), with the `target:` section removed. The `warmup:` section is removed when `include_warmups` is false. Exercise notes (`//` lines) are kept.
- **Keeping existing text:** the block goes between `[liftosaur-sync]` and `[/liftosaur-sync]` marker lines. Any text outside the markers, e.g. a note added in Intervals.icu, is preserved. A re-sync replaces only the marked block.
- **Not matched / timed out:** the description is left untouched.

## FIT rewrite

- **Kept from the watch FIT:** file ID, device info, records (heart rate and timestamps), events, laps, session, activity messages.
- **Replaced:** the watch's detected `set` messages are replaced with sets from the Liftosaur workout. Any `exercise_title` messages are removed and none are written.
  - Each set: exercise category and name, reps, weight, active or rest.
  - Warm-up sets are included or dropped based on the `include_warmups` option.
- **Set timing:** Liftosaur doesn't record when each set happened.
  - **Watch detected the same number of active sets:** use the watch's set start times and durations, and only replace exercise, reps and weight.
  - **Counts differ:** spread the Liftosaur sets evenly across the activity, with rest sets in between.
- **Validation:** the rewritten file is decoded again and checked:
  - record count and session totals are unchanged
  - the sets match the Liftosaur workout
  - the CRC is valid

  If anything fails, the original FIT is uploaded and the error is recorded.
- **Library:** `rustyfit`, pinned to an exact version and audited (see Dependency policy).

## Configuration (HA app options)

| Option | Default | Notes |
|---|---|---|
| `liftosaur_api_key` | — | secret |
| `intervals_athlete_id` | — | |
| `intervals_api_key` | — | secret |
| `strava_client_id` | — | |
| `strava_client_secret` | — | secret |
| `poll_interval_seconds` | 150 | |
| `sync_after` | — | ISO date. Nothing earlier is ever uploaded |
| `notify_service` | — | e.g. `notify.mobile_app_pixel` |
| `notify_after_minutes` | 30 | |
| `give_up_after_minutes` | 180 | |
| `include_warmups` | true | |
| `custom_exercises` | `[]` | list of `{liftosaur, equipment, fit_category, fit_name}` |
| `webhook_secret` | — | optional. Required on `/trigger` when set |
| `dry_run` | true | write the FITs and descriptions to `/share/liftosaur-sync/` instead of uploading or updating |

## State

- **File:** `/data/state.json`, written to a temporary file and renamed into place.
- **Contents:**
  - Strava tokens. The refresh token rotates and is persisted on every refresh.
  - For each Intervals.icu activity: status (`uploaded` / `pending` / `notified` / `failed` / `skipped`), matched Liftosaur workout ID, Strava activity ID, whether the Intervals.icu description was written, first-seen time, last error.
- **Pruning:** entries older than 60 days are removed. `sync_after` stops pruned activities from being picked up again.

## Error handling

- **Temporary failures** (network, 5xx, 429): retried on later polls, waiting longer after each failure (up to 1 h). Strava's rate-limit headers are respected.
- **Permanent failures** (401/403/422, FIT rejected): marked `failed`, one HA notification, shown on the status page.
- **Strava authorization revoked or refresh fails:** all uploads pause. One notification, and the status page shows "Reconnect Strava".
- **Liftosaur unavailable:** strength activities stay `pending`. The timeout still applies, so they're eventually uploaded unchanged.
- **`/trigger`:** reachable only on HA's internal network. If `webhook_secret` is set, the secret is required.

## Dependency policy

- `Cargo.lock` is committed. CI runs `cargo-deny` (licenses, advisories, bans, duplicates) and `cargo-audit`.
- **`rustyfit`:**
  - Pinned with `=X.Y.Z` and excluded from automated update tools.
  - Audited **before** it's adopted. The audit covers:
    - every `unsafe` block, build scripts and proc macros
    - its dependency tree
    - any network or filesystem access
    - maintainer and activity
    - a `cargo-fuzz` run of the decoder on real and corrupted FITs
  - Results are recorded in `docs/audits/rustyfit-X.Y.Z.md`. Any version bump requires reviewing the changes since the audited version.

## Testing

- **Unit tests:**
  - matching
  - the pending → notified → give-up timer (with an injected clock)
  - mapping lookup with overrides
  - Liftohistory parsing
  - description rendering, including replacing the marked block while keeping text outside it
- **FIT tests:** synthetic FITs built in the tests (these run in CI), plus real watch FITs kept in gitignored `tests/data/private/` and run when present. Each test rewrites, decodes again, and checks:
  - heart-rate records and session totals are unchanged
  - the sets match the expected Liftosaur workout
- **Sync engine tests** against in-memory fakes of the Intervals.icu, Strava and Liftosaur clients (each real client is tested separately against a local fake HTTP server):
  - run passes through unchanged
  - strength workout with a match: Strava upload plus Intervals.icu description update
  - strength workout with no match → notification → upload unchanged
  - Strava duplicate
  - Strava token refresh and revocation
- **Switch-over checklist:**
  1. Run with `dry_run: true` and inspect the rewritten FITs and descriptions.
  2. Upload one rewritten FIT to Strava manually and confirm the muscle map.
  3. Disconnect Garmin → Strava.
  4. Set `sync_after` to now and `dry_run: false`.

## Roles

I write the code myself, to stay sharp. Claude plans, audits the external crates, answers questions about external APIs and reviews each task. Claude doesn't implement unless asked.

## Build order

0. Create the public GitHub repo and push.
1. Audit `rustyfit` and every other direct dependency (Claude), then pin the versions.
2. **Spike:** rewrite one of my watch FITs, upload it to Strava by hand, and confirm the muscle map and heart rate. If Strava doesn't build a muscle map from FIT sets, stop and revisit the design (fallback: JSON upload with a heart-rate stream).
3. Build the exercise mapping table.
4. Build the core sync: Intervals.icu → match → rewrite → Strava, with state.
5. Add the Intervals.icu description.
6. Add notifications, the ingress status page and Strava OAuth.
7. Package as an HA app: Dockerfile, `config.yaml`, GHCR workflow.
8. Do the switch-over.

## Follow-ups

- Once the GitHub repo exists, open an issue: "Consider an Intervals.icu OAuth app for `ACTIVITY_UPLOADED` webhooks" (polling covers v1; `/trigger` is already in place).
