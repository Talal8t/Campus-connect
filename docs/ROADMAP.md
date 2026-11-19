# Competition Roadmap (10:00 AM → 3:00 PM)

| Time | Claude (backend core) | Codex (tests + mobile) | Owner |
|---|---|---|---|
| 10:00–10:45 | Solution, DbContext, first migration, lookup seed, Identity, register/login/me, `/health` | Flutter scaffold, generate Dart client, Prism mock, login/register screens; test project + Testcontainers harness | Provision RDS + EC2 |
| 10:45–11:45 | Categories, complaints create/list/detail with scope | Student screens; scope/forbidden tests for student | |
| 11:45–12:30 | Department endpoints, status machine, rowVersion, remarks | Dept admin screens; state-machine + concurrency tests | **P0 checkpoint 12:30** |
| 12:30–1:15 | Admin endpoints, dashboards, reports | Dashboards + charts; analytics scope tests | **First deploy to AWS (1:15)** |
| 1:15–2:00 | Session (refresh/logout) → G1 → G2 | Single-flight refresh + session tests → G1/G2 UI | |
| **2:00** | **FEATURE FREEZE** | | |
| 2:00–2:45 | Bug fixes from failing tests; prod admin seed | APK release build, Firebase/Drive upload, demo script | Smoke test 6 logins |
| 2:45–3:00 | | | Submit API URL + APK link |

Cut order if behind: G4 → G3 → G2 → G1 → attachments. Never cut: auth, submit, auto-routing, 403 isolation, status updates, bar chart, deployment. Session is a single bundle: ship it complete or not at all.
