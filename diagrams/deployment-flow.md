# Deployment Flow

Status: **PLANNED**. The repository has no Dockerfiles, compose file or CI pipeline yet; this diagram shows the recommended target, derived from how the apps are built today (`flutter build web`, `ng build`, `dotnet publish`).

```mermaid
flowchart LR
    Dev[Developer commit] --> CI["CI build (planned)"]
    CI --> F["flutter build web<br/>--dart-define APP_ENV, API_URL, ANGULAR_URL"]
    CI --> N["ng build<br/>(API_URL and PORTAL_ORIGINS must be set per environment)"]
    CI --> P["dotnet publish ILAPI.API"]
    CI --> S["ILDB scripts run_all.ps1/.sh<br/>ILPORTAL_ENV=Production"]

    subgraph Host["Environment (QA / UAT / Production)"]
        direction TB
        Portal["portal.example.com<br/>static host for Flutter build/web<br/>SPA fallback to index.html"]
        Mods["modules.example.com<br/>static host for Angular dist<br/>SPA fallback to index.html"]
        Api["api.example.com<br/>ASP.NET Core behind a reverse proxy, HTTPS"]
        Db[("PostgreSQL")]
    end
    F --> Portal
    N --> Mods
    P --> Api
    S --> Db
    Portal -. "iframe" .-> Mods
    Portal --> Api
    Mods --> Api
    Api --> Db
```

Requirements that follow from the code:

- The API's `Cors:Origins` must list the Flutter origin and the Angular origin.
- Angular's `PORTAL_ORIGINS` (in `ILWEBAngular/src/app/core/config.ts`) must list the Flutter origin, and Flutter's `ANGULAR_URL` must point at the Angular host.
- Both static hosts must return `index.html` for unknown paths (path URLs, no `#`).
- The Angular host must allow being framed by the Flutter origin (no `X-Frame-Options: DENY`; use `Content-Security-Policy: frame-ancestors <flutter origin>`).
