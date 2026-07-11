# potoku-platform-web — Task Breakdown

> React + Vite SPA. Read [tasks/README.md](README.md) first.
> Iron rules: **no hardcoded brand anything** (colors/name/logo/copy come from the brand manifest, ADR-012 — the tokens in `index.css` are fallback defaults only); **no hand-written API types** (generated client only, ADR-016); gallery routes are public and must work on a mid-range Android phone over 3G — admin can assume a laptop.
> Test infra: vitest + Testing Library (WEB-003). Every screen task ships loading / empty / error states — a screen that only handles the happy path is not done.

---

## Milestone M0

### WEB-001 · Generated API client + fetch wrapper
Milestone M0 · Size S · Level junior · Depends: API-008

**Spec**
- Vendor `potoku-api/clients/typescript` into `src/lib/api/generated/` with a `sync-client.sh` + version stamp (same pattern as BOOTH-005; replace both when publishing is decided — that decision has a marker in ADR-016).
- `src/lib/api/client.ts`: base URL from `import.meta.env.VITE_API_URL` (default `/api` → Vite proxy); wraps fetch with: JSON handling, typed `ApiError { status, problem: ProblemDetails }` from problem+json bodies, 15 s timeout via AbortController, network-vs-http error distinction.
- Auth token injection hook left as a seam (`getAuthHeader?: () => string` — WEB-020 fills it).

**AC**
- [ ] Gallery call against local compose works through the wrapper.
- [ ] Raw `fetch` is banned outside `lib/api/` (eslint `no-restricted-globals`/import rule — prove with a red throwaway commit).

**Tests**: `Problem_json_parsed_into_ApiError`, `Network_failure_distinguished_from_http_error`, `Timeout_aborts_at_15s` (fake timers), `410_and_404_surface_status_faithfully` (gallery contract depends on telling them apart).

**Edge cases**: API returns HTML (proxy misconfig) → typed `ApiError("invalid_response")`, not a JSON parse crash; empty 204 bodies.

---

### WEB-002 · Public gallery page (the M0 demo moment)
Milestone M0 · Size L · Level mid · Depends: API-006, WEB-001

**Context**: PRD F3 — this page IS the walking-skeleton payoff: scan QR, see your strip. It gets the most polish budget in M0.

