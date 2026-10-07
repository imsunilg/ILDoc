# 15 — Testing Strategy

## What exists (verified results from the latest runs)

| Layer | Tool | Location | Result |
|---|---|---|---|
| API unit | xUnit | `ILAPI/ILAPI.Tests/PremiumCalculatorTests.cs` | 10 / 10 pass |
| API authorization matrix | PowerShell against a running API | `ILAPI/scripts/verify-roles.ps1` | 82 checks pass (81 without the expired-token instance) |
| Angular unit | vitest via `ng test` | `ILWEBAngular/src/app/core/guards.spec.ts` | 5 / 5 pass |
| Flutter unit/widget | `flutter test` | `ILWEBFlutter/test/widget_test.dart` | 12 / 12 pass (also with `--dart-define=APP_ENV=production`) |
| Static analysis | `flutter analyze`, `ng build`, `dotnet build` | — | clean |
| Browser end-to-end | Puppeteer + Chrome (headless) | `ILWEBFlutter/scripts/e2e-portal.mjs` | passes for ADMIN, AGENT, UNDERWRITER, MANAGER, CUSTOMER |
| Data export/restore | scratch database, row counts | manual, `ILDB/load_data.ps1` | counts matched; second load idempotent |

## Automated scenarios

### Authentication (`verify-roles.ps1`)
Valid login for each role (success envelope, role in response and JWT, permissions present, `expiresIn > 0`); wrong password → 401; unknown user → 401; no token → 401; tampered token → 401; expired token → 401; logout → 200 and the refresh token is rejected afterwards.

### Authorization
Per role and endpoint (`customers`, `users`, `roles`, `reports/summary`, `underwriting/queue`, `quotes`, `POST motor`, `POST travel`) the expected `200/201/403` is asserted.

### Flutter
Login form fields, register mode, demo-login visibility by environment, auto-fill and "Demo Account" label, password visibility toggle, role menus (per-role menu lists), role titles.

### Angular
`authGuard` (blocks without handoff, allows with, waits for the handoff to finish), `roleGuard` (allowed / `/forbidden`), `permissionGuard`.

### Integration (browser, per role, using the real API and database)
1. Flutter login screen appears; choose the role with the demo button; "Demo Account: ROLE" shown.
2. Login → "`<Role>` Dashboard"; no Angular iframe.
3. Motor and Home (roles that have them): Flutter page appears, header unchanged, no iframe.
4. Travel and Liability (roles that have them): iframe present, Angular page rendered, header still visible, **no** header/nav element and no Sign in/Sign out/Dashboard text inside the iframe, no token/JWT in any URL.
5. Travel quote created through Angular (result shows a total); browser Back returns to the form.
6. Liability quote created through Angular.
7. Dashboard click returns to the Flutter dashboard and removes the iframe; browser Back/Forward change the Flutter route.
8. Sign out → Flutter login; Angular `/login` never visited.

### Quote generation
Motor/Home/Travel/Liability premium rules and validation are unit-tested in `PremiumCalculatorTests`; end-to-end creation is exercised for Motor/Travel by `verify-roles.ps1` and for Travel/Liability by the browser test.

## Manual checklist (before a release)

- [ ] Invalid login shows an error and stays on `/login`.
- [ ] Refresh on `/dashboard`, `/motor`, `/travel` keeps the session.
- [ ] Expired access token: next action refreshes silently; expired refresh token returns to `/login`.
- [ ] Customer cannot open `/motor` (redirected) and gets `403` from the API.
- [ ] Narrow window (< 900 px): navigation drawer works.
- [ ] Production build: no "Demo Login" section.

## Gaps (honest list)

- No API integration tests with `WebApplicationFactory`; controller behaviour is checked by the script only.
- No tests for `ApiClient` (Flutter) refresh logic or Angular `AuthService`/interceptor refresh paths.
- No Home/Liability-specific unit tests beyond the calculator.
- Browser test is Chrome only and requires a release build of Flutter.
- No CI to run any of this automatically.
- No load, accessibility or security (SAST/DAST) testing.

## Test data

Demo users and the exported quotes come from `ILDB`. The browser test creates real quote rows each run.
