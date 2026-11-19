# CampusConnect backend

ASP.NET Core 10, EF Core 10, SQL Server, and SignalR. This stack follows the human's explicit instruction on 2026-11-19 and supersedes the older Express/Prisma/Postgres guidance in the root Markdown files. `CONTRACT.md` and `contract/openapi.yaml` still define REST behavior. Frontend files are untouched.

## Run locally (PowerShell)

For Visual Studio Development launches, configure `JWT_SECRET` through the API project's **Manage User Secrets** menu. The project has a `UserSecretsId`, so ASP.NET Core loads that local configuration automatically in Development. Use a randomly generated key of at least 32 bytes; keep its value out of source control. Stop and restart debugging after configuring it. Production must supply its own key through deployment configuration.

Install the .NET 10 SDK and SQL Server 2022+ (or Windows SQL Server LocalDB). From `backend/`:

```powershell
dotnet tool restore
dotnet restore CampusConnect.sln --locked-mode
$env:JWT_SECRET = [Convert]::ToBase64String([Security.Cryptography.RandomNumberGenerator]::GetBytes(48))
$env:ConnectionStrings__DefaultConnection = 'Server=(localdb)\MSSQLLocalDB;Database=CampusConnect;Integrated Security=true;TrustServerCertificate=true'
$env:CORS_ORIGINS = 'http://localhost:5173'
dotnet run --project src/CampusConnect.Api -- --seed
dotnet run --project src/CampusConnect.Api --no-launch-profile --urls http://localhost:4000
```

The `.env.example` lists configuration; ASP.NET Core does **not** automatically load `.env`. Export variables in the shell or set them in the hosting environment. With Docker, set `MSSQL_SA_PASSWORD`, run `docker compose up -d`, and use the SQL authentication connection string shown in `.env.example`. Use a trusted SQL Server certificate in production; `TrustServerCertificate=true` is for local development.

`GET http://localhost:4000/api/v1/health` returns `{"status":"ok"}`. `--migrate` applies migrations and exits. `--seed` applies migrations, seeds demo accounts/data, and exits; it is explicit and idempotent. Never seed demo accounts into a production database. On a fresh non-demo database, provision the four departments and administrator accounts through an operational database process before accepting traffic.

## API and authorization

- Base path `/api/v1`; bearer JWT (HS256, seven days); bcrypt passwords.
- Registration always creates a student. Administrators are provisioned separately.
- All complaint reads and mutations use centralized role scope. Out-of-scope direct requests return 403. Department routing derives from category, never client input.
- Includes complaint create/list/detail, status transitions, timeline/comments, feedback, notifications/read-all, summary/department/status/trend/recurring analytics, CSV export, and Cloudinary upload signatures.
- JSON follows the OpenAPI envelope, including the **unwrapped** login/register response and support state. Health is also unwrapped.
- Status updates atomically persist the complaint, timeline, and notification. A concurrency token rejects stale updates. Ticket allocation uses a serializable per-year counter transaction.
- Anonymous students are hidden in both the complaint author and student timeline actor when a department admin reads the complaint.
- CORS uses explicit origins. Auth attempts are rate limited per remote IP. Behind a proxy, configure trusted forwarded headers before relying on per-client IP limits.

### Known contract conflict: support

The existing agreement requires that students can access only their own complaints and forbids supporting one's own complaint. Consequently no student can currently add/remove support. Both routes preserve the 403 boundary. A separate contract change must define a safe complaint discovery/support policy before enabling this feature; the REST spec has not been silently changed. Seeded support counts remain available in complaint views and analytics.

### SignalR

Optional transport at `/hubs/notifications`; existing REST notification polling remains available. Connect with the normal JWT using SignalR's `accessTokenFactory`. Listen for event `notification` with the same notification shape as REST. Group membership derives only from the authenticated user; clients cannot select other users' groups. Notifications are saved before broadcasting, so reconnecting clients can retrieve missed events via REST. This is an additive transport documented here, outside the unchanged REST OpenAPI specification.

```typescript
const connection = new HubConnectionBuilder()
  .withUrl(`${apiOrigin}/hubs/notifications`, { accessTokenFactory: () => token })
  .withAutomaticReconnect().build();
connection.on('notification', notification => refreshNotifications());
await connection.start();
```

Cloudinary signing requires the three `CLOUDINARY_*` variables. Credentials stay on the server; the browser uploads bytes directly. Configure allowed formats and upload size limits in the Cloudinary account before production.

## Verify and CI

### Try every endpoint

Open `CampusConnect.http` in VS Code with REST Client, or Visual Studio's HTTP editor. Seed/start the API first, then execute requests in file order. The named student/department/super-admin login requests supply reusable tokens; `createComplaint` supplies the complaint ID for the complete status/comment/feedback lifecycle. Set `@host` for a different server. Change the registration email when repeating registration, and create a fresh complaint when repeating the lifecycle.

The collection covers every OpenAPI method/path, filtering, CSV, upload signing, SignalR negotiation, and expected 403/422 responses. Upload signing needs the configured Cloudinary credentials; live WebSocket events need a SignalR client. Support examples deliberately return 403 pending a contract decision. The requested issue could not be created because GitHub integration issue-write access was denied; its ready-to-file draft is `issues/support-contract.md`.

```powershell
dotnet build CampusConnect.sln -c Release --no-restore
dotnet test CampusConnect.sln -c Release --logger trx
dotnet ef migrations has-pending-model-changes --project src/CampusConnect.Api
dotnet publish src/CampusConnect.Api -c Release -o publish
```

Tests use an isolated, randomly named SQL Server database, apply real migrations, and delete only that test database when finished. Default: Windows LocalDB. On other platforms set `TEST_DATABASE_URL` to a SQL Server connection string with database creation permissions; the test harness overrides the database name. No production connection string is used implicitly.

Tests cover the four mandated forbidden access cases, registration privilege escalation, automatic routing, status/remark validation, audit history, anonymous timeline privacy, feedback uniqueness, notification isolation and read-all behavior, scoped analytics, CSV filtering/formula escaping, Cloudinary signature generation and role/format validation, SignalR negotiation auth, and OpenAPI 3.1 response validation. Contract checks ensure every specified REST operation is registered and appears in the HTTP collection. Latest local validation: 39 tests passed against SQL Server LocalDB; Release build had zero warnings/errors. Upload-signing tests use fake test-only credentials and do not upload files to Cloudinary.

GitHub Actions `.github/workflows/backend-ci.yml` restores locked dependencies, builds with warnings as errors, checks model/migration drift, tests against a SQL Server 2022 service, and publishes API/test artifacts. Push the feature branch to activate it. The workflow is not a deployment pipeline.

## Deploy

Build the Dockerfile with `backend/` as its context, or publish the API to an ASP.NET Core host. Set the database connection string, a unique JWT secret, and the frontend CORS origin. Apply migrations once as a release step (`dotnet CampusConnect.Api.dll --migrate`) before starting the service. Health check: `/api/v1/health` (liveness, not database readiness). SQL Server needs persistent managed storage; the former Neon/Postgres deployment instructions do not apply. Multiple API instances require a SignalR backplane or managed SignalR service for cross-instance delivery.

Demo password `Campus@123`: `super@campusconnect.edu`, `cafeteria@campusconnect.edu`, `library@campusconnect.edu`, `labit@campusconnect.edu`, `academic@campusconnect.edu`, `student@campusconnect.edu`, `student2@campusconnect.edu`.
