# 11 — Issues and Solutions

Only issues evidenced by the repositories, their history or the test runs are listed. Commit hashes are from the **per-repository** histories (the single repository was split into four on 2026-10-07; `git log` in each folder shows them). "Before" refers to the state before commit `46ee03e` (Flutter) / `4c6fcc2` (Angular) / `3c937df` (API), "refactor: make Flutter the portal shell and Angular Travel/Liability embedded product modules".

## Summary

| # | Issue | Status |
|---|---|---|
| 1 | Duplicate header | Fixed |
| 2 | Duplicate dashboard | Fixed |
| 3 | Duplicate login | Fixed |
| 4 | Authentication handoff (token in URL) | Fixed |
| 5 | Routing conflict (Flutter vs Angular vs browser URL) | Fixed |
| 6 | Browser back/forward | Fixed, verified |
| 7 | Logout across both apps | Partly fixed — **known gap** |
| 8 | API duplication | Resolved (one API); some client-code duplication remains |
| 9 | UI inconsistency | Fixed (twice) |
| 10 | Debug banner | Not an application defect |
| 11 | `SecurityError` when checking `event.source` | Fixed |
| 12 | Async guards lost the injection context | Fixed |
| 13 | Staff users could not create quotes (no customer row) | Fixed |
| 14 | Flutter dev server unusable for headless browser tests | Worked around |
| 15 | Port 8080 shared with Apache | Worked around |
| 16 | `run_all.ps1` failed on PostgreSQL notices | Fixed |

---

## Issue 1 — Duplicate header

- **Symptoms:** screenshots showed the Flutter portal header (brand, Dashboard/Motor/Home/Travel/Liability, user, sign out) and an Angular page with its own blue header and the same navigation.
- **Root cause:** both apps had been built as complete portals (`ILWEBAngular` commits `7467241`, `022596a`: login, dashboard, header with permission-driven menu).
- **Investigation:** read `app.ts` (inline header template with `routerLink` menu) and the Flutter shell; confirmed both rendered a header.
- **Solution:** Flutter is the only shell. Angular's `App` became a content host with a `router-outlet` and status messages; the header, nav, user/sign-out were deleted.
- **Files changed:** `ILWEBAngular/src/app/app.ts` (+ `app.html`, `app.css`); `ILWEBFlutter/lib/shell/portal_shell.dart`, `lib/shared/angular_module_view*.dart`.
- **Final result:** one header. Browser test asserts the iframe has no `header`/`nav` element and no "Sign out/Sign in/Dashboard" text (`scripts/e2e-portal.mjs`).

## Issue 2 — Duplicate dashboard

- **Symptoms:** Flutter and Angular each had a "Get a quote" dashboard with product cards and a quote table.
- **Root cause:** Angular's `pages/dashboard.ts` duplicated Flutter's dashboard.
- **Solution:** deleted Angular's dashboard (and its routes); Flutter's `DashboardHome` is the only one. Angular cards that used to link to products were removed; Flutter cards navigate to `/travel` and `/liability`.
- **Files changed:** removed `ILWEBAngular/src/app/pages/dashboard.ts`; `ILWEBFlutter/lib/features/dashboard/dashboard_home.dart`.
- **Final result:** Angular has no dashboard route (`app.routes.ts` only has travel, liability, forbidden).

## Issue 3 — Duplicate login

- **Symptoms:** Angular had its own `/login` page plus a `/handoff` landing page.
- **Root cause:** Angular was a standalone portal.
- **Solution:** removed `pages/login.ts` and `pages/handoff.ts`; Angular's `AuthService` no longer calls `login`/`register`. Flutter's `LoginScreen` is the only sign-in.
- **Files changed:** removed `ILWEBAngular/src/app/pages/login.ts`, `pages/handoff.ts`; rewrote `core/auth.service.ts`.
- **Final result:** opened directly, Angular shows "open it from the portal". The test asserts no navigation to `:4200/login` ever occurs.

## Issue 4 — Authentication handoff

