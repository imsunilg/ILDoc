# 05 — Routing and Navigation

Diagram: [diagrams/routing-flow.md](diagrams/routing-flow.md)

## Ownership

| Route space | Owner | Defined in |
|---|---|---|
| Browser URL / every portal route | **Flutter** (`go_router`) | `ILWEBFlutter/lib/core/routing/router.dart`, `lib/shell/menu.dart` |
| Routes *inside* the Travel/Liability iframe | Angular router | `ILWEBAngular/src/app/app.routes.ts` |

## Flutter routes (verified from `allMenuItems` + `router.dart`)

| Route | Page | Frontend | Needs permission |
|---|---|---|---|
| `/login` | `LoginScreen` | Flutter | — (public) |
| `/dashboard` | `DashboardHome` | Flutter | `dashboard.view` |
| `/motor` | `MotorScreen` → `QuoteFormView('motor')` | Flutter | `motor.quote` |
| `/home` | `HomeScreen` → `QuoteFormView('home')` | Flutter | `home.quote` |
| `/travel` | `TravelScreen` → iframe `/travel` | **Angular** | `travel.quote` |
| `/liability` | `LiabilityScreen` → iframe `/liability` | **Angular** | `liability.quote` |
| `/customers` | `DataListView` (`GET /api/customers`) | Flutter | `customers.view` |
| `/quotes` | `DataListView` (`GET /api/quotes`) | Flutter | `quotes.view` |
| `/underwriting` | `DataListView` (`GET /api/underwriting/queue`) | Flutter | `underwriting.view` |
| `/users` | `DataListView` (`GET /api/users`) | Flutter | `users.manage` |
| `/roles` | `DataListView` (`GET /api/roles`) | Flutter | `roles.manage` |
| `/reports` | `DataListView` (`GET /api/reports/summary`) | Flutter | `reports.view` |
| `/profile` | `DataListView` (`GET /api/v1/auth/me`) | Flutter | `profile.view` |
| `/my-quotes` | `DataListView` (`GET /api/quotes`, own quotes) | Flutter | `quotes.own` |
| `/policies` | placeholder text ("You have no policies yet.") | Flutter | `policies.view` |

There are no `/motor/quote` or `/home/quote` sub-routes: the quote form *is* the `/motor` and `/home` page and shows the result below the form.

### Redirect rules (`buildRouter`)

1. Not signed in → `/login` (any other path).
2. Signed in and path is `/login` or `/` → `/dashboard`.
3. Path not in the menu table, or the user lacks every permission mapped to it → `/dashboard`.
4. `refreshListenable: ApiClient`, so login/logout/refresh-failure re-evaluates routing immediately.

### URL style

`usePathUrlStrategy()` in `lib/main.dart` gives `http://localhost:8080/liability` (not `/#/liability`). Consequence: the host must serve `index.html` for unknown paths (the local static server and `flutter run` do).

## Angular routes (verified from `app.routes.ts`)

| Route | Component | Guards |
|---|---|---|
| `/travel` | `TravelQuotePage` (form) | `authGuard`, `roleGuard` (AGENT, UNDERWRITER, MANAGER, ADMIN), `permissionGuard` (`travel.quote`) |
| `/travel/result` | `TravelResultPage` | same |
| `/liability` | `LiabilityQuotePage` | `authGuard`, `roleGuard`, `permissionGuard` (`liability.quote`) |
| `/liability/result` | `LiabilityResultPage` | same |
| `/forbidden` | `Forbidden` | — |
| `**` | redirect to `/forbidden` | — |

Angular has **no** `/login`, `/dashboard`, `/profile` or `/logout` routes. Not built: `/travel/quote`, `/travel/customer`, `/liability/quote`, `/liability/customer` (the form is the module's root route).

## How Angular is activated

Selecting the Travel/Liability route makes Flutter build `AngularModuleView`, which loads `<ANGULAR_URL>/<module>` in an iframe. Leaving the route disposes the widget and removes the iframe; Angular is not running while another page is shown.

## Back / forward

| Action | Result (verified by `ILWEBFlutter/scripts/e2e-portal.mjs`) |
|---|---|
| Dashboard → Travel → Get quote → Back | returns to the Travel form, header unchanged |
| Back again | returns to `/dashboard` |
| Forward | returns to the page just left |
| Same for Liability | same |

The iframe's own navigation joins the tab's session history, so Back steps through Angular pages first and then Flutter pages.

## Refresh and deep links

| Action | Behaviour |
|---|---|
| Refresh on `/dashboard`, `/motor`, … | Flutter restores the session from local storage and re-renders the same route |
| Refresh on `/travel` | Flutter reloads, recreates the iframe, repeats the handoff; the form is shown |
| Refresh on `/travel/result` (iframe URL, not the Flutter URL) | the Flutter URL is `/travel`, so the form loads; the result needs `history.state`, which is gone, so the result component would redirect to the form anyway |
| Opening a protected URL while signed out | redirected to `/login` (the target is not remembered after login — planned) |
| Opening `http://localhost:4200/travel` directly | Angular shows "open it from the portal" |

## Logout

`Sign out` (header icon, semantic label "Sign out") → `ApiClient.logout()` → router redirects to `/login`, iframe disposed. See [04](04_Authentication_Flow.md).

## Active menu indication

`PortalShell` marks the active item by comparing the current location with the item's route: orange icon and underline plus bold maroon text in the top navigation (`_TopNav`); selected tile in the drawer on narrow screens (< 900 px).
