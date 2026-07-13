# SimpleLogin Self-Host Runtime Verification — `app_2cd6ee777f8c`

**Runtime-verification Q&A report.** This document confirms, by **actually booting** a local
[SimpleLogin](https://github.com/simple-login/app) instance in its **default configuration** and
**observing live runtime behavior**, that a first-time self-hosted deployment is healthy across
**(a) user authentication** and **(b) alias-based email activity**. Every behavioral claim below is
paired with the exact command that produced it, the actual (unedited) captured output, and a
`file:line` citation naming the specific function/method. Statements not confirmed at runtime are
explicitly labeled **inferred**.

The report answers exactly **three questions**:

1. **Objective 1 — Startup readiness signals.** What appears after starting the system that confirms
   it is ready to handle user authentication and alias-based email activity?
2. **Objective 2 — New-user product walkthrough.** Register → verify → login → dashboard, with the
   visible confirmation at each transition (plus the error/edge states).
3. **Objective 3 — Behind-the-scenes background jobs / internal services.** The runtime indicators
   that background jobs and internal services are active and communicating.

---

## TL;DR — Health Verdict

**HEALTHY.** All three runtime tiers of SimpleLogin were booted in the canonical local configuration
and confirmed live:

| Tier | Entry point | Readiness signal (observed) | Verdict |
|------|-------------|------------------------------|---------|
| Web / auth | `python server.py` (`:7777`) | Flask banner + `Debug mode: on`; live HTTP 200/302; `auth`/`dashboard` blueprints mounted | ✅ live |
| Email / alias | `python email_handler.py` (`:20381`) | `Listen for port 20381` + `Start mail controller 0.0.0.0 20381`; SMTP banner `220 ... Python SMTP 1.4.2` | ✅ live |
| Background jobs | `python job_runner.py` | 10-second poll loop; `Take job` + `Job.state` `ready→taken→done` | ✅ live |
| Scheduler | `python cron.py -j <job>` | `Start running cronjob`; 15-entry yacron schedule in `crontab.yml` | ✅ live |
| Event listener | `python event_listener.py listener` | `Using PostgresEventSource`; Postgres `LISTEN/NOTIFY` channel functional | ✅ live |
| Datastore | PostgreSQL 13 | schema migrated to head `32f25cbf12f6`; 77 public tables | ✅ live |

- **User authentication** was exercised end-to-end over real HTTP: register → activation email
  (printed to log under the default) → verify (`User.activated` flips `False`→`True`) → login (HTTP
  302 into the dashboard) → authenticated dashboard render. All four error/edge paths reproduced.
- **Alias-based email activity** was exercised through the real aiosmtpd SMTP controller on `:20381`
  via `swaks`; the full forward pipeline (`handle()` → `handle_forward()` →
  `forward_email_to_mailbox()`) executed and returned `250 Message accepted for delivery`.
- **Background jobs / internal services** were confirmed live: the GDPR export job was picked up by
  `job_runner.py`, the Postgres `LISTEN/NOTIFY` substrate was demonstrated functional, and the
  event dispatcher's default no-op guard was observed firing.

Four runtime nuances/discrepancies were observed against a naive expectation and are documented
inline (see §6): the activation "email" is a **log line** (not a sent message), onboarding jobs are
**suppressed** by default, event dispatch is a deliberate **no-op** under the default config, and
the Werkzeug `* Running on ...` banner is **suppressed** because the `werkzeug` logger is disabled.

---

## 2. Environment & Exact Commands

### 2.1 Canonical stack (fixed versions)

The canonical prepared `python:3.10` Docker image was used (per the project setup instructions:
`ghcr.io/scaleapi/swe-atlas … simple-login__app__2cd6ee777f8c…`, derived runtime image
`sl-app:runtime`), where the locked dependency set installs cleanly. Versions were reproduced at
runtime:

```
$ docker exec sl-web /app/venv/bin/python -c "import sys,flask,werkzeug,aiosmtpd,sqlalchemy,redis,arrow; \
    print('python', sys.version.split()[0]); print('flask', flask.__version__); \
    print('werkzeug', werkzeug.__version__); print('aiosmtpd', aiosmtpd.__version__); \
    print('sqlalchemy', sqlalchemy.__version__); print('redis', redis.__version__); print('arrow', arrow.__version__)"
python 3.10.18
flask 1.1.2
werkzeug 1.0.1
aiosmtpd 1.4.2
sqlalchemy 1.3.24
redis 4.6.0
arrow 0.16.0
```

Fixed stack: **Python 3.10**, **PostgreSQL 13**, **Redis 6**
[`.github/workflows/main.yml`: python `3.10` L17/L40; `image: postgres:13` L47; `redis-version: 6`
L92-94].

### 2.2 Infrastructure (sidecar containers)

```
# PostgreSQL 13 (published to host 127.0.0.1:15432 -> container 5432)
docker run -d --name sl-postgres -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword \
    -e POSTGRES_DB=simplelogin -p 127.0.0.1:15432:5432 postgres:13
# Redis 6
docker run -d --name sl-redis -p 127.0.0.1:6379:6379 redis:6
```

Live connectivity confirmed:

```
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -tAc "SELECT version();"
PostgreSQL 13.23 (Debian 13.23-1.pgdg120+1) on x86_64-pc-linux-gnu, compiled by gcc ...
$ redis-cli -p 6379 PING
PONG
$ curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://localhost:7777/
302 http://localhost:7777/auth/login
$ printf 'QUIT\r\n' | nc 127.0.0.1 20381 | head -1
220 reverse-code-generator-71c836b7-xjgzz Python SMTP 1.4.2
```

### 2.3 Configuration (default `example.env`)

`.env` was created from `example.env` (`cp example.env .env`; git-ignored). The **only** deviation
made by the environment setup is the Postgres port (`5432`→`15432`) so one Postgres serves both dev
and the test suite. All pivotal defaults are unchanged and were confirmed at runtime:

| Key | Value | `example.env` line |
|-----|-------|--------------------|
| `URL` | `http://localhost:7777` | L6 |
| `NOT_SEND_EMAIL` | `true` | L19 |
| `EMAIL_DOMAIN` | `sl.local` | L22 |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:15432/simplelogin` | L75 (port set to 15432 by setup) |
| `FLASK_SECRET` | `secret` | L77 |
| `LOCAL_FILE_UPLOAD` | `true` | L136 |
| `DISABLE_ONBOARDING` | `true` | L150 |

`EVENT_WEBHOOK` defaults to `None` (unset) and `EVENT_WEBHOOK_DISABLE` defaults to `False`
[`app/config.py`]. `MEM_STORE_URI` defaults to `None` [`app/config.py:L568`].

**Two defaults change *what* the observable signals are — this report reflects the DEFAULT behavior:**

- `NOT_SEND_EMAIL=true` → the activation email is **printed to the log**, not sent. The log line **is**
  the local verification signal [`app/mail_sender.py:L130-137`].
- `DISABLE_ONBOARDING=true` → onboarding jobs are **suppressed** at registration
  [`app/models.py:L646-648`].

### 2.4 Migrate + seed (one-off)

```
# schema migration (config: alembic.ini)
docker exec -w /workspace sl-web /app/venv/bin/alembic upgrade head
# seed demo data -> creates john@wick.com / password + seed aliases (e0..e2@sl.local, etc.)
docker exec -w /workspace -e FLASK_APP=wsgi:app sl-web /app/venv/bin/flask dummy-data
```

Migration head confirmed live (idempotent — no pending migrations):

```
$ docker exec -w /workspace sl-web /app/venv/bin/alembic current
32f25cbf12f6 (head)
$ docker exec -w /workspace sl-web /app/venv/bin/alembic heads
32f25cbf12f6 (head)
```

### 2.5 Application entry points (per service)

Each service runs the canonical entry point via the prepared virtualenv interpreter, with the repo
mounted at `/workspace`:

```
# Web app (:7777)             CMD=["server.py"]        -> python server.py
# Email handler (:20381)      CMD=["email_handler.py"] -> python email_handler.py
# Background job runner        CMD=["job_runner.py"]    -> python job_runner.py
docker run -d --name sl-web  --network host -v <REPO>:/workspace -w /workspace \
    -e PATH=/app/venv/bin:/usr/local/bin:/usr/bin:/bin -e VIRTUAL_ENV=/app/venv \
    -e PYTHONUNBUFFERED=1 --entrypoint /app/venv/bin/python sl-app:runtime server.py
```

Verification tooling used from the host: `curl` / Python `requests` (HTTP to `:7777`), `swaks`
(SMTP to `:20381`), `psql` (Postgres), `redis-cli` (Redis).

### 2.6 Environment caveats (documented, NOT "fixed")

- **`cbor2 5.2.0` sdist build failure on newer base OSes** [`poetry.lock:L411-412`] — broken PEP 517
  metadata plus a `pkg_resources` import that newer setuptools drops. This is an **environment
  caveat only**, resolved by using the canonical `python:3.10` image; `pyproject.toml`/`poetry.lock`
  were **never** edited.
- **Postgres port reconciliation** — `CONTRIBUTING.md:L100` maps host `15432`→container `5432`, while
  `example.env:L75` `DB_URI` uses `localhost:5432`. In this environment the setup pointed `DB_URI` at
  `15432` so PostgreSQL is reachable where `DB_URI` points. This is a setup detail, **not** a
  SimpleLogin defect.

---

## 3. Objective 1 — Startup Readiness Signals

**Direct answer.** After starting the three entry points, readiness for **(a) user authentication**
and **(b) alias-based email activity** is confirmed by these observed signals: the logging init
banner `>>> init logging <<<`; the Flask dev-server banner ending in `* Debug mode: on` with live
HTTP responses on `:7777`; the migrated PostgreSQL schema at head `32f25cbf12f6`; the `auth` and
`dashboard` blueprints answering over HTTP; and the email handler's `Listen for port 20381` +
`Start mail controller 0.0.0.0 20381` with a live SMTP banner on `:20381`.

All three logs below were captured from a **fresh restart** (`docker restart sl-web sl-email
sl-jobs`) so each boot is clean and self-contained.

### 3.1 Web app — `python server.py` (`:7777`)

Command and complete captured startup log (`docker logs sl-web` from the fresh StartedAt):

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/wfcdyytwwmcpssqxivbj
Upload files to local dir
>>> init logging <<<
2026-07-13 17:03:40,555 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/snwdlbxtyvnfqgiievlo
Upload files to local dir
>>> init logging <<<
2026-07-13 17:03:42,381 - SL - DEBUG - 12 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-07-13 17:04:12,191 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0011053085327148438
```

Signals identified:

- **Logging init banner** — `print(">>> init logging <<<")` at import time [`app/log.py:L67`]. The
  application logger is named `SL` [`app/log.py:L79`] and the log format
  (`%(asctime)s - %(name)s - %(levelname)s - %(process)d - "%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s`)
  is defined at [`app/log.py:L12-14`] — visible in every `SL` line above.
- **Two `>>> init logging <<<` banners** — the Werkzeug reloader (`debug=True`) spawns a child; the
  parent runs as `process 1` and the reloader child as `process 12` (see the `DEBUG - 1` vs
  `DEBUG - 12` process ids). Each imports the app once, hence two banners. (Observed.)
- **Flask dev-server banner** — `create_app()` [`server.py:L139`]; when run directly, `__main__`
  [`server.py:L598`] calls `local_main()` [`server.py:L572`] which invokes
  `app.run(debug=True, port=7777)` [`server.py:L588`]. The Flask 1.1.2 click banner
  (`Serving Flask app "server"` … `* Debug mode: on`) prints, confirming the dev server started on
  the configured port.
- **PostgreSQL connectivity (migrated schema)** — `SQLALCHEMY_DATABASE_URI = DB_URI`
  [`server.py:L146`]. Proof the schema is migrated and queryable:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -tAc \
    "SELECT (SELECT version_num FROM alembic_version), (SELECT count(*) FROM information_schema.tables WHERE table_schema='public');"
32f25cbf12f6|77
$ docker exec sl-postgres psql -U myuser -d simplelogin -tAc "SELECT id,email,activated FROM users ORDER BY id;"
1|john@wick.com|t
2|winston@continental.com|t
```

- **Redis connectivity (observed nuance).** The guard `if MEM_STORE_URI:` [`server.py:L163`] →
  `initialize_redis_services(app, MEM_STORE_URI)` [`server.py:L165`] governs Redis wiring. Under the
  default `.env`, `MEM_STORE_URI` is unset and defaults to `None` [`app/config.py:L568`], so the
  guard is **False** and the web app does **not** connect to Redis by default — flask-limiter uses an
  in-memory store and Flask sessions are signed cookies. Redis 6 is nonetheless provisioned and live
  for parity/the full test suite. Observed:

```
$ docker exec sl-web /app/venv/bin/python -c "from app.config import MEM_STORE_URI; print('MEM_STORE_URI repr =', repr(MEM_STORE_URI))"
MEM_STORE_URI repr = None
$ redis-cli -p 6379 DBSIZE
(integer) 1115
```

- **Blueprint registration exposing `auth`/`dashboard`** — `register_blueprints()` [`server.py:L233`]
  registers `auth_bp` [`server.py:L234`], `dashboard_bp` [`server.py:L236`], plus
  `monitor`/`developer`/`phone`/`oauth` (`/oauth` L240 + `/oauth2` L241)/`onboarding`/`discover`/
  `internal`/`api` (through `server.py:L246`). Live proof over HTTP:

```
$ curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://localhost:7777/auth/login
200
$ curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://localhost:7777/dashboard/
302 http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F
$ curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://localhost:7777/
302 http://localhost:7777/auth/login
```

`GET /auth/login` → 200 (the `auth` blueprint renders the login page); `GET /dashboard/` → 302 to
`/auth/login?next=…` (the `dashboard` blueprint is mounted and its `@login_required` guard
[`app/dashboard/views/index.py:L56`] redirects the unauthenticated request). Each request is logged
by SimpleLogin's own `after_request` hook [`server.py:L284`], e.g.
`127.0.0.1 GET / ImmutableMultiDict([]) 302, takes …` in the boot log above.

### 3.2 Email handler — `python email_handler.py` (`:20381`)

Complete captured startup log:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/lonmaecfknemclxkcnoa
Upload files to local dir
>>> init logging <<<
2026-07-13 17:03:50,814 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
2026-07-13 17:03:51,522 - SL - INFO - 1 - "/workspace/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-13 17:03:51,524 - SL - DEBUG - 1 - "/workspace/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

Signals identified:

- **`Listen for port 20381`** — `LOG.i("Listen for port %s", args.port)` [`email_handler.py:L2403`];
  the argparse default port is `20381` [`email_handler.py:L2399`].
- **`Start mail controller 0.0.0.0 20381`** —
  `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)`
  [`email_handler.py:L2386`], emitted by `main(port)` [`email_handler.py:L2381`] after constructing
  `Controller(MailHandler(), hostname="0.0.0.0", port=port)` [`email_handler.py:L2383`]. This is the
  readiness signal for **alias-based email activity**: the aiosmtpd controller is bound and accepting
  SMTP on `:20381`. Confirmed by the live SMTP banner in §2.2 (`220 … Python SMTP 1.4.2`, aiosmtpd
  1.4.2).
- **PGP not loaded at boot (observed nuance, default config).** `load_pgp_public_keys()`
  [`email_handler.py:L2390`] is gated by `if LOAD_PGP_EMAIL_HANDLER:` [`email_handler.py:L2388`], and
  `LOAD_PGP_EMAIL_HANDLER = "LOAD_PGP_EMAIL_HANDLER" in os.environ` [`app/config.py:L339`] is `False`
  by default — so no `Finish load_pgp_public_keys` line is emitted here. The process then loops
  `while True: time.sleep(2)` [`email_handler.py:L2392-2393`].

### 3.3 Background job runner — `python job_runner.py`

Complete captured startup log:

```
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/kikhtiqxkxkfoyefkqed
Upload files to local dir
>>> init logging <<<
2026-07-13 17:04:00,806 - SL - DEBUG - 1 - "/workspace/app/utils.py:17" - <module>() -  - load words file: /workspace/local_data/test_words.txt
```

After the banner + word-list load, the `__main__` `while True` loop [`job_runner.py:L330`] runs
inside `create_light_app().app_context()` [`job_runner.py:L332`]; with no ready jobs
(`get_jobs_to_run()` [`job_runner.py:L307`] returns none) it logs nothing and sleeps 10 seconds
[`job_runner.py:L347`]. The loop actively picking up work (`Take job …`) is demonstrated in
**Objective 3 §5.1**.

### 3.4 One-time data init (context)

`init_app.local_init` runs `load_pgp_public_keys()` [`init_app.py:L13`] (logging
`Finish load_pgp_public_keys` [`init_app.py:L36`]), `add_sl_domains()` [`init_app.py:L39`], and
`add_proton_partner()` [`init_app.py:L59`]. **Observed:** these are invoked from the
`@app.cli.command("dummy-data")` command [`server.py:L490-497`] during `flask dummy-data`, **not** at
web-server boot — `Finish load_pgp_public_keys` appears **0** times in all three boot logs above.


---

## 4. Objective 2 — New-User Walkthrough (register → verify → login → dashboard)

**Direct answer.** The typical first-time experience works end-to-end. A fresh user registers via
`POST /auth/register` (HTTP 200, "waiting for activation" screen); the activation email is **printed
to the log** under `NOT_SEND_EMAIL=true` with subject `Just one more step to join SimpleLogin`;
`GET /auth/activate?code=…` flips `User.activated` from **`False`→`True`**, flashes
`Your account has been activated`, and issues an **HTTP 302** to `/dashboard/`; `POST /auth/login`
returns an **HTTP 302** to `/dashboard/`; and the authenticated `GET /dashboard/` renders **200**.
All four error/edge states were also reproduced.

All evidence below came over **real HTTP** to `http://localhost:7777` (a Python `requests` session
with a cookie jar and Flask-WTF CSRF tokens — canonical). Two temporary users were used:
`USER_A = blitzyqa-a-1783962716@gmail.com` (happy path) and
`USER_B = blitzyqa-b-1783962716@gmail.com` (edge paths); password `Blitzy-Test-Pw-123`.

