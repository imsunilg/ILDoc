# 03 — Flutter + Angular Integration

Diagrams: [flutter-angular-flow](diagrams/flutter-angular-flow.md), [authentication-flow](diagrams/authentication-flow.md)

## Mechanism (verified in source)

**Flutter hosts Angular in an `iframe`.** There is no web component, Angular Elements build, module federation or JavaScript global bridge. The two apps talk only through `window.postMessage` for the session handoff; everything else goes through the shared API.

| Concern | Implementation | File |
|---|---|---|
| Embedding | `AngularModuleView` creates an `HTMLIFrameElement` (`src = <ANGULAR_URL>/<module>`) and registers it with `ui_web.platformViewRegistry`; shown with `HtmlElementView` | `ILWEBFlutter/lib/shared/angular_module_view_web.dart` |
| Non-web fallback | placeholder widget so VM unit tests compile | `ILWEBFlutter/lib/shared/angular_module_view_stub.dart` (selected by `angular_module_view.dart` via `if (dart.library.js_interop)`) |
| Which routes embed | `/travel` → `TravelScreen`, `/liability` → `LiabilityScreen`, each is `AngularModuleView(module: ...)` | `lib/features/travel/travel_screen.dart`, `lib/features/liability/liability_screen.dart`, `lib/shell/menu.dart` |
| Angular entry | `src/main.ts` bootstraps `App` (`app.ts/html`): a content-only host showing the router outlet once the session is ready | `ILWEBAngular/src/app/app.ts`, `app.html` |
| Session handoff (Angular side) | `AuthService.connect()` → `requestHandoff()` → `exchange()` | `ILWEBAngular/src/app/core/auth.service.ts` |
| Allowed origins | `PORTAL_ORIGINS` (Flutter origins), `API_URL`, `STAFF_ROLES` | `ILWEBAngular/src/app/core/config.ts` |
| Token endpoint | `POST /api/v1/frontend-session` (alias of `/api/v1/auth/handoff`), exchange at `POST /api/v1/auth/handoff/exchange` | `ILAPI/ILAPI.API/Controllers/AuthController.cs` |

## Initialization sequence

1. User clicks **Travel** (or opens `/travel`). `go_router` shows `TravelScreen` inside `PortalShell`; the header and navigation stay.
2. `AngularModuleView.initState` creates the iframe, registers it, and adds a `message` listener on the Flutter window.
3. Angular bootstraps in the iframe. `App` calls `AuthService.connect()`. If `window.parent === window` (opened directly) the status becomes `standalone` and the page shows "open it from the portal".
4. Angular posts `{ type: 'il-ready' }` to its parent (`'*'` as target origin: the message holds no secret).
5. Flutter checks `event.origin == angularOrigin` (derived from `ANGULAR_URL`) and `data.type == 'il-ready'`, then calls `ApiClient.createFrontendSession()` → `POST /api/v1/frontend-session`.
6. Flutter posts `{ type: 'il-handoff', token }` to the iframe window with `targetOrigin = angularOrigin`.
7. Angular accepts the message only if `e.source === window.parent`, `e.origin ∈ PORTAL_ORIGINS`, `type == 'il-handoff'`; it calls `POST /api/v1/auth/handoff/exchange` and keeps the resulting session **in memory**.
8. Status becomes `ready`; the router outlet renders. Route guards (`authGuard`, `roleGuard`, `permissionGuard`) run after `whenReady()`.

## Handoff token properties (API side)

| Property | Value | Source |
|---|---|---|
| Format | 48 random bytes, base64url | `TokenService.NewOpaqueToken` |
| Stored | only its SHA-256 hash in `auth.handoff_tokens` | `TokenRepository.StoreHandoffAsync` |
| Lifetime | `Jwt:HandoffSeconds` = 60 | `appsettings.json` |
| Single use | `UPDATE ... SET used_at = now() WHERE used_at IS NULL AND expires_at > now() RETURNING user_id` | `ConsumeHandoffAsync` |
| Contains | nothing but a lookup key; it is not the access or refresh token and no password | |

## Navigation handoff

Flutter owns navigation. Angular is told only *which module* through the iframe URL path (`/travel` or `/liability`). Angular's own pages (`/travel` → `/travel/result`) change inside the iframe; the Flutter URL stays on `/travel`.

## Data handoff

None between the front ends. A quote created in Angular is saved by the API; Flutter's dashboard and `/quotes` list read it from the API. The Angular result page receives its quote through router `state` (`history.state.quote`), and redirects back to the form if reloaded without it.

## API communication

Both apps call the same API with `Authorization: Bearer <access token>`. Flutter uses `ApiClient` (`lib/core/api.dart`); Angular uses `HttpClient` with `authInterceptor` (`core/auth.interceptor.ts`).

## Error handling

| Situation | Behaviour |
|---|---|
| Angular opened outside the portal | "This module runs inside the portal…" |
| No reply to `il-ready` within 8 s, or exchange fails | status `failed` → "Session unavailable" |
| Access token expired in Angular | interceptor refreshes once; if that fails, requests a new handoff from Flutter |
| Role/permission missing | guard redirects to `/forbidden` ("Not available"); API would return 403 anyway |
| 403 from the quote POST | form shows "You do not have permission to create this quote." |

## Browser history, back, refresh

See [05_Routing_and_Navigation.md](05_Routing_and_Navigation.md). Refreshing `/travel` reloads Flutter (session restored from local storage), which recreates the iframe and repeats the handoff.

## Logout and session expiry

See [04_Authentication_Flow.md](04_Authentication_Flow.md) §Logout. Flutter logout removes the iframe; Angular's in-memory session disappears with it.

## Security notes (read before deploying)

- `postMessage` replies are origin-checked on both sides. The Flutter side does **not** compare `event.source` (touching the cross-origin window object throws a `SecurityError` in Dart); the origin check is the control there.
- The iframe is created with the default sandbox (none). If you add a `sandbox` attribute, include `allow-scripts allow-same-origin`.
- `PORTAL_ORIGINS` is hard-coded to `http://localhost:8090` / `http://127.0.0.1:8090`; it must be changed per environment (see [14](14_Build_and_Deployment.md)).
