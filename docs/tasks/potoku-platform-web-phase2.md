# Potoku Platform Web — Phase 2 tasks (Box pivot, M10 + M11)

> Split from [potoku-platform-web.md](potoku-platform-web.md) (MVP, M0–M8). Conventions, global DoD, and the cross-repo dependency table live in the [README](README.md). Design source: [ADR-021](../DECISIONS.md), [PRD §6 admin redirect + F7](../PRD.md).
>
> **Order**: WEB-104 (SaaS admin shell) lands before every other M10 web task — build new screens in the new shell, don't retrofit.

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

**Spec**: public marketing-adjacent signup page — **Potoku product-branded** (no tenant exists yet; tenant whitelabel per ADR-012 starts at their customer-facing surfaces, consistent with WEB-104's chrome-vs-content boundary): email + venue name → "check your email" → verified completion (password, tenant/store details) → land in a first-run **onboarding checklist**: pair your box (reuses the WEB-042 enrollment flow), see your trial status, sell your first session. Enumeration-safe copy (mirror API-100's indistinguishable responses). Mobile-first — operators sign up from a phone at an expo booth.

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
---

## Implementation log

### 2026-07-16 — WEB-104 complete (Codex)

- Replaced tenant-themed admin chrome with a Potoku product shell: persistent grouped sidebar, slim store-context topbar, user menu, and tablet/mobile navigation.
- Added a single semantic admin token file with slate surfaces, one restrained accent, role/state colors, an 8px spacing grid, and explicit dark-theme overrides.
- Restyled existing dashboard, booking, device, template, after-sales, package, brand, and store surfaces through isolated admin tokens; tenant-branded gallery, booking, receipt, and preview content remain outside the product-token scope.
- Preserved all existing route paths and added an admin-only nested route guard for package, template, brand, and store deep links.
- Added the required WEB-104 tests and public-route snapshots plus `docs/phase2-web104-qa.md` for PR screenshot QA.
- Validation: `npx eslint src`, 77 Vitest tests, `npm run build`, `git diff --check`, and the Impeccable deterministic design scan pass. Repository-wide `npm run lint` remains affected by pre-existing untracked `.github/skills` files outside this implementation.
- The generated web client has no API-100–103 or API-110/111 operations; on 2026-07-16 the user directed all Phase 2 web work to proceed, so remaining screens use a contained typed compatibility adapter until the generated contract catches up.

### 2026-07-16 — WEB-100 complete (Codex)

- Added product-branded `/signup` and stateless `/signup/complete?token=…` routes with enumeration-safe confirmation, rate-limit/network handling, and friendly expired-link recovery.
- Completion persists only tenant/store/timezone draft fields per token (never passwords), accepts auth tokens atomically, and lands at `/admin/onboarding` even when the verification link opens in another browser.
- Added a first-run checklist backed by onboarding status, with trial visibility, paired-box and first-sale progress, direct links to sell/earnings, and inline reuse of WEB-042's `RegistrationPanel`.
- Added signup/onboarding responsive styles and the five named WEB-100 tests.
- Validation after this step: typecheck passes; `SignupPage.test.tsx` passes 5/5.

### 2026-07-16 — WEB-101 complete (Codex)

- Added the tenant earnings route with today/week/month net and session summaries, daily net visualization, per-box totals, and an explicit gross/gateway-fee/platform-fee/net ledger.
- Added payout history, current unpaid balance, correction entries with mandatory visible reasons, and a first-sale onboarding empty state.
- Added authenticated CSV downloads with UTC period boundaries and store-scoped requests; staff requests are always pinned to their assigned store.
- Added mobile/container-responsive report layouts and locale-safe integer IDR formatting through `Intl.NumberFormat("id-ID")`.
- Validation after this step: typecheck passes; `EarningsPage.test.tsx` passes 5/5.

### 2026-07-16 — WEB-102 complete (Codex)

- Added a separately styled `/platform` shell and route tree with a router-level `PlatformAdmin` guard; tenant admins receive a product-branded 404 without platform navigation disclosure.
- Added the tenant operations list/detail surfaces, subscription and box summaries, invoice paid actions, payout recording with synchronous double-submit protection and a stable idempotency key, and reason-required compensating entries.
- Added exact 422 balance/amount feedback, friendly concurrent already-paid handling, settlement CSV downloads by UTC month, and server-confirmed audit feedback.
- Added cross-tenant fleet health plus explicit acting-tenant session context; authenticated API calls include `X-Acting-Tenant` and the shell keeps the audited context visibly pinned until exited.
- Validation after this step: typecheck passes; `PlatformPages.test.tsx` passes 6/6.

### 2026-07-16 — WEB-103 complete (Codex)

- Added the admin subscription page with lifecycle state, plan pricing, paired-box count, platform fee rate, payment instructions, and invoice history.
- Added shell-level Trial, PastDue, and Suspended banners; Active has no banner, and suspension copy explicitly preserves dashboards and customer gallery continuity while explaining that new selling pauses.
- Subscription state silently refetches after other successful API reads and on focus/visibility changes, so lifecycle transitions appear without a hard refresh.
- Added responsive subscription summaries and invoice rows plus the four named tests and a live-refetch edge-case test.
- Validation after this step: typecheck passes; `SubscriptionPage.test.tsx` passes 5/5 without warnings.

### 2026-07-16 — WEB-110 complete (Codex)

- Added tenant-scoped `/admin/incidents` and cross-tenant `/platform/incidents` queues for debounced box-offline episodes and `RefundFlagged` sessions.
- Refund rows show customer-paid amount, incident time, box, drafted compensation, and retained context when a box has since been revoked; platform rows additionally identify the tenant.
- Added audited acknowledgement with optimistic row and nav-badge updates plus full rollback if the server rejects the acknowledgement.
- Added live unacknowledged counts in tenant and platform navigation, with tenant mode deliberately exposing no tenant selector or tenant identity.
- Validation after this step: typecheck passes; `IncidentQueuePage.test.tsx` passes 4/4.

### 2026-07-16 — Phase 2 web final validation (Codex)

- All scoped tasks are complete: WEB-104, WEB-100, WEB-101, WEB-102, WEB-103, and WEB-110.
- Final validation: 21 Vitest files and 102 tests pass; `npx eslint src`, `npm run build`, and `git diff --check` pass.
- The Impeccable deterministic scan over all changed UI/CSS targets reports zero findings after replacing two side-tab accent patterns with quieter status cues.
- The generated OpenAPI client still predates API-100–103 and API-110/111; all Phase 2 calls remain isolated in `src/lib/api/phase2.ts` so they can be replaced mechanically when the generated contract lands.

*Not in any milestone: Midtrans recurring self-serve billing UI (ADR-021 upgrade), domain-based tenant resolution (ADR-012).*
