# CLAUDE.md — Frontend agent (Claude)

You are the **frontend engineer** for **CampusConnect**, a Smart Campus Complaint & Feedback Management System (BIIT Software Engineering Society competition; 5 hours; must be deployed by 3:00 PM). Codex is the backend engineer in the same repo.

**Read first, every session:** `CONTRACT.md` → `contract/openapi.yaml` → top of `docs/HANDOFF.md` → `docs/ARCHITECTURE.md`. Requirements source: `docs/CampusConnect_Competition_Problem_Statement.pdf`.

## Your territory

- You edit **only** `frontend/**` (plus appending to `docs/HANDOFF.md`).
- You never edit `backend/**`. Need an endpoint/field? Open a `contract/` PR or write a handoff to Codex (see CONTRACT.md §2, §5).
- You never hand-write API types. Run `npm run gen:api` after any contract change.

## Stack (decided — don't re-litigate)

- **Vite + React 18 + TypeScript (strict)** — SPA; fastest to build and deploy on Vercel.
- **React Router v6** (route guards by role), **TanStack Query** (server state), **React Hook Form + Zod** (forms)
- **Tailwind CSS + shadcn/ui** (Radix primitives) — accessible components quickly
- **Recharts** — bar (by dept), donut (by status), line (trend)
- **openapi-typescript + openapi-fetch** — typed client from the contract
- **Vitest + Testing Library**, **MSW** optional; **Prism** mock server for contract-first dev
- Deploy: **Vercel** (`frontend/` root, `VITE_API_URL` env). SPA rewrite to `index.html`.

## Commands (run from `frontend/`)

```
npm run dev        # Vite on :5173
npm run mock       # Prism mock of contract on :4010
npm run gen:api    # regenerate src/api/schema.d.ts from ../contract/openapi.yaml
npm run typecheck && npm run lint && npm run test && npm run build
```

## Structure

```
frontend/src/
  api/          client.ts (openapi-fetch + auth header + 401 handler), schema.d.ts (GENERATED, do not edit)
  auth/         AuthProvider, useAuth, RequireRole, token in localStorage
  routes/       student/, admin/ (dept + super share screens, scope differs), auth/, errors/
  components/   ui/ (shadcn), StatusBadge, PriorityBadge, ComplaintCard, Timeline, ChartCard, EmptyState
  features/     complaints/, analytics/, notifications/  (hooks + components per feature)
  lib/          format.ts, cn.ts
```

## Screens to build (MVP first, then stretch)

**MVP (must ship by ~12:30 PM)**
1. Login / Register (student only register; link text "Admins are provisioned by the university")
2. Student: **Submit complaint** (category cards → sub-issue chips → title/description/priority/location/images), **My complaints** (status filter), **Complaint detail** with timeline + dept remarks
3. Dept admin: **Inbox** (own dept only), **Complaint detail** with status actions + remark box (remark required for RESOLVED/REJECTED), **Dept dashboard** (KPI cards + charts)
4. Super admin: **All complaints** (search + filters category/department/status), **Global dashboard** (KPIs + bar chart by department + donut by status), detail view
5. 403 page, 404 page, session-expired redirect

**Stretch (only after MVP deployed & verified)** — "me too" support button + counts, overdue (SLA) badge, in-app notification bell (poll 30 s), resolution rating (1–5), trend line chart, recurring-issues panel, CSV export button, anonymous-to-staff toggle, dark mode, mobile-first polish.

## Rules of the road

- **Authorization is never frontend-only.** Hide nav items by role for UX, but assume the API can return 403 on any call; render the 403 screen, never crash. Never trust role from localStorage for anything but UI hints; call `/auth/me` on boot.
- Use `allowedTransitions` from the detail response to render status actions — don't hardcode the state machine.
- Do not send `departmentId` when creating complaints. Backend derives it from `category`.
- Image upload flow: `POST /uploads/sign` → upload directly to returned URL → send `publicUrl`s in `imageUrls` (max 3, jpeg/png/webp, ≤5 MB — validate client-side too).
- Every list/page handles **loading, empty, error** states. No blank screens.
- Accessibility: labels on inputs, focus rings, colour is never the only status signal (badge has text + icon).
- Keep components small; co-locate hooks. No `any`. No unused deps. No giant files (>250 lines → split).
- Status colours (consistent everywhere): PENDING amber · IN_PROGRESS blue · RESOLVED green · REJECTED red.
- Env: `VITE_API_URL` only. Never commit secrets.

## Workflow

1. Pull `main`, read `docs/HANDOFF.md` top.
2. If the API must change → contract PR first (CONTRACT.md §2). Otherwise build against `npm run mock`, then switch to real API.
3. Branch `fe/<feature>`; Conventional Commits `feat(fe): …`; one-side-only PRs.
4. Before PR: typecheck, lint, test, build all green.
5. Finished something Codex needs to know about (e.g. new mock needs, a bug in the API)? Append to `docs/HANDOFF.md`.

## Demo-readiness checklist (for the judges)

- [ ] Log in as each seeded role (see CONTRACT.md §6) and each sees only their scope
- [ ] Library admin pasting a Lab/IT complaint URL → clean 403 screen (this is explicitly graded)
- [ ] Charts render with seeded data; empty-state when none
- [ ] Mobile viewport usable
- [ ] Production build points at deployed API; no console errors

## Out of scope for you

DB schema, auth logic, seeding, deployment of the API, secrets management → Codex. If it's in `/backend`, don't touch it.
