# Hash Out — Optimization & Completion Plan

**Renamed** 2026-09-17 from *Boards* — the Django project package is now `hash_out/`; the `boards` app keeps its name (it holds boards, topics and posts).
**Status:** ✅ **All seven phases done.** Phases 0–6 merged (PR #5–#11); Phase 7 done on `feat/phase-7-production`. The app is deployable: hashed dependency lock, non-root images, gunicorn, WhiteNoise, split compose files and CI. Remaining work is listed under *What this plan did not do*.
**Audit basis:** 8-dimension review of every tracked source file, 199 findings, each re-verified against the code by a second pass (25 corrected, 0 withdrawn). Spot-checked by hand where the stakes were highest.
**Audited:** 2026-09-16 at `e8b0d45` · **Updated:** 2026-09-18 after Phase 7. Line numbers cite the audited commit; files touched in Phase 0 have shifted.

---

## 0. Executive summary

| | |
|---|---|
| Python (project) | 3,528 LOC |
| TypeScript/React | 2,694 LOC |
| FastAPI layer | 1,683 LOC, 25 endpoints |
| Django templates + vendored static | 381 LOC + 432 KB |
| Tests | ~~497 LOC legacy, all of it testing code this plan deletes~~ ✅ deleted in Phase 1 · 21 tests now (7 DB-free, 14 HTTP auth tests against Postgres) |
| Tracked junk | ~~`.env`, `db.sqlite3` (1 MB), 67 `.pyc` files, no root `.gitignore`~~ ✅ resolved in Phase 0 |
| Services declared | 6 (postgres, redis, elasticsearch, django, fastapi, celery, frontend) |
| Services actually needed | 3 (postgres, redis, + the two app processes) |

**Three findings that matter more than the rest:**

1. ✅ *Fixed in Phase 0 (`9079f33`).* **`slowapi` is imported and not declared.** `api/limiter.py:1` and `api/main.py:15-16` import it; `requirements.txt` never lists it. A clean install or `docker compose up` dies with `ModuleNotFoundError` before serving one request.
2. ✅ *Fixed in Phase 0 (`6f37d32`).* **`Topic.tags` does not exist.** `api/routers/topics.py:31` reads `topic.tags` and `:106` writes it; `boards/models.py:30-49` has no such field and no migration adds one. Every topic endpoint — trending on the home page, topic detail, board topic list, topic creation — raises `AttributeError` → 500.
3. ✅ *Fixed in Phase 1.* **The project carries three UI stacks and pays for all three.** Django templates + a rumour of HTMX + Next.js. Only Next.js works. The Django forum views are still routed and one of them (`boards/views.py:14-32`) is an **unauthenticated POST that creates topics attributed to `User.objects.first()`** — usually the superuser.

**The target architecture** — all three independently-written roadmaps converged on it:

> Next.js is the only thing that serves HTML. FastAPI is the only thing that serves JSON. Django is a library (ORM + auth hashers + migrations) plus `/admin/` as the moderation console.

Getting there is mostly deletion. Roughly 2,000 lines and 4 services leave; about 400 lines of genuinely new code arrive (moderation endpoints, mention parsing, a settings page, tests).

### Corrections to the alarming-sounding parts

I checked these myself rather than pass them on:

- **The committed `.env` holds placeholders**, not live secrets: `django-insecure-change-me-in-production-503-92o4`, `change-me-jwt-secret-key-in-production`, `your-openai-api-key-here`, dev Postgres creds. 
- **The committed `db.sqlite3` contains 0 users** (`select count(*) from auth_user` → 0), so no password hashes leaked.
- Therefore this is **not a credential emergency**. It is a real defect for two reasons: the tracked `.env` means the *next* real key gets committed automatically, and `hash_out/settings.py:21` hardcodes a **working Fernet key** as its default — which makes the CMS "encrypted secret" feature decryptable by anyone with the repo. That one is a genuine broken control.
- **The venv is one level up**, at `/home/akhil/code/test/hash_out/venv` (Python 3.12.5). It already had `slowapi 0.1.9` installed, which is why the missing declaration never showed locally. Bare `python` on this machine resolves to an unrelated project's venv — call the interpreter by full path.
- **The migration graph is clean.** `manage.py makemigrations --check --dry-run` → `No changes detected`, against a `boards_db` with 61 tables and all migrations applied. The `pending migrations` commit message (`d9f6ecf`) is stale — ignore it. (`Topic.tags` is consistently absent from model *and* database, which is exactly why the API 500s.)
- ✅ *Resolved in Phase 0 (`658b21b`).* **Every write used to fail on this machine:** `Board.objects.create(...)` raised `AuthenticationException(401)`. The local Elasticsearch 8.19 requires auth, `settings.py` supplies none, and `django-elasticsearch-dsl`'s default `RealTimeSignalProcessor` bulk-indexed inside every save.
- ✅ *Granted 2026-09-17.* **`boards_user` had no `CREATEDB` privilege.** Django's test runner and `pytest-django` both need to create `test_boards_db`, so no database-backed test can run until a Postgres superuser runs `ALTER ROLE boards_user CREATEDB;`. This blocks Phase 1's replacement tests.
- **A placeholder `OPENAI_API_KEY` is exported in the editor's process environment** (not in any shell profile — most likely the editor loaded the old `.env` at startup). Real environment variables beat `.env`, so restart the editor after editing `.env` or the AI endpoints keep sending the stale value.

---

## Phase 0 — Make it boot, and stop the bleeding ✅ Done

**Completed 2026-09-17** on `feat/inital-bug-fix`, 13 commits (`0ec17da` … `7616072`).

| Commit | What landed |
|---|---|
| `0ec17da` | Root `.gitignore`; 67 `__pycache__` files untracked |
| `9079f33` | `slowapi` declared in `requirements.txt` |
| `658b21b` | `ELASTICSEARCH_DSL_AUTOSYNC = False` — writes no longer 401 on the local ES; `.env.example` renamed to `env.sample` |
| `6f37d32` | `Topic.tags` field, migration `0005`, `tags` on `TopicCreate` and `TopicResponse` |
| `d439b4e` | Tags normalized on write (lowercased, trimmed, de-duplicated); `similar_topics` ranks by tag overlap, then same-board recency — replaces `ORDER BY RANDOM()` |
| `3e05a4d` | Search falls back to the ORM when Elasticsearch errors **or** returns 0 hits (with autosync off, the index is stale by design) |
| `6843ebf` | `.env` and `db.sqlite3` untracked |
| `b97e10c` | `api/__init__.py` reduced to a docstring; `api/tests/test_tags.py` sets up Django itself |
| `a4cce12` | `content.py`: one undecorated `_generate()` helper. Suggest-reply and summarize-topic now return 503 with no key, 502 on an upstream error, 404 on an unknown topic — instead of 500 on every call. The catch-all that echoed exception text to clients is gone |
| `ffef11f` | `requirements-dev.txt` (`pytest`, `pytest-django`) and `pytest.ini` |
| `bc03d35` | No `FERNET_KEY` default; startup refuses `DEBUG=False` on a dev or placeholder `SECRET_KEY`, `JWT_SECRET_KEY` or `FERNET_KEY`; `SiteSetting` raises on a missing key or bad token instead of returning ciphertext; decrypted secrets are never cached in Redis; `cms/tests.py` (6 tests) |
| `7001691` | Compose: `DATABASE_URL` replaces the dead `DB_HOST` ×3; Postgres, Redis and Elasticsearch no longer published; obsolete `version:` key removed |
| `7616072` | `frontend/.dockerignore`; `frontend/env.sample`; README env copy commands |

### Where it departed from the original plan

- **The startup check guards three secrets, not one.** `SECRET_KEY` and `JWT_SECRET_KEY` sit alongside `FERNET_KEY` — a production deploy on the default JWT secret lets anyone forge login tokens.
- **The Postgres port was removed, not scoped to `127.0.0.1`.** A host Postgres already on `127.0.0.1:5432` makes the scoped publish collide. Reach the compose database with `docker compose exec postgres psql`.
- **No `EMAIL_*` keys in `env.sample`.** The original item was wrong: `settings.py` doesn't read them. They land in Phase 4 with the code that does.
- **The AdSense keys went to `frontend/env.sample`**, not the root file — Next.js only reads `frontend/.env*`.
- **`email-validator` moved to Phase 2**, where `EmailStr` first needs it.
- **Bare `pytest` runs `api/` and `cms/` only** (`testpaths` in `pytest.ini`). The `accounts/` and `boards/` suites need `CREATEDB` and only cover the template views Phase 1 deletes.

### Exit criteria

```
git ls-files | grep -E '^\.env$|sqlite|__pycache__'   ✅ empty
git grep 'RVkCS56'                                    ✅ empty
python -c "import api.main"                           ✅ 44 routes
curl :8001/api/topics/trending                        ✅ 200
pytest                                                ✅ 7 passed
manage.py makemigrations --check --dry-run            ✅ No changes detected
docker compose config                                 ✅ valid · 0 infra ports published
docker compose up --build                             ⏸ not run — pulls and builds images; run before Phase 7
```

**History note:** the placeholder `.env` blobs stay reachable at `8cdc5a1`/`d299596` on the public remote. The values were placeholders, so `git filter-repo` is optional; any real deployment needs freshly generated keys regardless, and the startup check now enforces that.

**Before pulling on another clone:** `6843ebf` removes `.env` from the index, so pulling it **deletes the local `.env`**. Back it up first.

---

## Phase 1 — One UI stack: delete the Django SSR forum, HTMX and Wagtail ✅ Done

**Completed 2026-09-17** on `feat/phase-1-one-ui-stack`. 1,564 lines deleted, 265 added, across 72 files.

| Commit | What landed |
|---|---|
| `feat(api): enforce password validators, add change-password endpoint and auth tests` | `check_password_strength()` runs `AUTH_PASSWORD_VALIDATORS` on signup and reset-password (replaces the ad-hoc `len < 8`), 400 with every failed rule joined into `detail`. `POST /api/auth/change-password` (old + new, `get_current_user`, `5/minute`). Unused passlib `CryptContext` removed. `api/tests/test_auth.py`: 14 tests over `TestClient` — signup, duplicate username/email, 4 weak passwords, token pair shape, wrong password, refresh rejects an access token, `/me` rejects a refresh token and an unknown user, identical forgot-password message, reset and change-password run the validators |
| `refactor(api): remove HTMX fragment endpoints` | `POST /api/topics/board/{id}/htmx` and `POST /api/posts/topic/{id}/htmx` gone with their `HTMLResponse` imports. 43 routes |
| `refactor: delete Django template UI and Wagtail, make /admin/ the moderation console` | `templates/`, `static/`, board/account views, forms, urls, template tags and their test suites, `hash_out/asgi.py`, `cms/wagtail_hooks.py`. Django routes only `/admin/`. Settings lose Wagtail (13 apps, middleware, config), `corsheaders`, `widget_tweaks`, `STATICFILES_DIRS`, `LOGIN_*`. `cms` keeps only `SiteSetting`, registered in `/admin/` with the secret-masking form; `cms/migrations/0001` rewritten without the page models or the `wagtailcore` dependency. `/admin/` registers `Topic` (pin/lock editable in the list), `Post`, `Reaction`, `Notification`. Requirements drop `wagtail`, `django-cors-headers`, `django-widget-tweaks`, `passlib`; `httpx` → `requirements-dev.txt` |
| `chore(frontend): drop htmx.org, starter svgs, generated service worker and missing icons` | `npm uninstall htmx.org`; 5 starter SVGs deleted; `sw.js`/`workbox-*.js` untracked and gitignored; the `icons` array (two 404s) removed from `manifest.json`; HTMX comments fixed |

### Where it departed from the original plan

- **Wagtail went entirely**, not just the page models (decided 2026-09-17). Nothing outside the Wagtail dashboard read `SiteSetting`, the table had 0 rows, and the 4 pages rendered nowhere. Dropping it also removed the `mark_safe` XSS in `wagtail_hooks.py` (Phase 3) and ~20 transitive packages.
- **`cms/migrations/0001_initial.py` was rewritten, not followed by a `DeleteModel` migration.** A `DeleteModel` still needs `wagtailcore` installed to load `0001`. Safe because no populated deployment exists. The recorded migration name is unchanged, so existing databases need no `--fake`.
- **`Notification` is registered in `notifications/admin.py`**, which was filled in rather than deleted. `accounts/admin.py` was deleted as planned; `UserProfile` is not registered.
- **`TEMPLATES['DIRS']` emptied and `LOGIN_URL`/`LOGIN_REDIRECT_URL`/`LOGOUT_REDIRECT_URL` removed** — they named templates and URL names that no longer exist. `TEMPLATES` itself stays for the admin.
- **The auth tests use `django_db(transaction=True)`.** `TestClient` runs sync endpoints in worker threads with their own connections, which can't see rows inside a test-wrapping transaction. They also swap in `LocMemCache` and disable the rate limiter.
- **Change-password does not revoke existing JWTs** — tokens stay valid until they expire. Filed under Phase 2's `jti` work.

### Local environment changes (not in git)

- **Database:** 44 tables dropped (`wagtail*`, `taggit*`, the 4 `cms_*page` tables), 198 `django_migrations` rows and the stale content types removed. Backed up first with `pg_dump -Fc` (session scratchpad, `wagtail-tables-backup.dump`). **Other clones:** run the same cleanup, or recreate the database and `migrate`.
- **venv:** `wagtail`, `django-taggit`, `django-modelcluster`, `django-cors-headers`, `django-widget-tweaks`, `passlib` and their now-orphaned dependencies uninstalled; `pip install --dry-run -r requirements-dev.txt` installs nothing and `pip check` is clean.
- **`.env`:** `WAGTAIL_SITE_NAME` is now unused; delete it.

### Exit criteria

```
grep -rn 'render(' --include='*.py' .                  ✅ empty (wagtail_hooks.py deleted too)
git ls-files | grep -E '^templates/|^static/'           ✅ empty
git grep -i 'wagtail|htmx|corsheaders|passlib'          ✅ empty outside plans/
pytest                                                  ✅ 21 passed
manage.py check · makemigrations --check · migrate --check   ✅ clean
python -c "import api.main"                             ✅ 43 routes
/admin/ as superuser: index, Topic, Post, Reaction, Notification, SiteSetting   ✅ 200
SiteSetting secret via /admin/: encrypted at rest, masked in form, untouched mask keeps value   ✅
/ · /login/ · /signup/ · /cms-admin/ · /pages/          ✅ 404
API smoke: health, boards, trending, topic, board topics, search   ✅ 200
tsc --noEmit · manifest.json · docker compose config    ✅
```

---

## Phase 2 — One auth system, and a ban that actually bans ✅ Done

**Completed 2026-09-17** on `feat/phase-2-auth`. 508 lines added, 405 deleted, across 19 files.

| Commit | What landed |
|---|---|
| `feat(api): back rate limits with redis` | `Limiter(storage_uri=settings.REDIS_URL)` — limits survive restarts and apply across workers |
| `feat(auth): httpOnly cookie sessions with rotation, revocation and is_active checks` | **Cookies:** `/token` sets `access_token` (`Path=/api`) and `refresh_token` (`Path=/api/auth`), both `HttpOnly`, `SameSite=Lax`, `Secure` unless `DEBUG`; no token ever appears in a response body. New `POST /api/auth/logout`. **Identity:** tokens resolve by `user_id` with `is_active=True`; login uses `authenticate()` (rejects inactive users, constant-time for unknown usernames). **Revocation:** a `pwd` claim (first 16 chars of `get_session_auth_hash()`) kills every token on password change or reset; `fam`/`jti` claims + Redis give single-use refresh tokens, and a replay after a 30 s grace window revokes the family. **OTP:** `secrets`, `{otp, user_id}` in Redis, atomic `cache.incr` attempt counter, `hmac.compare_digest`, burned after 5 wrong guesses, email sent via `BackgroundTasks` calling the task function directly. **Signup:** `EmailStr` (`email-validator`), lowercased, case-insensitive uniqueness. **Profiles:** `PublicProfileResponse` (no `email`) on `GET /{username}` and search; search needs `q` ≥ 2 chars, `30/minute`. **WebSocket:** cookie auth and an `Origin` allowlist, both checked before `accept()`. **Frontend:** `lib/api.ts` exports `API_BASE`, `postForm`, `postMultipart`, sends `credentials: 'include'`, and de-duplicates refresh; all nine raw `fetch`/`getTokens` call sites moved onto it; `WebSocketProvider` connects only when logged in and reconnects on login/logout. 36 tests |

### Where it departed from the original plan

- **No Next.js route handlers or same-origin proxy.** FastAPI sets the cookies itself and the browser calls it directly with `credentials: 'include'`. Tested first: a Next rewrite proxies HTTP and even WebSocket upgrades fine, but it **does not add `X-Forwarded-For`** — it only passes through whatever the client sent. Behind it, FastAPI sees one IP for every user (every rate limit becomes global) or trusts a spoofable header. Calling the API directly keeps real client IPs and needs no proxy code. **Constraint this creates:** the frontend and API must be **same-site** (`localhost:3000` → `localhost:8001`, or `app.example.com` → `api.example.com`), or `SameSite=Lax` cookies are never sent. Documented in `env.sample` and `lib/api.ts`.
- **Password-change revocation uses a fingerprint claim, not a per-user generation counter in Redis.** Stateless, and it covers reset-password too. Change-password re-issues cookies for the current session.
- **Refresh reuse has a 30-second grace window.** Without it, two tabs refreshing together would read as token theft and log the user out everywhere. The client also de-duplicates refreshes within a tab.
- **The OTP email is sent from a `BackgroundTasks` job**, not synchronously, so a known email isn't measurably slower than an unknown one (the plan's synchronous call would have been a timing oracle for account enumeration). Celery's config is untouched until Phase 4.
- **`constr(min_length=8)` skipped:** `MinimumLengthValidator` already enforces it through `check_password_strength`.
- **WebSocket `Origin` check added** (not in the plan) — cookie-authenticated sockets are otherwise open to cross-site WebSocket hijacking from same-site origins. `CORS_ALLOWED_ORIGINS` moved back into `settings.py` as the one source for CORS and this check.
- **`get_optional_user` deleted now** (planned for Phase 6) — it depended on the removed `OAuth2PasswordBearer`.
- **`/verify-otp` counts as a guess** toward the 5-attempt limit.
- **Also done early from Phase 3:** `PostCard` reactions gated on `useAuth().user` (rollback still open); `WebSocketProvider` depends on `user` and skips anonymous visitors (backoff reconnect still open); `notifications.py` no longer has the `channel_name` `NameError` (disconnect detection and `print` → `logging` still open).

