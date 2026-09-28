# Hash Out — Go-live plan

**Follows** `OPTIMIZATION_PLAN.md`, whose seven phases are done. **Hosting: a single VPS** — section C has the concrete steps. This plan covers everything that
document lists under *What this plan did not do*, split into **A: what blocks going live** and
**B: what can wait until it is live**.

**Written 2026-09-18.** Effort figures are working days for one person.

## Decisions still open

| Decision | Blocks | Default if you don't decide |
|---|---|---|
| ~~**Where it runs**~~ ✅ **A VPS**, running the existing compose files (decided 2026-09-18) — see section C | — | — |
| **Domain names**, and single-origin vs split (see A1) | A2, A3 | `hashout.example` for the app, `/api/*` proxied to the API on the same host |
| **Email provider** (decided: transactional provider over SMTP) | A3 | — |
| **Where backups go** (object storage, another host) | A6 | Object storage in a different region from the server |
| **Error tracking** (Sentry or nothing) | A7 | Structured logs only, no third party |

---

## A. Blocks going live

### A1. Pick the topology — single origin, if you can (½ day, decide before A2)

Today the browser talks to the API directly and cookies work because the two hosts are *same-site*
(Phase 2). A reverse proxy lets you choose something better:

| | Split hosts (`app.` + `api.`) | **Single origin** (`/api/*` on one host) |
|---|---|---|
| Cookies | Same-site, works today | Same-origin, simpler |
| CORS | Required, must stay correct | Not needed at all |
| Anonymous page load | 2 wasted auth requests (recorded ceiling) | Can be removed — B4 |
| `/settings`, `/notifications` | Must stay client-rendered (recorded ceiling) | Can become server components — B4 |
| Rate limits | Real client IPs already | Correct **only** if the proxy sets `X-Forwarded-For` and uvicorn runs `--forwarded-allow-ips` |

Phase 2 rejected proxying because *Next's* rewrites don't set `X-Forwarded-For`. A real proxy
(Caddy, nginx) does, so that objection does not apply here. **Recommendation: single origin**, with
the proxy routing `/api/*` and `/media/*` to the API and everything else to Next.

**Exit:** the hostnames are written down, and `NEXT_PUBLIC_API_URL`, `CORS_ALLOWED_ORIGINS`,
`CSRF_TRUSTED_ORIGINS` and `ALLOWED_HOSTS` are set consistently with them.

### A2. Reverse proxy and TLS (1 day) — VPS specifics in [C3](#c3-caddy-in-front-a2)

