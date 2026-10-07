# 07 — API Integration

Project: `ILAPI` (.NET 9). Default dev URL used by both front ends: `http://localhost:5080` (started with `--urls http://localhost:5080`; `launchSettings.json` defaults to `5272`).

Response convention: **auth endpoints** return `{ "success": bool, "message": string, "data": ... }`. **Quote, product and admin endpoints** return the plain JSON object/array (errors as `{ "error": "..." }`).

## Endpoints (all verified in `ILAPI.API/Controllers`)

### Authentication — `AuthController`, base `api/v1/auth`

| Method | Endpoint | Auth | Purpose | Notes |
|---|---|---|---|---|
| POST | `/api/v1/auth/register` | public | create a CUSTOMER account | body `{ email, password (≥ 8), fullName, phone? }`; `409` if email exists |
| POST | `/api/v1/auth/login` | public | sign in | body `{ username, password }` (username or email) → `AuthData` |
| POST | `/api/v1/auth/refresh` | public (refresh token) | rotate tokens | body `{ refreshToken }` |
| POST | `/api/v1/auth/logout` | public (refresh token) | revoke refresh token | body `{ refreshToken }` |
| POST | `/api/v1/auth/handoff` and `/api/v1/frontend-session` | JWT | create one-time frontend token | → `{ token }` (60 s, single use) |
| POST | `/api/v1/auth/handoff/exchange` | public (one-time token) | exchange token for a session | body `{ token }` → `AuthData` |
| GET | `/api/v1/auth/me` | JWT | current user | → `UserInfo` (used by the Profile page) |

`AuthData` = `{ accessToken, refreshToken, expiresIn, user: { userId, username, displayName, role, permissions[] } }`.

### Products and quotes — `ProductsController`, `QuotesController` (file `QuotesController.cs`)

| Method | Endpoint | Auth / policy | Purpose |
|---|---|---|---|
| GET | `/api/products` | public | active products (`product.fn_active_products()`) |
| GET | `/api/quotes` | JWT; `quotes.view` → all quotes, `quotes.own` → own quotes, else `403` | quote list (max 200) |
| GET | `/api/quotes/{id}` | JWT; same rule | one quote (`404` if not visible) |
| POST | `/api/quotes/motor` | policy `motor.quote` | create Motor quote |
| POST | `/api/quotes/home` | policy `home.quote` | create Home quote |
| POST | `/api/quotes/travel` | policy `travel.quote` | create Travel quote |
| POST | `/api/quotes/liability` | policy `liability.quote` | create Liability quote |

Quote POST success: `201 Created` with `QuoteSummary { id, quoteNumber, status, productCode, productName, basePremium, tax, totalPremium, createdAt, expiresAt }`. Invalid input: `400 { error }`. Request fields are listed in [09](09_Quote_Generation_Flow.md).

### Read endpoints for admin pages — `AdminController`, base `api`

| Method | Endpoint | Policy | Source |
|---|---|---|---|
| GET | `/api/customers` | `customers.view` | `customer.customers` |
| GET | `/api/users` | `users.manage` | `auth.users` + roles |
| GET | `/api/roles` | `roles.manage` | `auth.roles` + permissions |
| GET | `/api/reports/summary` | `reports.view` | `quote.v_quote_summary` grouped by product/status |
| GET | `/api/underwriting/queue` | `underwriting.view` | quotes with status `quoted` |

### Other

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | ASP.NET health check including a PostgreSQL connectivity check |
| GET | `/openapi/v1.json` | OpenAPI document (Development only) |

**Not present (do not rely on them):** dedicated dashboard endpoints, customer CRUD, policy endpoints, endpoints to change users/roles (the Users/Roles APIs are read-only), per-product GET endpoints.

## Example

```http
POST /api/v1/auth/login
Content-Type: application/json

{ "username": "agent", "password": "<demo password, see 10>" }
```
```json
{ "success": true, "message": "Login successful",
  "data": { "accessToken": "<jwt>", "refreshToken": "<opaque>", "expiresIn": 900,
            "user": { "userId": "…", "username": "agent", "displayName": "Demo Agent",
                      "role": "AGENT", "permissions": ["customers.view", "dashboard.view", "…"] } } }
```

```http
POST /api/quotes/travel
Authorization: Bearer <jwt>

{ "destinationCode": "FR", "startDate": "2027-02-01", "endDate": "2027-02-07", "travellers": 2, "tripCost": 3000 }
```

## Code map

| File | Role |
|---|---|
| `ILAPI.API/Program.cs` | DI, JWT bearer, one authorization policy per permission, CORS (`Cors:Origins`), health check, startup DB check |
| `ILAPI.API/Services/TokenService.cs` | access JWT, opaque tokens, SHA-256 hashing, lifetimes |
| `ILAPI.API/Services/ClaimsExtensions.cs` | `User.UserId()` from the `sub` claim |
| `ILAPI.Core/Models.cs` | request/response records |
| `ILAPI.Core/Permissions.cs` | permission codes (= policy names = JWT claim values) |
| `ILAPI.Core/PremiumCalculator.cs` | premium rules |
| `ILAPI.Infrastructure/Repositories.cs` | all SQL (Dapper): users, tokens, products, quotes, admin reads |
| `ILAPI.Infrastructure/DependencyInjection.cs` | `NpgsqlDataSource`, repositories, `PostgresHealthCheck` |

## CORS

`appsettings.json` → `Cors:Origins`: `http://localhost:4200`, `http://localhost:8080`. Add the deployed Flutter and Angular origins per environment.

## Configuration and secrets

`ConnectionStrings:PostgreSQL` and `Jwt:Key` are placeholders in `appsettings.json` and must be supplied by `dotnet user-secrets`, environment variables (`ConnectionStrings__PostgreSQL`, `Jwt__Key`) or `appsettings.Development.local.json` (git-ignored). The API refuses to start with the placeholders.
