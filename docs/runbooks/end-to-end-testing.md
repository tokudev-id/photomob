# Potoku end-to-end testing

This runbook verifies the complete local journey across `potoku-api`,
`potoku-platform-web`, and the Electron booth in `potoku`. It covers booth
enrollment, heartbeat, hardware setup, a customer session, gallery delivery,
and the enrollment failure/lifecycle cases.

## Prerequisites

- Docker Desktop for the compose-backed API stack.
- Node.js 22 for the web and booth repositories.
- .NET 8 only when running the API directly instead of through compose.
- The three sibling repositories checked out beside this planning repository.

## 1. Start the API

The recommended full-stack path is Docker Compose because it includes
PostgreSQL, Redis, MinIO, migrations, and development seed data, and exposes the
API at `http://localhost:8080`:

```bash
cd /Users/thoriq/Personal/Project/Toku/potoku-api
docker compose up --build
curl -fsS http://localhost:8080/health/ready
```

For a genuinely fresh acceptance run, `docker compose down -v` may be run
first. This is destructive: it deletes the local development database and
media volumes.

The development seed creates:

- tenant `Potoku Dev`;
- store `Dev Store`;
- package `Dev Basic`;
- admin `admin@potoku.local` with password `PotokuDev123!`.

### Direct .NET alternative

The `http` launch profile exposes the API at `http://localhost:5096`:

```bash
cd /Users/thoriq/Personal/Project/Toku/potoku-api
docker compose up postgres redis
dotnet run --project src/Potoku.Api --launch-profile http
curl -fsS http://localhost:5096/health/ready
```