> **Note on the email domain (observed).** Registration validates the address via
> `email_can_be_used_as_mailbox()` [`app/email_utils.py:L569`], which requires resolvable MX records
> unless `SKIP_MX_LOOKUP_ON_CHECK` is set — and it is `False` by default [`app/config.py:L600`]. A
> `@gmail.com` address was therefore used because it resolves real MX; `NOT_SEND_EMAIL=true` means no
> mail actually leaves. The activation code is **not** logged (only the subject/from/to are), so it
> was read canonically from the `activation_code` table and then the **real** `GET /auth/activate`
> HTTP route was driven.

### 4.1 Happy path — Register [`app/auth/views/register.py:L32`]

HTTP transcript:

```
GET  /auth/register -> HTTP 200; csrf_token present=True
DB users(USER_A) BEFORE register: ''
POST /auth/register -> HTTP 200
  body contains 'register_waiting_activation' markers: True
DB users(USER_A) AFTER register (id|activated): '3|f'
DB activation_code(user_id=3) code= 'qebbdvxpgxlkzcvkoxistenbvgqxiz' len=30
```

Server-side log (`sl-web`) for the register call:

```
2026-07-13 17:11:56,659 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes 0.057295799255371094
2026-07-13 17:11:56,799 - SL - DEBUG - 12 - "/workspace/app/auth/views/register.py:85" - register() -  - create user blitzyqa-a-1783962716@gmail.com
2026-07-13 17:11:57,166 - SL - INFO - 12 - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-13 17:11:57,177 - SL - DEBUG - 12 - "/workspace/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-13 17:11:57,238 - SL - DEBUG - 12 - "/workspace/app/email_utils.py:303" - send_email() -  - send email to blitzyqa-a-1783962716@gmail.com, subject 'Just one more step to join SimpleLogin'
2026-07-13 17:11:57,240 - SL - DEBUG - 12 - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzyqa-a-1783962716@gmail.com'
2026-07-13 17:11:57,607 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.9125099182128906
```

