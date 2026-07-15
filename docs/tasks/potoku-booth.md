# potoku (booth) — Task Breakdown

> Electron booth app. Read [tasks/README.md](README.md) first.
> Iron rules for every task: **renderer never touches hardware** (all capture/print/persist crosses the typed IPC contract in `app/src/main/ipc/contract.ts`); hardware selection is constructed from booth-local settings (ADR-015); the booth must keep working offline with cached templates and a durable upload queue. A PR importing `fs`/`sharp`/`better-sqlite3` in renderer code gets bounced.
> Test infra: vitest (BOOTH-007) — unit tests for main-process modules run in Node; wizard reducer tests run pure (no Electron needed). Anything needing real Electron is a manual checklist item in the PR, explicitly listed.

---

## Milestone M0 — walking skeleton

### BOOTH-001 · WebcamCamera adapter
Milestone M0 · Size M · Level mid · Depends: —

**Context**: The webcam is M0's capture device AND the permanent failover (ADR-002). Architecture wrinkle: `getUserMedia` lives in the renderer, but hardware ownership lives in main — so this adapter is a *broker*.

**Spec**
- Main: `WebcamCamera implements ICamera` — `capture()` sends an IPC request to a hidden capture path in the renderer, awaits a JPEG (dataURL → Buffer), 3 s timeout. `getHealth()` = device enumerated + last capture ok. `deviceId` + `label` from constructor (settings later; M0: first available).
- Renderer capture module: owns one `<video>` off-DOM + `MediaStream`; on capture request draws to canvas at native resolution → `toDataURL("image/jpeg", 0.92)`. Stream opened once at init, not per shot (shot-to-shot latency budget: < 300 ms after warm-up).
- Extend the IPC contract: main→renderer request channel + response correlation ids (multiple in-flight captures must not cross wires).

**AC**
- [ ] `capture()` returns a real webcam JPEG in dev on macOS.
- [ ] Second capture ≤ 300 ms after first (warm stream).
- [ ] Unplugging the webcam mid-run → `getHealth().ok === false` within 5 s, `capture()` rejects with typed `CameraError("device_lost")`.

**Tests** (vitest, renderer capture module with mocked `mediaDevices`; main adapter with fake IPC)
- `Capture_resolves_jpeg_buffer_with_correct_correlation_id`
- `Concurrent_captures_resolve_to_their_own_callers`
- `Timeout_rejects_with_camera_error_timeout`
- `Device_lost_health_flips_false`
- Manual checklist: real capture on macOS; permission-denied path shows readable error.

**Edge cases**
- OS camera permission denied → typed `CameraError("permission")` — the wizard will show staff guidance, never a blank screen.
- deviceId disappears (USB re-plug, ADR-015) → re-enumerate by `label` match before failing.
- Renderer reload while capture in-flight → main's pending promise must reject (wire `webContents` destroyed handler), not hang forever.

**Out of scope**: live-view streaming to UI (M1 BOOTH-011 pattern; M0 shows the `<video>` directly), settings screen, DSLR.

---

### BOOTH-002 · Composer: sharp rasterizer over template-kit ops
Milestone M0 · Size M · Level mid · Depends: —

**Context**: ADR-004/016 — `@potoku/template-kit.composeOps()` produces draw ops; this task is the sharp interpreter + the M0 hardcoded template. **Also ships `template-config.schema.json` generation** (zod → JSON Schema) that API-011 consumes.

**Spec**
- `app/src/main/composer/sharp-rasterizer.ts`: `rasterize(ops, resolveAsset: (ref) => path|Buffer, canvas) => Promise<Buffer /* PNG */>`. Interpret ops in order: background (resize cover to canvas), photo (extract `s*` crop → resize to `d*` → composite at `dx,dy`; rotation 0 only in M0 — throw typed `UnsupportedOp` otherwise), frame (composite full-canvas, must be last).
- M0 template: one strip 2×6 (1200×3600, 4 slots) as a committed JSON fixture + placeholder frame PNG; validated against the schema in a test (dogfood).
- template-kit: add `npm run schema` emitting `dist/template-config.schema.json` (zod-to-json-schema); wire into build.
- Determinism: same inputs → byte-identical PNG (fix sharp options: no metadata timestamps, explicit compression settings). Golden-file test with committed 100×300 mini-template.

**AC**
- [ ] 4 mock photos + M0 template → correct strip PNG (visual check in PR + golden test).
- [ ] Composition of 4×(3000×2000) into 1200×3600 completes < 2 s on the dev Mac (PRD screen 8 budget).
- [ ] `schema.json` emitted and committed; API repo copy instructions in template-kit README.

**Tests**
- `Ops_are_interpreted_in_order_background_photos_frame` (spy on composite sequence)
- `Golden_mini_template_byte_identical`
- `Photo_count_mismatch_throws_before_any_io` (comes from composeOps — assert it propagates)
- `Missing_asset_ref_throws_typed_AssetMissing`
- `Rotation_nonzero_throws_UnsupportedOp` (M0 honesty)
- `Fixture_template_passes_schema_validation`

**Edge cases**: photo smaller than slot (upscale allowed — cover math handles it, but assert no crash at 100×100 source); frame PNG without alpha (opaque frame would hide photos — validate frame has alpha channel, warn log); non-sRGB JPEG input (force pipeline `.toColourspace('srgb')`).

**Out of scope**: canvas rasterizer (WEB-011), variants UI, print settings.

---

