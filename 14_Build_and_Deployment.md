# 14 — Build and Deployment

Diagram: [diagrams/deployment-flow.md](diagrams/deployment-flow.md)

**Status:** builds are verified locally. **Deployment automation is PLANNED** — the repositories contain no Dockerfile, compose file, CI workflow or hosting configuration. Everything under "Recommended" is a proposal derived from how the apps are built.

## Build commands (verified)

| Project | Command | Output |
|---|---|---|
| ILAPI | `dotnet publish ILAPI.API -c Release` (`dotnet build` verified) | self-contained set of DLLs; run with `dotnet ILAPI.API.dll` |
| ILWEBAngular | `ng build` (production configuration by default) | `ILWEBAngular/dist/ILWebAngular` (static files) |
| ILWEBFlutter | `flutter build web --dart-define=APP_ENV=production --dart-define=API_URL=<api> --dart-define=ANGULAR_URL=<angular host>` | `ILWEBFlutter/build/web` (static files) |
| ILDB | `ILPORTAL_ENV=Production ./run_all.ps1` (or `.sh`) | schema + production-safe seed (no demo users) |

## Runtime topology (recommended)

| Component | Hosting | Notes |
|---|---|---|
| Flutter web | any static host/CDN (nginx, IIS, S3 + CDN, Azure Static Web Apps) | **SPA fallback**: unknown paths return `index.html` (path URLs, no `#`) |
| Angular web | another static host/path | SPA fallback; must be frameable by the Flutter origin |
| ILAPI | container or app service behind a reverse proxy | HTTPS only; `/health` for probes |
| PostgreSQL | managed PostgreSQL 16 | application role with least privilege (not `postgres`) |

Flutter and Angular may share a host with different paths **only if** `ANGULAR_URL`, `PORTAL_ORIGINS` and the iframe URL are adjusted; same-origin hosting also removes the need for `postMessage` origin differences but is not what is implemented.

## Environment configuration

| Setting | Dev (current) | QA / UAT / Production |
|---|---|---|
| `Cors:Origins` (API) | `http://localhost:4200`, `http://localhost:8090` | the real Flutter and Angular origins (env var `Cors__Origins__0`, …) |
| `ConnectionStrings:PostgreSQL` | user-secrets | secret store / `ConnectionStrings__PostgreSQL` |
| `Jwt:Key` | user-secrets | secret store / `Jwt__Key` (≥ 32 chars, unique per environment) |
| `Jwt:Issuer`, `Jwt:Audience`, lifetimes | `ILAPI`, `ILPortal`, 15 min / 14 d / 60 s | same unless policy changes |
| Flutter `APP_ENV` | `development` (demo login on) | `qa` (demo login only with `ENABLE_DEMO_LOGIN=true`), `production` (demo login impossible) |
| Flutter `API_URL`, `ANGULAR_URL` | defaults to localhost | `--dart-define` at build time |
| Angular `API_URL`, `PORTAL_ORIGINS` | **hard-coded in `core/config.ts`** | must be made environment-specific (Angular `environment` files or a runtime `config.json`) — **PENDING** |
| Demo users | seeded by `08_demo_users.sql` | never (`ILPORTAL_ENV=Production`) |

## Pre-production checklist

- [ ] Angular configuration externalised (API URL, allowed portal origins).
- [ ] API `Cors:Origins` updated; `AllowedHosts` restricted.
- [ ] HTTPS everywhere; HSTS at the proxy.
- [ ] Angular host sends `Content-Security-Policy: frame-ancestors <Flutter origin>` (and no `X-Frame-Options: DENY`); Flutter host restricts `frame-src` to the Angular origin.
- [ ] Production `Jwt:Key` generated and stored in a secret manager; development key not reused.
- [ ] Database role with only the needed privileges; backups configured.
- [ ] Demo users absent: `SELECT count(*) FROM auth.users WHERE email LIKE '%@demo.local'` returns 0.
- [ ] Flutter built with `APP_ENV=production`; verify the bundle contains no demo credentials.
- [ ] SPA fallback tested by refreshing `/travel`, `/dashboard`.
- [ ] Rate limiting / lockout on `/api/v1/auth/login` (not implemented yet).
- [ ] Premium rates and tax rate replaced with approved values.

## Suggested pipeline (PLANNED)

1. Build and test each repository independently (`dotnet test`, `flutter analyze/test`, `ng test`).
2. Build artifacts with environment-specific settings.
3. Apply `ILDB` scripts to the target database (idempotent).
4. Deploy API, then Angular, then Flutter (the shell last, so it never points at a missing module).
5. Run `scripts/verify-roles.ps1` and the browser end-to-end test against the environment (demo users are not available in production; use a dedicated test tenant/accounts there).
