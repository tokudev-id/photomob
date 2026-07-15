# Gap: Booth device enrollment has no admin flow

Status: Implemented in software; physical DSLR certification remains  
Recorded: 2026-07-15  
Affected repositories: `potoku-api`, `potoku-platform-web`, `potoku`

End-to-end verification: [Potoku end-to-end testing](../runbooks/end-to-end-testing.md)

## Summary

The platform has the backend primitive for registering a booth and the booth has
a first-run pairing wizard, but there is no user-facing path connecting them.
An operator cannot provision a new webcam or DSLR booth from
`potoku-platform-web` without calling the API manually.

This is not a camera-adapter problem and must not be solved with a booth-side
authentication bypass. A device token identifies the booth computer and its
store. Webcam, DSLR, and automatic webcam fallback all run under the same
authenticated booth identity.

## Evidence in the current plan and implementations

- API-013 specifies that an admin creates a device and receives its plaintext
  device token exactly once. `potoku-api` implements this as authenticated
  `POST /api/devices`; the API stores only the token hash.
- BOOTH-022 says its device token comes from the admin's create-device screen.
  `potoku` implements the API URL, token validation, local staff-passcode setup,
  hardware selection handoff, and re-pairing protection.
- `potoku-platform-web` has no create-device endpoint wrapper or registration
  form. Its Devices surface currently covers health, alerts, and device detail.
- No task in `docs/tasks/potoku-platform-web.md` owns the create-device screen
  promised by BOOTH-022.

The result is a broken first-run journey:

```text
Admin can authenticate
  -> API can issue a device token
  -> no platform screen invokes issuance
  -> booth cannot pass first-run pairing
  -> camera selection and hardware check are unreachable
```

## Required product outcome

An authenticated admin can enroll a physical booth from the platform without
using Swagger, curl, database access, environment-file token injection, or a
development-only booth bypass. The same flow must provision booths that use:

- a webcam as the primary camera;
- a DSLR through digiCamControl;
- a DSLR with webcam fallback.

The enrollment flow provisions the booth identity. Camera choice remains local
hardware configuration per ADR-015.

## Recommended enrollment flow

Use a short-lived, one-time pairing code for human transfer while retaining the
long, high-entropy device token as the permanent API credential.

1. Admin opens **Devices** and selects **Register booth**.
2. Admin selects an active store and gives the booth a recognizable name.
3. The API creates a pending enrollment and returns a short pairing code with a
   clear expiry, recommended ten minutes.
4. The platform displays the code once, with expiry and cancellation controls.
5. On first boot, the booth accepts API URL plus pairing code.
6. The booth claims the code. The API atomically consumes it and returns the
   permanent device token only to the booth.
7. The booth validates the credential with a heartbeat, stores it locally, and
   asks the operator to create the 4–8 digit local staff passcode.
8. The booth opens hardware setup. The operator selects webcam or DSLR, tests a
   capture, configures printing, runs the hardware check, and saves.
9. The platform changes the pending enrollment to paired when the first
   authenticated heartbeat arrives.

The existing direct `POST /api/devices` token response may remain available for
controlled operator tooling, but it is not the primary UI flow. A permanent
token must never be shown in normal browser UI or copied through chat.

## Cross-repository work

### `potoku-api`

- Add an authenticated admin endpoint to create a pending enrollment for an
  active store and return `{ pairingCode, expiresAt, enrollmentId }`.
- Add a narrowly anonymous claim endpoint accepting the pairing code and booth
  metadata, returning `{ deviceId, deviceToken }` exactly once.
- Store only a hash of the pairing code and permanent device token.
- Make pairing codes random, single-use, short-lived, rate-limited, and scoped
  to tenant and store. Consume a code and create/activate the device in one
  transaction.
- Support cancellation and safe retry when the booth loses the claim response.
  Retrying the same claim from the same enrollment must not create two devices.
- Add token revocation and rotation ownership if API-013's planned revocation
  surface is not yet implemented.