### Known ceilings

- **Anonymous visitors cost two requests per page load** (`/me` 401, then `/refresh` 401) — JS can't see whether a refresh cookie exists. A non-httpOnly `logged_in` hint cookie would skip the second one.
- **Auth is checked when the WebSocket connects**, not during the connection. A user deactivated mid-connection keeps receiving their own notifications until it reconnects.
- **Swagger's "Authorize" button no longer applies.** Use "Try it out" on `POST /api/auth/token` at `:8001/docs`; the cookie then rides along on later calls.
- **Rate limits key on the socket peer.** Behind a reverse proxy (Phase 7), run uvicorn with `--forwarded-allow-ips` set to the proxy.

### Exit criteria

```
is_active=False → 401 on /me, /token, /refresh; WebSocket rejected     ✅ tests + real browser
reused refresh token rejected, family revoked (and grace-window race not)  ✅ tests
5 wrong OTPs burn the code                                              ✅ tests
grep -rn 'localStorage\|Bearer ' frontend/src                           ✅ empty
anonymous GET /api/profiles/<username> has no email                     ✅ tests
pytest                                                                  ✅ 36 passed
mutation check: removing is_active, family revocation, Origin check, public schema,
  OTP attempt limit or pwd fingerprint each fails a test                ✅ 6/6 caught
headless Chrome against next dev + uvicorn: login sets httpOnly cookies, nothing in
  localStorage/document.cookie, socket opens without ?token= and receives a notification,
  reload keeps the session, deleted access cookie → silent refresh with rotation,
  UI logout clears cookies, deactivated user shown logged out and refused login   ✅
manage.py check · makemigrations --check · tsc --noEmit · docker compose config · pip check   ✅
```

