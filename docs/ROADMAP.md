# Competition Roadmap (10:00 AM → 3:00 PM)

| Time | Claude (FE) | Codex (BE) | Sync |
|---|---|---|---|
| 10:00–10:30 | Scaffold Vite/Tailwind/shadcn, `gen:api`, Prism mock, auth pages | Scaffold Express/Prisma, schema, migrate, seed, `/health` | Merge scaffolds |
| 10:30–11:30 | Student submit + my complaints + detail | Auth, categories, complaints create/list/get with scope | Switch FE from mock → real for auth |
| 11:30–12:30 | Admin inbox + detail + status/remark UI; route guards; 403/404 | Status machine, comments, timeline, notifications rows | **MVP checkpoint 12:30** |
| 12:30–1:15 | Dashboards + Recharts (bar, donut) | Analytics endpoints + scope tests | **Deploy both (1:15)** |
| 1:15–2:00 | Stretch: support, SLA badge, bell, rating, trend, CSV | Stretch: support, feedback, recurring, CSV, uploads | |
| **2:00** | **FEATURE FREEZE** — bug fixes only | same | |
| 2:00–2:45 | Polish, mobile, empty states, demo script | Prod seed, CORS, env check, smoke tests | |
| 2:45–3:00 | Submit URL | | **Hit /health, test all 6 logins** |

Cut order if behind: CSV → anonymous → trend → recurring → rating → notifications → support. Never cut: auth, submit, routing, RBAC 403, status updates, bar chart.
