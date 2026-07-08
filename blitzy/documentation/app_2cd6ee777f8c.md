# How SimpleLogin Creates a New Email Alias — End‑to‑End, Runtime‑Verified

> **Scope of this document.** This is a read‑only investigation answer. It explains, *from observed runtime behavior* of the running SimpleLogin Flask application, exactly what happens when a user creates a new email alias: the request the frontend sends, the backend response, the database changes, the background/follow‑up work, and the failure behavior. Every factual claim is backed by either **raw captured output** (shown before any summary) or a **`file:line`** reference into the source. Statements that could only be established by reading (not running) are explicitly **labeled `[inferred]`**.
>
> All observations were made by driving the **real HTTP entry points** (the web form `POST /dashboard/` and the JSON API `POST /api/alias/random/new`) against a live `gunicorn` server. The HTTP‑level controls on those paths were therefore genuinely exercised: **CSRF validation, `flask_login`'s `@login_required`, the API‑key `Authentication` decorator, and the `flask‑limiter` `@limiter.limit(...)` decorator were all active**. Two further controls that decorate the creation core — the Redis‑backed token bucket inside `Alias.create` and the `@parallel_limiter.lock("alias_creation")` concurrency guard — were **present on the code path but ran as no‑ops** in this canonical dev configuration because `MEM_STORE_URI` is unset (proven at runtime in §7.5). No internal helper was called directly to fabricate a result. No repository source file was modified; the only persistent artifact is this document.
>
> **Redaction convention.** The *only* redacted values in captured **output** anywhere in this document are the opaque `slapp=` session‑cookie payloads, which are replaced by `<redacted …>` while preserving every cookie attribute (`Expires`, `HttpOnly`, `Path`, `SameSite`). All statuses, headers, log lines, JSON bodies, and CSRF tokens are shown **verbatim** as captured — the full 91‑character CSRF token appears in the §3 request capture and the §2.5 login flow. One further substitution appears only in **command *input*** (never in output): in the §5 reproduction command templates the per‑request `csrf_token` value — which is session‑bound and rotates on every request — is written as `<valid token>` for brevity; this is an input placeholder, not a redaction of captured output (every captured response in those blocks is verbatim). CSRF tokens and the local test API key `freecode` are runtime‑local fixtures (not production secrets) and are shown in full.

---

## 1. TL;DR — Direct Answer

**Both entry points converge on the same persistence core, so the alias‑creation database effects are identical; only the transport and the response differ.** The web form handler `dashboard.index` and the JSON API `new_random_alias` both call `Alias.create_new_random` (`app/models.py:L1721`) → `Alias.create` (`app/models.py:L1628-L1692`).

- **Frontend request (O4).** The dashboard "Random Alias" button submits an **`application/x-www-form-urlencoded` `POST` to `/dashboard/`** carrying two fields: `form-name=create-random-email` and `csrf_token=<…>` (plus an *optional* `generator_scheme` when the "By Random Words" / "By UUID" dropdown items are used). Evidence: `templates/dashboard/index.html:L50-L58` (main button), `L68-L86` (dropdown variants); captured request in §3.

- **Backend response (O5).**
  - **Web → HTTP `302 FOUND`**, `Location: http://localhost:7777/dashboard/?highlight_alias_id=<id>&query=&sort=&filter=` (`app/dashboard/views/index.py:L113-L121`).
  - **API → HTTP `201 CREATED`** with a JSON body of **17 keys** = the 16 keys from `serialize_alias_info_v2` (`app/api/serializer.py:L55-L93`) plus a top‑level `alias` string (`app/api/views/new_random_alias.py:L114-L116`).

- **Database changes (O6).** In the default, seeded, single‑mailbox configuration the **alias‑creation core touches exactly THREE tables**, confirmed stable across repeated runs:
  1. `alias` — one **INSERT** (`app/models.py:L1660`).
  2. `daily_metric` — the day's row is **UPDATE**d (`nb_alias += 1`) on every creation after the first of the calendar day, or **INSERT**ed on the first creation of a new day (`app/models.py:L1661`, `DailyMetric.get_or_create_today_metric` `app/models.py:L3280`).
  3. `alias_audit_log` — one **INSERT** with `action='create'`, `message='New alias created'` (`app/models.py:L1688-L1690`, `app/alias_audit_log_utils.py:L18-L32`).
  There are **no writes** to `sync_event`, `alias_mailbox`, `users`, or `alias_used_on` in this default flow (all observed at zero‑delta; §5). **API‑transport caveat:** the API path *additionally* updates the `api_key` row's usage counters (`times`, `last_used`) as an authentication side effect **before** the alias‑creation core runs (`app/api/base.py:L30-L32`); this is auth bookkeeping on a fourth table that is **not** part of the creation core and **never** occurs on the cookie‑authenticated web path (§5.4). So the precise direct answer is: *the 3‑table creation core is identical for both flows; the API transport touches one extra table (`api_key`) for auth accounting.*

- **Background tasks / follow‑up work (O7).** **None** in default configuration. Alias creation emits exactly one creation log line (`app/dashboard/views/index.py:L110`, web path only) and then event dispatch **short‑circuits** on the *second* early return in `EventDispatcher.send_event` because `EVENT_WEBHOOK` is unset — logging `Not sending events because webhook is not configured and allowed to be empty` (`app/events/event_dispatcher.py:L61-L65`). Consequently **no `sync_event` row is written and no `NOTIFY` is issued**, so `event_listener.py` and `job_runner.py` do no alias‑creation follow‑up — confirmed by running both workers and by `job=0` / `sync_event=0` around every creation (§6).

- **Failure behavior (O8).** Each failure surfaces distinctly and, on refusal, writes **nothing** to the alias‑creation tables (all shown with before/after DB probes in §7):
  - **Free‑plan cap** (`MAX_NB_EMAIL_FREE_PLAN=5`): API → **`400`** JSON error; web → **`302`** + flash *"You need to upgrade your plan to create new alias."*
  - **Trashed‑alias reuse**: `Alias.create` raises **`AliasInTrashError`** (`app/models.py:L1647-L1652`); on the random endpoint's hostname branch it is **caught** and the endpoint falls back to a random alias (still `201`); on the custom‑alias endpoint a trashed email is pre‑checked and returns **`409`**.
  - **Invalid CSRF** (web): flash *"Invalid request"* + **`302`** back to `request.url`.
  - **Invalid API `mode`**: **`400`** `{"error":"<mode> must be either word or uuid"}`.
  - **Rate limiting**: **`429`** `{"error":"Rate limit exceeded"}` once the `flask‑limiter` bucket (`ALIAS_LIMIT="100/day;50/hour;5/minute"`) is exceeded — observed admitting **10** requests before refusing (2 gunicorn workers × the per‑minute `5`, each worker holding its own in‑memory bucket; §7.5).
  - **Network**: no genuine outbound socket dependency exists on the default alias‑creation commit path, so no true network‑I/O failure was exercised; this is stated and evidenced as *not run* in §7.5, not asserted.

> **Critical nuance carried throughout.** The seeded user `john@wick.com` is **premium** (active subscription), so the free‑plan cap does *not* apply to him; the cap was therefore reproduced with a **separately created, runtime‑only free user** (`freebie@sl.local`), clearly labeled wherever used (§7.1, §2.4).

---

## 2. Environment & Methodology

### 2.1 Platform, canonical setup, and how the app was run

The application was built and run inside the attached Docker container, which ships the **canonical dev runtime** (Python 3.10, the exact `poetry.lock` pins in a prebuilt `venv`, and the prebuilt frontend assets). The canonical local procedure is defined in `CONTRIBUTING.md` (`poetry sync` at `L31`; then the run-locally steps at `L77-L107`: `cd static && npm install`; `cp example.env .env` + set `DB_URI`; start `postgres:13`; `alembic upgrade head && flask dummy-data && python3 server.py`). The two dependency‑installation steps (`poetry sync`, `npm install`) were satisfied **offline** by the image's prebuilt `venv` and `static/node_modules`; I therefore **evidence their equivalent result** (exact versions/pins present) and then **ran the schema, seed, and serve steps live**, capturing raw output for each.

**(1) `poetry sync` equivalent — Python runtime and installed pins.** The prebuilt `venv` contains the `poetry.lock` dependency set:

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python --version && venv/bin/pip list | wc -l'
Python 3.10.18
183

$ docker exec sl-app bash -lc 'cd /app && venv/bin/pip list | grep -iE "^(Flask|SQLAlchemy|alembic|gunicorn|psycopg2-binary|redis|Flask-Login|Flask-WTF|arrow|bcrypt) "'
alembic                       1.4.3
arrow                         0.16.0
bcrypt                        3.2.0
Flask                         1.1.2
Flask-Login                   0.5.0
Flask-WTF                     0.14.3
gunicorn                      20.0.4
psycopg2-binary               2.9.3
redis                         4.6.0
SQLAlchemy                    1.3.24
```

`pip list` prints 183 lines (a 2‑line header + **181 installed packages**), matching the `poetry.lock` pinset; the key runtime pins above (`Flask 1.1.2`, `SQLAlchemy 1.3.24`, `gunicorn 20.0.4`) are exactly those in `pyproject.toml`.

**(2) `npm install` equivalent — frontend assets.** The prebuilt `static/node_modules` is present:

```
$ docker exec sl-app bash -lc 'node --version; npm --version; ls /app/static/node_modules | wc -l; ls /app/static/node_modules | tr "\n" " "'
v18.19.0
9.2.0
12
@sentry bootbox font-awesome htmx.org intro.js jquery multiple-select parsleyjs qrious toastr tslib vue
```

The 12 top‑level packages include the frontend libraries the dashboard uses (`vue`, `htmx.org`, `parsleyjs`, `toastr`, `bootbox`, `intro.js`), matching `static/package.json`.

**(3) `.env` / `DB_URI` (the `cp example.env .env` step).** The canonical dev config:

```
$ docker exec sl-app bash -lc 'cd /app && grep -E "^(URL|DB_URI|EMAIL_DOMAIN|NOT_SEND_EMAIL|DISABLE_ONBOARDING)=" .env'
URL=http://localhost:7777
NOT_SEND_EMAIL=true
EMAIL_DOMAIN=sl.local
DB_URI=postgresql://myuser:mypassword@sl-db:5432/simplelogin
DISABLE_ONBOARDING=true

$ docker exec sl-app bash -lc 'cd /app && grep -cE "^MEM_STORE_URI=|^EVENT_WEBHOOK=" .env'
0
```

`MEM_STORE_URI` and `EVENT_WEBHOOK` have **no uncommented assignment** (count `0`), i.e. both are unset — the canonical dev default that makes the Redis limiters no‑ops (§7.5) and short‑circuits event dispatch (§6).

**(4) PostgreSQL 13 startup + DB‑port reconciliation.** Postgres runs in container `sl-db`; the app reaches it over the Docker‑network alias `sl-db:5432` (the port in `DB_URI`), while the host publishes it on `55432` (host‑side only). I ran all SQL through the container, so the effective endpoint was `sl-db:5432`:

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc 'select version();'
PostgreSQL 13.23 (Debian 13.23-1.pgdg13+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 14.2.0-19) 14.2.0, 64-bit
```

Redis (container `sl-redis`) is reachable but, per `.env`, is not wired to the limiters:

```
$ docker exec sl-app bash -lc 'redis-cli -h sl-redis ping'
PONG
```

**(5) `alembic upgrade head` — schema migration (run live).** The full log was captured to `/tmp/scratch_qna/o1_alembic.log`. First lines:

```
$ docker exec sl-app bash -lc 'cd /app && alembic upgrade head 2>&1 | tee /tmp/scratch_qna/o1_alembic.log' ; head -12 /tmp/scratch_qna/o1_alembic.log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/kyeksiotgwkwlgipdtpv
Upload files to local dir
>>> init logging <<<
2026-07-08 05:38:20,439 - SL - DEBUG - 2103 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 5e549314e1e2, empty message
INFO  [alembic.runtime.migration] Running upgrade 5e549314e1e2 -> 3cd10cfce8c3, empty message
INFO  [alembic.runtime.migration] Running upgrade 3cd10cfce8c3 -> 0256244cd7c8, empty message
```

Last lines (the head revision the schema ends at) and the exact step count (complete slices via `tail`/`grep -c`, so nothing is elided inside a block):

```
$ tail -3 /tmp/scratch_qna/o1_alembic.log
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at

$ grep -c 'Running upgrade' /tmp/scratch_qna/o1_alembic.log
255
```