---

## Phase 3 — Fix what a user hits in the first ten minutes ✅ Done

**Completed 2026-09-17** on `feat/phase-3-bug-sweep`.

| Commit | What landed |
|---|---|
| `fix(api): validate uploads by image content and harden /media` | `read_image()` in `upload.py`, shared by post images and avatars: size cap, `Image.open().verify()`, extension from **Pillow's detected format** — client filename and Content-Type are ignored. Avatars get a UUID name (no client filename, no stale browser cache). `/media` is a `StaticFiles` subclass adding `X-Content-Type-Options: nosniff` and `Content-Security-Policy: default-src 'none'; sandbox`, and serving anything that isn't JPEG/PNG/GIF/WEBP as `application/octet-stream`. Profile `website` must be `http(s)://`. Shared API test fixtures in `api/tests/conftest.py` |
| `fix(boards): bump topics on reply, atomic counters, safe slug migration` | `post_save` receiver on new `Post` → `Topic…update(last_updated=now)`. `views_count` and `reputation_score` via `F()` (`Greatest(…, 0)` on removal). No reputation for reacting to your own post; `emoji: Literal["👍", "❤️"]`. Migration `0003` adds `Board.slug` nullable and unindexed → `RunPython` backfill with `-<pk>` on collision → `AlterField(unique=True)`. `boards/tests.py` joins `testpaths` |
| `fix(api): release notification subscriptions when the socket closes` | `asyncio.wait` over the Redis forwarder and a `receive()` loop — whichever ends first cancels the other, then unsubscribe and close. Failures logged, not printed. Redis URL from `settings.REDIS_URL` (router and `notifications/models.py`) |
| `fix(frontend): pagination, quotes, reactions, socket reconnect, media rewrite, a11y` | Board page effect depends on `page` (pagination was dead); board and topic pages ignore stale responses. Quote reply splits on a real newline. Failed reactions roll back. `WebSocketProvider` reconnects with 1 s → 30 s backoff and stops on logout; the unreachable desktop-notification branch is deleted. `/media/:path*` rewrite to the API (`API_INTERNAL_URL`, set to `http://fastapi:8001` in compose) — uploaded images and avatars load from the app origin. Navbar: `/settings` link removed, `aria-expanded`/`aria-haspopup`, Escape and outside click close every menu. `id`/`htmlFor` on the new-topic and profile forms, `aria-label` on the reply editor |

