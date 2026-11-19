# Architecture & Decisions

Decisions are made for a **5-hour, deploy-by-3PM** build graded on: working features, server-side RBAC, DB integration, analytics, deployment, and ability to *explain* architecture, APIs and DB design.

## System

```
 Browser (Vite React SPA, Vercel)
        │  HTTPS, Bearer JWT
        ▼
 Express API  /api/v1  (Render)  ──►  PostgreSQL (Neon)
        │                         
        └─ signs uploads ──► Cloudinary  ◄── browser uploads image directly
```

## ADRs (short)

| # | Decision | Why | Rejected |
|---|---|---|---|
| 1 | **Monorepo**, `/frontend` `/backend` `/contract` | One PR history, shared contract, easy for two agents | Two repos (contract drift) |
| 2 | **Contract-first OpenAPI 3.1** | Lets FE and BE work in parallel from minute 0; Prism mock; generated types | Ad-hoc JSON, tRPC (couples agents/languages) |
| 3 | **Vite SPA (React+TS)** not Next.js | No SSR needed behind login; simplest deploy; fewer failure modes | Next.js (more moving parts under time pressure) |
| 4 | **Express + Prisma + Postgres** | Familiar, fast, relational fits complaints/analytics (GROUP BY) | Mongo (weak for aggregates/joins), NestJS (boilerplate) |
| 5 | **JWT in `Authorization` header** | Frontend/back on different domains → avoids cross-site cookie/CSRF pitfalls | httpOnly cookies (SameSite/CORS pain across vercel.app/onrender.com) |
| 6 | **Server-side scope helper** (`lib/scope.ts`) | One choke point = provable RBAC; 403 on direct URL | Per-route ad-hoc checks |
| 7 | **Department derived from category on server** | PDF §4.2 automatic routing; prevents client spoofing | Client sends departmentId |
| 8 | **Signed direct uploads (Cloudinary)** | Render FS is ephemeral; no file handling in API | Multer to disk, base64 in DB |
| 9 | **Admins are seeded**, no admin signup | PDF says not required; fewer attack surface | Admin registration |
| 10 | **Polling for notifications** (30 s) | Zero infra vs WebSockets; good enough | WebSockets/SSE |
| 11 | **Status as a server-enforced state machine** | Integrity + auditability; UI reads `allowedTransitions` | Free-form status |

## Added features beyond the PDF (judged positively)

Ranked by value/effort — build in this order after MVP:

1. **Complaint timeline / audit trail** (who changed what, when) — transparency theme
2. **Ticket number** `CC-2026-0042` — feels real, easy to quote
3. **Priority + SLA overdue flag** — dept admins see what's late
4. **"Me too" support** — surfaces recurring issues, dedupes noise
5. **In-app notifications** — students see responses without refreshing
6. **Resolution rating (1–5)** — closes the feedback loop (it *is* "Complaint & Feedback" Management)
7. **Recurring-issues analytics** — PDF objective: "monitor recurring issues"
8. **Trend line + donut charts** in addition to the required bar chart
9. **CSV export** for admins
10. **Anonymous-to-staff** option — encourages honest reports on staff behaviour (super admin still sees)

Explicitly **not** doing (time): email/SMS, AI classification, WebSockets, multi-language.

## Database (ER summary)

```
Department 1─* User(DEPT_ADMIN)
Department 1─* Complaint
User(STUDENT) 1─* Complaint
Complaint 1─* TimelineEntry
Complaint *─* User via Support
Complaint 1─0..1 Feedback
User 1─* Notification
```

## Seed

See `AGENTS.md` §Seed. Demo credentials in `CONTRACT.md` §6.

## Deployment

| Piece | Host | Notes |
|---|---|---|
| Frontend | Vercel | root `frontend`, env `VITE_API_URL`, SPA rewrite |
| API | Render | root `backend`, health `/api/v1/health`, env per `.env.example` |
| DB | Neon | pooled URL for app; `prisma migrate deploy` on start |
| Images | Cloudinary | unsigned presets disabled; server signs |

Set `CORS_ORIGINS` on Render to the Vercel URL. Free Render sleeps — **hit `/health` 5 min before judging**.

## Explaining it to judges (60-second pitch)

"Contract-first: one OpenAPI spec drives a typed React client and an Express/Prisma API. Complaints route to departments from their category on the server; every query passes through one scope function so a Library admin hitting a Lab URL gets a 403. Status changes follow a state machine and produce an audit timeline plus notifications. Analytics are SQL aggregates scoped by role."
