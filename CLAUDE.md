# CLAUDE.md — Claude: core backend & migrations

You are the **backend core engineer** for **CampusConnect** (Smart Campus Complaint & Feedback Management System; 5-hour competition, deployed by 3:00 PM). Codex writes tests and the mobile client. The owner (human) is the integrator.

**Read first, every session:** `CONTRACT.md` → `contract/openapi.yaml` → `docs/ARCHITECTURE.md` → top of `docs/HANDOFF.md`. Requirements: `docs/CampusConnect_Competition_Problem_Statement.pdf`.

## Your territory

- You edit **`backend/src/**`** and **own all EF Core migrations** (you are the *only* migration author).
- You do **not** edit `backend/tests/**`, `mobile/**`, `contract/**`, `docs/**` (except appending to `docs/HANDOFF.md`). Propose contract/doc changes via PR to the owner.
- Codex will report API defects as failing tests + handoffs. Fix them in `src`.

## Stack (decided)

- **.NET 10 · ASP.NET Core Web API (controllers or minimal APIs, be consistent) · C# nullable enabled, warnings as errors**
- **EF Core 10 + SQL Server** (AWS RDS for SQL Server Express/Standard in prod; Docker `mcr.microsoft.com/mssql/server` locally)
- **ASP.NET Core Identity** (users/roles/password hashing) + **JWT bearer**; own `RefreshToken` table (hashed, rotating, family-based)
- **FluentValidation** for DTO validation → RFC 7807 `ValidationProblemDetails` (400)
- **AWSSDK.S3** for pre-signed attachment URLs · **Serilog** logging · **AspNetCore.RateLimiting** on `/api/auth/*`
- Hosting: **EC2 (Ubuntu) + Nginx reverse proxy → Kestrel**, systemd service; DB on **RDS**. See `docs/DEPLOYMENT.md`.

## Layout

```
backend/
  CampusConnect.sln
  src/CampusConnect.Api/   Program.cs, Controllers|Endpoints/, Auth/, Domain/, Data/(DbContext, Migrations/, Seed/), Services/, Dtos/
  tests/                   (Codex's — do not edit)
```
Keep Domain / Data / Api as folders (or projects) — don't over-engineer for 5 hours.

## Data model (EF Core, SQL Server)

`Departments(int)` · `Categories(int, DepartmentId FK)` · `AspNetUsers(Guid)` with `Name, StudentId?, DepartmentId?` · `Complaints(bigint, Ticket unique, StudentId, CategoryId, DepartmentId, Title, Description, Location?, Status, CreatedAt, UpdatedAt, ResolvedAt?, RowVersion [Timestamp])` · `Remarks(bigint, ComplaintId, AuthorId, Body, CreatedAt)` · `ComplaintAttachments(bigint, ComplaintId, ObjectKey)` · `RefreshTokens(bigint, UserId, FamilyId, TokenHash, ExpiresAt, FamilyExpiresAt, ReplacedByHash?, RevokedAt?, CreatedAt)`.
Optional gates (separate later migrations, never required for P0): `ComplaintHistory`, `Notifications`, `Feedback`.
Indexes: `Complaints(DepartmentId, Status)`, `Complaints(StudentId)`, `Complaints(CategoryId)`, `Complaints(CreatedAt)`.
Seed lookup data (4 departments, 4 categories w/ issue examples) in a migration/`HasData`; **never seed passwords in code.**

## Non-negotiables (graded)

1. **Server-side authorization.** All complaint reads/writes go through one `ComplaintScope` helper:
   Student → `StudentId == userId`; DepartmentAdmin → `DepartmentId == user.DepartmentId` (loaded from DB/claim set by the server); SuperAdmin → all but **read-only**.
   Never take a department from the client. Single-resource out-of-scope → **403**, not 404, no data.
   Use `[Authorize(Roles=…)]` on controllers **and** scope in the service layer.
2. **Routing:** `DepartmentId = Category.DepartmentId` on create.
3. **State machine:** Pending→InProgress|Resolved|Rejected; InProgress→Resolved|Rejected; terminal after. Illegal → 409 `InvalidTransition`. Rejected requires `remark` (400). Compute `allowedTransitions` per caller.
4. **Concurrency:** `rowVersion` required on status update; `DbUpdateConcurrencyException` → 409 `StaleRowVersion`.
5. **Session model** exactly as in `CONTRACT.md §4` — access 900 s, rotating hashed refresh, family revoke on replay, logout idempotent. Ship it all or nothing.
6. Errors are RFC 7807 with the `code` values in the spec. Never leak stack traces/EF exceptions. Login failure message is generic. Rate-limit auth.
7. Gate-dependent writes (history, notifications) are wrapped in the **same transaction** as the status change and only run when the gate is enabled (`/api/features`).

## Commands

```
dotnet restore && dotnet build -warnaserror
dotnet run --project backend/src/CampusConnect.Api        # :5080
dotnet ef migrations add <Name> -p backend/src/CampusConnect.Api
dotnet ef database update -p backend/src/CampusConnect.Api
dotnet format --verify-no-changes
```
Env/config (`appsettings.json` + env vars, no secrets committed): `ConnectionStrings__Default`, `Jwt__Key`, `Jwt__Issuer`, `Jwt__Audience`, `Features__History|Notifications|Feedback|Export|Attachments`, `Aws__Region`, `Aws__S3Bucket`, `SEED_ADMIN_PASSWORD` (dev only).

## Order of work

P0: solution + DbContext + migration + lookup seed → Identity + register/login/me → categories → complaints create/list/detail with scope → department list/detail/status/remarks/dashboard → admin list/detail/dashboard/reports → `/health` → **deploy to AWS** → Session (refresh/logout) → G1 history → G2 notifications → G3 feedback → G4 export, attachments.

## Handoff

After each milestone append to `docs/HANDOFF.md`: base URL, what's live, seeded demo accounts, known gaps. Reply to Codex's BLOCKING items first.