- Regenerate OpenAPI and both TypeScript client snapshots.

### `potoku-platform-web`

- Add a typed enrollment API wrapper from the generated contract.
- Add **Register booth** to the Devices page for admins only.
- Provide store selection, booth name, submit/loading/error states, the pairing
  code, expiry countdown, Copy, Cancel, and Generate new code actions.
- Show pending, paired, expired, revoked, and offline states distinctly.
- Never persist a pairing code or permanent token in `localStorage`, logs,
  analytics, URLs, or error reports.
- Make the successful end state explain the booth's next step: choose hardware
  and run the hardware check.

### `potoku`

- Replace permanent-token entry in the normal first-run journey with pairing
  code claim. Keep API URL validation and heartbeat verification.
- Persist only the returned permanent token, never the pairing code.
- Preserve local staff-passcode creation and passcode-gated re-pairing.
- After pairing, allow the operator to select Webcam, DSLR, or configured
  primary/fallback without changing device identity.
- Provide actionable expired, already-used, cancelled, network, and invalid-code
  messages without falling back to an authentication bypass.

## Security and lifecycle invariants

- Creating an enrollment requires an authenticated tenant admin.
- Claiming an enrollment grants access only to its assigned store.
- Pairing codes expire, are one-time, and are protected by per-IP and
  per-enrollment rate limits.
- Permanent device tokens remain high entropy, are stored hashed by the API,
  and are returned only once to the booth.
- Device tokens and pairing codes never appear in logs, URLs, telemetry, crash
  reports, or platform persistence.
- Revocation takes effect on the next booth request.
- Re-pairing an already configured booth requires the local staff passcode.
- Camera changes never mint or require a different device credential.

## Acceptance criteria

- [ ] From a fresh development database, an admin can log in, open Devices,
      register "Local Booth", and receive a pairing code without API tooling.
- [ ] A fresh booth can claim the code, create its local passcode, and reach
      hardware setup.
- [ ] Reusing, guessing, cancelling, or claiming an expired code fails with a
      typed response and useful UI copy.
- [ ] A lost claim response can be retried safely without creating a duplicate
      device or exposing a second permanent token to another claimant.
- [ ] The first authenticated heartbeat marks the correct device online in the
      platform.
- [ ] The paired booth completes Test capture with a webcam on macOS/dev.
- [ ] The same enrollment flow completes Test capture with a DSLR through
      digiCamControl on Windows certification hardware.
- [ ] Switching a paired booth between webcam and DSLR does not require a new
      token or re-pairing.
- [ ] Revoking the device causes the next authenticated booth request to return
      401 and surfaces a staff-action state in the booth.
- [ ] OpenAPI drift gates and API, web, and booth automated tests pass.

## Test matrix

| Area | Required coverage |
| --- | --- |
| API | creation authorization, tenant/store scoping, expiry boundary, one-time consume, concurrent claims, retry after lost response, cancellation, rate limit, token secrecy, revocation |
| Platform | admin-only action, store/name validation, countdown, typed failures, copy/cancel/regenerate, no browser persistence, paired heartbeat state |
| Booth | fresh install, invalid/expired/used code, network retry, atomic token persistence, passcode creation, passcode-gated re-pair, webcam and DSLR configuration |
| End to end | admin registration -> booth claim -> heartbeat -> hardware setup -> webcam capture; repeat on certified DSLR hardware |

## Non-goals

- Removing device authentication for local development.
- Treating a webcam as a separate trust model or weaker type of booth.
- Moving hardware selection to the server; ADR-015 keeps it booth-local.
- Replacing booking/session codes, which serve customers and are unrelated to
  staff device enrollment.

## Planning follow-up

Create explicit cross-repository tasks and dependency links before marking the
gap scheduled. At minimum, the platform task that BOOTH-022 already assumes must
exist. If the short-code enrollment design is accepted, add API and booth tasks
as a coordinated contract change and record the enrollment decision in
`docs/DECISIONS.md`.