When using this alternative, the web development proxy must also target port
5096. The committed Vite configuration currently targets port 8080. See
[Development proxy troubleshooting](#development-proxy-troubleshooting).

## 2. Start the platform web application

```bash
cd /Users/thoriq/Personal/Project/Toku/potoku-platform-web
nvm use 22
npm ci
npm run dev
```

Vite normally opens `http://localhost:5173`. If that port is occupied, it may
select 5174 or another free port.

## 3. Prepare and start the booth

Use this untracked `.env` for a hardware-free run:

```dotenv
POTOKU_API_URL=http://localhost:8080
POTOKU_GALLERY_BASE_URL=http://localhost:5173
POTOKU_CAMERA=mock
```

Use port 5096 for `POTOKU_API_URL` only when the API is launched directly with
the .NET `http` profile. Do not define `POTOKU_DEVICE_TOKEN` during a first-run
enrollment test; it bypasses the pairing wizard.

```bash
cd /Users/thoriq/Personal/Project/Toku/potoku
nvm use 22
npm install
npm run dev
```

On macOS, an existing booth profile is stored at:

```text
~/Library/Application Support/@potoku/booth
```

Quit the booth and rename that directory to create a clean profile. Preserve it
as a backup instead of deleting it if existing local sessions or settings
matter.

## 4. Enrollment happy path

1. Open the platform web application and log in as `admin@potoku.local` using
   `PotokuDev123!`.
2. Open **Devices** and select **Register booth**.
3. Select **Dev Store**, name the booth `Local Booth`, and generate a code.
4. In the booth, enter the API URL and the displayed eight-character pairing
   code.
5. Create a 4–8 digit local staff passcode and complete pairing.
6. Confirm that **Finish booth setup** opens.
7. Select **Mock camera** and **No printer**, then save settings.
8. Run **Test capture**.
9. Run **Hardware check** and verify three captures, composition, skipped print,
   and API heartbeat succeed.
10. Return to **Devices**. Within approximately ten seconds, confirm that
    `Local Booth` is paired and online.

The platform must never display or persist the permanent device token. The
pairing code must not appear in a URL, browser storage, analytics, or logs.

## 5. Complete customer session and gallery

1. In the platform, open **Bookings** and create a new walk-in.
2. Use a valid phone such as `081234567890`, select **Dev Basic**, and select
   **Now**.
3. Record the Rp0 development payment and copy the resulting six-digit session
   code.
4. At the booth, start a session and enter the code.
5. Capture several photos and select exactly four for the bundled 2x6 template.
6. Confirm composition and continue through printing to the QR screen.
7. Record the gallery short code and four-digit PIN.
8. Open the gallery URL, enter the PIN, and verify the pending state transitions
   to the composed strip and raw captures.
9. Open or download the returned media.

When scanning the QR code from a phone, `localhost` is the phone itself. Expose
Vite on the LAN and set the booth gallery URL to the development computer's LAN
address:

```bash
npm run dev -- --host 0.0.0.0
```

```dotenv
POTOKU_GALLERY_BASE_URL=http://192.168.x.x:5173
```

## 6. Enrollment lifecycle matrix

| Scenario | Procedure | Expected result |
| --- | --- | --- |
| Invalid code | Enter a random eight-character code | Actionable invalid-code message |
| Cancelled | Generate a code, cancel it in the platform, then enter it in the booth | Cancelled-code message |
| Expired | Generate a code and wait ten minutes before claiming | Expired-code message |
| Reused | Pair successfully, then claim the same code from a fresh booth profile | Already-used message and no second device |
| Lost response | Interrupt after claim but before heartbeat, restore the API, and retry | Same device credential is recovered; no duplicate device |
| Offline | Close the booth and wait more than three minutes | Platform shows Offline |
| Revoked | Revoke the device, then attempt a session activation | Next request is 401 and booth shows a staff-action state |
| Re-pair | Tap the top-left corner five times within three seconds | Current staff passcode is required |
| Passcode lockout | Enter an incorrect settings passcode five times | Settings is locked for thirty seconds |
| Camera switch | Change Mock/Webcam/DSLR and restart | Same device identity; no new enrollment |

For deterministic lost-response testing, pause the Electron main process after
`claimDeviceEnrollment(...)` returns but before heartbeat, stop the API, resume
the booth, then restore the API and retry the same code. The booth must preserve
its claim ID across the retry.

## 7. Webcam and DSLR certification

For a macOS webcam run, set `POTOKU_CAMERA=webcam`, grant camera permission,
select the webcam, save, and restart before running Test capture and Hardware
check. Camera adapter changes take effect after restart.

For Windows DSLR certification:

1. Start the digiCamControl web server at `http://127.0.0.1:5513`.
2. Connect the supported camera and select **DSLR via digiCamControl**.
3. Configure the capture folder and webcam fallback.
4. Save and restart the booth.
5. Complete the DSLR and printer checklists in the booth repository's
   `docs/hardware-notes.md`.

## 8. Automated quality gates

API:

```bash
cd /Users/thoriq/Personal/Project/Toku/potoku-api
dotnet test Potoku.Api.sln
./scripts/check-client-drift.sh
```

Platform web:

```bash
cd /Users/thoriq/Personal/Project/Toku/potoku-platform-web
nvm use 22
npm run check
```

Booth:

```bash
cd /Users/thoriq/Personal/Project/Toku/potoku
nvm use 22
npm run rebuild:node -w @potoku/booth
npm run ci
```

`npm run dev` rebuilds `better-sqlite3` for Electron. Restore its Node ABI with
`rebuild:node` before running the Node-based booth suite.

## Development proxy troubleshooting

The web client deliberately calls relative URLs such as `/api/brand`. This
keeps browser requests same-origin and lets Vite proxy them to the API, avoiding
CORS differences between development and production. Therefore a browser
request shown as `http://localhost:5173/api/brand` or
`http://localhost:5174/api/brand` is expected; it is not evidence that the
frontend is acting as the API.

The committed `vite.config.ts` proxies `/api` to `http://localhost:8080`. If the
API is instead running from the .NET launch profile on port 5096, either:

1. use Docker Compose so the API is available on port 8080; or
2. configure the Vite proxy target to `http://localhost:5096` and restart Vite.

A Vite-generated 500 with an unavailable proxy target commonly means the
proxy could not connect upstream. Compare these responses before debugging the
brand handler:

```bash
curl -i http://localhost:5096/api/brand
curl -i http://localhost:8080/api/brand
curl -i http://localhost:5174/api/brand
```

The anonymous brand endpoint does not need browser cookies or an Authorization
header. Avoid copying authentication cookies into diagnostics or issue reports.
