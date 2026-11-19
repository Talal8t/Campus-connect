# CONTRACT.md — Claude ⇄ Codex working agreement

Two AI agents build one product in one repo. This file is the **treaty**. If a rule here conflicts with `CLAUDE.md` or `AGENTS.md`, this file wins.

| Agent | Owns | Never edits |
|---|---|---|
| **Claude** (frontend) | `/frontend/**` | `/backend/**` |
| **Codex** (backend) | `/backend/**` | `/frontend/**` |
| **Shared** | `/contract/**`, `/CONTRACT.md`, `/docs/**`, root configs | — (see change rules) |

## 1. Single source of truth

`contract/openapi.yaml` (OpenAPI 3.1) defines every endpoint, schema, enum, error shape and role rule.

- **Backend** implements it exactly and validates requests/responses against it (zod schemas mirror the spec; a contract test fails CI on drift).
- **Frontend** never hand-writes API types. It runs `npm run gen:api` → `frontend/src/api/schema.d.ts` (via `openapi-typescript`) and uses a typed client.
- If code and spec disagree, **the spec is right** until a contract PR says otherwise.

## 2. Change protocol (how to change the API)

1. Open a branch named `contract/<short-name>` that touches **only** `contract/`, `CONTRACT.md` (and `docs/` if needed). No app code.
2. PR title starts with `contract:`. Describe: what changed, why, breaking? (yes/no).
3. The **other** agent (or the human) approves. Merge to `main`.
4. Only then do both sides rebase and implement their half in separate PRs:
   - `feat(be): …` — Codex
   - `feat(fe): …` — Claude
5. Additive changes (new optional field, new endpoint) are non-breaking. Removing/renaming/retyping anything is **breaking** → needs an explicit human OK and a note in `docs/CHANGELOG.md`.

> Fast path for the competition clock: a contract PR may be self-merged if it is purely additive **and** the other agent is notified in the PR description. Never self-merge a breaking change.

## 3. Branching & PRs

- `main` is always deployable. Never push to `main` directly (except the human during the last 30 min).
- Branch prefixes: `fe/…` (Claude), `be/…` (Codex), `contract/…` (either), `docs/…`.
- A PR touches **one side only**. If you need the other side to change, file a handoff (§5), do not edit it.
- Conventional Commits: `feat(fe):`, `fix(be):`, `contract:`, `docs:`, `chore:`.
- Squash-merge. Keep PRs under ~400 changed lines where possible.

## 4. Conventions both sides must follow

| Topic | Rule |
|---|---|
| Base path | `/api/v1` |
| Auth | `Authorization: Bearer <JWT>`; token from `/auth/login` or `/auth/register` |
| Roles | `STUDENT`, `DEPT_ADMIN`, `SUPER_ADMIN` |
| IDs | opaque strings (cuid). Frontend must never parse them |
| Dates | ISO-8601 UTC strings |
| Envelope | Success: `{ "data": … }` (lists add `page,pageSize,total`). Error: `{ "error": { code, message, fields? } }` |
| Error codes | `UNAUTHORIZED` 401 · `FORBIDDEN` 403 · `NOT_FOUND` 404 · `CONFLICT`/`INVALID_TRANSITION` 409 · `VALIDATION_FAILED` 422 · `INTERNAL` 500 |
| Naming | JSON is `camelCase`; enums are `UPPER_SNAKE` |
| CORS | Backend allows only origins in `CORS_ORIGINS` env |

### Authorization is server-side only (non-negotiable — the PDF grades this)

- The backend decides scope for every request: STUDENT → own complaints; DEPT_ADMIN → own department; SUPER_ADMIN → all.
- Out-of-scope direct access always returns **403** (never 404, never data), as the PDF requires.
- The frontend hides things for UX **only**; it must never be the sole guard, and must render the 403 page gracefully.
- Department is **derived from category on the server**. The client never sends `departmentId` on create.

## 5. Handoffs (agent ↔ agent communication)

Agents don't share memory. They communicate via files:

- `docs/HANDOFF.md` — append-only log. Newest entry on top. Format:

  ```
  ### 2026-11-19 10:42 · Claude → Codex · BLOCKING|FYI
  What I need / what changed / how to verify.
  ```
- `BLOCKING` items must be answered before the sender proceeds past the dependency; mock around it in the meantime.
- Each agent reads the top of `docs/HANDOFF.md` and `git log --oneline -20` at the start of every session.

## 6. Parallel-work enablers (so nobody waits)

- **Frontend can start immediately** against a mock: `npm run mock` runs Prism on `contract/openapi.yaml` at `:4010`. Set `VITE_API_URL=http://localhost:4010/api/v1`.
- **Backend seeds** deterministic demo data (`npm run db:seed`): see `docs/ARCHITECTURE.md §Seed`. Frontend may rely on those credentials in demos.
- Shared demo credentials (seeded, password for all: `Campus@123`):

  | Role | Email |
  |---|---|
  | Super Admin | `super@campusconnect.edu` |
  | Cafeteria Admin | `cafeteria@campusconnect.edu` |
  | Library Admin | `library@campusconnect.edu` |
  | Lab/IT Admin | `labit@campusconnect.edu` |
  | Academic Admin | `academic@campusconnect.edu` |
  | Student | `student@campusconnect.edu` |

## 7. Definition of done (per feature)

- [ ] Spec updated first (if API surface changed) and merged
- [ ] Implemented on **one** side, typed, linted, builds
- [ ] Tests: backend → integration test incl. a **forbidden-access** case; frontend → at least a render/happy-path test for key screens
- [ ] Works against the other side's real or mock server
- [ ] Deployed preview not broken (`/api/v1/health` green; FE loads)

## 8. Competition clock

Hard deadline **3:00 PM** deployed. Milestones are in `docs/ROADMAP.md`. At **2:00 PM** feature freeze: only bug fixes, deploy, demo script, README. Anything unfinished is cut, not rushed.

## 9. Disputes

If the two agents disagree on an interface, the human decides. Until then, the spec on `main` stands.
