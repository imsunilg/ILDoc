# 02 — Architecture

Diagram: [diagrams/overall-architecture.md](diagrams/overall-architecture.md)

## Logical architecture

```text
Browser
 └── Flutter web app  (shell)  ── http://localhost:8080 (dev)
       ├── Flutter pages: login, dashboard, Motor, Home, lists, profile
       └── iframe ──► Angular web app (modules) ── http://localhost:4200 (dev)
                         ├── /travel      /travel/result
                         └── /liability   /liability/result
 both ──► ILAPI (ASP.NET Core, JWT) ── http://localhost:5080 (dev)
              └── Npgsql/Dapper ──► PostgreSQL 16, database "ILPortal"
```

## Layers inside each project

| Project | Layers (actual folders) |
|---|---|
| ILAPI | `ILAPI.API` (controllers, `Program.cs`, `TokenService`) → `ILAPI.Infrastructure` (`Repositories.cs`, `DependencyInjection.cs`) → `ILAPI.Core` (models, `PremiumCalculator`, `Permissions`). `ILAPI.Tests` references Core. |
| ILWEBFlutter `lib/` | `core/` (api client, config, routing, theme) · `shell/` (header, navigation, menu definition) · `features/` (auth, dashboard, motor, home, travel, liability) · `shared/` (quote form, data list, Angular module host, brand logo) |
| ILWEBAngular `src/app/` | `core/` (config, auth/session service, interceptor + guards) · `shared/` (quote form, quote result, models, forbidden) · `travel/` · `liability/` |
| ILDB | numbered SQL scripts `00`–`08`, `run_all.ps1/.sh`, `load_data.ps1`, `data/` |

## Key design decisions

| # | Decision | Reason / where |
|---|---|---|
| 1 | Flutter is the shell; Angular is content only | One header, one dashboard, one login. See [11](11_Issues_and_Solutions.md) issues 1–3. |
| 2 | Angular is embedded as an **iframe** | Smallest change to both apps; each keeps its own router and build. `ILWEBFlutter/lib/shared/angular_module_view_web.dart` |
| 3 | Session handoff by `postMessage` with a **one-time token** | No tokens in URLs; no second login. [04](04_Authentication_Flow.md) |
| 4 | One API, one database | Neither front end touches PostgreSQL. [07](07_API_Integration.md), [08](08_Database_Architecture.md) |
| 5 | Permissions in the database, issued as JWT claims, enforced by ASP.NET Core policies | Menus only mirror them; the API is the authority. [10](10_Role_Based_Access.md) |
| 6 | Premium rules in C# (`PremiumCalculator`), rates in PostgreSQL | Pure, unit-tested logic; rates changeable by data. [09](09_Quote_Generation_Flow.md) |
| 7 | Clean path URLs in Flutter (`usePathUrlStrategy`) | No `#` in URLs; hosts need an SPA fallback. [05](05_Routing_and_Navigation.md) |

## Implemented vs planned

| Area | Status |
|---|---|
| Flutter shell, routing, role menus, Motor/Home pages | Implemented |
| Angular Travel/Liability modules, guards, handoff | Implemented |
| API auth, policies, quotes, admin read endpoints | Implemented |
| Database schema, seed, demo users, data export | Implemented |
| Docker, docker-compose, CI/CD, remote repositories, environment-specific Angular config | **Planned** (nothing exists in the repos) |
| Customer-details / coverage selection steps, policies, payments, underwriting decisions | **Planned** (see [17](17_Future_Enhancements.md)) |
