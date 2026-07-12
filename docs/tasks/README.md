# Potoku — Engineering Task Breakdown

> Senior-engineer task breakdown of the [PRD](../PRD.md) for junior/mid engineers. One file per program. Every task is written to be picked up **without asking what it means** — if you still have to ask, that's a bug in the task: flag it.

| File | Repo | Program |
|---|---|---|
| [potoku-api.md](potoku-api.md) | [potoku-api](https://github.com/tokudev-id/potoku-api) | .NET 8 backend (Clean Architecture + DDD) |
| [potoku-booth.md](potoku-booth.md) | [potoku](https://github.com/tokudev-id/potoku) | Electron booth app |
| [potoku-platform-web.md](potoku-platform-web.md) | [potoku-platform-web](https://github.com/tokudev-id/potoku-platform-web) | React + Vite SPA (admin + SaaS + gallery) |

---

## How to read a task

```
### API-001 · Title
Milestone M0 · Size M · Level junior|mid · Depends: —
Context   — why this exists, which PRD/ADR it serves
Spec      — exactly what to build (endpoints, types, UI states)
Steps     — suggested order of work
AC        — acceptance criteria: checkboxes reviewers verify
Tests     — named test cases you MUST write (Given/When/Then)
Edge cases — failure modes you MUST handle (each one needs a test or an explicit AC)
Out of scope — things you must NOT build here (scope creep guard)
```

- **Size**: S ≤ half day · M ≤ 2 days · L ≤ 5 days. If your task is blowing past its size, stop and talk — don't silently grow it.
- **Level** is a suggestion, not a gate.
- **Depends** may cross repos (e.g. `BOOTH-005 ← API-008`). The dependency must be *merged*, not just "nearly done".

## Global Definition of Done (applies to every task)

1. Code + tests merged to `develop` via PR; CI green (build, typecheck, all tests).
2. Every test case listed in the task exists and passes; every edge case is either tested or explicitly asserted in AC.
3. No hardcoded brand strings/colors/copy (ADR-012) — reviewers reject on sight.
4. Money is integer IDR; every domain row hangs off `StoreId` (ADR-007).
5. Errors are handled, not swallowed: user-facing failures have a designed state, background failures are logged with context.
6. Public surface documented: API tasks update OpenAPI annotations; UI tasks include a screenshot in the PR.
7. **No TODO comments without a linked task ID.**

## Milestone order & cross-repo sync points

```
M0  API-001..009  →  BOOTH-001..007  →  WEB-001..003     "photo → QR → phone"
M1  API-010..015  →  BOOTH-010..020  +  WEB-010..011     "stranger prints a strip"
M2  API-020..028  →  WEB-020..025    +  BOOTH-021..022   "receipt code starts the booth"
M3  API-030..036  →  WEB-030..032    +  BOOTH-023        "day-6 link extended in 10s"
M4  API-040..042  +  BOOTH-024..027  +  WEB-040..041     "unplug the network; nothing lost"

—— post-M4 gate: M5 opens only after M4 is green (API-042 + BOOTH-027 chaos suites) ——

M5  API-050..053  →  WEB-050..053                        "customer books & pays from their phone"
M6  API-060..062  →  BOOTH-030..031  +  WEB-060          "boomerang in the gallery; media on S3"
M7  API-070..071  →  WEB-070..071                        "one login runs five stores"
M8  WEB-080..081  (no API/booth work)                    "a template designed without touching JSON"
```

Key cross-repo dependencies (blocking, both merged before dependent starts):

| Dependent | Needs |
|---|---|
| BOOTH-005 (API client) | API-008 (OpenAPI + TS codegen) |
| BOOTH-004 (upload queue) | API-004 (media upload endpoint) |
| WEB-002 (gallery M0) | API-006 (gallery endpoints) |
| BOOTH-013 (template sync) | API-012 (template manifest) |
| WEB-010 (template manager) | API-010..012, WEB-011 (canvas rasterizer) |
| BOOTH-021 (code entry vs bookings) | API-024..025 |
| WEB-023 (receipt print) | API-023 (bookings/payments) |
| BOOTH-023 (reprint pickup) | API-034 (print job queue) |
| WEB-050 (self-booking flow) | API-050 (public booking surface) |
| WEB-051 (checkout) | API-051..052 (gateway + webhooks) |
| WEB-060 (animated tile) | API-062, BOOTH-031 (asset producer) |
| BOOTH-031 (animated assembly) | API-062 (Animated media kind) |
| WEB-070 (store switcher) | API-070 (store CRUD) |

## Shared conventions

- **Session codes** (booth entry): 6 digits, keypad-friendly, unique among the store's non-terminal sessions, validated with rate limiting. **Delivery shortCodes** (gallery): ≥10 chars Crockford base32 (~50 bits), no ambiguous glyphs. Do not confuse the two.
- **Idempotency**: every booth→API write carries a client-generated key (media checksum, event UUID). Retries must be safe — the booth WILL retry.
- **Time**: store UTC (`timestamptz` / `DateTimeOffset`); the booth/store timezone is presentation-only. Never compare local times server-side.
- **Template config**: source of truth is the zod schema in `@potoku/template-kit` (ADR-016). The API validates the *generated JSON Schema*; never re-model it in C#.