Nothing in the repo terminates TLS. Add a proxy service to the production compose file (or the
platform's equivalent):

- **TLS** on the public hostname(s), HTTP → HTTPS redirect, HTTP/2.
- **Body limit** `6m` — uploads are spooled in full before the API's 5 MB check (`api/routers/upload.py`).
- **`X-Forwarded-Proto` and `X-Forwarded-For`**, plus `TRUSTED_PROXY_IPS` for the API's
  `--forwarded-allow-ips` (already wired in `docker-compose.prod.yml`).
- **WebSocket upgrade** on `/api/notifications/ws` — notifications are dead without it.
- **Long cache** for `/_next/static` and `/static`; **no cache** for HTML.
- Publish only the proxy's ports; drop the `3000`/`8001`/`8000` port mappings from the production overlay.

**Exit:** `curl -I https://<host>/` is 200 over TLS; `http://` redirects; an upload over 6 MB is
refused by the proxy, not by the app; a notification arrives over the WebSocket; `/admin/` is
reachable (and consider restricting it by IP or putting it behind the proxy's basic auth).

### A3. Real email (½ day)

`EMAIL_BACKEND` still defaults to the console, so **signup cannot complete for anyone but you**.

- Pick the provider, verify the sending domain, add **SPF** and **DKIM** records (and a DMARC record
  at `p=none` to start).
- Set `EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend`, `EMAIL_HOST`, `EMAIL_PORT`,
  `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `EMAIL_USE_TLS`, and a `DEFAULT_FROM_EMAIL` on the
  verified domain. No code changes — `api/auth.py` already sends through Django.
- Check the sender reputation basics: a real reply-to, and an unsubscribe link is *not* needed for
  transactional mail.

**Exit:** a signup from a fresh address receives the code within a minute and the account activates;
a password reset does the same; both land in the inbox, not spam (check with a Gmail and an
Outlook address).

### A4. Secrets and configuration (½ day)

- Generate `SECRET_KEY`, `JWT_SECRET_KEY`, `FERNET_KEY` on the host (the app refuses to start with
  `DEBUG=False` on the sample values). **`FERNET_KEY` is not rotatable without re-encrypting
  `cms.SiteSetting` rows** — store it somewhere you can recover it.
- Keep `.env` out of the image (`.dockerignore` already does) and off git; back it up separately.
- Set `DEBUG` unset or `False`, `ALLOWED_HOSTS`, `CORS_ALLOWED_ORIGINS`, `CSRF_TRUSTED_ORIGINS`,
  `TRUSTED_PROXY_IPS`, `API_INTERNAL_URL`, and `OPENAI_API_KEY` if you want the AI features.
- **`NEXT_PUBLIC_API_URL` is baked in at build time** — changing it means rebuilding the frontend image.

**Exit:** `manage.py check --deploy --fail-level WARNING` passes on the host with the real `.env`.

### A5. First deploy and its runbook (½ day)

Write `plans/RUNBOOK.md` (or a README section) with the exact sequence, then follow it once:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.yml -f docker-compose.prod.yml run --rm fastapi python manage.py migrate
docker compose -f docker-compose.yml -f docker-compose.prod.yml run --rm fastapi python manage.py loaddata seed_boards
docker compose -f docker-compose.yml -f docker-compose.prod.yml run --rm fastapi python manage.py createsuperuser
```

- **Upgrading a deployment that predates the Phase 7 image:** `chown -R 10001:10001` the media volume
  once, or the API cannot write uploads.
- Include a rollback line (`git checkout <previous tag> && up -d --build`) and note that migrations
  are not automatically reversible.
- Smoke list after every deploy: sign up, verify, post, upload an image, react, get a notification,
  search, and load `/admin/`.

**Exit:** the runbook was followed start to finish by someone who did not write it, on a clean host.

### A6. Backups (½ day) — VPS specifics in [C7](#c7-backups-on-a-vps-a6)

There is no backup of anything. Two volumes hold state: `postgres_data` and `media_data`.

- **Nightly** `pg_dump -Fc` plus an incremental copy of `media/`, pushed to object storage in another
  region. 30 daily and 6 monthly copies is a sane start.
- Encrypt the archive; it contains password hashes and email addresses.
- **Restore drill:** restore last night's dump into a scratch database and boot the app against it.
  A backup nobody has restored is not a backup.

**Exit:** a restore drill done once, timed, and the timing written in the runbook.

### A7. Logs, errors and uptime (½ day)

`hash_out/settings.py` has no `LOGGING` block, so Django's logs go nowhere useful and the API's
`logger.exception` calls land in container stdout only.

- Add a `LOGGING` config: JSON or key=value to stdout, `WARNING` for libraries, `INFO` for the app.
- Send exceptions somewhere you will look (Sentry's free tier, or at minimum log aggregation).
- Point an uptime monitor at `/api/health` (already exists) and the frontend's `/`.
- Set `CONN_MAX_AGE=60` and `CONN_HEALTH_CHECKS=True`: every request currently opens a new
  Postgres connection, which is the cheapest latency win available.

**Exit:** a deliberate 500 shows up in the error tracker with a stack trace; the uptime monitor
alerts when you stop the API.

**Phase A total: ~3½ days**, plus ~1 day of VPS setup (section C): the app is live, backed up and observable.

---

## B. After it is live

### B1. Search that matches words, not substrings (1 day)

`api/routers/search.py` uses `icontains`, so "post" matches "compost" and ranking does not exist.
Move to Postgres full-text: `SearchVector` on `Topic.subject`/`Post.message`, a `GIN` index, and
`SearchRank` ordering; keep `icontains` as the fallback for queries under three characters.
No new service. **Exit:** searching a common word returns the most relevant topic first, and the
query plan uses the GIN index.

### B2. Rate limits per account, not just per IP (½ day)

`api/limiter.py` keys on the client IP, so one NAT can exhaust another user's budget and a single
account can spread abuse across addresses. Key on the user id when the request is authenticated,
falling back to the IP. **Exit:** two users behind one IP do not share a login budget; one account
hitting the API from many IPs is still limited.

### B3. Notifications socket rechecks auth (½ day)

Auth is checked only at connect (`api/routers/notifications.py`), so a banned user keeps receiving
their notifications until the socket drops. Re-verify every 60 seconds inside the existing
`asyncio.wait` loop, or subscribe to a revocation channel and close on a match.
**Exit:** deactivating a user in `/admin/` closes their socket within a minute.

### B4. Only if you chose single origin in A1 (1 day)

- `/settings` and `/notifications` become server components; the cookie now reaches the Next server.
- Drop the anonymous `/me` + `/refresh` probe on every page load: with one origin, a non-httpOnly
  `logged_in` hint cookie (or the server-side session read) removes both requests.
- Delete the CORS middleware and `CORS_ALLOWED_ORIGINS` once nothing is cross-origin. Keep the
  WebSocket `Origin` check.

**Exit:** an anonymous page load makes zero auth requests; `grep -rn "'use client'" src/app/*/page.tsx`
covers only the auth forms.

### B5. Moderation depth and an audit trail (1–2 days)

Staff can pin, lock and delete topics, and delete posts. There is no record of who did what: Django
logs admin actions, but API moderation writes nothing. Add a small `ModerationLog` (actor, action,
target, reason, timestamp) written by the staff endpoints and readable in `/admin/`. Consider soft
delete (`deleted_at`) so a mistaken delete is recoverable. **Exit:** every staff action appears in
the log with its actor, and a deleted post can be restored.

### B6. Load and capacity check (½ day)

Nothing has ever been measured. Seed a few thousand topics and posts, then run a short load test
against the topic list, topic detail and search. Watch for: query counts (the Phase 5 tests pin the
list endpoints), gunicorn worker saturation, and Redis memory. **Exit:** a documented number of
requests per second at an acceptable p95, and the first bottleneck named.

### B7. Odds and ends (½ day, optional)

- **Rename the database** `boards_db`/`boards_user` → `hash_out_*`: cheap now, annoying later
  (`ALTER DATABASE … RENAME`, `ALTER ROLE … RENAME`, then update `.env`; scram-sha-256 keeps the password).
- **Media to object storage** if you outgrow one host: `STORAGES['default']` swaps to S3; the
  `/media` proxy route follows.
- **Refresh-token grace window** (30 s, Phase 2) — revisit only if token theft becomes a real concern.
- **Dependency updates**: `uv pip compile --upgrade` plus `pip-audit`, monthly. CI already audits.

**Phase B total: ~4–5 days**, none of it blocking.

---

## Order

```
C1-C2 the box ─► A1 topology ─► A2/C3 Caddy + TLS ─┬─► A5 first deploy ─► A6/C7 backups ─► A7/C5 logs and alerts
C4 DNS ─────────────────────────────────────────────┤
A3 email ───────────────────────────────────────────┤
A4 secrets ─────────────────────────────────────────┘
                                   then, in any order: B1 search · B2 per-user limits
                                   B3 socket recheck · B4 (only if single origin)
                                   B5 audit trail · B6 load test · B7 odds and ends
```

**Hard constraints**

1. A1 before A2 — the hostnames decide the proxy config and the frontend build args.
2. A3 before inviting anyone: without SMTP, signup cannot complete.
3. A4 before A5 — the app refuses to start with `DEBUG=False` on sample secrets.
4. A6 before the forum has content worth losing.
5. B4 only if A1 chose single origin; it is wasted work otherwise.
6. B7's database rename is easiest before real data exists.
7. DNS (C4) resolves before the first `up`, or Caddy's first certificate attempt fails and backs off.

---

## C. VPS execution detail

Decided 2026-09-18: **one VPS running the existing compose files**, with Caddy in front. This section
turns A1–A7 into the specific steps for that host. It assumes the **single-origin** topology from A1
(one hostname, `/api/*` proxied to the API); the split-host variant is noted where it differs.

### C1. The box

| | Start with | Why |
|---|---|---|
| **Size** | 2 vCPU, 4 GB RAM, 60–80 GB SSD | Postgres, Redis, two app processes and Caddy fit; the build itself is the memory spike |
| **OS** | Ubuntu LTS or Debian stable | Both have current Docker packages |
| **Region** | Nearest your users | Everything is one box, so latency is the network hop |

**Disk is the thing that bites.** Building both images on the host needs ~3 GB free beyond the
images themselves — that lesson came from building them locally at 97% full. Either keep 20 GB free,
or build elsewhere and pull (C6).

### C2. Base setup (~1 hour, part of A5)

```bash
adduser deploy && usermod -aG sudo deploy          # no day-to-day root
# SSH: keys only — in /etc/ssh/sshd_config set
#   PasswordAuthentication no, PermitRootLogin no
apt install unattended-upgrades fail2ban            # patches and brute-force cover
ufw default deny incoming && ufw allow 22,80,443/tcp && ufw enable
```

- **Docker publishes ports around `ufw`** by writing its own iptables rules. That is safe here only
  because the compose files publish nothing but Caddy's 80/443 — keep it that way (A2 removes the
  `3000`/`8001`/`8000` mappings). Verify with `ss -ltnp` after the first deploy: only 22, 80, 443.
- Install Docker from Docker's own repository (distro packages lag), then `systemctl enable docker`
  so the stack returns after a reboot — the compose services already carry `restart: unless-stopped`.
- Put the checkout in `/srv/hash_out`, owned by `deploy`. `chmod 600 .env`.

### C3. Caddy in front (A2)

Add to `docker-compose.prod.yml` — one service, two volumes, and it gets certificates on its own:

```yaml
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data      # certificates — losing this means re-issuing
      - caddy_config:/config
    depends_on:
      fastapi:
        condition: service_healthy
```

`Caddyfile` (single origin; replace the hostname and email):

```
hashout.example {
    encode zstd gzip
    request_body { max_size 6MB }        # uploads are spooled before the app's 5 MB check

    handle /api/* { reverse_proxy fastapi:8001 }   # WebSocket upgrades pass through as-is
    handle /media/* { reverse_proxy fastapi:8001 }
    handle /admin/* { reverse_proxy django:8000 }  # add `basic_auth` or a remote_ip matcher if you want it private
    handle /static/* { reverse_proxy django:8000 }
    handle { reverse_proxy frontend:3000 }
}
```

- Caddy sets `X-Forwarded-For` and `X-Forwarded-Proto` itself; Django's `SECURE_PROXY_SSL_HEADER`
  and the API's `--forwarded-allow-ips` already expect them.
- **`TRUSTED_PROXY_IPS`**: with no published app ports, Caddy is the only thing that can reach the
  API, so `*` is defensible — but pin it to the compose network's subnet
  (`docker network inspect hash_out_default`) if you want belt and braces.
- **`/admin/` needs the `admin` profile running**: `docker compose … --profile admin up -d django`.
  Skip it and the two admin routes 502.
- Split-host variant: two site blocks (`hashout.example` → frontend, `api.hashout.example` →
  fastapi), and `NEXT_PUBLIC_API_URL` points at the API hostname.

### C4. DNS and the app's own URLs (A1/A4)

- `A` (and `AAAA`) record for the hostname → the VPS IP. Let it propagate **before** the first
  `docker compose up`, or Caddy's first certificate attempt fails and retries with backoff.
- Then, in `.env`: `ALLOWED_HOSTS=hashout.example`, `CSRF_TRUSTED_ORIGINS=https://hashout.example`,
  `NEXT_PUBLIC_API_URL=https://hashout.example` (single origin — the browser and the app share it),
  `API_INTERNAL_URL=http://fastapi:8001`, `TRUSTED_PROXY_IPS=…`.
- `CORS_ALLOWED_ORIGINS` still gates the **WebSocket Origin check**, so set it to
  `https://hashout.example` even when nothing is cross-origin any more.
- Email DNS (A3) lives here too: SPF, DKIM and DMARC on the sending domain.

### C5. Container logs will fill the disk (A7)

Docker's default `json-file` driver has **no size limit**; a chatty API on a 60 GB box is a slow
outage. Add to each service (or `/etc/docker/daemon.json` as a default):

```yaml
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
```

Pair it with a disk alert at 80% — the uptime monitor from A7 can watch a small status endpoint, but
a `df` check in the nightly backup script is enough to start.

### C6. Deploying without a build on the box (optional, ~½ day)

Building on a 4 GB VPS works but competes with Postgres for memory and disk. When that bites:

- Build in CI, push to a registry (GHCR is free for private images), and have the VPS `pull` —
  `docker-compose.prod.yml` swaps `build:` for `image:`.
- Remember the frontend's `NEXT_PUBLIC_API_URL` is a **build arg**, so the registry image is tied to
  the hostname it was built for.
- Keep `docker system prune -af --filter "until=168h"` in the weekly cron either way.

### C7. Backups on a VPS (A6)

```bash
# nightly, as deploy: dump inside the container, then push off the box
docker compose -f docker-compose.yml -f docker-compose.prod.yml exec -T postgres \
  pg_dump -Fc -U "$POSTGRES_USER" "$POSTGRES_DB" > /srv/backups/db-$(date +%F).dump
docker run --rm -v hash_out_media_data:/media -v /srv/backups:/out alpine \
  tar czf /out/media-$(date +%F).tar.gz -C /media .
# then: restic/rclone to object storage in another region, encrypted, 30 daily + 6 monthly
```

- The VPS provider's snapshots are **not** this: they are whole-disk, usually same-region, and
  restoring one loses everything since the snapshot. Keep both if you like, but the off-box dump is
  the one that survives losing the account.
- Also back up `.env` (it holds `FERNET_KEY`, which cannot be rotated without re-encrypting
  `cms.SiteSetting`) and the `caddy_data` volume, or plan to re-issue certificates.
- **Restore drill on the same box:** `pg_restore` into a scratch database, point a throwaway compose
  stack at it, load a page. Time it, write the time in the runbook.

### C8. First-hour checklist (A5, on the real host)

```
ss -ltnp                                  only 22, 80, 443 published
https://hashout.example/                  200, valid certificate, HTTP redirects
/api/health                               {"status":"ok"}
sign up with a real address               code arrives by email, account activates
post, upload an image, react              image renders from /media/, reaction sticks
open a second browser                     notification arrives over the WebSocket
/admin/ (admin profile up)                loads, styled (WhiteNoise), superuser can log in
docker compose ps                         all services healthy after `reboot`
df -h                                     comfortable free space after the build
backup script run by hand                 dump lands in object storage, restore drill passes
```

**VPS-specific extra: ~1 day** on top of phase A's 3½ (the box, firewall, Caddy, log limits, cron).
