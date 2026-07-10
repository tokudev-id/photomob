# PhotoMob — Architecture Decision Records

> Why it's built this way. Format: Context → Decision → Trade-off accepted. Decisions here are settled — reopen only with new evidence, not new opinions.

---

## ADR-001: Full-TypeScript monorepo, three apps (2026-07-10)

**Context**: End-to-end system (API, admin web, booth kiosk) built and operated by a very small team fluent in NestJS/React.
**Decision**: One pnpm-workspaces monorepo — `apps/api` (NestJS), `apps/web` (Next.js), `apps/booth` (Electron+React) — with `packages/shared` as the single source of truth for contracts and the template schema.
**Trade-off**: Electron app size/updater complexity vs. one language, shared types, and atomic cross-app changes. No Nx/Turbo until build times actually hurt.

## ADR-002: DSLR via digiCamControl bridge behind `ICamera` (2026-07-10)

**Context**: "DSLR integration must be achievable easily" — but official SDKs (Canon EDSDK) mean brand lock-in, a license process, and native Node bindings before the flow is proven.
**Decision**: digiCamControl's local HTTP webserver (capture, live view, settings) wrapped by a `DigiCamControlCamera` adapter. `ICamera` is the only camera surface the app knows; `WebcamCamera` (fallback/failover) and `MockCamera` (dev/CI) implement it too.
**Trade-off**: Live view is polled MJPEG (~15fps) — below EDSDK smoothness — and dCC is a companion process to babysit. Accepted because failover to webcam keeps sessions alive, and EDSDK remains a drop-in adapter if M1 validation says live view isn't good enough. **Validate live view first thing in M1.**

## ADR-003: Electron kiosk for the booth (2026-07-10)

**Context**: The booth needs fullscreen lockdown, local hardware access (dCC, hot folder, SQLite queue), crash recovery, and offline tolerance. A plain browser app (boothlev's model) can do none of that.
**Decision**: Electron with a hard main/renderer split — hardware and infra only in main, pure wizard UI in renderer, typed IPC between.
**Trade-off**: Heavier runtime and an installer to maintain vs. a Chrome-kiosk + separate "hardware bridge service" (two things to install, two things to crash). One artifact won.

## ADR-004: Compose on the booth, upload composed + raws (2026-07-10)

**Context**: The print must come out seconds after "Print!", including when Wi-Fi is down. Composition (photos + template layers) could run on booth or server.
**Decision**: `sharp` composition in the booth's main process; the composed PNG goes to the printer immediately and to the API as the canonical deliverable alongside all raw shots. The same composition function renders admin previews server-side to prevent drift.
**Trade-off**: Booth PC needs the horsepower (any modern mini-PC is fine) and template assets must be cached locally (needed for offline anyway).

## ADR-005: v1 booking is walk-in + staff-managed; online booking deferred (2026-07-10)

**Context**: Full online self-booking needs slot calendars, a payment gateway, webhooks, and refund handling — the biggest scope block in the brief, and none of it is needed to operate one store.
**Decision**: Staff creates bookings and *records* payments (cash/QRIS/transfer happen outside the system). Session codes gate the booth. Online booking + Midtrans/Xendit is a post-M4 milestone with a clean seam (new public surface + webhook handler; see ARCHITECTURE §12).
**Trade-off**: No customer self-service in v1. Accepted for speed to a production store — "success feeling first."

## ADR-006: Hot-folder printing behind `IPrinter` (2026-07-10)

**Context**: Dye-sub photo printers (DNP/HiTi) ship vendor hot-folder utilities that handle color profiles, media sizes, and 2-up strip cutting far better than raw Windows printing from Node/Electron.
**Decision**: `HotFolderPrinter` (drop print-ready PNG into a watched folder) as default; `SpoolerPrinter` (pdf-to-printer) as fallback for plain printers.
**Trade-off**: Job status is inferred, not confirmed — a silent jam needs the staff-alert + one-click reprint path rather than driver-level status. Accepted; revisit only if reprints become frequent.

## ADR-007: PostgreSQL + Prisma; money as integer IDR; `storeId` everywhere (2026-07-10)

**Context**: Bookings/payments/sessions are relational and transactional. Multi-store is a stated later ambition.
**Decision**: PostgreSQL with Prisma (schema-first, migration story fits a small team). Amounts are integer rupiah. Every domain table carries `storeId` from day one.
**Trade-off**: Prisma's query ceiling vs. TypeORM's NestJS idiom — accepted for DX and migration safety; raw SQL escape hatch exists. `storeId` now costs one column and saves a future migration crawl.

## ADR-008: One Next.js app for admin + public gallery (2026-07-10)

**Context**: The customer gallery is ~2 pages (gallery, expired). A fourth deployable just for it is infrastructure without a payoff.
**Decision**: `apps/web` with route groups — `(admin)` auth-protected, `(gallery)` public at `/g/[code]` — split into its own app only if branding/scale ever demands it.
**Trade-off**: Shared build/deploy for two audiences. Accepted at this size; the route-group seam keeps the split cheap.

## ADR-009: Coarse server-side session states + append-only event log (2026-07-10)

**Context**: The booth wizard has ~10 UI steps that will be redesigned often. Mirroring them in the DB couples the server to UI churn (a boothlev-style docs/code drift generator).
**Decision**: Server knows `CREATED → ACTIVE → COMPLETED / ABANDONED / EXPIRED` only. Everything finer is a `SessionEvent` row (capture_done, print_failed, fallback_webcam, ...) — powering after-sales debugging and dashboards without state-machine bloat.
**Trade-off**: Dashboard "where are they now" is inferred from last event, not a column. Fine.

## ADR-010: Digital delivery is expiring short-link + PIN, pre-issued (2026-07-10)

**Context**: Customers must reach their media instantly from a QR at the booth, links must expire, and media must eventually be purged. Accounts are hostile UX at a booth.
**Decision**: Delivery link (`/g/<shortCode>`, ≥50 bits entropy, 4-digit PIN, rate-limited, signed media URLs) is **pre-issued at session activation**, so the QR renders even before uploads finish ("photos on the way"). Link TTL and media retention are per-package settings; a logged daily job expires links and hard-deletes media.
**Trade-off**: A pre-issued link can briefly point at an empty gallery on slow Wi-Fi. Accepted — never trap a customer at the booth waiting for upload.

## ADR-011: Templates are versioned data + assets, never code (2026-07-10)

**Context**: The brief's core requirement — "the app is treated like a template which we can add style or frame." boothlev hardcoded 16 templates in a JS array; every new style was a deploy.
**Decision**: Template = zod-validated JSON config + PNG layers, uploaded and activated in admin, versioned immutably once referenced by a session, synced to booths with local caching. Rendering is one deterministic layer pipeline shared by booth print and admin preview.
**Trade-off**: v1 template authoring is JSON-by-hand (with live preview) — a visual designer is deferred. Accepted: admins add styles without a developer, which is the requirement; the designer is polish.
