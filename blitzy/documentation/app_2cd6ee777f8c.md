# SimpleLogin Runtime Behavior Investigation — Evidence-Backed Answers (Q1–Q8)

This document empirically verifies **eight runtime behaviors** of the SimpleLogin
application's authentication, session-management, and email-forwarding subsystems.
Every answer is grounded in **what actually happened when the system was run** — not in
code reading alone.

**Deliverable location (required):** this file is
`blitzy/documentation/app_2cd6ee777f8c.md`. The filename is derived from the active source
branch `app_2cd6ee777f8c` per the "SWE-AtlasQnA-Repo" rule (`<source_branch_name>.md`); the
parent directory `blitzy/documentation/` was created as part of this deliverable. It is the
**only** file added to the repository.

For each question you will find:

- **(a) Command(s)** — the exact shell / HTTP / SQL command executed (fenced), using the
  container's real executables (`/app/venv/bin/*`).
- **(b) Raw output** — the complete captured output that command produced (fenced).
- **(c) Answer** — the concrete, resolved conclusion.
- **(d) `file:line` reference(s)** — the governing source location, naming the responsible
  function/method.
- **OBSERVED vs INFERRED** — every value is labelled; observed values are preferred and
  inferred (code-derived) explanations are called out explicitly.

**Methodology (run-first):** the canonical stack was built and started *before* any answer
was written, and each answer was derived from captured runtime output. The real canonical
entry points were exercised — the running Flask web app on `http://localhost:7777` (served by
gunicorn) and the standalone SMTP `email_handler.py` on port `20381`. No SimpleLogin source
file was modified; the investigation is strictly read-only (proven in **Read-Only Compliance
Proof** below) and the only artifact produced is this document.

**Security note:** reusable runtime authentication material (session-cookie HMAC signatures,
API-key secret values, and the two session-protection token values inside the pickled
session) is **redacted** in this document with clearly marked placeholders; identifiers
(UUIDs) needed to explain behavior are shown. All local dev credentials are public defaults
from `example.env` / `CONTRIBUTING.md`. All runtime session/key material was invalidated at
teardown. See **Security & Privacy Handling** below for the full policy and proof.

---

## Environment / Run Setup

All commands were executed **inside the canonical Docker container** built from the image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (image id `ea242796bbce`).
The pod host runs Python 3.13 and cannot host SimpleLogin's pinned Python-3.10 stack, so the
container is the canonical environment. Poetry is not installed in the image; the pinned
virtualenv is invoked directly via `/app/venv/bin/{python,gunicorn,flask,alembic,pip}`.

### Repository state (canonical `/app` checkout)

```bash
/app/venv/bin/python --version
git -C /app rev-parse HEAD
git -C /app rev-parse --abbrev-ref HEAD
```

```text
Python 3.10.18
2cd6ee777f8c2d3531559588bcfb18627ffb5d2c
HEAD
```

The `/app` checkout is at commit `2cd6ee777f8c` (matching the branch suffix
`app_2cd6ee777f8c`) in **detached-HEAD** state (`rev-parse --abbrev-ref HEAD` → `HEAD`). The
checkout carries **pre-existing, image-baked** tracked modifications that are **not** produced
by this investigation; they are enumerated and proven unchanged in **Read-Only Compliance
Proof** below.

### Pinned dependencies (exact `poetry.lock` pins present in `/app/venv`)

```bash
/app/venv/bin/pip freeze | grep -iE '^(Flask|Flask-Login|Werkzeug|itsdangerous|redis|SQLAlchemy|psycopg2-binary|arrow|bcrypt|gunicorn|aiosmtpd|alembic|Flask-Limiter)=='
```

```text
Flask-Limiter==1.4
Flask-Login==0.5.0
Flask==1.1.2
SQLAlchemy==1.3.24
Werkzeug==1.0.1
aiosmtpd==1.4.2
alembic==1.4.3
arrow==0.16.0
bcrypt==3.2.0
gunicorn==20.0.4
itsdangerous==1.1.0
psycopg2-binary==2.9.3
redis==4.6.0
```

### Infrastructure (bundled inside the image)

```bash
pg_lsclusters
redis-cli INFO server | grep redis_version
```

```text
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

redis_version:7.0.15
```

- **PostgreSQL 15** — online on port `5432`; role `myuser` / `mypassword`, database
  `simplelogin` (owner `myuser`). *(The stated requirement is PostgreSQL 13+; the image ships
  15, which satisfies it.)*
- **Redis 7.0.15** — online on port `6379`. **⚠ NONCONFORMING-VERSION NOTE (see finding
  resolution):** the checkpoint nominally targets **Redis 6**, but the attached image bundles
  **Redis 7.0.15** and no Redis-6 build is obtainable offline in this environment
  (`apt-cache policy redis-server` → candidate `7.0.15` only). The Redis-dependent behaviors
  under test (Q3 pickle serialization, the `session:<uuid4>` key name, and TTL selection) are
  governed entirely by SimpleLogin's application code (`app/session.py`) and the pinned
  `redis-py 4.6.0` client using only `GET`/`SET`/`SETEX`/`TTL`/`TYPE` on a single string value
  — operations whose semantics are identical across Redis 6 and 7. These results are therefore
  **version-invariant** for the behaviors observed; they are nonetheless labelled as obtained
  on **Redis 7.0.15**, and full Redis-6 conformance is **not** claimed.

### `.env` configuration

`.env` is a copy of `example.env` and is **git-ignored** (`git -C /app check-ignore .env` →
`.env`, exit 0), so reading/editing it is **not** a source modification. The values relevant
to this investigation:

```bash
grep -nE '^(URL|NOT_SEND_EMAIL|EMAIL_DOMAIN|DB_URI|FLASK_SECRET|MEM_STORE_URI)=' /app/.env
```

```text
6:URL=http://localhost:7777
19:NOT_SEND_EMAIL=true
22:EMAIL_DOMAIN=sl.local
75:DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
77:FLASK_SECRET=secret
199:MEM_STORE_URI=redis://localhost
```

**PostgreSQL port reconciliation (all three references addressed):** the repository contains
three different Postgres port values — `5432` (`example.env:L75`, and `CONTRIBUTING.md:L73/L75`
for the test DB), `15432` (`CONTRIBUTING.md:L100`, the `docker run … -p 15432:5432 postgres:13`
host mapping), and `35432` (`CONTRIBUTING.md:L94`, the `.env` `DB_URI` edit for that
docker-based workflow). Those `15432`/`35432` values belong to the CONTRIBUTING workflow where
Postgres runs in a **separate** container with a remapped host port. In **this** canonical
image PostgreSQL is bundled and listens on `5432` directly, and `.env`'s `DB_URI` already uses
`5432` (`example.env:L75`), so **no port edit was needed** — the app connects on `5432` out of
the box.

**Redis session store (decisive for Q3 & Q4):** `MEM_STORE_URI` defaults to `None`
(`app/config.py:L568`), and `create_app()` installs the Redis services **only** `if
MEM_STORE_URI:` (`server.py:L163-L165`, `initialize_redis_services(app, MEM_STORE_URI)`). That
call swaps Flask's default client-side cookie session for the server-side `RedisSessionStore`.
`.env` sets `MEM_STORE_URI=redis://localhost`, so the real `RedisSessionStore` path is active
(confirmed by the presence of `session:*` keys in Redis in Q3). This is a canonical-config
prerequisite (Redis is a stated stack component), **not** a bypass.

### Canonical entry points — how they were started

The web app is served with **gunicorn** (the canonical production WSGI launcher,
`gunicorn wsgi:app -b 0.0.0.0:7777`; `wsgi.py` is `from server import create_app; app =
create_app()`), using **two** sync workers (`-w 2`). Running under gunicorn (debug **off**)
yields production-accurate HTTP responses — important for Q2, where Flask's dev-mode
interactive debugger would otherwise replace the real HTTP 500 with an HTML debugger page. The
database was migrated and the dev account seeded via the documented
`alembic upgrade head && flask dummy-data` step (`CONTRIBUTING.md:L106`) at environment
bootstrap; re-running `alembic upgrade head` is idempotent and confirms the schema is at head.

```bash
# migration state (idempotent; DB already at head)
cd /app && /app/venv/bin/alembic current
# web app (canonical WSGI launcher), two sync workers, logs captured to /tmp/inv/gunicorn.log
cd /app && FLASK_APP=server.py /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30 > /tmp/inv/gunicorn.log 2>&1 &
# standalone SMTP email handler on port 20381, logs to /tmp/inv/eh.log
cd /app && FLASK_APP=server.py /app/venv/bin/python email_handler.py > /tmp/inv/eh.log 2>&1 &
```

```text
# alembic current
32f25cbf12f6 (head)

# gunicorn startup (/tmp/inv/gunicorn.log)
[2026-07-13 18:05:51 +0000] [5054] [INFO] Starting gunicorn 20.0.4
[2026-07-13 18:05:51 +0000] [5054] [INFO] Listening at: http://0.0.0.0:7777 (5054)
[2026-07-13 18:05:51 +0000] [5054] [INFO] Using worker: sync
[2026-07-13 18:05:51 +0000] [5055] [INFO] Booting worker with pid: 5055
[2026-07-13 18:05:51 +0000] [5056] [INFO] Booting worker with pid: 5056

# email_handler startup (/tmp/inv/eh.log)
2026-07-13 18:05:58,391 - SL - INFO - 5069 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 18:05:58,393 - SL - DEBUG - 5069 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Both listeners were confirmed up:

```bash
nc -z -w2 localhost 7777 && echo "7777 OPEN"
nc -z -w2 localhost 20381 && echo "20381 OPEN"
```

```text
7777 OPEN
20381 OPEN
```

The seeded dev account was confirmed present:

```bash
PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin \
  -c "SELECT id,email,activated,is_admin FROM users WHERE email='john@wick.com';"
```

```text
 id |     email     | activated | is_admin
----+---------------+-----------+----------
  1 | john@wick.com | t         | t
