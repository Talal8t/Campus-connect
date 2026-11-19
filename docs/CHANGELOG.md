# Contract Changelog

## 2.0.0 — BREAKING (issue #1, owner decision)
Baseline moved from Node/Express/Prisma/Postgres + web SPA to **.NET 10 / EF Core 10 / SQL Server on AWS + Flutter APK**.
- Base path `/api/v1` → `/api`; role-specific resources `/api/department/*` and `/api/admin/*`
- Roles `STUDENT|DEPT_ADMIN|SUPER_ADMIN` → `Student|DepartmentAdmin|SuperAdmin`; statuses → `Pending|InProgress|Resolved|Rejected`
- IDs: GUID users, int lookups, int64 transactional (were cuid strings)
- Envelope removed; errors now RFC 7807 with `code`; validation 422 → 400; paging `items/pageNumber/pageSize/totalCount`
- Session: 7d JWT → 900 s access token + rotating hashed refresh token (`/auth/refresh`, `/auth/logout`)
- Workflow: Pending may go directly to Resolved; **SuperAdmin is read-only**; no reopen; `rowVersion` concurrency (409)
- Removed from baseline: priority, SLA/overdue, "me too", anonymous-to-staff, `/analytics/*` (replaced by `/api/*/dashboard` and `/api/admin/reports/*`)
- Optional gates G1–G4 added behind `/api/features`
- Uploads: Cloudinary → S3 pre-signed URLs; no committed admin passwords

## 1.0.0
Initial Node-based contract (superseded).
