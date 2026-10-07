# 01 — Project Overview

The **IL Insurance Portal** is a web portal for quoting four insurance products (Motor, Home, Travel, Public Liability) with role-based access. It is built as one user-facing portal from four repositories.

## Repositories

Each folder under `ILPortal/` is its own git repository (split from an earlier single repository on 2026-10-07; see `12_Implementation_Progress.md`).

| Repository | Technology | Responsibility |
|---|---|---|
| `ILWEBFlutter` | Flutter 3.47 (web), `go_router`, `provider` | **The portal shell.** Login, header, top navigation, dashboard, profile, logout, Motor, Home, and the list pages (Customers, Quotes, Users, Roles, Reports, Underwriting). Hosts the Angular modules. |
| `ILWEBAngular` | Angular 22, vitest | **Feature modules only**: Travel and Liability quote pages. No login, header, dashboard or logout. |
| `ILAPI` | .NET 9, ASP.NET Core, Dapper, Npgsql, BCrypt.Net | The one backend: JWT authentication, roles/permissions, premium calculation, quote persistence. |
| `ILDB` | PostgreSQL 16 SQL scripts | Database `ILPortal`: 10 schemas, tables, indexes, functions, view, seed data, demo users, data export/loader. |
| `ILDoc` | Markdown + Mermaid | This documentation. (Not a git repository.) |

Loose files at the `ILPortal/` root (`README.md`, `.gitignore`, `.env`, `.env.example`, `ILPortal.code-workspace`, `scripts/`) belong to no repository. `.env` holds a local database password and is never committed.

## What the portal does

| Capability | Where |
|---|---|
| Sign in / out, session refresh | Flutter + API (`/api/v1/auth/*`) |
| Role-aware menu and dashboard | Flutter (`lib/shell/menu.dart`, `portal_shell.dart`) |
| Motor and Home quotes | Flutter pages → API |
| Travel and Liability quotes | Angular modules (iframe) → API |
| Quote history | Dashboard card list, `/quotes`, `/my-quotes` |
| Admin views: customers, users, roles, reports, underwriting queue | Flutter list pages → API read endpoints |

## Why two front-end technologies

This is a documented project decision (the portal is meant to show Flutter and Angular working as one application), not a technical necessity. The practical consequences:

| Aspect | Consequence |
|---|---|
| Team skills | Product teams can own a module in the framework they know (Angular for Travel/Liability) while the portal shell stays in Flutter. |
| Independent delivery | An Angular module can be built and tested on its own (`ng test`, `ng build`) and shipped without rebuilding Flutter. |
| Cost | A session handoff, duplicated API-client code, a duplicated visual theme and two toolchains must be maintained. `11_Issues_and_Solutions.md` records the problems this caused. |

Rule that keeps the cost bounded: **the shell is written once (Flutter); Angular only renders product content.**

## Current status

See the status block in [README.md](README.md). In short: all four products produce quotes end to end; authentication, roles and the Flutter/Angular session handoff work; unit tests and a browser end-to-end test pass; deployment automation and a GitHub remote do not exist yet.