(1 row)
```

The SimpleLogin logger runs at **DEBUG** (`app/log.py`), and `after_request()` logs every HTTP
transaction (method, path, **status code**, timing) at `server.py:L284`; the global
`error_handler()` logs exceptions at `server.py:L390`. These are the source of the log evidence
used for Q2 and Q8, captured from `/tmp/inv/gunicorn.log`.

### Confirming login (canonical `/auth/login` flow)

The real login route is **`/auth/login`** (the auth blueprint sets `url_prefix="/auth"`).
Authentication uses a Flask-WTF form with a CSRF token, so an authenticated `slapp` cookie is
obtained by `GET /auth/login` (to receive a session + CSRF token), extracting the hidden
`csrf_token`, then `POST /auth/login` with the credentials and that token, using a persistent
cookie jar. The literal token is extracted programmatically (no placeholder):

```bash
JAR=/tmp/inv/auth_jar.txt
CSRF=$(curl -s -c "$JAR" http://localhost:7777/auth/login \
  | grep -o 'name="csrf_token"[^>]*value="[^"]*"' | grep -o 'value="[^"]*"' | cut -d'"' -f2)
curl -s -i -c "$JAR" -b "$JAR" \
  --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" \
  http://localhost:7777/auth/login | grep -iE '^HTTP|^Location|^Set-Cookie'