Mapping to code:

- `register()` [`register.py:L32`] logs `create user %s` [`register.py:L85`] → observed
  `create user blitzyqa-a-1783962716@gmail.com`.
- On success it calls `send_activation_email()` [`register.py:L95`] and renders
  `auth/register_waiting_activation.html` [`register.py:L104`] → HTTP 200 with the
  `register_waiting_activation` marker.
- **Activation email printed to log (default `NOT_SEND_EMAIL=true`)** — `MailSender.send()`
  [`app/mail_sender.py:L126`] hits `if config.NOT_SEND_EMAIL:` [`app/mail_sender.py:L130`], logs
  `send email with subject '%s', from '%s' to '%s'` [`app/mail_sender.py:L131-135`] and returns
  `True` [`app/mail_sender.py:L137`] — no real send. The subject
  `Just one more step to join SimpleLogin` [`app/email_utils.py:L128`; `send_activation_email()`
  begins at `app/email_utils.py:L125`] **is the local verification signal**, captured verbatim above.
- The activation code originates from `ActivationCode.create(user_id=user.id, code=random_string(30))`
  [`register.py:L120`] and is embedded in
  `activation_link = f"{URL}/auth/activate?code={activation.code}"` [`register.py:L124`]. Observed
  code `qebbdvxpgxlkzcvkoxistenbvgqxiz` is exactly 30 chars.
- The `Disable onboarding emails` [`app/models.py:L647`] and event-dispatch no-op
  [`app/events/event_dispatcher.py:L62`] lines are cross-referenced in Objective 3 (§5.2, §5.3).

### 4.2 Happy path — Verify [`app/auth/views/activate.py:L17`] — `User.activated` False→True

HTTP transcript (state captured **before** and **after** the GET):

```
User.activated BEFORE activate: 'f' (expect f)
GET  /auth/activate?code=<30 chars> -> HTTP 302
  Location header: http://localhost:7777/dashboard/
User.activated AFTER activate: 't' (expect t)
GET  /dashboard/ (session A, authed) -> HTTP 200
  flash 'Your account has been activated' present on dashboard: True
```

Server-side log:

```
2026-07-13 17:11:57,805 - SL - DEBUG - 12 - "/workspace/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.delint275@sl.local, subject 'Welcome to SimpleLogin'
2026-07-13 17:11:57,808 - SL - DEBUG - 12 - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.delint275@sl.local'
2026-07-13 17:11:57,809 - SL - DEBUG - 12 - "/workspace/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
2026-07-13 17:11:57,809 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'qebbdvxpgxlkzcvkoxistenbvgqxiz')]) 302, takes 0.08571839332580566
2026-07-13 17:11:57,858 - SL - DEBUG - 12 - "/workspace/app/dashboard/views/index.py:172" - index() -  - Show intro to <User 3 blitzyqa-a-1783962716@gmail.com blitzyqa-a-1783962716@gmail.com>
2026-07-13 17:11:58,243 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.3927128314971924
```

Mapping to code (state transition captured **before / during / after**):

- **Before:** `User.activated` is `'f'`. The column is a Boolean defaulting to `False`
  [`app/models.py:L358`].
