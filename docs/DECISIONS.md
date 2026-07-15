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
**Update 2026-07-11**: Toku confirmed the system must be **camera-brand-agnostic**. This *reinforces* this decision — dCC drives Canon/Nikon/Sony through the same adapter, so multi-brand is a config concern (per-store camera profile + certified-hardware matrix), not a code concern. Brand-locked SDKs (EDSDK) stay what they were: per-brand escape hatches.

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
**Update 2026-07-11**: "DNP vs HiTi" dissolved — **both are supported**, one adapter each: DNP → `HotFolderPrinter` (official Hot Folder Print utility), HiTi → `SpoolerPrinter` (driver-based; HiTi ships no hot-folder utility, and the spooler path actually reports *real* job status). Printer brand is per-store config + a certified-hardware matrix entry, exactly like cameras (ADR-002 update). Remaining question is purchasing only: which unit to certify first in M1.

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
**Update 2026-07-11**: PIN policy settled — **store-level config, default ON** (a whitelabel brand-config field per ADR-012). The PIN's job is leaked/abandoned links (QR left on the booth table, link forwarded outside the group); brute-force is already killed by short-code entropy. Per-booking toggles rejected: one more decision per sale, inconsistent customer experience.

## ADR-011: Templates are versioned data + assets, never code (2026-07-10)

**Context**: The brief's core requirement — "the app is treated like a template which we can add style or frame." boothlev hardcoded 16 templates in a JS array; every new style was a deploy.
**Decision**: Template = zod-validated JSON config + PNG layers, uploaded and activated in admin, versioned immutably once referenced by a session, synced to booths with local caching. Rendering is one deterministic layer pipeline shared by booth print and admin preview.
**Trade-off**: v1 template authoring is JSON-by-hand (with live preview) — a visual designer is deferred. Accepted: admins add styles without a developer, which is the requirement; the designer is polish.

## ADR-012: Whitelabel base — brand is data, tenant-scoped from day one (2026-07-11)

**Context**: Toku wants PhotoMob as a base product serving **both** futures: sold as a single-brand install to a client, or run as a multi-brand SaaS. Building SaaS machinery now would violate our right-sizing stance; ignoring it would bake a migration crawl into every table and hardcoded brand string.
**Decision**: Single-tenant is just multi-tenant with one tenant row. `TENANT` sits above `STORE`; brand config (theme tokens, logo, name, fonts, copy, domain, gallery-PIN default) is zod-validated **data on the tenant**, served as a brand manifest that web and booth hydrate their themes from — zero hardcoded brand anywhere; PRD §6 "clean but fun" is the seed/default config. Standalone deployment = same artifact, one tenant row. **Deferred until a second client exists**: tenant self-signup, billing, plan limits, domain-based tenant resolution, tenant admin console.
**Trade-off**: One extra level of indirection (tenant → store) and a manifest fetch that every UI must respect from day one — cheap. In exchange, "convert to SaaS" is adding rows and a resolver, never a schema migration or brand-string hunt.

## ADR-013: Receipt printing — print-CSS + silent kiosk printing now, ESC/POS behind `IReceiptPrinter` later (2026-07-11)

**Context**: The counter prints a purchase receipt carrying the session code. Research on browser→thermal integration: QZ Tray is LGPL but **silent printing requires a paid certificate**; WebUSB/WebSerial is Chrome-only with per-device permission UX; direct ESC/POS over LAN TCP:9100 (node-thermal-printer) is clean but needs a store-local sender — and the API lives on a VPS that can't reach the store LAN.
**Decision**: v1 = the staff web app renders an 80mm print-CSS receipt; the counter Chrome runs `--kiosk-printing` for silent output through the printer's Windows driver. Works with any receipt printer (Epson TM-T82 class, Xprinter, ULTRON — all ~Rp 650rb–2.4jt locally), zero new components, zero certificates. ESC/POS direct (TCP:9100 adapter behind an `IReceiptPrinter` seam, sender = a store-local process) is the documented upgrade if driver printing proves annoying (speed, cutter control, drawer kick).
**Trade-off**: Driver dependency and no cash-drawer/cutter commands in v1. Also: the receipt carries the **session code only** — the gallery QR + PIN can't be on it because the delivery link is pre-issued at session *activation* (ADR-010), which doesn't exist at purchase time; pre-issuing at session creation (TTL anchored to completion) is a known evolution if receipts should carry the gallery link.

## ADR-014: Duration-based capture; selection count = template slot count (2026-07-11)

**Context**: Toku redefined capture: customers get a **shooting window**, not a fixed shot count — but must end by choosing what gets printed, while every shot still reaches them digitally.
**Decision**: `PACKAGE` defines shooting-window seconds + a hard shot cap (default ~30, bounds storage/upload/selection UX). The capture loop (countdown → fire → preview) repeats until the window ends or the customer taps "I'm done". The Select screen requires picking **exactly `template.slots.length`** shots (polaroid = 1, strip = 4) — the template already knows its shape, so no separate selection-count config can drift from it. "Shoot more" re-enters capture while window time remains (replaces per-slot retake). All shots — picked or not — upload and appear in the gallery.
**Trade-off**: Variable shot volume per session (upload/storage varies; the cap bounds it) and a slightly heavier Select screen (pick 4 of up to 30 vs 4 of 8). Accepted: shooting freely *is* the fun, and the cap plus film-strip UI keeps selection manageable.

