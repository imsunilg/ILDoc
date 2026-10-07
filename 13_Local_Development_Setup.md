# 13 — Local Development Setup

Commands below are the ones used in this project. Windows/PowerShell paths are shown; adapt for macOS/Linux (`run_all.sh`).

## Prerequisites (versions used)

| Tool | Version used | Notes |
|---|---|---|
| Git | any recent | |
| .NET SDK | 9.0.305 | API targets `net9.0` |
| PostgreSQL | 16 (client `psql 16.11`) | service on `localhost:5432`; `psql` may not be on `PATH` (Windows: `C:\Program Files\PostgreSQL\16\bin`) |
| Node.js | 24.x (v24.18.0) | for Angular |
| Angular CLI | 22.x | `npm i -g @angular/cli` |
| Flutter SDK | 3.47.4 stable (includes Dart ≥ 3.13) | installed at `C:\src\flutter`; `flutter config --enable-web` |
| Chrome | current | Flutter web and the browser test |

## Layout

Clone/keep the four repositories side by side:

```text
ILPortal/
  ILAPI/   ILDB/   ILWEBFlutter/   ILWEBAngular/   ILDoc/
```

(There are no remotes yet; once pushed, `git clone` each one into `ILPortal/`.)

## 1. Database (ILDB)

```powershell
$env:Path += ";C:\Program Files\PostgreSQL\16\bin"
$env:PGPASSWORD = "<your local postgres password>"   # session only, never in a file
cd ILDB
./run_all.ps1                 # creates ILPortal, schemas, tables, seed data and demo users
# optional: ./load_data.ps1   # loads the exported data (data/*.sql)
```

Production-like (no demo users): `$env:ILPORTAL_ENV='Production'` before `run_all.ps1`.

## 2. API (ILAPI)

Secrets are stored once, outside the repo:

```powershell
cd ILAPI/ILAPI.API
dotnet user-secrets set "ConnectionStrings:PostgreSQL" "Host=localhost;Port=5432;Database=ILPortal;Username=postgres;Password=<password>"
dotnet user-secrets set "Jwt:Key" "<random string, at least 32 characters>"
```

Run (the front ends expect port 5080):

```powershell
cd ILAPI
dotnet run --project ILAPI.API --urls http://localhost:5080
# check: http://localhost:5080/health  -> Healthy
dotnet test                       # 10 unit tests
```

## 3. Angular modules (ILWEBAngular)

```powershell
cd ILWEBAngular
npm install
npm start                         # ng serve -> http://localhost:4200
npm test                          # ng test (vitest) — set CI=true for a single run
npm run build
```

Opened directly, the app only says it must be opened from the portal.

## 4. Flutter portal (ILWEBFlutter)

```powershell
$env:Path += ";C:\src\flutter\bin"
cd ILWEBFlutter
flutter pub get
flutter run -d chrome --web-port 8090      # http://localhost:8090
flutter analyze ; flutter test
```

Compile-time options (`--dart-define=...`): `APP_ENV` (`development` default, `qa`, `production`), `ENABLE_DEMO_LOGIN`, `API_URL` (default `http://localhost:5080`), `ANGULAR_URL` (default `http://localhost:4200`).

## 5. Integration configuration checklist

| Setting | Where | Dev value |
|---|---|---|
| API URL used by Flutter | `lib/core/config.dart` / `--dart-define=API_URL` | `http://localhost:5080` |
| Angular URL used by Flutter | `--dart-define=ANGULAR_URL` | `http://localhost:4200` |
| API URL used by Angular | `ILWEBAngular/src/app/core/config.ts` (`API_URL`) | `http://localhost:5080` |
| Allowed portal origins in Angular | same file (`PORTAL_ORIGINS`) | `http://localhost:8090`, `http://127.0.0.1:8090` |
| CORS origins in the API | `ILAPI.API/appsettings.json` → `Cors:Origins` | `http://localhost:4200`, `http://localhost:8090` |

## 6. Run everything

Start in this order: PostgreSQL → API (5080) → Angular (4200) → Flutter (8090). Open **http://localhost:8090** and use a "Demo Login" button (development builds only).

## 7. Verify

| Check | Command |
|---|---|
| API roles and permissions | `ILAPI/scripts/verify-roles.ps1` (add `-ExpiredBase http://localhost:5081` with a second API instance started with `Jwt__AccessMinutes=-2` to test expiry) |
| Browser end-to-end | `flutter build web`, serve `build/web` on 8090, then `node ILWEBFlutter/scripts/e2e-portal.mjs <Role> <screenshotDir>` (needs `puppeteer-core` installed next to the script) |

## Secrets handling

Never commit `.env`, `*.local.json`, `secrets.json`, passwords, JWT keys. Each repository's `.gitignore` covers these.
