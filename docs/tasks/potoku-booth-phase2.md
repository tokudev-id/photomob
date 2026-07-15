# Potoku Booth — Phase 2 tasks (Box pivot, M9 + M11)

> Split from [potoku-booth.md](potoku-booth.md) (MVP, M0–M6). Conventions, global DoD, and the cross-repo dependency table live in the [README](README.md). Design source: [ADR-018..020](../DECISIONS.md), [PRD F7](../PRD.md). Iron rules unchanged: renderer never touches hardware; business config server-owned.

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
Milestone M9 · Size S · Level mid · Depends: BOOTH-041, API-092 (`/api/box/purchases/current` lives there; API-110 only hardens the failure tail)

**Spec**: on boot in self-service mode, ask the API for a paid-unconsumed purchase (`/api/box/purchases/current`); if found → "Continue your session" screen (big, friendly, 60s timeout → apology + operator contact from brand config, and a `recovery_timeout` session event — API-110 (M11) escalates these to `RefundFlagged`; in M9 they're operator-visible via session events). Never re-charge.

**Tests**: `Boot_with_paid_unconsumed_offers_continue`, `Continue_enters_session_with_original_package`, `Timeout_shows_apology_and_flags`, `Clean_boot_goes_to_attract`.

**Edge cases**: recovery offer races a new customer touching attract → recovery screen wins on boot, only on boot.

### BOOTH-044 · Self-service user journey, copy & idle behavior
Milestone M9 · Size M · Level mid · Depends: BOOTH-040..043

**Context**: F2's journey assumes staff within shouting distance; F7 has nobody. Every screen the customer sees in self-service mode must survive confusion, walk-aways, and "who do I ask?" alone. This task owns the *journey*, BOOTH-041/042 own the mechanics.

**Spec**
- **Attract**: price + a 3-step "how it works" strip (pay → pose → print), ID/EN toggle, sample strips. No "ask our staff" anywhere.
- **Copy audit**: every staff-referencing string in the wizard gets a self-service variant (i18n key level, not `if (mode)` in JSX). Errors that previously said "call staff" now show the operator contact + a "get help" QR from brand config.
- **Paid-customer abandonment**: the customer PAID — walking away must still deliver value. Idle timeout at Capture/Select/Style → auto-advance: auto-select the first `slotCount` shots (pick order), compose, print the thermal strip, and hold the QR screen for a long dwell (config, ~2 min) before returning to attract. Media always uploads; the gallery link is issued regardless (F2 rule). Session events record `auto_completed`.
- **Timer visibility**: session timer + "your photos are safe" reassurance copy on every post-payment screen.
- **Thank-you**: confetti, "photos are on your phone" reminder, auto-return to attract.

**Tests**: `No_staff_copy_in_selfservice_snapshot` (i18n key sweep), `Idle_at_select_autopicks_composes_prints_and_shows_qr`, `Idle_at_capture_advances_with_captured_shots`, `Autocomplete_emits_auto_completed_event`, `Help_qr_renders_operator_contact`, `Staffed_mode_copy_unchanged`.

**Edge cases**: idle with **zero** shots captured (paid, then vanished before first capture) → no print, gallery link still issued with raws-if-any, `RefundFlagged` is NOT raised (service was available — record `abandoned_paid` for the operator's dashboard instead); language toggle persists per session only, resets at attract.

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
---

*Not in any milestone: Android/PWA box shell (ADR-018 evolution — becomes real only if box hardware cost blocks sales), dye-sub photo tier for the Box (ADR-020 — config + certification, no new architecture).*