### BOOTH-003 · Wizard state machine (typed reducer)
Milestone M0 · Size M · Level junior · Depends: —

**Context**: The wizard is a pure state machine (ARCHITECTURE §4) — boothlev's god-file is structurally banned. M0 flow: `attract → capture → select → qr → thankyou → attract`.

**Spec**
- `renderer/src/machine/`: `WizardState` discriminated union (one variant per screen, each carrying exactly the data that screen needs — no grab-bag context object); `wizardReducer(state, event)`; events: `START`, `SHOT_TAKEN(photo)`, `CAPTURE_DONE`, `TOGGLE_SELECT(shotId)`, `CONFIRM_SELECTION`, `COMPOSED(result)`, `DELIVERY_READY(link)`, `FINISH`, `RESET`, `ERROR(kind)`.
- Illegal event in a state → **no-op + dev-mode console.warn** (booth must never crash from a stray event; document this policy in the file header).
- Selection rule (ADR-014): `CONFIRM_SELECTION` only fires when `selected.length === template.slots.length` — reducer enforces, UI disables.
- Every screen is a component receiving `(state, dispatch)` — zero business logic in components.

**AC**: full happy path drivable purely by dispatching events in a test, no DOM.

**Tests** (pure, no React)
- `Happy_path_attract_to_thankyou` (scripted event sequence, assert each state)
- `Illegal_event_is_noop` (parameterized: every event against every wrong state)
- `Confirm_blocked_until_exact_slot_count`, `Toggle_beyond_slot_count_replaces_oldest_or_noop` (decide: **no-op + UI hint**; test it)
- `Reset_from_any_state_returns_to_attract`
- `Error_event_carries_kind_and_prior_screen` (for the M1 error screens)

**Edge cases**: double-dispatch of `COMPOSED` (async race) → second is no-op; `RESET` mid-compose → compose result arriving later must be discarded (stale-token pattern in the effect layer — test the token, not just the reducer).

---

### BOOTH-004 · Durable upload queue (SQLite)
Milestone M0 · Size L · Level mid · Depends: API-004 merged

**Context**: PRD F3.1 — uploads survive bad Wi-Fi, app restarts, and machine reboots. This is the most safety-critical booth module: **a captured photo, once enqueued, must eventually reach the API or scream**.

**Spec**
- `main/sync/upload-queue.ts` on `better-sqlite3` (WAL mode), table `upload_jobs(id, sessionId, filePath, kind, checksum, shotIndex, state pending|inflight|done|dead, attempts, nextAttemptAt, lastError, createdAt)`.
- Enqueue = write file to spool dir (`userData/spool/{sessionId}/`) + insert row **in that order**; both fsync'd before `enqueue()` resolves.
- Worker loop: single-flight per queue, oldest-first, exponential backoff `min(2^attempts × 5s, 5 min)` + full jitter; POST to API-004 with stored checksum; 200/replay-200 → `done` + delete spooled file; 4xx (non-retryable: 415/422/413) → `dead` + staff-alert event; network/5xx → retry forever (never `dead` on transient).
- Restart recovery: on boot, `inflight` rows → `pending` (crash mid-upload; API idempotency makes replay safe).
- Observability: queue depth + oldest-age exposed via IPC for the (future) settings/health screen; both logged every 60 s when non-empty.

**AC**
- [ ] Kill API → captures keep enqueueing; restart API → drains fully, zero manual action.
- [ ] Kill the booth app mid-upload → relaunch → item retried and completes exactly once server-side (checksum idempotency proven end-to-end).
- [ ] 422 poison item goes `dead` without blocking the rest of the queue.

**Tests** (fake API server in-process: programmable to fail N times, then succeed)
- `Enqueue_then_drain_happy_path`
- `Backoff_schedule_follows_exponential_with_jitter_bounds`
- `Restart_recovers_inflight_to_pending` (new queue instance over same DB file)
- `Non_retryable_4xx_marks_dead_and_continues_queue`
- `Transient_5xx_retries_indefinitely` (assert attempts grows, never dead)
- `Spool_file_deleted_only_after_done`
- `Enqueue_is_atomic_no_row_without_file_no_file_without_row` (fault injection between the two steps)

**Edge cases**
- Disk full on spool write → enqueue rejects with typed error → wizard shows staff alert (photo is still in memory — offer one retry before losing it; document the UX contract for BOOTH-014).
- Clock jump backwards (NTP) → `nextAttemptAt` comparison must use monotonic-ish guard (if `nextAttemptAt` > now + max backoff, clamp).
- Same file enqueued twice (double `SHOT_TAKEN` bug upstream) → unique index on `(sessionId, checksum)`, second insert no-ops.

**Out of scope**: template cache (BOOTH-013 shares the DB file but not this table), event batching (BOOTH-020).

---

### BOOTH-005 · Generated API client + M0 session bootstrap
Milestone M0 · Size S · Level junior · Depends: API-008 merged

**Spec**: consume `potoku-api/clients/typescript` (copy-vendored with version stamp + `npm run sync-client` script until publishing is decided — the script and a README note ARE part of this task); thin wrapper in `main/api/` adding: base URL from config file (`userData/booth-config.json`, hand-edited in M0 — the settings screen replaces this in BOOTH-018), timeouts (10 s), typed error mapping (network vs 4xx vs 5xx). M0 dev flow: on wizard `START`, call create-session then activate (no codes yet), thread `{ sessionId, gallery }` into the machine.

