# potoku-api — Task Breakdown

> .NET 8, Clean Architecture. Read [tasks/README.md](README.md) first for conventions and global DoD.
> Layer rule for every task here: domain rules in `Potoku.Domain`, orchestration in `Potoku.Application` (plain handlers — **no MediatR**), EF/Redis/IO in `Potoku.Infrastructure`, HTTP in `Potoku.Api`. A PR that leaks EF into Application gets bounced.

---

## Milestone M0 — walking skeleton

### API-001 · EF Core baseline migration + dev seed
Milestone M0 · Size S · Level junior · Depends: —

**Context**: The scaffold has a `PotokuDbContext` but no migrations. Everything else builds on this.

**Spec**
- Create initial migration for the existing model (Tenant, Store, Session, SessionEvent).
- `dotnet ef` tooling wired; migrations live in `Infrastructure/Persistence/Migrations`.
- Startup behavior: `Database.Migrate()` on boot **only** when `Database:MigrateOnStartup=true` (true in docker-compose, false in tests).
- Dev seed (idempotent, runs only when DB has zero tenants): one Tenant ("Potoku Dev", brandConfig = default theme JSON), one Store ("Dev Store").

**Steps**: add `Microsoft.EntityFrameworkCore.Tools` → `dotnet ef migrations add Initial` → seed class `DevSeeder` called from Program behind config flag → verify `docker compose up` gives a migrated, seeded DB.

**AC**
- [ ] Fresh `docker compose up` → API healthy, tables exist, seed present exactly once (restart does not duplicate).
- [ ] `dotnet ef migrations script` runs clean.
- [ ] Migration filenames reviewed — no `PendingModelChanges` warning on build.

**Tests**
- `Migrate_from_empty_database_succeeds` — Given empty Postgres (Testcontainers, see API-009), When migrate, Then all 4 tables exist.
- `Seed_is_idempotent` — When seeder runs twice, Then still exactly 1 tenant / 1 store.

**Edge cases**
- Seed must not run when any tenant exists (a wiped-then-restored prod DB must not get a "Potoku Dev" tenant).
- Concurrent boot (2 API replicas migrating simultaneously) — acceptable for now, but document the risk in the migration README section.

**Out of scope**: any new entity, auth, multi-tenant resolution.

---

### API-002 · Error model: ProblemDetails everywhere
Milestone M0 · Size S · Level junior · Depends: —

**Context**: The scaffold has an inline try/catch middleware. Replace with a real exception→response mapping so every future task inherits consistent errors.

**Spec**
- Exception middleware mapping: `DomainException` → 409 `application/problem+json`; `ValidationException`/model binding → 400 with field errors; `NotFoundException` (create it in Application) → 404; anything else → 500 with correlation id, full exception logged, **no stack trace in the body**.
- Response shape: RFC 7807 (`type`, `title`, `status`, `detail`, `traceId`).

**AC**
- [ ] `POST /api/sessions/activate` with unknown code returns 409 problem+json (manual + integration test).
- [ ] Unhandled exception returns 500 problem+json containing `traceId`; log line carries the same `traceId`.

**Tests**
- `DomainException_maps_to_409_problem_details`
- `Unknown_route_returns_404_problem_details`
- `Unhandled_exception_hides_internals_and_logs_traceId`