**Spec**
- Route `/g/:code` states (explicit, each designed): **loading** (skeleton tiles); **pending** ("photos on the way 📸", poll every 5 s, max 3 min then a "taking longer than usual — pull to refresh" state); **ready** (composed strip hero first, then raw grid; tap → full-screen viewer with swipe; per-photo download via signed URL; "Download all" = sequential downloads M0, zip is a later nicety — say so in UI copy? No: just offer per-photo + all); **expired** (410 → friendly page with store name/contact from the 410 body per API-031; M0 interim: generic copy until that lands — leave marker `WEB: API-031`); **not found** (404 → "check your link" page); **error** (network → retry button).
- Mobile-first: 360 px wide baseline, images `loading="lazy"`, signed URLs requested per-view (they expire in 15 min — refetch gallery on viewer open if older than 10 min).
- `noindex` meta (belt to API's suspenders). No auth, no admin chrome — separate layout shell.

**AC**
- [ ] Real phone over LAN: scan booth QR → gallery in < 3 s on Fast 3G throttle (Lighthouse run attached to PR).
- [ ] Pending→ready transition works live (start gallery open before booth upload finishes).
- [ ] All 6 states reachable and snapshot-tested.

**Tests** (Testing Library + MSW mocking the generated client's endpoints)
- `Pending_polls_then_renders_ready` (fake timers)
- `Polling_stops_after_ready` (no zombie intervals — assert timer count)
- `Polling_gives_up_at_3min_with_refresh_state`
- `410_renders_expired_404_renders_not_found` (distinct)
- `Signed_urls_refetched_when_stale` 
- `Download_uses_signed_url_untouched` (no URL mangling)

**Edge cases**: gallery opened while offline (customer in a basement) → error state with retry, content cached by the browser stays visible if already loaded; 30 raws on a cheap phone → virtualize? No — lazy images + thumbnail-sized rendering are enough at n≤30 (shot cap), assert no full-res fetch until viewer opens; screen-reader labels on viewer controls (a11y baseline now, full pass WEB-041).

---

### WEB-003 · Test + CI infra
Milestone M0 · Size S · Level junior · Depends: —

**Spec**: vitest + @testing-library/react + MSW (mock at network layer so the generated client runs for real); GitHub Actions: typecheck, lint (eslint flat config incl. the no-raw-fetch rule and a `no-hex-colors-outside-tokens` custom rule — regex-lint `#[0-9a-f]{3,8}` outside `index.css`/token files, ADR-012 enforcement), test, build. Preview deploy optional-later.

**AC**: red PR on: type error, raw fetch, hex color in a component, failing test (prove each with throwaway commits).

---

## Milestone M1

### WEB-011 · Canvas rasterizer (template-kit parity)
Milestone M1 · Size M · Level mid · Depends: template-kit published or vendored (see BOOTH-005 pattern)

**Context**: ADR-016 — the SPA has no Node, so admin previews render `composeOps()` on `<canvas>`. Parity with the booth's sharp rasterizer is the whole point: what admin approves is what prints.

**Spec**
- Vendor/depend on `@potoku/template-kit` (same sync-script pattern; ONE decision for all three repos — raise it in standup, it's ADR-016's open marker).
- `src/lib/compose/canvas-rasterizer.ts`: same op interpretation contract as BOOTH-002 (order: background → photos → frame; crop via `drawImage(img, sx, sy, sw, sh, dx, dy, dw, dh)`); asset resolver = `HTMLImageElement` loader with decode await; output: canvas → `toBlob("image/png")` for download, or live canvas for preview.
- **Parity fixture**: commit the same golden mini-template + inputs used by BOOTH-002's golden test; assert ops-level identity (same `composeOps` output) + dimensional identity of the render (canvas pixels ≈ sharp pixels within a small tolerance — encoder differences make byte-equality wrong; compare downsampled RGBA with per-channel tolerance ≤2).

**Tests**: `Ops_identical_to_kit_fixture` (literally import fixture JSON), `Render_dimensions_match_canvas_spec`, `Pixel_parity_within_tolerance_vs_committed_sharp_render` (the sharp PNG from BOOTH-002's golden test is committed here as the reference), `Missing_image_rejects_with_AssetMissing`, `Rotation_nonzero_throws_UnsupportedOp` (mirror BOOTH-002's honesty).

**Edge cases**: CORS on asset images (must be same-origin/signed URLs — set `crossOrigin` correctly or toBlob taints); huge canvas (1200×3600) on low-end laptop — render at preview scale (≤600px wide) for live preview, full-res only on explicit export; `image/png` toBlob memory spike — one render at a time (queue).

### WEB-010 · Template manager (admin)
Milestone M1 · Size L · Level mid · Depends: API-010..012, WEB-011, WEB-020 auth seam (M1 interim: behind a dev flag until login lands)

**Spec**
- List (templates + versions, active badge) / detail / create flows.
- Create/edit draft version: JSON config editor (plain `<textarea>` + Monaco later — textarea now) with **client-side zod validation** (template-kit schema) live-reporting pointer paths; asset upload dropzone (PNG only, size check client-side too); slot visualizer overlay (rectangles drawn over canvas preview — cheap, catches out-of-bounds instantly before the API 422s).
- Preview panel: WEB-011 rasterizer with committed sample photos (3 neutral portraits in `public/samples/`) + variant switcher.
- Activate button with confirm dialog ("versions become immutable once used — new changes need a new version").

**AC**
- [ ] Upload PRD §F5 example config + 2 PNGs → preview renders → activate → booth sync picks it up (cross-repo demo with BOOTH-013, coordinate the demo in PR).
- [ ] Invalid config never reaches the API (client zod catches; API 422 still rendered faithfully if it slips through).

**Tests**: `Invalid_json_shows_pointer_errors_inline`, `Slot_out_of_bounds_flagged_visually_and_blocking`, `Activate_confirm_flow`, `Api_422_errors_mapped_to_fields`, `Variant_switcher_recomposes_preview`.

**Edge cases**: config references asset not yet uploaded → preview shows placeholder + warning chip, activate disabled (mirrors API-011 activation rule client-side); editing an *active* version → UI forces "duplicate to v{n+1}" flow (never lets you think you're editing live).

---

## Milestone M2

### WEB-020 · Auth: login, token lifecycle, protected routes
Milestone M2 · Size L · Level mid · Depends: API-020

**Spec**
- Login page (brand logo/colors from manifest — first consumer of WEB-025's hydration, coordinate); access token in memory only, refresh token in `httpOnly` cookie? — **No**: API issues body tokens; store refresh in `localStorage` with the accepted risk documented, access in memory; silent refresh on 401-once-then-retry (single-flight: concurrent 401s trigger ONE refresh, queued retries).
- Route guard: `/admin/*` requires session; role-gate hook `useRequires(role)` hiding admin-only nav (server enforces anyway — UI is UX, not security; comment says so).
- Logout: revoke call + local wipe + redirect. Idle-logout after 8 h (staff shift), warning toast at 7 h 55 m.

**AC**: refresh rotation works through the UI (leave a tab 20 min, act — no logout, one refresh); reused-rotated-token family kill (API-020) lands you on login with "signed out for security" message.

**Tests** (MSW): `401_triggers_single_refresh_with_queued_retries` (fire 3 parallel calls), `Failed_refresh_redirects_login_and_wipes`, `Role_gate_hides_admin_nav_for_staff`, `Deep_link_after_login_returns_to_target`, `Logout_revokes_and_wipes`.

**Edge cases**: two tabs — refresh rotation race (tab A rotates, tab B holds dead token) → storage event listener syncs tokens across tabs (test with simulated storage event); clock-skewed client must not pre-emptively discard valid tokens (trust server 401s, don't parse `exp` locally for logout decisions).

### WEB-021 · Packages CRUD screens
Milestone M2 · Size M · Level junior · Depends: API-022, WEB-020

**Spec**: list (active + archived toggle), create/edit form with the API-022 validation mirrored client-side (price IDR input with thousands display but integer value — `Rp 150.000` renders, `150000` submits; duration/window/cap as bounded steppers; window ≤ duration cross-field validation live), archive with confirm.

**Tests**: `Idr_input_formats_display_submits_integer`, `Window_greater_than_duration_blocked_inline`, `Archived_hidden_by_default_toggle_shows`, `Server_422_mapped_to_fields`.

**Edge cases**: editing package used by today's bookings → banner "existing sessions keep their old values" (API-015 snapshot semantics surfaced to humans).

### WEB-022 · Booking flow (staff counter screen)
Milestone M2 · Size L · Level mid · Depends: API-023..024, WEB-020, WEB-021

**Context**: PRD F1 — staff does this 50× a day. Speed metric: walk-in booking ≤ 30 s, keyboard-navigable end to end.

**Spec**
- New booking: name, phone (auto-normalizes 08xx → +628xx visibly on blur), package select (price/duration shown), party size, now/scheduled toggle (scheduled → datetime picker, store tz); create → payment panel (amount prefilled from package, method radio cash/QRIS/transfer, idempotency key generated per panel-open) → on confirmed: **code screen** — session code huge (readable across a counter), issue/re-issue button, "print receipt" (WEB-023).
- Booking list: today default, search by name/phone/code (debounced 300 ms), status chips, row → detail with payments, cancel (confirm + reason when after payment).
- Override check-in path (unpaid): reason textarea mandatory ≥10 chars, marked prominently as audited.

**AC**: complete walk-in flow ≤ 30 s with keyboard only (Tab order test + manual video in PR); re-issue voids old code with visible warning (API-024 semantics surfaced).

**Tests**: `Phone_normalization_display_and_submit`, `Payment_idempotency_key_stable_across_double_click` (rapid double-submit → one payment), `Scheduled_past_datetime_blocked`, `Reissue_shows_void_warning_and_new_code`, `Override_requires_reason_min_length`, `Search_debounce_and_race` (fast typing → only last request's result renders — abort stale).

**Edge cases**: package archived between form-open and submit → server 422 mapped to a package-reselect prompt; offline counter (store Wi-Fi blip) → optimistic UI is FORBIDDEN here (money) — spinner + honest failure with retry, never fake success.

### WEB-023 · Receipt print view (80 mm, ADR-013)
Milestone M2 · Size M · Level junior · Depends: WEB-022

**Spec**
- `/print/receipt/:bookingId` — minimal route rendering: brand header (logo mono-friendly, name), booking summary (package, party, amount, method, staff initials, datetime), **session code in ≥24 pt monospace**, footer copy from brand config; `@media print` CSS for 80 mm roll (`width: 72mm` printable, no margins trickery — test on the real printer), auto-`window.print()` on load when `?auto=1` (kiosk-printing flow), window closes after print event.
- Runbook `docs/runbooks/counter-print.md` in this repo: Chrome `--kiosk-printing` shortcut setup, default-printer config, paper size driver settings (part of the task — staff will follow it verbatim).
- NO gallery QR/PIN on the receipt (ADR-013 — link doesn't exist yet; the footer says "you'll get a QR at the booth").

**AC**: real 80 mm thermal print is legible, code scannable-by-eye at arm's length, total length ≤ 15 cm of paper (thermal paper costs money); photo of a real receipt in the PR.

**Tests**: `Renders_all_booking_fields`, `Auto_print_fires_once_on_query_flag`, `No_pin_or_gallery_url_present` (regression guard for the ADR-013 correction), print-CSS snapshot.

**Edge cases**: long customer names (25+ chars) wrap, never truncate the code; reprint from booking detail reuses the same route (idempotent — it's just paper).

### WEB-024 · Bookings day view + check-in board
Milestone M2 · Size S · Level junior · Depends: WEB-022

**Spec**: today-at-a-glance: scheduled timeline + walk-ins, status columns (Confirmed / CheckedIn / Completed / NoShow-Cancelled), auto-refresh 30 s (visibility-aware: pause when tab hidden), click-through to detail.

**Tests**: `Groups_by_status`, `Refresh_pauses_when_hidden` (visibilitychange fake), `Empty_day_state`.

### WEB-025 · Brand config editor + theme hydration (whitelabel heart)
Milestone M2 · Size L · Level mid · Depends: API-027, WEB-020

**Spec**
- **Hydration (runtime, every visitor)**: on SPA boot, fetch brand manifest (ETag-cached) → set CSS custom properties on `:root` (`--brand-accent` etc.), document title, favicon, logo slots; gallery pages included (customer sees the store's brand, not Potoku's); manifest failure → shipped defaults, zero visual breakage (test it).
- **Editor (admin)**: form for name, colors (color inputs writing token values with contrast warnings — WCAG AA check accent-on-surface, warn not block), fonts (curated dropdown, not free-text — font loading is a rabbit hole), copy overrides (booth strings table: key → default → override), logo upload (PNG only per API-027, preview at 3 sizes), `galleryPinRequired` toggle with explainer text; live preview pane = a mini gallery + mini admin header re-rendered with draft tokens **before save**.
- Save → API validates (422 pointers mapped) → hydration cache invalidated.

**AC**: change accent color → admin UI + a gallery link (new tab) both reflect it after save without hard refresh (broadcast channel or refetch-on-focus — pick, test); contrast warning on white-on-yellow.

**Tests**: `Manifest_hydrates_css_vars_and_title`, `Manifest_failure_uses_defaults`, `Draft_preview_does_not_leak_to_live_tokens` (draft scoped to preview pane), `Contrast_warning_below_AA`, `Copy_override_roundtrip`.

**Edge cases**: partial manifest (older API) → per-field defaults, never all-or-nothing; XSS via copy overrides → all copy rendered as text nodes, never `dangerouslySetInnerHTML` (regression test with a `<script>` payload string).

---

## Milestone M3

### WEB-030 · After-sales workspace
Milestone M3 · Size L · Level mid · Depends: API-033..034, WEB-020

**Context**: PRD F4 + success metric "any after-sales action in under 1 minute". Design for the phone-call scenario: customer on the line, staff typing.

**Spec**: single search box (code / phone / name — API decides matching), results with session status + booking + link state (active/expired + expiry date); detail panel actions, each with confirm + result toast: **Resend link** (shows new shortCode+PIN once, warns old link dies, copy-to-clipboard + "send via WhatsApp" `wa.me` deep link with prefilled message from brand copy), **Extend** (day stepper ≤30, shows new expiry), **Recover** (only on Abandoned-with-assets, per API rules — button hidden otherwise), **Reprint** (asset picker → queue → job status chip polling until Done/Failed).

**AC**: timed manual run of each action < 60 s from search focus (video in PR); every action's audit reason field where API requires it.

**Tests**: `Search_by_each_key_renders_results`, `Resend_displays_new_credentials_once_and_warns`, `Extend_updates_expiry_display`, `Recover_hidden_when_ineligible`, `Reprint_polls_job_status_to_terminal`, `Wa_deeplink_encodes_message_and_number`.

**Edge cases**: resend while customer's gallery is open (they'll get killed mid-viewing) → warning copy states it; double-click resend → one call (button disables in-flight); phone search with 08 vs +62 input both hit (normalization parity with WEB-022).

### WEB-031 · Gallery PIN gate
Milestone M3 · Size M · Level junior · Depends: API-030, WEB-002

**Spec**: gallery `401 pin_required` → PIN screen (4 big digit boxes, auto-advance, paste support); unlock token kept in `sessionStorage` (per-tab, dies with tab — deliberate); `attemptsRemaining` surfaced ("2 tries left"); 429 lockout → countdown from `Retry-After`; store with PIN off → straight through (regression test).

**Tests**: `Pin_screen_on_401_then_gallery_on_unlock`, `Attempts_remaining_rendered`, `Lockout_countdown_from_retry_after`, `Token_scoped_per_tab_sessionstorage`, `Pin_paste_fills_boxes`.

**Edge cases**: unlock token expires mid-viewing (15 min) → next signed-URL refresh 401s → re-show PIN gate *preserving scroll position*; iOS keyboard: `inputmode="numeric"`.

### WEB-032 · Dashboard
Milestone M3 · Size M · Level junior · Depends: API-035, WEB-020

**Spec**: today tiles (sessions by status, revenue IDR-formatted, completion rate vs the 95% PRD target — red under target), popular templates (7d bar list), device tiles (online/offline from LastSeenAt, camera/printer/disk chips from snapshot, click → alert history WEB-040); auto-refresh 60 s visibility-aware; store tz passed to API (API-035 contract).

**Tests**: `Tiles_render_zero_state_for_fresh_store`, `Completion_rate_thresholds_color`, `Device_offline_badge_from_lastseen`, `Tz_param_sent`.

**Edge cases**: revenue with overpayments (API-023) shows a footnote chip; a store with 3 booths — tiles grid scales (design for 1–6).

---

## Milestone M4

### WEB-040 · Device health & alerts UX
Milestone M4 · Size M · Level junior · Depends: API-040, WEB-032

**Spec**: alerts inbox (unacked default, filter by device/kind, ack button with optimistic UI + rollback on failure); device detail: heartbeat sparkline (last 24 h), hardware snapshot pretty-printed (camera model, printer binding, disk %, queue depth — from BOOTH heartbeats), hardware-check report history (BOOTH-019 payloads rendered as pass/fail tables).

**Tests**: `Ack_optimistic_with_rollback`, `Snapshot_renders_all_known_fields_and_tolerates_unknown` (forward-compat: unknown snapshot keys render raw, don't crash), `Check_report_table_from_fixture_payload`.

### WEB-041 · i18n (ID/EN) + a11y pass
Milestone M4 · Size L · Level mid · Depends: all customer-facing screens

**Spec**: extract customer-facing strings (gallery, PIN gate, receipt) + staff-facing high-traffic screens to an i18n layer (lightweight — `react-i18next` or a typed dictionary; pick, justify in PR); language: gallery follows `navigator.language` ID/EN with manual toggle; brand copy-overrides (WEB-025) layer on top of the active locale. A11y: axe run in CI for gallery + PIN + login (zero serious violations), focus traps in modals, viewer keyboard nav.

**AC**: switching language live-swaps gallery copy; axe CI job green; brand override still wins over locale default (precedence test).

**Tests**: `Locale_detection_and_toggle`, `Brand_override_beats_locale`, `Axe_zero_serious_on_public_pages` (CI), `Modal_focus_trap`.

**Edge cases**: mixed: brand overrides only one locale → other locale falls back to its default (not the overridden other-locale string); currency/date formatting via `Intl` with explicit `id-ID` (never rely on host locale for money display).

---

*Not in any milestone: online self-booking UI, payment gateway checkout, multi-store switcher UI (data model is ready; UI is post-M4 per §12), visual template designer.*
