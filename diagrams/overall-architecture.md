# Overall Architecture

Verified against the code: Flutter is the shell; Angular is loaded into it as an iframe for Travel and Liability only; one API; one PostgreSQL database.

```mermaid
flowchart TD
    User([User in browser]) --> Flutter["Flutter web shell<br/>ILWEBFlutter (port 8090)"]

    subgraph Shell["Flutter owns"]
        Login[Login]
        Header[Header + top navigation]
        Dash[Dashboard]
        Motor[Motor quote page]
        Home[Home quote page]
        Misc["Quotes, Customers, Users, Roles,<br/>Reports, Underwriting, Profile"]
    end
    Flutter --- Shell

    Flutter -->|"iframe in content area"| Angular["Angular modules<br/>ILWEBAngular (port 4200)"]
    subgraph Mods["Angular owns"]
        Travel[Travel quote + result]
        Liability[Liability quote + result]
    end
    Angular --- Mods

    Flutter -->|"HTTPS JSON + JWT"| API["ILAPI (.NET 9)<br/>port 5080 in dev"]
    Angular -->|"HTTPS JSON + JWT"| API
    API -->|"Npgsql / Dapper"| DB[("PostgreSQL 16<br/>database ILPortal (ILDB scripts)")]
```

| Box | Repository / folder | Role |
|---|---|---|
| Flutter web shell | `ILWEBFlutter` | login, header, navigation, dashboard, Motor, Home, admin list pages |
| Angular modules | `ILWEBAngular` | Travel and Liability only |
| API | `ILAPI` | authentication, authorization, premium calculation, persistence |
| Database | `ILDB` | schema, seed data, demo users, data export scripts |
| Documentation | `ILDoc` | this documentation |

Neither front end connects to PostgreSQL directly.