### Where it departed from the original plan

- **Uploads trust Pillow, not the client.** The plan mapped the (client-sent) Content-Type to an extension and added `verify()`; the format Pillow detects is the one fact the client can't spoof, so that alone picks the extension.
- **`/media` also gets a sandboxing CSP and an octet-stream fallback**, so a non-image that already sits in `media/` (from before this phase) downloads instead of rendering.
- **The edge body-size limit is not done** — there is no reverse proxy in the repo yet. Filed under Phase 7.
- **Migration `0003`'s interim column is also `db_index=False`.** Otherwise a fresh database creates the `_like` index twice and `migrate` fails. Existing databases keep the unique constraint name `boards_board_slug_key`; fresh ones get `boards_board_slug_f5548cf1_uniq`. Django finds constraints by introspection, so later migrations work on both.
- **Only a new post bumps its topic**, not an edit.
- **The emoji whitelist is exactly what `PostCard` offers** (👍 ❤️); the database had no other reactions. Widen both together.
- **The board page doesn't `setLoading(true)` on page change** (React's `set-state-in-effect` lint rule forbids it); it keeps showing the previous page until the next one arrives, and a stale-response guard stops out-of-order responses.
- **Desktop notifications were deleted**, not wired to `requestPermission()`.
- **Added:** profile `website` must be an `http(s)` URL. It renders as a link, so `javascript:` there was stored XSS (React 19 happens to block it at render; now the API refuses it).
- **The socket disconnect test runs a real uvicorn server.** Starlette's `TestClient` cancels the handler when the test closes the socket, which hid the bug (checked: the old loop passes under `TestClient`).
- **The manual pass is an automated headless-Chrome run** (below), not a checklist.
- **Environment note:** a stale `frontend/.next` made Turbopack fail with `Can't resolve 'tailwindcss' in '…/hash_out/hash_out'` and hang every page. `rm -rf frontend/.next` fixes it.

### Exit criteria

```
pytest                                                              ✅ 55 passed
mutation check — each fix removed in turn fails a test:
  Pillow format check, nosniff, octet-stream fallback, self-reaction guard,
  reputation floor, F() view counter (40 concurrent GETs), reply bump,
  website validation, migration 0003 backfill, socket disconnect      ✅ 10/10 caught
fresh-DB migrate vs existing DB: boards_board indexes                ✅ same (constraint name differs, see above)
headless Chrome against next dev + uvicorn, seeded 25-topic board:
  page 2 shows 5 different topics                                    ✅
  labels find Subject/Tags/Message                                   ✅
  menus: aria-expanded, Escape, outside click; no Settings link      ✅
  quote keeps "> line one\n> line two"                               ✅
  evil.html with image/png refused; real PNG renders from :3000/media, nosniff   ✅
  reply moves the oldest topic to the top of page 1                  ✅
  reaction: forced 500 rolls back, real one persists, author +1 reputation       ✅
  avatar renders from /media/avatars/<uuid>.png                      ✅
  API killed and restarted: socket reconnects and receives a notification        ✅
  no uncaught page errors                                            ✅
manage.py check · makemigrations --check · tsc --noEmit · docker compose config · pip check   ✅
```