The migration applied **255 upgrade steps**, from the empty database up to head revision **`32f25cbf12f6`** (`alias_audit_log_index_created_at`) — note the `alias_audit_log` table used later (§5) is created by these migrations.

**(6) `flask dummy-data` — seeding (run live).** Full captured output (`/tmp/scratch_qna/o1_seed.log`, shown in its entirety):

```
$ docker exec sl-app bash -lc 'cd /app && FLASK_APP=wsgi.py flask dummy-data 2>&1 | tee /tmp/scratch_qna/o1_seed.log' ; cat /tmp/scratch_qna/o1_seed.log
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/lkejcseyvzebwkigxjuw
Upload files to local dir
>>> init logging <<<
2026-07-08 05:38:33,260 - SL - DEBUG - 2117 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 05:38:34,329 - SL - WARNING - 2117 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-08 05:38:34,350 - SL - DEBUG - 2117 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-08 05:38:34,642 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:34,645 - SL - DEBUG - 2117 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-08 05:38:34,667 - SL - DEBUG - 2117 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email writes_magnon871@sl.local
2026-07-08 05:38:34,677 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:34,785 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:34,795 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:34,816 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:34,855 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:34,912 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:34,960 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:34,971 - SL - DEBUG - 2117 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-ipzidxyivd
2026-07-08 05:38:34,978 - SL - DEBUG - 2117 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-cxeioulzzx
2026-07-08 05:38:35,253 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:35,260 - SL - DEBUG - 2117 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-08 05:38:35,288 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:35,301 - SL - INFO - 2117 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:38:35,312 - SL - INFO - 2117 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

Note the seed itself already exercises the event short‑circuit (`event_dispatcher.py:62`) and the onboarding‑disable path (`models.py:647` — this is why no `job` rows are seeded, §6). The seed's own random alias (`writes_magnon871@sl.local`) becomes alias id 2 (§5, §2.6 baseline).

**(7) Serve with `gunicorn` (run live) + why not `python3 server.py`.** `server.py:L588` binds `127.0.0.1` only (`app.run(debug=True, port=7777)` — the verbatim two‑argument call, no omitted arguments), which is not reachable from the host; the container therefore serves host‑accessible via `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2`. The Flask app object is the same factory `create_app` (`server.py:L139`) imported by `wsgi.py` (`wsgi.py:L1-L3`). Captured startup:

```
$ head -5 /tmp/scratch_qna/gunicorn_full.log
[2026-07-08 05:38:52 +0000] [2158] [INFO] Starting gunicorn 20.0.4
[2026-07-08 05:38:52 +0000] [2158] [INFO] Listening at: http://0.0.0.0:7777 (2158)
[2026-07-08 05:38:52 +0000] [2158] [INFO] Using worker: sync
[2026-07-08 05:38:52 +0000] [2159] [INFO] Booting worker with pid: 2159
[2026-07-08 05:38:52 +0000] [2160] [INFO] Booting worker with pid: 2160

$ docker exec sl-app bash -lc 'ps -o pid,cmd -C gunicorn'
  PID CMD
 2158 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30
 2159 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30
 2160 /app/venv/bin/python /app/venv/bin/gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 30
```

- **Server:** `gunicorn/20.0.4`, WSGI entry `wsgi:app`, bound `0.0.0.0:7777`, **2 workers** (master PID 2158, workers 2159 & 2160). The two‑worker fact matters for the in‑memory rate limiter (§7.5).

**(8) Health check** (confirms the app is live and redirects anonymous users to login):

```
$ curl -sS -i http://localhost:7777/ | head -5
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
```

### 2.2 Datastore, cache, and configuration values (read at runtime)

Key config values, read from inside the running app process:

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
from app import config
for k in [\"MEM_STORE_URI\",\"DISABLE_RATE_LIMIT\",\"EVENT_WEBHOOK\",\"EVENT_WEBHOOK_DISABLE\",
          \"MAX_NB_EMAIL_FREE_PLAN\",\"ALIAS_LIMIT\",\"ALIAS_CREATE_RATE_LIMIT_FREE\",
          \"ALIAS_CREATE_RATE_LIMIT_PAID\",\"FIRST_ALIAS_DOMAIN\"]:
    print(k, \"=\", getattr(config,k))
" 2>/dev/null | grep -E "="'
MEM_STORE_URI = None
DISABLE_RATE_LIMIT = False
EVENT_WEBHOOK = None
EVENT_WEBHOOK_DISABLE = False
MAX_NB_EMAIL_FREE_PLAN = 5
ALIAS_LIMIT = 100/day;50/hour;5/minute
ALIAS_CREATE_RATE_LIMIT_FREE = [(10, 900), (50, 3600)]
ALIAS_CREATE_RATE_LIMIT_PAID = [(50, 900), (200, 3600)]
FIRST_ALIAS_DOMAIN = sl.local
```

These correspond to `app/config.py`: `MAX_NB_EMAIL_FREE_PLAN` default 5 (`L124`), `ALIAS_LIMIT` (`L448`), `ALIAS_CREATE_RATE_LIMIT_FREE/PAID` (`L554-L558`), `MEM_STORE_URI` (`L568`), `DISABLE_RATE_LIMIT` (`L602`), `EVENT_WEBHOOK` default `None` (`L612`), `EVENT_WEBHOOK_DISABLE` (`L616`). Other bounding defaults from `.env`: `URL=http://localhost:7777`, `EMAIL_DOMAIN=sl.local`, `NOT_SEND_EMAIL=true`, `DISABLE_ONBOARDING=true`.

### 2.3 Logging

`app/log.py` configures a single logger named `SL` writing to stdout, captured by gunicorn to `/tmp/gunicorn.log` inside `sl-app`. `LOG.d`=debug, `LOG.i`=info, `LOG.w`=warning, `LOG.e`=exception (shortcut assignments at `app/log.py:L74-L77`; `LOG = _get_logger("SL")` at `app/log.py:L79`). Every HTTP request is logged by `after_request()` (`server.py:L284`) in the form `<ip> <METHOD> <path> <args> <status>, takes <t>`. The leading `<ip>` is the client address as gunicorn sees it and is therefore **vantage‑dependent**: the captures below were driven by `curl` running **inside `sl-app`**, so they show `127.0.0.1`; the identical request driven from the **host** side is logged with the Docker bridge gateway address instead — observed directly: `172.18.0.1 GET / ImmutableMultiDict([]) 302`. This `<ip>` is not part of the in‑scope O7 creation line (`app/dashboard/views/index.py:110`, which carries no client IP), so it affects no claim here. All log excerpts below are sliced from `/tmp/gunicorn.log` (backed up as `/tmp/scratch_qna/gunicorn_full.log`).

### 2.4 Users, credentials, and the premium nuance

