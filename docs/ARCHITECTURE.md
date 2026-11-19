# Architecture & Decisions

Baseline from issue #1 (owner decision): **.NET 10 · EF Core 10 · SQL Server · AWS · mobile app via APK**. Judges grade working features, server-side RBAC, DB integration, analytics, deployment and your ability to explain architecture, APIs and DB design.

## System

```
 Flutter Android app (APK)           AWS
 ┌────────────────────┐   HTTPS   ┌───────────────────────────────┐
 │ Riverpod, dio      │──────────►│ Nginx :443 → Kestrel :5080    │
 │ access JWT (memory)│           │ ASP.NET Core 10 API (EC2)     │──► RDS SQL Server
 │ refresh (Keystore) │           │   Identity + JWT + EF Core    │
 └─────────┬──────────┘           └──────────────┬────────────────┘
           │  PUT image via pre-signed URL        │ signs URLs
           └──────────────────────────────────────▼
                                              S3 bucket (attachments)
 APK shared via Firebase App Distribution or Google Drive
```

## ADRs

| # | Decision | Why |
|---|---|---|
| 1 | Monorepo: `backend/`, `mobile/`, `contract/`, `docs/` | Single history; contract shared by both agents |
| 2 | **Contract-first OpenAPI 3.1** | Codex builds the app on a Prism mock while Claude builds the API; generated Dart client; contract test catches drift |
| 3 | **ASP.NET Core 10 + EF Core 10 + SQL Server** | Owner/Talal baseline; strong typing, Identity, rowversion concurrency built in |
| 4 | **AWS**: EC2 + Nginx (API), RDS SQL Server (DB), S3 (images) | Required hosting; Nginx terminates TLS; RDS keeps DB separate from the app host |
| 5 | **Flutter Android APK** | Apps ship as an installable file for Firebase App Distribution/Drive; one codebase, good charts (`fl_chart`) |
| 6 | **Access JWT 900 s in memory + rotating hashed refresh token** | Short-lived credentials; replay of a rotated token revokes the family. Mobile sends it in body (no cookies/CSRF on native) |
| 7 | **Server-side scope helper** | One choke point = provable RBAC; out-of-scope → 403 |
| 8 | Department **derived from category** server-side | PDF §4.2 automatic routing; no client spoofing |
| 9 | **`rowVersion` optimistic concurrency** on status updates | Two admins can't silently overwrite each other |
| 10 | **SuperAdmin read-only by default** | PDF: super admin views/monitors; only dept admins resolve. Least privilege |
| 11 | **Admins provisioned**, passwords never committed | PDF: no admin registration needed; secrets come from env/user-secrets/CLI |
| 12 | **Pre-signed S3 uploads** | API never handles file bytes; EC2 disk not used for storage |
| 13 | **Polling (30 s)** for notifications; SignalR deferred | Zero extra infra; REST state is persisted so SignalR can be added later without a contract break |
| 14 | RFC 7807 errors, validation 400 | Standard, easy for the Dart client to parse via `code` |

## Delivery gates

| Gate | Scope | Tables |
|---|---|---|
| **P0** | Register/login, categories, submit, auto-route, own-complaint views, dept inbox/detail/status/remarks, super admin all+filters+search, dashboards, bar chart, `/health`, deployed | core tables |
| **Session** | refresh/logout, rotation, replay revocation, single-flight client | `RefreshTokens` |
| **G1** | Timeline/audit (`/history`) | `ComplaintHistory` |
| **G2** | Notifications (polling) | `Notifications` |
| **G3** | 1–5 resolution rating | `Feedback` |
| **G4** | CSV export, recurring-issues extras, attachments | — / S3 |

Optional tables arrive in **later migrations**; P0 never depends on them. When a gate is enabled, its writes join the status-change transaction. `GET /api/features` tells the app what's on.

## Workflow (state machine)

```
Pending ──► InProgress ──► Resolved (terminal)
   │            └────────► Rejected (terminal, remark required)
   ├───────────────────► Resolved
   └───────────────────► Rejected
```
Only the owning department's admin transitions. SuperAdmin reads everything, changes nothing.

## Database (ER summary)

```
Department 1─* Category 1─* Complaint *─1 Student(User)
Department 1─* DepartmentAdmin(User)
Complaint 1─* Remark · 1─* Attachment · 1─* (History|Notification|Feedback gates)
User 1─* RefreshToken (family)
```
Types: user GUID · lookups int · transactional bigint · `Complaints.RowVersion` = SQL `rowversion`. Ticket `CC-<year>-<6-digit>`.

## Features beyond the PDF
Ticket numbers, audit timeline (G1), in-app notifications (G2), resolution rating (G3), CSV export and recurring-issues report (G4), optimistic concurrency, rotating refresh tokens, image attachments. **Not doing:** email/SMS, AI classification, SignalR, iOS build.

## 60-second pitch for judges
"Contract-first: one OpenAPI spec drives both the ASP.NET API and the generated Flutter client. A complaint's department comes from its category on the server; every query passes one scope function, so a Library admin hitting a Lab complaint gets 403 — and tests prove it. Status changes follow a state machine with optimistic concurrency. Sessions use 15-minute JWTs with rotating refresh tokens. It runs on EC2/RDS/S3 and installs from an APK."
