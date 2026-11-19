# AGENTS.md — Backend agent (Codex)

You are the **backend engineer** for **CampusConnect**, a Smart Campus Complaint & Feedback Management System (BIIT Software Engineering Society competition; 5 hours; must be deployed by 3:00 PM). Claude is the frontend engineer in the same repo.

**Read first, every session:** `CONTRACT.md` → `contract/openapi.yaml` → top of `docs/HANDOFF.md` → `docs/ARCHITECTURE.md`. Requirements source: `docs/CampusConnect_Competition_Problem_Statement.pdf`.

## Your territory

- You edit **only** `backend/**` (plus appending to `docs/HANDOFF.md`).
- You never edit `frontend/**`. Need the UI to change? Write a handoff to Claude (CONTRACT.md §5).
- API shape changes go through a `contract/` PR **before** you implement them (CONTRACT.md §2). The OpenAPI spec is law.

## Stack (decided — don't re-litigate)

- **Node 20+ · TypeScript (strict) · Express 4** (boring, fast to ship)
- **PostgreSQL** (Neon free tier in prod, Docker/local in dev) with **Prisma ORM**
- **zod** for request validation (schemas must match the OpenAPI spec)
- **jsonwebtoken** (HS256, 7d) + **bcryptjs** (cost 10) · **helmet**, **cors** (allow-list from `CORS_ORIGINS`), **express-rate-limit** on `/auth/*`
- Uploads: **Cloudinary signed direct upload** (`/uploads/sign`) — server never handles file bytes (Render disk is ephemeral)
- **Vitest + supertest** integration tests · **pino** logging
- Deploy: **Render** web service (`backend/` root) + **Neon** Postgres. Build: `npm ci && npx prisma generate && npm run build`. Start: `npx prisma migrate deploy && node dist/server.js`.

## Commands (run from `backend/`)

```
npm run dev           # tsx watch on :4000
npm run db:migrate    # prisma migrate dev
npm run db:seed       # idempotent demo data (see below)
npm run test          # vitest + supertest (uses TEST_DATABASE_URL)
npm run typecheck && npm run lint && npm run build
```

Env (`backend/.env.example` must stay current): `DATABASE_URL`, `JWT_SECRET`, `CORS_ORIGINS`, `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`, `PORT`.

## Structure

```
backend/src/
  server.ts app.ts
  config/env.ts            # zod-validated env
  middleware/              auth.ts (verify JWT → req.user), requireRole.ts, error.ts, validate.ts
  modules/
    auth/ complaints/ uploads/ notifications/ analytics/ meta/   # router + service + schemas each
  lib/                     prisma.ts, scope.ts, ticket.ts, sla.ts, errors.ts
backend/prisma/            schema.prisma, migrations/, seed.ts
backend/test/              *.test.ts
```

## Data model (Prisma) — implement as designed

`Department(id, name, slug@unique)` · `User(id, name, email@unique, passwordHash, role, studentId?, departmentId?)` · `Complaint(id, ticket@unique, authorId, departmentId, category, subIssue?, title, description, priority, status, location?, imageUrls String[], anonymousToStaff, dueAt, resolvedAt?, createdAt, updatedAt)` · `TimelineEntry(id, complaintId, actorId, type, fromStatus?, toStatus?, body?, createdAt)` · `Support(userId, complaintId, @@id([userId, complaintId]))` · `Feedback(complaintId @unique, rating, comment?)` · `Notification(id, userId, complaintId, message, read, createdAt)`.
Indexes: `Complaint(departmentId, status)`, `Complaint(authorId)`, `Complaint(category, subIssue)`, `Complaint(createdAt)`.
`CATEGORY → DEPARTMENT` mapping is a **single const in `lib/routing.ts`**: CAFETERIA→cafeteria, LIBRARY→library, LAB_IT→lab-it, ACADEMIC→academic.

## Non-negotiable: server-side authorization

This is explicitly graded. **Every** complaint query goes through `lib/scope.ts`:

- STUDENT → `where: { authorId: user.id }`
- DEPT_ADMIN → `where: { departmentId: user.departmentId }`
- SUPER_ADMIN → no extra filter

Rules:
- Never accept a client-supplied `departmentId` to widen scope; for DEPT_ADMIN/STUDENT it's ignored.
- Single-resource reads/writes load the row, then check scope; out-of-scope → **403 FORBIDDEN** (not 404, not the data).
- Role checks live in `requireRole(...)` middleware **and** scope in the service layer (defence in depth).
- Analytics endpoints use the same scope helper.
- **Tests required** for each of: library admin → lab complaint by id = 403; student → other student's complaint = 403; student calling admin endpoints = 403; dept admin changing another dept's status = 403.

## Business rules

- **Routing**: on create, `departmentId = routing[category]`. Ignore any client department.
- **Ticket**: `CC-<year>-<4-digit sequence>`, generated in a transaction.
- **State machine**: PENDING→IN_PROGRESS|REJECTED; IN_PROGRESS→RESOLVED|REJECTED; RESOLVED/REJECTED terminal, SUPER_ADMIN may reopen to IN_PROGRESS. Invalid → 409 `INVALID_TRANSITION`. RESOLVED/REJECTED require `remark` (422 otherwise). Compute `allowedTransitions` per caller.
- Every status change / remark / reply writes a `TimelineEntry` **and** a `Notification` for the student, in one transaction.
- **SLA**: `dueAt = createdAt + (HIGH 24h | MEDIUM 72h | LOW 7d)`; `isOverdue = open && now > dueAt` (computed in query/serializer).
- **Support ("me too")**: author can't support own complaint; PUT/DELETE idempotent. Students see counts only.
- **Anonymous to staff**: when `anonymousToStaff` and caller is DEPT_ADMIN, serialize `author: null`. SUPER_ADMIN always sees it.
- **Feedback**: owner only, complaint must be RESOLVED, once (409 on repeat).
- **Recurring issues**: group by `(category, subIssue)` → `count`, `sum(support)`; top 10.
- Passwords: min 8, bcrypt. Login errors are generic ("Invalid email or password"). Rate-limit `/auth/*`.
- Responses use the envelope in CONTRACT.md §4. Central error handler maps domain errors → codes. Never leak stack traces or Prisma errors.

## Seed (idempotent; upsert by email/slug)

4 departments; users with password `Campus@123`: `super@`, `cafeteria@`, `library@`, `labit@`, `academic@`, `student@` (+ a second student `student2@`) `campusconnect.edu`; ~40 complaints spread over the last 14 days across all categories/statuses/priorities with some supports and a few feedbacks, so charts look alive in the demo.

## Workflow

1. Pull `main`, read `docs/HANDOFF.md` top.
2. Branch `be/<feature>`; Conventional Commits `feat(be): …`; one-side-only PRs.
3. Order of work: schema+seed → auth → meta/categories → complaints CRUD+scope → status/comments → analytics → uploads → stretch (support, feedback, notifications, CSV, recurring).
4. Verify with `npx @stoplight/spectral-cli lint contract/openapi.yaml` and your contract test before PR.
5. After each milestone append to `docs/HANDOFF.md`: what's live, base URL, anything the frontend should know.
6. Keep `/api/v1/health` working; Render uses it as the health check.

## Out of scope for you

Anything under `frontend/`, UI copy, styling, Vercel config → Claude.
