# CampusConnect

Smart Campus Complaint & Feedback Management System — BIIT Software Engineering Society Application Development Competition.

Students report issues (Cafeteria, Library, Labs & IT, Teaching & Academic); complaints auto-route to the responsible department; department admins respond; the Super Admin monitors everything with dashboards and charts.

## Stack
**ASP.NET Core (.NET 10) · EF Core 10 · SQL Server** on **AWS** (EC2 + Nginx, RDS, S3) · **Flutter** Android app distributed as an APK (Firebase App Distribution or Google Drive).

## Repo layout

| Path | Owner | What |
|---|---|---|
| `backend/src/` | Claude | API + EF Core migrations |
| `backend/tests/` | Codex | xUnit integration + contract tests |
| `mobile/` | Codex | Flutter client |
| `contract/openapi.yaml` | owner | API source of truth |
| `CONTRACT.md` | owner | Working agreement between the agents |
| `CLAUDE.md` / `AGENTS.md` | owner | Instructions for Claude / Codex |
| `docs/` | owner | Requirements PDF, architecture, deployment, roadmap, handoff log |

## Quick start
```bash
# database
docker run -e ACCEPT_EULA=Y -e MSSQL_SA_PASSWORD='<strong-pw>' -p 1433:1433 -d mcr.microsoft.com/mssql/server:2022-latest
# backend
cd backend && dotnet run --project src/CampusConnect.Api      # http://localhost:5080
# mobile
cd mobile && flutter pub get && flutter run --dart-define=API_BASE_URL=http://10.0.2.2:5080
```
No admin passwords are committed; see `CONTRACT.md` §6. Deployment: `docs/DEPLOYMENT.md`.
