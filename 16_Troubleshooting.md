# 16 — Troubleshooting

Format: **Symptom → Possible cause → How to verify → Solution.**

## Startup

| Symptom | Possible cause | Verify | Solution |
|---|---|---|---|
| API exits immediately: "ConnectionStrings:PostgreSQL is not configured" or "Jwt:Key must be set" | secrets missing (placeholders in `appsettings.json`) | `dotnet user-secrets list` in `ILAPI.API` | set both secrets (see [13](13_Local_Development_Setup.md)) |
| API logs "Database connection to ILPortal FAILED" / `/health` Unhealthy | PostgreSQL not running, wrong password, database missing | `Get-Service postgresql-x64-16`; `psql -h localhost -U postgres -d ILPortal -c "select 1"` | start the service; re-run `run_all.ps1`; fix the connection string |
| `psql` not recognised | not on `PATH` | `where psql` | add `C:\Program Files\PostgreSQL\16\bin` to `PATH` |
| `run_all.ps1`: "PGPASSWORD is not set" | env var missing | `$env:PGPASSWORD` | set it in the session |
| Flutter does not start | SDK not on `PATH`, web not enabled | `flutter doctor`; `flutter config` | add `C:\src\flutter\bin`; `flutter config --enable-web`; `flutter pub get` |
| `flutter run -d web-server` page blank in a headless browser | debug build needs the tool's debug connection (Issue 14) | console `Library not defined` | use `flutter run -d chrome` or serve a release build |
| Angular does not start | wrong Node version, `npm install` not run, port 4200 busy | `node -v`; `npm ls @angular/core`; `Get-NetTCPConnection -LocalPort 4200` | install deps; free the port or `ng serve --port` (then update `ANGULAR_URL` and API CORS) |
| The portal URL shows "Server is up and running." or something unrelated | another program (e.g. the Apache `httpd` from the PostgreSQL install) owns the port on IPv4 while Flutter is on IPv6 (Issue 15) | `Get-NetTCPConnection -LocalPort 8090` shows who listens | use a free port: `flutter run --web-port <port>` and update `Cors:Origins` and Angular `PORTAL_ORIGINS` (the project uses **8090**) |

## API / authentication

| Symptom | Possible cause | Verify | Solution |
|---|---|---|---|
| Browser console: CORS error from `localhost:8090`/`4200` | origin not in `Cors:Origins`; API not restarted | response lacks `Access-Control-Allow-Origin`; `curl -H "Origin: …"` | add the origin to `appsettings.json` / env var and restart |
| Everything returns 401 | expired access token, wrong `Jwt:Key` after a restart with a different key, clock skew | decode the JWT (`exp`, `iss`, `aud`); compare keys | log in again; use one stable key per environment |
| 403 on a page the menu shows | stale permissions in the JWT, or seed changed | decode `permission` claims; compare with `GET /api/v1/auth/me` | log out/in; fix `role_permissions` |
| Login: "Invalid username or password" for a demo user | demo users not seeded (`ILPORTAL_ENV=Production` or `08` not run) | `select username from auth.users` | re-run `run_all.ps1` without `ILPORTAL_ENV` |
| "Cannot reach the server" on login | API not running / wrong `API_URL` | open `/health` | start API; set `--dart-define=API_URL` |
| Logout does not seem to end the session | stored session restored from local storage | browser storage key `flutter.il.session` | clear site data; check `POST /api/v1/auth/logout` returns 200 |

## Flutter + Angular integration

| Symptom | Possible cause | Verify | Solution |
|---|---|---|---|
| Travel/Liability area shows "Session unavailable" | handoff failed: origin mismatch, API down, token expired | iframe console; network call `POST /api/v1/frontend-session` and `/auth/handoff/exchange` | make `ANGULAR_URL` and Angular `PORTAL_ORIGINS` match the real origins; check CORS |
| Area stays "Loading…" | Flutter never answered `il-ready` (listener missing or origin check failed) | Flutter console | confirm `angularOrigin` equals the iframe origin exactly (scheme + host + port) |
| iframe is blank / refused | host sends `X-Frame-Options: DENY` or CSP blocks framing | iframe network response headers | allow framing from the Flutter origin only |
| Angular page says "This module runs inside the portal" | opened directly at `:4200` | — | open the portal at `:8090` |
| **Angular header appears** | an old Angular build with the portal shell is being served | view source / DOM of the iframe | rebuild/restart Angular; `app.ts` must only contain the router outlet |
| **Duplicate dashboard** | same as above, or Flutter route `/dashboard` plus a stale Angular dashboard route | check `app.routes.ts` has no dashboard route | remove stale build output (`dist/`, `.angular/cache`) |
| Wrong colours / seam between Flutter and Angular | palettes out of sync | compare `app_theme.dart` and `styles.css` variables | align the values; iframe background in `angular_module_view_web.dart` |

## Routing

| Symptom | Possible cause | Verify | Solution |
|---|---|---|---|
| Refresh on `/travel` returns 404 | host lacks SPA fallback (path URLs) | request `/travel` directly | add rewrite to `index.html` |
| Back button leaves the app | iframe history entries consumed; or very first page | check history depth | expected for the first entry; otherwise verify `go_router` routes |
| URL redirects to `/dashboard` unexpectedly | route unknown or no permission | compare route with `menu.dart`; user's permissions | grant permission or use a valid route |

## Quotes / database

| Symptom | Possible cause | Verify | Solution |
|---|---|---|---|
| Quote POST returns 400 `{error}` | validation (age 17–100, end ≥ start date, unknown country/coverage…) | read the message | correct the input |
| Quote POST returns 500 | database error (missing seed rows, FK) | API log | re-run `06_seed_data.sql`; check `product.products` active |
| `fn_next_quote_number` errors | sequence missing (old DB) | `\ds quote.*` | re-run `03_tables.sql` |
| `functions not found` | `05_functions.sql` not run | `\df quote.*` | `run_all.ps1` |
| Quote list empty for a customer | `quotes.own` only shows own quotes | check `customer.customers.user_id` | create a quote as that user |
| Data load fails on FK | loaded before schema/seed | error names the table | run `run_all.ps1` first, load files in order |

## Useful diagnostics

```powershell
Invoke-RestMethod http://localhost:5080/health
Invoke-RestMethod http://localhost:5080/api/products
# decode a JWT payload (do not paste real tokens into tickets)
```
