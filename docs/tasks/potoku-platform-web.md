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

### WEB-042 · Admin booth enrollment
Milestone M4 · Size M · Level mid · Depends: API-043, WEB-020, WEB-040

**Spec**: admin-only Register booth action on Devices: active-store picker, recognizable name, create/loading/error states, one-time code with second-by-second expiry, Copy, Cancel, and Generate new code. List Pending, Paired, Offline, Expired, Cancelled, and Revoked distinctly; allow cancellation and token revocation. Pairing codes and device tokens never enter local/session storage, URLs, analytics, or logs. Success copy sends the operator to booth-local camera/printer setup and hardware check.

**Tests**: admin role gate; store/name validation; countdown; typed create/cancel failures; regenerate cancels the old code; no browser persistence; first heartbeat moves Pending to Paired.

---

## Milestone M5 — online booking & checkout

> M5 starts only after M4 is green. Public pages here get the gallery bar: brand-hydrated, mobile-first, mid-range Android over 3G.

### WEB-050 · Public self-booking flow
Milestone M5 · Size L · Level mid · Depends: API-050, WEB-001, WEB-025 (brand hydration)

**Context**: PRD §5 "Later" promoted to M5; ADR-005's deferred scope opens. This is a revenue surface — same polish budget as WEB-002.

**Spec**
- Route `/book` (public layout, brand-hydrated). Wizard steps, each its own URL (`/book/package`, `/book/slot`, `/book/details`, `/book/review`) so back button works; draft state in `sessionStorage` keyed by a client-generated `draftKey` (uuid).
- **Package**: cards (name, price IDR-formatted, session duration, prints included) from the public packages endpoint. **Slot**: 7-day date strip + slot grid from the availability endpoint; unavailable slots visibly disabled. **Details**: name + phone (08xx → +628xx normalization on blur — parity with WEB-022). **Review** → `POST` create pending booking (with `draftKey` as idempotency key) → hand off to WEB-051 checkout.
- Payment deadline from the API is visible from review onward ("complete payment within 15:00", live countdown).

**AC**
- [ ] Full flow phone-usable ≤ 90 s to checkout handoff (manual video in PR).
- [ ] Slot taken between render and submit → API 409 `slot_taken` → returned to slot step with friendly copy + refreshed grid (the race path is tested, not hoped away).
- [ ] Loading / empty ("no slots today") / error states designed on every step.

**Tests**: `Slot_grid_renders_availability_states`, `Conflict_409_returns_to_slot_step_and_refreshes`, `Phone_normalization_parity_with_staff_flow`, `Back_button_preserves_draft`, `Double_submit_one_booking_via_draftKey`, `Deadline_countdown_renders_api_value`.