```

```text
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=f6b7156a-d101-481d-a8e8-103d26430534.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:06:15 GMT; HttpOnly; Path=/; SameSite=Lax
```

The **302 redirect to `/dashboard/`** confirms a successful login with the canonical dev
credentials `john@wick.com / password` (`CONTRIBUTING.md:L108-L109`). The resulting
authenticated cookie jar (`slapp` session UUID `f6b7156a-d101-481d-a8e8-103d26430534`) was
reused for the API/session questions.

**Observation tooling vs project dependencies:** `curl`, `redis-cli`, `psql`, and `swaks` are
external observation tools and are **not** SimpleLogin dependencies. `aiosmtpd` **is** a pinned
project dependency (`aiosmtpd==1.4.2`, used by `email_handler.py` itself); in Q5 it is *also*
used to stand up a throwaway local mail sink. No packages were added, updated, or removed.

---

## Q1 — API auth via browser session (no API key)

**Question:** What actual HTTP status code and response body structure come back when an
authenticated browser session calls an `/api/*` endpoint **without** the API-key header?

**(a) Commands**

```bash
# (1) authenticated slapp cookie only, NO Authentication header
curl -sS -i -b /tmp/inv/auth_jar.txt http://localhost:7777/api/user_info
# (2) contrast: no cookie, no key
curl -sS -i http://localhost:7777/api/user_info
```

**(b) Raw output**

```text
# (1) session cookie, NO Authentication header
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:10:15 GMT
Connection: close
Content-Type: application/json
Content-Length: 244
Access-Control-Allow-Origin: *
Set-Cookie: slapp=f6b7156a-d101-481d-a8e8-103d26430534.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:10:15 GMT; HttpOnly; Path=/; SameSite=Lax

{"can_create_reverse_alias":true,"connected_proton_address":null,"email":"john@wick.com","in_trial":false,"is_premium":true,"max_alias_free_plan":5,"name":"John Wick","profile_picture_url":"http://localhost:7777/static/upload/profile_pic.svg"}
```

```text
# (2) contrast: no cookie, no key
HTTP/1.1 401 UNAUTHORIZED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:10:15 GMT
Connection: close
Content-Type: application/json
Content-Length: 26
Access-Control-Allow-Origin: *
Set-Cookie: slapp=8229faa2-ff2b-4758-a7bd-8b9eba3adc8c.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:10:15 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Wrong api key"}
```

**(c) Answer (OBSERVED)**

1. **Status code:** `200 OK`.
2. **Body structure:** a flat JSON object (`Content-Type: application/json`,
   `Content-Length: 244`) with exactly these keys: `can_create_reverse_alias` (bool),
   `connected_proton_address` (null), `email` (str), `in_trial` (bool), `is_premium` (bool),
   `max_alias_free_plan` (int), `name` (str), `profile_picture_url` (str). For `john@wick.com`
   the full body is shown verbatim above.
3. **Contrast proves the fallback:** with **no** cookie and **no** key the identical endpoint
   returns `401 UNAUTHORIZED` with body `{"error":"Wrong api key"}` (`Content-Length: 26`).
   Therefore the `200` was produced by the **session fallback**, not by any API key.

**(d) `file:line` references**

- API key read from the **non-standard `Authentication`** header: `app/api/base.py:L17` —
  `api_code = request.headers.get("Authentication")`, in `authorize_request()`.
- Session fallback (no key present, user authenticated): `app/api/base.py:L20-L25` —
  `if not api_key: if current_user.is_authenticated: … g.user = current_user`.
- The `401 "Wrong api key"` branch (no key, not authenticated): `app/api/base.py:L27` —
  `return jsonify(error="Wrong api key"), 401`.
- Endpoint `user_info()` guarded by `@require_api_auth`: `app/api/views/user_info.py:L50-L52`;
  body assembled by `user_to_dict()` at `app/api/views/user_info.py:L28`.

---

## Q2 — Privileged operation via browser session (non-standard error)

**Question:** What specific (non-standard) status code and exact error message text does the
system actually return when a privileged operation is attempted with **only** a browser
session? *(Treated as "observe, do not assume".)*

Because `DELETE /api/user` is a destructive endpoint, the user's state and the account-deletion
job queue were read **before and after** both attempts to prove that **no deletion occurred or
was scheduled**.

**(a) Commands**

```bash
# SAFETY BEFORE
PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin -c \
  "SELECT id,email,activated,disabled,delete_on FROM users WHERE email='john@wick.com';" -c \
  "SELECT count(*) AS delete_account_jobs FROM job WHERE name='delete-account';"

# (Q2a) privileged endpoint DELETE /api/user (@require_api_sudo) with ONLY the browser session
curl -sS -i -X DELETE -b /tmp/inv/auth_jar.txt http://localhost:7777/api/user

# (Q2b) contrast: key-authenticated caller whose key has NO active sudo mode
#        (key created canonically via POST /api/api_key; secret held in $Q2KEY)
curl -sS -i -X DELETE -H "Authentication: $Q2KEY" http://localhost:7777/api/user

# new server-log lines produced by the requests
tail -n +43 /tmp/inv/gunicorn.log

# SAFETY AFTER (identical query)
```

**(b) Raw output — SAFETY BEFORE**

```text
 id |     email     | activated | disabled | delete_on
----+---------------+-----------+----------+-----------
  1 | john@wick.com | t         | f        |
(1 row)

 delete_account_jobs
---------------------
                   0
(1 row)
```

**(b) Raw output — Q2a (browser session only)**

```text
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:11:18 GMT
Connection: close
Content-Type: application/json
Content-Length: 27
Access-Control-Allow-Origin: *
Set-Cookie: slapp=f6b7156a-d101-481d-a8e8-103d26430534.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:11:18 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Internal error"}
```

**(b) Raw output — Q2b (key-authenticated caller without active sudo → 440)**

```text
HTTP/1.1 440 UNKNOWN
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:11:18 GMT
Connection: close
Content-Type: application/json
Content-Length: 22
Access-Control-Allow-Origin: *
Set-Cookie: slapp=e457603e-9e21-4a32-9085-4c6f96dfbec5.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:11:18 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Need sudo"}
```

**(b) Raw output — new server-log lines (`/tmp/inv/gunicorn.log`)**

```text
2026-07-13 18:11:18,560 - SL - ERROR - 5055 - "/app/server.py:390" - error_handler() -  - 'NoneType' object has no attribute 'sudo_mode_at'
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/app/api/base.py", line 69, in decorated
    if not check_sudo_mode_is_active(g.api_key):
  File "/app/app/api/base.py", line 47, in check_sudo_mode_is_active
    return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= arrow.now().shift(
AttributeError: 'NoneType' object has no attribute 'sudo_mode_at'
2026-07-13 18:11:18,561 - SL - DEBUG - 5055 - "/app/server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 500, takes 0.0043833255767822266
2026-07-13 18:11:18,576 - SL - DEBUG - 5055 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/api_key ImmutableMultiDict([]) 201, takes 0.006478786468505859
2026-07-13 18:11:18,598 - SL - DEBUG - 5055 - "/app/server.py:284" - after_request() -  - 127.0.0.1 DELETE /api/user ImmutableMultiDict([]) 440, takes 0.0037653446197509766
```

**(b) Raw output — SAFETY AFTER**

```text
 id |     email     | activated | disabled | delete_on
----+---------------+-----------+----------+-----------
  1 | john@wick.com | t         | f        |
(1 row)

 delete_account_jobs
---------------------
                   0
(1 row)
```

**(c) Answer (OBSERVED)**

- With **only a browser session**, the privileged operation returns **HTTP `500 INTERNAL
  SERVER ERROR`** with the exact body **`{"error":"Internal error"}`**. This is *not* a standard
  authorization error (not 401/403) and is *not* the `440 "Need sudo"` one might expect.
- **Root cause (observed in the traceback, reported as-found — NOT patched, per read-only
  scope):** on the browser-session path `authorize_request()` sets `g.api_key = None`. The
  `@require_api_sudo` wrapper then calls `check_sudo_mode_is_active(g.api_key)`, which
  dereferences `api_key.sudo_mode_at` on a `None` object, raising `AttributeError: 'NoneType'
  object has no attribute 'sudo_mode_at'`. The global exception handler converts that into
  `{"error":"Internal error"}` / `500`.
- **The `440 "Need sudo"` path does exist**, but only for a **key-authenticated** caller whose
  key has no active sudo mode (Q2b, using a freshly created key whose `sudo_mode_at` is `NULL`):
  that returns **`440 UNKNOWN`** with body `{"error":"Need sudo"}`. (`440` is a non-standard
  status code, so Werkzeug's reason phrase is literally `UNKNOWN`.)
- **Safety (OBSERVED):** the before/after queries are **identical** — `john@wick.com` remains
  `activated=t`, `disabled=f`, `delete_on` NULL, and the `delete-account` job count stays `0`.
  Both attempts failed **before** reaching `delete_user()`'s body, so **no deletion occurred and
  none was scheduled**. (A *successful* delete would emit `LOG.w("schedule delete account
  job …")` and `Job.create(name="delete-account", …)` then return `{"ok":true}` — none of which
  appears in the log or the job table.)

**(d) `file:line` references**

- `g.api_key = None` on the browser-session path: `app/api/base.py:L42` — `g.api_key = api_key`
  (where `api_key` is `None`), in `authorize_request()`.
- The failing dereference: `app/api/base.py:L46-L49`, specifically `L47`
  `return api_key.sudo_mode_at and g.api_key.sudo_mode_at >= …`, in `check_sudo_mode_is_active()`.
- The wrapper that invokes it: `app/api/base.py:L69` — `if not check_sudo_mode_is_active(g.api_key):`
  in `require_api_sudo`'s `decorated()`; the `440` return is `app/api/base.py:L70` —
  `return jsonify(error="Need sudo"), 440`.
- The `{"error":"Internal error"}` / `500`: `server.py:L389-L392` — `error_handler()` logs the
  exception at `L390` (`LOG.e(e)`) and returns `jsonify(error="Internal error"), 500` at `L392`
  for `/api/` paths.
- Endpoint `delete_user()` (`DELETE /api/user`, `@require_api_sudo`): route at
  `app/api/views/user.py:L11`, function `delete_user()` at `L14`; the deletion is *scheduled* via
  `Job.create(name=config.JOB_DELETE_ACCOUNT, …)` where `JOB_DELETE_ACCOUNT = "delete-account"`
  (`app/config.py:L306`).

---

## Q3 — Session data format in storage

**Question:** Capture the raw bytes from the storage backend; what format are they in; what
keys are present in the deserialized data for an authenticated session; and how is the session
storage key structured?

**(a) Commands** *(all Python runs use `/app/venv/bin/python`, which has `itsdangerous`/`redis`
on its path — the system `python` does not)*

```bash
# fresh authenticated session into a dedicated jar
JAR=/tmp/inv/q3_jar.txt
CSRF=$(curl -s -c "$JAR" http://localhost:7777/auth/login \
  | grep -o 'name="csrf_token"[^>]*value="[^"]*"' | grep -o 'value="[^"]*"' | cut -d'"' -f2)
curl -s -c "$JAR" -b "$JAR" \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" http://localhost:7777/auth/login -o /dev/null
COOKIE=$(grep slapp "$JAR" | awk '{print $NF}')   # full slapp cookie value <uuid>.<sig>
UUID=$(echo "$COOKIE" | cut -d. -f1)

# session TYPE + TTL straight from Redis
redis-cli TYPE "session:$UUID"
redis-cli TTL  "session:$UUID"

# unsign the cookie, read the raw pickle bytes, and pickle.loads them (see /tmp/inv/q3.py)
/app/venv/bin/python /tmp/inv/q3.py "$COOKIE"
```

`/tmp/inv/q3.py` (temporary observation script, removed at teardown) unsigns the cookie with
the same signer the app uses, reads `session:<uuid>` from Redis, prints the raw bytes
(`redis-cli --no-raw GET` equivalent) with only the two session-protection secret **values**
masked, then `pickle.loads` and enumerates keys/values.

**(b) Raw output**

```text
=== Q3 step 2: redis TYPE/TTL (redis-cli) ===
$ redis-cli TYPE session:1411267f-44d9-425b-add6-e36f2529ea11
string
$ redis-cli TTL session:1411267f-44d9-425b-add6-e36f2529ea11
604800

=== Q3 step 3: unsign + raw bytes + pickle.loads (ALL via /app/venv/bin/python) ===
unsigned UUID   = 1411267f-44d9-425b-add6-e36f2529ea11
redis key       = session:1411267f-44d9-425b-add6-e36f2529ea11
redis TYPE      = string
redis TTL       = 604800 seconds
raw len         = 300 bytes
first 3 bytes   = b'\x80\x04\x95' (pickle PROTO opcode: 0x80 0x04 => protocol 4)
raw bytes (redis-cli --no-raw GET), verbatim with 2 secret values masked:
b'\x80\x04\x95!\x01\x00\x00\x00\x00\x00\x00}\x94(\x8c\n_permanent\x94\x88\x8c\x06_fresh\x94\x88\x8c\ncsrf_token\x94\x8c(<REDACTED-csrf_token-40-hex>\x94\x8c\x08_user_id\x94\x8c$0fb423fb-e47b-41b7-bfcd-c21453f33dca\x94\x8c\x03_id\x94\x8c\x80<REDACTED-_id-128-hex>\x94\x8c\tsudo_time\x94J\xbc+Uju.'
pickle.loads    = dict
deserialized keys = ['_permanent', '_fresh', 'csrf_token', '_user_id', '_id', 'sudo_time']
--- key : value(type) ---
  '_permanent': True  <bool>
  '_fresh'    : True  <bool>
  'csrf_token': <REDACTED session-protection value, len=40> <str>
  '_user_id'  : '0fb423fb-e47b-41b7-bfcd-c21453f33dca'  <str>
  '_id'       : <REDACTED session-protection value, len=128> <str>
  'sudo_time' : 1783966652  <int>
```

**(c) Answer (OBSERVED)**

1. **Raw bytes:** shown above — a **300-byte** Redis `string` value beginning `\x80\x04\x95…`
   and ending `…\tsudo_time\x94J\xbc+Uju.`. (The two session-protection secret *values* — the
   40-hex `csrf_token` and 128-hex `_id` — are masked in this document; every pickle opcode,
   key name, and length prefix is shown verbatim, and the session was invalidated at teardown.)
2. **Format:** **Python `pickle`, protocol 4** — the leading `\x80\x04` is the pickle `PROTO 4`
   opcode. `pickle.loads` yields a `dict`.
3. **Deserialized key set (authenticated session):**
   `['_permanent', '_fresh', 'csrf_token', '_user_id', '_id', 'sudo_time']` — i.e.
   `_permanent=True`, `_fresh=True`, `csrf_token` (40-hex str), `_user_id` (the Flask-Login user
   id, a UUID string `0fb423fb-…`), `_id` (the Flask-Login session-protection identifier, a
   128-hex str), and `sudo_time` (int epoch). *(Note the real set includes `_permanent` and
   `sudo_time` in addition to the commonly expected `_user_id`/`_fresh`/`_id`/`csrf_token`.)*
4. **Key structure:** the Redis storage key is **`session:<uuid4>`** (here
   `session:1411267f-44d9-425b-add6-e36f2529ea11`). The **cookie** itself is
   `<uuid4>.<hmac-signature>` (an `itsdangerous.Signer` value with `salt="session"`,
   `key_derivation="hmac"`). The observed **TTL is `604800` seconds (exactly 7 days)** because
   the session contains `_user_id` (authenticated); unauthenticated sessions get 300 s.

**(d) `file:line` references**

- Value serialization = Python pickle: `app/session.py:L91` — `val = pickle.dumps(dict(session))`,
  in `RedisSessionStore.save_session()`.
- Storage-key structure `session:<id>`: `app/session.py:L18` (`SESSION_PREFIX = "session"`) and
  `app/session.py:L44-L45` (`_get_key()` → `f"{SESSION_PREFIX}:{session_Id}"`).
- Cookie signer (`<uuid>.<hmac>`): `app/session.py:L38-L41` — `_get_signer()` →
  `itsdangerous.Signer(app.secret_key, salt="session", key_derivation="hmac")`; `app.secret_key
  = FLASK_SECRET` (`server.py:L151`), `FLASK_SECRET="secret"` (`example.env:L77`).
- TTL rule (7-day vs 300 s): `app/session.py:L95-L96` (`if "_user_id" not in session: ttl = 300`)
  in `save_session()`, with `permanent_session_lifetime = timedelta(days=7)` (`server.py:L207`);
  `604800 = 7 × 86400`.
- Session cookie name `slapp`: `app/config.py:L199` (`SESSION_COOKIE_NAME = "slapp"`).

---

## Q4 — Session identifier behavior during login (fixation test)

**Question:** If you capture the session ID from the cookie **before** login, then complete
login, does the identifier value stay the same or get replaced? Show the actual before/after
values.

**(a) Commands** *(two independent runs; each uses a fresh cookie jar; identifiers unsigned
with `/app/venv/bin/python`)*

```bash
# BEFORE: GET /auth/login on a fresh jar -> capture Set-Cookie slapp (from response headers)
curl -s -D /tmp/inv/q4_get.hdr -c /tmp/inv/q4_jar.txt http://localhost:7777/auth/login
# (extract csrf_token from the returned page)
# AFTER: POST /auth/login with correct creds on the SAME jar -> capture Set-Cookie slapp
curl -s -D /tmp/inv/q4_post.hdr -c /tmp/inv/q4_jar.txt -b /tmp/inv/q4_jar.txt \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  --data-urlencode "csrf_token=$CSRF" http://localhost:7777/auth/login -o /dev/null
# unsign BOTH cookies to reveal the embedded UUID identifier:
/app/venv/bin/python -c 'import sys,itsdangerous; \
  print(itsdangerous.Signer("secret",salt="session",key_derivation="hmac").unsign(sys.argv[1]).decode())' "<cookie>"
```

The full orchestration (both runs, header capture, unsigning, and the SAME/CHANGED verdict) is
the temporary script `/tmp/inv/q4_run.sh` (removed at teardown).

**(b) Raw output**

```text
----- RUN 1 : BEFORE = GET /auth/login (fresh jar, unauthenticated) -----
BEFORE Set-Cookie (from GET response headers):
Set-Cookie: slapp=96f40087-b416-484b-9e22-bb06c865e685.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:18:16 GMT; HttpOnly; Path=/; SameSite=Lax
----- RUN 1 : POST /auth/login (correct creds, SAME jar) -----
POST status + AFTER Set-Cookie (from POST response headers):
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=96f40087-b416-484b-9e22-bb06c865e685.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:18:16 GMT; HttpOnly; Path=/; SameSite=Lax
BEFORE uuid = 96f40087-b416-484b-9e22-bb06c865e685
AFTER  uuid = 96f40087-b416-484b-9e22-bb06c865e685
UUID VERDICT = SAME (identifier NOT rotated on login)
FULL-COOKIE VERDICT = BYTE-IDENTICAL before vs after

----- RUN 2 : BEFORE = GET /auth/login (fresh jar, unauthenticated) -----
BEFORE Set-Cookie (from GET response headers):
Set-Cookie: slapp=576870dd-5b3e-4f10-98be-646077a84d39.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:18:16 GMT; HttpOnly; Path=/; SameSite=Lax
----- RUN 2 : POST /auth/login (correct creds, SAME jar) -----
POST status + AFTER Set-Cookie (from POST response headers):
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/
Set-Cookie: slapp=576870dd-5b3e-4f10-98be-646077a84d39.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:18:16 GMT; HttpOnly; Path=/; SameSite=Lax
BEFORE uuid = 576870dd-5b3e-4f10-98be-646077a84d39
AFTER  uuid = 576870dd-5b3e-4f10-98be-646077a84d39
UUID VERDICT = SAME (identifier NOT rotated on login)
FULL-COOKIE VERDICT = BYTE-IDENTICAL before vs after
```

**(c) Answer (OBSERVED)**

1. **Before value (pre-login), RUN 1:** embedded identifier UUID
   **`96f40087-b416-484b-9e22-bb06c865e685`**.
2. **After value (post-login, on the `302` response), RUN 1:** embedded identifier UUID
   **`96f40087-b416-484b-9e22-bb06c865e685`** — the same.
3. **Verdict:** the identifier **stays the SAME** — it is **not replaced** on login. In fact the
   entire signed cookie is **byte-identical** before and after (the deterministic `Signer` HMAC
   over an unchanged UUID yields the same signature). **RUN 2 independently reproduces this**
   (different fresh UUID `576870dd-…`, again unchanged across login). Two consistent runs
   confirm SimpleLogin does **not** rotate the session identifier at login, so login does not
   itself defend against session fixation.

**(d) `file:line` references**

- Identifier reuse: `app/session.py:L68-L80` — `open_session()` calls
  `extract_and_validate_session_id()` and generates a fresh UUID
  (`ServerSession(session_id=str(uuid.uuid4()))`, `L71`/`L80`) **only** when no valid signed
  identifier is present; otherwise it reuses the validated `session_id`.
- Fresh UUID only on purge/logout: `app/session.py:L61-L64` — `purge_session()` (`session_id =
  str(uuid.uuid4())` at `L64`).
- Flask-Login session protection: `app/extensions.py:L8` — `login_manager.session_protection =
  "strong"`. **INFERRED (corroborated by research):** `"strong"` compares a client-derived
  identifier and can mark the session non-fresh / delete it on mismatch, but it does **not**
  regenerate the cookie's session identifier at `login_user()`. The **observed** byte-identical
  before/after values above are the primary evidence; this note only explains *why* they match.

---

## Q5 — Email-forwarding header handling

**Question:** If an incoming email has a custom `X-` header, a `Received` header, and a `Reply-To`
header, which of those three actually make it through to the forwarded message and which get
stripped? Send an actual test email and examine the result.

**How the forwarded message was captured.** In dev the app runs with `NOT_SEND_EMAIL=true`, which
suppresses the outbound send. To observe the *real* forwarded bytes, the outbound leg was pointed
at a local capture SMTP sink: `NOT_SEND_EMAIL` was temporarily commented out and
`POSTFIX_SERVER`/`POSTFIX_PORT` set to the sink, `email_handler.py` was restarted, a real inbound
message was sent via `swaks`, and the message the handler emitted was captured verbatim. All
changes were reverted afterward (proof below). The inbound target is the seeded, enabled alias
`e1@sl.local` (`alias.id=5`) which forwards to mailbox `john@wick.com` (`mailbox.id=1`).

**(a) Commands** *(the full observation scripts are `/tmp/inv/q5_sink.py` and the shell steps
below, all removed at teardown)*

```bash
# 0) back up .env, then TEMPORARILY point the outbound forward at a local capture sink
cp /app/.env /tmp/inv/env.bak
sed -i 's/^NOT_SEND_EMAIL=true/#NOT_SEND_EMAIL=true  # Q5-TEMP/' /app/.env
printf '\nPOSTFIX_SERVER=localhost\nPOSTFIX_PORT=2525\n' >> /app/.env

# 1) start an aiosmtpd sink on 127.0.0.1:2525 that writes envelope.content to a file
/app/venv/bin/python /tmp/inv/q5_sink.py &            # writes /tmp/inv/q5_sink/forwarded.eml

# 2) restart email_handler.py so it reads the new .env
#    (kill the old pid, then: FLASK_APP=server.py /app/venv/bin/python email_handler.py)

# 3) send a REAL inbound message to the alias with all three test headers
swaks --server 127.0.0.1:20381 \
      --from sender-outside@external.test --to e1@sl.local \
      --h-Subject "Q5 header forwarding test" \
      --header    "X-Custom-Header: q5-custom-VALUE-abc123" \
      --add-header "Received: from evilrelay.example (evilrelay.example [203.0.113.55]) by mx.inbound; Mon, 13 Jul 2026 00:00:00 +0000" \
      --header    "Reply-To: replyto-ORIG@external.test" \
      --body "Q5 forwarding body: testing X-Custom-Header, Received, Reply-To fate."

# 4) read the captured forwarded message
cat /tmp/inv/q5_sink/forwarded.eml

# 5) cleanup: restore .env, stop the sink, restart email_handler in canonical config
cp /tmp/inv/env.bak /app/.env && diff -u /tmp/inv/env.bak /app/.env
```

**(b) Raw output**

*Step 0 — the exact (temporary) `.env` change (unified diff, reverted at cleanup):*

```diff
--- /tmp/inv/env.bak
+++ /app/.env
@@ -16,7 +16,7 @@
 # Only print email content, not sending it, for local development
-NOT_SEND_EMAIL=true
+#NOT_SEND_EMAIL=true  # Q5-TEMP: disabled to capture forwarded mail
@@ -197,3 +197,7 @@
 MEM_STORE_URI=redis://localhost
+
+# Q5-TEMP: route outbound forward to local capture sink (reverted at cleanup)
+POSTFIX_SERVER=localhost
+POSTFIX_PORT=2525
```

*Steps 1–2 — sink up and handler restarted on the new config:*

```text
Q5 sink listening on 127.0.0.1:2525
2026-07-13 18:35:57,850 - SL - INFO - 7309 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
```

*Step 3 — the inbound `swaks` transcript (the message actually delivered to the handler):*

```text
<-  220 8de7fd3e9430 Python SMTP 1.4.2
 -> MAIL FROM:<sender-outside@external.test>
<-  250 OK
 -> RCPT TO:<e1@sl.local>
<-  250 OK
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Mon, 13 Jul 2026 18:36:16 +0000
 -> To: e1@sl.local
 -> From: sender-outside@external.test
 -> Subject: Q5 header forwarding test
 -> Message-Id: <20260713183616.007333@8de7fd3e9430>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 -> Received: from evilrelay.example (evilrelay.example [203.0.113.55]) by mx.inbound; Mon, 13 Jul 2026 00:00:00 +0000
 -> X-Custom-Header: q5-custom-VALUE-abc123
 -> Reply-To: replyto-ORIG@external.test
 ->
 -> Q5 forwarding body: testing X-Custom-Header, Received, Reply-To fate.
 -> .
<-  250 Message accepted for delivery
```

*The handler's own log for this message — note the explicit header rewrites:*

```text
... handle() ... reply-to:replyto-ORIG@external.test ... headers:[('Date', ...), ('To', 'e1@sl.local'),
    ('From', 'sender-outside@external.test'), ('Subject', 'Q5 header forwarding test'),
    ('Message-Id', ...), ('X-Mailer', 'swaks ...'),
    ('Received', 'from evilrelay.example (evilrelay.example [203.0.113.55]) by mx.inbound; ...'),
    ('X-Custom-Header', 'q5-custom-VALUE-abc123'), ('Reply-To', 'replyto-ORIG@external.test'),
    ('Content-Transfer-Encoding', '7bit')]
... "/app/email_handler.py:867" - forward_email_to_mailbox() - From header,
    new:"sender-outside at external.test" <sender-outside_at_external_test_uhcmcgq@sl.local>,
    old:sender-outside@external.test
... "/app/email_handler.py:873" - forward_email_to_mailbox() - Reply-To header,
    new:"replyto-ORIG at external.test" <replyto-orig_at_external_test_lloihftr@sl.local>, old:None
... "/app/email_handler.py:893" - Forward mail from sender-outside@external.test to john@wick.com
```

*Step 4 — the FULL captured forwarded message (verbatim bytes the handler emitted to the sink):*

```text
Date: Mon, 13 Jul 2026 18:36:16 +0000
Subject: Q5 header forwarding test
Message-Id: <20260713183616.007333@8de7fd3e9430>
Content-Transfer-Encoding: 7bit
X-SimpleLogin-Type: Forward
X-SimpleLogin-EmailLog-ID: 4
X-SimpleLogin-Envelope-From: sender-outside@external.test
X-SimpleLogin-Original-From: sender-outside@external.test
X-SimpleLogin-Envelope-To: e1@sl.local
From: "sender-outside at external.test"
 <sender-outside_at_external_test_uhcmcgq@sl.local>
Reply-To: "replyto-ORIG at external.test"
 <replyto-orig_at_external_test_lloihftr@sl.local>
To: e1@sl.local

Q5 forwarding body: testing X-Custom-Header, Received, Reply-To fate.
```

*Precise classification of the three test headers (run against the header block only):*

```text
[X-Custom-Header]                          count=0 -> STRIPPED
[Received]                                 count=0 -> STRIPPED
[original Received token evilrelay.example] count=0 -> ABSENT
[Reply-To header present?]                 count>=1 -> PRESENT (rewritten)
[original Reply-To value replyto-ORIG@external.test] count=0 -> ORIGINAL ABSENT
[rewritten Reply-To reverse-alias] replyto-orig_at_external_test_lloihftr@sl.local
```

*Step 5 — cleanup proof (repository/config returned to canonical state):*

```text
--- 1) restore .env from backup ---
diff env.bak vs restored .env (empty => identical):
  [IDENTICAL - .env fully restored]
--- 2) stop capture sink (exact pid) ---
killed sink pid 7235
killed sink pid 7237
2525 closed (sink stopped)
--- 3) restart email_handler in ORIGINAL (canonical) config ---
new email_handler pid: 7422
20381 OPEN (handler back up)
--- 4) confirm NOT_SEND_EMAIL restored (config sees it) ---
19:NOT_SEND_EMAIL=true
```

**(c) Answer (OBSERVED)**

Of the three test headers, **none of the original values survive to the forwarded message**:

| Inbound header | Value sent | Fate in forwarded message | Why |
|----------------|-----------|---------------------------|-----|
| Custom `X-` header | `X-Custom-Header: q5-custom-VALUE-abc123` | **STRIPPED** (absent) | not on the keep-list, removed by `delete_all_headers_except` |
| `Received` | `Received: from evilrelay.example …` | **STRIPPED** (absent) | not on the keep-list, removed by `delete_all_headers_except` |
| `Reply-To` | `Reply-To: replyto-ORIG@external.test` | **original STRIPPED, then REWRITTEN** to `"replyto-ORIG at external.test" <replyto-orig_at_external_test_lloihftr@sl.local>` | stripped with everything else, then re-added pointing at the **reverse-alias** contact |

Details drawn directly from the capture:

- The forward path builds an explicit **whitelist** (`headers_to_keep`) and calls
  `delete_all_headers_except(msg, headers_to_keep)` — every header **not** on the keep-list is
  deleted. Neither `X-Custom-Header` nor `Received` is on the keep-list, so both are gone from the
  forwarded message (verified `count=0` in the header block). The original `Received` token
  `evilrelay.example` is likewise absent.
- **`Reply-To` is a special case, not a survivor.** It is not on the keep-list, so it is stripped
  by `delete_all_headers_except` too — this is proven by the handler log line `Reply-To header,
  new:… old:None` (the `old:None` shows `Reply-To` had *already been removed* by the time the
  rewrite runs). The handler then **re-adds** a brand-new `Reply-To` whose value is the
  **reverse-alias** address of the reply-to contact (`replyto-orig_at_external_test_lloihftr@sl.local`),
  created because the inbound `Reply-To` differed from the alias. So the header name reappears, but
  the **original value does not survive** (verified `count=0` for `replyto-ORIG@external.test`).
- For completeness (OBSERVED): the `From` header is likewise rewritten to a reverse-alias
  (`sender-outside_at_external_test_uhcmcgq@sl.local`), and SimpleLogin **adds** its own control
  headers to the forwarded message — `X-SimpleLogin-Type: Forward`, `X-SimpleLogin-EmailLog-ID`,
  `X-SimpleLogin-Envelope-From`, `X-SimpleLogin-Original-From`, `X-SimpleLogin-Envelope-To`. The
  surviving *pass-through* headers are the keep-listed ones present on the input: `Date`,
  `Subject`, `Message-Id`, `To`, and `Content-Transfer-Encoding`.

**(d) `file:line` references**

- Whitelist + strip: `email_handler.py:L793` builds `headers_to_keep` (`headers.FROM`, `TO`, `CC`,
  `SUBJECT`, `DATE`, `MESSAGE_ID`, `REFERENCES`, `IN_REPLY_TO`, `SL_QUEUE_ID`, `LIST_UNSUBSCRIBE`,
  `LIST_UNSUBSCRIBE_POST` + `MIME_HEADERS`); `email_handler.py:L810` calls
  `delete_all_headers_except(msg, headers_to_keep)`. The helper itself is
  `app/email_utils.py:L536` (`delete_all_headers_except`).
- `Received`/`Reply-To`/`X-` are absent from the keep-list — header-name constants:
  `app/email/headers.py:L14` (`RECEIVED = "Received"`), `app/email/headers.py:L13`
  (`REPLY_TO = "Reply-To"`); `MIME_HEADERS` (the only `Content-*`/`MIME-*` kept) at
  `app/email/headers.py:L44-L51`. There is no `X-Custom-Header` constant anywhere, so it can never
  be on the keep-list.
- `From` rewrite to reverse-alias: `email_handler.py:L866` —
  `add_or_replace_header(msg, "From", contact.new_addr())` (`add_or_replace_header` at
  `app/email_utils.py:L505`).
- `Reply-To` re-add (only when a reply-to contact exists): `email_handler.py:L869-L873`
  (`add_or_replace_header(msg, "Reply-To", reply_to_contact.new_addr())`); the reply-to contact is
  created at `email_handler.py:L594` (`get_or_create_reply_to_contact`) when the inbound
  `Reply-To` differs from the alias.
- Forward entry point: `email_handler.py:L679` (`forward_email_to_mailbox`).
## Q6 — Alias-creation token expiry window

**Question:** Experimentally verify the signed-suffix expiry window: test whether a token works
immediately, then wait and test if it still works, until you find the boundary where it stops
working.

**Approach (magnitude/timing per the run-first rule).** A real API key (created canonically via
`POST /api/api_key`, name `q6-key-v2`, code redacted — tail `…fxojrk`) is used to fetch a real
signed alias suffix from `GET /api/v5/alias/options`, then submit it to `POST
/api/v2/alias/custom/new` at three ages: **immediately (age ≈ 0 s)**, **just under the hypothesised
600 s window (age ≈ 595 s)**, and **just over it (age ≈ 607 s)**. The whole cycle is run **twice**
(staggered, in parallel) to confirm the boundary is stable. Crucially, the over-boundary submission
uses a **distinct alias prefix**, so a failure there can only be token expiry (`412`) and not a
duplicate-alias collision (`409`).

**(a) Commands** *(no placeholders — `$KEY` is the created key's code, `$ss` is a real
`signed_suffix`; the full script is `/tmp/inv/q6_run.sh`, removed at teardown)*

```bash
KEY=$(cat /tmp/inv/q6_key.txt)                 # code of the canonically-created key q6-key-v2
BASE=http://localhost:7777

# fetch a real signed suffix from the options endpoint
fetch_suffix() {
  curl -s -H "Authentication: $KEY" "$BASE/api/v5/alias/options" \
    | /app/venv/bin/python -c 'import sys,json;print(json.load(sys.stdin)["suffixes"][0]["signed_suffix"])'
}

# submit prefix+signed_suffix to the real alias-creation endpoint; print "HTTP=<code> BODY=<body>"
create_alias() {   # args: prefix signed_suffix
  curl -s -w '\n__HTTP__%{http_code}' -H "Authentication: $KEY" -H "Content-Type: application/json" \
       -X POST "$BASE/api/v2/alias/custom/new" \
       --data "{\"alias_prefix\":\"$1\",\"signed_suffix\":\"$2\"}"
}

# one cycle: immediate (age~0), then age one suffix across the 600s line at ~595s and ~607s
ss_imm=$(fetch_suffix); t=$(date +%s); create_alias "q6imm$t" "$ss_imm"      # expect 201
ss_b=$(fetch_suffix);   t0=$(date +%s)                                       # boundary suffix
#   wait until (now - t0) >= 595 ...
create_alias "q6under$(date +%s)" "$ss_b"                                    # expect 201
#   wait until (now - t0) >= 607 ...  (DISTINCT prefix -> rules out 409 duplicate)
create_alias "q6over$(date +%s)" "$ss_b"                                     # expect 412 expired
```

**Scale/stability used:** two full cycles, run in parallel with a 3 s stagger; each cycle brackets
the boundary with one under-window and one over-window submission of the **same** aged suffix.

**(b) Raw output** *(verbatim `/tmp/inv/q6_run1.log` and `/tmp/inv/q6_run2.log`; complete response
bodies, real wall-clock + epoch timestamps)*

```text
===== RUN 1 =====
[18:09:02 epoch=1783966142] ===== Q6 RUN 1 key-tail=...fxojrk window_hypothesis=600s =====
[18:09:02 epoch=1783966142] RUN1 IMMEDIATE age=0.0s -> HTTP=201 BODY={"alias":"q6r1imm1783966142@old.com","creation_date":"2026-07-13 18:09:02+00:00","creation_timestamp":1783966142,"disable_pgp":false,"email":"q6r1imm1783966142@old.com","enabled":true,"id":19,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
[18:09:02 epoch=1783966142] RUN1 fetched boundary suffix at t0=1783966142; will submit UNDER(~595s) then OVER(~607s)
[18:18:57 epoch=1783966737] RUN1 UNDER(pre-600) age=595.0s -> HTTP=201 BODY={"alias":"q6r1under1783966737@old.com","creation_date":"2026-07-13 18:18:57+00:00","creation_timestamp":1783966737,"disable_pgp":false,"email":"q6r1under1783966737@old.com","enabled":true,"id":21,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
[18:19:09 epoch=1783966749] RUN1 OVER(post-600) age=607.0s -> HTTP=412 BODY={"error":"Alias creation time is expired, please retry"}
[18:19:09 epoch=1783966749] RUN1 DONE

===== RUN 2 =====
[18:09:05 epoch=1783966145] ===== Q6 RUN 2 key-tail=...fxojrk window_hypothesis=600s =====
[18:09:05 epoch=1783966145] RUN2 IMMEDIATE age=0.0s -> HTTP=201 BODY={"alias":"q6r2imm1783966145@old.com","creation_date":"2026-07-13 18:09:05+00:00","creation_timestamp":1783966145,"disable_pgp":false,"email":"q6r2imm1783966145@old.com","enabled":true,"id":20,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
[18:09:05 epoch=1783966145] RUN2 fetched boundary suffix at t0=1783966145; will submit UNDER(~595s) then OVER(~607s)
[18:19:00 epoch=1783966740] RUN2 UNDER(pre-600) age=595.0s -> HTTP=201 BODY={"alias":"q6r2under1783966740@old.com","creation_date":"2026-07-13 18:19:00+00:00","creation_timestamp":1783966740,"disable_pgp":false,"email":"q6r2under1783966740@old.com","enabled":true,"id":22,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
[18:19:12 epoch=1783966752] RUN2 OVER(post-600) age=607.0s -> HTTP=412 BODY={"error":"Alias creation time is expired, please retry"}
[18:19:12 epoch=1783966752] RUN2 DONE
```

**(c) Answer (OBSERVED, with the source constant INFERRED/corroborated)**

- The token **works immediately** and **still works at age ≈ 595 s** (both runs: `HTTP 201`, alias
  created — ids 19/21 for run 1, ids 20/22 for run 2), and **stops working at age ≈ 607 s** (both
  runs: `HTTP 412` with body `{"error":"Alias creation time is expired, please retry"}`).
- **OBSERVED boundary:** the expiry line lies in the interval **(595 s, 607 s]** — the last
  observed success is at 595 s and the first observed failure is at 607 s. This is reproduced
  **identically in both runs**, so the boundary is stable.
- Because the over-boundary request used a **distinct alias prefix**, the `412` cannot be a
  duplicate-alias error (`409`); it is specifically the **expiry** response, confirming the failure
  is the token ageing out rather than a uniqueness collision.
- **INFERRED (corroborated by source):** the exact window is **600 s**. The observed bracket
  `(595 s, 607 s]` straddles 600, and the source validates the suffix with
  `signer.unsign(signed_suffix, max_age=600)` (`app/alias_suffix.py:L40`). The runtime evidence
  brackets the boundary; the precise `600` comes from source and is labelled inferred accordingly.

**(d) `file:line` references**

- Expiry enforcement: `app/alias_suffix.py:L40` — `signer.unsign(signed_suffix, max_age=600)`,
  inside `check_suffix_signature()` (`app/alias_suffix.py:L37`); the signer is
  `TimestampSigner(config.CUSTOM_ALIAS_SECRET)` at `app/alias_suffix.py:L11`.
- Secret derivation: `app/config.py:L201` — `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"`.
- Endpoint behaviour: `app/api/views/new_custom_alias.py:L70` calls `check_suffix_signature(...)`;
  on `SignatureExpired` it returns `412` at `app/api/views/new_custom_alias.py:L73`
  (`"Alias creation time is expired, please retry"`); the success path returns `201` at
  `app/api/views/new_custom_alias.py:L110-L111`.
## Q7 — API-key usage statistics

**Question:** When you make several API calls with the same key, what fields get updated in
the database and what are the actual values you observe after making the calls?

**(a) Commands** *(the key is created through the real canonical entry point `POST /api/api_key`;
DB rows are read with `psql` in expanded (`-x`) mode so every column is visible; the full
runnable script is `/tmp/inv/q7_run.sh`, removed at teardown)*

```bash
JAR=/tmp/inv/auth_jar.txt
BASE=http://localhost:7777
PSQL="env PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin"

# 1) create a FRESH key through the real endpoint (session cookie, no key yet)
curl -sS -i -b "$JAR" -H 'Content-Type: application/json' \
     -X POST "$BASE/api/api_key" -d '{"device":"q7-key-v2"}'
CODE=$(…extract "api_key" from the JSON body…)          # redacted in this doc
ID=$($PSQL -tAc "SELECT id FROM api_key WHERE name='q7-key-v2' ORDER BY id DESC LIMIT 1;")

# full-row reader used for BEFORE/AFTER (all 8 relevant columns)
dump_row() { $PSQL -x -c "SELECT id, name, times, last_used, updated_at, created_at, \
                                 user_id, sudo_mode_at FROM api_key WHERE id=$ID;"; }

# 2) RUN 1 — BEFORE, then N=5 key-authenticated calls, then AFTER
dump_row
for i in 1 2 3 4 5; do
  curl -sS -o /dev/null -w '%{http_code}\n' -H "Authentication: $CODE" "$BASE/api/user_info"
done
dump_row

# 3) RUN 2 — same key, N=5 more calls (second stable run)
dump_row
for i in 1 2 3 4 5; do
  curl -sS -o /dev/null -w '%{http_code}\n' -H "Authentication: $CODE" "$BASE/api/user_info"
done
dump_row

# 4) CONTROL — an unused key stays NULL
$PSQL -x -c "SELECT id,name,times,last_used,updated_at FROM api_key WHERE id=2;"
```

**Scale used:** N = 5 key-authenticated `GET /api/user_info` calls per run, over **two** runs on
the **same** key (10 calls total), confirming the counter behaviour is stable across runs.

**(b) Raw output** *(verbatim; API-key `code` and the session cookie HMAC signature redacted)*

```text
===== STEP 1: CREATE API KEY canonically (POST /api/api_key, session cookie) =====
--- COMMAND ---
curl -sS -i -b $JAR -H 'Content-Type: application/json' -X POST http://localhost:7777/api/api_key -d '{"device":"q7-key-v2"}'
--- RESPONSE (status line + headers + body; Set-Cookie HMAC redacted) ---
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:23:19 GMT
Connection: close
Content-Type: application/json
Content-Length: 75
Access-Control-Allow-Origin: *
Set-Cookie: slapp=f6b7156a-d101-481d-a8e8-103d26430534.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:23:19 GMT; HttpOnly; Path=/; SameSite=Lax

{"api_key":"<REDACTED-60-char-key-code>"}

--- provenance (key code REDACTED, showing last 6 chars only) ---
key name = q7-key-v2 ; db id = 7 ; code = ...<REDACTED>...xnztjj

===== STEP 2: RUN 1 (N=5 key-authenticated calls) =====
--- BEFORE RUN 1 (fresh key) ---
COMMAND: psql ... SELECT id,name,times,last_used,updated_at,created_at,user_id,sudo_mode_at FROM api_key WHERE id=7;
-[ RECORD 1 ]+---------------------------
id           | 7
name         | q7-key-v2
times        | 0
last_used    |
updated_at   |
created_at   | 2026-07-13 18:23:19.517093
user_id      | 1
sudo_mode_at |

--- N=5 GET /api/user_info with header 'Authentication: <key>' ---
COMMAND (x5): curl -sS -o /dev/null -w '%{http_code}' -H "Authentication: $CODE" http://localhost:7777/api/user_info
  call 1 -> HTTP 200
  call 2 -> HTTP 200
  call 3 -> HTTP 200
  call 4 -> HTTP 200
  call 5 -> HTTP 200
--- AFTER RUN 1 ---
-[ RECORD 1 ]+---------------------------
id           | 7
name         | q7-key-v2
times        | 5
last_used    | 2026-07-13 18:23:19.680111
updated_at   | 2026-07-13 18:23:19.680358
created_at   | 2026-07-13 18:23:19.517093
user_id      | 1
sudo_mode_at |


  (sleeping 3s so the RUN 2 timestamps are visibly distinct)

===== STEP 3: RUN 2 (N=5 more key-authenticated calls, SAME key) =====
--- BEFORE RUN 2 (== AFTER RUN 1) ---
-[ RECORD 1 ]+---------------------------
id           | 7
name         | q7-key-v2
times        | 5
last_used    | 2026-07-13 18:23:19.680111
updated_at   | 2026-07-13 18:23:19.680358
created_at   | 2026-07-13 18:23:19.517093
user_id      | 1
sudo_mode_at |

--- N=5 GET /api/user_info with header 'Authentication: <key>' ---
  call 1 -> HTTP 200
  call 2 -> HTTP 200
  call 3 -> HTTP 200
  call 4 -> HTTP 200
  call 5 -> HTTP 200
--- AFTER RUN 2 ---
-[ RECORD 1 ]+---------------------------
id           | 7
name         | q7-key-v2
times        | 10
last_used    | 2026-07-13 18:23:22.843411
updated_at   | 2026-07-13 18:23:22.843679
created_at   | 2026-07-13 18:23:19.517093
user_id      | 1
sudo_mode_at |


===== STEP 4: CONTROL — an UNUSED key (id=2 Firefox) still shows NULL last_used/updated_at =====
COMMAND: psql ... SELECT id,name,times,last_used,updated_at FROM api_key WHERE id=2;
-[ RECORD 1 ]-------
id         | 2
name       | Firefox
times      | 0
last_used  |
updated_at |
```

**(c) Answer (OBSERVED)**

Making key-authenticated calls updates **three** columns on the `api_key` row — not two. The
observed values on key `id=7` (name `q7-key-v2`, freshly created for this test) were:

| Column | BEFORE run 1 | AFTER run 1 (5 calls) | AFTER run 2 (5 calls) | Changed? |
|--------|--------------|-----------------------|-----------------------|----------|
| `times` | `0` | `5` | `10` | **YES** — `+1` per call (`+5` per run) |
| `last_used` | `NULL` | `2026-07-13 18:23:19.680111` | `2026-07-13 18:23:22.843411` | **YES** — set on every call |
| `updated_at` | `NULL` | `2026-07-13 18:23:19.680358` | `2026-07-13 18:23:22.843679` | **YES** — set on every UPDATE |
| `id` | `7` | `7` | `7` | no |
| `name` | `q7-key-v2` | `q7-key-v2` | `q7-key-v2` | no |
| `created_at` | `2026-07-13 18:23:19.517093` | *(unchanged)* | *(unchanged)* | no |
| `user_id` | `1` | `1` | `1` | no |
| `sudo_mode_at` | `NULL` | `NULL` | `NULL` | no |

Key facts drawn directly from the capture:

- **`times`** is an integer counter incremented by exactly one per key-authenticated request:
  `0 → 5 → 10` across the two runs of five calls each. Both runs show the identical `+5` delta,
  demonstrating the behaviour is stable across runs.
- **`last_used`** transitions from `NULL` (never-used key) to a timestamp, and is **rewritten on
  every call** — the run-2 value (`…18:23:22.843411`) is later than the run-1 value
  (`…18:23:19.680111`), reflecting the most recent call, not the first.
- **`updated_at`** is the third, easily-overlooked field. It is inherited from `ModelMixin`
  (`onupdate=arrow.utcnow`) and therefore fires automatically on **any** row UPDATE, including the
  `times`/`last_used` write. It moves `NULL → …680358 → …843679` in lock-step with the calls.
- **Observed micro-detail:** within the same request, `updated_at` lands a few hundred
  microseconds **after** `last_used` (`…680358` vs `…680111` in run 1; `…843679` vs `…843411`
  in run 2). This is because they come from two distinct clock reads — `last_used = arrow.now()`
  is assigned in the request handler while `updated_at = arrow.utcnow()` is evaluated by
  SQLAlchemy's `onupdate` hook at flush/commit time — so `updated_at` is always marginally later.
- **Unchanged fields:** `id`, `name`, `created_at`, `user_id`, and `sudo_mode_at` never change.
  In particular `created_at` (`2026-07-13 18:23:19.517093`) is identical before and after all ten
  calls, cleanly separating "row created" from "row updated".
- **Control:** the never-used key `id=2` (`Firefox`) still shows `times=0`, `last_used=NULL`,
  `updated_at=NULL`, confirming these two timestamps are `NULL` until the first update rather than
  being populated at creation.

**(d) `file:line` references**

- Per-call update logic (executed inside `authorize_request()` on the key-authenticated path):
  `app/api/base.py:L30` — `api_key.last_used = arrow.now()`; `app/api/base.py:L31` —
  `api_key.times += 1`. These run for every request that presents a valid `Authentication` key
  header.
- `ApiKey` model columns: `app/models.py:L2359` — `times` (`Integer`, default `0`);
  `app/models.py:L2358` — `last_used` (`ArrowType`); `app/models.py:L2360` — `sudo_mode_at`.
- **`updated_at` provenance (the third field):** `ApiKey(Base, ModelMixin)` at
  `app/models.py:L2350` inherits `updated_at = sa.Column(ArrowType, default=None,
  onupdate=arrow.utcnow)` at `app/models.py:L65` (with `created_at` at `app/models.py:L64`).
  Because `onupdate` fires on every UPDATE, the `times`/`last_used` write also refreshes
  `updated_at` — this is why three fields change, not two. **OBSERVED** (the captured rows) and
  corroborated by source.
- Canonical key creation: `create_api_key()` at `app/api/views/user_info.py:L109-L128`
  (`ApiKey.create(...)` then `return jsonify(api_key=api_key.code), 201`) — the `HTTP 201` and
  `{"api_key": …}` body above come from this route.
## Q8 — Failed login (wrong credentials)

**Question:** When a login attempt fails with wrong credentials, what specific log messages
appear and what HTTP response details come back? (Actual logged output and response, not what
the code says should happen.)

**(a) Commands** *(the failed request is correlated to its exact log line by snapshotting the
`gunicorn.log` line count immediately before and after the POST; the full script is
`/tmp/inv/q8_run.sh`, removed at teardown)*

```bash
BASE=http://localhost:7777
JAR=/tmp/inv/q8_jar.txt
LOG=/tmp/inv/gunicorn.log

# 1) fresh session + CSRF token
CSRF=$(curl -s -c "$JAR" "$BASE/auth/login" \
  | grep -o 'name="csrf_token"[^>]*value="[^"]*"' | grep -o 'value="[^"]*"' | cut -d'"' -f2)

# 2) snapshot the log length, POST wrong credentials, snapshot again -> the delta is THIS request
BEFORE=$(wc -l < "$LOG")
curl -sS -i -c "$JAR" -b "$JAR" \
     --data-urlencode "email=john@wick.com" \
     --data-urlencode "password=deliberately-wrong-pw" \
     --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login"
AFTER=$(wc -l < "$LOG")
sed -n "$((BEFORE+1)),${AFTER}p" "$LOG"      # <-- the same-request log slice

# 3) rate-limiter: repeat failed POSTs in the same minute and record each status
for i in $(seq 1 13); do
  curl -s -o /dev/null -w '%{http_code}\n' -c "$JAR" -b "$JAR" \
       --data-urlencode "email=john@wick.com" --data-urlencode "password=wrong-$i" \
       --data-urlencode "csrf_token=$CSRF" "$BASE/auth/login"
done
```

**(b) Raw output** *(verbatim; session-cookie HMAC signatures redacted)*

```text
===== PART A: single correlated failed login =====
--- step 1: GET /auth/login -> fresh jar + csrf ---
csrf_token captured (length=91)

--- step 2: snapshot gunicorn.log line count BEFORE the POST ---
log lines before = 93

--- step 3: POST wrong credentials ---
--- FULL RESPONSE headers (status line + all headers; Set-Cookie HMAC redacted) ---
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:29:51 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 7017
Set-Cookie: slapp=e673e274-d0f2-48f7-8679-b069523619d2.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:29:51 GMT; HttpOnly; Path=/; SameSite=Lax

--- Content-Length header (proves full HTML re-render, not a redirect) ---
Content-Length: 7017
--- flash message text extracted from response body ---
Email or password incorrect

--- step 4: gunicorn.log slice emitted by THIS request ---
log lines after = 94 (delta 1)
2026-07-13 18:29:51,778 - SL - DEBUG - 5055 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 200, takes 0.24021482467651367

===== PART B: rate-limiter observation (source: '10/minute', deduct_when=failed) =====
Repeated failed POSTs in the same minute; recording HTTP status of each.
  failed attempt 1 -> HTTP 200
  failed attempt 2 -> HTTP 200
  failed attempt 3 -> HTTP 200
  failed attempt 4 -> HTTP 200
  failed attempt 5 -> HTTP 200
  failed attempt 6 -> HTTP 200
  failed attempt 7 -> HTTP 200
  failed attempt 8 -> HTTP 200
  failed attempt 9 -> HTTP 200
  failed attempt 10 -> HTTP 429
  failed attempt 11 -> HTTP 429
  failed attempt 12 -> HTTP 429
  failed attempt 13 -> HTTP 429

--- capture a limited (429) response in full (headers + body) ---
HTTP/1.1 429 TOO MANY REQUESTS
Server: gunicorn/20.0.4
Date: Mon, 13 Jul 2026 18:29:54 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 5622
Set-Cookie: slapp=e673e274-d0f2-48f7-8679-b069523619d2.<REDACTED-HMAC-sig>; Expires=Mon, 20-Jul-2026 18:29:54 GMT; HttpOnly; Path=/; SameSite=Lax

<!DOCTYPE html>
<html lang="en" ... >   (5622-byte themed SimpleLogin HTML error page; head shown)
```

**(c) Answer (OBSERVED)**

*HTTP response details:*

- The failed login returns **`HTTP/1.1 200 OK`** — **not** a `401`/`403`. The server re-renders
  the login form in place. This is confirmed by `Content-Type: text/html; charset=utf-8` and a
  **7017-byte** body (a full HTML page, not a redirect and not a JSON error).
- The response body contains the flash message with the **exact text `Email or password
  incorrect`** (extracted verbatim from the body above).
- A fresh anonymous session cookie is set (`slapp=e673e274-…`, HMAC redacted); the user is **not**
  logged in.

*Log messages (correlated to this exact request via the before/after line-count delta = 1):*

- Exactly **one** log line is emitted for the failed login, and it comes from the generic
  request logger, **not** from the login handler:
  `… - SL - DEBUG - 5055 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login
  ImmutableMultiDict([]) 200, takes 0.24021482467651367`.
- **There is no dedicated "login failed" log line.** OBSERVED: the delta slice contains only the
  `after_request()` line. This matches the source — the failed branch
  (`app/auth/views/login.py:L45-L50`) contains **no** `LOG.*` call (the only `LOG.d` calls,
  `L30`/`L33`, are in the *already-authenticated* branch, which this logged-out request never
  enters).
- The logged `ImmutableMultiDict([])` is the empty **`request.args`** (query string), **not** the
  POST form — so the submitted password never appears in the log (`server.py:L284-L290` logs
  `request.args`, not `request.form`). OBSERVED and consistent with source.
- **New Relic event — INFERRED (source-derived), not observable in logs:** the failed branch calls
  `LoginEvent(LoginEvent.ActionType.failed).send()` (`login.py:L50`), and `send()` performs
  `newrelic.agent.record_custom_event("LoginEvent", {"action": "failed", "source": "web"})`
  (`app/events/auth_event.py:L22-L24`). This dispatches to the New Relic agent, **not** to
  stdout/stderr, so — as OBSERVED above — it produces **no** line in `gunicorn.log`. Its emission
  is therefore inferred from source, while its absence from the logs is observed.

*Rate-limiter effect (OBSERVED, bounded):*

- The route is decorated `@limiter.limit("10/minute", deduct_when=… g.deduct_limit)`
  (`login.py:L22-L23`), and the failed branch sets `g.deduct_limit = True` (`login.py:L47`), so
  **only failed attempts consume the budget**. Exercising this: after the single failed attempt in
  Part A plus nine more in Part B (ten failed attempts total in the minute), the **eleventh**
  failed attempt (Part B attempt 10) flips to **`HTTP 429 TOO MANY REQUESTS`**, and every
  subsequent attempt in the window stays `429`. The `429` response is itself a themed HTML page
  (`Content-Length: 5622`). This directly observes the documented `10/minute` limit: ten failures
  are allowed per minute, the eleventh is blocked.

**(d) `file:line` references**

- HTTP 200 re-render (not 401/403): `app/auth/views/login.py:L74` — `return
  render_template("auth/login.html", …)` is reached after the failed branch falls through.
- Exact flash text: `app/auth/views/login.py:L49` — `flash("Email or password incorrect",
  "error")`.
- Failed-credential branch (no `LOG.*` here): `app/auth/views/login.py:L45` — `if not user or not
  user.check_password(...)`; `L47` — `g.deduct_limit = True`.
- Request log line origin: `server.py:L284` — the `after_request()` `LOG.d(...)` that logs
  `remote_addr method path args status_code`, i.e. `request.args` (not the form body), defined at
  `server.py:L273-L290`.
- New Relic custom event (INFERRED / source-derived): `app/auth/views/login.py:L50` —
  `LoginEvent(LoginEvent.ActionType.failed).send()`; `app/events/auth_event.py:L22-L24` —
  `newrelic.agent.record_custom_event("LoginEvent", {...})`.
- Rate limit: `app/auth/views/login.py:L22-L23` — `@limiter.limit("10/minute",
  deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit)`.
## Coverage Confirmation

A final pass confirming every question **and every named sub-part** was addressed with observed
evidence. "Sub-parts" include each `e.g./such as/including` item, treated as required coverage.

| Q | Named sub-parts asked | Where answered | Observed result |
|---|-----------------------|----------------|-----------------|
| Q1 | (a) actual HTTP status code; (b) response body structure — for a browser session with **no** API-key header | Q1 (a)-(c) | `HTTP 200 OK` + JSON user-info body (`email`, `name`, `is_premium`, …); no-cookie contrast `401 {"error":"Wrong api key"}` |
| Q2 | (a) specific (non-standard) status code; (b) exact error message text — privileged op via browser session | Q2 (b)-(c) | Browser-session path → **`HTTP 500` `{"error":"Internal error"}`** (from a `NoneType.sudo_mode_at` dereference); the non-standard **`440` `{"error":"Need sudo"}`** is the key-without-sudo path — both captured; no account deleted (pre/post state proven) |
| Q3 | (a) raw bytes; (b) their format; (c) keys present in deserialized data; (d) how the session key is structured | Q3 (b)-(c) | raw **pickle protocol-4** bytes (`\x80\x04\x95…`); 6 keys `_permanent, _fresh, csrf_token, _user_id, _id, sudo_time`; key = `session:<uuid4>`; TTL `604800` (7 d) authenticated |
| Q4 | (a) session id **before** login; (b) **after** login; (c) same or replaced; (d) actual before/after values | Q4 (b)-(c) | **SAME** (byte-identical cookie), two runs (`96f4…` and `576870dd…`); identifier **not** rotated at login |
| Q5 | fate of (a) custom `X-` header; (b) `Received` header; (c) `Reply-To` header; (d) send an actual email and examine | Q5 (b)-(c) | `X-Custom-Header` **stripped**; `Received` **stripped**; `Reply-To` original **stripped then rewritten** to a reverse-alias — real `swaks` email sent, forwarded message captured |
| Q6 | (a) works immediately; (b) wait and re-test; (c) find the boundary where it stops | Q6 (b)-(c) | `201` at age 0 s and 595 s; `412 "Alias creation time is expired…"` at 607 s; boundary **(595 s, 607 s]** (two stable runs); source window `max_age=600` |
| Q7 | (a) which fields get updated; (b) the actual values observed after the calls | Q7 (b)-(c) | **THREE** fields change: `times` (`0→5→10`), `last_used` (`NULL→timestamp`), **`updated_at`** (`NULL→timestamp`); `created_at`/`id`/`name`/`user_id`/`sudo_mode_at` unchanged; two runs of N=5 |
| Q8 | (a) specific log messages; (b) HTTP response details | Q8 (b)-(c) | `HTTP 200` re-render + flash `Email or password incorrect`; a single `after_request()` log line (`server.py:284`); no dedicated failed-login log line; New Relic event INFERRED; rate limiter `10/minute` → `429` observed |

All eight questions and all listed sub-parts are covered with captured runtime evidence; observed
values are distinguished from inferred (source-derived) ones throughout.

---

## Security & Privacy Handling

This investigation exercised real authentication material against a running system. The following
policy governs what is shown versus redacted in this document, and how live material was
invalidated.

**Data classification**

1. **Local-dev public defaults — shown (needed for reproducibility).** These are published in the
   repository itself and are not secrets in this context: `FLASK_SECRET=secret`
   (`example.env:L77`), the DB credentials `myuser`/`mypassword` and `DB_URI`
   (`example.env:L75`), and the dev login `john@wick.com` / `password`
   (`CONTRIBUTING.md:L108-L109`). They are shown because the results cannot be reproduced without
   them and they carry no meaning outside this throwaway local container.
2. **Reusable runtime secrets — redacted.** Material that could authenticate a request if reused
   is redacted with explicit markers:
   - **Session-cookie HMAC signatures** — the `slapp` cookie is shown as
     `<uuid>.<REDACTED-HMAC-sig>`. The UUID (the session *identifier*) is shown because it **is**
     the evidence for Q3/Q4; only the HMAC signature (which authenticates the cookie) is masked.
   - **API-key `code` values** — never printed literally; created keys are read into a shell
     variable and only a short non-reconstructable tail (e.g. `…fxojrk`) is shown for provenance.
   - **Session-protection token values inside the pickle** — in Q3 the *raw pickle bytes are shown
     verbatim* (the user explicitly asked for them) except that the two reusable secret **values**
     — `csrf_token` (40 hex) and the Flask-Login `_id` (128 hex) — are masked as
     `<REDACTED-…>`. All structural bytes, key names, length prefixes, and pickle opcodes are
     preserved so the format is fully verifiable.

**Why identifiers are shown but signatures/keys are not.** UUID session identifiers and reverse-
alias addresses are needed to *explain* the behavior (fixation, key structure, header rewrite) and
are useless for authentication on their own; the HMAC signature, API-key code, and session-
protection token values are the parts that would grant access, so those are the parts removed.

**Invalidation at teardown.** All live material captured here was invalidated when the environment
was torn down (see **Cleanup Verification** below): the Redis session store was flushed
(`redis-cli FLUSHALL`), which destroys every `session:<uuid>` value shown in Q3/Q4 (the redacted
HMAC then signs nothing); and every API key created for the investigation (`q6-key-v2`,
`q7-key-v2`, `q2-contrast-key`) was deleted from the `api_key` table, invalidating those codes.
The container itself is ephemeral.

---

## Read-Only Compliance Proof

The rule set requires that **no existing file in the source repository is modified** and that the
only addition is this answer document. This was verified by capturing `git status` of the `/app`
checkout **before** the investigation and again **after** it (post-teardown), and showing they are
identical.

**Baseline — captured before any observation:**

```bash
git -C /app status --porcelain
```

```text
 M app/spamassassin_utils.py
 M local_data/dkim.key
 M local_data/jwtRS256.key
 M local_data/jwtRS256.key.pub
 M local_data/paddle.key.pub
 M local_data/private-pgp.asc
 M local_data/test_words.txt
 M static/package-lock.json
```

**Final — after all eight questions and teardown:**

```bash
git -C /app status --porcelain
git -C /app status --porcelain --untracked-files=all | grep '^??' || echo 'no untracked files'
```

```text
 M app/spamassassin_utils.py
 M local_data/dkim.key
 M local_data/jwtRS256.key
 M local_data/jwtRS256.key.pub
 M local_data/paddle.key.pub
 M local_data/private-pgp.asc
 M local_data/test_words.txt
 M static/package-lock.json
no untracked files
```

**Interpretation:**

- The two listings are **identical**: the same eight tracked files, and **no untracked files**.
  The investigation added, removed, and modified **nothing** in `/app`.
- Those eight `M` entries are **pre-existing, image-baked** modifications present at container
  start (build-time key/asset generation and a `spamassassin_utils.py` tweak baked into the
  image) — **not** produced by this work. They appear in the baseline, before a single observation
  was run.
- `.env` does **not** appear because it is git-ignored (`git -C /app check-ignore .env` → exit 0).
  It was edited only for Q5 and then restored to a byte-identical copy of its backup (`diff -u`
  empty), so even the ignored file is unchanged.
- All temporary observation scripts (`/tmp/inv/*.py`, `/tmp/inv/*.sh`) live **outside** the `/app`
  tree and were removed at teardown.
- The **only** file added to the repository is this document,
  `blitzy/documentation/app_2cd6ee777f8c.md`, in the separate deliverable checkout.

---

## Cleanup Verification

After all evidence was captured, the environment was torn down: investigation-created database
rows were deleted, the Redis store was flushed (invalidating every session and rate-limit key),
the web app and SMTP handler were stopped, and all temporary observation scripts under `/tmp/inv`
were removed. The `.env` file had already been restored during Q5.

**Teardown commands**

```bash
# 1) delete investigation-created DB rows (FK-safe order); keep only dummy-data keys Chrome/Firefox
psql ... -c "DELETE FROM email_log WHERE contact_id IN (SELECT id FROM contact WHERE website_email LIKE '%external.test');"
psql ... -c "DELETE FROM contact  WHERE website_email LIKE '%external.test';"
psql ... -c "DELETE FROM alias    WHERE email LIKE 'q6%';"
psql ... -c "DELETE FROM api_key  WHERE id >= 3;"
# 2) invalidate all runtime session / rate-limit material
redis-cli FLUSHALL
# 3) stop the canonical processes (by exact pid)
kill 5052 5054 5055 5056 7422
# 4) remove all temporary observation scripts/logs
rm -rf /tmp/inv
```

**Proof — before vs after**

| Item | Before teardown | After teardown |
|------|-----------------|----------------|
| `api_key` rows | 7 (Chrome, Firefox, + 5 investigation keys) | **2** (`Chrome`, `Firefox` — dummy-data only) |
| `alias` rows `LIKE 'q6%'` | 11 (ids 12–22) | **0** |
| `contact` rows `LIKE '%external.test'` | 2 (ids 6, 7) | **0** |
| `email_log` for those contacts | 1 (id 4) | **0** |
| Redis `DBSIZE` | 8 | **0** |
| `gunicorn` / `email_handler` processes | running (pids 5054/5055/5056, 7422) | **none running** |
| Port `7777` (web) | open | **closed** (connection refused) |
| Port `20381` (SMTP) | open | **closed** |
| `/tmp/inv` scripts/logs | 43 files | **removed** (`No such file or directory`) |

```text
# after teardown
$ redis-cli DBSIZE
0
$ ps -eo pid,cmd | grep -E 'gunicorn wsgi:app|email_handler.py' | grep -v grep || echo NONE
NONE
$ nc -z -w2 localhost 7777 && echo OPEN || echo CLOSED
CLOSED
$ nc -z -w2 localhost 20381 && echo OPEN || echo CLOSED
CLOSED
$ psql ... -c "SELECT id,name,times FROM api_key ORDER BY id;"
 1 | Chrome  | 1
 2 | Firefox | 0
$ test -e /tmp/inv && echo STILL EXISTS || echo REMOVED
REMOVED
```

This satisfies the read-only + cleanup requirement: the SimpleLogin repository is unmodified (see
**Read-Only Compliance Proof**), all temporary scripts are gone, and all captured session/key
material has been invalidated. The only lasting change anywhere is this single answer document.

