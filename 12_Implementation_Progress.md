# 12 — Implementation Progress

Chronology is taken from the git histories (all work happened on 2026-10-06 and 2026-10-07). Hashes are per-repository. Status words: COMPLETED, PARTIAL, PENDING (nothing is "IN PROGRESS" at the time of writing).

## Change history at a glance

```text
Bootstrap prompt (database + GitHub rules)
   ↓
PostgreSQL database + schemas + tables + seed          (ILDB)
   ↓
.NET API: premium calculator, repositories, JWT, quotes (ILAPI)
   ↓
Angular app built as a full portal (login, dashboard, header)   ┐ two similar portals
Flutter app built as a full portal (login, dashboard, quotes)   ┘
   ↓
Role system: roles, permissions, demo users, v1 auth    (all repos)
   ↓
Integration attempt: Flutter opens Angular with /handoff?token=
   ↓
Duplicate header / dashboard / login observed (screenshots)
   ↓
Decision: Flutter = shell, Angular = Travel/Liability modules
   ↓
iframe + postMessage handoff, go_router, Angular stripped to modules
   ↓
Path URLs, UI redesign, per-component html/css/ts, data export
   ↓
Four separate repositories (ILAPI, ILDB, ILWEBFlutter, ILWEBAngular)
```

## Phases

### Phase 1 — Project setup and repository rules — COMPLETED
- **Implemented:** folder layout, secret-handling rules (placeholders in `appsettings.json`, `.env` git-ignored, `PGPASSWORD`), conventional commits. Later split into four repos (see Phase 13).
- **Dependencies:** git, PostgreSQL 16, .NET 9 SDK, Node, Flutter, Angular CLI (GitHub CLI installed).
- **Issues/resolution:** the secret scanner's own pattern contained the password literal; replaced by reading the value from `.env` (history rewritten locally before any push).
- **Current state:** local repositories only; **no GitHub remote** (CLI not authenticated).

### Phase 2 — Database — COMPLETED
- **Implemented:** `ILPortal` database, 10 schemas, tables, indexes, 3 functions, 1 view, seed data, `run_all` scripts. (`ILDB` `0d4df3b`, `d737a0f`)
- **Resolution notes:** `NOTICE` handling in PowerShell (Issue 16).
- **Current state:** idempotent, verified twice and on a scratch database.

### Phase 3 — API — COMPLETED (see testing gaps)
- **Implemented:** solution (Core/Infrastructure/API/Tests), premium calculator + 10 unit tests, Npgsql/Dapper repositories, health check, products and quote endpoints. (`ILAPI` `6436166`, `f3961b6`, `b40c11f`)
- **Gap:** no integration tests for controllers (covered by scripts).

### Phase 4 — Flutter application — COMPLETED
- **Implemented:** web client with login, dashboard, quote forms, session in `shared_preferences`. (`ILWEBFlutter` `e96977c`)

### Phase 5 — Angular application — COMPLETED (then restructured)
- **Implemented:** first as a full portal (login, handoff page, dashboard, quote forms) (`ILWEBAngular` `7467241`); later reduced to modules (Phase 9).

### Phase 6 — Authentication and roles — COMPLETED
- **Implemented:** `/api/v1/auth` with envelope, JWT with `role` and `permission` claims, refresh tokens, logout, roles/permissions tables, permission policies, role-gated admin endpoints, five demo users, Flutter demo-login buttons (environment-gated), Angular guards. (`ILAPI` `dd07e68`, `ILDB` `36c01d7`, `b8d91af`, `ILWEBFlutter` `9d1cd00`, `ILWEBAngular` `022596a`)
- **Verification:** `scripts/verify-roles.ps1` (82 checks: logins, JWT roles, 401/403 matrix, logout, expired token).

### Phase 7 — First Flutter/Angular integration — SUPERSEDED
- **Implemented:** Flutter button "open web portal" opened Angular in a new tab with a one-time token in the URL (`e96977c`).
- **Issues:** duplicate shells and a token in the URL (Issues 1–4). Replaced in Phase 9.

### Phase 8 — Decision: Flutter shell — COMPLETED
- Architecture rules recorded in `ILPortal/README.md` and [02](02_Architecture.md).

### Phase 9 — Integration rework (iframe + handoff + routing) — COMPLETED
- **Implemented:** Flutter `go_router` routes and `PortalShell`; `AngularModuleView`; API `frontend-session` alias; Angular `app.ts` content host, `core/auth.service.ts` handoff, guards, travel/liability modules, shared quote form/result. (`ILWEBFlutter` `46ee03e`, `ILWEBAngular` `4c6fcc2`, `ILAPI` `3c937df`)
- **Issues:** `SecurityError` (11), guard injection context (12).
- **Current state:** works for all five roles in the browser test.

### Phase 10 — Product routing and quote generation — COMPLETED (single-step forms)
- **Implemented:** Motor/Home in Flutter, Travel/Liability in Angular, one pipeline in the API; staff customer-row creation (Issue 13).
- **Not implemented:** multi-step wizards, customer-details step, coverage choice other than Motor.

### Phase 11 — Integration fixes and polish — COMPLETED
- Path URLs (`ad549f5`), README per repo (`6e42509`, `48f46dc`, `2115a67`), per-component `.ts/.html/.css` in Angular (`5296080`), visual redesign (`fc48ce3`, `f2428b5`).

### Phase 12 — Testing — PARTIAL
- **Done:** 10 .NET unit tests; 12 Flutter tests; 5 Angular tests; `verify-roles.ps1`; browser e2e `e2e-portal.mjs` for the 5 roles; data-load verification.
- **Missing:** API integration tests (WebApplicationFactory), CI, accessibility and performance tests, cross-browser runs.

### Phase 13 — Data export and repository split — COMPLETED
- INSERT-script export and loader (`ILDB` `84d529b`); four separate git repositories created from the monorepo history with `git subtree split`; the monorepo `.git` was moved to a backup outside the project. Note: `ILDB` history contains one commit labelled `feat(flutter)` (`b8d91af`) because it touched `ILDB/run_all.sh` and the seed script.

### Phase 14 — Documentation — COMPLETED (this folder)

### Pending / not started
| Item | Status |
|---|---|
| GitHub remotes and pushes | PENDING (`gh auth login` required) |
| Dockerfiles / docker-compose / CI | PENDING |
| Environment-specific Angular configuration (`API_URL`, `PORTAL_ORIGINS`) | PENDING (hard-coded constants) |
| Policies, payments, underwriting workflow | PENDING |
| Audit logging (`audit.audit_log` is unused) | PENDING |

## Commit history by repository

Run `git log --reverse --format='%h %ad %s' --date=short` inside each folder. At the time of writing: ILAPI 6 commits, ILDB 6, ILWEBFlutter 6, ILWEBAngular 6.