- **During:** `activate()` [`activate.py:L17`] reads `code` [`activate.py:L24`], looks it up
  [`activate.py:L26`], sets **`user.activated = True`** [`activate.py:L49`], calls `login_user()`
  [`activate.py:L50`], deletes the code [`activate.py:L53`], flashes
  `Your account has been activated` (category `success`) [`activate.py:L56`], sends the welcome email
  [`activate.py:L58`] (observed: welcome email to `simplelogin-newsletter.delint275@sl.local` — which
  proves `User.create()` provisioned the newsletter alias, see §4.6), and **redirects (HTTP 302)** to
  the next URL or `dashboard.index` [`activate.py:L61-67`]; the `redirect user to dashboard` log is
  from [`activate.py:L66`].
- **After:** `User.activated` is `'t'`. The subsequent authenticated `GET /dashboard/` renders **200**
  and the `Your account has been activated` flash is present.

### 4.3 Happy path — Login [`app/auth/views/login.py:L25`] (fresh session)

HTTP transcript (a fresh, unauthenticated session B):

```
GET / (unauth session B):
  GET / -> HTTP 302 -> Location http://localhost:7777/auth/login
GET  /auth/login -> HTTP 200; csrf present=True
POST /auth/login (correct pw, activated) -> HTTP 302
  Location header: http://localhost:7777/dashboard/
GET  /dashboard/ (session B, authed) -> HTTP 200
  dashboard title/marker present: True
```

Server-side log:

```
2026-07-13 17:11:58,541 - SL - DEBUG - 12 - "/workspace/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 3 blitzyqa-a-1783962716@gmail.com blitzyqa-a-1783962716@gmail.com> in
2026-07-13 17:11:58,542 - SL - DEBUG - 12 - "/workspace/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-13 17:11:58,542 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.24187827110290527
2026-07-13 17:11:58,765 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.21746206283569336
```

Mapping to code:

- `login()` [`login.py:L25`] on success emits `LoginEvent.success` [`login.py:L71`] and calls
  `after_login(user, next_url)` [`login.py:L72`].
- `after_login()` [`app/auth/views/login_utils.py:L12`] logs `log user %s in` [`login_utils.py:L35`]
  → observed `log user <User 3 …> in`; calls `login_user()` [`login_utils.py:L36`]; sets
  `session["sudo_time"]` [`login_utils.py:L37`]; and returns an **HTTP 302** redirect to the next URL
  or `dashboard.index` [`login_utils.py:L44-45`] → observed `Location: http://localhost:7777/dashboard/`.
- **Inferred (branch not exercised):** `after_login()` first branches to FIDO
  [`login_utils.py:L20-27`] / OTP-MFA [`login_utils.py:L28-33`]; with no MFA configured on the temp
  user it falls through to the dashboard redirect. This MFA branch is **inferred** (not exercised).
- **Inferred:** `LoginEvent.success` / `RegisterEvent.success` [`app/events/auth_event.py`] `.send()`
  execute but emit **no** log line without a New Relic backend — a no-op analytics path (inferred).

### 4.4 Dashboard guard [`app/dashboard/views/index.py:L55-56`]

```
GET / (unauth) -> HTTP 302 -> http://localhost:7777/auth/login
GET /dashboard/ (unauth) -> HTTP 302 -> http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F
```

`@dashboard_bp.route("/", methods=["GET", "POST"])` [`app/dashboard/views/index.py:L55`] is guarded
by `@login_required` [`app/dashboard/views/index.py:L56`]; an unauthenticated request is redirected
(HTTP 302) to `/auth/login` by flask-login, preserving the `next` parameter. The authenticated
render (§4.2, §4.3) returns 200.

### 4.5 Edge paths (all exercised over real HTTP)

**Wrong password** [`login.py:L45-50`] — flash `Email or password incorrect` (category `error`),
`LoginEvent.failed` [`login.py:L50`]:

```
POST /auth/login (USER_A, wrong pw) -> HTTP 200
  flash 'Email or password incorrect' present: True
```
Rendered flash (SimpleLogin renders flashes as `toastr` JS in the base template):
```
<!-- Categories: success (green), info (blue), warning (yellow), danger (red) --> <script>toastr.error("Email or password incorrect");</script>
```

**Login before activation** [`login.py:L63-69`] — flash
`Please check your inbox for the activation email. You can also have this email re-sent` (category
`error`), `LoginEvent.not_activated`, and `show_resend_activation=True` [`login.py:L64`]:

```
POST /auth/register USER_B -> HTTP 200
DB users(USER_B) activated: 'f' (expect f)
POST /auth/login (USER_B, not activated) -> HTTP 200
  flash 'Please check your inbox for the activation email. You can also have this email re-sent' present: True
  show_resend_activation surfaced (resend link/button present): True
```
Rendered:
```
<!-- Categories: success (green), info (blue), warning (yellow), danger (red) --> <script>toastr.error("Please check your inbox for the activation email. You can also have this email re-sent")
resend token in page: resend_activation
```
(The resend flow itself: `resend_activation()` [`app/auth/views/resend_activation.py:L19`] flashes
`An activation email has been sent to you. Please check your inbox/spam folder.`
[`resend_activation.py:L38`] — inferred from source; the resend form was surfaced but not submitted.)

**Invalid activation code** [`activate.py:L28-36`] — HTTP **400** `Activation code cannot be found`:

```
GET /auth/activate?code=<bogus> -> HTTP 400
  body 'Activation code cannot be found' present: True
```
Rendered (`activate.html` inline error via the `error=` template var):
```
<div class="display-3 text-muted mb-5"> <i class="si si-exclamation"></i> Activation code cannot be found </div>
```

**Expired activation code** [`activate.py:L38-46`] — HTTP **400** `Activation code was expired`:

```
seeded temp expired activation_code id=3
INSERT 0 1 code=blitzyexpired1783962716 (expired 2h ago) [TEST-SETUP DB row]
GET /auth/activate?code=<expired> -> HTTP 400
  body 'Activation code was expired' present: True
```
Rendered:
```
<div class="display-3 text-muted mb-5"> <i class="si si-exclamation"></i> Activation code was expired </div>
```
`ActivationCode` uses `code = sa.Column(String(128))` [`app/models.py:L1208`] with a **1-hour**
default expiry (`_expiration_1h` = `arrow.now().shift(hours=1)`) [`app/models.py:L1186,L1212`]; the
model's `is_expired()` compares `expired < arrow.now()` [`app/models.py:L1214-1215`]. To observe
expiry without waiting an hour, a temporary `activation_code` row with a past `expired` value was
inserted (**labeled TEST-SETUP** — a DB seed), and then the **real** `GET /auth/activate` HTTP route
was driven; the observed **400** came over the canonical route.

**Already-authenticated activation** (bonus edge) [`activate.py:L18-22`] — HTTP **400**
`You are already logged in`:

```
GET /auth/activate (session A, authenticated) -> HTTP 400
  body 'You are already logged in' present: True
```

Server-side log confirming the three activation edge requests each returned 400:

```
2026-07-13 17:12:00,490 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'THIS_CODE_DOES_NOT_EXIST_000000')]) 400, takes 0.008833169937133789
2026-07-13 17:12:00,569 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'blitzyexpired1783962716')]) 400, takes 0.004903316497802734
2026-07-13 17:12:00,579 - SL - DEBUG - 12 - "/workspace/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'anything')]) 400, takes 0.006253242492675781
```

### 4.6 State reference — `User.create()`

`User.create()` [`app/models.py:L602`] provisions a verified `Mailbox` [`app/models.py:L611`] and
the first alias with `prefix="simplelogin-newsletter"` [`app/models.py:L636`], setting
`newsletter_alias_id` [`app/models.py:L643`]. This is confirmed at runtime by the welcome email in
§4.2 being addressed to `simplelogin-newsletter.delint275@sl.local`. Under `DISABLE_ONBOARDING=true`
it logs `Disable onboarding emails` and returns early [`app/models.py:L646-648`] (Objective 3, §5.2).