**Edge cases**: exception thrown *after* response started (log-only, don't crash the pipeline); cancellation (`OperationCanceledException` when client disconnects) must not be logged as error.

**Out of scope**: localization of error messages.

---

### API-003 · Create session + session-code generator
Milestone M0 · Size M · Level junior · Depends: API-001, API-002

**Context**: PRD F1 issues a session code; in M0 there are no bookings yet, so a dev/staff-less create endpoint drives the skeleton. The code generator built here is the real one M2 will reuse.

**Spec**
- Domain: `SessionCode.Generate(Random source)` → 6-digit numeric string, no leading-zero stripping ("012345" is valid and stays 6 chars).
- `CreateSessionHandler`: input `storeId`; generates a code, retries on uniqueness collision (max 5 attempts against *non-terminal* sessions of that store: status ∉ {Completed, Abandoned, Expired}), persists Session (Created).
- `POST /api/sessions` → 201 `{ sessionId, code }`. M0: unauthenticated (device token comes in API-013 — leave a `// AUTH: API-013` marker).
- DB: partial unique index on `(StoreId, Code)` WHERE status is non-terminal (raw SQL in migration; EF can't express partial indexes fluently for enum-as-string — document how you did it).

**AC**
- [ ] 201 with 6-digit code; code visible in DB.
- [ ] Two stores can hold the same code simultaneously; one store cannot (while both non-terminal).
- [ ] A completed session's code is reusable for a new session in the same store.

**Tests**
- `Generate_returns_six_digits_including_leading_zeros` (property-style: 1000 iterations, all match `^\d{6}$`).
- `Collision_retries_then_succeeds` — fake repo returns "exists" 4×, Then 5th code persisted.
- `Collision_exhaustion_throws_DomainException` — 5 collisions → 409, message actionable.
- Integration: `Same_code_allowed_across_stores`, `Duplicate_code_within_store_rejected_by_index` (simulate race by inserting directly).

**Edge cases**
- RNG must be `RandomNumberGenerator` (crypto), not `Random` — codes are guessable-by-design short, so don't make it worse.
- Index race: two concurrent creates draw the same code → one gets a unique-violation → handler must catch and retry, not 500.

**Out of scope**: booking linkage (M2), time windows (M2), rate limiting (API-028).

---

### API-004 · Media upload (idempotent, checksummed)
Milestone M0 · Size L · Level mid · Depends: API-001, API-002

**Context**: The booth uploads raws + composed via a retrying queue (PRD F3.1). Retries are the *normal case* — idempotency is the whole point of this task.

**Spec**
- `IStorageProvider` in Application: `SaveAsync(stream, relativePath, ct)`, `OpenReadAsync`, `DeleteAsync`, `ExistsAsync`. Infrastructure impl `LocalDiskStorage` rooted at config `Storage:Root` (volume in compose).
- Domain: `MediaAsset` entity — `SessionId, Kind (RawShot|Composed|Thumb), Path, Checksum (SHA-256 hex), Bytes, CreatedAt`. Session invariant: assets attach only while status is Active/Completed/Abandoned (never Created/Expired).
- `POST /api/sessions/{id}/media` — multipart: file + `kind` + `checksum` + `shotIndex?`. Behavior:
  - Recompute SHA-256 server-side; mismatch with client checksum → 422 (transfer corruption).
  - Same `(sessionId, checksum)` already stored → **200 with the existing asset id** (idempotent replay), not 409.
  - Validate: MIME ∈ {image/jpeg, image/png}, sniff magic bytes (don't trust Content-Type), size ≤ `Media:MaxBytes` (default 25 MB).
  - Store to `{storeId}/{sessionId}/{kind}-{checksum[..12]}.{ext}`; write DB row + file; on DB failure after file write, delete the file (compensate).
- `GET /api/sessions/{id}/media` → list (id, kind, bytes, createdAt) — dev convenience + gallery building block.

**Steps**: storage abstraction → domain entity + migration → handler with idempotency → controller + multipart limits → integration tests against Testcontainers + temp dir.

**AC**
- [ ] Upload → file on disk at deterministic path + DB row.
- [ ] Re-upload identical bytes → 200, same asset id, exactly one file/row.
- [ ] Checksum mismatch → 422; wrong MIME (real PDF renamed .jpg) → 415; oversize → 413.
- [ ] Upload to a `Created` session → 409 (booth must activate first).

**Tests**
- `Upload_persists_file_and_row`
- `Replay_same_checksum_returns_existing_asset` (call twice, assert 1 row, 1 file)
- `Checksum_mismatch_returns_422_and_stores_nothing`
- `Magic_byte_sniffing_rejects_fake_jpeg_415`
- `Oversize_returns_413_and_stores_nothing`
- `Upload_to_created_session_returns_409`
- `Db_failure_after_file_write_removes_orphan_file` (fault-injecting fake DbContext or repo)

**Edge cases**
- Two concurrent uploads of the same checksum: unique index on `(SessionId, Checksum)`; loser catches violation → return winner's asset (idempotent), no orphan file.
- Disk full: `IOException` → 503 problem details + log; no partial DB row.
- `shotIndex` duplicates allowed (booth may re-shoot); it's metadata, not identity.
- Path traversal: extension derived from sniffed type, never from client filename; client filename never touches the filesystem.

**Out of scope**: thumbnails (M3), S3 provider, resumable/chunked upload.

---

### API-005 · Delivery link domain — pre-issued at activation
Milestone M0 · Size M · Level mid · Depends: API-003

**Context**: ADR-010 — the QR must render before uploads finish. Activation both starts the timer *and* issues the delivery link.

**Spec**
- Domain: `DeliveryLink` — `SessionId (unique), ShortCode, PinHash, ExpiresAt, ViewCount`. `ShortCode`: 10 chars Crockford base32 from 64 random bits (crypto RNG), globally unique. PIN: 4 digits, generated always (enforcement toggled later, API-030); hash with `PasswordHasher<T>` or PBKDF2 — never store plaintext; **return plaintext PIN exactly once** in the activation response.
- Extend `ActivateSessionHandler`: on success also create DeliveryLink (TTL from config `Delivery:DefaultTtlDays` = 7 until packages exist) and return `{ sessionId, endsAt, gallery: { shortCode, pin, expiresAt } }`.
- Complete flow: `POST /api/sessions/{id}/complete` — allowed from Active/Abandoned (ADR-009), requires ≥1 Composed asset, else 409.

**AC**
- [ ] Activation response contains shortCode + plaintext PIN; DB has only the hash.
- [ ] ShortCode collision handled by regenerate-and-retry (unique index backstop).
- [ ] `complete` without a composed asset → 409 with actionable message.

**Tests**
- `Activate_issues_link_with_valid_shortcode_format` (`^[0-9A-HJKMNP-TV-Z]{10}$` — Crockford, no I/L/O/U)
- `Pin_is_hashed_at_rest_and_never_logged` (assert log sink captured nothing matching the PIN)
- `Activate_twice_conflict_still_leaves_single_link`
- `Complete_requires_composed_asset`
- `Complete_from_abandoned_succeeds` (staff recovery path, ADR-009)

**Edge cases**: activation transaction must be atomic (session status + link — one SaveChanges); if link creation fails, activation must roll back (booth will retry the whole call — verify replay after rollback works).

**Out of scope**: PIN *verification* (API-030), TTL from package (M2), link extension (API-033).

---

### API-006 · Public gallery endpoints + signed media URLs
Milestone M0 · Size L · Level mid · Depends: API-004, API-005

**Context**: PRD F3 — customer opens `/g/<code>` on their phone. Media must never be served by direct path (ARCHITECTURE §10).

**Spec**
- `GET /api/gallery/{shortCode}` → 200 `{ status: "ready"|"pending", composed: MediaRef[], raws: MediaRef[], expiresAt }`.
  - `pending` when the session is Active or has zero Composed assets yet ("photos on the way").
  - Unknown shortCode → 404. Expired (`now > ExpiresAt`) → **410 Gone** (WEB-002 renders the friendly page off 410 — contract, don't change it).
  - Each `MediaRef` = `{ id, kind, url }` where `url` is signed: `/api/media/{assetId}?exp={unix}&sig={hmac}`.
- Signing: HMAC-SHA256 over `{assetId}|{exp}` with key `Delivery:SigningKey` (min 32 bytes, fail startup if shorter). TTL 15 min. Constant-time compare.
- `GET /api/media/{assetId}` (anonymous): verify sig + exp, stream file with correct Content-Type, `Cache-Control: private, max-age=900`, `X-Robots-Tag: noindex`. Also `noindex` header on the gallery JSON.
- Increment `ViewCount` on gallery hits (best-effort, not transactional).

**AC**
- [ ] Pending → ready transition observable: gallery shows `pending`, upload composed + complete, gallery shows `ready` with media.
- [ ] Tampered `sig`/`exp` → 403; expired sig → 403; direct asset id without sig → 403.
- [ ] Expired link → 410; unknown → 404 (distinguishable).

**Tests**
- `Gallery_pending_before_composed_upload_then_ready_after`
- `Gallery_unknown_code_404`, `Gallery_expired_410`
- `Signed_url_streams_with_correct_content_type`
- `Tampered_signature_403`, `Expired_signature_403`, `Missing_signature_403`
- `Signature_for_asset_A_rejected_for_asset_B`
- `Startup_fails_with_short_signing_key`

**Edge cases**
- File missing on disk but row exists → 404 + error log (media/DB drift is an ops signal, not a 500).
- shortCode lookup must be case-insensitive on Crockford confusables? **No** — normalize to uppercase on both issue and lookup; test lowercase input works.
- Range requests: out of scope, but must not error — ignore Range, return 200 full body.

**Out of scope**: PIN gate (API-030), rate limiting (API-028), thumbnails.

---

### API-007 · Compose hardening: readiness, config validation
Milestone M0 · Size S · Level junior · Depends: API-001

**Spec**
- Split `/health/live` (always 200 if process up) and `/health/ready` (checks Postgres reachable + migrations applied + Redis *if configured*).
- compose: healthchecks point at `/health/ready`; API waits on postgres healthy (already) — verify end-to-end.
- Startup config validation (fail fast, clear message): `ConnectionStrings:Postgres` present; `Storage:Root` writable; `Delivery:SigningKey` length (with API-006).

**AC**: `docker compose up` from clean clone → ready in <60s; killing postgres flips ready to 503 within 30s. Missing signing key → process exits with a message that names the config path.

**Tests**: `Ready_fails_when_postgres_down` (Testcontainers stop), `Live_succeeds_when_postgres_down`, `Startup_throws_on_unwritable_storage_root`.

**Edge cases**: Redis absent + not configured → ready must still pass (Redis is optional, ADR-016).

---

### API-008 · OpenAPI contract + TS client generation
Milestone M0 · Size M · Level mid · Depends: API-003..006

**Context**: OpenAPI is the *only* contract with the TS repos (ADR-016). Sloppy annotations here poison two codebases.

**Spec**
- Every endpoint: typed request/response DTOs (no anonymous objects), `[ProducesResponseType]` for every status it can return (incl. 409/410/422), operation ids stable (`Sessions_Activate`, `Gallery_Get` — codegen uses them as function names).
- Enums as strings in the contract.
- `scripts/generate-client.sh`: exports `swagger.json` (via `dotnet run` + curl or Swashbuckle CLI) and runs `openapi-typescript` — output committed to a `clients/typescript/` folder in *this* repo so TS repos can copy/depend on it; CI job fails if regenerated output differs from committed (contract drift gate).
- README section: "changing an endpoint = regenerate + commit the client".

**AC**
- [ ] `swagger.json` committed; codegen deterministic (two runs, zero diff).
- [ ] CI drift gate red when an endpoint changes without regeneration (prove it in the PR by temporarily breaking it).
- [ ] Generated client compiles under `tsc --strict`.

**Tests**: contract snapshot test (`swagger.json` stable modulo intended changes); no endpoint without operation id (assert in a unit test over the generated document).

**Edge cases**: file-upload endpoints — verify multipart is expressed correctly in OpenAPI (known Swashbuckle weak spot; hand-annotate if needed).

---

### API-009 · Integration test harness (Testcontainers)
Milestone M0 · Size M · Level mid · Depends: API-001

**Context**: Every integration test above assumes this exists. Build it early; API-003..006 test against it.

**Spec**
- `tests/Potoku.Api.IntegrationTests`: `WebApplicationFactory<Program>` + Testcontainers Postgres per test class (xUnit `IAsyncLifetime` collection fixture); migrations applied once per container; each test runs in a transaction rolled back OR per-test schema truncation (pick one, document why).
- Temp-dir storage root per run, cleaned after.
- Make `Program` testable (`public partial class Program {}` marker).
- CI: Docker available; tests tagged `[Trait("Category","Integration")]` so unit tests can run without Docker.

**AC**: `dotnet test` green locally with Docker; unit-only filter green without Docker; a sample end-to-end test (create session → activate → upload → complete → gallery ready) passes and is the living skeleton spec.

**Tests**: the harness IS the deliverable; the E2E walking-skeleton test named `WalkingSkeleton_end_to_end` is required.

**Edge cases**: parallel test collections must not share containers; port collisions on CI (Testcontainers handles it — verify).

---

## Milestone M1 — real booth support

### API-010 · Template + TemplateVersion domain
Milestone M1 · Size M · Level mid · Depends: API-009

**Spec**
- `Template` (StoreId, Name, Category) + `TemplateVersion` (TemplateId, Version int asc, ConfigJson jsonb, AssetsPath, Active bool, CreatedAt). Invariants (domain-enforced + tested): version numbers strictly increasing; **a version referenced by any session is immutable** (edits create version N+1); exactly 0..1 active version per template; activating N deactivates N−1.
- `Session` gains `TemplateVersionId?` set during the booth flow (nullable until Select).
- Endpoints (admin; auth marker for API-020): `POST /api/templates` (create draft v1), `PUT /api/templates/{id}/versions/{v}` (draft only), `POST /api/templates/{id}/versions/{v}/activate`, `GET /api/templates`.

**AC**: editing a referenced version → 409 with "create a new version" message; activation flips exactly one active flag (transactional).

**Tests**: `Edit_referenced_version_409`, `Activate_deactivates_previous`, `Version_numbers_monotonic`, `Draft_edit_succeeds`.

**Edge cases**: concurrent activation of v2 and v3 → last-write wins but never two actives (unique filtered index on `(TemplateId, Active) WHERE Active`); deleting a template with sessions → forbidden (soft-archive flag instead).

### API-011 · Template config validation via generated JSON Schema
Milestone M1 · Size M · Level mid · Depends: API-010; cross-repo: template-kit exports `schema.json` (BOOTH-002 ships it)

**Spec**
- `@potoku/template-kit` build emits `template-config.schema.json` (zod → JSON Schema). Copy is versioned into this repo at `contracts/template-config.schema.json` with the kit version noted.
- Validate `ConfigJson` on create/update with `JsonSchema.Net`; violations → 422 listing JSON-pointer paths. Extra semantic checks the schema can't express: slots within canvas bounds; `variants[].id` unique; referenced asset filenames exist under `AssetsPath` at *activation* time (draft may dangle).
- **Never** model template config as C# properties (ADR-016) — jsonb in, jsonb out.

**AC**: invalid config (slot out of bounds / missing frame) → 422 with pointer paths; valid sample from the PRD §F5 example passes bit-for-bit.

**Tests**: `Valid_prd_example_passes`, `Slot_out_of_canvas_422`, `Duplicate_variant_ids_422`, `Activation_with_missing_asset_file_409`, `Schema_file_matches_kit_version_note` (guard test reminding humans to sync).

**Edge cases**: schema evolution — an old stored version whose config no longer matches a *newer* schema must still be readable (validate only on write, never on read).

### API-012 · Template assets + booth sync manifest
Milestone M1 · Size M · Level mid · Depends: API-010, API-011

**Spec**
- `POST /api/templates/{id}/versions/{v}/assets` (multipart, PNG only, magic-byte sniffed, ≤ 15 MB each) stored under `templates/{templateId}/v{n}/`.
- `GET /api/booth/templates/manifest` (device-scoped once API-013 lands): all *active* versions for the device's store → `{ templateVersionId, templateId, version, category, configJson, assets: [{ file, sha256, url }] }`; manifest itself carries a top-level hash so the booth can short-circuit "nothing changed".
- Asset download endpoint with checksum header; long cache OK (versioned paths are immutable).

**AC**: manifest hash stable across calls when nothing changed; changes when a version activates. Booth can fully reconstruct a template from manifest + downloads (verified by BOOTH-013's integration fixture).

**Tests**: `Manifest_hash_changes_on_activation`, `Assets_immutable_after_activation_409`, `Manifest_excludes_drafts_and_inactive`.

**Edge cases**: version activated *between* a booth's manifest fetch and asset download → old version's assets must still download (immutability makes this safe — test it).

### API-013 · Device registration + device-token auth
Milestone M1 · Size L · Level mid · Depends: API-009

**Spec**
- `Device` entity: StoreId, Name, TokenHash, Status (Active/Revoked), LastSeenAt, HardwareSnapshotJson (ADR-015).
- Admin creates a device → response shows plaintext token **once** (`ptk_` + 32 random bytes base64url); stored hashed (SHA-256 — high-entropy token, no need for slow hash; document the reasoning).
- Auth: `Authorization: Device ptk_...` → authentication handler resolves device, sets claims (deviceId, storeId); policy `BoothOnly` guards `/api/booth/*`, sessions create/activate/media/events, manifest. Revoked/unknown → 401; wrong store's resource → 404 (not 403 — don't leak existence).
- Heartbeat: `POST /api/booth/heartbeat` `{ health: {...}, hardwareConfig: {...} }` → updates LastSeenAt + snapshot.

**AC**: all booth endpoints 401 without token; token grants only its own store's data; revocation effective on next request (no caching bug).

**Tests**: `Missing_token_401`, `Revoked_token_401`, `Cross_store_session_access_404`, `Heartbeat_updates_lastseen_and_snapshot`, `Token_shown_once_and_stored_hashed`.

**Edge cases**: constant-time hash compare; token in logs — assert never logged (log-sink test like API-005); clock skew doesn't matter (no expiry on device tokens; revocation is the kill switch — documented).

### API-014 · Session event ingestion (append-only, idempotent batch)
Milestone M1 · Size M · Level junior · Depends: API-013

**Spec**
- `POST /api/booth/sessions/{id}/events` — batch `[{ clientEventId (uuid), kind, at, payloadJson? }]`. Dedupe on `(SessionId, ClientEventId)` unique index; replays return 200 with per-event `accepted|duplicate`. Kinds are free strings ≤64 chars (ADR-009) — do NOT enum them.
- Events accepted for sessions in any status except Created (booth events imply activation happened) — Expired sessions still accept late-arriving uploads' events (offline booth catching up).

**AC**: batch replay is fully idempotent; 500-event batch < 1s locally.

**Tests**: `Batch_dedupes_by_client_event_id`, `Replay_returns_duplicate_markers`, `Events_on_expired_session_accepted`, `Event_on_created_session_409`, `Oversized_kind_422`.

**Edge cases**: `at` timestamps from a booth with wrong clock — store as given, also store server `ReceivedAt`; dashboards use ReceivedAt when `at` is implausible (>24h skew). Batch partially invalid → reject whole batch 422 (booth retries whole batch; partial acceptance breaks its queue semantics).

### API-015 · Package-driven session parameters
Milestone M1 · Size S · Level junior · Depends: API-003 (full packages arrive M2)

**Spec**: interim `Package` entity (StoreId, Name, PriceIdr int, SessionDurationSec, ShootingWindowSec, ShotCap, LinkTtlDays, RetentionDays, AllowedCategories) + seed one "Dev Basic" (10 min / 3 min window / cap 30 / TTL 7 / retention 30). Session stores `PackageId`; activation reads duration from package (drop the request's `DurationMinutes` — **breaking contract change, regenerate client**, API-008 gate will catch it). Booth needs window/cap: include in activation response.

**AC**: activation response now `{ sessionId, endsAt, shootingWindowSec, shotCap, gallery: {...} }`; client regenerated + committed.

**Tests**: `Activation_uses_package_duration`, `Response_contains_window_and_cap`.

**Edge cases**: package deleted after session created → activation still works (snapshot values onto Session at create; don't join at activate).

---

## Milestone M2 — store operations

### API-020 · Staff auth: users, roles, JWT + refresh
Milestone M2 · Size L · Level mid · Depends: API-009

**Spec**
- `User`: TenantId, StoreId?, Email (unique per tenant), PasswordHash (ASP.NET `PasswordHasher`), Role (Admin|Staff), Status.
- `POST /api/auth/login` → access JWT (15 min; claims: userId, tenantId, storeId, role) + refresh token (30 days, opaque, hashed at rest, **rotating**: each refresh invalidates the old one; reuse of a rotated token revokes the whole family — token-theft tripwire).
- `POST /api/auth/refresh`, `POST /api/auth/logout` (revokes family).
- Policies: `AdminOnly`, `StaffOrAdmin`. Apply to template/package/user admin endpoints retroactively (sweep API-010..012 markers).
- Login rate limit: 5 failures / 15 min per email+IP → 429 (in-memory now; Redis via API-028's limiter if present).

**AC**: expired access token → 401 with `WWW-Authenticate`; refresh rotation works; reused rotated token kills the family (integration-tested); no endpoint from the M0/M1 sweep remains anonymous except gallery/media/health.

**Tests**: `Login_wrong_password_401_and_rate_limited_429_after_5`, `Refresh_rotates_and_old_token_dies`, `Rotated_token_reuse_revokes_family`, `Staff_cannot_create_users_403`, `Anonymous_sweep_test` (walks OpenAPI doc, asserts every path has auth metadata or an explicit allowlist entry — this test is the guard for future endpoints).

**Edge cases**: user disabled mid-session → next request 401 (check status in validation, not just signature); clock skew ±2 min tolerated on `exp`; password hash upgrades (rehash-on-login when iteration count changes — document).

### API-021 · Tenant/store scoping enforcement
Milestone M2 · Size M · Level mid · Depends: API-020

**Spec**: `ICurrentContext` (tenantId, storeId?, role) from claims; EF **global query filters** on TenantId for every tenant-owned entity (via Store join or denormalized TenantId — denormalize, it's simpler and index-friendly); writes stamp ids from context, never from request bodies. Admin with no storeId sees all stores of the tenant; staff locked to their store. Cross-tenant access returns 404.

**AC**: a request can never read/write another tenant's rows even with a forged id in the URL (integration test with two seeded tenants is the proof); no handler contains a manual `Where(x => x.TenantId ==)` — filters do it (review checklist).

**Tests**: `Cross_tenant_read_is_404_for_every_resource` (parameterized over: sessions, templates, packages, bookings, devices, users), `Staff_cannot_touch_other_store`, `Body_supplied_storeId_is_ignored`.

**Edge cases**: background jobs (purge) run without a user context — they must bypass filters explicitly via `IgnoreQueryFilters()` with a justifying comment; gallery endpoints are tenant-less by design (shortCode is the capability) — assert they still work.

### API-022 · Packages CRUD (admin)
Milestone M2 · Size S · Level junior · Depends: API-020, API-021, API-015

**Spec**: full CRUD; validation: price ≥ 0 int IDR, 60 ≤ duration ≤ 3600s, 30 ≤ window ≤ duration, 1 ≤ shotCap ≤ 100, TTL 1..90 days, retention ≥ TTL, categories ⊆ known enum. Soft-delete (archived) — bookings reference packages historically.

**Tests**: validation matrix (each bound in/out), `Archived_package_hidden_from_staff_booking_flow_but_visible_on_old_bookings`, `Retention_shorter_than_ttl_422`.

**Edge cases**: editing a package does NOT retro-change existing sessions (values were snapshotted, API-015) — regression test.

### API-023 · Bookings + payment recording
Milestone M2 · Size L · Level mid · Depends: API-022

**Spec**
- `Booking`: StoreId, CustomerName, CustomerPhone, PackageId, PartySize, ScheduledAt?, Status (Confirmed|CheckedIn|Completed|Cancelled|NoShow) + `Payment` child rows (AmountIdr, Method cash|qris|transfer, RecordedBy, RecordedAt) — payments are records, not gateway objects (ADR-005).
- Rules (domain): booking confirms only when Σ payments ≥ package price; check-in requires Confirmed **or** staff override with mandatory reason → audit row (API-026); cancel allowed until CheckedIn; phone normalized E.164-ish ID format (+62 rewrite of leading 0).
- Endpoints: create booking (walk-in defaults ScheduledAt=now), add payment, cancel, list/search (by date range, name, phone — indexed), get one.

**AC**: underpaid booking cannot check in (409); override path writes audit with actor + reason; walk-in create→pay→confirmed in ≤2 calls.

**Tests**: `Confirm_requires_full_payment`, `Partial_then_final_payment_confirms`, `Checkin_without_payment_needs_override_reason`, `Cancel_after_checkin_409`, `Phone_08xx_normalized_to_628xx`, `Search_by_phone_finds_normalized`, `Overpayment_allowed_and_flagged_in_response` (tips happen — record, don't block).

**Edge cases**: duplicate rapid "add payment" clicks → idempotency key on payment create (client-generated uuid, same dedupe pattern as API-014); scheduled booking for yesterday → 422; NoShow transition is staff/manual or purge-job (API-032 marks stale Confirmed+scheduled past +grace).

### API-024 · Booking → session code issuance
Milestone M2 · Size M · Level mid · Depends: API-023, API-003

**Spec**: `POST /api/bookings/{id}/issue-code` — allowed on Confirmed; creates Session (Created) bound to booking with **activation window**: `[ScheduledAt − 15 min, ScheduledAt + package.duration + 60 min grace]` for scheduled, `[now, now + 24h]` for walk-in; re-issue allowed while previous session still Created (voids it → Expired + event) — "customer lost the receipt" path.

**Tests**: `Issue_on_unconfirmed_409`, `Reissue_voids_previous_created_session`, `Reissue_after_activation_409` (session running — use after-sales instead), `Window_computed_for_scheduled_and_walkin`.

**Edge cases**: booking cancelled after code issued → session must be voided in the same transaction; two staff issuing simultaneously → one wins (unique open-session-per-booking index).

### API-025 · Booth activation honors bookings + windows
Milestone M2 · Size M · Level mid · Depends: API-024, API-013

**Spec**: rework activate: lookup by `(deviceStore, code)`; checks in order (each distinct 4xx + machine-readable `reason` code for the booth UI): unknown code → 404 `code_unknown`; **booking cancelled → 409 `booking_cancelled` (checked first among the 409s — cancelling voids the session per API-024, so `already_used`/`outside_window` would misreport a code the customer never used)**; outside window → 409 `outside_window` (+`windowStartsAt` so the booth can say "come back at 14:45"); already used → 409 `already_used`. Success stamps `CheckedIn` on the booking.

**AC**: reason codes in OpenAPI as string enum (booth switches on them — BOOTH-021 contract); client regenerated.

**Tests**: one per reason code; `Activation_checks_in_booking`; `Early_activation_includes_windowStartsAt`.

**Edge cases**: device from store A, code from store B → 404 (scoping, API-021 proves it); clock: window math in UTC against `ScheduledAt` stored UTC — DST-free Indonesia, but test a +8 store timezone display value anyway (server logic must not use local time).

### API-026 · Audit log
Milestone M2 · Size S · Level junior · Depends: API-020

**Spec**: `AuditEntry` (TenantId, ActorUserId, Action string, TargetType, TargetId, ReasonText?, DataJson?, At). Write-only via `IAuditWriter`; wired into: check-in override, link resend/extend (M3), retention override, user role changes, device revoke, template activation. `GET /api/audit` admin-only, filter by target/date.

**Tests**: `Override_checkin_writes_audit_with_reason`, `Audit_is_append_only` (no update/delete endpoint exists — assert via OpenAPI walk).

**Edge cases**: audit write failure must fail the parent action (same transaction) — an unaudited override is worse than a failed one.

### API-027 · Brand manifest (whitelabel surface)
Milestone M2 · Size M · Level mid · Depends: API-021; cross-repo: token contract with WEB-025/BOOTH renderer

**Spec**: `GET /api/brand` (anonymous, resolved: single-tenant install → the tenant; else by `Host` header against `Tenant.Domain`, fallback default) → `{ name, logoUrl, colors: {...}, fonts: {...}, copy: {...}, galleryPinRequired }`; ETag + 5 min cache (Redis if present, memory else). `PUT /api/admin/brand` (admin) validates against brand-config JSON Schema (same generated-from-zod pattern as API-011).

**Tests**: `Etag_304_roundtrip`, `Unknown_host_returns_default_brand`, `Invalid_brand_config_422_with_pointers`, `Cache_invalidated_on_update`.

**Edge cases**: logo asset upload (PNG/SVG — sanitize SVG or forbid it; **forbid SVG v1**, document why: script injection); manifest must never 500 the booth — on any resolution error return default brand + error log.

### API-028 · Rate limiting (gallery + auth)
Milestone M2 · Size M · Level mid · Depends: API-006, API-020

**Spec**: ASP.NET `RateLimiter` middleware; policies: gallery lookup 30/min/IP, media 120/min/IP, login per API-020, activation 10/min/device. Redis-backed sliding window when Redis configured (shared across replicas), in-memory fallback; 429 + `Retry-After`.

**Tests**: `Gallery_31st_request_in_minute_429`, `Limits_are_per_ip_not_global`, `Redis_and_memory_paths_both_enforced` (parameterized fixture).

**Edge cases**: X-Forwarded-For only trusted from the reverse proxy (configure `ForwardedHeaders` narrowly — spoofed XFF must not bypass); health endpoints exempt.

---

## Milestone M3 — delivery & after-sales

### API-030 · Gallery PIN enforcement (store-config default ON)
Milestone M3 · Size M · Level mid · Depends: API-006, API-027

**Spec**: when `galleryPinRequired` (brand config, ADR-010 update): gallery GET without valid PIN → 401 `pin_required`; `POST /api/gallery/{shortCode}/unlock` `{ pin }` → short-lived gallery access token (JWT, 15 min, claims: shortCode) accepted via header for gallery + signed-URL issuance. 5 wrong PINs / 15 min / shortCode+IP → 429 lockout. Constant-time compare on hash.

**Tests**: `Pin_off_store_needs_no_pin`, `Wrong_pin_401_then_lockout_429`, `Unlock_token_scoped_to_its_shortcode_only`, `Expired_unlock_token_401`.

**Edge cases**: PIN entry UX needs "how many attempts left" → include `attemptsRemaining` in 401 body; toggling store config OFF must immediately admit previously locked galleries.

### API-031 · Link expiry + friendly states
Milestone M3 · Size S · Level junior · Depends: API-005, API-015

**Spec**: TTL from package snapshot at issue time (replace API-005's config default); hourly `IHostedService` sweep marks nothing (expiry is computed from `ExpiresAt` at read time — job only emits metrics/logs of newly-expired count). 410 body includes `{ storeName, storeContact }` from brand — "ask the store".

**Tests**: `Ttl_snapshot_from_package`, `Read_after_expiry_410_with_store_contact`.

**Edge cases**: extension (API-033) after expiry must resurrect the link (410 → 200) — expiry is a timestamp, not a state machine.

### API-032 · Retention purge job
Milestone M3 · Size L · Level mid · Depends: API-004, API-015

**Spec**: daily job (config hour, default 04:00 store-local → compute UTC): hard-delete media files + rows past `RetentionDays` (session-package snapshot); then delete empty session dirs; write `PurgeRun` row (started, finished, deletedCount, bytesFreed, errorsCount) — the log IS the feature (PRD F3.5). Batch of 500/loop with cancellation support; file-delete failure → log, skip, retry next run (row stays until file confirmed gone — never orphan a row-less file).
Also: mark stale Confirmed scheduled bookings NoShow (grace 24h) and Created sessions past window → Expired.

**Tests**: `Purges_only_past_retention`, `Row_survives_if_file_delete_fails_then_purged_next_run`, `Purge_run_row_written_with_counts`, `Stale_booking_marked_noshow`, `Job_is_idempotent_when_rerun`.

**Edge cases**: retention extended by staff (API-033) mid-window → purge respects the *extended* date (read live value, snapshot is the floor not ceiling — decide + document: extension writes new RetentionUntil on session, purge honors max(snapshot, override)); disk race with concurrent download → delete after closing readers is OS-handled (POSIX) but Windows file locks → catch IOException, retry next run.

### API-033 · After-sales operations
Milestone M3 · Size L · Level mid · Depends: API-023, API-030, API-026

**Spec**: staff endpoints — find sessions (`?code=|phone=|name=|date=`, joins booking, paginated); `POST /sessions/{id}/resend-link` (regenerate shortCode+PIN, old code dies, audit); `POST /sessions/{id}/extend-link` (`{ days ≤ 30 }`, audit); `POST /sessions/{id}/recover` (Abandoned→Completed if ≥1 asset, issue link if missing — ADR-009 recovery); `POST /sessions/{id}/reprint` → enqueue print job (API-034).

**Tests**: `Resend_invalidates_old_shortcode` (old → 404, new → 200), `Extend_past_expiry_resurrects_410_to_200`, `Recover_abandoned_with_assets_completes_and_issues_link`, `Recover_without_assets_409`, `Every_action_audited` (parameterized).

**Edge cases**: resend on an Active session (customer still in booth) → allowed but flagged in response (staff may be pranked — give them the info); search by phone uses normalized form (API-023).

### API-034 · Booth print-job queue
Milestone M3 · Size M · Level mid · Depends: API-013, API-033

**Spec**: `PrintJob` (StoreId, SessionId, MediaAssetId, Status Queued|PickedUp|Done|Failed, RequestedBy, attempts); booth polls `GET /api/booth/print-jobs?limit=5` → atomically marks PickedUp (row-lock, `FOR UPDATE SKIP LOCKED`); booth reports `POST /api/booth/print-jobs/{id}/result` `{ ok, detail }`; Failed with attempts <3 → back to Queued.

**Tests**: `Two_booths_polling_never_get_same_job` (parallel integration test), `Failed_job_requeues_max_3`, `Result_on_foreign_job_404`.

**Edge cases**: booth dies after pickup → jobs PickedUp >10 min revert to Queued (sweep in API-032's job); duplicate result posts → idempotent (status already terminal → 200 no-op).

### API-035 · Dashboard endpoints
Milestone M3 · Size M · Level junior · Depends: API-023, API-014, API-040 (device health optional)

**Spec**: `GET /api/dashboard/today` → sessions (by status), revenue (Σ payments today, int IDR), completion rate, popular templates (7-day window), device tiles (LastSeenAt, health snapshot). All queries single round-trip each, indexed, `AsNoTracking`; response < 300 ms on 100k-session dataset (seeded perf test — generous but bounded).

**Tests**: `Revenue_sums_only_today_store_tz`, `Completion_rate_ignores_expired`, `Perf_guard_under_300ms_on_seeded_data` (Category=Perf, not in default run).

**Edge cases**: store timezone for "today" — the ONE place local time is allowed; take `tz` query param from the client, compute UTC range server-side; empty store (no data) returns zeros, not 404s/nulls.

### API-036 · Public-surface security hardening pass
Milestone M3 · Size S · Level mid · Depends: API-006, API-030

**Spec**: verify/enforce as tests, fix gaps: noindex on all public responses; CORS locked to platform-web origin(s) config; HSTS behind proxy; security headers (nosniff, frame-deny); gallery/media/unlock covered by rate limits; PIN/token/shortCode never in logs (extend the log-sink assertion to a shared fixture).

**AC/Tests**: `Security_headers_present_on_public_endpoints` (parameterized), `Cors_rejects_unknown_origin`, `Sensitive_values_never_logged_e2e`.

---

## Milestone M4 — hardening

### API-040 · Device health alerts
Milestone M4 · Size M · Level mid · Depends: API-013

**Spec**: heartbeat gap > 3 min → device Offline (computed, not stored state); health snapshot deltas (camera ok→fail, printer ok→fail, disk <10%) create `DeviceAlert` rows; `GET /api/admin/alerts` + acknowledged flag. (Push channels — email/WA — out of scope, task the integration later.)

**Tests**: `Gap_marks_offline`, `Health_transition_creates_single_alert_not_spam` (flapping camera → alert on transition only), `Ack_hides_from_default_list`.

**Edge cases**: booth clock skew doesn't matter (offline computed from server ReceivedAt).

### API-041 · Backup & restore runbook + compose service
Milestone M4 · Size S · Level junior · Depends: API-007

**Spec**: compose sidecar cron `pg_dump` nightly to `backups/` volume + media rsync target doc; `docs/runbooks/restore.md` with a **tested** restore procedure (restore into a fresh container, walking-skeleton E2E passes against it — that test is the AC).

**Tests**: scripted `Restore_smoke` (bash, CI nightly job, not xUnit).

**Edge cases**: backup while purge job runs — pg_dump is transactional, fine; document media/db snapshot skew (media rsync after dump → possible orphan files, acceptable, purge cleans).

### API-042 · Upload resilience verification (chaos pass)
Milestone M4 · Size M · Level mid · Depends: API-004, API-014, BOOTH-004 merged

**Spec**: scripted chaos suite against compose: kill API mid-upload, kill Postgres mid-upload, duplicate every booth request ×2 — assert: zero lost media (every checksum the booth reports eventually has a row+file), zero duplicates, gallery consistent. Deliverable = repeatable script + CI nightly + a short findings doc; bugs found become tasks.

**AC**: suite green 3 consecutive nightly runs.

### API-043 · One-time booth device enrollment
Milestone M4 · Size L · Level senior · Depends: API-013, API-020, API-028; cross-repo: WEB-042, BOOTH-022

**Spec**: admin creates a ten-minute `DeviceEnrollment` for an active tenant store and receives an eight-character code once; anonymous claim is IP- and code-rate-limited, row-locked, atomically creates the device, stores only keyed hashes, and returns the permanent token only to the booth. A persisted booth claim UUID makes a lost-response retry derive the same credential without creating another device. Admin can cancel enrollments and revoke or rotate tokens. Enrollment list states are Pending, Paired (first heartbeat received and online), Offline, Expired, Cancelled, and Revoked. OpenAPI and both TS client snapshots change together.

**Tests**: authorization and store scope; expiry boundary; cancellation; concurrent one-winner claims; same-claim retry; different-claim rejection; per-IP and per-code limits; revoked token rejected on next request; no plaintext code/token at rest or in audit payloads.

**Edge cases**: a claimed booth that loses the HTTP response re-enters the same human code and reuses its locally persisted claim UUID; a different booth never receives that credential. Camera selection is not part of enrollment and never changes identity (ADR-015, ADR-017).

---

## Milestone M5 — online booking & payment gateway

> Gate: M5 starts only after M4 is green (API-042 chaos suite, 3 consecutive nightly runs). ADR-005's seam opens here: a **new public surface + webhook handler**; booth, sessions, delivery stay untouched (§12).

### API-050 · Public booking surface: availability + pending bookings
Milestone M5 · Size L · Level mid · Depends: API-023, API-024, API-028

**Context**: ADR-005 deferred exactly this. Online bookings converge into the *same* Booking aggregate staff use (API-023) — a second booking model would fork every downstream rule.

**Spec**
- Store gains `OnlineBookingConfig` (jsonb, zod → JSON Schema pattern like brand config, API-027): `enabled`, per-weekday open hours, `slotGranularityMin` (default 30), `capacityPerSlot` (default 1 — simultaneous sessions the store absorbs), `horizonDays` (default 7). Admin endpoint to edit, 422 with pointers on violations.
- Public, anonymous, rate-limited (API-028 limiter, new policy 20/min/IP):
  - `GET /api/public/booking/packages` → active packages, display fields only (name, priceIdr, duration, prints) — snapshot-test that no internal fields leak.
  - `GET /api/public/booking/availability?date=` → slots within the horizon: `{ startsAt, available }`. Available = count of bookings overlapping the slot (status ∈ {PendingPayment, Confirmed, CheckedIn}, overlap window = package session duration) < capacity.
  - `POST /api/public/booking` `{ packageId, slotStartsAt, name, phone, partySize, draftKey }` → Booking status **PendingPayment**, `PaymentDeadline = now + Booking:PaymentTtlMin` (default 15), price snapshotted (API-015 discipline extended to bookings); returns `{ statusToken }` — 128-bit random capability, **hashed at rest** (refresh-token pattern, API-020). `draftKey` = idempotency key (API-014 dedupe pattern).
- Booking gains: `Source (Staff|Online)`, status `PendingPayment`, `PaymentDeadline?`, `StatusTokenHash?`. Staff walk-ins never enter PendingPayment — their flow is byte-for-byte unchanged (regression tests prove it).
- Capacity check transactional: advisory lock on `(storeId, slotStartsAt)` around check+insert — two customers racing the last slot → one wins, the other gets 409 `slot_taken` (machine-readable reason, WEB-050 switches on it — same contract style as API-025).

**AC**
- [ ] Concurrent creates on the last slot → exactly one PendingPayment, the other 409 `slot_taken` (parallel integration test).
- [ ] `enabled=false` store → all public booking endpoints 404.
- [ ] A PendingPayment booking holds its slot (availability reflects it until deadline).

**Tests**: `Availability_excludes_full_slots`, `Pending_booking_holds_slot`, `Concurrent_last_slot_one_winner`, `Disabled_store_404`, `DraftKey_replay_returns_same_booking`, `Phone_normalized_same_as_staff_path`, `Public_package_dto_leaks_nothing` (snapshot), `Staff_walkin_flow_unchanged`.

**Edge cases**: slot in the past → 422; partySize bounds from config; status token unknown → generic 404 (enumeration guard); tokens never logged (extend the shared log-sink fixture, API-036). Slot math in UTC; store hours interpreted in store tz — the one local-time spot, same rule as API-035.

**Out of scope**: payment (API-051..052), notification channels (WA/email — unscheduled, see footer), any staff-UI change (WEB-053 handles display).

---

### API-051 · `IPaymentGateway` + checkout creation
Milestone M5 · Size L · Level mid · Depends: API-050

**Context**: §12 — "payment gains gateway fields". The gateway collects the **customer's session payment for online bookings** — nothing else (SaaS billing is ADR-012-deferred and out of scope). Provider confirmed by Toku 2026-07-12: **Midtrans first** (Snap); Xendit stays a possible second adapter behind the same interface.

**Spec**
- `IPaymentGateway` (Application): `CreateCheckoutAsync(booking) → { provider, mode: Redirect|Popup, url?, token?, scriptUrl?, gatewayOrderId, expiresAt }`; `ParseAndVerifyNotificationAsync(request) → GatewayEvent { gatewayOrderId, eventId, status: Settled|Pending|Expired|Cancelled|Refunded, grossAmountIdr, raw }`; `QueryOrderStatusAsync(gatewayOrderId)`; `CancelOrderAsync(gatewayOrderId)`. One Infrastructure adapter per provider; config `Payments:Provider` + keys from env — startup fails if enabled without keys (API-007 pattern).
- `Payment` gains `Source (Staff|Gateway)`, `Provider?`, `GatewayOrderId?`, `GatewayEventId?`, `GatewayStatus?`. Gateway payments are **never** creatable via the staff add-payment endpoint (domain rule + test) — ADR-005's "payments are records" stays true for staff; the gateway path is the only writer of gateway rows.
- `POST /api/public/booking/{statusToken}/checkout` → creates/returns the gateway transaction; **idempotent while the deadline is live** (re-request returns the same open checkout, never a duplicate order); after deadline → 410. `gatewayOrderId` = bookingId + attempt suffix — never reuse an order id after a terminal gateway state (provider constraint, document it).
- Amount = the booking's snapshotted price, integer IDR only — adapter hard-fails on any other currency.

**AC**: checkout payload is gateway-agnostic (WEB-051 renders it without knowing the provider — no provider names in the OpenAPI contract beyond the enum); sandbox roundtrip documented in the repo README (manual checklist).

**Tests**: `Checkout_idempotent_while_pending`, `Checkout_after_deadline_410`, `Staff_endpoint_rejects_gateway_source`, `Startup_fails_when_enabled_without_keys`, adapter units against recorded sandbox fixtures: `Signature_computed_correctly`, `Amount_maps_integer_idr`, `Non_idr_currency_throws`.

**Edge cases**: package price edited after booking created → checkout uses the snapshot (test it); checkout requested for an already-Confirmed booking → 409 `already_paid`.

---

### API-052 · Gateway webhook handler
Milestone M5 · Size L · Level mid · Depends: API-051

**Context**: The webhook is the **only** trusted path to "paid" — the customer's redirect back to the SPA is UX, never truth.

**Spec**
- `POST /api/webhooks/payments/{provider}` — anonymous, exempt from IP rate limits (gateways burst; the signature is the gate): verify signature per adapter → forged = 401, nothing written; dedupe on `GatewayEventId` unique index → replays return 200 (gateways retry until they see 2xx).
- **Verify-then-trust**: on Settled, re-query the gateway's status API (`QueryOrderStatusAsync`) — never trust the callback body alone (spoofed-notification defense). Amount mismatch vs the booking's snapshot → do NOT confirm: write an alert + audit row (API-026), hold for manual resolution.
- Settled → in **one transaction**: gateway Payment row + booking `PendingPayment→Confirmed` (reuses API-023's Σ payments ≥ price rule) + session code issued (API-024 handler; activation window from the scheduled slot). Expire/Cancel events → booking `Expired` (slot frees itself via the availability math).
- `GET /api/public/booking/{statusToken}` → status, and once Confirmed: session code + slot + store display info. This is WEB-052's poll target — code appears **only** on Confirmed.
- Out-of-order events (Expire arriving after Settle): terminal-precedence — Settled wins, the late event is logged and ignored.

**AC**
- [ ] Sandbox end-to-end: create → checkout → pay → webhook → status shows Confirmed + code (manual checklist recorded in PR).
- [ ] Same eventId replayed ×5 → exactly one Payment row.
- [ ] Forged signature → 401 and zero writes.

**Tests**: `Settlement_confirms_and_issues_code_atomically`, `Replay_idempotent_by_event_id`, `Forged_signature_rejected_writes_nothing`, `Amount_mismatch_holds_with_alert_never_confirms`, `Late_expire_after_settle_ignored`, `Status_exposes_code_only_when_confirmed`, `Unknown_order_returns_200_with_warn_log`.

**Edge cases**: webhook for an unknown order → 200 + warn log (a 4xx makes the gateway retry forever and drowns ops); webhook arriving before the checkout HTTP response returned → safe because the order row commits before the payload is returned (assert the ordering); notification body stored raw on the Payment row for forensics.

---

### API-053 · Pending-expiry sweep + lifecycle glue
Milestone M5 · Size M · Level junior · Depends: API-050, API-052, API-032

**Spec**
- Frequent hosted-service sweep (every minute, joins the API-032 job family): `PendingPayment` past `PaymentDeadline` → `Expired` + event; gateway order cancelled best-effort (`CancelOrderAsync` — failure logged, never fatal; the gateway's own expiry is the backstop).
- Late settlement AFTER local expiry (customer paid at 15:01): booking stays Expired, the payment is still recorded + `requires_attention` alert/audit — staff resolves; refund is outside the system in v1 (document).
- Dashboard (API-035) gains online counters: pending now, and today's revenue split gateway vs staff-recorded.
- New endpoints/enums → client regenerated + committed (API-008 drift gate).

**Tests**: `Sweep_expires_past_deadline_only`, `Expired_slot_reappears_in_availability`, `Sweep_idempotent_on_rerun`, `Cancel_failure_logged_not_thrown`, `Late_settlement_records_payment_flags_attention_keeps_expired`, `Dashboard_splits_gateway_vs_staff_revenue`.

**Edge cases**: settle webhook and sweep racing the same booking → row lock; first committer wins, the other observes the new state and no-ops (test both interleavings).

---

## Milestone M6 — media at scale + animated assets

### API-060 · S3-compatible `IStorageProvider`
Milestone M6 · Size L · Level mid · Depends: API-004, API-032

**Context**: §13's known risk — "single-VPS media storage: disk fills". The seam is already real in the repo: `Application/Abstractions/IStorageProvider.cs` (`SaveAsync/OpenReadAsync/DeleteAsync/ExistsAsync`) with `LocalDiskStorage` behind DI, and every stored path is already a relative object key (`{storeId}/{sessionId}/{kind}-{checksum}.ext`, `templates/{id}/v{n}/…`) — the adapter is a registration swap, zero call-site changes.

**Spec**
- `S3Storage : IStorageProvider` (AWSSDK.S3; path-style addressing option for MinIO/R2 compat); config `Storage:Provider = local|s3` + endpoint/bucket/region/keys; startup validation: bucket reachable + writable (API-007 pattern).
- Object keys mirror the existing relative paths 1:1 — no scheme translation anywhere.
- Media streaming (API-006's signed proxy) is **unchanged**: it reads through `IStorageProvider`, and proxying preserves noindex, rate limits, and HMAC signing. S3-presigned direct URLs are an explicit non-goal now (note as a later bandwidth optimization).
- compose gains a `minio` service for dev; integration tests parameterized so the whole API-004 storage matrix runs against **both** providers (Testcontainers MinIO).
- Purge job (API-032) verified against S3: delete semantics, already-deleted-object tolerance.

**AC**: walking-skeleton E2E green with `Storage:Provider=s3` against MinIO; API-004's full test matrix passes on both providers via one parameterized fixture.

**Tests**: provider-parameterized fixture over the existing storage tests, plus `Startup_fails_on_unreachable_bucket`, `Purge_tolerates_missing_object`, `Stream_from_s3_correct_content_type`, `Sdk_errors_map_to_same_storage_error_type` (503 semantics identical to local disk-full).

**Edge cases**: 25 MB upload roundtrip (SDK multipart threshold territory — assert it); slow first-byte from cold object storage must not trip the proxy's timeouts (measure, set explicit HttpClient timeout).

---

### API-061 · Local→S3 migration tool + cutover runbook
Milestone M6 · Size M · Level mid · Depends: API-060

**Spec**
- CLI under `tools/` (invoked via a `scripts/` wrapper): copies every `MediaAsset` file local→bucket, verifies SHA-256 against the DB row after upload, `--dry-run`, resumable (skips objects already present with matching checksum), final report (migrated / skipped / failed / bytes).
- `docs/runbooks/s3-cutover.md`: migrate while running on local → delta pass for writes that landed during the first pass → flip `Storage:Provider` → verify (walking-skeleton E2E + spot-check gallery) → retire the volume. Purge job paused during migration (runbook step, with the re-enable reminder).

**Tests**: `Migrates_and_verifies_checksum`, `Resume_skips_existing_matching`, `Corrupt_upload_detected_and_retried`, `Dry_run_writes_nothing`, `Report_counts_accurate`.

**Edge cases**: DB row whose file is missing on disk (drift — API-006's known edge) → report + skip, never abort the run; checksum mismatch after upload → delete the bad object, retry once, then report as failed.

---

### API-062 · Animated media kind (GIF/boomerang contract)
Milestone M6 · Size M · Level mid · Depends: API-004; cross-repo: BOOTH-030..031 produce, WEB-060 renders

**Context**: §12 — "new MEDIA_ASSET.kind + composer step + gallery tile". The API side is deliberately small: accept, store, serve — the booth encodes (BOOTH-031), the web renders (WEB-060).

**Spec**
- `MediaAsset.Kind` gains `Animated`; upload accepts content types image/gif, image/webp, video/mp4 **for that kind only** (magic-byte sniffed exactly like API-004 — stills kinds reject video and vice versa); size cap `Media:MaxAnimatedBytes` (default 50 MB).
- Gallery `MediaRef` gains `contentType` — WEB-060 switches tile rendering on it (that field is the contract; don't make the web sniff).
- Range requests: keep API-006's behavior (ignore Range, 200 full body) — mp4 playback works without ranges at these sizes; revisit only on real buffering complaints (note it).
- Session completion rule unchanged: `complete` still requires ≥1 **Composed** still (API-005) — animated is additive, never a substitute for the printable output (booth enforces the same, BOOTH-031).
- Template-kit's `animation` schema bump lands in `contracts/template-config.schema.json` (API-011 sync pattern, version note updated). OpenAPI + TS client regenerated (API-008 gate).

**Tests**: `Animated_accepts_gif_webp_mp4_only`, `Still_kind_rejects_video_415`, `Oversize_animated_413`, `Gallery_ref_carries_content_type`, `Magic_sniff_rejects_renamed_avi`, `Complete_still_requires_composed_still` (regression).

**Edge cases**: 50 MB cap vs API-004's 25 MB general cap — the multipart limit must be per-kind, not global (test both bounds); purge (API-032) and migration (API-061) treat Animated like any other kind (no special-casing to forget).

---

## Milestone M7 — multi-store operations

### API-070 · Store management CRUD + user-store assignment
Milestone M7 · Size M · Level mid · Depends: API-020, API-021

**Context**: §12 — the data model has been multi-store since day one (`StoreId` on every row; Tenant→Store already in `Domain/Tenants`); what's missing is the ability to create/manage the second store at runtime instead of via seed.

**Spec**
- Admin endpoints: create / edit / archive store. Archive hides the store from pickers and rejects new bookings/sessions; history stays readable. Keep `User.StoreId?` semantics exactly as API-020 defined them (admin = tenant-wide/null, staff = exactly one store) — "assign user" = set StoreId, audited (API-026). No multi-store-per-user role matrix; that's SaaS-console scope (unscheduled).
- Isolation proof extended: API-021's cross-*tenant* test matrix gains a second-store-same-tenant dimension — every store-scoped resource (sessions, templates, packages, bookings, devices) creatable under store B and invisible to store-A staff.

**Tests**: `Second_store_resources_isolated_from_store_a_staff` (parameterized), `Archive_hides_from_pickers_keeps_history_readable`, `Archived_store_rejects_new_bookings_and_sessions`, `Staff_reassignment_audited_and_effective_next_request`, `Last_active_store_cannot_be_archived`.

**Edge cases**: archiving a store with sessions active *today* → 409 with an actionable message (finish the day first); store create seeds nothing — packages/templates/devices are set up per store by admin (the checklist goes in the ops docs, not code).

---

### API-071 · Cross-store aggregates
Milestone M7 · Size M · Level junior · Depends: API-035, API-070

**Spec**: `GET /api/dashboard/overview?tz=` (admin-only, staff 403): per-store rows (sessions by status today, revenue today, completion rate, devices online/total) + tenant totals. One grouped query per metric — never a per-store fan-out (assert query count with an EF interceptor); `AsNoTracking`; perf guard ≤ 500 ms on a 5-store × 100k-session seed (Category=Perf, like API-035). The single-store API-035 endpoint is untouched — staff keep it.

**Tests**: `Rows_per_active_store_plus_totals`, `Query_count_bounded`, `Staff_403`, `Archived_excluded_by_default_included_on_optin`, `Perf_guard_overview`.

**Edge cases**: `includeArchived=true` opt-in flags archived rows; tenant with zero stores → empty rows + zero totals, not 404; tz handling identical to API-035 (client sends tz, server computes the UTC range).

---

*Milestone M8 (visual template designer) needs **no API work** — the designer (WEB-080..081) emits ordinary template config through API-010..012; that's ADR-011's payoff. Not in any milestone: SaaS machinery (self-signup, billing, plan limits, domain-based tenant resolution — ADR-012), ESC/POS receipt path (ADR-013), gallery-link-on-receipt evolution (§12), notification channels (WA/email for bookings and device alerts), AI features. M5 starts only after M4 is green — do not start post-M4 work early.*
