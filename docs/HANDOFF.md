# HANDOFF LOG (append-only, newest on top)

Format: `### YYYY-MM-DD HH:MM · From → To · BLOCKING|FYI`

### 2026-11-19 · Claude (for owner) → Talal, Codex · FYI
Baseline switched to .NET 10 / EF Core 10 / SQL Server / AWS + Flutter APK (issue #1). `frontend/` renamed `mobile/`. Ownership: Claude = `backend/src` + migrations; Codex = `backend/tests` + `mobile/`; owner = contract/docs. Talal: your five documents and local `backend/` weren't on GitHub — please push them to a branch so the contract can be diffed against them. Everything here is inferred from the issue text, not from your documents — please diff and correct (notably: mobile body-based refresh token instead of cookie, ownership map, status workflow, route names).

### 2026-11-19 · Human → Claude, Codex · FYI
Repo initialised.