---

## 5. Objective 3 — Behind-the-Scenes Background Jobs / Internal Services

**Direct answer.** Background work and internal services are active and communicating. The
`job_runner.py` loop polls the `Job` table every **10 seconds** and drives a job's state
`ready(0)→taken(1)→done(2)`; the aiosmtpd email handler on `:20381` runs the full alias-forwarding
pipeline; the `cron.py`/yacron scheduler defines 15 scheduled jobs (most frequent:
`send_undelivered_mails` every 5 minutes); and the tiers coordinate through **shared PostgreSQL
tables** plus the Postgres `LISTEN/NOTIFY` channel `simplelogin_sync_events` (and Redis, though the
default web app does not connect to Redis — §3.1). Two default-config nuances are part of the correct
answer: onboarding jobs are **suppressed**, and event dispatch is a deliberate **no-op** (§5.3).

### 5.1 Job runner dispatch — `Job.state` ready→taken→done + 10s poll

Because `DISABLE_ONBOARDING=true` suppresses onboarding jobs (§5.2), a job that runs **now** was
triggered through the canonical dashboard GDPR "export data" flow —
`POST /dashboard/account_setting` with `form-name=send-full-user-report`
[`app/dashboard/views/account_setting.py:L130-131`] →
`ExportUserDataJob(current_user).store_job_in_db()` [`app/jobs/export_user_data_job.py:L173`] →
`Job.create(name=JOB_SEND_USER_REPORT, run_at=arrow.now())` [`export_user_data_job.py:L186`;
`JOB_SEND_USER_REPORT = "send-user-report"` at `app/config.py:L309`].

To capture the **before** state, `sl-jobs` was stopped, the job was created, then `sl-jobs` was
started while rapidly polling `Job.state`:

```
$ docker stop sl-jobs
$ python3 obj3_export.py     # logs in john@wick.com, POSTs the GDPR export
POST /auth/login john@wick.com -> HTTP 302 Location http://localhost:7777/dashboard/
POST /dashboard/account_setting form-name=send-full-user-report -> HTTP 200
```

**Before** (runner stopped) — `Job.state = 0` (`ready`):

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -tAc \
    "SELECT id,name,payload,taken,state,attempts,run_at FROM job WHERE id=1;"
1|send-user-report|{"user_id": 1}|f|0|0|2026-07-13 17:17:26
```

**During → After** (`docker start sl-jobs`, rapid poll of `state,taken,taken_at,attempts`):

```
$ docker start sl-jobs
DISTINCT STATES OBSERVED: ['0', '1', '2']
  t+0.03s : state=0 | taken=false | taken_at=                          | attempts=0
  t+1.65s : state=1 | taken=true  | taken_at=2026-07-13 17:17:51.986175 | attempts=1
  t+1.84s : state=2 | taken=true  | taken_at=2026-07-13 17:17:51.986175 | attempts=1
```

`sl-jobs` log during pickup:

```
2026-07-13 17:17:51 - SL - DEBUG - 1 - "/workspace/job_runner.py:334" - <module>() -  - Take job <Job 1 send-user-report {'user_id': 1}>
2026-07-13 17:17:51 - SL - DEBUG - 1 - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Your SimpleLogin data', from '"SimpleLogin (noreply)" <noreply@sl.local>' to 'john@wick.com'
```

Mapping to code — the `__main__` loop [`job_runner.py:L330`] runs inside
`create_light_app().app_context()` [`job_runner.py:L332`], logs `Take job %s` [`job_runner.py:L334`],
sets `job.taken=True` / `job.taken_at=now` / `job.state=JobState.taken.value` [`job_runner.py:L339`],
increments `attempts`, commits, calls `process_job(job)` [`job_runner.py:L188`], then sets
`job.state=JobState.done.value` [`job_runner.py:L344`] and commits. The `JobState` enum is
`ready=0` / `taken=1` / `done=2` / `error=3` [`app/models.py:L253-257`]; the `Job` model is at
[`app/models.py:L2683`]. The observed `['0','1','2']` is exactly `ready→taken→done`.

**10-second poll cadence** [`job_runner.py:L347`] — measured by drip-feeding marker jobs
`blitzy-cadence-probe-0..4` (an unknown job name → `Unknown job name %s` [`job_runner.py:L304`]) and
recording the server-set `taken_at` of each pickup:

```
probe-0: 2026-07-13 17:18:32.079936
probe-1: 2026-07-13 17:18:52.111445   gap 20.03s  (2 cycles)
probe-2: 2026-07-13 17:19:02.131398   gap 10.02s  (1 cycle)
probe-3: 2026-07-13 17:20:12.221862   gap 70.09s  (7 cycles)
probe-4: 2026-07-13 17:20:22.230304   gap 10.01s  (1 cycle)
```

Every gap is an integer multiple of ~10s, and **two clean consecutive one-cycle gaps** were observed
(probe-1→2 = 10.02s; probe-3→4 = 10.01s), confirming `time.sleep(10)` [`job_runner.py:L347`] is
stable across ≥2 observations. (The multi-cycle gaps occur when a probe was inserted a few seconds
after the runner had just polled, so it waited for the next full cycle.)

### 5.2 `DISABLE_ONBOARDING` nuance (why the export job was needed)

Under the default `DISABLE_ONBOARDING=true`, `User.create()` logs `Disable onboarding emails` and
returns early [`app/models.py:L646-648`] — **no** onboarding jobs are enqueued at registration.
Observed live in the Objective 2 registration log (§4.1):

```
2026-07-13 17:11:57,177 - SL - DEBUG - 12 - "/workspace/app/models.py:647" - create() -  - Disable onboarding emails
```

(Onboarding jobs are also future-dated — `run_at = arrow.now().shift(days=1/2/3)`
[`app/models.py:L651-665`] — so even when enabled they would not dispatch immediately.) This is why
the immediately-runnable GDPR export job was used to observe the runner acting on real work.

### 5.3 Event dispatch — Discrepancy #1 (which guard fires under the default)

A real event dispatch was triggered by creating an alias as `john@wick.com` over HTTP
(`POST /dashboard/` with `form-name=create-random-email`), and the event-dispatch log line was
observed in isolation:

```
POST /dashboard/ form-name=create-random-email -> HTTP 302 Location http://localhost:7777/dashboard/?highlight_alias_id=14
```
```
2026-07-13 ... - SL - INFO - "/workspace/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
```

**Observed reality:** under the default `example.env` (`EVENT_WEBHOOK` unset ⇒ `None`,
`EVENT_WEBHOOK_DISABLE=False`, `send_event(skip_if_webhook_missing=True)`),
`EventDispatcher.send_event()` short-circuits at the **second** guard —
`if not config.EVENT_WEBHOOK and skip_if_webhook_missing:` → logs
`Not sending events because webhook is not configured and allowed to be empty`
[`app/events/event_dispatcher.py:L61-65`] — and returns. It does **not** reach the partner-user guard.
This same `event_dispatcher.py:62` line was **also** observed during registration (§4.1), fired by the
alias-created event inside `User.create()`.

The three guards, in order, plus the success path [`app/events/event_dispatcher.py`]:

| # | Condition | Log line | Lines |
|---|-----------|----------|-------|
| 1 | `if config.EVENT_WEBHOOK_DISABLE:` | `Not sending events because webhook is disabled` | L57-59 |
| 2 | `if not config.EVENT_WEBHOOK and skip_if_webhook_missing:` | `Not sending events because webhook is not configured and allowed to be empty` **← fires under default** | L61-65 |
| 3 | `if not partner_user:` | `Not sending events because there's no partner user for user {user}` | L68-70 |
| ✓ | (all guards passed) | `Sent event to the dispatcher` | L84 |

Because dispatch short-circuits at guard #2 **before** `PostgresDispatcher.send()`
[`app/events/event_dispatcher.py:L23-26`], **no** `NOTIFY` is emitted by the app under the default —
confirmed by the `sync_event` table staying empty across the alias creation:

```
$ docker exec sl-postgres psql -U myuser -d simplelogin -tAc "SELECT count(*) FROM sync_event;"
0
```
(alias count for john went 10→11 from the `create-random-email`, but `sync_event` stayed `0`.)

### 5.4 Inter-process substrate — Postgres `LISTEN`/`NOTIFY`

`NOTIFICATION_CHANNEL = "simplelogin_sync_events"` [`app/events/event_dispatcher.py:L14`];
`PostgresDispatcher.send()` creates a `SyncEvent` and executes
`NOTIFY simplelogin_sync_events, '<id>'` [`app/events/event_dispatcher.py:L23-26`], consumed by
`PostgresEventSource(EVENT_LISTENER_DB_URI)` in the listener [`event_listener.py:L35`].

The listener was started and observed subscribing to the channel:

```
$ docker exec -w /workspace sl-web /app/venv/bin/python event_listener.py listener
>>> init logging <<<
2026-07-13 ... - SL - INFO - "/workspace/event_listener.py:34" - main() -  - Using PostgresEventSource
2026-07-13 ... - SL - INFO - "/workspace/event_listener.py:43" - main() -  - Starting with HttpEventSink
2026-07-13 ... - SL - INFO - "/workspace/app/events/event_source.py:49" - __listen() -  - Starting to listen to events
```

`main()` [`event_listener.py:L29`] in LISTENER mode logs `Using PostgresEventSource`
[`event_listener.py:L34`]; subcommands are `listener` [L72] / `dead_letter` [L78] / `debug` [L84] /
`run` [L87].

To prove the channel itself is live, a raw `LISTEN`/`NOTIFY` round-trip was performed over `psql`
(**labeled non-canonical** for the application dispatch path, since the app itself short-circuits at
guard #2 under the default and never emits on the channel — §5.3; this demonstrates the substrate the
app *would* use):

```
# session A
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -c "LISTEN simplelogin_sync_events;" -c "SELECT pg_sleep(3);"
# session B (concurrently)
$ psql "postgresql://myuser:mypassword@localhost:15432/simplelogin" -c "NOTIFY simplelogin_sync_events, 'blitzy-substrate-probe';"
# session A received:
Asynchronous notification "simplelogin_sync_events" with payload "blitzy-substrate-probe" received from server process with PID 968.
```

This confirms the `NOTIFY simplelogin_sync_events` channel [`app/events/event_dispatcher.py:L26`] that
the web app uses to signal the listener is functional end-to-end.

### 5.5 Email-forwarding pipeline — `swaks` → `:20381`

A message was sent through the real aiosmtpd SMTP controller to a **real seed alias** `e1@sl.local`
(created by `flask dummy-data`), which forwards to mailbox `john@wick.com`:

```
$ swaks --to e1@sl.local --from external-sender@ext.com --server 127.0.0.1:20381 \
        --header "Subject: Blitzy runtime forward test" --body "Objective 3 forwarding probe"
... <- 250 Message accepted for delivery
```

`sl-email` log — the complete forward routing chain:

```
2026-07-13 ... - "/workspace/email_handler.py:2343" - _handle() -  - New message, mail from external-sender@ext.com, rctp tos ['e1@sl.local']
2026-07-13 ... - "/workspace/email_handler.py:2202" - handle() -  - Forward phase external-sender@ext.com(...) -> e1@sl.local
2026-07-13 ... - "/workspace/email_handler.py:580" - handle_forward() -  - Create or get contact for from_header:external-sender@ext.com
2026-07-13 ... - "/workspace/app/contact_utils.py:110" - create_contact() -  - Created contact <Contact 2 external-sender@ext.com 5> for alias <Alias 5 e1@sl.local>
2026-07-13 ... - "/workspace/app/dmarc.py:33" -  - DMARC check disabled
2026-07-13 ... - "/workspace/email_handler.py:688" - forward_email_to_mailbox() -  - Forward <Contact 2 ...> -> <Alias 5 e1@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-13 ... - "/workspace/email_handler.py:740" -  - Create <EmailLog 2> for <Contact 2 ...>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-13 ... - "/workspace/email_handler.py:867" -  - From header, new:"external-sender at ext.com" <external-sender_at_ext_com_yteev@sl.local>
2026-07-13 ... - "/workspace/app/mail_sender.py:131" - send() -  - send email with subject 'Blitzy runtime forward test', from '"external-sender at ext.com" <external-sender_at_ext_com_yteev@sl.local>' to 'e1@sl.local'
2026-07-13 ... - "/workspace/email_handler.py:2367" - _handle() -  - Finish mail_from external-sender@ext.com, ... return code '250 Message accepted for delivery'
```

Mapping to code — the aiosmtpd data callback `MailHandler.handle_DATA()` [`email_handler.py:L2289`]
routes into `handle()` [`email_handler.py:L1945`], which (this being a message from an external
sender **to** an alias) enters the **forward** phase and calls `handle_forward()`
[`email_handler.py:L536`]. The pipeline creates/looks up the sender `Contact`, resolves
`Alias → Mailbox`, writes an `EmailLog`, rewrites the `From` header to a reverse-alias, and (under
`NOT_SEND_EMAIL=true`) logs the final relay as a no-op — returning `250 Message accepted for
delivery`. The routing decision (`Alias 5 e1@sl.local → Mailbox 1 john@wick.com`) **is** the signal
that alias-based forwarding is live, even though nothing physically leaves the host in local default.

### 5.6 Cron / yacron schedule (`crontab.yml`)

The `cron.py` scheduler is driven by `crontab.yml`; each yacron entry runs
`python /code/cron.py -j <job>`. The canonical entry point was exercised directly with the most
frequent job:

```
$ docker exec -w /workspace sl-web /app/venv/bin/python cron.py -j send_undelivered_mails
>>> init logging <<<
2026-07-13 17:23:42,010 - SL - DEBUG - 122 - "/workspace/cron.py:1263" - <module>() -  - Start running cronjob
2026-07-13 17:23:42,012 - SL - DEBUG - 122 - "/workspace/cron.py:1314" - <module>() -  - Sending undelivered emails
```

`send_undelivered_mails` is a no-op here (no undelivered `EmailLog` rows pending), but
`cron.py:1263 "Start running cronjob"` and `cron.py:1314 "Sending undelivered emails"` prove the
canonical yacron→`cron.py` entry point runs.

Full parsed schedule (`schedule | name | command | concurrencyPolicy`):

```
0 0 * * *      | SimpleLogin growth stats                             | python /code/cron.py -j stats
15 1 * * *     | SimpleLogin Delete Old Monitoring records            | python /code/cron.py -j delete_old_monitoring
15 2 * * *     | SimpleLogin Custom Domain check                      | python /code/cron.py -j check_custom_domain
15 3 * * *     | SimpleLogin HIBP check                               | python /code/cron.py -j check_hibp  [Forbid]
15 4 * * *     | SimpleLogin Notify HIBP breaches                     | python /code/cron.py -j notify_hibp  [Forbid]
15 5 * * *     | SimpleLogin Delete Logs                              | python /code/cron.py -j delete_logs
30 5 * * *     | SimpleLogin Delete Old data                          | python /code/cron.py -j delete_old_data
15 6 * * *     | SimpleLogin Poll Apple Subscriptions                 | python /code/cron.py -j poll_apple_subscription
15 8 * * *     | SimpleLogin Notify Trial Ends                        | python /code/cron.py -j notify_trial_end
15 9 * * *     | SimpleLogin Notify Manual Subscription Ends          | python /code/cron.py -j notify_manual_subscription_end
15 10 * * *    | SimpleLogin Notify Premium Ends                      | python /code/cron.py -j notify_premium_end
15 11 * * *    | SimpleLogin delete users scheduled to be deleted     | python /code/cron.py -j delete_scheduled_users  [Forbid]
*/5 * * * *    | SimpleLogin send unsent emails                       | python /code/cron.py -j send_undelivered_mails  [Forbid]
0 * * * *      | SimpleLogin clear alias_audit_log old entries        | python /code/cron.py -j clear_alias_audit_log  [Forbid]
0 * * * *      | SimpleLogin clear user_audit_log old entries         | python /code/cron.py -j clear_user_audit_log  [Forbid]
```

