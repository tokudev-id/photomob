# Potoku API — Phase 2 tasks (Box pivot, M9–M11)

> Split from [potoku-api.md](potoku-api.md) (MVP, M0–M7). Conventions, global DoD, and the cross-repo dependency table live in the [README](README.md). Design source: [ADR-018..021](../DECISIONS.md), [PRD F7](../PRD.md), [gap dispositions](../gaps/mvp-to-phase2.md).
>
> **Gate**: M9 starts only after M5's payment spine (API-051..052) is merged.

## Milestone M9 — Potoku Box: pay-to-start walking skeleton

> The Box pivot (ADR-018..020). Everything here reuses the M5 payment spine: same Booking aggregate, same webhook-is-truth rule, same gateway abstraction.

### API-090 · Device operating mode
Milestone M9 · Size S · Level junior · Depends: API-013, API-043

**Context**: ADR-018 — one booth app, two products. Mode is business config, therefore server-owned (ADR-015).

**Spec**: `Device.OperatingMode` enum `Staffed | SelfService` (default `Staffed`, migration backfills). Admin can set it on the device detail endpoint (audited, API-026 pattern). `Package` (API-022) gains a `SelfServiceEligible` flag (default false, admin-editable, audited) — the heartbeat snapshot contains **only eligible packages**, so staffed-tier packages never appear on a box. Heartbeat response carries the effective mode + that snapshot so the booth needs no extra round-trip; the snapshot is **display-only** — API-092 re-validates price/package server-side at purchase time. OpenAPI + both TS client snapshots regenerated.

**Tests**: `Mode_defaults_staffed`, `Mode_change_audited_and_delivered_on_next_heartbeat`, `Selfservice_heartbeat_includes_pricing_snapshot`, `Snapshot_contains_only_selfservice_eligible_packages`, `Staffed_heartbeat_omits_pricing_snapshot`.

**Edge cases**: mode flips while a session is active → booth applies it only from the next attract screen (assert the contract documents this; enforcement is BOOTH-040).

### API-091 · Dynamic QRIS charge via Midtrans Core API
Milestone M9 · Size M · Level mid · Depends: API-051 (gateway seam + webhook infra)

**Context**: ADR-019. M5's Snap checkout serves online booking; the Box needs a raw dynamic QR string to render on-screen. Same `IPaymentGateway` seam, new capability — never a second gateway abstraction.

**Spec**: extend the gateway port with `CreateQrisChargeAsync(orderId, grossAmount, expiryMinutes)` → `{ qrString, expiresAt, gatewayRef }` using Midtrans Core API (`payment_type: qris`, acquirer per config). Webhook handling reuses API-052 verbatim (signature check + verify-then-trust status re-query); QRIS settlement maps to the same paid transition. Sandbox config documented in the compose stack.

**Tests**: `Charge_returns_qr_and_expiry`, `Webhook_settlement_marks_paid_after_requery`, `Webhook_without_requery_confirmation_ignored`, `Expired_charge_never_transitions_paid`, `Amount_mismatch_rejected_and_alerted`.

**Edge cases**: Midtrans 5xx/timeout on charge create → typed `gateway_unavailable` Problem Details (box shows "try again", no booking row leaks — create charge first or roll back); duplicate webhook deliveries idempotent (existing API-052 guarantee, add a QRIS-flavored test).

### API-092 · Box purchase flow (pay-to-start session)
Milestone M9 · Size L · Level senior · Depends: API-090, API-091, API-025

**Context**: ADR-019 — money before session, box is the booth, no session code.