---

## Phase 4 — Six services → three ✅ Done

**Completed 2026-09-18** on `feat/phase-4-three-services`. 457 lines deleted, 161 added, across 20 files; 3 files removed; 2 services, 3 Python packages and 1 volume gone.

| Commit | What landed |
|---|---|
| `refactor: delete elasticsearch and celery, search runs on the ORM` | **Search:** `boards/documents.py` deleted and `search.py` rewritten — `_search_elasticsearch` and the fallback dance gone; one result list paginated **across** types (each type used to skip the same offset, so page 2 repeated rows); `total` and a new per-type `counts` map computed from the same filtered sets; `type` is a `Literal`; `Q(name|description)` instead of a `|` queryset union; post hits link to `/topics/<topic_id>`, not the post id. `api/tests/test_search.py` |
| *(same commit)* | **ES:** the `ELASTICSEARCH_*` settings, the `django_elasticsearch_dsl` app, both requirements, the compose service, its `es_data` volume, the three `ELASTICSEARCH_URL` lines and the `env.sample` keys. **Celery:** `hash_out/celery.py`, `api/tasks.py` (5 of its 6 tasks had no caller), the `CELERY_*` settings, `celery[redis]`, the worker service, the `hash_out/__init__.py` import. `send_otp_email()` is now a plain function in `api/auth.py`. `EMAIL_HOST`/`PORT`/`USER`/`PASSWORD`/`USE_TLS` and `DEFAULT_FROM_EMAIL` settings added (the OTP path always assumed them) and documented in `env.sample` |
| `refactor(api): delete the dead cache decorator` | `api/deps.py` keeps only `paginate()`; the never-applied `cached` decorator, `cache_key`, `invalidate_cache` and all ten call sites are gone. `CACHES` falls back to `LocMemCache` when `REDIS_URL` is empty (the rate limiter already falls back to in-memory), so the suite runs without Redis |
| `docs: describe the three services in the readme` | README: no Elasticsearch/Celery rows or steps, Redis described as required (sessions, reset codes, rate limits, pub/sub), `docker compose up` documented, structure tree updated |

### Where it departed from the original plan

- **`api/tasks.py` was deleted outright**, not trimmed: with Celery gone its last live function is `send_otp_email`, which now sits in `api/auth.py` beside its only caller.
- **The Django service stays in compose but behind a `profiles: ["admin"]` flag**, so a plain `docker compose up` starts exactly postgres, redis, fastapi and frontend; `docker compose --profile admin up django` when you need `/admin/`.
- **Search gained a `counts` field** (per-type totals) rather than only fixing `total` — Phase 6's type tabs need it, and it comes from the same queries.
- **Redis stays required in practice.** The `LocMemCache` fallback is per-process: fine for tests or a bare checkout, useless for sessions across workers. The notification WebSocket still needs a real Redis.
- **Postgres full-text search was not added.** `icontains` is what the ORM path already did; revisit when search quality matters (a `SearchVector` + GIN index costs no new service).

### Exit criteria

```
docker compose config --services                      ✅ postgres, redis, fastapi, frontend (django behind --profile admin)
docker compose up -d postgres redis                   ✅ both healthy, then torn down with down -v
docker compose up --build (full stack)                ⏸ not run — disk is at 97%; Phase 7 rewrites both Dockerfiles
git grep -i 'elasticsearch|celery' outside plans/     ✅ empty
pytest                                                ✅ 59 passed
mutation check: post→topic link, total from counts, cross-type paging   ✅ 3/3 caught
REDIS_URL empty → LocMemCache + in-memory rate limits, api.main imports  ✅
venv after uninstalling both packages: pip check, requirements-dev satisfied   ✅
manage.py check · makemigrations --check · tsc --noEmit · docker compose config   ✅
```

---

## Phase 5 — One rendering model, one API contract ✅ Done

**Completed 2026-09-18** on `feat/phase-5-server-components`.

| Commit | What landed |
|---|---|
| `perf(api): flat list queries, indexes and complete list schemas` | **N+1s gone:** boards annotate `topics_total`/`posts_total`/`last_post_at` (one query for any number of boards), topics annotate `posts_total`, posts `select_related('created_by__profile','updated_by')` + `prefetch_related('reactions')` + an annotated `author_post_count` that `get_user_badges` now takes as an argument. The four `Board`/`Topic` helper methods behind the old per-row queries are deleted. **Indexes:** `Topic(board, -is_pinned, -last_updated)`, `Topic(-views_count, -last_updated)`, `Post(topic, created_at)`, `Notification(recipient, is_read, -created_at)` (migrations `boards.0006`, `notifications.0002`). **Schemas:** `UserBrief.badges`, `PostResponse.reactions`, and `BoardListResponse`/`TopicListResponse`/`PostListResponse` so every list endpoint has a `response_model`. **Also:** the `save_user_profile` receiver is gone (it rewrote the profile on every `User.save()`), and `notifications/models.py` publishes through one module-level Redis client. `api/tests/test_queries.py` |
| `refactor(frontend): server-render the board, topic, profile and search pages` | Four route pages are now async server components reading `params`/`searchParams` and fetching through `lib/server-api.ts` (which uses `API_INTERNAL_URL` inside Docker). Client islands hold what needs state: `<NewTopicForm>`, `<Conversation>` (posts + reply box, because Quote Reply writes into the box) and `<ProfileCard>`; each calls `router.refresh()` after a write. `loading.tsx` + `error.tsx` per segment. `src/types/api.d.ts` is generated from the OpenAPI schema with `src/types/index.ts` naming the aliases; the six hand-written interfaces are gone |
| `chore(frontend): drop tailwind, next-pwa and the postcss config` | `@ducanh2912/next-pwa` (a webpack plugin the Turbopack build can never run), `tailwindcss` + `@tailwindcss/postcss` (imported, zero utility classes used), `postcss.config.mjs`, the `withPWA` wrapper and the `@import "tailwindcss"` line |

### Where it departed from the original plan