- **Symptoms/root cause:** the first integration (Flutter `e96977c`) opened Angular at `http://localhost:4200/handoff?token=<one-time token>` in a new tab. That meant a token in the URL (history, logs, referrer) and a second, separate Angular app.
- **Solution:** iframe + `postMessage`. Angular asks (`il-ready`), Flutter replies (`il-handoff`) with a 60-second single-use token to the Angular origin only; Angular exchanges it for an in-memory session. API alias `POST /api/v1/frontend-session` added.
- **Files changed:** `ILWEBFlutter/lib/shared/angular_module_view_web.dart`, `lib/core/api.dart` (`createFrontendSession`); `ILWEBAngular/src/app/core/auth.service.ts`, `core/config.ts`; `ILAPI.API/Controllers/AuthController.cs`.
- **Final result:** the e2e test checks that no URL seen by any frame contains `token` or `eyJ`.

## Issue 5 — Routing conflict (Flutter router, Angular router, browser URL)

- **Symptoms:** Flutter used in-widget page switching and `Navigator.push` for quotes, so the URL never reflected the page; Angular had its own full route table, and opening it as a second app meant two independent address bars.
- **Root cause:** no single owner of the URL.
- **Solution:** Flutter owns the browser URL through `go_router` (`ShellRoute` + one `GoRoute` per menu item, redirect guard). Angular routes exist only inside the iframe and are product-scoped. Flutter then switched from hash URLs to **path URLs** (`ad549f5`).
- **Files changed:** `ILWEBFlutter/lib/core/routing/router.dart`, `lib/shell/menu.dart`, `lib/shell/portal_shell.dart`, `lib/main.dart`, `pubspec.yaml` (`go_router`, `flutter_web_plugins`).
- **Final result:** URLs such as `http://localhost:8080/liability`; unknown or forbidden paths redirect to `/dashboard`. Hosts need an SPA fallback.

## Issue 6 — Browser back/forward