**Spec**: device-authenticated `POST /api/box/purchases` (self-service devices only, staffed devices 403): the **server's current package config is the price authority** — the heartbeat snapshot is display-only; a stale price/package in the request → typed `price_changed` refusal (box refreshes and re-offers). Creates a walk-in Booking `PendingPayment` (source=`box`) + QRIS charge, returns `{ purchaseId, qrString, amount, expiresAt }`. **Settlement extends API-052's branch — one money truth**: `source=box` Settled → in **one transaction**: gateway `Payment` row (exactly as API-052 writes it) + booking Confirmed + session created bound to the purchasing device (reuse API-025's activation internals, skip the code) + `EarningsEntry` (API-093). Dashboard revenue (API-035) thus includes box sales with zero new code. `GET /api/box/purchases/{id}` for polling → `pending | paid | expired`; on paid the response carries the ready session. `GET /api/box/purchases/current` returns this device's paid purchase whose session is non-terminal (boot recovery — BOOTH-043's contract lives HERE in M9; API-110 only hardens it). Per-device rate limit on creates; at most one pending purchase per device (creating a new one voids the old charge). A background sweeper expires stale purchases.

**Tests**: `Staffed_device_403`, `Stale_price_refused_with_price_changed`, `Purchase_creates_pendingpayment_booking_and_charge`, `Poll_pending_then_paid_returns_session`, `Settlement_writes_payment_row_confirms_booking_creates_session_and_earnings_atomically`, `Second_pending_purchase_voids_first`, `Expired_purchase_returns_expired_and_frees_device`, `Another_devices_purchase_404`, `Current_returns_paid_unconsumed_purchase_else_404`.

**Edge cases**: webhook lands *after* charge expiry (customer paid at second 899) → honor the money: purchase resurects to paid, session created — never swallow a settled payment; device revoked between create and poll → 401, charge voided by sweeper.

### API-093 · Operator earnings ledger
Milestone M9 · Size M · Level mid · Depends: API-092

**Context**: ADR-019 — platform Midtrans collects; the ledger is the operators' money truth and Toku's payout source. Correct from day one.

**Spec**: the ledger is **derived from the gateway `Payment` row, never a second money truth** — written in API-092's settlement transaction, one `EarningsEntry` per box `Payment` (tenant, store, device, session, paymentId FK, grossIDR = payment amount, gatewayFeeIDR from config rate, platformFeeIDR from a **config-default rate in M9** — API-101 migrates the source to the tenant's plan in M10 — netIDR; all integer IDR, ADR-007). Invariant, as a named test: `Σ(EarningsEntry.gross) == Σ(box gateway Payments)` over any period. `GET /api/earnings?from&to` (admin: tenant-wide; staff: own store) with daily totals. No mutation endpoints — corrections are compensating entries (platform-admin only, M10).

**Tests**: `Paid_purchase_appends_entry_with_correct_split`, `Entry_immutable`, `Entry_gross_always_equals_linked_payment_amount` (the one-money-truth invariant), `Totals_by_day_and_store`, `Staff_scoped_to_store`, `Rounding_never_loses_a_rupiah` (fee math property test: gross = fees + net always).

**Edge cases**: fee config changes → entries keep the rate captured at write time (snapshot, not reference); refund flag (M11) compensates, never edits.

---

## Milestone M10 — sell the box: SaaS machinery

### API-100 · Tenant self-signup
Milestone M10 · Size L · Level senior · Depends: API-020, API-026

**Context**: ADR-021 activates ADR-012's deferred half. Public surface — treat as hostile.

**Spec**: public `POST /api/signup` → creates `PendingSignup` (email + operator/venue name, hashed verification token, 24h expiry) and sends a verification link (notification seam may be a logged stub behind an interface — the email channel is a "Later" item, but the seam is not). Verified completion sets password and atomically creates tenant + first store + admin user + `Trial` subscription. Heavily rate-limited (per-IP and per-email), audited, enumeration-safe (identical response whether the email exists or not).

**Tests**: `Signup_verify_creates_tenant_store_admin_trial_atomically`, `Expired_token_rejected`, `Token_single_use`, `Duplicate_email_response_indistinguishable`, `Rate_limits_enforced`, `New_tenant_isolated` (API-021 matrix gains a self-signup-created tenant).

**Edge cases**: verified-but-abandoned completion (no password set) → resumable from the same link until expiry; tenant name collisions allowed (id is the key, name is display).

### API-101 · Subscriptions: plans, lifecycle, enforcement signal
Milestone M10 · Size L · Level senior · Depends: API-100, API-090

**Context**: ADR-021 — recorded billing (ADR-005 philosophy), gateway automation later.

**Spec**: `Plan` (per-box monthly price, platform fee rate, limits) + `Subscription` per tenant with lifecycle `Trial → Active → PastDue → Suspended` driven by invoice records: monthly `Invoice` rows generated by a job (amount = plan × paired boxes), platform-admin marks paid (audited). Overdue > grace days → `PastDue`; > suspend threshold → `Suspended`. Heartbeat response gains `serviceState: InService | NotInService` — `Suspended` ⇒ `NotInService`, applied by the booth only from attract (ADR-021 rule; enforcement UX is BOOTH-050). **Server-side guard, not just booth UX**: API-092's purchase creation rejects (`subscription_suspended` Problem Details) when the tenant is Suspended — a stale or tampered box must not be able to sell. In-flight purchases/sessions at suspension time still settle and deliver (never strand paid money). Fee source migration: API-093's `platformFeeIDR` switches from the M9 config-default rate to the tenant's plan rate (entries keep snapshotting at write time). Trial converts on first invoice paid.

**Tests**: `Invoice_amount_tracks_paired_box_count`, `Lifecycle_transitions_on_grace_and_suspend_thresholds`, `Suspended_heartbeat_says_notinservice`, `Suspended_tenant_purchase_create_rejected_serverside`, `Inflight_purchase_at_suspension_still_settles`, `Active_session_never_killed_by_suspension` (integration: suspend mid-session → session completes, media delivers), `Earnings_fee_rate_comes_from_plan_after_migration`, `Mark_paid_reactivates_and_audited`.

**Edge cases**: box paired mid-month → prorate next invoice (simple day-based proration, documented); suspended tenant's *gallery links keep working* (customers already paid — never punish them for the operator's bill).

