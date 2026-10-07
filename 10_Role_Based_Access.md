# 10 — Role-Based Access

Model: **users → roles → permissions** in PostgreSQL (`auth.user_roles`, `auth.role_permissions`). The API puts the role and permission codes in the JWT and enforces them with ASP.NET Core policies. Flutter and Angular only mirror them to shape menus and routes.

## Permissions (14, from `06_seed_data.sql` / `ILAPI.Core/Permissions.cs`)

`dashboard.view`, `motor.quote`, `home.quote`, `travel.quote`, `liability.quote`, `customers.view`, `quotes.view` (all quotes), `quotes.own` (own quotes), `underwriting.view`, `users.manage`, `roles.manage`, `reports.view`, `profile.view`, `policies.view`.

## Role matrix (verified in the database)

| Permission | ADMIN | AGENT | UNDERWRITER | MANAGER | CUSTOMER |
|---|:-:|:-:|:-:|:-:|:-:|
| dashboard.view | ✔ | ✔ | ✔ | ✔ | ✔ |
| motor.quote | ✔ | ✔ | | | |
| home.quote | ✔ | ✔ | | | |
| travel.quote | ✔ | ✔ | ✔ | ✔ | |
| liability.quote | ✔ | ✔ | ✔ | ✔ | |
| customers.view | ✔ | ✔ | | ✔ | |
| quotes.view | ✔ | ✔ | ✔ | ✔ | |
| quotes.own | | | | | ✔ |
| underwriting.view | | | ✔ | | |
| users.manage | ✔ | | | | |
| roles.manage | ✔ | | | | |
| reports.view | ✔ | | ✔ | ✔ | |
| profile.view | | | | | ✔ |
| policies.view | | | | | ✔ |

## Roles in practice

| Role | Purpose | Menu (top navigation) | Products | Dashboard |
|---|---|---|---|---|
| ADMIN | system administration | Dashboard, Motor, Home, Travel, Liability, Customers, Quotes, Users, Roles, Reports | all four | "Admin Dashboard": product cards for all four + recent quotes (all) |
| AGENT | sells products | Dashboard, Motor, Home, Travel, Liability, Customers, Quotes | all four | cards for all four + recent quotes (all) |
| UNDERWRITER | reviews quotes | Dashboard, Travel, Liability, Quotes, Underwriting, Reports | Travel, Liability | cards for Travel/Liability + recent quotes (all) |
| MANAGER | oversight | Dashboard, Travel, Liability, Customers, Quotes, Reports | Travel, Liability | cards for Travel/Liability + recent quotes (all) |
| CUSTOMER | self-service | Dashboard, My Profile, My Quotes, My Policies | none (cannot create quotes) | no product cards; own quotes |

The menu list comes from `allMenuItems` in `ILWEBFlutter/lib/shell/menu.dart`, filtered by `ApiClient.has(permission)`.

> Design note: Underwriter and Manager can create Travel/Liability quotes because the Angular access list in the requirements included them, although the sample menu for those roles did not list those products. Change `role_permissions` in `06_seed_data.sql` to alter this.

## Enforcement layers

| Layer | Mechanism | Strength |
|---|---|---|
| API | `[Authorize(Policy = "<permission>")]`; `quotes.view`/`quotes.own` checks inside `QuotesController` | **authoritative** (`401`/`403`) |
| Flutter | menu filtering; `redirect()` sends unauthorised routes to `/dashboard`; dashboard cards filtered | convenience |
| Angular | `authGuard`, `roleGuard` (AGENT/UNDERWRITER/MANAGER/ADMIN), `permissionGuard`; failures go to `/forbidden` | convenience |

The frontends never send a role; the API reads the role/permissions from the signed JWT.

## Demo users (development only)

Created by `ILDB/08_demo_users.sql`; **never seeded when `ILPORTAL_ENV=Production`**. Passwords are stored as bcrypt hashes. These are public sample credentials for local development:

| Role | Username | Password |
|---|---|---|
| ADMIN | `admin` | `Admin@123` |
| AGENT | `agent` | `Agent@123` |
| UNDERWRITER | `underwriter` | `Underwriter@123` |
| MANAGER | `manager` | `Manager@123` |
| CUSTOMER | `customer` | `Customer@123` |

The Flutter login page shows "Demo Login" buttons that fill these in when `enableDemoLogin` is true (`ILWEBFlutter/lib/core/config.dart`: on in `development`, opt-in for `qa` with `ENABLE_DEMO_LOGIN=true`, always off for `production`). The demo credentials are compiled out of a production build (verified by searching the built `main.dart.js`).

## Changing access

Edit the `role_permissions` insert in `ILDB/06_seed_data.sql` and re-run it (it is additive; add a `DELETE` for removals, as was done for ADMIN/underwriting). Add a new permission by adding it to `Permissions.cs`, the seed, and the front-end menu table.