- **Detail pages are not cached.** The plan said `next: { revalidate: 60 }` everywhere, mirroring the home page. Caught in the browser: a cached fetch survives `router.refresh()`, so a just-posted topic or reply stayed invisible for up to a minute. `fetchApi` now defaults to no caching; the home board list, trending and "similar topics" pass `60` explicitly.
- **The search page was converted too** (not in the plan). It was the last client page fetching in an effect, and it was the only remaining ESLint error. Its form is a plain GET, so it needed no client code at all.
- **Aggregation silently drops `Meta.ordering`.** Adding `annotate(Count(...))` removed the `ORDER BY` from all three list queries — the topic list came back oldest-first until `_topics()`, `_boards()` and `_posts()` re-applied `order_by(*Model._meta.ordering)` explicitly. The query tests now assert order as well as query count.
- **`Count(..., distinct=True)` on the board annotations**, or the two joins multiply each other's rows and every count is wrong. Asserted in the tests.
- **`api/tests/test_queries.py` calls the router functions directly** rather than going through `TestClient`: query capture is per-connection, and `TestClient` runs handlers in worker threads. Each test compares a small seed against a large one, so a per-row query fails the test without hard-coding a magic number.
- **Auth pages stay client components.** The exit criterion's glob (`app/*/*/page.tsx`) also matches `auth/login`, `auth/signup` and `auth/forgot-password`; they are forms with no server data, so they keep `'use client'`.

### Exit criteria

```
npm run build: /boards/[id], /topics/[id], /profile/[username], /search   ✅ ƒ (server-rendered on demand)
'use client' in the four converted route pages                            ✅ none
list endpoints cost the same queries at 5 rows and 50                     ✅ boards ≤3, topics ≤4, posts ≤5, trending ≤2
mutation check: ordering ×3, board count distinct, post N+1, topic N+1,
  profile receiver                                                        ✅ 7/7 caught
pytest                                                                    ✅ 64 passed
eslint                                                                    ✅ 0 errors (was 2), 3 warnings (was 9)
headless Chrome: board/topic/search/profile HTML arrives rendered, ?page=2 differs
  in the HTML, new topic and reply appear via router.refresh() and in the next
  server render, quote keeps line breaks, profile edit + avatar persist,
  a missing topic hits the segment error boundary with the navbar intact   ✅
manage.py check · makemigrations --check · tsc --noEmit · docker compose config   ✅
```

---

## Phase 6 — Complete the product ✅ Done

**Completed 2026-09-18** on `feat/phase-6-complete`. Two product calls were yours: **add email verification** and **delete AdBanner**.