### API-102 · Platform-admin surface
Milestone M10 · Size M · Level mid · Depends: API-101, API-093

**Context**: ADR-021 — Toku above tenants. New privilege tier, audited from endpoint one.

**Spec**: `platform_admin` role (seeded, never self-signup-able; JWT claim distinct from tenant admin). Endpoints: list tenants + subscription state, mark invoice paid, record payout (against a tenant's unpaid ledger balance, idempotency key), compensating earnings entry with reason, cross-tenant fleet health (device list + last heartbeat + alerts). Every endpoint audited with actor + tenant target. Tenant-scoped endpoints keep rejecting platform-admin tokens that don't impersonate — no silent god-mode reads; impersonation is explicit (`X-Acting-Tenant` + audit) or out of scope for v1 (pick explicit-header, log it).

**Tests**: `Tenant_admin_403_on_platform_endpoints`, `Platform_admin_403_on_tenant_endpoints_without_acting_header`, `Payout_reduces_unpaid_balance_idempotently`, `Compensation_requires_reason_and_audits`, `Anonymous_sweep_updated` (the API-028 public allowlist gains only `/api/signup`).

**Edge cases**: payout larger than unpaid balance → 422 (never negative balances); two concurrent payouts → idempotency key + row lock, one winner.

### API-103 · Settlement export
Milestone M10 · Size S · Level junior · Depends: API-093, API-102

**Spec**: `GET /api/platform/settlements?period=YYYY-MM` (platform-admin) and `GET /api/earnings/export` (tenant admin): CSV per tenant/store — entries, gross/fees/net, payouts applied, closing unpaid balance. Deterministic ordering, UTF-8 BOM for Excel, streamed not buffered.

**Tests**: `Csv_matches_ledger_totals`, `Balance_carries_between_periods`, `Empty_period_yields_header_only`, `Streaming_under_memory_cap` (100k-entry seed).

**Edge cases**: period boundaries in store timezone? No — UTC period boundaries, documented in the header row (consistency beats local-month niceties; revisit only if operators complain).

---

## Milestone M11 — box field-hardening (API side)

### API-110 · Payment edge-case hardening + reconciliation
Milestone M11 · Size L · Level senior · Depends: API-092, API-093

**Spec**: (1) recovery hardening — the `GET /api/box/purchases/current` endpoint **exists since API-092**; this task adds the failure tail: unrecoverable sessions transition to `RefundFlagged` (compensating ledger entry auto-drafted, operator + platform-admin notified via alert seam). (2) Nightly reconciliation job: re-query Midtrans for every non-terminal purchase older than 1h; heal missed webhooks (paid-at-gateway → run the paid path), flag mismatches. (3) Charge-expiry sweeper formalized with metrics.

**Tests**: `Crash_after_paid_recovers_same_session`, `Unrecoverable_paid_session_flags_refund_and_compensates`, `Reconciliation_heals_missed_webhook`, `Reconciliation_flags_gateway_mismatch`, `Sweeper_idempotent`.

**Edge cases**: recovery claimed twice (box restarted twice) → same session both times, idempotent; reconciliation running concurrently with a live webhook → row lock, single paid transition (reuse API-092's guarantee).

### API-111 · Operator outage + refund visibility
Milestone M11 · Size S · Level junior · Depends: API-110, API-036 (device health alerts)

**Spec**: extend device health alerts to operator-facing: box offline > threshold or `RefundFlagged` creates an alert row the operator sees (WEB-110) — the notification *channel* (WA/email) stays a Later seam. Alert acknowledge endpoint, audited.

**Tests**: `Offline_threshold_creates_operator_alert`, `Refund_flag_creates_alert`, `Ack_audited`, `Alerts_tenant_scoped`.

**Edge cases**: flapping connectivity → alert debounce (one alert per outage episode, not per missed heartbeat).

---
---

*Not in any milestone: Midtrans recurring subscriptions + Iris auto-payout (ADR-019/021 upgrades), domain-based tenant resolution (ADR-012), notification channels (WA/email — the seams exist after API-100/111, the channels don't).*