**AC**: full M0 loop against local compose API works; wrapper never throws raw fetch errors into the wizard (typed `ApiError` only).

**Tests**: `Wrapper_maps_network_error`, `Wrapper_maps_409_domain_conflict`, `Timeout_aborts_at_10s` (fake timers).

**Edge cases**: API unreachable on `START` → wizard error state with retry — the attract screen must never dead-end (this is the demo failure mode on bad Wi-Fi).

---

### BOOTH-006 · QR / delivery screen
Milestone M0 · Size S · Level junior · Depends: BOOTH-003, BOOTH-005

**Spec**: screen 9 (PRD F2): big QR (`qrcode` lib, SVG render) of `{galleryBaseUrl}/g/{shortCode}` (base URL from booth config — NOT hardcoded, whitelabel), short URL text, PIN display, "photos on the way" note (uploads may still be draining — truthful, ADR-010). Auto-advance to thankyou after 60 s or FINISH tap.

**Tests**: `Qr_encodes_exact_url` (decode in test), `Pin_rendered_when_present_hidden_when_disabled`, `Auto_advance_timer_fires_finish`.

**Edge cases**: shortCode with confusable glyphs — display in groups of 5 with Crockford font hints; QR must stay scannable at 1.5 m (min module size — manual checklist with a real phone).

---

### BOOTH-007 · Test + CI infra
Milestone M0 · Size S · Level junior · Depends: —

