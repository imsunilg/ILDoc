# 17 — Future Enhancements

All items are **PLANNED / not implemented**. Priorities reflect risk and effort, not a commitment.

## Security and authentication (high)

| Item | Why |
|---|---|
| Revoke Angular's refresh token when Flutter logs out (e.g. `il-logout` message or revoke-all-for-user) | closes the known gap in [04](04_Authentication_Flow.md) |
| Move session storage to HttpOnly, Secure, SameSite cookies (backend-for-frontend) | removes tokens from script-readable storage |
| Rate limiting and lockout on `/api/v1/auth/login` | brute-force protection |
| Content-Security-Policy `frame-ancestors` / `frame-src` between the two hosts | enforce who may embed whom |
| Remember the target URL across login | better deep-link UX |
| Audit log writes (`audit.audit_log` exists but is unused) | traceability |
| Application database role with least privilege | production hardening |

## Product and business (high/medium)

| Item | Notes |
|---|---|
| Multi-step quote flows (customer details, coverage, review) | currently a single form per product |
| Staff selecting a customer for a quote | today staff quotes attach to the staff member's own customer row |
| Approved rating tables and tax rules | current rates/formulas are placeholders |
| Policy issuance, payments, renewals, documents | menu item "My Policies" is a placeholder |
| Underwriting workflow (approve/refer/decline, status changes) | `underwriting/queue` is read-only; `quote.fn_expire_quotes()` is not scheduled |
| User and role administration (create/edit) | `/api/users`, `/api/roles` are read-only |
| Customer self-service quoting (permission change) | currently customers cannot create quotes |

## Architecture (medium)

| Item | Notes |
|---|---|
| Environment-driven Angular configuration (`API_URL`, `PORTAL_ORIGINS`) | required for QA/UAT/Production |
| OpenAPI-generated clients for Flutter and Angular | removes duplicated request/response models |
| Shared design tokens (single source for the palette) | today kept in sync by hand |
| Generic module contract (module manifest: route, permission, URL) so Flutter can host more Angular modules without code changes | |
| Optional alternative to iframes (module federation or Angular Elements) if tighter integration is ever needed | evaluate cost against the isolation iframes give |
| Move quote creation + premium logic to PostgreSQL functions only if the business wants rules in the database (the current design keeps them in C#) | |

## Quality and operations (medium)

| Item | Notes |
|---|---|
| API integration tests (`WebApplicationFactory` + test database) | |
| Tests for token refresh in `ApiClient` and Angular `AuthService`/interceptor | |
| CI pipeline running all test suites and the browser test | |
| Dockerfiles and docker-compose (PostgreSQL, API, two static hosts) | |
| Structured logging, metrics, health dashboards | |
| Accessibility audit (Flutter semantics, Angular forms) and cross-browser tests | |
| Push the four repositories to GitHub and add branch protection | `gh auth login` first |