The **most frequent** job is `send_undelivered_mails` at `*/5 * * * *` (every 5 minutes,
`concurrencyPolicy: Forbid`) — command at [`crontab.yml:L78`], schedule at [`crontab.yml:L80`],
policy at [`crontab.yml:L82`]. Daily `stats` (`0 0 * * *`) is at [`crontab.yml:L3,L5`];
`check_custom_domain` (`15 2 * * *`) at [`crontab.yml:L15,L17`]; `check_hibp` (`15 3 * * *`) at
[`crontab.yml:L21,L23`]; the hourly audit-log cleanups (`0 * * * *`) at [`crontab.yml:L87,L94`].
`crontab-all-hosts.yml` is the multi-host variant.

### 5.7 The shared substrate (as observed, not modified)

The three tiers (web / email / background) coordinate through:

- **PostgreSQL shared tables** — `users`, `activation_code`, `job`, `sync_event`, `alias`, `contact`,
  `email_log`, `mailbox` — written by one tier and read/acted-on by another (e.g., the web app writes
  a `Job` row that `job_runner.py` picks up; the email handler writes `Contact`/`EmailLog` rows).
- **Postgres `LISTEN`/`NOTIFY`** on channel `simplelogin_sync_events` [`event_dispatcher.py:L14`] —
  the web app's dispatcher would `NOTIFY` and `event_listener.py` `LISTEN`s (§5.4).
- **Redis** — sessions and rate limiting; provisioned and live, though the **default** web app does
  not connect to it (`MEM_STORE_URI=None`, §3.1).


---

## 6. Inferred-vs-Observed Notes & Environment Caveats

### 6.1 Discrepancies observed against a naive expectation

- **Discrepancy #1 (event-dispatch guard).** Under the default config, `send_event()` short-circuits
  at **guard #2** (`Not sending events because webhook is not configured and allowed to be empty`
  [`app/events/event_dispatcher.py:L61-65`]), **not** the partner-user guard [`L68-70`]. Observed
  twice (registration alias event §4.1; explicit alias creation §5.3). This means **no** `NOTIFY` is
  emitted by the app under the default (`sync_event` count stays 0). **(Observed.)**
- **Discrepancy #2 (activation template path).** The real templates are
  `templates/emails/transactional/activation.txt` and `…/activation.html` — **not**
  `templates/transactional/…`. `render()` [`app/email_utils.py:L72`] joins
  `os.path.join(config.ROOT_DIR, "templates", "emails")`, i.e. it prepends `emails/`. Verified on
  disk:

```
$ ls templates/emails/transactional/activation.txt templates/emails/transactional/activation.html
templates/emails/transactional/activation.html
templates/emails/transactional/activation.txt
$ ls templates/transactional/activation.txt
ls: cannot access 'templates/transactional/activation.txt': No such file or directory
```

- **Discrepancy #3 (code-structure doc).** The three-entry-point architecture (webapp =
  `wsgi.py`/`server.py`; email handler = `email_handler.py`; job runner = `job_runner.py`) is
  documented in **`CONTRIBUTING.md` §Code structure** (`## Code structure` at `CONTRIBUTING.md:L141`;
  `wsgi.py and server.py: the webapp` at L145; `email_handler.py: the email handler` at L146) —
  **not** in `docs/code-structure.md`, which is a 9-line `# TODO` note about `local_data/` JWT keys.
- **Discrepancy #4 (Werkzeug "Running on" banner suppressed).** The Werkzeug
  `* Running on http://127.0.0.1:7777` line does **not** appear in the web startup log:

```
$ docker logs sl-web 2>&1 | grep -ic "running on"
0
```
  With Werkzeug 1.0.1, `run_simple` emits the "Running on" line via the `werkzeug` logger, which is
  **disabled** at [`app/log.py:L70-71`] (`logging.getLogger("werkzeug").disabled = True`). The Flask
  1.1.2 click banner (`Serving Flask app "server"` … `* Debug mode: on`) is **not** routed through
  that logger, so it still prints. The disabled `werkzeug` logger is also why per-request access
  lines are absent — SimpleLogin logs requests itself via `after_request` [`server.py:L284`]. The
  web-bind readiness signal under this config is therefore the Flask banner + `* Debug mode: on` +
  live HTTP + the SimpleLogin `after_request` lines (all shown in §3.1). **(Observed — this
  contradicts the naive expectation of a "Running on" line.)**

### 6.2 Inferred (not runtime-confirmed) statements

- **MFA branch in `after_login()`** [`login_utils.py:L20-33`] — with no FIDO/OTP configured on the
  temp user, `after_login()` falls through to the dashboard redirect. The MFA branches were **not**
  exercised (no MFA enrolled), so they are **inferred** from source.
- **Analytics events** — `LoginEvent`/`RegisterEvent` `.send()` [`app/events/auth_event.py`] execute
  on the login/register paths but emit **no** log line without a New Relic backend. The no-op is
  **inferred** (absence of a log line is consistent with the no-op path).
- **Resend-activation flash text** — `An activation email has been sent to you. Please check your
  inbox/spam folder.` [`app/auth/views/resend_activation.py:L38`] is **inferred** from source; the
  resend form was surfaced (§4.5) but not submitted.

### 6.3 Environment caveats

- **`cbor2 5.2.0` sdist build failure on newer OSes** [`poetry.lock:L411-412`] — environment caveat
  only; resolved by using the canonical `python:3.10` image. `pyproject.toml`/`poetry.lock` were
  never edited.
- **Postgres port reconciliation** — `CONTRIBUTING.md:L100` uses host `15432`; `example.env:L75`
  `DB_URI` uses `5432`. The setup pointed `DB_URI` at `15432`; a setup detail, not a defect (§2.6).
- **Redis not connected by default** — `MEM_STORE_URI=None` [`app/config.py:L568`] so the default web
  app uses an in-memory limiter + signed-cookie sessions; Redis is live for parity (§3.1).
- **IPv6 sysctl** — the canonical environment requires `net.ipv6.conf.*.disable_ipv6=0` for a subset
  of `mail_sender` behavior; this is an environment setting, not a source concern.

---

## 7. Coverage Pass — Every Named Item Answered

Legend: **Value** = the concrete observed value; **file:line** = the function/method that performs
the work; all rows are backed by observed runtime evidence in the referenced section.

### Objective 1 — startup readiness

| Item | Value (observed) | file:line | § |
|------|------------------|-----------|---|
| Logging init banner | `>>> init logging <<<` | `app/log.py:L67` | 3.1 |
| Logger name | `SL` | `app/log.py:L79` | 3.1 |
| Log format | `%(asctime)s - %(name)s - …` | `app/log.py:L12-14` | 3.1 |
| Flask dev-server bind | `* Debug mode: on` + live HTTP `:7777` | `server.py:L588` (`app.run(...,port=7777)`) | 3.1 |
| Werkzeug "Running on" | **absent** (logger disabled) | `app/log.py:L70-71` | 3.1 / 6.1 |
| PostgreSQL connectivity | head `32f25cbf12f6`, 77 tables | `server.py:L146` | 3.1 |
| Redis connectivity | not connected by default (`MEM_STORE_URI=None`) | `server.py:L163-165`; `app/config.py:L568` | 3.1 |
| Email listen banner | `Listen for port 20381` | `email_handler.py:L2403` (default `email_handler.py:L2399`) | 3.2 |
| Mail controller banner | `Start mail controller 0.0.0.0 20381` | `email_handler.py:L2386` (`L2383`,`L2381`) | 3.2 |
| Blueprints `auth`/`dashboard` | `GET /auth/login` 200; `GET /dashboard/` 302 | `server.py:L233,L234,L236` | 3.1 |
| Job runner boot | banner + silent 10s loop | `job_runner.py:L330,L332,L347` | 3.3 |
| One-time data init | runs in `flask dummy-data`, not boot | `init_app.py:L13,L36,L39,L59`; `server.py:L490-497` | 3.4 |

