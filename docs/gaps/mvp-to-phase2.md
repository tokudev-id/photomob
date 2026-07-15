# Gaps: MVP (M0–M8) → Phase 2 (Box pivot, M9–M11)

Status: open — dispositions assigned
Recorded: 2026-07-15
Context: all M0–M8 software milestones are implemented. This inventories what
remains open from the MVP and assigns each item a disposition under the Box
pivot ([ADR-018](../DECISIONS.md)..021, milestones M9–M11).

## 1. Hardware gates — software done, physical proof missing

| # | Gap | Disposition |
|---|---|---|
| 1 | DSLR capture + live-view certification on a real body (top M1 risk, never closed) | **Decoupled from revenue.** The Box is webcam-only (ADR-018); DSLR certification now gates only the premium staffed tier. Do before selling that tier — not before M9. |
| 2 | DNP/HiTi physical print certification (`potoku/docs/hardware-notes.md` checklist) | Same as #1 — gates the premium tier and the Box photo-upgrade tier (ADR-020), not the thermal launch. |
| 3 | E2E runbook never run against real hardware ([runbook](../runbooks/end-to-end-testing.md)) | **Run the software E2E now** (webcam + no printer) as the M9 baseline; extend the runbook with the F7 pay path during M9. |

## 2. Engineering debt from reviews — never scheduled

| # | Gap | Disposition |
|---|---|---|
| 4 | Brand config validation hand-rolled (should follow the generated-schema pattern, per API-011's lesson); zero brand/rate-limit tests | **Schedule into M9** as prep hygiene — the Box multiplies tenants and brands (signup, ADR-021), so brand config becomes hostile input. |
| 5 | PIN rate limiter in-memory per-replica | Blocks multi-replica API only. Schedule with the first scale-out, at latest M10 (signup traffic). |
| 6 | Gallery JWT shares `Delivery:SigningKey` (needs per-purpose key) | Small, security-relevant, tenant-count multiplies exposure → **fold into M10** (API-100's security pass). |
| 7 | No golden-PNG compose parity test (booth sharp ↔ web canvas) | Still worth doing; unchanged priority. BOOTH-042 adds a thermal golden fixture — do the color one in the same PR spirit. |
| 8 | `sync-client` strategy undecided; API-043 concurrency/rate-limit matrix unverified in CI | Decide sync-client before M9 (the box purchase surface will churn the contract); verify the API-043 matrix in CI during M9. |

## 3. Deliberately unscheduled items — pivot dispositions

| # | Item | Disposition |
|---|---|---|
| 9 | SaaS machinery (ADR-012 deferred) | **Promoted to core** → ADR-021, M10 (API-100..103, WEB-100..103). |
| 10 | ESC/POS printing (ADR-013 "later") | **Promoted to core** → ADR-020, M9 (BOOTH-042) — now prints the product itself, not just receipts. |
| 11 | In-booth payment | New surface, never previously planned → ADR-019, M9 (API-091..092, BOOTH-041). Reuses M5's gateway + webhook spine. |
| 12 | Notifications (WA/email) | Still deferred, but M10/M11 build the **seams** (signup verification sender, operator alerts) — only the channels stay "Later". |
| 13 | EDSDK adapter | Still deferred; premium-tier concern only. |

## 4. New gaps the pivot itself creates (tracked, not yet tasks)

- **Refund execution**: M11 flags and compensates refunds in the ledger; actually
  *sending money back* (Midtrans refund API vs manual transfer) is undecided —
  needs a small ADR before real-money launch.
- **Merchant-of-record obligations**: platform-collected QRIS (ADR-019) makes
  Toku responsible for disputes/chargebacks and possibly tax invoicing
  (e-Faktur) on session sales — business/legal homework, not code, but it gates
  going live with real money.
- **Box hardware BOM**: the Windows touch device, enclosure, mounting, thermal
  unit, and cost target per box have no doc yet — owned by BOOTH-050/051's
  provisioning work in M11, but sourcing should start during M9.
- **Trial abuse**: self-signup + trial (API-100/101) invites throwaway tenants;
  rate limits exist, but a real abuse review belongs in M10's security pass.