**Spec**: vitest workspace config (main-process tests in Node env, renderer tests in jsdom); `npm test` at root runs template-kit + app suites; GitHub Actions: lint (add eslint flat config with the no-hardware-in-renderer import ban as a rule: `no-restricted-imports` of `sharp`, `better-sqlite3`, `node:fs` under `renderer/`), typecheck, test, electron-vite build on ubuntu; windows job builds only (no packaging yet — that's BOOTH-025).

**AC**: red PR on: failing test, renderer importing `node:fs` (prove with a throwaway commit in the PR).

---

## Milestone M1 — real booth

### BOOTH-010 · DigiCamControlCamera adapter
Milestone M1 · Size L · Level mid · Depends: BOOTH-001 (ICamera patterns), a Windows machine + dCC + one DSLR

**Context**: ADR-002. dCC webserver on `localhost:5513`. **This task validates the project's top technical risk — timebox the live-view spike to week 1 and report findings before polishing.**

**Spec**
- `capture()`: `GET /?slc=capture` → then resolve the exact file: watch the configured session folder (chokidar) AND poll `list camera1.lastcaptured`; whichever confirms first wins; 4 s timeout → typed `CameraError("capture_timeout")` (supervisor policy consumes this, BOOTH-012).
- `initialize()`: verify dCC reachable, ≥1 camera in `list cameras`, apply per-store camera profile (ISO/aperture/shutter from settings) via `set` commands; verify session folder writable.
- `getHealth()`: periodic `list cameras` (10 s cache); empty → unhealthy.
- All dCC quirks live HERE — nothing dCC-shaped leaks past `ICamera`.
- Deliverable includes `FakeDccServer` (in-repo tiny HTTP server mimicking the endpoints) — all unit tests run against it; real-hardware results go in `docs/hardware-notes.md` (start of the certification matrix, ADR-015).

**AC**
- [ ] Real DSLR: 10 consecutive captures, zero missed files, each resolved < 4 s (manual checklist w/ evidence in PR).
- [ ] dCC not running → `initialize()` fails with actionable message (settings screen shows it verbatim).
- [ ] Live-view spike findings documented: measured fps, latency, subjective verdict vs the ~15fps expectation (ADR-002 escape-hatch decision input).

**Tests** (vs FakeDccServer)
- `Capture_resolves_via_folder_watch_first`
- `Capture_resolves_via_poll_when_watch_misses` (fake writes file without fs event — simulate by pre-writing)
- `Capture_timeout_after_4s_typed_error`
- `Init_applies_camera_profile_set_commands_in_order`
- `Health_empty_camera_list_unhealthy`
- `Concurrent_capture_calls_serialized` (dCC is single-camera — queue them, never interleave)

**Edge cases**
- dCC returns HTTP 200 with an error *body* (it does this) — parse body, don't trust status.
- Two files appear (RAW+JPEG dual-format camera) — take the JPEG, log the config smell, add to hardware-notes.
- Filename collision from camera counter reset → resolve by mtime newest, not name.
- dCC process dies mid-session → next capture fails → supervisor (BOOTH-012) handles; this adapter just reports honestly.

### BOOTH-011 · Live view stream to renderer
Milestone M1 · Size M · Level mid · Depends: BOOTH-010

**Spec**: main polls `GET /liveview.jpg` in a tight loop (target 15fps, adaptive: if fetch takes >66 ms, skip — never queue); frames pushed over a dedicated IPC channel as JPEG buffers; renderer paints onto `<canvas>` (not `<img>` src churn — GC pressure). Drop-not-buffer policy: only the newest frame matters. Stream lifecycle owned by wizard screen mount/unmount (get-ready + capture screens only — don't stream during select).

**Tests**: `Backpressure_drops_frames_never_queues` (slow consumer fake), `Stream_stops_on_unsubscribe` (no orphan polling loop — assert timer cleared), `Fps_adapts_to_slow_fetches`.

**Edge cases**: liveview endpoint 404 while camera "healthy" (dCC quirk when live view not started) → send `startliveview` command once, then resume; renderer devtools paused → main must not balloon memory (drop policy covers it — test with unconsumed channel).

### BOOTH-012 · Camera supervisor: failover + recovery
Milestone M1 · Size L · Level mid · Depends: BOOTH-010, BOOTH-001

**Spec** (ARCHITECTURE §5 policy, verbatim): capture timeout → one retry; second failure or disconnect → mark DSLR unhealthy → swap active camera to `WebcamCamera` **mid-session** (wizard sees a `CAMERA_DEGRADED` event → banner "camera assist mode"), emit `fallback_webcam` session event (BOOTH-020), staff alert via heartbeat payload; background probe every 30 s; DSLR restored **only between sessions** (never mid-capture-sequence). Supervisor implements `ICamera` itself (decorator) — the wizard/capture flow never knows which camera is live.

**Tests** (fake cameras with scripted failures)
- `Single_timeout_retries_once_then_succeeds_no_failover`
- `Two_consecutive_failures_swap_to_webcam_and_emit_events`
- `Restore_waits_for_session_end` (probe healthy mid-session → still webcam; after `RESET` → DSLR)
- `Failover_when_webcam_also_dead_surfaces_fatal_error` (both down → wizard fatal screen "get staff", session events logged)
- `Health_reflects_active_camera`

**Edge cases**: failure during the *countdown* (pre-capture) vs during capture — both must recover into a re-armed countdown, not a lost shot slot; probe succeeding then immediately failing (flapping) → require 2 consecutive healthy probes before eligible-for-restore.

### BOOTH-013 · Template sync + offline cache
Milestone M1 · Size L · Level mid · Depends: API-012, BOOTH-004 (shares SQLite file)

**Spec**: on boot + every 15 min: fetch manifest (API-12); short-circuit on unchanged top hash; else download changed assets (verify sha256 before commit), upsert into `template_cache` tables + asset files under `userData/templates/{templateVersionId}/`; **atomic swap** — a version is visible to the wizard only when config + all assets are verified (staging dir → rename). Offline: serve last-known catalog forever (PRD screen 3: works offline). Expose `getActiveTemplates()` via IPC.

**Tests**: `Unchanged_hash_skips_downloads`, `Checksum_mismatch_discards_and_retries_next_cycle`, `Partial_download_never_visible` (kill fake server mid-asset → catalog unchanged), `Offline_serves_cache`, `Removed_template_disappears_after_sync_but_files_kept_for_referenced_sessions`.

**Edge cases**: disk pressure — cache eviction only for versions not in current manifest AND not referenced by an incomplete local session; first-boot with no network and no cache → wizard shows "no templates yet" staff state, not a crash.

### BOOTH-014 · Capture flow — shooting window (ADR-014)
Milestone M1 · Size L · Level mid · Depends: BOOTH-003, BOOTH-011, BOOTH-012

**Spec**: extend the machine: `capture` state carries `{ windowEndsAt, shots: Shot[], cap }` (window/cap from activation response, API-015); loop = 5-4-3-2-1 countdown (skippable? **no** — rhythm is the product) → `capture()` → 2 s preview → next; exits on window end / cap reached / "I'm done" tap; session-timer (package duration) always visible and independent of window timer; each shot: save to spool + enqueue upload (BOOTH-004) + `capture_done` event *immediately* (crash-safety: a shot taken is a shot saved). Session journal (crash recovery groundwork, full recovery BOOTH-024): append `{step, shotIds}` to `userData/journal/{sessionId}.jsonl` after every transition.

**Tests** (fake timers + fake camera)
- `Window_expiry_mid_countdown_finishes_that_shot_then_exits` (never lose a fired shot)
- `Cap_reached_exits_loop`
- `Im_done_exits_after_current_preview`
- `Each_shot_enqueued_before_next_countdown_starts`
- `Camera_error_during_loop_shows_retry_state_not_crash` (supervisor fatal → staff screen; degraded → banner + continue)
- `Journal_appended_per_shot` 

**Edge cases**: 0 shots taken when window ends (customer froze) → jump to a "no photos — shoot again?" state granting one 60 s grace window (session timer permitting) — PRD's "customer never loses photos" spirit, decided here, document in PRD F2 notes; storage-full enqueue failure mid-loop → pause loop, staff alert, shots kept in memory for one retry.

### BOOTH-015 · Select screen (slots-follow-template)
Milestone M1 · Size M · Level junior · Depends: BOOTH-014, BOOTH-013

**Spec**: film-strip of all shots (spool-file thumbnails, generated at capture time 320px); tap to fill next empty slot, tap a filled slot to clear; live mini-preview of the template with picks placed (cheap: absolutely-positioned `<img>`s over the frame PNG — NOT the rasterizer; note the honest-preview limitation); confirm enabled at exactly `slots.length` picks (machine already enforces); "Shoot more" returns to capture if session time remains AND cap not reached.

**Tests**: `Pick_fills_slots_in_order`, `Unpick_frees_slot`, `Confirm_gate_exact_count`, `Shoot_more_hidden_when_cap_or_time_exhausted`, `Thumbnails_lazy_load` (perf: 30 shots × 320px — assert no full-res loads).

**Edge cases**: 30 shots on a 1080p touch screen — horizontal scroll with momentum, targets ≥64 px (PRD §6); shot file corrupt/missing (spool deleted by mistake) → tile shows retry-thumbnail state, pick disabled for that shot.

### BOOTH-016 · Style/variant + confirm + compose
Milestone M1 · Size M · Level junior · Depends: BOOTH-015, BOOTH-002

**Spec**: variant chips from template (`variants[]`; skip screen entirely when empty); confirm screen shows the *real* composed preview: run the actual sharp rasterizer at 1/3 scale (fast) — the print is what they saw; "Print!" → full-res compose → enqueue composed upload + `POST complete` (API-005) → print (BOOTH-017) → QR screen regardless of print outcome (PRD F2 resilience rule).

**Tests**: `Variant_swap_recomposes_preview`, `No_variants_skips_screen`, `Print_failure_still_reaches_qr_and_emits_print_failed`, `Composed_uploaded_and_session_completed`, `Preview_and_print_use_same_ops` (spy: composeOps called once, rasterized twice at different scales).

**Edge cases**: compose >2 s budget blown on weak hardware → progress animation + measured timing logged (perf data for M4); complete-call fails offline → session completes *locally*, complete retried by queue semantics (add `complete` as a queue job type — small but crucial: **completion must be durable**, extend BOOTH-004 job kinds).

### BOOTH-017 · Printer adapters: hot-folder + spooler
Milestone M1 · Size L · Level mid · Depends: BOOTH-002; real printer for certification

**Spec**: `IPrinter` — `print(pngPath, settings): Promise<PrintJobHandle>`, `getHealth()`. `HotFolderPrinter` (DNP): atomic drop = write to temp name in the watched folder → rename to final (utilities scan on rename); job "done" inferred when the utility consumes (file disappears, poll 1 s, 60 s timeout → `unknown` status not `failed` — honesty per ADR-006); `SpoolerPrinter` (HiTi/others): `pdf-to-printer` (or raw win32 print of PNG wrapped in PDF — investigate, document choice) targeting the settings-selected Windows printer; real status from spooler where available. `print_ok`/`print_failed`/`print_unknown` session events.

**Tests** (temp-dir fake hot folder; fake spooler wrapper)
- `Hotfolder_drop_is_atomic_rename` (watcher sees only final name)
- `Consumed_file_infers_done`
- `Timeout_reports_unknown_not_failed`
- `Spooler_maps_printer_offline_to_typed_error`
- `Print_call_never_blocks_caller_beyond_500ms` (fire-and-track — QR screen must not wait, assert async contract)

**Edge cases**: hot folder on a disconnected mapped drive → health unhealthy at *init*, not first print; duplicate print of same file name (reprint) → unique temp names always; paper-out mid-job is invisible to hot-folder (document: staff alert comes from customer/timeout — ADR-006 accepted trade-off, link it).

### BOOTH-018 · Settings screen (F6, ADR-015)
Milestone M1 · Size L · Level mid · Depends: BOOTH-001, BOOTH-010, BOOTH-017

**Spec**
- Entry: 5 taps within 3 s in top-left 100×100 px on attract → passcode pad (passcode hash in settings file; default set at first-run wizard); locked while session active.
- Panels: **Camera** (unified list: dCC cameras + `enumerateDevices()` webcams; pick primary + fallback; live preview of highlighted device; Test capture button showing result), **Printer** (mode: hot-folder → folder picker | driver → `getPrintersAsync()` list; Test print button — prints the committed test-strip PNG), **Connection** (API URL, device token entry — replaces BOOTH-005's hand-edited file; Test connection = heartbeat roundtrip), **Hardware check** (BOOTH-019 button), **About** (versions, queue depth, cache age).
- Persistence: `userData/settings.json`, zod-validated (schema in `app/src/main/settings/schema.ts`), atomic write (temp+rename), config version field + migration hook; every change → `settings_changed` device event + immediate heartbeat with new snapshot (API-013).
- Selection persists `deviceId` AND `label`; boot resolution: byId → byLabel → configured-kind default → health warning chain (ADR-015).

**Tests**: `Gesture_recognizer_5_taps_3s_zone` (unit), `Passcode_gate_and_lockout_after_5_wrong` (30 s local lockout), `Settings_write_is_atomic` (kill between temp+rename → old file intact), `Invalid_settings_file_falls_back_to_defaults_plus_alert` (corrupt JSON on disk), `Device_reresolution_by_label_when_id_changes`, `Locked_while_session_active`.

**Edge cases**: changing camera while its preview streams → dispose old stream before opening new (device busy errors); token pasted with whitespace → trim; settings opened on first-run (no passcode yet) → forced setup wizard path.

### BOOTH-019 · Hardware check sequence
Milestone M1 · Size M · Level junior · Depends: BOOTH-018, BOOTH-016, BOOTH-017

**Spec**: one button runs: camera init → capture ×3 (timings) → compose test template with the 3 shots → print it → API heartbeat roundtrip; per-step pass/fail/timing UI; result persisted to settings file + sent as device event (`hardware_check`, payload = report) — this feeds the self-service certification matrix (ADR-015).

**Tests**: `Report_marks_first_failing_step_and_skips_dependents`, `Report_persisted_and_event_emitted`, `Timings_recorded_per_step`.

**Edge cases**: print step on booth without printer configured → step "skipped", overall = partial pass (camera-only booths are valid for M0-style installs).

### BOOTH-020 · Session event reporting (batched, offline-safe)
Milestone M1 · Size S · Level junior · Depends: API-014, BOOTH-004

**Spec**: `events` table in the queue DB; every wizard/hardware event appended locally with uuid; batcher flushes ≤50 events every 10 s (or on flush-worthy events: `print_failed`, `fallback_webcam` → immediate) to API-014; dedupe server-side by uuid, so resend-on-doubt.

**Tests**: `Batch_flush_at_50_or_10s`, `Priority_events_flush_immediately`, `Offline_buffers_and_drains`, `Server_duplicate_response_marks_done`.

**Edge cases**: unbounded growth when offline for days → cap table at 10k events, drop-oldest with a `events_dropped` marker event (observability of the loss).

---

## Milestone M2

### BOOTH-021 · Code entry screen vs bookings
Milestone M2 · Size M · Level junior · Depends: API-025, BOOTH-003

**Spec**: screen 2 (PRD F2): big keypad, 6 digits, auto-submit on 6th; switch on API-025 reason codes: `code_unknown` → shake + "check your code"; `outside_window` → friendly "your session starts at {windowStartsAt}" (booth-local tz display); `already_used` / `booking_cancelled` → "see our staff"; network fail → retry affordance (attract never dead-ends). 5 wrong codes in 5 min → 60 s local cooldown (defense-in-depth with API-028).

**Tests**: `Reason_code_to_copy_mapping_complete` (parameterized over the enum — compile break when API adds a code), `Local_cooldown_after_5_failures`, `Autosubmit_on_sixth_digit`, `Backspace_and_clear_work`.

**Edge cases**: staff testing with the booth offline → dev-mode bypass behind settings flag, banner-marked, impossible in kiosk build (compile-time flag).

### BOOTH-022 · First-run pairing wizard
Milestone M2 · Size M · Level mid · Depends: BOOTH-018, API-043

**Spec**: fresh install boot → API URL → one-time pairing code → local 4–8 digit staff passcode. Before claim, persist only a random claim UUID; never persist the human code. Claim returns the permanent token directly to main process, which verifies it with an authenticated heartbeat before atomically saving it and clearing the retry UUID. Typed invalid/expired/used/cancelled/rate-limit/network messages keep the operator in the flow; a lost claim response is safe to retry. Land on settings for Webcam/DSLR/fallback selection (BOOTH-018) and hardware check (BOOTH-019). Re-pairing remains staff-passcode-gated and camera changes retain identity.

**Tests**: `Fresh_boot_enters_wizard_when_no_token`, `Invalid_code_stays_on_step_with_error`, `Lost_claim_response_reuses_claim_id`, `Completion_persists_only_device_token`, `Repair_requires_current_passcode`.

---

## Milestone M3

### BOOTH-023 · Reprint job pickup
Milestone M3 · Size S · Level junior · Depends: API-034, BOOTH-017

**Spec**: poll print-jobs endpoint every 30 s when idle (never mid-session); download asset by signed URL, print via active `IPrinter`, report result; visible in settings About panel (last jobs).

**Tests**: `Polls_only_when_idle`, `Reports_success_and_failure`, `Download_checksum_verified_before_print`.

**Edge cases**: job for a template/media older than cache → asset comes from API not cache (signed URL path) — no cache dependency.

---

## Milestone M4

### BOOTH-024 · Crash recovery (resume at Select)
Milestone M4 · Size L · Level mid · Depends: BOOTH-014 journal

**Spec**: on boot, scan journals: found un-finalized session + session still Active server-side (or offline: within its local window) → resume wizard at Select with journaled shots; else finalize: enqueue whatever exists, emit `session_recovered_media` event, mark journal done. Journal compaction: delete after session completed + uploads drained.

**Tests**: `Resume_at_select_with_shots_after_simulated_crash` (write journal → new app instance over same userData), `Expired_session_finalizes_instead_of_resuming`, `Journal_cleanup_after_drain`.

**Edge cases**: journal references missing spool file (partial crash) → drop that shot, log, resume with the rest; corrupted journal line → skip line, keep parsing (jsonl resilience).

### BOOTH-025 · Kiosk lockdown + packaging + updater
Milestone M4 · Size L · Level mid · Depends: all M1 · **Windows machine required**

**Spec**: kiosk `BrowserWindow` flags (fullscreen, frame:false, alwaysOnTop, closable:false in kiosk build), block accelerators (Alt-F4/Ctrl-W handled; Windows-key lockdown is OS-config per §11 runbook — write `docs/runbooks/booth-setup.md` as part of this task); electron-builder NSIS on GitHub Actions windows-latest (tag-triggered), electron-updater checking on idle only, staff-confirmed from settings, never mid-session; watchdog: relaunch-on-crash (electron `app.relaunch` on uncaught + a scheduled-task watchdog documented in the runbook).

**Tests**: CI produces installable NSIS artifact (job green + manual install checklist); `Update_prompt_never_during_active_session` (unit: updater gate checks wizard state).

**Edge cases**: updater on metered/offline store connection → silent skip, retry next idle; downgrade protection (electron-updater default — verify).

### BOOTH-026 · Abandoned session handling
Milestone M4 · Size S · Level junior · Depends: BOOTH-014, API-014

**Spec**: idle >3 min on any customer screen (PRD F2) → 30 s "still there?" countdown overlay → auto-close: enqueue all media, emit `session_abandoned` + call abandon endpoint (add to API if missing — coordinate), reset to attract. Timer pauses during capture countdowns (activity = taps OR camera activity).

**Tests**: `Idle_prompt_then_reset`, `Any_tap_resets_idle_timer`, `Media_enqueued_on_abandon`, `Capture_activity_counts_as_activity`.

### BOOTH-027 · Offline/chaos verification (booth side)
Milestone M4 · Size M · Level mid · Depends: BOOTH-004, BOOTH-013, BOOTH-020, API-042

**Spec**: scripted scenario suite (can be semi-manual with a network-toggle helper): full session with Wi-Fi killed at each of 6 designated moments (activation done→capture, mid-capture, pre-compose, pre-complete, QR screen, post-session drain) — assert per PRD M4: nothing lost, QR always shown, queue drains on reconnect. Findings doc + bugs become tasks.

**AC**: all 6 scenarios pass on a real Windows booth build; report committed to `docs/chaos-report-m4.md`.

---

## Milestone M6 — GIF/boomerang

> Booth has no M5 tasks (online booking is API+web only — the booth already honors bookings via API-025). M6 starts only after M4's chaos suite (BOOTH-027) is green.

### BOOTH-030 · Burst capture for animated shots
Milestone M6 · Size L · Level mid · Depends: BOOTH-011, BOOTH-014; cross-repo: API-062 (contract)

**Context**: PRD "Later" promoted to M6. §12: GIF = new media kind + composer step + gallery tile. Capture source is the **live-view stream** (DSLR still-burst via dCC is ~1 fps — too slow); frames come from the same pipe BOOTH-011 built, so this works identically on DSLR live view and webcam fallback.

**Spec**
- Template config gains `animation?: { kind: "gif"|"boomerang", frames: number, fps: number }` — template-kit zod schema bump; the regenerated `template-config.schema.json` flows to the API contracts copy (API-011 sync pattern, coordinate in PR).
- Wizard: when the selected template declares `animation`, the capture loop adds an animated-shot step: countdown → grab `frames` consecutive live-view frames at target `fps` (drop-not-buffer policy stays) → loop preview → keep/retake.
- Frameset persisted to spool as a directory + manifest (frame order, timings); journal appended after persist — same crash-safety bar as BOOTH-014.
- Boomerang is an *assembly* concern (forward + reversed at encode time, BOOTH-031) — never capture twice.

**AC**
- [ ] Animated template produces a smooth ≥12 fps loop preview on the booth.
- [ ] Still-only templates unaffected (regression: M1 capture flow untouched).
- [ ] Webcam fallback captures animation too (frames via the BOOTH-001 broker path).

**Tests**: `Animation_declared_template_enters_burst_step`, `Frame_count_and_fps_within_tolerance` (fake stream), `Retake_discards_previous_frameset`, `Journal_appended_after_frameset_persisted`, `Still_template_flow_unchanged`.

**Edge cases**: live-view fps below target (slow dCC) → extend wall-clock to reach frame count, hard cap 5 s, warn log; memory bounded (frames spill to spool as they arrive, never all-in-RAM); crash mid-frameset → BOOTH-024 recovery drops the incomplete frameset, keeps stills.

**Out of scope**: encoding (BOOTH-031), gallery tile (WEB-060), printing animated output (never — print is stills only).

---

### BOOTH-031 · Animated asset assembly + upload
Milestone M6 · Size M · Level mid · Depends: BOOTH-030, API-062 merged

**Spec**
- Assembler in `main/composer/`: frameset → animated output. **Investigate & decide in-task** (document in PR): sharp animated WebP/GIF vs ffmpeg-static MP4. Decision criteria: file size at ~3 s / 480p, iOS Safari playback (WEB-060 renders `<video muted autoplay playsinline>` for mp4, `<img>` for gif/webp), encode time < 3 s on booth hardware. The cross-repo contract is `MediaAsset.Kind = Animated` + honest content type (API-062 accepts gif/webp/mp4) — not the container.
- Boomerang: forward + reversed sequence, duplicate endpoint frames dropped.
- Output enqueued on the BOOTH-004 queue (new job kind, same idempotency/checksum semantics); QR/gallery copy already says "photos on the way" — no UI change here.

**Tests**: `Boomerang_mirrors_without_duplicate_endpoints`, `Encode_within_size_budget` (fixture frameset → ≤ configured MB), `Enqueue_flows_through_existing_queue_semantics`, `Assembly_failure_marks_shot_failed_not_session_crash`.

**Edge cases**: encode failure (codec missing/bad frameset) → session continues stills-only, `animated_failed` event + staff alert — an animation must never block the print or the QR; force sRGB on frames (BOOTH-002 rule).

---

## Milestone M9 — Potoku Box: self-service mode

> ADR-018: same app, second product. The Box is this booth on a small Windows touch device, webcam camera, thermal printer, no staff. Iron rules unchanged: renderer never touches hardware; business config server-owned.

### BOOTH-040 · Operating mode: self-service wizard branch
Milestone M9 · Size M · Level mid · Depends: BOOTH-021, API-090

**Spec**: heartbeat delivers `operatingMode` + pricing snapshot (API-090). `SelfService` swaps the wizard entry: attract (price shown) → **package & pay** (BOOTH-041) instead of code entry; everything from template-pick onward is the existing flow untouched. Mode changes apply **only at attract** — an active session always finishes under the mode it started with. Staffed devices render exactly as today (zero regression).

**Tests**: `Selfservice_attract_shows_price_and_enters_pay`, `Staffed_flow_unchanged`, `Mode_flip_mid_session_applies_at_next_attract`, `Missing_pricing_snapshot_keeps_box_in_attract_with_operator_alert`.

**Edge cases**: heartbeat delivers `NotInService` (API-101) → "not in service" screen from attract only; dev-mode bypass (BOOTH-021) must NOT exist in self-service — pay is the only door (compile-time assert like the kiosk flag).

### BOOTH-041 · Package & pay screen (QRIS)
Milestone M9 · Size L · Level mid · Depends: BOOTH-040, API-092

**Spec**: package pick (or straight to pay when one package) → `POST /api/box/purchases` via main process → render QR (client-side QR encode of `qrString`), amount big, expiry countdown second-by-second → poll purchase status (main process, jittered interval) → paid → celebration → session begins with the API-returned session. Typed states: `gateway_unavailable` (retry button), expired (new QR button), network loss mid-poll (auto-resume polling — payment may have landed), cancel/back to attract voids politely. The QR string never persists to disk; no payment state survives in the renderer beyond the screen.

**Tests**: `Pay_screen_renders_qr_amount_countdown`, `Paid_poll_advances_to_session`, `Expired_offers_new_qr`, `Network_loss_resumes_polling_and_finds_paid`, `Back_to_attract_voids_purchase`, `Qr_string_never_persisted`.

**Edge cases**: customer pays at the last countdown second → poll continues a grace window past expiry before declaring expired (API-092 honors late webhooks — the box must too); double-tap on package → single purchase (in-flight guard).

### BOOTH-042 · Thermal printer adapter + mono strip render
Milestone M9 · Size L · Level senior · Depends: BOOTH-015 (IPrinter seam), API-092

**Spec**: ADR-020. New `ThermalPrinter : IPrinter` — ESC/POS raster (`GS v 0`) over TCP:9100 and USB (escpos-usb or raw write, decide in-task, document). New thermal render target in main (sharp): 576px-wide mono raster — brand header (from brand config, ADR-012), dithered (Floyd–Steinberg) photo stack from the *selected* shots, gallery QR + PIN, footer copy. The full-color composed strip still renders/uploads unchanged — thermal is an additional output, not a replacement. Settings screen gains thermal printer setup (interface picker, test print); hardware check (BOOTH-019) covers it.

**Tests**: `Raster_is_576px_mono_with_qr` (golden fixture), `Dither_output_deterministic`, `Print_failure_never_blocks_qr_screen`, `Test_print_from_settings`, `Color_composed_output_unchanged` (byte-compare against existing golden).

**Edge cases**: paper-out mid-print (where detectable via status callback) → operator alert + reprint offer, QR screen unaffected; 58mm rolls (384px) behind config — certify 80mm first, don't hardcode 576.

### BOOTH-043 · Paid-session recovery
Milestone M9 · Size S · Level mid · Depends: BOOTH-041, API-110 contract stub (recovery endpoint may land as API-092's paid-poll until M11)

**Spec**: on boot in self-service mode, ask the API for a paid-unconsumed purchase (`/api/box/purchases/current`); if found → "Continue your session" screen (big, friendly, 60s timeout → apology + operator contact from brand config, session flagged for refund server-side). Never re-charge.

**Tests**: `Boot_with_paid_unconsumed_offers_continue`, `Continue_enters_session_with_original_package`, `Timeout_shows_apology_and_flags`, `Clean_boot_goes_to_attract`.

**Edge cases**: recovery offer races a new customer touching attract → recovery screen wins on boot, only on boot.

---

## Milestone M11 — box field-hardening (booth side)

### BOOTH-050 · Box kiosk profile + connectivity states
Milestone M11 · Size M · Level mid · Depends: BOOTH-025, BOOTH-040, API-101

**Spec**: box flavor of the kiosk build: Windows auto-login + app watchdog (restart on crash, log the episode), "Be right back" screen when offline in self-service (payment needs the network; in-flight paid sessions continue on offline queues), `NotInService` screen (subscription, API-101) with operator-facing note in settings. Provisioning doc: image checklist from OS install → enrolled box (pairs with the runbook in photomob docs).

**Tests**: `Offline_selfservice_shows_be_right_back_from_attract_only`, `Inflight_session_survives_offline`, `Notinservice_applies_from_attract_only`, `Watchdog_restart_lands_attract_or_recovery`.

**Edge cases**: offline AND paid-unconsumed on boot → recovery screen queues until connectivity returns (never lose the customer's money to a dead Wi-Fi moment).

### BOOTH-051 · Thermal certification + supported-hardware matrix
Milestone M11 · Size M · Level junior · Depends: BOOTH-042

**Spec**: certify ≥2 physical units (Epson TM-T82 class + one Xprinter-class budget unit): print quality at our dither, speed, paper-out behavior, USB vs TCP reliability, cutter. Output = supported-thermal matrix in `docs/hardware-notes.md` + any adapter fixes. Same pattern as the M1 DSLR/dye-sub gates: software done ≠ certified.

**Tests**: physical checklist (documented, evidence photos in the PR); regression tests for any adapter fix.

**Edge cases**: unit needs a vendor-specific raster quirk → quirk table in config, never `if (model ===` in the adapter body.

---

*Not in any milestone: EDSDK adapter (only if BOOTH-010's spike verdict demands it — that's a new ADR first), visual template designer (WEB-080..081 — the designer emits ordinary template config; the booth consumes it unchanged), Android/PWA box shell (ADR-018 evolution — becomes real only if box hardware cost blocks sales), dye-sub photo tier for the Box (ADR-020 — config + certification, no new architecture).*