### Objective 2 — register → verify → login → dashboard

| Item | Value (observed) | file:line | § |
|------|------------------|-----------|---|
| Register route | `POST /auth/register` → 200 | `register.py:L32` | 4.1 |
| `create user` log | `create user blitzyqa-a-…@gmail.com` | `register.py:L85` | 4.1 |
| Activation email (NOT_SEND_EMAIL) | subject `Just one more step to join SimpleLogin` | `mail_sender.py:L131-135`; `email_utils.py:L128` | 4.1 |
| Waiting-activation render | `register_waiting_activation.html` | `register.py:L104` | 4.1 |
| Activation code | 30-char `qebbdvxpgxlkzcvkoxistenbvgqxiz` | `register.py:L120,L124` | 4.1 |
| Verify route | `GET /auth/activate` → 302 `/dashboard/` | `activate.py:L17,L61-67` | 4.2 |
| `User.activated` transition | **`f` → `t`** | `activate.py:L49`; `app/models.py:L358` | 4.2 |
| Activation flash | `Your account has been activated` | `activate.py:L56` | 4.2 |
| Login route | `POST /auth/login` → 302 `/dashboard/` | `login.py:L25,L71-72` | 4.3 |
| `after_login` | `log user <User 3 …> in` → 302 | `login_utils.py:L12,L35,L44-45` | 4.3 |
| Dashboard render | authed `GET /dashboard/` → 200 | `dashboard/views/index.py:L55-56` | 4.2/4.3 |
| Dashboard guard | unauth `GET /` and `/dashboard/` → 302 `/auth/login` | `dashboard/views/index.py:L56` | 4.4 |
| Edge: wrong password | 200, `Email or password incorrect` | `login.py:L45-50` | 4.5 |
| Edge: login-before-activation | 200, `Please check your inbox for the activation email. You can also have this email re-sent` | `login.py:L63-69` | 4.5 |
| Edge: invalid code | **400**, `Activation code cannot be found` | `activate.py:L28-36` | 4.5 |
| Edge: expired code | **400**, `Activation code was expired` | `activate.py:L38-46`; `models.py:L1208,L1212,L1214-1215` | 4.5 |
| Edge: already-authenticated | **400**, `You are already logged in` | `activate.py:L18-22` | 4.5 |
| Newsletter alias provisioned | welcome email to `simplelogin-newsletter.delint275@sl.local` | `app/models.py:L602,L636,L643` | 4.2/4.6 |

### Objective 3 — background jobs / internal services

| Item | Value (observed) | file:line | § |
|------|------------------|-----------|---|
| Job runner dispatch | `Take job <Job 1 send-user-report …>` | `job_runner.py:L334` | 5.1 |
| `Job.state` transition | **`0`(ready) → `1`(taken) → `2`(done)** | `job_runner.py:L339,L344`; `app/models.py:L253-257` | 5.1 |
| Canonical job trigger | GDPR export `send-user-report`, `run_at=now` | `export_user_data_job.py:L173,L186`; `config.py:L309` | 5.1 |
| Unknown job | `Unknown job name` (cadence probes) | `job_runner.py:L304` | 5.1 |
| 10-second poll | two clean ~10s gaps (10.02s, 10.01s) | `job_runner.py:L347` | 5.1 |
| DISABLE_ONBOARDING | `Disable onboarding emails` | `app/models.py:L646-648` | 5.2 |
| Event guard (Discrepancy #1) | `Not sending events because webhook is not configured and allowed to be empty` | `event_dispatcher.py:L61-65` | 5.3 |
| Event guard #1 | `Not sending events because webhook is disabled` | `event_dispatcher.py:L57-59` | 5.3 |
| Event guard #3 | `Not sending events because there's no partner user for user {user}` | `event_dispatcher.py:L68-70` | 5.3 |
| Event success path | `Sent event to the dispatcher` | `event_dispatcher.py:L84` | 5.3 |
| NOTIFY channel | `simplelogin_sync_events` (raw round-trip works) | `event_dispatcher.py:L14,L26` | 5.4 |
| Listener | `Using PostgresEventSource` | `event_listener.py:L34,L35` | 5.4 |
| Email forward pipeline | `handle()` → `handle_forward()`, `250 Message accepted for delivery` | `email_handler.py:L1945,L536,L2289` | 5.5 |
| Alias→mailbox routing | `Alias 5 e1@sl.local → Mailbox 1 john@wick.com` | `email_handler.py:L688` | 5.5 |
| Cron entry point | `Start running cronjob` / `Sending undelivered emails` | `cron.py:L1263,L1314` | 5.6 |
| yacron most-frequent job | `send_undelivered_mails` `*/5 * * * *` `[Forbid]` | `crontab.yml:L78,L80,L82` | 5.6 |
| Substrate | Postgres shared tables + `LISTEN`/`NOTIFY` + Redis | `event_dispatcher.py:L14` | 5.7 |

All named items across the three objectives are answered with a concrete value, a `file:line`
citation, and observed evidence in the referenced section. ✅


---

## 8. Cleanup Confirmation & Read-Only Guarantee

This was a **read-only** investigation. Per the user directive — *"You can create anything temporary
that you need for testing like new users or aliases but remove them when you are done and please do
not make any changes to the source code."* — all temporary entities and observation scripts created
during the investigation were removed, and no existing repository file was modified. The **only**
tracked change to the repository is this one new file, `blitzy/documentation/app_2cd6ee777f8c.md`.

Temporary entities created during the investigation and removed afterward:

- **Temporary users** `blitzyqa-a-1783962716@gmail.com` (id 3) and `blitzyqa-b-1783962716@gmail.com`
  (id 4) and their cascaded rows (mailbox, aliases including the `simplelogin-newsletter.*` alias,
  and `activation_code` rows).
- **Seeded expired `activation_code`** `blitzyexpired1783962716` (the TEST-SETUP row for the expired
  edge, §4.5).
- **Job rows** id 1 (`send-user-report`) and id 2–6 (`blitzy-cadence-probe-0..4`).
- **Contact** id 2 (`external-sender@ext.com`) and its `EmailLog` (id 2) from the `swaks` forward.
- **Random alias** id 14 created for `john@wick.com` via `create-random-email` (§5.3).
- All observation scripts and captured-output files under `/tmp/obs/` (untracked, outside the repo).

Baseline seed data from `flask dummy-data` (e.g. `john@wick.com`, `winston@continental.com`,
`e0..e2@sl.local`) was left intact as the canonical starting state.

Post-cleanup `git status` proving the working tree is unchanged except this one new file
(captured on branch `blitzy-d7c2d64b-4eea-4bae-9c05-a7e6d6e3887f`, before staging/committing the
document):

```
$ git rev-parse --abbrev-ref HEAD
blitzy-d7c2d64b-4eea-4bae-9c05-a7e6d6e3887f

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/app_2cd6ee777f8c.md

$ git status --porcelain pyproject.toml poetry.lock   # empty output => both unchanged
```

The single `??` entry is the answer document itself; nothing else is added, modified, or deleted.
`pyproject.toml` and `poetry.lock` were **not** touched at any point (the `cbor2` matter was handled
purely by using the canonical image — §2.6). The DB was restored to its `flask dummy-data` baseline
(only `john@wick.com` and `winston@continental.com` remain; all temporary users/aliases/jobs/
contacts/`activation_code` rows removed), and all observation scripts were deleted.

