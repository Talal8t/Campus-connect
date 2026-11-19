# CampusConnect

Smart Campus Complaint & Feedback Management System — BIIT Software Engineering Society Application Development Competition.

Students report issues (Cafeteria, Library, Labs & IT, Teaching & Academic), complaints auto-route to the responsible department, admins manage and respond, and dashboards show what's going wrong on campus.

## Repo layout

| Path | Owner | What |
|---|---|---|
| `frontend/` | Claude | React + TypeScript SPA |
| `backend/` | Codex | Express + Prisma + PostgreSQL API |
| `contract/openapi.yaml` | shared | API source of truth |
| `CLAUDE.md` / `AGENTS.md` | — | Instructions for the frontend / backend agent |
| `CONTRACT.md` | — | Working agreement between the two agents |
| `docs/` | shared | Requirements PDF, architecture, roadmap, handoff log |

## Roles

Student · Departmental Admin (Cafeteria, Library, Lab/IT, Academic) · Super Admin. Authorization is enforced **server-side**.

## Quick start

```bash
# backend
cd backend && cp .env.example .env && npm i && npm run db:migrate && npm run db:seed && npm run dev
# frontend
cd frontend && npm i && npm run gen:api && npm run dev
```

Demo logins and password: see `CONTRACT.md` §6. Architecture & decisions: `docs/ARCHITECTURE.md`.
