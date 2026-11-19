# Deployment (AWS + APK distribution)

Owner: the integrator (human). Claude supplies the build artifacts/config notes; Codex supplies the APK.

## 1. Database — Amazon RDS for SQL Server
- Engine: SQL Server **Express** (free-tier eligible) or Standard; `db.t3.micro/small`; public access **off**; security group allows 1433 **only from the EC2 security group**.
- Create DB `CampusConnect`. Connection string goes to EC2 env (`ConnectionStrings__Default`), never to git.
- Apply schema: `dotnet ef database update` from a trusted machine through the EC2 host (SSH tunnel), or `dotnet ef migrations bundle` → run bundle on EC2.

## 2. API — EC2 (Ubuntu 22.04+) + Nginx
1. `dotnet publish backend/src/CampusConnect.Api -c Release -o publish` (self-contained linux-x64 optional) → copy to `/opt/campusconnect/`.
2. systemd unit `campusconnect.service`: `ExecStart=/opt/campusconnect/CampusConnect.Api`, `Environment=ASPNETCORE_URLS=http://127.0.0.1:5080`, `ASPNETCORE_ENVIRONMENT=Production`, plus env from `/etc/campusconnect.env` (chmod 600): `ConnectionStrings__Default`, `Jwt__Key` (≥32 random bytes), `Jwt__Issuer`, `Jwt__Audience`, `Features__*`, `Aws__Region`, `Aws__S3Bucket`.
3. Nginx: `server { listen 443 ssl; server_name api.<domain>; location / { proxy_pass http://127.0.0.1:5080; proxy_set_header Host $host; proxy_set_header X-Forwarded-For $remote_addr; proxy_set_header X-Forwarded-Proto $scheme; } }` + Let's Encrypt (`certbot --nginx`). Enable ForwardedHeaders in the app.
4. Security group: 443 open to the world, 22 only your IP. Health check: `GET /health`.
5. **No domain?** Use an Elastic IP + a free DNS name (e.g., DuckDNS) so a real certificate can be issued. Android release builds require HTTPS (cleartext is blocked), so don't skip TLS.
6. Create admin accounts: `dotnet CampusConnect.Api.dll seed-admins` with passwords from env at run time.

## 3. Attachments — S3
Private bucket, block public access, CORS allow `PUT` from `*` (mobile), EC2 instance role with `s3:PutObject`/`GetObject` on the bucket only. API issues short-lived pre-signed URLs.

## 4. Mobile APK
```
cd mobile
flutter build apk --release --dart-define=API_BASE_URL=https://api.<domain>
# output: build/app/outputs/flutter-apk/app-release.apk
```
**Distribute (pick one):**
- **Firebase App Distribution** — create a Firebase project + Android app (matching the `applicationId`), then `firebase appdistribution:distribute app-release.apk --app <APP_ID> --testers "judge@…" --release-notes "…"` (or upload in the console). Testers install via the invite link.
- **Google Drive** — upload the APK, share "Anyone with the link"; testers enable *Install unknown apps*.

Give judges: the APK link, the API base URL, and the demo accounts (shared privately, not in git).

## 5. Pre-demo checklist
- [ ] `GET https://api.<domain>/health` = 200
- [ ] Sign in with each of the 6 demo roles from the installed APK
- [ ] Library admin cannot open a Lab/IT complaint (403 screen)
- [ ] Bar chart shows seeded data
- [ ] RDS/EC2 running (stop nothing; check free-tier/billing alarms)
