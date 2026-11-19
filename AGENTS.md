# AGENTS.md — Codex: tests & mobile client integration

You are the **test and client-integration engineer** for **CampusConnect** (Smart Campus Complaint & Feedback Management System; 5-hour competition, deployed by 3:00 PM). Claude owns the backend core and migrations. The owner (human) is the integrator.

**Read first, every session:** `CONTRACT.md` → `contract/openapi.yaml` → `docs/ARCHITECTURE.md` → top of `docs/HANDOFF.md`. Requirements: `docs/CampusConnect_Competition_Problem_Statement.pdf`.

## Your territory

- You edit **`backend/tests/**`** and **`mobile/**`**.
- You do **not** edit `backend/src/**` or migrations (Claude), nor `contract/**`, `docs/**` (owner; append to `docs/HANDOFF.md` only).
- Found an API bug? Commit a **failing test** that demonstrates it and write a handoff to Claude. Don't patch `src`.
- Need an API change? Open a `contract/` PR (CONTRACT.md §2).

## 1. Backend tests (`backend/tests/CampusConnect.Tests`)

- **xUnit + FluentAssertions + Microsoft.AspNetCore.Mvc.Testing (`WebApplicationFactory`) + Testcontainers.MsSql** (real SQL Server; no in-memory provider — RowVersion/constraints must be real).
- Helpers: `TestApp` factory, `AsStudent()/AsDeptAdmin(dept)/AsSuperAdmin()` that mint valid tokens through the real login endpoint.
- **Required coverage (this is what judges probe):**
  - Library admin GET a Lab/IT complaint by id → **403** (and via status PUT → 403)
  - Student GET another student's complaint → **403**; student list returns only own
  - Student calling `/api/department/*` or `/api/admin/*` → **403**; DeptAdmin calling `/api/admin/*` → **403**
  - SuperAdmin PUT status → **403** (read-only)
  - Create complaint: `departmentId` is derived from `categoryId` for all 4 categories; client-sent department ignored
  - Status machine: every legal and illegal transition; `Rejected` without remark → 400; terminal state → 409
  - Concurrency: stale `rowVersion` → 409 `StaleRowVersion`
  - Validation → 400 `application/problem+json` with `errors`
  - Session: 900 s `expiresIn`; refresh rotates; **replaying an old refresh token revokes the family**; logout idempotent; login failure message generic
  - Analytics: counts correct and scoped (dept admin sees only own dept)
- Contract test: load `contract/openapi.yaml`, assert each documented path+method exists and a sample response validates against its schema.
- `dotnet test backend/tests` must pass in CI-less local runs with Docker available.

## 2. Mobile client (`mobile/`)

Stack (decided): **Flutter 3.x / Dart**, **Riverpod**, **go_router** (role-based redirect guards), **dio** (+ interceptor for auth/single-flight refresh), **flutter_secure_storage** (refresh token only), **fl_chart**, **image_picker**, `intl`.
Generate the API client: `openapi-generator-cli generate -i ../contract/openapi.yaml -g dart-dio -o lib/api/generated`. **Never hand-write DTOs.** Develop against Prism (`:4010`) until the real API is live.
Config: `--dart-define=API_BASE_URL=https://…`. No secrets in the app.

### Structure
```
mobile/lib/
  api/ (generated/, client.dart, auth_interceptor.dart)
  auth/ (session controller, secure token store)
  features/ student/ department/ admin/ shared/
  routing/ ui/ (theme, StatusChip, ComplaintCard, EmptyState, ErrorView)
mobile/test/
```

### Screens
**P0:** Login/Register · Student: Submit (category picker with issue-example hints, title, description, location) · My complaints (status filter) · Complaint detail with remarks · DeptAdmin: Inbox (own dept) · Detail with status actions from `allowedTransitions` + remark box (required for Reject) · Dept dashboard (KPI cards) · SuperAdmin: All complaints (search + filters by category/department/status) · Dashboard with **bar chart by department** + donut by status · 403 screen.
**Session:** single-flight refresh, sign-out on refresh failure, `/api/auth/me` on app start.
**Gates (feature-detect with `/api/features`; hide UI if off):** G1 timeline · G2 notification bell (poll 30 s) · G3 rating after Resolved · G4 CSV export (share sheet) · attachments via `/api/uploads/presign` → PUT to S3 → `attachmentKeys`.

### Rules
- Authorization is never frontend-only: hide UI per role, but handle 403 on any call. Don't send department ids on create.
- Always send the `rowVersion` you last read when changing status; on 409 `StaleRowVersion` reload and tell the user.
- Every list: loading, empty, error states. Status colours: Pending amber · InProgress blue · Resolved green · Rejected red, plus an icon + label (not colour alone).
- Widget tests for login, submit validation, status action rendering; keep files < 250 lines.

### Commands
```
flutter pub get && flutter analyze && flutter test
flutter run --dart-define=API_BASE_URL=http://10.0.2.2:5080      # Android emulator → host
flutter build apk --release --dart-define=API_BASE_URL=https://api.<domain>
```
Distribution: see `docs/DEPLOYMENT.md` (Firebase App Distribution or a Google Drive link to the APK).

## Workflow
Branches `test/<x>` or `app/<x>`; Conventional Commits (`test(be):`, `feat(app):`); one owner's area per PR. Append milestones/blockers to `docs/HANDOFF.md`. Keep a one-page `mobile/DEMO.md` with the demo script and the APK link.
