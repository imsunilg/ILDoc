# 06 — Product Flow

## Responsibility matrix (verified)

| Product | UI owner | UI file(s) | Session | API endpoint | Tables written |
|---|---|---|---|---|---|
| Motor | **Flutter** | `ILWEBFlutter/lib/features/motor/motor_screen.dart` → `lib/shared/quote_form_view.dart` | Flutter's own | `POST /api/quotes/motor` | `quote.quotes`, `motor.vehicle_quotes` |
| Home | **Flutter** | `lib/features/home/home_screen.dart` → `quote_form_view.dart` | Flutter's own | `POST /api/quotes/home` | `quote.quotes`, `home.property_quotes` |
| Travel | **Angular** | `ILWEBAngular/src/app/travel/` + `shared/quote-form`, `shared/quote-result` | handed over by Flutter | `POST /api/quotes/travel` | `quote.quotes`, `travel.trip_quotes` |
| Liability | **Angular** | `ILWEBAngular/src/app/liability/` + shared components | handed over by Flutter | `POST /api/quotes/liability` | `quote.quotes`, `liability.liability_quotes` |

All four share one backend and one database; the front ends never talk to PostgreSQL.

## User flows

### Login

```text
User → Flutter /login → POST /api/v1/auth/login → ILAPI → PostgreSQL (auth.users, roles, permissions)
     → JWT + refresh token + role + permissions → Flutter /dashboard
```

### Motor / Home (Flutter)

```text
Flutter header → Motor (or Home) → /motor (/home) → QuoteFormView
  → POST /api/quotes/motor (home) with JWT → ILAPI (policy motor.quote / home.quote)
  → PremiumCalculator → INSERT quote + detail → PostgreSQL → result card in the same page
```

### Travel / Liability (Angular inside Flutter)

```text
Flutter header → Travel (or Liability) → /travel (/liability)
  → iframe → Angular QuoteForm → (session handoff, see 03)
  → POST /api/quotes/travel (liability) with Angular's JWT → ILAPI (policy travel.quote / liability.quote)
  → PremiumCalculator → INSERT quote + detail → PostgreSQL
  → Angular /travel/result (result page) — Flutter header unchanged
```

## Product availability by role

| Product | ADMIN | AGENT | UNDERWRITER | MANAGER | CUSTOMER |
|---|---|---|---|---|---|
| Motor | yes | yes | no | no | no |
| Home | yes | yes | no | no | no |
| Travel | yes | yes | yes | yes | no |
| Liability | yes | yes | yes | yes | no |

Travel and Liability are additionally guarded in Angular by a role check (`AGENT`, `UNDERWRITER`, `MANAGER`, `ADMIN`) and by the API. Customers can view their own quotes but cannot create quotes in the current permission set (this follows the requested menu layout; changing it is a seed-data change, see [10](10_Role_Based_Access.md)).

## Dashboard behaviour

`DashboardHome` fetches `GET /api/products` and shows a card only for products whose `<code>.quote` permission the user holds, plus the quote list (`GET /api/quotes`: all quotes for roles with `quotes.view`, own quotes for `quotes.own`). The card button navigates to `/motor`, `/home`, `/travel` or `/liability`.