The seeded users were inspected read‑only through the running app process:

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
from app.models import User
j=User.get(1)
print(\"john email=\",j.email,\"| is_premium=\",j.is_premium(),
      \"| lifetime_or_active_subscription=\",j.lifetime_or_active_subscription(),
      \"| can_create_new_alias=\",j.can_create_new_alias())
print(\"john active_subscription=\", j.get_active_subscription())
" 2>/dev/null | grep -E "john"'
john email= john@wick.com | is_premium= True | lifetime_or_active_subscription= True | can_create_new_alias= True
john active_subscription= <Subscription PlanEnum.monthly 2026-07-18>
```

- `john@wick.com` / `password` is **premium** — an active monthly `Subscription` seeded at `app/fake_data.py:L106-L119`. Because `can_create_new_alias()` returns `True` immediately for a user with `lifetime_or_active_subscription()` (`app/models.py:L867`, `L746`), **the free‑plan cap is never reached for john.** He was used for all happy‑path and non‑cap tests.
- To exercise the **free‑plan cap** (O8), a **free user was created at runtime** (labeled controlled setup, not a source change): `freebie@sl.local` (id 3) with runtime‑local API key `freecode`, pre‑filled to the 5‑alias cap. Its state was confirmed live (§7.1): `can_create_new_alias() == False`.
- **API authentication** uses the `Authentication` HTTP header carrying an API‑key `code` (`app/api/base.py:L16-L34`). John's seeded keys are `code` (Chrome) and `codeFF` (Firefox) (`app/fake_data.py:L121-L125`). API calls therefore need **no session cookie**.

### 2.5 Authenticating the test user (O2) — captured login flow

Login was performed through the **real auth blueprint** (`app/auth/views/login.py:L21-L25`), which on success calls `after_login()` (`app/auth/views/login_utils.py:L35-L45`) → `login_user(user)` and redirects to the dashboard. The full three‑step flow was captured.

**Step 1 — `GET /auth/login`: fetch the login page + the anonymous session cookie + CSRF token.**

```
$ curl -sS -i -c /tmp/scratch_qna/john.cookies http://localhost:7777/auth/login -o /tmp/scratch_qna/login_page.html -D /tmp/scratch_qna/login_get.hdr ; cat /tmp/scratch_qna/login_get.hdr
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:39:37 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 6918
Vary: Cookie
Set-Cookie: slapp=<redacted anonymous session cookie>; Expires=Wed, 15-Jul-2026 05:39:37 GMT; HttpOnly; Path=/; SameSite=Lax

$ grep -oE 'name="csrf_token"[^>]*value="[^"]+"' /tmp/scratch_qna/login_page.html | head -1 | sed -E 's/.*value="([^"]+)".*/\1/'
IjQ5MjUwYWExYzU1YTIxZTQyZTA1NjM3MWNmNTcxOTY3NWQxYjc0NGMi.ak3imQ.vJ5tOQv5A1LroDOS7F1y0XSFoe8
```

`GET /auth/login` returns **`200`** with a 6918‑byte HTML page, sets an **anonymous** `slapp` session cookie (7‑day expiry, `HttpOnly`, `SameSite=Lax`), and embeds a 91‑character Flask‑WTF CSRF token in the form.

**Step 2 — `POST /auth/login`: submit credentials + CSRF, receive the authenticated session + redirect.**

```
$ curl -sS -i -b /tmp/scratch_qna/john.cookies -c /tmp/scratch_qna/john.cookies -D /tmp/scratch_qna/login_post.hdr -o /dev/null \
       -X POST http://localhost:7777/auth/login \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'email=john@wick.com&password=password&csrf_token=IjQ5MjUwYWExYzU1YTIxZTQyZTA1NjM3MWNmNTcxOTY3NWQxYjc0NGMi.ak3imQ.vJ5tOQv5A1LroDOS7F1y0XSFoe8' ; cat /tmp/scratch_qna/login_post.hdr
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:39:45 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Vary: Cookie
Set-Cookie: slapp=<redacted authenticated session cookie>; Expires=Wed, 15-Jul-2026 05:39:45 GMT; HttpOnly; Path=/; SameSite=Lax
```

Successful login returns **`302`** with `Location: http://localhost:7777/dashboard/` (the `after_login` redirect target) and a **new, authenticated** `slapp` cookie. The corresponding server log confirms the login:

```
$ grep -E 'after_login|POST /auth/login' /tmp/scratch_qna/gunicorn_full.log | head -2
2026-07-08 05:39:45,102 - SL - DEBUG - 2159 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-07-08 05:39:45,110 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2609682083129883
```

**Step 3 — authenticated `GET /dashboard/`: prove the session works.**

```
$ curl -sS -i -b /tmp/scratch_qna/john.cookies http://localhost:7777/dashboard/ -o /tmp/scratch_qna/dash.html -D /tmp/scratch_qna/dash.hdr ; head -6 /tmp/scratch_qna/dash.hdr
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:39:54 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 99587

$ grep -c 'john@wick.com' /tmp/scratch_qna/dash.html ; grep -o 'name="form-name" value="create-random-email"' /tmp/scratch_qna/dash.html | head -1
13
name="form-name" value="create-random-email"
```

The authenticated dashboard returns **`200`** (99587‑byte page), mentions `john@wick.com` 13 times, and contains the `create-random-email` form — proving the session cookie authenticates the user and that the alias‑creation control is present.

### 2.6 The read‑only row‑count probe + baseline

To measure database deltas I used a temporary, **SELECT‑only** probe (kept under `/tmp/scratch_qna/probe.sh`, outside the repo tree, deleted at cleanup). It runs a single `UNION ALL` count over every candidate table:

```
$ docker exec sl-app bash -lc 'cat /tmp/scratch_qna/probe.sh'
#!/usr/bin/env bash
# Read-only row-count probe over every candidate table (SELECT only).
PGPASSWORD=mypassword psql -h sl-db -U myuser -d simplelogin -tA -c "
  select 'alias='         || count(*) from alias            union all
  select 'alias_audit_log=' || count(*) from alias_audit_log  union all
  select 'alias_mailbox='  || count(*) from alias_mailbox    union all
  select 'alias_used_on='  || count(*) from alias_used_on    union all
  select 'daily_metric='   || count(*) from daily_metric     union all
  select 'job='            || count(*) from job              union all
  select 'sync_event='     || count(*) from sync_event       union all
  select 'users='          || count(*) from users;"

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh'    # baseline, immediately after seed
alias=11
alias_audit_log=11
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2
```

The candidate table set covers the alias‑creation core (`alias`, `daily_metric`, `alias_audit_log`), the entities that *could* be created as side effects (`alias_mailbox`, `alias_used_on`, `users`), and the event/async surfaces (`sync_event`, `job`). Snapshots were taken **before and after** each creation, and creations were repeated to confirm magnitude stability.

> **Baseline notes.** The seed created **11 aliases** (`alias=11`, `users=2` = john + winston) and one `daily_metric` row for the current day, so same‑day creations **UPDATE** that row rather than inserting a new one. The first‑of‑day **INSERT** sub‑case is demonstrated separately in §5.3 by deleting the current day's `daily_metric` row at runtime (a labeled, runtime‑only DB action) and observing the re‑INSERT.

### 2.7 Repository integrity & cleanup

All observation scripts live under `/tmp/scratch_qna` inside the `sl-app` container (and a host backup under `/tmp/qna_capture`) — never inside the repository tree. They are deleted at completion, leaving only this document. Repository‑unchanged verification (the entire working‑tree delta against the last upstream **source** commit reduces to the single added `blitzy/documentation/app_2cd6ee777f8c.md`, with **no source file** differing) is performed at the end and reported in §8.4.

> **Runtime DB mutations disclosure.** The observations mutated the *running database only* (new aliases with ids 12–39 in the documented runs — re‑running or re‑verifying the flow simply appends further ids — plus a runtime‑created free user id 3, one deleted/trashed alias, incremented API‑key usage counters, and a temporarily deleted `daily_metric` row). These are ephemeral runtime state, **not** repository changes; no tracked file was touched.

---

## 3. O4 — What request the frontend sends to the backend

### 3.1 Raw captured request

The exact request the browser issues when the "Random Alias" button is clicked, captured by driving the same request against the running server as authenticated `john`. The `--trace-ascii` capture shows the request line, headers, and body verbatim (only the `slapp` cookie payload is redacted):

```
$ curl -sS --trace-ascii /tmp/scratch_qna/req1.trace -b /tmp/scratch_qna/john.cookies \
       -o /dev/null -D /tmp/scratch_qna/resp1.hdr \
       -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=IjQ5MjUwYWExYzU1YTIxZTQyZTA1NjM3MWNmNTcxOTY3NWQxYjc0NGMi.ak3i2A.wNU0E7AQRmr4NvAaF45Zuy6MOok'

# ---- request as sent on the wire (reconstructed from req1.trace; cookie payload redacted) ----
POST /dashboard/ HTTP/1.1
Host: localhost:7777
User-Agent: curl/7.88.1
Accept: */*
Cookie: slapp=<redacted authenticated session cookie for john@wick.com>
Content-Type: application/x-www-form-urlencoded
Content-Length: 132

form-name=create-random-email&csrf_token=IjQ5MjUwYWExYzU1YTIxZTQyZTA1NjM3MWNmNTcxOTY3NWQxYjc0NGMi.ak3i2A.wNU0E7AQRmr4NvAaF45Zuy6MOok
```

The raw `curl` trace lines that establish the above (the `Send header` byte count and the `Send data` body) are, verbatim:

```
$ sed -n '/=> Send header/,/=> Send data/p' /tmp/scratch_qna/req1.trace | sed -n '1,3p'
=> Send header, 494 bytes (0x1ee)
0000: POST /dashboard/ HTTP/1.1
001b: Host: localhost:7777

$ sed -n '/=> Send data, 132/,+2p' /tmp/scratch_qna/req1.trace
=> Send data, 132 bytes (0x84)
0000: form-name=create-random-email&csrf_token=IjQ5MjUwYWExYzU1YTIxZTQ
0040: yZTA1NjM3MWNmNTcxOTY3NWQxYjc0NGMi.ak3i2A.wNU0E7AQRmr4NvAaF45Zuy6
```

> **Field-order note (wire order is transport-constructed, not semantically significant).** The body order shown above (`form-name` then `csrf_token`) is the order in which the `curl --data` string was written, not the browser's. A genuine browser form submit sends **`csrf_token` first**, because the template renders `{{ csrf_form.csrf_token }}` (`templates/dashboard/index.html:L51`) *before* the `form-name` hidden input (`templates/dashboard/index.html:L52`) — the browser serializes fields in DOM order. This has **no effect** on the request: `application/x-www-form-urlencoded` is parsed into an unordered mapping, so `form-name` and `csrf_token` resolve identically regardless of order, and the byte length is the same (**132 bytes**). Every asserted property (method, path, content-type, both field names/values, length, optional `generator_scheme`) is order-independent.

### 3.2 What this shows, and the evidence

- **Method + URL:** `POST /dashboard/` — **not** `POST /`. The form (`templates/dashboard/index.html:L50` `<form method="post">`) has **no `action` attribute**, so it submits to the current document URL, which is `/dashboard/` because the dashboard blueprint is mounted with `url_prefix="/dashboard"` (`app/dashboard/base.py:L3-L8`). The root `/` view redirects authenticated users to `dashboard.index` (`server.py:L251-L255`), confirmed live: `GET /` (authenticated) → `302 Location /dashboard/`. **This corrects the plain "`POST /`" reading — the real target is `/dashboard/`.**
- **Content type:** `application/x-www-form-urlencoded` (a classic HTML form submit; no JSON, no multipart), body length **132 bytes**.
- **Body fields (exactly two required):**
  - `form-name=create-random-email` — selects the random‑alias branch inside the multiplexed dashboard handler (`app/dashboard/views/index.py:L97` guards on this value).
  - `csrf_token=<91‑char token>` — the Flask‑WTF CSRF token rendered by the form.
- **Optional third field `generator_scheme`:** the main "Random Alias" button does **not** send it (template `L50-L58`); the dropdown items add it — **"By Random Words" → `generator_scheme=1`** (`AliasGeneratorEnum.word.value`, template `L68-L76`) and **"By UUID" → `generator_scheme=2`** (`AliasGeneratorEnum.uuid.value`, template `L78-L86`). Runtime confirmation of the values through the web path:

```
$ curl -sS -o /dev/null -w '%{http_code}\n' -b /tmp/scratch_qna/john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid token>'                         # no generator_scheme
302
$ curl -sS -o /dev/null -w '%{http_code}\n' -b /tmp/scratch_qna/john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&generator_scheme=1&csrf_token=<valid token>'      # word
302
$ curl -sS -o /dev/null -w '%{http_code}\n' -b /tmp/scratch_qna/john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&generator_scheme=2&csrf_token=<valid token>'      # uuid
302
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select id,email from alias where id in (37,38,39) order by id;"
37|punnet_secede692@sl.local
38|boldly_roping392@sl.local
39|866179b7-087b-454b-bb5f-d9702f2f7920@sl.local
```

Omitting `generator_scheme` (id 37) and `=1` (id 38) produced **word‑style** aliases (`punnet_secede692@sl.local`, `boldly_roping392@sl.local`); `generator_scheme=2` (id 39) produced a **uuid‑style** alias (`866179b7-087b-454b-bb5f-d9702f2f7920@sl.local`). This confirms `word=1`, `uuid=2`.

---

## 4. O5 — The backend response (status codes, payloads, metadata)

### 4.1 Web flow → HTTP 302 redirect

Raw response headers for a successful web creation (the first web run, alias id 12):

```
$ cat /tmp/scratch_qna/resp1.hdr
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:40:40 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 339
Location: http://localhost:7777/dashboard/?highlight_alias_id=12&query=&sort=&filter=
Vary: Cookie
Set-Cookie: slapp=<redacted session cookie carrying the success flash>; Expires=Wed, 15-Jul-2026 05:40:40 GMT; HttpOnly; Path=/; SameSite=Lax
```

- **Status: `302 FOUND`.** The body is the standard 339‑byte redirect HTML pointing at the `Location`.
- **`Location: /dashboard/?highlight_alias_id=<id>&query=&sort=&filter=`.** The newly created alias's id is passed as `highlight_alias_id` so the redirected page can highlight it. Evidence: `app/dashboard/views/index.py:L113-L121`. The id in `Location` is the only part that varies per creation (observed `12`, then `13` on the second run — §5.2).
- **Metadata:** a fresh `Set-Cookie: slapp=<redacted …>` carries the success flash *"Alias &lt;email&gt; has been created"* (`app/dashboard/views/index.py:L111`), rendered on the next page load.

### 4.2 API flow → HTTP 201 with serialized JSON

Raw response (headers + body) for a successful API creation (no cookie; API‑key header only), captured to `/tmp/scratch_qna/api_create.body`:

```
$ curl -sS -i -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: code'
HTTP/1.1 201 CREATED
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:41:46 GMT
Connection: close
Content-Type: application/json
Content-Length: 406
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<redacted fresh anonymous session>; Expires=Wed, 15-Jul-2026 05:41:46 GMT; HttpOnly; Path=/; SameSite=Lax

{"alias":"foxier_joined409@sl.local","creation_date":"2026-07-08 05:41:46+00:00","creation_timestamp":1783489306,"disable_pgp":false,"email":"foxier_joined409@sl.local","enabled":true,"id":15,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
```

- **Status: `201 CREATED`.** Metadata: `Content-Type: application/json`, `Content-Length: 406`, `Access-Control-Allow-Origin: *`. The `Set-Cookie` is an incidental fresh anonymous session (API auth is by header, not cookie).
- **Keys are alphabetically sorted** because Flask's `JSON_SORT_KEYS` defaults to `True`.

### 4.3 Every JSON key enumerated → the source line that produces it

The body has **17 keys**: 16 from `serialize_alias_info_v2` (`app/api/serializer.py:L55-L93`) plus the top‑level `alias` added by the endpoint (`app/api/views/new_random_alias.py:L115`, `jsonify(alias=alias.email, **serialize_alias_info_v2(...))`).

| JSON key | Observed value (id 15) | Source line |
|----------|------------------------|-------------|
| `alias` | `"foxier_joined409@sl.local"` | `app/api/views/new_random_alias.py:L115` (top‑level, = the alias email) |
| `id` | `15` | `app/api/serializer.py:L58` |
| `email` | `"foxier_joined409@sl.local"` | `app/api/serializer.py:L59` |
| `creation_date` | `"2026-07-08 05:41:46+00:00"` | `app/api/serializer.py:L60` (`created_at.format()`) |
| `creation_timestamp` | `1783489306` | `app/api/serializer.py:L61` (`created_at.timestamp`) |
| `enabled` | `true` | `app/api/serializer.py:L62` |
| `note` | `null` | `app/api/serializer.py:L63` |
| `name` | `null` | `app/api/serializer.py:L64` |
| `nb_forward` | `0` | `app/api/serializer.py:L66` |
| `nb_block` | `0` | `app/api/serializer.py:L67` (from `alias_info.nb_blocked`) |
| `nb_reply` | `0` | `app/api/serializer.py:L68` |
| `mailbox` | `{"email":"john@wick.com","id":1}` | `app/api/serializer.py:L70` |
| `mailboxes` | `[{"email":"john@wick.com","id":1}]` | `app/api/serializer.py:L71-L74` |
| `support_pgp` | `false` | `app/api/serializer.py:L75` (`mailbox_support_pgp()`) |
| `disable_pgp` | `false` | `app/api/serializer.py:L76` |
| `latest_activity` | `null` | `app/api/serializer.py:L77` (populated only if a `latest_email_log` exists, `L80-L92`) |
| `pinned` | `false` | `app/api/serializer.py:L78` |

`latest_activity` is `null` for a freshly created alias because there is no email log yet — the field is populated only inside the `if alias_info.latest_email_log:` block at `app/api/serializer.py:L80-L92`, which is not reached for a new alias. (The `serialize_alias_info_v2` dict literal spans `app/api/serializer.py:L56-L79`; the comment `# Alias field` sits at `L57`, so the first key `id` is at `L58`.)

### 4.4 Direct comparison — the responses differ, the creation effects do not

The web and API responses differ **only** in transport and shape (`302` redirect vs `201` + JSON). The **alias‑creation database work behind them is identical** because both call the same `Alias.create` (proved by the identical 3‑table delta in §5.4). The one database difference is *not* in the creation core but in **API‑auth bookkeeping**: the API request also bumps the `api_key` row's `times`/`last_used` before creation (`app/api/base.py:L30-L32`; §5.4), which the cookie‑based web path does not. This is the central "no difference where it matters" result, stated precisely.

---

## 5. O6 — Database changes (records inserted/updated, how many tables, related entities)

**Direct answer:** a successful creation in the default single‑mailbox configuration **touches exactly three tables** in its creation core — `alias` (INSERT), `daily_metric` (INSERT on the first alias of a calendar day, otherwise UPDATE of `nb_alias`), and `alias_audit_log` (INSERT). No related entities (`alias_mailbox`, `alias_used_on`, `users`, `sync_event`) are written on this path. The count is stable across runs. On the **API** path only, one additional table (`api_key`) is updated for authentication accounting **before** the creation core (§5.4) — not part of the 3‑table core.

### 5.1 Web creation, run 1 (same day → `daily_metric` UPDATE)

```
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh'   # BEFORE run 1
alias=11
alias_audit_log=11
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2

$ curl -sS -o /dev/null -D - -b /tmp/scratch_qna/john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid token>' | grep -iE '^HTTP|^Location'
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=12&query=&sort=&filter=

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh'   # AFTER run 1
alias=12
alias_audit_log=12
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2
```

**Delta run 1:** `alias` **+1**, `alias_audit_log` **+1**, `daily_metric` row‑count **+0** (the existing day row was UPDATED — `nb_alias` went `11 → 12`, shown in §5.3), everything else **+0**.

### 5.2 Web creation, run 2 — identical input, magnitude stability

```
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh'   # BEFORE run 2
alias=12
alias_audit_log=12
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2

$ curl -sS -o /dev/null -D - -b /tmp/scratch_qna/john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid token>' | grep -iE '^HTTP|^Location'
HTTP/1.1 302 FOUND
Location: http://localhost:7777/dashboard/?highlight_alias_id=13&query=&sort=&filter=

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh'   # AFTER run 2
alias=13
alias_audit_log=13
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2
```

**Delta run 2:** identical shape — `alias` **+1**, `alias_audit_log` **+1**, `daily_metric` row‑count **+0** (UPDATE, `nb_alias 12 → 13`), all else **+0**. **The number of tables touched is stable at THREE across both runs.** (It remained 3 on every subsequent creation in §5.4 and §7 as well.)

### 5.3 The `daily_metric` INSERT‑vs‑UPDATE sub‑cases

`Alias.create` does `DailyMetric.get_or_create_today_metric().nb_alias += 1` (`app/models.py:L1661`). `get_or_create_today_metric` (`app/models.py:L3280`) fetches the row for today's date or **creates** it if absent. So:

- **Same day (row already exists):** the row is **UPDATE**d; `daily_metric` row‑count does not change. Proof — `nb_alias` incremented across the two runs above:

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select id, date, nb_alias from daily_metric where date = current_date;"
1|2026-07-08|13          # after run 2 — same single row (id 1), nb_alias climbed 11 -> 12 -> 13
```

- **First alias of a new day (no row yet):** the row is **INSERT**ed. Demonstrated with a **labeled runtime‑only DB action** — delete today's `daily_metric` row, then create an alias and watch the row re‑appear:

```
$ docker exec sl-db psql -U myuser -d simplelogin -c "delete from daily_metric where date = current_date;"   # LABELED runtime setup only
DELETE 1
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep daily_metric   # BEFORE
daily_metric=0

$ curl -sS -o /dev/null -D - -b /tmp/scratch_qna/john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid token>' | grep -i '^HTTP'
HTTP/1.1 302 FOUND

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep -E 'alias=|alias_audit_log=|daily_metric='   # AFTER
alias=14
alias_audit_log=14
daily_metric=1

$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select id, date, nb_alias from daily_metric where date = current_date;"
2|2026-07-08|1           # fresh row INSERTed (id 2), created with nb_alias=0 then incremented to 1
```

So on the **first‑of‑day** creation the touched‑table row‑count delta is `alias +1`, `alias_audit_log +1`, `daily_metric +1` (INSERT); on **every later same‑day** creation it is `alias +1`, `alias_audit_log +1`, `daily_metric +0` (UPDATE). **Either way the set of distinct tables touched is the same three.**

### 5.4 API creation re‑confirms the identical 3‑table core (+ the `api_key` auth write)

```
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh'   # BEFORE API call
alias=14
alias_audit_log=14
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2

$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select code,times,last_used from api_key where code='code';"   # api_key BEFORE
code|0|

$ curl -sS -o /dev/null -w 'HTTP %{http_code}\n' -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: code'
HTTP 201

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh'   # AFTER API call
alias=15
alias_audit_log=15
alias_mailbox=1
alias_used_on=0
daily_metric=1
job=0
sync_event=0
users=2

$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select code,times,last_used from api_key where code='code';"   # api_key AFTER
code|1|2026-07-08 05:41:46.505096
```

The alias‑creation delta is identical to the web path (`alias +1`, `alias_audit_log +1`, `daily_metric` UPDATE, all else +0), **proving both entry points share `Alias.create`** (`app/models.py:L1628-L1692`).

> **API‑transport‑only write (a fourth table, not part of the creation core).** API‑key authentication updates the `api_key` row's usage counters **before** creation (`app/api/base.py:L30-L32`, `times += 1` at `L31`, `last_used = arrow.now()` at `L30`, `Session.commit()` at `L32`): for key `code`, `times` went `0 → 1` and `last_used` advanced to the request time. This is auth bookkeeping on `api_key`, **distinct from** the three creation tables, and does not occur on the cookie‑authenticated web path. So: *3‑table creation core (both flows) + 1 `api_key` auth‑accounting table (API flow only).* Note that this counter is **best‑effort, not concurrency‑safe**: `api_key.times += 1` (`app/api/base.py:L31`) is a non‑atomic read‑modify‑write performed *outside* the `alias_creation` parallel lock, so under truly concurrent API calls the increment can lose updates — unlike `daily_metric.nb_alias`, which is incremented inside the creation core. The single‑call value (`0 → 1`) shown above is exact; the concurrent behavior is a property of the (unchanged) application source and is not part of the alias‑creation guarantees.

### 5.5 New‑row contents

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc \
  "select id,email,user_id,mailbox_id,enabled,note from alias where id=12;"
12|virgin_kiting230@sl.local|1|1|t|

$ docker exec sl-db psql -U myuser -d simplelogin -tAc \
  "select id,user_id,alias_id,alias_email,action,message from alias_audit_log where id=12;"
12|1|12|virgin_kiting230@sl.local|create|New alias created
```

- The `alias` row carries `mailbox_id=1` **directly on the row** (`app/dashboard/views/index.py:L106` sets `alias.mailbox_id = current_user.default_mailbox_id`). Because the single default mailbox is stored on the alias row itself, **no `alias_mailbox` join row is created** — that table is only used for *additional* mailboxes (`app/models.py:L2939`; `app/fake_data.py:L155-L157`).
- The `alias_audit_log` row has `action='create'` (`AliasAuditLogAction.CreateAlias = "create"`, `app/alias_audit_log_utils.py:L8`) and `message='New alias created'` (`app/models.py:L1688-L1690`).

### 5.6 What is NOT written (related‑entity absences, default flow)

Across every successful creation above, these stayed at zero‑delta, and each has a code reason:

| Table | Delta | Why (cause → effect) |
|-------|-------|----------------------|
| `sync_event` | **+0** | Event dispatch short‑circuits before the DB write because `EVENT_WEBHOOK` is unset (`app/events/event_dispatcher.py:L61-L65`); the write path `SyncEvent.create` + `NOTIFY` (`L24-L26`) is never reached. See §6. |
| `alias_mailbox` | **+0** | The single default mailbox lives on the `alias` row (`mailbox_id`); the join table is for extra mailboxes only (`app/models.py:L2939`). |
| `users` | **+0** | The partner‑flag UPDATE fires only when `FLAG_PARTNER_CREATED` is set on the alias and the user's `FLAG_CREATED_ALIAS_FROM_PARTNER` is unset (`app/models.py:L1663-L1667`); not the case in the default flow. |
| `alias_used_on` | **+0** | Written only on the **API** path when a `hostname` query param is supplied (`app/api/views/new_random_alias.py:L109-L112`); see the conditional in §7.6. |
| `job` | **+0** | Random alias creation enqueues no async job (§6); onboarding jobs are disabled (`DISABLE_ONBOARDING=true`, `app/models.py:L647`). |

---

## 6. O7 — Background tasks, follow‑up events, and additional work (from the logs)

**Direct answer:** in the default configuration there is **no background or follow‑up work** after the initial write. Alias creation produces one creation log line (web path) and then event dispatch **short‑circuits**; nothing is enqueued and nothing is notified.

### 6.1 The per‑request log slice for one web creation

The complete slice for the web creation of alias 12 (`virgin_kiting230@sl.local`), sliced by timestamp from the captured gunicorn log — verbatim, no elision:

```
$ awk 'NR>=28 && NR<=31' /tmp/scratch_qna/gunicorn_full.log
2026-07-08 05:40:40,874 - SL - DEBUG - 2160 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email virgin_kiting230@sl.local
2026-07-08 05:40:40,887 - SL - INFO - 2160 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:40:40,894 - SL - DEBUG - 2160 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 12 virgin_kiting230@sl.local> for user <User 1 John Wick john@wick.com>
2026-07-08 05:40:40,898 - SL - DEBUG - 2160 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /dashboard/ ImmutableMultiDict([]) 302, takes 0.04178810119628906
```

- **Creation log line:** `app/dashboard/views/index.py:110` — `LOG.d("create new random alias %s for user %s", alias, current_user)`, a DEBUG line emitted on the **web path only**.
- **Event short‑circuit line:** `app/events/event_dispatcher.py:62` — `LOG.i("Not sending events because webhook is not configured and allowed to be empty")`, an INFO line.
- **Request‑completion line:** `server.py:284` — the `after_request` log showing the `302`.

For the **API** path, the successful creation of alias 15 logs the event short‑circuit and the request completion, but **no `index.py:110` creation line** (that log lives only in the web handler):

```
$ awk '/05:41:46/ && (/event_dispatcher.py:62/ || /POST \/api\/alias\/random\/new/)' /tmp/scratch_qna/gunicorn_full.log
2026-07-08 05:41:46,506 - SL - INFO - 2159 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-08 05:41:46,513 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 201, takes 0.027766704559326172

$ grep -c 'index.py:110' /tmp/scratch_qna/gunicorn_full.log      # all such lines are web POST /dashboard/ creations
6
```

### 6.2 Why event dispatch short‑circuits — the three early returns

`EventDispatcher.send_event` (`app/events/event_dispatcher.py:L49-L84`) has three guard clauses; the **second** one fires here:

```
# app/events/event_dispatcher.py:L57-L70
if config.EVENT_WEBHOOK_DISABLE:                                   # L57  -> False (unset)
    LOG.i("Not sending events because webhook is disabled")
    return
if not config.EVENT_WEBHOOK and skip_if_webhook_missing:           # L61  -> True  (EVENT_WEBHOOK is None)  <-- FIRES
    LOG.i("Not sending events because webhook is not configured and allowed to be empty")   # L62 (the observed line)
    return
partner_user = EventDispatcher.__partner_user(user.id)             # L67  (not reached)
if not partner_user:
    LOG.i(f"Not sending events because there's no partner user for user {user}")
    return
```

Because `EVENT_WEBHOOK is None` (§2.2), the second `return` executes and the write path is never reached:

```
# app/events/event_dispatcher.py:L24-L26  (PostgresDispatcher.send — UNREACHED in default config)
def send(self, event: bytes):
    instance = SyncEvent.create(content=event, flush=True)                 # would INSERT sync_event
    Session.execute(f"NOTIFY {NOTIFICATION_CHANNEL}, '{instance.id}';")    # would NOTIFY simplelogin_sync_events
```

**Consequence, proven by the probe:** `sync_event` stays at `0` on every creation (§5), i.e. **no `sync_event` row and no `NOTIFY`**. The `AliasCreated` protobuf event is still *built* in `Alias.create` (`app/models.py:L1680-L1687`, fields `id, email, note, enabled=True, created_at` per `proto/event.proto`), but it is discarded at the second early return.

### 6.3 The async workers do no alias‑creation follow‑up (run with real output)

Both background workers were **run** (bounded with `timeout`, which exits with code 124 on expiry — an expected, non‑error stop for a long‑running daemon), and the event/job tables were probed around a creation.

- **`job_runner.py`** — `process_job` (`job_runner.py:L188`) dispatches queued `Job` rows; the `JOB_SEND_ALIAS_CREATION_EVENTS` branch is `job_runner.py:L295-L302`, and the `__main__` polling loop (`job_runner.py:L329-L347`, `time.sleep(10)` at `L347`) waits for rows. Random alias creation enqueues **no `Job`**, so the worker only prints its startup banner and idles:

```
$ docker exec sl-app bash -lc 'cd /app && timeout 14 venv/bin/python job_runner.py' 2>&1 | tee /tmp/scratch_qna/o7_jobrunner.log ; echo "exit=$?"
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ynzwctnajifcipsxuatd
Upload files to local dir
>>> init logging <<<
2026-07-08 05:42:37,775 - SL - DEBUG - 2448 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
exit=124
```

  `exit=124` is the `timeout` expiry (the daemon never terminates on its own); there is **no alias‑creation follow‑up output**. The `job` table is `0` both before and after a creation:

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select count(*) from job;"    # before and after alias creation
0
```

- **`event_listener.py`** — its `main()` (`event_listener.py:L29-L47`) selects a `PostgresEventSource` (`L34`) and a sink (`L39-L40`) and then blocks on Postgres `LISTEN/NOTIFY` (`events/event_source.py:L49`). Started in dry‑run it initialized its source/sink and idled, consuming nothing (no `NOTIFY` was ever issued, §6.2):

```
$ docker exec sl-app bash -lc 'cd /app && timeout 12 venv/bin/python event_listener.py listener --dry-run' 2>&1 | tee /tmp/scratch_qna/o7_eventlistener.log ; echo "exit=$?"
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/sdolhgwkztrwnnhrsuai
Upload files to local dir
>>> init logging <<<
2026-07-08 05:43:04,757 - SL - DEBUG - 2466 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-08 05:43:04,916 - SL - INFO - 2466 - "/app/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-08 05:43:04,936 - SL - INFO - 2466 - "/app/event_listener.py:40" - main() -  - Starting with ConsoleEventSink
2026-07-08 05:43:04,956 - SL - INFO - 2466 - "/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
exit=124
```

  After `Starting to listen to events` the process blocks (again `exit=124` from `timeout`), and `sync_event` is `0` before and after any creation:

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select count(*) from sync_event;"    # before and after alias creation
0
```

> The bulk job `send_alias_creation_events_for_user` exists (`app/jobs/event_jobs.py:L9-L52`) but is a **backfill** over *all* of a user's aliases, triggered elsewhere (not by a single alias creation) and itself calls `EventDispatcher.send_event`, which short‑circuits in this config. So even if it ran, it would emit nothing here. **The no‑follow‑up conclusion is thus confirmed by observation** — `job=0`, `sync_event=0`, idle workers with real output above — **not merely inferred.**

---

## 7. O8 — Failure behavior (validation, DB, network) across response, logs, and DB

Each branch below shows, in order: the **raw request**, the **raw response**, the **relevant log line(s)**, and a **before/after DB probe**. On every refusal the alias‑creation tables are unchanged (zero write).

### 7.1 Free‑plan alias cap (`MAX_NB_EMAIL_FREE_PLAN = 5`)

The guard is `user.can_create_new_alias()` (`app/models.py:L867`). Its body checks `lifetime_or_active_subscription()` (`L746`) first and, only for non‑subscribers, compares the alias count against `max_alias_for_free_account()` (`L858`, = `MAX_NB_EMAIL_FREE_PLAN` = 5) — and the docstring notes the cap applies **"even in the free trial"**. **Because `john@wick.com` is premium (§2.4), the cap does not apply to him**, so it was reproduced with a runtime‑created free user.

**Runtime free‑user setup (labeled controlled setup, not a source change).** A free user, a local API key `freecode`, and enough aliases to reach the cap were created through the ORM at runtime, then the resulting state was confirmed:

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
from app.models import User, Alias, ApiKey
from app.extensions import db
from app.db import Session
u = User.create(email=\"freebie@sl.local\", password=\"password\", name=\"Free Bee\", activated=True)
Session.commit()
ApiKey.create(user_id=u.id, name=\"free-key\", code=\"freecode\"); Session.commit()
# top up to the 5-alias cap (User.create already made 1 newsletter alias)
while Alias.filter_by(user_id=u.id).count() < 5:
    Alias.create_new_random(user=u); Session.commit()
print(\"free user id=\", u.id, \"alias_count=\", Alias.filter_by(user_id=u.id).count())
" 2>/dev/null | grep -E "free user"'
free user id= 3 alias_count= 5

$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
from app.models import User, Alias, ApiKey
u=User.get(3)
print(\"email=\",u.email,\"| is_premium(trial)=\",u.is_premium(),\"| lifetime_or_active_subscription=\",u.lifetime_or_active_subscription(),\"| can_create_new_alias=\",u.can_create_new_alias())
print(\"alias_count=\", Alias.filter_by(user_id=3).count(), \"| api_keys=\", [(k.name,k.code,k.times) for k in ApiKey.filter_by(user_id=3).all()])
" 2>/dev/null | grep -E "email=|alias_count="'
email= freebie@sl.local | is_premium(trial)= True | lifetime_or_active_subscription= False | can_create_new_alias= False
alias_count= 5 | api_keys= [('free-key', 'freecode', 1)]
```

> **Precise nuance:** the new free user is in its initial trial, so `is_premium()` returns `True`, **but** the cap guard uses `lifetime_or_active_subscription()` (which is `False` — no paid subscription), and with 5 aliases `can_create_new_alias()` returns `False`. `freecode` is a **runtime‑local test API key**, not a production secret.

**API surface → HTTP 400** (`app/api/views/new_random_alias.py:L35-L43`):

```
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | tr '\n' ' '     # BEFORE
alias=21 alias_audit_log=21 alias_mailbox=1 alias_used_on=0 daily_metric=1 job=0 sync_event=0 users=3

$ curl -sS -i -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: freecode'
HTTP/1.1 400 BAD REQUEST
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:43:47 GMT
Connection: close
Content-Type: application/json
Content-Length: 141
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<redacted fresh anonymous session>; Expires=Wed, 15-Jul-2026 05:43:47 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"You have reached the limitation of a free account with the maximum of 5 aliases, please upgrade your plan to create more aliases"}

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | tr '\n' ' '     # AFTER (identical — zero write)
alias=21 alias_audit_log=21 alias_mailbox=1 alias_used_on=0 daily_metric=1 job=0 sync_event=0 users=3
```

Log (the guard's DEBUG line at `new_random_alias.py:36`, then the `400` request‑completion):

```
$ grep -E 'cannot create new random alias|POST /api/alias/random/new .* 400' /tmp/scratch_qna/gunicorn_full.log | grep '05:43:47'
2026-07-08 05:43:47,564 - SL - DEBUG - 2159 - "/app/app/api/views/new_random_alias.py:36" - new_random_alias() -  - user <User 3 Free Bee freebie@sl.local> cannot create new random alias
2026-07-08 05:43:47,592 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 400, takes 0.04593992233276367
```

**Web surface → HTTP 302 + flash** (else branch of `can_create_new_alias`, `app/dashboard/views/index.py:L123`):

```
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep '^alias='     # BEFORE
alias=21

$ curl -sS -i -b /tmp/scratch_qna/free.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=<valid freebie token>'
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:44:04 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 309
Location: http://localhost:7777/dashboard/?query=&sort=&filter=&page=0
Vary: Cookie
Set-Cookie: slapp=<redacted session cookie carrying the upgrade flash>; Expires=Wed, 15-Jul-2026 05:44:04 GMT; HttpOnly; Path=/; SameSite=Lax

$ curl -sS -b /tmp/scratch_qna/free.cookies http://localhost:7777/dashboard/ | grep -o 'You need to upgrade your plan to create new alias.'
You need to upgrade your plan to create new alias.

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep '^alias='     # AFTER (zero write)
alias=21
```

Log (the `302` request‑completion):

```
$ grep -E 'POST /dashboard/ .* 302' /tmp/scratch_qna/gunicorn_full.log | grep '05:44:04'
2026-07-08 05:44:04,991 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /dashboard/ ImmutableMultiDict([]) 302, takes 0.010960578918457031
```

- The web redirect goes to the fall‑through `/dashboard/?query=&sort=&filter=&page=0` — **note the absence of `highlight_alias_id`**, which distinguishes a refusal from a success (§4.1). The flash *"You need to upgrade your plan to create new alias."* renders on the followed page. Alias count unchanged (`21 → 21`, **zero write**).

### 7.2 Trashed‑alias reuse → `AliasInTrashError` (two distinct surfaces)

`Alias.create` refuses to re‑mint an email that sits in the trash tables, raising `AliasInTrashError` (`app/models.py:L1647-L1652`) — reproduced verbatim with line numbers:

```python
# app/models.py:L1647-L1652 (verbatim)
1647:         # make sure alias is not in global trash, i.e. DeletedAlias table
1648:         if DeletedAlias.get_by(email=email):
1649:             raise AliasInTrashError
1650:
1651:         if DomainDeletedAlias.get_by(email=email):
1652:             raise AliasInTrashError
```

**Surface A — random endpoint's hostname branch (raises, then catches, then falls back → still 201).** First create a website‑derived alias on john's custom domain `old.com` via `?hostname=groupon.com` (id 22), then delete it so its email lands in the (domain) trash, then re‑request it:

```
$ curl -sS -i -X DELETE http://localhost:7777/api/aliases/22 -H 'Authentication: code'
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:44:35 GMT
Connection: close
Content-Type: application/json
Content-Length: 17
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<redacted anonymous session>; Expires=Wed, 15-Jul-2026 05:44:35 GMT; HttpOnly; Path=/; SameSite=Lax

{"deleted":true}

$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select id,email,domain_id from domain_deleted_alias where email like 'groupon%';"
1|groupon@old.com|2                 # groupon@old.com is now in the domain trash

$ curl -sS -X POST 'http://localhost:7777/api/alias/random/new?hostname=groupon.com' -H 'Authentication: code'
{"alias":"doughy_tuneup610@sl.local","creation_date":"2026-07-08 05:44:35+00:00","creation_timestamp":1783489475,"disable_pgp":false,"email":"doughy_tuneup610@sl.local","enabled":true,"id":23,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
```

The response is **`201` with a *random* alias (`doughy_tuneup610@sl.local`, id 23), not `groupon@old.com`** — the trashed email was refused and the endpoint fell back to a random one. The log shows the exact exception path:

```
$ grep -E 'has deleted alias <Alias 22|Moving <Alias 22|Use groupon.com|create new alias groupon|is in trash|hostname., .groupon' /tmp/scratch_qna/gunicorn_full.log | grep '05:44:35'
2026-07-08 05:44:35,133 - SL - INFO - 2159 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 1 John Wick john@wick.com> has deleted alias <Alias 22 groupon@old.com>
2026-07-08 05:44:35,138 - SL - INFO - 2159 - "/app/app/alias_utils.py:360" - delete_alias() -  - Moving <Alias 22 groupon@old.com> to domain 2 trash <DomainDeletedAlias 1 groupon@old.com>
2026-07-08 05:44:35,213 - SL - DEBUG - 2159 - "/app/app/api/views/new_random_alias.py:55" - new_random_alias() -  - Use groupon.com to create new alias
2026-07-08 05:44:35,238 - SL - DEBUG - 2159 - "/app/app/api/views/new_random_alias.py:82" - new_random_alias() -  - create new alias groupon@old.com
2026-07-08 05:44:35,240 - SL - INFO - 2159 - "/app/app/api/views/new_random_alias.py:92" - new_random_alias() -  - Alias groupon@old.com is in trash
2026-07-08 05:44:35,262 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([('hostname', 'groupon.com')]) 201, takes 0.05560183525085449
```

The line `new_random_alias.py:92 Alias groupon@old.com is in trash` is the `except AliasInTrashError:` handler at `app/api/views/new_random_alias.py:L91-L93`, catching the raise from `app/models.py:L1651-L1652` (the `DomainDeletedAlias` branch — `groupon@old.com` is a custom‑domain alias, so the delete moved it to *domain* trash). DB effect: the delete + random‑create left the audit log at **+2** (one `delete`, one `create`), confirmed:

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select id,alias_id,action,message from alias_audit_log order by id desc limit 3;"
24|23|create|New alias created
23|22|delete|Alias deleted by user action
22|22|create|New alias created
```

**Surface B — custom‑alias endpoint pre‑checks trash → HTTP 409 (client‑visible error).** The v2/v3 custom endpoints explicitly check the trash tables *before* calling `Alias.create` and return `409` (`app/api/views/new_custom_alias.py:L83-L88`). Using a real signed suffix obtained from the options endpoint:

```
$ curl -sS 'http://localhost:7777/api/v5/alias/options' -H 'Authentication: code' | python3 -m json.tool | grep -A1 '"suffix": "@old.com"'
            "signed_suffix": "@old.com.ak3j3w.V50SRrydgNo9yzLr9HCyHAp7rS4",
            "suffix": "@old.com"

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | tr '\n' ' '     # BEFORE
alias=22 alias_audit_log=24 alias_mailbox=1 alias_used_on=1 daily_metric=1 job=0 sync_event=0 users=3

$ curl -sS -i -X POST http://localhost:7777/api/v2/alias/custom/new -H 'Authentication: code' \
       -H 'Content-Type: application/json' \
       -d '{"alias_prefix":"groupon","signed_suffix":"@old.com.ak3j3w.V50SRrydgNo9yzLr9HCyHAp7rS4"}'
HTTP/1.1 409 CONFLICT
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:45:03 GMT
Connection: close
Content-Type: application/json
Content-Length: 49
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<redacted fresh anonymous session>; Expires=Wed, 15-Jul-2026 05:45:03 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"alias groupon@old.com already exists"}

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | tr '\n' ' '     # AFTER (identical — zero write)
alias=22 alias_audit_log=24 alias_mailbox=1 alias_used_on=1 daily_metric=1 job=0 sync_event=0 users=3
```

Log:

```
$ grep -E 'full alias already used groupon|POST /api/v2/alias/custom/new .* 409' /tmp/scratch_qna/gunicorn_full.log
2026-07-08 05:45:03,749 - SL - DEBUG - 2160 - "/app/app/api/views/new_custom_alias.py:87" - new_custom_alias_v2() -  - full alias already used groupon@old.com
2026-07-08 05:45:03,749 - SL - DEBUG - 2160 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/v2/alias/custom/new ImmutableMultiDict([]) 409, takes 0.02162647247314453
```

**Zero DB writes.** So the trash guard surfaces as a caught‑and‑fallback (`201`, random) on the random endpoint and as an explicit **`409`** on the custom endpoint.

### 7.3 Invalid CSRF (web) → flash "Invalid request" + 302

CSRF validation failure is handled at `app/dashboard/views/index.py:L88-L90` (`if not csrf_form.validate(): flash("Invalid request","warning"); return redirect(request.url)`):

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select count(*) from alias;"   # BEFORE
22
$ curl -sS -i -b /tmp/scratch_qna/john.cookies -X POST http://localhost:7777/dashboard/ \
       -H 'Content-Type: application/x-www-form-urlencoded' \
       --data 'form-name=create-random-email&csrf_token=GARBAGE_INVALID_TOKEN_xxx'
HTTP/1.1 302 FOUND
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:45:16 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 271
Location: http://localhost:7777/dashboard/
Vary: Cookie
Set-Cookie: slapp=<redacted session cookie carrying the warning flash>; Expires=Wed, 15-Jul-2026 05:45:16 GMT; HttpOnly; Path=/; SameSite=Lax

$ curl -sS -b /tmp/scratch_qna/john.cookies http://localhost:7777/dashboard/ | grep -o 'Invalid request'
Invalid request
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select count(*) from alias;"   # AFTER
22
```

Log:

```
$ grep -E 'POST /dashboard/ .* 302' /tmp/scratch_qna/gunicorn_full.log | grep '05:45:16'
2026-07-08 05:45:16,965 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /dashboard/ ImmutableMultiDict([]) 302, takes 0.003892660140991211
```

- **`302 FOUND`** with `Location: http://localhost:7777/dashboard/` — exactly `request.url` (the dashboard root, **no** query string and **no** `highlight_alias_id`), which distinguishes it from both the success redirect (§4.1) and the free‑plan refusal redirect (§7.1). The flash *"Invalid request"* renders on the followed page; alias count unchanged (`22 → 22`, **zero write**).

### 7.4 Invalid API `mode` → HTTP 400

`mode` is validated at `app/api/views/new_random_alias.py:L98-L104` (`word`/`uuid`, else 400):

```
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep '^alias='     # BEFORE
alias=22
$ curl -sS -i -X POST 'http://localhost:7777/api/alias/random/new?mode=banana' -H 'Authentication: code'
HTTP/1.1 400 BAD REQUEST
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:45:29 GMT
Connection: close
Content-Type: application/json
Content-Length: 47
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<redacted fresh anonymous session>; Expires=Wed, 15-Jul-2026 05:45:29 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"banana must be either word or uuid"}

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep '^alias='     # AFTER (zero write)
alias=22
```

Log:

```
$ grep -E "POST /api/alias/random/new .*'mode'.* 400" /tmp/scratch_qna/gunicorn_full.log
2026-07-08 05:45:29,388 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([('mode', 'banana')]) 400, takes 0.006555795669555664
```

For contrast, the valid modes both succeed (creating ids 24 and 25):

```
$ curl -sS -o /dev/null -w '%{http_code}\n' -X POST 'http://localhost:7777/api/alias/random/new?mode=word' -H 'Authentication: code'
201
$ curl -sS -o /dev/null -w '%{http_code}\n' -X POST 'http://localhost:7777/api/alias/random/new?mode=uuid' -H 'Authentication: code'
201
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select id,email from alias where id in (24,25) order by id;"
24|carder_sutler916@sl.local
25|46182fa2-c9d9-4ef5-83c4-b08e1e21ca6f@sl.local
```

`?mode=word` → a word‑style alias (`carder_sutler916@sl.local`, id 24); `?mode=uuid` → a uuid‑style alias (id 25). DB delta of the invalid `banana` request is **zero** (probe identical before/after).

### 7.5 Rate limiting → HTTP 429, and the exact taxonomy of controls

The endpoints carry `@limiter.limit(ALIAS_LIMIT)` where `ALIAS_LIMIT="100/day;50/hour;5/minute"` (`app/config.py:L448`). Driving the API past the per‑minute bucket with a 20‑request burst, probing the DB before and after:

```
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep -E '^alias=|^alias_audit_log='     # BEFORE burst
alias=24
alias_audit_log=26

$ for i in $(seq 1 20); do
    curl -sS -o /dev/null -w '%{http_code} ' -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: code'
  done; echo
201 201 201 201 201 201 201 201 201 429 201 429 429 429 429 429 429 429 429 429

$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep -E '^alias=|^alias_audit_log='     # AFTER burst
alias=34
alias_audit_log=36
```

**Exactly 10 requests returned `201` and 10 returned `429`, and the DB grew by exactly 10** (`alias 24 → 34`, `alias_audit_log 26 → 36`); the `429`s wrote **nothing**. Raw 429 response:

```
$ curl -sS -i -X POST http://localhost:7777/api/alias/random/new -H 'Authentication: code'
HTTP/1.1 429 TOO MANY REQUESTS
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 05:47:25 GMT
Connection: close
Content-Type: application/json
Content-Length: 32
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<redacted fresh anonymous session>; Expires=Wed, 15-Jul-2026 05:47:25 GMT; HttpOnly; Path=/; SameSite=Lax

{"error":"Rate limit exceeded"}
```

Log (the custom handler at `server.py:364`) — note it fired on **both** workers, which is the key to the count:

```
$ grep -E 'rate_limited|POST /api/alias/random/new .* 429' /tmp/scratch_qna/gunicorn_full.log | grep '05:47:03' | head -4
2026-07-08 05:47:03,230 - SL - WARNING - 2159 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/alias/random/new, user:<flask_login.mixins.AnonymousUserMixin object at 0x7a7822fb3340>
2026-07-08 05:47:03,230 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 429, takes 0.0006582736968994141
2026-07-08 05:47:03,272 - SL - WARNING - 2160 - "/app/server.py:364" - rate_limited() -  - Client hit rate limit on path /api/alias/random/new, user:<flask_login.mixins.AnonymousUserMixin object at 0x7a7822a8aef0>
2026-07-08 05:47:03,272 - SL - DEBUG - 2160 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([]) 429, takes 0.0006740093231201172
```

- **Why 10, not 5.** `flask‑limiter`'s in‑memory storage (used because `MEM_STORE_URI` is unset) is **per‑gunicorn‑worker**, and there are **2 workers** (§2.1). Each worker holds its own `5/minute` bucket, so the two together admit ~`2 × 5 = 10` before refusing — and the WARNING log confirms *both* workers (`2159` and `2160`) enforced the limit. The exact interleaving (`…201 201 201 429 201 429…`) reflects round‑robin distribution of the 20 requests across the two workers.
- **Key function** (`app/extensions.py:L14-L23`): the limiter keys by `userid:{id}` when `current_user.is_authenticated`, else `ip:{addr}`. On the API path there is no login session, so `current_user` is anonymous (the log's `user:<AnonymousUserMixin>` confirms this) and the bucket is keyed by **client IP**.

**Exactly TWO rate‑limit layers, plus a SEPARATE concurrency lock** (this corrects any "three rate‑limit layers" reading — the parallel limiter is a mutual‑exclusion lock, not a rate bucket):

| Control | Kind | Mechanism | Runtime state |
|---------|------|-----------|---------------|
| Rate‑limit layer 1 | **rate limit** | `flask‑limiter` `@limiter.limit(ALIAS_LIMIT)` (`app/config.py:L448`) | **Active**, in‑memory storage (no `MEM_STORE_URI`). Produced the observed `429`. |
| Rate‑limit layer 2 | **rate limit** | in‑`Alias.create` Redis token bucket `rate_limiter.check_bucket_limit` (`app/models.py:L1632-L1641`, `ALIAS_CREATE_RATE_LIMIT_{FREE,PAID}`) | **No‑op** — `rate_limiter.lock_redis is None`, so it returns immediately (`app/rate_limiter.py:L28-L29`). |
| Concurrency lock (not a rate limit) | **mutual exclusion** | `@parallel_limiter.lock(name="alias_creation")` (`app/parallel_limiter.py`) | **No‑op** — `parallel_limiter.lock_redis is None`, so the decorator is a pass‑through (`L51-L52`). |

Runtime proof of the two no‑ops:

```
$ docker exec sl-app bash -lc 'cd /app && venv/bin/python -c "
import app.rate_limiter as rl, app.parallel_limiter as pl
from app import config
print(\"rate_limiter.lock_redis =\", rl.lock_redis)
print(\"parallel_limiter.lock_redis =\", pl.lock_redis)
print(\"MEM_STORE_URI =\", config.MEM_STORE_URI)
print(\"DISABLE_RATE_LIMIT =\", config.DISABLE_RATE_LIMIT)
" 2>/dev/null | grep -E "="'
rate_limiter.lock_redis = None
parallel_limiter.lock_redis = None
MEM_STORE_URI = None
DISABLE_RATE_LIMIT = False
```

Both are `None` because `MEM_STORE_URI` is unset, so `server.py`'s Redis‑services init never wired them. `DISABLE_RATE_LIMIT=False`, so the active `flask‑limiter` layer is enabled (as the `429` proves). Since `john` is premium, the (no‑op) layer‑2 bucket would use the **PAID** thresholds if it were active (`app/models.py:L1634-L1641`) — but it is a no‑op here regardless.

> **"Network" failure mode — explicitly not exercised, with the reason.** The prompt's "network" failure class has **no genuine outbound‑socket surface on the default alias‑creation commit path**, so no true network‑I/O failure was triggered — this is stated as **not run**, not asserted as behavior. The evidence for the absence of an outbound dependency: (a) event egress via Postgres `NOTIFY` never fires because dispatch short‑circuits before `SyncEvent.create`/`NOTIFY` (`app/events/event_dispatcher.py:L61-L65`, unreached `L24-L26`; §6); (b) no email is sent (`NOT_SEND_EMAIL=true`, §2.2); (c) the Redis‑backed limiter/lock are no‑ops (`lock_redis is None`, above), so `Alias.create` performs **no network call** — its writes go only to the local Postgres over `sl-db:5432`. The nearest observable "too much traffic" surface is the throttling `429` shown above, which was exercised. **[The absence of an outbound network dependency is inferred from configuration + code; the throttling surface itself is observed.]**

### 7.6 Conditional writes (caveats) — when the "extra" tables would be touched

These are **not** part of the default 3‑table flow; each is gated and was either exercised or shown absent:

- **`alias_used_on` (API only, `hostname` supplied)** — **exercised.** `POST /api/alias/random/new?hostname=example.com` wrote one `alias_used_on` row (`app/api/views/new_random_alias.py:L109-L112`). Because `john` has `include_website_in_one_click_alias` enabled and owns custom domain `old.com`, the endpoint minted a website‑derived alias `example@old.com` (id 36) and recorded the originating hostname:

```
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep '^alias_used_on='   # BEFORE
alias_used_on=1
$ curl -sS -o /dev/null -w 'HTTP %{http_code}\n' -X POST 'http://localhost:7777/api/alias/random/new?hostname=example.com' -H 'Authentication: code'
HTTP 201
$ docker exec sl-app bash -lc 'bash /tmp/scratch_qna/probe.sh' | grep '^alias_used_on='   # AFTER (+1)
alias_used_on=2
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select id,alias_id,hostname,user_id from alias_used_on order by id desc limit 1;"
3|36|example.com|1
```

  This is an **API‑only** write and never occurs on the web random‑alias path.

- **`users` UPDATE (partner‑created flag)** — **not hit; observed absent.** Fires only if `new_alias.flags & FLAG_PARTNER_CREATED > 0 and user.flags & FLAG_CREATED_ALIAS_FROM_PARTNER == 0` (`app/models.py:L1663-L1667`). The seeded/free users are not partner‑created, so `users` stayed at zero‑delta in every run (`users=2` for john‑only runs, `users=3` after the free user existed, never changing on a creation). **[Read‑verified + observed‑absent.]**

- **`sync_event` row + `NOTIFY`** — **not hit; observed absent.** Occurs only when `EVENT_WEBHOOK` is set *and* the user is a partner user (`app/events/event_dispatcher.py:L61-L70`, write at `L24-L26`). With `EVENT_WEBHOOK` unset the second early return fires (§6), so `sync_event` stayed `0` on every creation. **[Read‑verified + observed‑absent.]**

### 7.7 Malformed `hostname` (API only) → HTTP 500, `EmailSyntaxError` (a validation‑failure edge)

**Direct answer:** a `?hostname=` value **whose extracted registrable domain still contains characters that are invalid in an email local‑part** (e.g. `<`, `>`, `(`, `)`) does **not** produce a controlled `4xx` on the API endpoint — it returns **HTTP `500`** `{"error":"Internal error"}` and writes **nothing** to the database. This is a genuine **validation‑failure** edge of the O8 surface: the hostname is turned into an invalid alias local‑part, and the resulting `email_validator.EmailSyntaxError` raised deep inside `Alias.create` is **not caught** by the endpoint — its `try/except` (`app/api/views/new_random_alias.py:L83-L90`) catches only `AliasInTrashError` (`L91-L93`), so the exception escapes and the global handler converts it to a `500`. It affects only the API path *and* only when the user has one‑click website aliases enabled (`user.include_website_in_one_click_alias`, `app/api/views/new_random_alias.py:L54`) — as `john` does (he owns custom domain `old.com`).

**Why only *this* class of malformed hostname (exhaustive, not every malformed value).** The endpoint derives the alias prefix by taking `tldextract.extract(hostname).domain` and passing it through `convert_to_id(...)` (`app/api/views/new_random_alias.py:L58-L60`), and `convert_to_id` (`app/utils.py:L50-L56`) only lowercases, unidecodes, and strips spaces — it does **not** remove `<`, `>`, `(`, `)`. So the `500` occurs precisely when those invalid characters land in the *domain* part that `tldextract` extracts. Observed directly:

```
$ docker exec sl-app bash -lc "cd /app && venv/bin/python -c \"
import tldextract; from app.utils import convert_to_id
for h in ['<script>alert(1)</script>\' OR 1=1--.example', '<script>bad<.evil']:
    d = tldextract.extract(h).domain
    print(repr(h), '-> domain=', repr(d), '-> convert_to_id=', repr(convert_to_id(d)))\""
"<script>alert(1)</script>' OR 1=1--.example" -> domain= '<script>alert(1)<' -> convert_to_id= '<script>alert(1)<'
'<script>bad<.evil' -> domain= 'evil' -> convert_to_id= 'evil'
```

The first keeps the invalid chars in the domain → invalid local‑part `<script>alert(1)<@old.com` → `EmailSyntaxError` → `500`. The second's bad chars fall in the *subdomain* (discarded), so `tldextract` yields a clean domain `evil`, the alias `evil@old.com` is valid, and the request **succeeds with `201`** — **not** a `500`. So this is *not* "any malformed hostname → 500"; it is specifically the class whose extracted domain retains email‑invalid characters.

For the `201` (clean‑domain) class, note the endpoint is **idempotent by hostname**: it computes a deterministic `suggested_alias` and only calls `Alias.create` when that alias does not already exist for the user; if it exists and has a matching `AliasUsedOn` row it is *reused* (`app/api/views/new_random_alias.py:L66-L80`, reuse logged at `L77`). Both sub‑cases observed through the real endpoint — a **first** call for a fresh registrable domain **creates** a row (`+1` on `alias`), while a **repeat** call for the same hostname **reuses** it (no new row), both returning `201`:

```
# First call — fresh registrable domain (bad char in subdomain, discarded): CREATES a row
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select count(*) from alias;"   # BEFORE
124
$ docker exec sl-app bash -lc "curl -sS -X POST \
    'http://localhost:7777/api/alias/random/new?hostname=%3Cbad%3E.qafreshcp5.com' -H 'Authentication: code'"
{"alias":"qafreshcp5@old.com","creation_date":"2026-07-08 10:26:46+00:00","creation_timestamp":1783506406,"disable_pgp":false,"email":"qafreshcp5@old.com","enabled":true,"id":127,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select count(*) from alias;"   # AFTER  (+1)
125

# Repeat call — same hostname whose alias already exists: REUSES it (no new row)
$ docker exec sl-db psql -U myuser -d simplelogin -tAc \
    "select id||'|'||alias_id||'|'||hostname from alias_used_on where alias_id=(select id from alias where email='evil@old.com');"
11|123|<script>bad<.evil                                   # existing AliasUsedOn for this hostname
$ docker exec sl-app bash -lc "curl -sS -X POST \
    'http://localhost:7777/api/alias/random/new?hostname=%3Cscript%3Ebad%3C.evil' -H 'Authentication: code'"
{"alias":"evil@old.com","creation_date":"2026-07-08 10:20:20+00:00","creation_timestamp":1783506020,"disable_pgp":false,"email":"evil@old.com","enabled":true,"id":123,"latest_activity":null,"mailbox":{"email":"john@wick.com","id":1},"mailboxes":[{"email":"john@wick.com","id":1}],"name":null,"nb_block":0,"nb_forward":0,"nb_reply":0,"note":null,"pinned":false,"support_pgp":false}
$ docker exec sl-db psql -U myuser -d simplelogin -tAc "select count(*) from alias;"   # AFTER (unchanged — reuse; note creation_date 10:20:20 predates this call)
125
```

Observed through the real HTTP endpoint (client `127.0.0.1`, inside `sl-app`; `slapp` cookie payload redacted per §2.7 convention):

```
$ docker exec sl-db psql -U myuser -d simplelogin -tAc \
  "select 'alias='||count(*) from alias union all select 'alias_audit_log='||count(*) from alias_audit_log \
   union all select 'alias_used_on='||count(*) from alias_used_on union all select 'sync_event='||count(*) from sync_event order by 1;"   # BEFORE
alias=120
alias_audit_log=124
alias_used_on=8
sync_event=0

$ docker exec sl-app bash -lc "curl -sS -i -X POST \
    'http://localhost:7777/api/alias/random/new?hostname=%3Cscript%3Ealert(1)%3C%2Fscript%3E%27%20OR%201%3D1--.example' \
    -H 'Authentication: code'"
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: gunicorn/20.0.4
Date: Wed, 08 Jul 2026 10:18:53 GMT
Connection: close
Content-Type: application/json
Content-Length: 27
Access-Control-Allow-Origin: *
Vary: Cookie
Set-Cookie: slapp=<redacted anonymous session cookie>

{"error":"Internal error"}

$ docker exec sl-db psql -U myuser -d simplelogin -tAc \
  "select 'alias='||count(*) from alias union all select 'alias_audit_log='||count(*) from alias_audit_log \
   union all select 'alias_used_on='||count(*) from alias_used_on union all select 'sync_event='||count(*) from sync_event order by 1;"   # AFTER (identical — zero write)
alias=120
alias_audit_log=124
alias_used_on=8
sync_event=0
```

The `500` body is produced by the global API error handler `@app.errorhandler(Exception)`, which logs the exception and returns `jsonify(error="Internal error"), 500` for any `/api/` path (`server.py:L389-L392`). The verbatim server log for this request shows the full chain — the hostname is accepted (`app/api/views/new_random_alias.py:L55`), an invalid `suggested_alias` (`<script>alert(1)<@old.com`) is built and passed to `Alias.create` (`L82`, call at `L84`), and `Alias.get_custom_domain` → `validate_email` raises:

```
$ docker exec sl-app bash -lc "awk '/10:18:53,703/,/POST \/api\/alias\/random\/new.*500/' /tmp/gunicorn.log"   # exact lines emitted by the request above
2026-07-08 10:18:53,703 - SL - DEBUG - 2159 - "/app/app/api/views/new_random_alias.py:55" - new_random_alias() -  - Use <script>alert(1)</script>' OR 1=1--.example to create new alias
2026-07-08 10:18:53,715 - SL - DEBUG - 2159 - "/app/app/api/views/new_random_alias.py:82" - new_random_alias() -  - create new alias <script>alert(1)<@old.com
2026-07-08 10:18:53,717 - SL - ERROR - 2159 - "/app/server.py:390" - error_handler() -  - The email address contains invalid characters before the @-sign: (, ), <, >.
Traceback (most recent call last):
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1950, in full_dispatch_request
    rv = self.dispatch_request()
  File "/app/venv/lib/python3.10/site-packages/flask/app.py", line 1936, in dispatch_request
    return self.view_functions[rule.endpoint](**req.view_args)
  File "/app/venv/lib/python3.10/site-packages/flask_limiter/extension.py", line 702, in __inner
    return obj(*a, **k)
  File "/app/app/api/base.py", line 58, in decorated
    return f(*args, **kwargs)
  File "/app/app/parallel_limiter.py", line 52, in decorated
    return f(*args, **kwargs)
  File "/app/app/api/views/new_random_alias.py", line 84, in new_random_alias
    alias = Alias.create(
  File "/app/app/models.py", line 1656, in create
    custom_domain = Alias.get_custom_domain(email)
  File "/app/app/models.py", line 1617, in get_custom_domain
    alias_domain = validate_email(
  File "/app/venv/lib/python3.10/site-packages/email_validator/__init__.py", line 223, in validate_email
    local_part_info = validate_email_local_part(parts[0],
  File "/app/venv/lib/python3.10/site-packages/email_validator/__init__.py", line 337, in validate_email_local_part
    raise EmailSyntaxError("The email address contains invalid characters before the @-sign: %s." % bad_chars)
email_validator.EmailSyntaxError: The email address contains invalid characters before the @-sign: (, ), <, >.
2026-07-08 10:18:53,718 - SL - DEBUG - 2159 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /api/alias/random/new ImmutableMultiDict([('hostname', "<script>alert(1)</script>' OR 1=1--.example")]) 500, takes 0.027210474014282227
```

- **Magnitude stability.** Reproduced identically across **≥2 runs** — `HTTP 500`, `Content-Length: 27`, body `{"error":"Internal error"}` every time, and the before/after DB probe is unchanged on every run (zero write to `alias`, `alias_audit_log`, `alias_used_on`, `sync_event`).
- **Contrast (this is the malformed input, not the `hostname` parameter in general).** A *valid* hostname on the same call succeeds: `?hostname=qafresh-cp5verify.com` → `HTTP 201` (and writes one `alias_used_on` row, per §7.6). So the `500` is specific to the malformed value, not to using `hostname`.
- **Root cause (read‑verified chain, corroborated by the traceback above):** `app/api/views/new_random_alias.py:L84` (`Alias.create`) → `app/models.py:L1656` (`custom_domain = Alias.get_custom_domain(email)`) → `app/models.py:L1617` (`validate_email(...)`, imported at `app/models.py:L17`) raises `EmailSyntaxError`; because the endpoint's `except` only handles `AliasInTrashError` (`app/api/views/new_random_alias.py:L91-L93`), the exception propagates to the global handler (`server.py:L389-L392`).
- **Read‑only note.** This is an observed property of the **unchanged** SimpleLogin source. Per this investigation's read‑only mandate, no source file was modified to "fix" it — the behavior is reported exactly as the running system exhibits it. (A production hardening would validate/normalize `hostname`, or catch `EmailSyntaxError` and return `400`; that is a source change outside this investigation's scope.)

---

## 8. Nuance, caveats, and configuration boundaries

### 8.1 The single most important gotcha — the happy‑path user is premium

`john@wick.com` is **premium** (active monthly subscription, `app/fake_data.py:L106-L119`), so `can_create_new_alias()` short‑circuits `True` (`app/models.py:L867`, `L746`) and the free‑plan cap **cannot** be observed with him. All cap testing used the runtime‑created free user `freebie@sl.local` (§7.1). Any reader reproducing this must not expect the seeded user to hit the cap.

### 8.2 Default‑configuration invariants (the boundary within which the "3 tables / no background work" answer holds)

- `EVENT_WEBHOOK` unset → event dispatch short‑circuits → **no `sync_event`, no `NOTIFY`, no background follow‑up** (§6).
- `MEM_STORE_URI` unset → Redis limiters not initialized → **rate‑limit layer 2** (the in‑`Alias.create` bucket) and the **`alias_creation` parallel concurrency lock** are **no‑ops**; only **rate‑limit layer 1** (`flask‑limiter`, in‑memory) is active (§7.5). The HTTP‑level controls (CSRF, `@login_required`, API‑key auth, `flask‑limiter`) are genuinely in force; the two Redis‑backed decorators are present on the code path but do nothing.
- Seeded user has a single default mailbox → **no `alias_mailbox`** row (§5.5).
- `NOT_SEND_EMAIL=true` → no outbound email is transmitted on any path.

### 8.3 When the conditional writes *would* occur (outside the default flow)

- **4th table `alias_used_on`** — API path only, when `?hostname=` is supplied (§7.6). Exercised: `201` + one `alias_used_on` row.
- **`users` UPDATE** — only for partner‑created aliases (`FLAG_PARTNER_CREATED`), `app/models.py:L1663-L1667`. Not reachable in the default flow.
- **`sync_event` + `NOTIFY`** — only when `EVENT_WEBHOOK` is set *and* the user is a partner user, `app/events/event_dispatcher.py:L61-L70` / `L24-L26`. Not reachable in the default flow.
- **API‑transport `api_key` update** — the API auth decorator bumps `api_key.times`/`last_used` (`app/api/base.py:L30-L32`); this is auth bookkeeping, not part of the alias‑creation core, and is absent on the web path (§5.4).

### 8.4 Repository‑unchanged & cleanup verification

All observation scripts lived under `/tmp/scratch_qna` inside the `sl-app` container (host backup `/tmp/qna_capture`), never in the repository tree, and were deleted at completion; the app's own `/tmp/gunicorn.log` inside the container was read but left intact. This answer document is committed on the working branch, so repository integrity is proven by diffing the **entire working tree** against the last upstream **source** commit (`2cd6ee77`, the parent of the documentation commit): the only path that differs is this document, and **no source file changed**. Captured live at cleanup:

```
$ git status --porcelain                                  # clean working tree after committing the answer document (no output)

$ git diff --name-status 2cd6ee77                         # entire delta vs the last upstream SOURCE commit
A	blitzy/documentation/app_2cd6ee777f8c.md

$ git diff --name-only 2cd6ee77 | grep -vE '^blitzy/'     # any SOURCE (.py/.html/.js/.env/…) file changed? (no output = none)

$ git submodule status                                    # (no .gitmodules; no submodules — no output)
```

That is: relative to the last upstream source commit, the working tree adds exactly one file — this document under `blitzy/documentation/` — and every tracked source file is byte‑for‑byte identical.

> The running database was mutated during observation (new aliases, a runtime free user, a trashed alias, incremented API‑key counters, a temporarily deleted `daily_metric` row). That is ephemeral runtime state; **no tracked repository file was modified, added, or deleted** other than the answer document.

---

## 9. Coverage pass — every named item answered

Re‑reading the original question and confirming each named item is addressed:

| Named item asked | Answered in | Direct result |
|------------------|-------------|---------------|
| **Frontend request** | §3 | `application/x-www-form-urlencoded` `POST /dashboard/` with `form-name=create-random-email` + `csrf_token` (+ optional `generator_scheme`) |
| — its status codes / payloads / metadata (of response) | §4 | web `302` (+ `Location`, flash cookie); API `201` (+ JSON, headers) |
| **Backend response** | §4 | web `302` redirect; API `201` + 17‑key JSON, every key mapped to a serializer line |
| **Database changes** | §5 | 3‑table creation core: `alias` INSERT, `daily_metric` INSERT/UPDATE, `alias_audit_log` INSERT (API adds `api_key` auth write) |
| — which records inserted/updated | §5.5, §5.3, §5.4 | new `alias` + `alias_audit_log` rows shown; `daily_metric` INSERT‑vs‑UPDATE shown; `api_key` counter update shown |
| — how many tables touched | §5.1–§5.4 | **exactly 3** in the creation core, stable across ≥2 runs; +1 (`api_key`) on API transport |
| — are related entities created | §5.6, §7.6 | not in default flow (`alias_mailbox`/`users`/`sync_event`/`alias_used_on` all +0); `alias_used_on` only for API `hostname` |
| **Background tasks / follow‑up events / additional work** | §6 | none in default config; creation `LOG.d` + event short‑circuit `LOG.i`; no `NOTIFY`, no job; both workers run and idle |
| — inspect logs | §6.1, throughout | raw log slices shown for creation and every error branch |
| **Failure — validation** | §7.3, §7.4, §7.7 | invalid CSRF → `302` + "Invalid request"; invalid `mode` → `400`; malformed API `?hostname=` whose extracted domain keeps email‑invalid chars → `500` (uncaught `EmailSyntaxError`), zero write |
| **Failure — DB** | §7.2 | trashed‑alias reuse → `AliasInTrashError` (caught → `201` random fallback / custom → `409`) |
| **Failure — network** | §7.5 | rate limit → `429` (observed); true socket failure **not run** (no outbound dependency on the commit path — evidenced) |
| — free‑plan limit (an implied failure) | §7.1 | API `400`, web `302` + upgrade flash; zero DB write; free‑user setup shown |
| — how each error surfaces in response / logs / DB | §7 (all) | each branch shows raw response + log line + before/after DB probe |

All "e.g. / such as / including" items (status codes, payloads, metadata; records inserted/updated; table count; related entities; validation + DB + network failure modes) are covered above.

---

## 10. Evidence index (`file:line` references used)

- **Entry points & app factory:** `server.py:L139` (`create_app`), `server.py:L251-L255` (root `/` redirect), `server.py:L284` (`after_request` request log), `server.py:L364` (`rate_limited` 429 handler), `server.py:L494` (`dummy_data`), `server.py:L588` (dev bind 7777); `wsgi.py:L1-L3` (gunicorn entry importing `create_app`).
- **Auth (web/login):** `app/auth/views/login.py:L21-L25` (route); `app/auth/views/login_utils.py:L35-L45` (`after_login`, `login_user` + dashboard redirect).
- **Web alias handler:** `app/dashboard/views/index.py:L55-L121` — CSRF fail `L88-L90`; create‑random branch `L97`; `can_create_new_alias` guard `L98`; `Alias.create_new_random` `L104`; `mailbox_id` `L106`; `Session.commit` `L108`; creation `LOG.d` `L110`; success flash `L111`; `302` redirect `L113-L121`; upgrade‑flash else branch `L123`.
- **Dashboard blueprint prefix:** `app/dashboard/base.py:L3-L8` (`Blueprint(..., url_prefix="/dashboard")`).
- **Frontend template:** `templates/dashboard/index.html:L50-L58` (main button), `L68-L76` (word), `L78-L86` (uuid).
- **JSON API (random):** `app/api/views/new_random_alias.py` — route `L21`, `@limiter.limit` `L22`, `@require_api_auth` `L23`, `@parallel_limiter.lock` `L24`; free‑plan `400` guard `L35-L43` (log `L36`); hostname branch `L53-L93` (`Alias.create` `L82-L90`, `except AliasInTrashError` `L91-L93`, "is in trash" log `L92`); `mode` `400` `L98-L104`; `Session.commit` `L107`; `alias_used_on` `L109-L112`; `201` `L114-L116` (top‑level `alias` `L115`).
- **JSON API (custom):** `app/api/views/new_custom_alias.py:L83-L88` (trash pre‑check → `409`, log `L87`).
- **API auth:** `app/api/base.py:L16-L34` (`Authentication` header `L17`, `ApiKey.get_by` `L18`, `last_used` `L30`, `times += 1` `L31`, `Session.commit` `L32`, `g.user` `L34`).
- **Serializer:** `app/api/serializer.py:L55-L93` (`serialize_alias_info_v2`; dict literal `L56-L79`, keys `id`=`L58` … `pinned`=`L78`; `latest_activity` populate block `L80-L92`).
- **Models:** `app/models.py` — `Alias.create` `L1628-L1692` (`AliasInTrashError` `L1647-L1652`, `alias` INSERT `L1660`, `daily_metric` `L1661`, partner‑flag `users` `L1663-L1667`, event build `L1680-L1687`, `send_event` `L1687`, `emit_alias_audit_log` `L1688-L1690`); `create_new_random` `L1721`; `lifetime_or_active_subscription` `L746`; `is_premium` `L787`; `max_alias_for_free_account` `L858`; `can_create_new_alias` `L867`; `Disable onboarding emails` `L647`; `DailyMetric.get_or_create_today_metric` `L3280`; `generate_random_alias_email` `L1459`; `AliasAuditLog` `L3810`; `AliasMailbox` `L2939`; rate bucket select `L1632-L1641`.
- **Audit utils:** `app/alias_audit_log_utils.py:L8` (`CreateAlias="create"`), `L18-L32` (`emit_alias_audit_log`).
- **Event pipeline:** `app/events/event_dispatcher.py:L24-L26` (`SyncEvent.create` + `NOTIFY`), `L49-L84` (`send_event` three early returns; the one that fires `L61-L65`, log at `L62`); `proto/event.proto` / `app/events/generated/event_pb2.py` (`AliasCreated` fields).
- **Alias delete (trash setup):** `app/api/views/alias.py:L152-L173` (`DELETE`), `app/alias_utils.py:L336-L372` (`delete_alias` → `DeletedAlias`/`DomainDeletedAlias`; deleted‑log `L346`, domain‑trash move `L360`).
- **Rate limiting / concurrency:** `app/config.py:L448` (`ALIAS_LIMIT`), `L554-L558` (`ALIAS_CREATE_RATE_LIMIT_*`); `app/extensions.py:L14-L23` (limiter key func `L14-L20`, `limiter` `L23`); `app/rate_limiter.py:L28-L29` (no‑op guard); `app/parallel_limiter.py:L51-L52` (no‑op guard).
- **Config / seeding:** `app/config.py:L124` (`MAX_NB_EMAIL_FREE_PLAN`), `L568` (`MEM_STORE_URI`), `L602` (`DISABLE_RATE_LIMIT`), `L612` (`EVENT_WEBHOOK`), `L616` (`EVENT_WEBHOOK_DISABLE`); `app/fake_data.py:L44-L55` (john), `L106-L119` (subscriptions), `L121-L125` (API keys `code`/`codeFF`), `L155-L157` (`AliasMailbox` extra‑mailbox only); `init_app.py:L44` (`add_sl_domains`).
- **Background workers:** `event_listener.py:L29-L47` (`main`; `PostgresEventSource` `L34`, `ConsoleEventSink` `L40`), `events/event_source.py:L49` (`__listen`); `job_runner.py:L188` (`process_job`), `L295-L302` (`JOB_SEND_ALIAS_CREATION_EVENTS` branch), `L329-L347` (`__main__` loop, `time.sleep(10)` `L347`); `app/jobs/event_jobs.py:L9-L52` (`send_alias_creation_events_for_user` backfill).
- **Logging:** `app/log.py:L74-L77` (`LOG.d/i/w/e` shortcuts), `L79` (`LOG = _get_logger("SL")`).

---

*End of investigation. This document is the sole persistent artifact; all temporary observation scripts were removed and the repository left byte‑for‑byte unchanged apart from this file.*