**Edge cases**: package archived mid-flow → 422 mapped to package re-pick; store with online booking disabled → `/book` renders a "call us" page with store contact from the brand manifest (API returns 404 — don't error-splash); scheduled slot crossing midnight store-time — display in store tz, submit UTC (WEB-022 discipline).

---

### WEB-051 · Gateway checkout
Milestone M5 · Size L · Level mid · Depends: API-051, API-052, WEB-050

**Context**: The gateway (Midtrans/Xendit) is the API's secret — the SPA renders whatever checkout payload it's handed and never hardcodes a provider.

**Spec**
- Consume API-051's checkout payload: `{ mode: "redirect"|"popup", url?, token?, scriptUrl? }` — popup mode loads the gateway script from `scriptUrl` (Midtrans Snap shape), redirect mode navigates. No provider names in web code.
- Return path (`/book/status/:token?from=gateway` or popup callback) → **verifying** state: poll the public booking status until Confirmed (webhook processed), 5 s interval, max 2 min → then an honesty state "payment received, still confirming — this page will update" with manual refresh. **Never fake success.**
- Gateway failure/expiry → retry payment (new checkout call, same booking while deadline live) or restart flow when the booking expired.

**Tests** (MSW): `Popup_mode_loads_script_and_invokes_with_token`, `Redirect_mode_navigates`, `Verifying_polls_until_confirmed`, `Poll_timeout_shows_honesty_state_not_success`, `Retry_payment_reuses_booking_while_deadline_live`, `Expired_booking_offers_restart`.

**Edge cases**: popup blocked → inline "continue to payment" link fallback; user pays then kills the tab before returning → WEB-052's status page is the recovery path (copy on it says so); paying an already-settled booking → API rejects → map to "already paid" → status page.

---

### WEB-052 · Booking status + code delivery page
Milestone M5 · Size M · Level junior · Depends: API-052, API-053, WEB-051

**Context**: Online bookings have no staff receipt (ADR-013) — this page IS the customer's receipt and their session code carrier.

**Spec**
- `/book/status/:token` (public, capability token from booking create — bookmarkable). States: **pending-payment** (deadline countdown + pay button), **confirmed** (**session code huge** — same visual weight as WEB-022's code screen; store name/address, scheduled time, "enter this code at the booth"), **expired** (re-book CTA), **cancelled**.
- Confirmed extras: copy-code button, add-to-calendar (client-generated `.ics`), `wa.me` share of the status link (send-to-self — the "my receipt" move).
- Poll while pending-payment (10 s, visibility-aware like WEB-024); stop on terminal status.

**Tests**: `Each_status_renders_designed_state`, `Code_visible_only_when_confirmed`, `Ics_carries_slot_time_in_store_tz`, `Poll_stops_on_terminal_status`, `Refetch_on_focus_shows_reissued_code`.

**Edge cases**: unknown/malformed token → generic not-found (never confirm a booking exists — enumeration guard); staff re-issued the code after confirmation (API-024) → page shows current server truth (refetch on focus, test it).

---

### WEB-053 · Online bookings in staff views
Milestone M5 · Size S · Level junior · Depends: WEB-022, WEB-024, API-052

**Spec**: additive to existing screens, no new routes: booking list/detail/day-board gain a source chip (`online` / `walk-in`); gateway payment rows render read-only (provider, reference, settled time — no edit/delete; staff-recorded payments unchanged); pending-payment online bookings visible with their deadline.

**Tests**: `Source_chip_renders_both_kinds`, `Gateway_payment_rows_read_only`, `Pending_online_booking_shows_deadline`.

**Edge cases**: staff adds a cash payment to a gateway-pending booking (customer walked in and paid at the counter instead) → allowed; API resolves state — surface its response faithfully, no client-side second-guessing.

---

## Milestone M6 — animated gallery

### WEB-060 · Gallery animated tile
Milestone M6 · Size M · Level junior · Depends: API-062, WEB-002; cross-repo: BOOTH-031 produces the assets

**Spec**: gallery renders `kind=animated` assets as a first-class tile (after composed, before raws): switch on the `contentType` the media ref carries — `<video muted autoplay loop playsinline>` for mp4, `<img>` for gif/webp; tap → fullscreen loop in the viewer; download preserves the original file.

**Tests**: `Mp4_renders_video_muted_autoplay_playsinline`, `Gif_renders_img`, `Download_uses_signed_url_untouched`, `Gallery_without_animated_unchanged` (regression).

**Edge cases**: iOS low-power mode blocks autoplay → poster frame + play affordance (design the paused state, test it); unknown future content type → download-only tile, never a broken player.

---

## Milestone M7 — multi-store UI

### WEB-070 · Store context switcher
Milestone M7 · Size M · Level mid · Depends: API-070, WEB-020

**Context**: §12 — data model was multi-store from day one (every row hangs off StoreId); this makes it visible. UI-only scoping is UX; the server enforces (API-021), same stance as WEB-020's role gate.

**Spec**: admin-only header switcher (staff never see it — they're store-locked server-side): store list + "All stores" offered only where supported (dashboard); selection persisted per-user (`localStorage`) and injected by the API client from context; every store-scoped screen (bookings, templates, packages, devices, after-sales) refetches on switch.

**Tests**: `Switch_refetches_active_screens`, `Staff_never_sees_switcher`, `Persisted_selection_restored_on_boot`, `All_stores_offered_only_on_dashboard`.

**Edge cases**: deep link to store B's resource while store A is selected → auto-switch with a toast (the resource wins over the sticky selection); selected store archived since last visit → fall back to first active store + toast; single-store tenant → switcher hidden entirely (the pre-M7 experience is the default, not a regression).

---

### WEB-071 · Cross-store dashboard + store management
Milestone M7 · Size L · Level mid · Depends: API-070, API-071, WEB-032, WEB-070

**Spec**
- "All stores" dashboard: per-store comparison table (sessions today, revenue, completion vs the 95% target with threshold colors, devices online) from API-071's aggregate — one call, no per-store fan-out; single-store selection keeps the existing WEB-032 tiles.
- Store management (admin): list / create / edit / archive store (confirm dialog quotes API-070's rules), assign users to stores (role visible), devices-per-store overview.

**Tests**: `Aggregate_table_renders_per_store_rows_and_totals`, `Store_create_form_validation`, `Archive_confirm_flow_surfaces_409_reasons`, `User_assignment_roundtrip`, `Single_store_selection_keeps_existing_tiles`.

**Edge cases**: archived store's history remains in aggregates (row flagged); 1–10 stores — table stays readable (sticky header, sortable columns); revenue footnote chip for overpayments carries over from WEB-032.

---

## Milestone M8 — visual template designer

### WEB-080 · Designer canvas (slots on frame)
Milestone M8 · Size L · Level mid · Depends: WEB-010, WEB-011

**Context**: ADR-011 deferred the designer as polish; the JSON editor remains the escape hatch. The designer is a **config generator** — its output is ordinary template config through the existing API-010..012 endpoints; API and booth change zero (the ADR-011 payoff).

**Spec**
- New "Design" tab in WEB-010's editor, two-way synced with the JSON tab (designer edits update the JSON; valid JSON edits update the canvas; invalid JSON disables the canvas with a notice).
- Canvas preset picker (category → canvas size), frame PNG rendered as the working background, slot rects drawn / dragged / resized with handles: snap to 10 px grid + edge/center guides; slot list panel with numeric x/y/w/h inputs for precision and slot order (order = selection order on the booth); `fit` per slot. Rotation stays hidden until the rasterizers support ≠0 (WEB-011/BOOTH-002 both throw — don't offer what can't render).
- Out-of-bounds slots can't be drawn (what API-011 would 422 is blocked at draw time).

**Tests**: `Drag_updates_config_json`, `Numeric_inputs_and_drag_stay_in_sync`, `Snap_to_grid_and_guides`, `Out_of_bounds_blocked_at_draw_time`, `Json_edit_reflects_on_canvas`, `Invalid_json_disables_canvas_with_notice`.

**Edge cases**: overlapping slots — allowed (some designs overlap) but warn chip; huge frame PNG → client-side reject above API-012's 15 MB before upload; touch devices — handles get ≥44 px hit areas (admins do use tablets).

---

### WEB-081 · Designer: layers, variants, guided create
Milestone M8 · Size M · Level mid · Depends: WEB-080

**Spec**
- Layers panel mirroring the composition contract visually (background → photos → frame — same order BOOTH-002 rasterizes); optional background layer upload.
- Variant editor: list (id, name, frame PNG per variant), preview switcher reusing WEB-010's; dimension mismatch vs canvas blocked with message.
- "New template" guided path: category → canvas → upload frame → **auto-suggest slots** (alpha-region scan of the frame PNG finds transparent windows — best-effort bounding boxes, fully editable after) → name → save draft. Success metric: a template author never opens the JSON tab.

**Tests**: `Layers_panel_order_matches_composition_contract`, `Variant_add_remove_roundtrip`, `Alpha_scan_finds_rect_windows` (fixture frame with 4 windows), `Suggested_slots_editable`, `Variant_dimension_mismatch_blocked`.

**Edge cases**: non-rectangular transparent windows → bounding boxes suggested + UI copy stating the limitation; frame with no transparent regions (opaque) → zero suggestions + the BOOTH-002 no-alpha warning surfaced client-side.

---

## Milestone M10 — sell the box: SaaS surfaces

> ADR-021 activates the deferred SaaS console. Two audiences appear: the **box operator** (a tenant admin whose primary screen is their phone) and the **platform admin** (Toku). Keep them visually distinct — an operator must never wonder whose money they're looking at.

### WEB-104 · Admin workspace restyle — modern SaaS shell ⚠️ do FIRST in M10
Milestone M10 · Size L · Level senior · Depends: WEB-020 (admin shell), WEB-041 (i18n/formatting rules)

**Context**: Toku's direction (2026-07-15): the admin side should read as a **modern, clean SaaS admin panel**, not photobooth-themed. PRD §6 updated — "clean, but fun" is now customer-facing only. This lands *before* WEB-100..103 so every new SaaS screen is built inside the new shell, not retrofitted.

**Spec**
- **New admin shell**: persistent left sidebar (grouped nav: Operate / Money / Devices / Settings; platform-admin group only for that role), slim topbar (store/tenant context switcher, user menu), content area with max-width + consistent page header pattern (title, description, primary action).
- **Design system pass**: neutral palette (slate surfaces, near-black ink, ONE restrained accent), quiet cards, dense readable tables (sticky header, row hover, empty/loading/error states standardized), consistent form patterns, 8px spacing grid, subtle borders over shadows. Kill admin-side playfulness: no confetti, no pastels, no display font — those stay in booth/gallery.
- **Scope boundary**: customer-facing surfaces are **untouched** — gallery, public booking (WEB-050..053), signup marketing page keep tenant branding per ADR-012. The admin shell is *product*-styled (Potoku identity), not tenant-brand-styled; tenant brand appears only inside content that previews it (template previews, gallery preview).
- **Mechanics**: theme tokens in one file (no hex in components — the existing lint rule now protects the *admin* palette too), dark-mode-ready token structure (shipping light-only is fine, tokens must not preclude dark), responsive to tablet width (operators on phones use WEB-100/101 which must stay mobile-first).
- Migrate existing admin pages (dashboard, bookings, devices, templates, after-sales, stores) into the shell — screen-by-screen visual QA checklist in the PR.

**Tests**: `No_hardcoded_hex_in_admin_components` (lint), `Sidebar_groups_match_role` (staff/admin/platform-admin), `Existing_routes_render_inside_new_shell` (smoke per route), `Customer_surfaces_visually_unchanged` (snapshot on gallery + booking routes), `Tables_render_empty_loading_error_states`.

**Edge cases**: deep links and role gating must survive the nav restructure (no route paths change — restyle, not rearchitecture); template preview inside the neutral shell still renders tenant brand faithfully (the preview is content, the chrome is product).

### WEB-100 · Public signup + operator onboarding
Milestone M10 · Size L · Level mid · Depends: API-100, WEB-042 (enrollment UI), WEB-104 (shell)

**Spec**: public marketing-adjacent signup page (brandable, ADR-012): email + venue name → "check your email" → verified completion (password, tenant/store details) → land in a first-run **onboarding checklist**: pair your box (reuses the WEB-042 enrollment flow), see your trial status, sell your first session. Enumeration-safe copy (mirror API-100's indistinguishable responses). Mobile-first — operators sign up from a phone at an expo booth.

**Tests**: `Signup_happy_path_to_onboarding`, `Expired_token_friendly_retry`, `Duplicate_email_indistinguishable_copy`, `Checklist_reflects_pairing_and_first_sale`, `Mobile_viewport_no_horizontal_scroll`.

**Edge cases**: verified in a different browser than signup → completion works statelessly from the link; abandoned mid-completion → resumable until token expiry.

### WEB-101 · Operator earnings dashboard
Milestone M10 · Size M · Level mid · Depends: API-093, API-103, WEB-104 (shell)

**Spec**: tenant-side earnings view: today/this-week/this-month cards (net, sessions), daily chart, per-box breakdown, entry list (gross/fees/net — fee lines always visible, never a surprise), payout history + current unpaid balance, CSV export (API-103). Mobile-first; integer IDR via `Intl` `id-ID` (WEB-041 rule).

**Tests**: `Cards_match_ledger_fixtures`, `Fee_breakdown_always_rendered`, `Payout_history_and_balance`, `Export_downloads_csv`, `Staff_sees_own_store_only`.

**Edge cases**: zero-earnings tenant → welcoming empty state pointing at the onboarding checklist, not a blank table; compensating (negative) entries render explicitly as corrections with their reason.

### WEB-102 · Platform-admin console
Milestone M10 · Size L · Level senior · Depends: API-102, API-103, WEB-104 (shell)

**Spec**: separate route tree + visual shell (`/platform`): tenant list (subscription state, boxes, unpaid balance), tenant detail (invoices → mark paid, payouts → record with idempotency, compensations with mandatory reason), settlement export per period, cross-tenant fleet health (device status/alerts). Explicit acting-tenant banner whenever impersonation context is active (API-102 header). Role-gated at router level; tenant admins never see the routes exist.

**Tests**: `Tenant_admin_gets_404_shell_on_platform_routes`, `Mark_paid_flow_with_audit_toast`, `Payout_idempotent_on_double_click`, `Compensation_requires_reason`, `Fleet_health_renders_cross_tenant`, `Acting_tenant_banner_always_visible_when_set`.

**Edge cases**: payout > unpaid balance → API 422 surfaced with the exact numbers; two tabs marking the same invoice paid → second gets a friendly already-paid state.

### WEB-103 · Subscription status surfaces
Milestone M10 · Size S · Level junior · Depends: API-101, WEB-104 (shell)

**Spec**: tenant-side subscription page (plan, box count, invoices, state) + global banner states: Trial (days left), PastDue (grace countdown, pay-instructions copy from brand/platform config), Suspended (what still works: galleries + dashboards; what doesn't: selling). Copy must say customer galleries stay live.

**Tests**: `Banner_states_match_subscription_fixture`, `Suspended_copy_promises_gallery_continuity`, `Invoice_list_renders`, `No_banner_when_active`.

**Edge cases**: state changes while the app is open → banner updates on next data fetch, no hard refresh required.

---

## Milestone M11 — box field-hardening (web side)

### WEB-110 · Refund & incident queue
Milestone M11 · Size M · Level mid · Depends: API-110, API-111

**Spec**: operator-side alert inbox: box offline episodes, `RefundFlagged` sessions (customer paid, session unrecoverable — show amount, time, box, and the drafted compensation), acknowledge action (audited). Platform-admin sees the same cross-tenant (in WEB-102's shell). Badge count in the operator nav.

**Tests**: `Refund_flag_renders_amount_and_draft_compensation`, `Ack_clears_badge_and_audits`, `Offline_episodes_debounced_one_row_each`, `Tenant_scoping`.

**Edge cases**: alert for a since-revoked box → still visible (money questions outlive devices).

---

*Not in any milestone: AI features (background removal, beauty filters), Midtrans recurring self-serve billing UI (ADR-021 upgrade), domain-based tenant resolution (ADR-012).*
