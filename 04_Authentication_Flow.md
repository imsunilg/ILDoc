# 04 — Authentication Flow

Diagram: [diagrams/authentication-flow.md](diagrams/authentication-flow.md)

## Components

| Piece | Implementation | File |
|---|---|---|
| Credential check | `FindByLoginAsync` (username **or** email, case-insensitive `citext`) then `BCrypt.Net.BCrypt.Verify` | `ILAPI.API/Controllers/AuthController.cs`, `ILAPI.Infrastructure/Repositories.cs` |
| Password hashing | bcrypt. Registered users: `BCrypt.HashPassword` (library default cost). Demo users: `crypt(pw, gen_salt('bf', 11))` from pgcrypto, so no plain text is stored | `AuthController.Register`, `ILDB/08_demo_users.sql` |
| Access token | JWT, HS256, issuer `ILAPI`, audience `ILPortal`, **15 minutes** (`Jwt:AccessMinutes`) | `Services/TokenService.cs` |
| JWT claims | `sub`, `email`, `preferred_username`, `name`, `role`, one `permission` claim per permission | `TokenService.CreateAccessToken` |
| Signing key | `Jwt:Key`, **not in the repo** (user-secrets / `Jwt__Key` env var); startup fails if missing, a placeholder, or < 32 chars | `Program.cs` |
| Refresh token | 48 random bytes; only the SHA-256 hash is stored (`auth.refresh_tokens`); **14 days**; rotated on every refresh (old one revoked) | `TokenService`, `TokenRepository.ConsumeRefreshAsync` |
| Validation | `JwtBearer`: issuer, audience, signature, lifetime, 30 s clock skew; `MapInboundClaims = false`, role claim `role` | `Program.cs` |
| Authorization | one policy per permission code, `RequireClaim("permission", code)` | `Program.cs`, `ILAPI.Core/Permissions.cs` |

## Login (Flutter)

1. `LoginScreen` → `ApiClient.login(username, password)` → `POST /api/v1/auth/login`.
2. On success the response envelope carries `accessToken`, `refreshToken`, `expiresIn` (seconds) and `user { userId, username, displayName, role, permissions[] }`.
3. `ApiClient` stores the whole session as JSON in browser storage via `shared_preferences` (key `il.session`) and notifies listeners; `go_router`'s `refreshListenable` redirects `/login` → `/dashboard`.
4. Wrong password and unknown user return the same `401` message ("Invalid username or password"); the failed username (never the password) is logged at warning level.

## Session state

| App | Where the session lives | Notes |
|---|---|---|
| Flutter | `ApiClient` (memory) + `shared_preferences` (browser local storage) | survives refresh; contains the refresh token (see risks) |
| Angular | `AuthService` signal, **memory only** | re-obtained through the handoff on every page load of the module |

## Angular authentication

Angular has no login. Section 2 of the [diagram](diagrams/authentication-flow.md) and [03](03_Flutter_Angular_Integration.md) describe the handoff. Angular ends up with its **own** access and refresh token pair for the same user, issued by `handoff/exchange`.

## Refresh and expiry

- `401` on any API call → exactly one `POST /api/v1/auth/refresh` → retry once. Implemented in `ApiClient._send` (Flutter) and `authInterceptor` (Angular).
- Flutter: refresh failure clears the session and the router returns to `/login`.
- Angular: refresh failure asks Flutter for a new handoff token; if that fails the module shows "Session unavailable".
- Verified: a token issued with a past expiry is rejected with `401` (`scripts/verify-roles.ps1`, expired-token check).

## Logout

| Step | Behaviour |
|---|---|
| Flutter | `ApiClient.logout()` → `POST /api/v1/auth/logout` (revokes Flutter's refresh token, best effort) → delete stored session → redirect to `/login` |
| Angular | the iframe is destroyed with the page; its in-memory session is lost |
| Server | the 15-minute access tokens are stateless and remain valid until they expire |

## Unauthorized and forbidden requests

| Case | Response |
|---|---|
| No / invalid / expired token | `401` |
| Valid token, missing permission | `403` |
| Flutter menu item or route without permission | hidden / redirected to `/dashboard` (UI convenience only) |

## Known limitations (be explicit)

1. **Angular's refresh token is not revoked on Flutter logout.** It lives only in iframe memory, but a copy would stay valid for up to 14 days if exfiltrated. Mitigation planned: revoke all of a user's tokens on logout (see [17](17_Future_Enhancements.md)).
2. Flutter keeps the refresh token in browser local storage (readable by any script on the origin). Acceptable for development; for production prefer an HttpOnly cookie flow.
3. Permissions in a JWT are fixed until it expires (≤ 15 minutes); a role change takes effect at the next refresh/login.
4. There is no account lockout or rate limiting on `/login` yet.

## Secrets policy

No password, key or token is committed. The database password lives in `ILPortal/.env` (git-ignored) and `dotnet user-secrets`; the JWT key in user-secrets. Demo credentials are documented in [10_Role_Based_Access.md](10_Role_Based_Access.md) and exist only for development.