- **Symptoms:** with in-widget navigation, Back left the portal.
- **Root cause:** no history entries per page (see Issue 5).
- **Solution:** `go_router` pushes history entries; the iframe's navigation joins the tab's session history.
- **Verification:** `e2e-portal.mjs`: Travel quote → result → Back lands on the Travel form with the header intact; Back/Forward between Dashboard and Liability works for all five roles. (Tooling note: Puppeteer's `page.goBack()` waits for a navigation event that same-document history never fires, so the script calls `history.back()`.)
- **Final result:** works.

## Issue 7 — Logout across both apps

- **Symptoms:** none visible, but analysis shows an asymmetry.
- **Root cause:** Angular holds its own session (access + refresh token) obtained through the handoff; Flutter's logout revokes only Flutter's refresh token.
- **Solution so far:** Flutter logout destroys the iframe, discarding Angular's in-memory session; Flutter's refresh token is revoked server-side.
- **Remaining gap:** Angular's refresh token stays valid in the database (≤ 14 days) until used or expired, and access tokens live ≤ 15 minutes. Both are only reachable by someone who already captured them. Planned fix: tie the Angular session to the Flutter session and revoke all refresh tokens of the user on logout, or ask Angular to call `/logout` on a `il-logout` message.
- **Files:** `ILWEBFlutter/lib/core/api.dart` (`logout`), `ILAPI.API/Controllers/AuthController.cs` (`Logout`).

## Issue 8 — API duplication

- **Symptoms:** risk of "Angular API" and "Flutter API".
- **Solution:** both call the single `ILAPI` (`API_URL` in Flutter `config.dart`, `API_URL` in Angular `core/config.ts`). Auth endpoints moved to `/api/v1/auth` with one envelope; both clients were updated.
- **Remaining duplication:** each front end has its own HTTP client, token refresh logic and field definitions. A generated OpenAPI client is a future option.
- **Final result:** one backend, one database.

## Issue 9 — UI consistency

- **Symptoms:** Angular used a blue header/cards theme that did not resemble Flutter's content area.
- **Solution:** (1) Angular restyled to Flutter's Material look inside the iframe (`4c6fcc2`); (2) both restyled to a common orange/maroon palette (`fc48ce3`, `f2428b5`) with CSS variables in `ILWEBAngular/src/styles.css` mirroring `ILWEBFlutter/lib/core/theme/app_theme.dart`; the iframe background matches the Flutter page background.
- **Files:** the two theme files above, `shared/quote-form/quote-form.css`, `shared/quote-result/quote-result.css`, `angular_module_view_web.dart` (iframe background).
- **Final result:** the Travel page screenshot shows the Angular card inside the Flutter content area with no visible seam. The two palettes are kept in sync by hand.

## Issue 10 — Debug banner

- **Symptoms:** the screenshots show a red "DEBUG" ribbon in the corner of the Flutter pages.
- **Root cause:** Flutter's `MaterialApp` shows it in debug mode (`flutter run`); it is framework UI, not application UI.
- **Solution:** none needed. It does not appear in release builds (`flutter build web`), which is what the browser tests use. The app does not depend on it.

## Issue 11 — `SecurityError` on `event.source`

- **Symptoms:** Travel/Liability never loaded; the page console showed `SecurityError: Blocked a frame ... from accessing a cross-origin frame`; Angular stayed on "Loading…" then "Session unavailable".
- **Root cause:** the first version of `AngularModuleView._onMessage` compared `event.source != _iframe.contentWindow`; touching the cross-origin window object from Dart throws.
- **Investigation:** headless-browser test, `pageerror` output.
- **Solution:** removed the comparison; the origin check (`event.origin == angularOrigin`) remains the control. Documented in [03](03_Flutter_Angular_Integration.md).
- **Files changed:** `ILWEBFlutter/lib/shared/angular_module_view_web.dart`.
- **Final result:** all modules render for every permitted role.

## Issue 12 — Guards lost the injection context

- **Symptoms:** unit tests `roleGuard` / `permissionGuard` failed with `NG0203: The Router token injection failed`.
- **Root cause:** after the guards became `async` (they wait for the handoff), `inject(Router)` ran after `await`.
- **Solution:** inject `Router` before awaiting.
- **Files changed:** `ILWEBAngular/src/app/core/auth.interceptor.ts`; covered by `core/guards.spec.ts`.

## Issue 13 — Staff users could not create quotes

- **Symptoms:** none observed in a running system; identified while implementing role-based quoting and fixed before staff quotes were ever tested. Only registered customers had a `customer.customers` row, so a quote insert for an agent/admin would have violated the foreign key.
- **Solution:** `QuoteRepository.CreateAsync` inserts a customer row from `auth.users` (`ON CONFLICT (user_id) DO NOTHING`) before the quote. Verified by the role matrix (`scripts/verify-roles.ps1`: ADMIN/AGENT motor and travel → `201`).
- **Files changed:** `ILAPI.Infrastructure/Repositories.cs`.
- **Note:** staff quotes are attached to the staff member's own customer row; selecting a real customer is planned.

## Issue 14 — Flutter dev server and headless browser tests

- **Symptoms:** `flutter run -d web-server` pages failed to initialise in headless Chrome (`Library not defined: org-dartlang-app:///web_entrypoint.dart`).
- **Root cause:** the DDC debug build expects the Flutter tool's debug connection.
- **Solution:** the browser test runs against a **release build** (`flutter build web`) served by a small static server on 8080. Documented in the ILWEBFlutter README.

## Issue 15 — Port 8080 shared with Apache

- **Symptoms:** `http://localhost:8080` answered from an unrelated Apache `httpd`.
- **Root cause:** Apache listens on IPv4 `0.0.0.0:8080`; Flutter bound IPv6 `::1`.
- **Solution:** none required in the app; use `localhost` consistently, or choose another port and update `Cors:Origins`, Angular `PORTAL_ORIGINS` and Flutter's `--web-port`.

## Issue 16 — `run_all.ps1` aborted on PostgreSQL notices

- **Symptoms:** second run failed with "NativeCommandError" on `extension ... already exists, skipping`.
- **Root cause:** PowerShell treats stderr output as an error with `$ErrorActionPreference = 'Stop'`.
- **Solution:** `PGOPTIONS='-c client_min_messages=warning'` at the top of `run_all.ps1` (and `run_all.sh`).
- **Final result:** scripts are re-runnable (verified twice, and on a scratch database).
