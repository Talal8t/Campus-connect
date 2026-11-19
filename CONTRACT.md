# CONTRACT.md — Claude ⇄ Codex working agreement

Baseline (decided by the owner, issue #1): **ASP.NET Core (.NET 10) + EF Core 10 + SQL Server on AWS; Flutter Android app distributed as an APK via Firebase App Distribution or a Google Drive link.**

This file is the treaty between the two agents. If `CLAUDE.md` or `AGENTS.md` conflict with it, this file wins.

## 0. Authority chain (highest first)

1. `CONTRACT.md` (this file) — process, ownership, conventions
2. `contract/openapi.yaml` — every endpoint, schema, enum, error shape
3. `docs/ARCHITECTURE.md` — stack, data model, workflow, session model, gates
4. `CLAUDE.md` / `AGENTS.md` — per-agent instructions
5. `docs/CampusConnect_Competition_Problem_Statement.pdf` — original requirements

If code and spec disagree, the spec is right until a contract PR says otherwise.

## 1. Ownership

| Area | Owner | Notes |
|---|---|---|
| `backend/src/**`, EF Core **migrations** | **Claude** | Claude is the **single migration owner**. No one else creates or edits migrations |
| `backend/tests/**`, `mobile/**` (client integration) | **Codex** | Tests treat the API as a black box (WebApplicationFactory + SQL Server container). Codex builds the Flutter client against the contract |
| `contract/**`, `docs/**`, `CONTRACT.md`, `CLAUDE.md`, `AGENTS.md`, root/CI/infra files | **Integrator = the human owner** | Agents *propose* via `contract/` or `docs/` PRs; the owner approves/merges |

An agent never edits another owner's area. Need a change there? Handoff (§5) or a PR the owner/area-owner approves.
Found a backend bug while testing? Codex writes a failing test + a handoff; Claude fixes it in `src`.

## 2. Contract change protocol

1. Branch `contract/<name>` touching only `contract/`, `CONTRACT.md`, `docs/`. PR title starts with `contract:`.
2. State: what changed, why, breaking? (renamed/removed/retyped = breaking).
3. Owner approves and merges. Only then both sides implement in separate PRs: `feat(be):` (Claude) and `feat(app):` / `test(be):` (Codex).
4. Breaking changes need an explicit owner OK and an entry in `docs/CHANGELOG.md`.
5. Additive, optional changes may be merged by the owner without waiting for the other agent, but must be announced in `docs/HANDOFF.md`.

## 3. Branching & PRs

- `main` is always deployable. No direct pushes (owner may in the last 30 minutes).
- Prefixes: `be/…` (Claude), `app/…` and `test/…` (Codex), `contract/…`, `docs/…`.
- One owner's area per PR. Conventional Commits. Squash-merge.

## 4. Wire conventions (both sides)

| Topic | Rule |
|---|---|
| Base path | `/api` (health check at `/health`) |
| Auth | `Authorization: Bearer <access JWT>` |
| Roles | `Student`, `DepartmentAdmin`, `SuperAdmin` (exact strings) |
| Status | `Pending`, `InProgress`, `Resolved`, `Rejected` |
| IDs | user = GUID string · department/category = int32 · complaint/remark/notification = int64 |
| Dates | ISO-8601 UTC |
| JSON | `camelCase` properties, PascalCase enum values |
| Success shape | Plain resource/array. Lists: `{ items, pageNumber, pageSize, totalCount }` |
| Errors | RFC 7807 `application/problem+json` with stable `code`. Validation → **400** with `errors{field:[msgs]}` |
| Codes | 400 validation · 401 unauthenticated/expired · 403 out of scope · 404 missing · 409 conflict / invalid transition / stale rowVersion · 429 rate limit |
| Concurrency | Every complaint carries opaque `rowVersion`; status updates must echo it; stale → 409 `StaleRowVersion` |

### Session model (all-or-nothing — do not ship half)

- Access JWT lifetime **900 s**, held in **memory only** on the client.
- Refresh token: opaque, random, **stored hashed**, **single-use rotation**, 7-day sliding expiry, **30-day family cap**. Reuse of a rotated token **revokes the entire family** (`RefreshReused`).
- Mobile stores the refresh token in the OS secure store (Keystore) and sends it in the request body. (A future web client would use the same-origin HttpOnly cookie variant; not in scope.)
- Client uses **single-flight refresh**: on 401 `TokenExpired`, one refresh runs, parallel requests wait and retry once. Failed refresh → sign out.
- `logout` revokes the family and returns 204 even without a token.

### Authorization is server-side (graded)

- Scope: `Student` → own complaints; `DepartmentAdmin` → own department (from the server-held user record, not a client value); `SuperAdmin` → all, **read-only** (cannot change status).
- Out-of-scope direct access → **403**, never the data. Department is derived from the category on the server.
- The app hides UI by role for UX only and must render 403 gracefully.

## 5. Handoffs

`docs/HANDOFF.md`, append-only, newest on top: `### YYYY-MM-DD HH:MM · From → To · BLOCKING|FYI`. Both agents read the top of it and `git log --oneline -20` at session start. Only the owner edits other docs; agents append to HANDOFF.

## 6. Parallel work enablers

- `npm run mock` equivalent: `npx @stoplight/prism-cli mock contract/openapi.yaml -p 4010` — Codex can build the app before the API exists.
- The Dart client is **generated** from the spec (`openapi-generator-cli generate -g dart-dio`); never hand-write DTOs.
- Backend exposes Swagger UI at `/swagger` in Development; a contract test fails if routes/DTOs drift from `openapi.yaml`.
- **Credentials policy:** no admin password is committed. Development seeding reads passwords from `dotnet user-secrets` / env (`SEED_ADMIN_PASSWORD`); a dev-only seeder runs only when `ASPNETCORE_ENVIRONMENT=Development`. Production admins are created by a one-off CLI command with externally supplied passwords (`dotnet CampusConnect.Api.dll seed-admins`). Demo accounts: `super@`, `cafeteria@`, `library@`, `labit@`, `academic@`, `student@` + `campusconnect.edu`.

## 7. Definition of done

- [ ] Spec updated and merged first (if the API surface changed)
- [ ] Implemented in the owner's area; builds; analyzers/format clean
- [ ] Tests: each endpoint has a happy path **and** a forbidden-scope test; state machine + rowVersion + refresh replay covered
- [ ] Works against the other side's real or mock server
- [ ] `/health` green after deploy

## 8. Competition clock

Hard deadline **3:00 PM**, deployed. **2:00 PM feature freeze.** Gates: P0 → Session → G1 → G2 → G3 → G4. Unfinished gates are cut, never rushed. Detailed schedule: `docs/ROADMAP.md`.

## 9. Disputes

The owner decides. Until then, `main` stands.