| Commit | What landed |
|---|---|
| `feat(api): email verification on signup, and an optional-auth dependency` | Signup emails a 6-digit code and login is refused until it comes back (`403`, distinct from a ban's `401`). `POST /api/auth/verify-email` and `/resend-verification`, both enumeration-safe and attempt-limited. The code helpers are now shared with the password-reset flow. `UserProfile.email_verified` (migration `accounts.0002`, existing profiles backfilled to verified) |
| `feat(api): moderation, mentions, my reactions and notification management` | **Moderation:** staff-only `PATCH /api/topics/{id}` (pin/lock, partial, via `.update()` so moderating doesn't bump the topic) and `DELETE` (posts first — the FKs are `PROTECT`). **Mentions:** `api/mentions.py` parses `@username` on topic create, post create and post edit, notifying up to 10 real users, skipping the author and anyone already notified. **Reactions:** `my_reactions` on `PostResponse`, plus `GET /api/posts/topic/{id}/my-reactions` for server-rendered pages. **Notifications:** pagination, `unread-count`, `read-all`, `DELETE /{id}`, and mark-read returns the row. **Optional auth:** `optional_user` dependency. `is_staff` on `UserResponse` |
| `feat(frontend): moderation, editing, notifications, settings and search tabs` (also deletes `AdBanner` and its five usages) | `PostCard` rewritten: reaction state shows which are yours and toggles, plus inline edit and delete for the author or staff. `TopicModeration` bar for staff. `/notifications` page (paginated, mark-all, delete) and `/settings` page (profile + change password), both linked from the user menu. Search gets type tabs with per-type counts and pagination. Profiles show reputation and badges. Signup gained the verification step |
| `chore: remove the adsense configuration` | The five env vars, the Django settings block and the README row; the AdSense script left the layout with the component |

### Where it departed from the original plan

- **Email verification uses `UserProfile.email_verified`, not `is_active`.** The plan's `is_active=False` conflates *unverified* with *banned*: a banned user would be told "verify your email", which also reveals that their password was right. Verified state is its own field; `is_active` still means banned.
- **`my_reactions` needed a second endpoint.** Phase 5 made the topic page server-rendered, and the Next server has no user cookie, so the list response's `my_reactions` is always empty there. The browser asks `GET /api/posts/topic/{id}/my-reactions` for one small per-viewer map instead of re-fetching every post.
- **The plan's `@`-stripping bug does not exist.** Checked in a real browser: the autocomplete inserts `@username ` with the `@` intact. Nothing to fix.
- **Moderation is pin/lock/delete only** — no subject editing. `TopicModerate` ignores anything else.
- **`/notifications` and `/settings` are client pages**, unlike the Phase 5 conversions: both render only the caller's own data, which the server cannot read.
- **Email verification's practical catch:** `EMAIL_BACKEND` still defaults to the console backend, so in dev the code is printed in the API log, not delivered. A real SMTP host is needed before signup works for anyone else (Phase 7 deployment).

### Bugs found while testing this phase

- **The settings form wiped what you typed.** The profile fetch resolved after the user started typing and overwrote the fields, then "Profile saved." reported success for an empty bio. The form now renders only once its data has arrived.
- **A moderated topic jumped to the top** of the board list, because `.save()` refreshed `last_updated` (`auto_now`). Moderation writes with `.update()`; there's a test.

### Exit criteria

```
pytest                                                              ✅ 85 passed (14 new)
mutation check: verification gate, staff gate, topic-delete cascade, mention
  exclusions, per-viewer my_reactions, pin-without-bump                ✅ 6/6 caught
headless Chrome, full product pass:
  signup → login refused → emailed code → signed in                    ✅
  reaction marked as mine, survives a reload, toggles off              ✅
  post edited and deleted from the UI                                  ✅
  @mention lands in the other user's notifications                     ✅
  notifications: mark-all clears the badge, delete empties the list    ✅
  settings: profile saved, password changed, reputation on the profile ✅
  moderation row hidden from normal users; staff pinned, locked, deleted ✅
  search type tabs filter and show per-type counts                     ✅
  no uncaught page errors                                              ✅
mention autocomplete inserts "@username " (plan claimed it strips @)   ✅ verified, no bug
npm run build · tsc --noEmit · eslint (0 errors) · manage.py check ·
  makemigrations --check · docker compose config                       ✅
```

---

## Phase 7 — Production shape and CI ✅ Done

**Completed 2026-09-18** on `feat/phase-7-production`. Both images were built and the production
stack was run end to end.

| Commit | What landed |
|---|---|
| `feat: production settings, gunicorn and whitenoise` | `DEBUG` defaults to **False**; an `if not DEBUG` block adds `SECURE_PROXY_SSL_HEADER`, SSL redirect, HSTS (1 year, preload), nosniff, referrer policy, `X_FRAME_OPTIONS=DENY`, secure/httpOnly/SameSite session and CSRF cookies, and `CSRF_TRUSTED_ORIGINS`. WhiteNoise middleware + `CompressedManifestStaticFilesStorage` serve the admin's assets. `gunicorn` and `whitenoise` added |
| `chore(deps): lock with hashes, drop python-jose, bump cryptography` | `requirements.lock` (52 packages, `--generate-hashes`), installed with `--require-hashes` in the image and CI. `python-jose` → **PyJWT**: jose pulls `ecdsa`, whose advisory has no fix. `cryptography` 42.0.5 → `>=46.0.6` (six advisories). `pip-audit` on the lock: clean |
| `feat(docker): non-root images, standalone frontend, split compose` | Backend image: no build toolchain (all wheels), `collectstatic` at build, uid 10001, gunicorn `CMD`. Frontend: three stages (deps → builder → runner), `output: "standalone"`, `NEXT_PUBLIC_API_URL` as a build ARG, `USER node`. Compose split into shared / `override` (dev bind mounts and reloaders) / `prod` (no mounts, gunicorn with uvicorn workers, `--forwarded-allow-ips`, restart policies). API healthcheck on `/api/health`, and the frontend waits for it |
| `ci: lint, test, deploy checks and dependency audit` | `.github/workflows/ci.yml`: backend job (Postgres + Redis services) runs `import api.main`, `ruff`, `pytest`, `makemigrations --check`, `check --deploy --fail-level WARNING` with generated secrets, and `pip-audit`; frontend job runs `tsc`, `eslint` and `next build`. `[tool.ruff]` in `pyproject.toml` |
| `docs: seed fixture and a README for deploying` | `boards/fixtures/seed_boards.json` (three starter boards, with a test that it still loads) and a rewritten README: Docker and non-Docker quick starts, the checks, and a production section (secrets, proxy, `client_max_body_size`, rebuild-on-`NEXT_PUBLIC_*`, relocking) |

### Where it departed from the original plan

- **`python-jose` was replaced by PyJWT** (not in the plan). `pip-audit` flagged its transitive `ecdsa` dependency, and that advisory has no fixed version. The JWT code is ~5 lines different; all tests passed unchanged.
- **`uv pip compile` generated the lock**, not `pip-compile` — `uv` was already on the machine and the output format is the same. The command is in the README.
- **Ruff runs with `E,F,I,B,UP`**, and three ignores that are documented in `pyproject.toml`: `B008` (FastAPI's `Depends()` defaults), `B904` (handlers translating exceptions), and `E402` for the two files that must call `django.setup()` before importing models. Fixing the rest turned up three broken annotations (`QuerySet` was never imported — a string annotation nobody evaluated), plus unused imports and stale typing.
- **The dev compose stops the frontend image at its `deps` stage** rather than building the production bundle for a bind-mounted dev server.
- **`check --deploy` runs with `--fail-level WARNING`** and freshly generated secrets, so a short or obviously-generated `SECRET_KEY` fails CI.

### Found by actually running the production stack

- **The API crashed on boot as a non-root user.** The `media_data` volume mounts over `/app/media`, and a fresh named volume inherits the image's ownership — which was root, so `upload.py` could not create `media/uploads`. The image now creates `media/uploads` and `media/avatars` owned by the app user. **Upgrading an existing deployment needs a one-time `chown -R 10001:10001 /app/media` on the volume.**

### Exit criteria

```
docker compose build (both images)                         ✅ backend 288MB, frontend built
docker compose -f … -f docker-compose.prod.yml up -d       ✅ postgres, redis, fastapi (healthy), frontend
  migrate + loaddata seed_boards                            ✅ 3 boards
  GET :8001/api/health · /api/boards/                       ✅ 200, the seeded boards
  GET :3000/ (next start, standalone)                       ✅ 200 in 0.15s, renders the seeded boards
  admin with DEBUG=False and real secrets                   ✅ 200, /static/admin/css/base.<hash>.css
  that asset through WhiteNoise                             ✅ 200, immutable cache, nosniff
prod compose bind mounts of the repo                       ✅ none
pip-audit -r requirements.lock --require-hashes            ✅ no known vulnerabilities
ruff check .                                               ✅ all checks passed
pytest                                                     ✅ 86 passed
manage.py check --deploy --fail-level WARNING              ✅ no issues
tsc --noEmit · eslint (0 errors) · npm run build           ✅
```

*(The images and volumes from this run were removed afterwards — the disk was at 99%.)*

---

## What this plan did not do

**Planned in [GO_LIVE_PLAN.md](GO_LIVE_PLAN.md)** — everything below is scheduled there as phase A (blocks going live, ~3½ days) or phase B (after it is live, ~4–5 days).

- **No deployment target.** There is no nginx/Caddy config, no TLS, no host. The README says what the proxy must do (`client_max_body_size`, `X-Forwarded-Proto`, `TRUSTED_PROXY_IPS`).
- **No real email.** `EMAIL_BACKEND` still defaults to the console, so verification and reset codes are printed, not delivered. Signup does not work for other people until SMTP is configured.
- **The database is still `boards_db`/`boards_user`** — the optional rename from the Phase 1 notes.
- **Known ceilings, all recorded in their phases:** anonymous visitors cost two auth requests per page load; the WebSocket only checks auth at connect; refresh-token reuse has a 30-second grace window; search is `icontains`, not full-text; `/notifications` and `/settings` are client-rendered.
- **Not covered:** rate limiting per user (only per IP), moderation of posts beyond delete, audit logging, backups, and any load testing.

---

## Deletion ledger

Everything below is verified unreferenced or superseded. Roughly **2,000 LOC, 432 KB of vendored assets, 4 services, 7 Python packages, 4 npm packages.**

**Delete outright**
- ✅ *Phase 1:* ~~`templates/` · `static/` · `boards/views.py` · `boards/urls.py` · `boards/forms.py` · `boards/templatetags/` · `accounts/views.py` · `accounts/forms.py` · `hash_out/asgi.py` · `notifications/views.py` + `notifications/tests.py` + `accounts/admin.py`~~ (`notifications/admin.py` now registers `Notification`)
- ~~`api/__init__.py:5-47` (stale duplicate app)~~ ✅ `b97e10c` · ~~`api/deps.py:19-32` + 10 `invalidate_cache()` calls~~ ✅ Phase 4 · ~~`api/auth.py:11,22` (unused `CryptContext`)~~ ✅ + ~~`:105-112` (`get_optional_user`)~~ ✅ Phase 2 · ~~`api/routers/topics.py:117-153` + `api/routers/posts.py:101-142` (HTMX fragments)~~ ✅ · ~~`api/tasks.py:7-30,101-107` (ES tasks)~~ ✅ whole file deleted in Phase 4
- ✅ *Phase 4:* ~~`boards/documents.py` + the ES config · `hash_out/celery.py` + the Celery config · `api/tasks.py`~~ · ~~`cms/models.py:15-70` (template-less Page models)~~ ✅ with all of Wagtail
- ~~`accounts/tests/` + `boards/tests/` — 497 LOC testing only deleted code~~ ✅ replaced by `api/tests/test_auth.py`
- ~~`frontend/public/{next,vercel,file,globe,window}.svg`~~ ✅ · dead imports: `api/main.py:18` (`os` twice), ~~`topics.py:4`~~ ✅, `topics.py:12`, `posts.py:12`, `notifications.py:5,11`, `search.py:4`, ~~`content.py:10`~~ ✅, `deps.py:6,10`, ~~`cms/models.py:10`~~ ✅, ~~`docker-compose.yml:2` (obsolete `version:`)~~ ✅

**Untrack (keep on disk)** — ~~`.env`, `db.sqlite3`, 67 `__pycache__` entries~~ ✅ Phase 0 · ~~`frontend/public/sw.js`, `frontend/public/workbox-*.js`~~ ✅ Phase 1

**Drop from requirements.txt** — ✅ *Phase 4:* ~~`elasticsearch`, `django-elasticsearch-dsl`, `celery[redis]`~~ · ✅ Phase 1: ~~`django-widget-tweaks`, `django-cors-headers`, `passlib[bcrypt]`, `httpx` (→ `requirements-dev.txt`), `wagtail`~~

**Drop from package.json** — ~~`htmx.org`~~ ✅, `@ducanh2912/next-pwa` (can never regenerate under Turbopack), `tailwindcss` + `@tailwindcss/postcss` (imported, zero classes used)

**Keep despite zero import hits** — `Pillow` (`UserProfile.avatar` and the upload verify; commented in `requirements.txt`), `psycopg2-binary` (the DB driver, loaded by name), `python-multipart` (FastAPI `UploadFile`).

**Deliberately keep**
- `Topic.slug` — currently written and never read, but it is the right URL shape later. If you truly want it gone, also remove `topics.py:22` and `schemas.py:57` or every topic response breaks.
- FastAPI *and* Django as two processes. They share one settings module, one ORM, one migration graph and one `User` table — it is one application with two entrypoints, not a service boundary. Porting 1,683 LOC to DRF buys architectural tidiness and nothing a user can see.

---

## Sequencing summary

```
Phase 0  ½ d   boot + hygiene        ✅ done 2026-09-17
Phase 1  1-2 d one UI stack          ✅ done 2026-09-17
Phase 2  2-3 d auth + is_active      ✅ done 2026-09-17
Phase 3  1-2 d visible bug sweep     ✅ done 2026-09-17
Phase 4  1-2 d 6 services → 3        ✅ done 2026-09-18
Phase 5  2-3 d server components      ✅ done 2026-09-18
Phase 6  3-5 d complete the product  ✅ done 2026-09-18
Phase 7  2-3 d production + CI       ✅ done 2026-09-18
                                     ≈ 3 weeks solo to a deployable, complete v1
```

**All phases are done.** What is left before this is live: a proxy with TLS, a real SMTP host, and a machine to run it on.

**Hard ordering constraints**
1. ~~Phase 0 before anything — the API does not import, so nothing else is verifiable.~~ ✅ Satisfied.
2. ~~Password validation + change-password endpoint + replacement tests before deleting the Django UI.~~ ✅ Satisfied in Phase 1.
3. ~~`api/__init__.py` truncation before any Celery/tasks work — importing `api.tasks` otherwise drags in a second `django.setup()`.~~ ✅ Satisfied (`b97e10c`).
4. ~~Fix `migration 0003` before any deployment has more than one board.~~ ✅ Satisfied in Phase 3.
5. ~~Fix `api/schemas.py` before generating TypeScript types.~~ ✅ Satisfied in Phase 5.
6. ~~Fix the N+1 queries before adding indexes.~~ ✅ Satisfied in Phase 5 (both in one commit; the shapes were settled first).
7. ~~Client-fetch consolidation before the httpOnly-cookie switch.~~ ✅ Satisfied in Phase 2.
8. ~~`CREATEDB` on `boards_user` before Phase 1's replacement tests.~~ ✅ Satisfied.

---

## Appendix — audit method and confidence

Eight parallel reviewers (api, django, frontend, security, removal, infra, completeness, coherence), each reading every file in its dimension; then one adversarial verifier per dimension re-opened every cited file and re-graded. 199 findings stood, 25 with corrected actions, 0 withdrawn — and 30 additional findings came from the verifiers themselves.

**Treat with care:** a 0-withdrawal rate means the verify pass was better at correcting than at killing. The claims I re-checked by hand and can vouch for directly: the missing `slowapi` declaration, the absent `Topic.tags` (`AttributeError` reproduced on a model instance), the duplicate app in `api/__init__.py`, the broken `content.py` internal call, the `.env`/`db.sqlite3` tracking with no `.gitignore`, the placeholder (not live) secret values, the 0-row committed database, the hardcoded working `FERNET_KEY`, the unauthenticated `boards/views.py` write path, the absent websocket-token guard, and the Elasticsearch 401 that blocks every write (reproduced via `Board.objects.create`).

**Executed against the real environment** (venv at `/home/akhil/code/test/hash_out/venv`): `import api.main` (44 routes); `makemigrations --check --dry-run` (clean); after Phase 0, `pytest` (7 passed), the endpoint smoke test (health, trending, topic, similar, boards, search, `/auth/me` — all 200), the AI endpoints (503 with no key, 502 upstream, 404 unknown topic), the startup check in all three cases (`DEBUG=False` on dev keys refuses to start; real keys start; `DEBUG=True` starts), and `docker compose config`. **Not executed:** migration replay from zero (the `0003` slug defect is read from source), any `docker compose up`, and any production build — verify those as you reach them.