## ADR-015: Hardware config is booth-local, user-selectable, heartbeat-reported (2026-07-11)

**Context**: Toku requires the booth to be configurable on the client side — staff pick the printer and the camera (DSLR via digiCamControl or any plugged webcam) from the app itself, persisted. Config files edited by developers would kill the whitelabel base promise (clients must self-serve setup).
**Decision**: The rule: **hardware config is booth-local; business config is server-owned.** A PIN-gated settings screen (hidden gesture to open, locked while a session is ACTIVE) enumerates real devices — DSLRs via dCC `list cameras`, webcams via `enumerateDevices()`, printers via Electron `getPrintersAsync()` or hot-folder path picker — and writes a zod-validated local JSON settings file (versioned, export/import for provisioning). Selection parameterizes which `ICamera`/`IPrinter` adapter is constructed; primary + fallback camera are both user-chosen. Settings include **Test capture / Test print** and a "run hardware check" sequence (capture → compose → sample print) so misbindings surface immediately and hardware certification becomes self-service. The device heartbeat carries a config snapshot so admin sees every booth's hardware remotely; config changes are logged as device events.
**Trade-off**: Webcam deviceIds are not perfectly stable across USB re-plugs — persist deviceId + label, re-resolve on boot, degrade with a health warning. Server holds a *copy* (snapshot), not the truth, for hardware — accepted: hardware bindings are physical facts of one PC, and offline booths must keep working from local truth.

## ADR-016: Stack pivot — .NET backend, polyrepo split, product codename "Potoku" (2026-07-11)

**Context**: Toku redirected the stack: backend in **.NET (ASP.NET Core)** with PostgreSQL, Redis, Docker, Clean Architecture + DDD + SOLID; development split into three repos (this repo remains **planning docs only**); platform web as **React + Vite SPA**. Supersedes the backend half of ADR-001 (full-TS monorepo) and the ORM in ADR-007 (Prisma → EF Core); PostgreSQL, integer-IDR money, storeId/tenantId scoping all stand.
**Decision**:
- **Repos**: [`potoku`](https://github.com/tokudev-id/potoku) = booth app (Electron + React, npm workspaces incl. `@potoku/template-kit`); [`potoku-api`](https://github.com/tokudev-id/potoku-api) = .NET 8 LTS backend (Domain / Application / Infrastructure / Api projects, EF Core + Npgsql, Redis behind cache/rate-limit abstractions, docker-compose); [`potoku-platform-web`](https://github.com/tokudev-id/potoku-platform-web) = React + Vite SPA (admin/client management, SaaS platform surface, public gallery `/g/:code`). `photomob` = planning docs only.
- **Cross-stack contracts**: the .NET API is the single source via OpenAPI → generated TS clients in both TS repos (replaces `packages/shared` DTOs).
- **Template schema + composition stay TypeScript** in `@potoku/template-kit`: one pure-TS **layout engine** (slot math — the drift-sensitive part) + two thin rasterizers: sharp (booth print), canvas (browser admin preview — required since a Vite SPA has no Node server). The API stores template config as opaque JSON validated against a JSON Schema generated from the zod source.
- **DDD, right-sized**: tactical patterns (aggregates, domain events) for Sessions/Bookings/Delivery; thin CRUD stays thin inside the clean layers. **No MediatR** (commercial license since 2025) — plain use-case handlers.
- **Redis earns its keep in SaaS mode**: gallery rate limiting, brand/template manifest cache, distributed locks — behind `IDistributedCache`-style abstractions so a single-store install can run without it.
- **.NET 8 LTS** now (newest SDK on the dev machine); .NET 10 LTS upgrade later is a TFM bump.
**Trade-off**: Two ecosystems (C# + TS) — shared-type superpower of ADR-001 is lost and replaced by OpenAPI codegen discipline; polyrepo means cross-repo changes are no longer atomic and `@potoku/template-kit` eventually needs publishing (GitHub Packages) for platform-web to consume — until then the SPA vendors the schema types. Accepted on Toku's direction; the seams (ICamera/IPrinter/IStorageProvider, coarse states, template-as-data) survive the pivot untouched.

## ADR-017: One-time pairing code for booth identity enrollment (2026-07-15)

**Context**: API-013 could mint a permanent booth token and BOOTH-022 could accept one, but no admin UI connected them. Copying a permanent high-entropy credential through a browser or chat made first-run setup both unusable and unnecessarily exposed.
**Decision**: An admin creates a ten-minute, store-scoped enrollment and transfers an eight-character code. The API stores a keyed hash, rate-limits claim by IP and code, row-locks consumption, creates the device atomically, and returns a deterministic high-entropy token only to the claiming booth. A random claim UUID persisted by the booth permits the same credential to be derived after a lost response; a different claimant receives `already_used`. The browser never sees the permanent token, and the booth never persists the pairing code. The first authenticated heartbeat changes the platform state from Pending to Paired. Revocation is checked on every device request. Hardware choice remains booth-local under ADR-015 and never changes identity.
**Trade-off**: Enrollment adds a short-lived table, rate limiting, and a retry identifier to a previously simple token form. The deterministic recovery credential depends on a server-side key, but avoids storing plaintext or reversibly encrypted device tokens while still making network-loss retries safe.
