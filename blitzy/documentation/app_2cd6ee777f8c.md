# SimpleLogin — Runtime Verification for a First-Time Operator

**Branch:** `app_2cd6ee777f8c` · **HEAD:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
**Canonical environment:** Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (tag of `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`).

This document answers, **from live observation of a locally running SimpleLogin deployment**, the four questions a first-time operator asks:

- **Q1 — Readiness signals.** What observable signals, in the **logs** or the **UI**, prove the platform is up and ready to handle user authentication and alias-based email activity?
- **Q2 — New-user experience.** Walking through **register → verify address → log in**, what visible behavior confirms each step works and forwards the user into the dashboard?
- **Q3 — Behind the scenes.** What does the application do internally during that flow — the **background jobs** and **internal services** that support email forwarding and identity verification — and what is observable at runtime showing these pieces are active and communicating?
- **Q4 — Constraints.** Temporary test artifacts may be created but must be removed; **no source code may be changed.**

## How to read this document (evidence discipline)

Every behavioral claim below is presented with three things placed **next to the claim**:

1. the **exact command** that was run (in a fenced block immediately before its output),
2. the **complete, unedited observed output** (real log lines / HTTP status / DB values), and
3. a **`file:line` citation** naming the enclosing function/method.

All observations were taken by **running the real entry points** (`server.py`, `job_runner.py`, `email_handler.py`, `cron.py`, `event_listener.py`) and driving the real HTTP endpoints and the real database. Every command can be re-run to reproduce its output (§Q4.4), with one honest qualification: a handful of fields are **assigned fresh on every run** — process ids, wall-clock timestamps, the per-process `GNUPGHOME /tmp/<random>` paths, session-cookie bytes (shown as `<redacted>`), and the `Content-Length` of full debug-toolbar-rendered HTML pages. These are **annotated as per-run-variable wherever they appear**, and the *invariant* signal (the status code, the log message, the state transition, the marker text) is what each claim actually rests on. Where a value comes from the sandbox image rather than the pristine canonical build, it is explicitly **labeled non-canonical** and the cause is explained (§0.7).

Commands were executed inside the running canonical container via `docker exec sl-canonical …`; the container holds the repo at `/app` on the same HEAD as this branch. The configuration file lives **outside** the repository at `/root/sl.env` (§0.4), so the source tree is never touched (proven in §Q4.1).

---

## Operator flow at a glance

The diagram below maps the whole investigation — from provisioning the instance, through the three observation areas (each a question this document answers), to the mandatory cleanup — and shows where in the document each part is proven.

```mermaid
flowchart TD
    A["Provision: PostgreSQL + Redis + out-of-repo CONFIG<br/>(§0.1–§0.4)"] --> B["alembic upgrade head + flask dummy-data<br/>seeds john@wick.com, winston@continental.com (§0.5)"]
    B --> C["Start entry points: server.py :7777, job_runner.py,<br/>email_handler.py :20381, event_listener.py"]
    C --> Q1["Q1 — Readiness signals<br/>startup banners, GET / → 302 /auth/login,<br/>/git /live /exception, worker banners (§Q1)"]
    Q1 --> Q2["Q2 — New-user flow<br/>register → verify (activated f→t) → login → dashboard (§Q2)"]
    Q2 --> Q3["Q3 — Behind the scenes<br/>job_runner (ready→taken→done), Postgres LISTEN/NOTIFY,<br/>two mail paths, email_handler, cron (§Q3)"]
    Q3 --> Q4["Q4 — Cleanup<br/>delete probe rows → baseline restored,<br/>targeted-PID teardown, 1-file git proof (§Q4)"]
    Q4 --> Z["Repository unchanged except one new .md"]
```

> **Before you expose this anywhere:** the instance observed here is the Flask **debug** dev-server with the **Debug Toolbar** enabled — a **trusted-local-only** posture that leaks the session-signing secret and the database URI into ordinary pages and ships no browser-hardening headers. It is the correct thing to *observe* (it is the documented local entry point), but must never be port-forwarded to an untrusted network. This is documented with live, secret-redacted evidence in **§0.8**.

The core new-user experience (Q2) is the following request/state sequence, observed end-to-end:

```mermaid
sequenceDiagram
    participant U as Browser (curl)
    participant W as server.py (Flask :7777)
    participant DB as PostgreSQL
    U->>W: POST /auth/register (email, password, csrf)
    W->>DB: User.create(...) + ActivationCode.create(random_string(30))
    W-->>U: 200 register_waiting_activation.html
    Note over W: NOT_SEND_EMAIL=true → activation email is LOGGED
    U->>W: GET /auth/activate?code=…
    W->>DB: users.activated f → t; delete single-use ActivationCode
    W-->>U: 302 → /dashboard/
    U->>W: POST /auth/login (valid credentials)
    W-->>U: 302 → /dashboard/  (after_login)
    U->>W: GET /dashboard/ (session cookie)
    W-->>U: 200 dashboard/index.html (Alias / Mailbox UI)
```

---


## §0 — Environment, run recipe, and the logging backbone

### 0.1 Canonical environment and observed versions

The build/run environment is the Docker image named above. It ships a pre-built virtualenv at `/app/venv` (Python 3.10) plus PostgreSQL and Redis. The interpreter and the pinned packages relevant to this investigation, read from the running venv:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && python --version && \
  pip freeze | grep -iE "^Flask==|^Werkzeug==|^Flask-Login==|^Flask-WTF==|^WTForms==|^SQLAlchemy==|^psycopg2-binary==|^Flask-Migrate==|^redis==|^aiosmtpd==|^bcrypt==|^pyotp==|^webauthn==|^yacron==|^coloredlogs==|^alembic==|^gunicorn=="'
```

```
Python 3.10.18
aiosmtpd==1.4.2
alembic==1.4.3
bcrypt==3.2.0
coloredlogs==14.0
Flask==1.1.2
Flask-Login==0.5.0
Flask-Migrate==2.5.3
Flask-WTF==0.14.3
gunicorn==20.0.4
psycopg2-binary==2.9.3
pyotp==2.4.0
redis==4.6.0
SQLAlchemy==1.3.24
webauthn==0.4.7
Werkzeug==1.0.1
WTForms==2.3.3
yacron==0.11.2
```

The `Werkzeug/1.0.1 Python/3.10.18` pair is independently confirmed by the HTTP `Server:` header captured throughout §Q1. Two observed versions differ from the technical-specification dependency table and are reported here as **observed, not as the spec claims** (Run-First discipline): `aiosmtpd==1.4.2` (spec table says `1.2`) and `redis==4.6.0` (spec table says `4.5.3`). These differences do not affect the authentication or readiness behavior under investigation.

### 0.2 The three documented entry points (plus two workers)

`CONTRIBUTING.md` names the three entry points:

```bash
docker exec sl-canonical bash -c 'sed -n "145,147p" /app/CONTRIBUTING.md'
```

```
- wsgi.py and server.py: the webapp.
- email_handler.py: the email handler.
- cron.py: the cronjob.
```

Two additional long-running processes complete the runtime: `job_runner.py` (background jobs) [CONTRIBUTING.md:L228] and `event_listener.py` (event consumer). The local dev web server is `app.run(debug=True, port=7777)` — the last statement of the dev bootstrap at [server.py:L588]. `wsgi.py` is the production entry point (`from server import create_app; app = create_app()`), in contrast to the dev `server.py`. The `flask dummy-data` CLI that seeds the demo database is `dummy_data()` at [server.py:L490-L497]; it calls `fake_data()` [server.py:L495], `add_sl_domains()` [server.py:L496], and `add_proton_partner()` [server.py:L497] (see §0.3, and the accurate description in §Q3.6).

### 0.3 Local run recipe

The source repository's documented setup uses **Poetry** [CONTRIBUTING.md:L23,L31], and the canonical local recipe is [CONTRIBUTING.md:L106,L109]:

```bash
docker exec sl-canonical bash -c 'sed -n "31p;106p;109p" /app/CONTRIBUTING.md'
```

```
poetry sync
alembic upgrade head && flask dummy-data && python3 server.py
then open http://localhost:7777, you should be able to login with `john@wick.com / password` account.
```

**Important environment distinction (dependency accuracy).** The `poetry sync` step in `CONTRIBUTING.md` is the *source-repository* recipe for building the dependency set from scratch. The **canonical image already ships that dependency set pre-built** in `/app/venv`, and it does **not** include the `poetry` binary — so an operator inside the image skips `poetry sync` and simply activates the venv:

```bash
docker exec sl-canonical bash -c 'command -v poetry || echo "poetry: NOT FOUND on PATH"; ls -d /app/venv && /app/venv/bin/python --version'
```

```
poetry: NOT FOUND on PATH
/app/venv
Python 3.10.18
```

The database is initialized with `alembic upgrade head`. The complete migration run ends at head `32f25cbf12f6`:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env alembic upgrade head 2>&1 | tail -3'
```

```
INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at
```

The **live** database revision (not merely the migration-graph head) is confirmed two independent ways — `alembic current` and the `alembic_version` table:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env alembic current 2>&1 | tail -1'
docker exec sl-canonical bash -c "PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin -c 'SELECT version_num FROM alembic_version;'"
```

```
32f25cbf12f6 (head)
 version_num
--------------
 32f25cbf12f6
(1 row)
```

Here `alembic current` (the applied revision in *this* database) equals `alembic heads` (the graph head) because the database is fully migrated; the `alembic_version.version_num` value is the authoritative on-disk revision.

### 0.4 Canonical local-dev config knobs (out-of-repo) and env-over-file precedence

> ⚠️ **TRUSTED-LOCAL-ONLY RUNTIME.** The web entry point exercised throughout this document is the Flask **debug** dev-server — `app.run(debug=True, port=7777)` at [server.py:L588], with the **Flask Debug Toolbar** enabled at [server.py:L576-L582]. That combination is intended for a single operator on `localhost` and is **not safe to expose on any network**: the toolbar renders the application's config (including the session-signing `SECRET_KEY` and the full database URI) into every normal HTML page, and the responses carry no browser-hardening headers. Knowledge of `SECRET_KEY` alone is enough to forge an authenticated session for any user (including the seeded admin). §0.8 documents this posture with live, **secret-redacted** evidence. Hardening the product (disabling the toolbar, running under `wsgi.py`+`gunicorn` with debug off, adding security headers) is **out of scope** here — this task is strictly read-only (§Q4.1) and observes the runtime as-is; it does not change it.

To honor the read-only constraint (Q4), the runtime config is a copy of `example.env` placed **outside** the checkout at `/root/sl.env`, and SimpleLogin is pointed at it with `CONFIG=/root/sl.env`. This works because `get_abs_path` passes any absolute path through unchanged:

```bash
docker exec sl-canonical bash -c 'sed -n "14,20p" /app/app/config.py'
```

```python
def get_abs_path(file_path: str):
    """append ROOT_DIR for relative path"""
    # Already absolute path
    if file_path.startswith("/"):
        return file_path
    else:
        return os.path.join(ROOT_DIR, file_path)
```

`CONFIG` is consumed at import time by the `load_dotenv` block in `config.py`:

```bash
docker exec sl-canonical bash -c 'sed -n "65,71p" /app/app/config.py'
```

```python
config_file = os.environ.get("CONFIG")
if config_file:
    config_file = get_abs_path(config_file)
    print("load config file", config_file)
    load_dotenv(get_abs_path(config_file))
else:
    load_dotenv()
```

**Precedence caveat (cause → effect).** `load_dotenv()` at [app/config.py:L69,L71] runs with `python-dotenv`'s default `override=False`, so **a variable already present in the process environment is NOT overwritten by `/root/sl.env`** — the environment wins over the file. This matters when an operator (or a test harness) exports a value before launching a process: the exported value takes effect, not the file's. In this investigation no conflicting variable was exported for the app/worker processes, so the `/root/sl.env` values below are the effective ones.

The observed knobs (and their code citations):

```bash
docker exec sl-canonical bash -c 'grep -nE "^URL=|^NOT_SEND_EMAIL=|^EMAIL_DOMAIN=|^DISABLE_ONBOARDING=|^DB_URI=" /root/sl.env'
```

```
6:URL=http://localhost:7777
19:NOT_SEND_EMAIL=true
22:EMAIL_DOMAIN=sl.local
75:DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
150:DISABLE_ONBOARDING=true
```

- `URL=http://localhost:7777` — [example.env:L6]
- `NOT_SEND_EMAIL=true` — makes the mailer **log** email content instead of sending it — [example.env:L19]; consumed by the guard `if config.NOT_SEND_EMAIL:` at [app/mail_sender.py:L130].
- `EMAIL_DOMAIN=sl.local` — [example.env:L22]
- `DISABLE_ONBOARDING=true` — suppresses onboarding-job scheduling — [example.env:L150]; consumed by `if config.DISABLE_ONBOARDING:` in `User.create` at [app/models.py:L646-L648] (observed live during seeding: `Disable onboarding emails` at [app/models.py:L647], §0.3 seed log below).

### 0.5 Genuine first-run bootstrap: `flask dummy-data` (fresh database)

The demo data is seeded by the `flask dummy-data` CLI on the freshly-migrated database. Because the database was created fresh for this investigation, the run below is a **genuine first run**, and its complete output (27 lines) is shown in full:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env flask dummy-data'
```

```
load config file /root/sl.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/qjaytelpkadxbbeckpqs
Upload files to local dir
>>> init logging <<<
2026-07-14 23:12:24,437 - SL - DEBUG - 122 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-14 23:12:26,290 - SL - WARNING - 122 - "/app/server.py:494" - dummy_data() -  - reset db, add fake data
2026-07-14 23:12:26,290 - SL - DEBUG - 122 - "/app/app/fake_data.py:41" - fake_data() -  - create fake data
2026-07-14 23:12:26,679 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:26,682 - SL - DEBUG - 122 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-14 23:12:26,702 - SL - DEBUG - 122 - "/app/app/models.py:1459" - generate_random_alias_email() -  - generate email word_list878@sl.local
2026-07-14 23:12:26,714 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:26,768 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:26,777 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:26,795 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:26,808 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:26,825 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:26,833 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:26,844 - SL - DEBUG - 122 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo-znhipdtcnb
2026-07-14 23:12:26,850 - SL - DEBUG - 122 - "/app/app/models.py:1255" - generate_oauth_client_id() -  - generate oauth_client_id demo2-jyqpqhtfll
2026-07-14 23:12:27,124 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:27,125 - SL - DEBUG - 122 - "/app/app/models.py:647" - create() -  - Disable onboarding emails
2026-07-14 23:12:27,148 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:27,161 - SL - INFO - 122 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-14 23:12:27,170 - SL - INFO - 122 - "/app/init_app.py:44" - add_sl_domains() -  - Add sl.local to SL domain
```

The last line is the **first-run domain-seeding signal** — `LOG.i("Add %s to SL domain", alias_domain)` in `add_sl_domains` at [init_app.py:L44]. It fires only on a first run: `add_sl_domains` takes the `else` branch (create) when the domain does not yet exist, and logs `%s is already a SL domain` at [init_app.py:L42] on any later run:

```bash
docker exec sl-canonical bash -c 'sed -n "39,45p" /app/init_app.py'
```

```python
def add_sl_domains():
    for alias_domain in ALIAS_DOMAINS:
        if SLDomain.get_by(domain=alias_domain):
            LOG.d("%s is already a SL domain", alias_domain)
        else:
            LOG.i("Add %s to SL domain", alias_domain)
            SLDomain.create(domain=alias_domain, use_as_reverse_alias=True)
```

That the `sl.local` row was created **by this run** (not pre-existing) is proven by its `created_at` matching the log timestamp `23:12:27,17…` (`SLDomain` → table `public_domain`, model at [app/models.py:L3116-L3119]):

```bash
docker exec sl-canonical bash -c "PGPASSWORD=mypassword psql -h localhost -U myuser -d simplelogin -c \"SELECT id, domain, created_at FROM public_domain WHERE domain='sl.local';\""
```

```
 id |  domain  |         created_at
----+----------+----------------------------
  2 | sl.local | 2026-07-14 23:12:27.171174
(1 row)
```

The seed also exercises two local-dev short-circuits visible above: `Disable onboarding emails` at [app/models.py:L647] (because `DISABLE_ONBOARDING=true`), and the repeated `Not sending events because webhook is not configured…` at [app/events/event_dispatcher.py:L62] (explained in §Q3.2). The two demo users it creates are `john@wick.com` [app/fake_data.py:L45] and `winston@continental.com` [app/fake_data.py:L236].

> **Why this block's timestamps predate the rest of the document.** This `flask dummy-data` seed is a **one-time database bootstrap** — it runs once against a freshly-migrated database (here `2026-07-14 23:12`, pid `122`) and its effects (the seeded users, domains, and demo rows) persist. The application processes are then (re)started many times against that same seeded database; the request-handling captures from §Q1.2 onward come from the current investigation's restart (`2026-07-15 08:09…`, §Q1.1). The two dates are therefore expected and consistent: the DB was seeded once, the app was booted later. Re-running `flask dummy-data` is deliberately **not** done here because it would reset the database and invalidate the ids used throughout §Q2–§Q4.

### 0.6 The logging backbone (explained once; every log line below follows this shape)

A single logger named `"SL"` is created at [app/log.py:L79] (`LOG = _get_logger("SL")`), set to DEBUG at [app/log.py:L51] (`logger.setLevel(logging.DEBUG)`), and writes to stdout with the format string at [app/log.py:L12-L15]:

```bash
docker exec sl-canonical bash -c 'sed -n "12,15p" /app/app/log.py'
```

```python
_log_format = (
    "%(asctime)s - %(name)s - %(levelname)s - %(process)d - "
    '"%(pathname)s:%(lineno)d" - %(funcName)s() - %(message_id)s - %(message)s'
)
```

The line `>>> init logging <<<` is `print`ed at import time — [app/log.py:L67]. A representative **real** line (the same `GET /` line captured live in §Q1.2) and its field mapping:

```
2026-07-15 08:09:37,907 - SL - DEBUG - 4455 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.00101470947265625
```

| Field | Value | Format token |
|-------|-------|--------------|
| asctime | `2026-07-15 08:09:37,907` | `%(asctime)s` |
| name | `SL` | `%(name)s` |
| levelname | `DEBUG` | `%(levelname)s` |
| process | `4455` | `%(process)d` |
| pathname:lineno | `"/app/server.py:284"` | `"%(pathname)s:%(lineno)d"` |
| funcName | `after_request()` | `%(funcName)s()` |
| message_id | *(empty)* | `%(message_id)s` |
| message | `127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.00101470947265625` | `%(message)s` |

**Important consequence:** Flask/Werkzeug's own request logger is *disabled* at [app/log.py:L70-L71] (`log = logging.getLogger("werkzeug"); log.disabled = True`). That is **why you will NOT see** the classic `127.0.0.1 - - [..] "GET / HTTP/1.1" 200` Werkzeug lines, nor the `* Running on http://127.0.0.1:7777/` line. Instead, requests are logged by SimpleLogin's own `after_request` hook at [server.py:L284].

### 0.7 Known non-canonical sandbox substitutions (disclosed explicitly)

Inside this specific image, a few files differ from the pristine canonical build. They are **image artifacts, not SimpleLogin defects**, and they do not alter the authentication/email readiness behavior under investigation:

```bash
docker exec sl-canonical bash -c 'cd /app && git status --porcelain'
```

```
 M app/spamassassin_utils.py
 M local_data/dkim.key
 M local_data/jwtRS256.key
 M local_data/jwtRS256.key.pub
 M local_data/paddle.key.pub
 M local_data/private-pgp.asc
 M local_data/test_words.txt
 M static/package-lock.json
```

The behaviorally relevant one is `app/spamassassin_utils.py`, image-patched to import the stdlib `re` instead of the canonical `re2`:

```bash
docker exec sl-canonical bash -c 'sed -n "8p" /app/app/spamassassin_utils.py'   # container (patched)
```

```
import re
```

The canonical tracked source instead uses `import re2 as re` at [app/spamassassin_utils.py:L8]. Because of this patch, `email_handler.py` boots normally in this image and its startup banners were captured **live** (§Q1.5/§Q3.5); the banner text and `file:line` sites are the canonical ones — only the underlying RE2 binding differs. **Crucially, all of these differences exist only inside the container's `/app` working tree; the host branch `app_2cd6ee777f8c` where this document is authored is untouched (proven in §Q4.1).**

### 0.8 Security posture of the local debug runtime (trusted-local-only)

**Direct answer.** The web entry point every section below drives is a **Flask debug server** with the **Debug Toolbar** enabled, so the running instance is safe **only** for a single trusted operator on `localhost`. This is not a SimpleLogin defect and it is not something this task changes — `app.run(debug=True, port=7777)` at [server.py:L588] is the *documented* local entry point a first-time operator runs, so observing it is exactly what the Run-First method requires. But an operator must understand three concrete, live-observed exposures that come with it, and must never port-forward this server to an untrusted network. All evidence below is captured live and **redacted** — no secret value or forged credential is reproduced.

**(A) The Debug Toolbar renders `SECRET_KEY` and the full database URI into every normal HTML page.** The toolbar is wired in at [server.py:L576-L582] (`from flask_debugtoolbar import DebugToolbarExtension` … `DebugToolbarExtension(app)`), the debug server is started at [server.py:L588], and the request pipeline even special-cases the toolbar's own routes at [server.py:L263,L278] (`request.path.startswith("/_debug_toolbar")`). Its "Config" panel serializes `app.config` into the body of ordinary `200` pages. Fetching the login page and counting markers (**never printing values**) shows the toolbar and the two sensitive config keys embedded in the body:

```bash
docker exec sl-canonical bash -c 'curl -s -o /tmp/obs/login_body.html http://localhost:7777/auth/login'
docker exec sl-canonical bash -c "grep -oE '_debug_toolbar/static|flDebug|flask-debugtoolbar' /tmp/obs/login_body.html | sort | uniq -c"
docker exec sl-canonical bash -c "for k in SECRET_KEY SQLALCHEMY_DATABASE_URI; do echo -n \"\$k in body: \"; grep -oc \"\$k\" /tmp/obs/login_body.html; done"
```

```
      4 _debug_toolbar/static
    661 flDebug
      1 flask-debugtoolbar
SECRET_KEY in body: 1
SQLALCHEMY_DATABASE_URI in body: 1
```

That the two keys carry **real, sensitive values** (rather than being empty placeholders) is confirmed the same way the toolbar obtains them — from `app.config` — while printing only **lengths and a boolean**, never the values:

```bash
# The one-off app-context invocation prints the usual config-load / GNUPGHOME / init-logging
# preamble (see §0.6) before the results; the in-container `grep` keeps only the three result
# lines so the shown output is exactly what the command emits and is stable run-to-run.
docker exec -i sl-canonical bash -c 'cd /app && CONFIG=/root/sl.env /app/venv/bin/python - 2>&1 | grep -E "present:|credentials"' <<'PY'
from server import create_app
app = create_app()
sk = app.config.get("SECRET_KEY"); db = app.config.get("SQLALCHEMY_DATABASE_URI")
print("SECRET_KEY present:", sk is not None, "len:", len(sk) if sk else 0)
print("SQLALCHEMY_DATABASE_URI present:", db is not None, "len:", len(db) if db else 0)
print("DB URI embeds credentials ('://' and '@'):", ("://" in (db or "")) and ("@" in (db or "")))
PY
```

```
SECRET_KEY present: True len: 6
SQLALCHEMY_DATABASE_URI present: True len: 57
DB URI embeds credentials ('://' and '@'): True
```

The `SECRET_KEY` is 6 bytes because it is sourced from the **already-public** `FLASK_SECRET=secret` in `example.env` (the tracked template) via `config.py`; the 57-byte `SQLALCHEMY_DATABASE_URI` is the `DB_URI` from §0.4 (of the shape `postgresql://myuser:***@localhost:5432/simplelogin`), which embeds the database username and password. Both are **local-dev throwaway values**, but the toolbar would expose *whatever* real secret and DB URI a misconfigured non-local deployment used — that is the risk. (Values are not reproduced here beyond the already-public `example.env` template they derive from.)

**(B) Because the session cookie is signed — not encrypted — disclosing `SECRET_KEY` enables session forgery for any user.** The `slapp` cookie sent on a normal unauthenticated `GET /` (§Q1.2) is a Flask/`itsdangerous` value whose payload is plain base64; only a trailing HMAC (computed with `SECRET_KEY`) protects its integrity. Decoding the **anonymous** payload (which contains no secret) demonstrates the structure:

```bash
docker exec -i sl-canonical /app/venv/bin/python - <<'PY'
import base64
payload = "eyJfZnJlc2giOmZhbHNlLCJfcGVybWFuZW50Ijp0cnVlfQ"   # from a normal unauthenticated GET / (no secret)
print(base64.urlsafe_b64decode(payload + "=" * (-len(payload) % 4)).decode())
PY
```

```
{"_fresh":false,"_permanent":true}
```

Cause → effect: the payload is world-readable and only integrity-bound by the HMAC, so a party who has read `SECRET_KEY` from the disclosure in (A) can mint a **validly-signed** cookie asserting any `user_id` — e.g. the seeded admin `john@wick.com` (`id=1`) — and reach the authenticated `/dashboard/` (the QA checkpoint demonstrated exactly this: a forged session yielding a `200` admin dashboard). Per the read-only constraint and the no-secret-reproduction rule, the working forgery and the secret bytes are **deliberately not reproduced**; the mechanism plus the enabling disclosure in (A) are the evidence.

**(C) Responses carry no browser-hardening headers.** The full header set on both the root redirect and a `200` page (§Q1.2 shows the same) contains only content, caching-neutral, cookie, and server-identity fields — and the `Server` header additionally discloses exact component versions:

```bash
docker exec sl-canonical bash -c "curl -s -D - -o /dev/null http://localhost:7777/auth/login | grep -iE '^(content-security-policy|strict-transport-security|x-frame-options|x-content-type-options|cache-control|server):' || echo '(no CSP/HSTS/X-Frame-Options/X-Content-Type-Options/Cache-Control present)'"
```

```
Server: Werkzeug/1.0.1 Python/3.10.18
```

Only the `Server:` line matches — i.e. **none** of `Content-Security-Policy`, `Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options`, or `Cache-Control` is present, while `Server:` reveals `Werkzeug/1.0.1 Python/3.10.18`. In a browser-facing deployment their absence would leave the app open to clickjacking, MIME-sniffing, and mixed-content downgrade, and would let sensitive pages be cached.

**Why this is fine for THIS investigation, and what production does instead.** The debug server is the canonical *local* entry point (§0.2/§0.3), so exercising it is correct; the toolbar and the interactive debugger are **gated on debug mode** and simply do not load under a production launch. Production runs the WSGI entry point `wsgi.py` under `gunicorn` (README production guide) with debug **off**, which removes the toolbar disclosure and the debugger. **Hardening the product — disabling the toolbar outside debug, adding the security headers, rotating `FLASK_SECRET` — is out of scope for this read-only, observe-only task** (§Q4.1); the only in-scope mitigation an operator needs is the one this setup already follows: the dev server binds `127.0.0.1` inside the container and is reached via `docker exec`, so it is never exposed on an untrusted interface.

---

## §Q1 — Readiness signals (in logs OR UI) that prove the platform is ready

**Direct answer.** The platform announces readiness through a layered set of signals: (a) the **Flask dev-server startup banner** and the `>>> init logging <<<` line in the web log; (b) an **unauthenticated `GET /` redirecting `302 → /auth/login`** (proving the auth wiring is live) followed by the login page returning **`200 OK`**; (c) the **monitor health endpoints** `/git` (200 `dev`), `/live` (200 `live`), and `/exception` (500 by design); (d) the **`job_runner.py`** worker coming up and entering its poll loop; and (e) the **`email_handler.py`** SMTP controller binding its port. The first-run **domain-seeding** line `Add sl.local to SL domain` is a further bootstrap-readiness signal, already shown live in §0.5. Each remaining signal is demonstrated below. (Cookie values in HTTP responses are shown as `<redacted>`; every other header/status/body byte is verbatim — see §Q2 note on redaction.)

### Q1.1 — Flask dev-server startup banner + `>>> init logging <<<`

The web app was started through its real entry point and its PID captured:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && \
  CONFIG=/root/sl.env nohup python server.py > /tmp/obs/server.log 2>&1 & echo "SERVER_PID=$!"'
sleep 20
docker exec sl-canonical bash -c 'sed -n "1,23p" /tmp/obs/server.log'
```

```
SERVER_PID=4447
load config file /root/sl.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/ebkycjreafyddpgrqsnt
Upload files to local dir
>>> init logging <<<
2026-07-15 08:09:04,999 - SL - DEBUG - 4448 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
load config file /root/sl.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/zidjlgfobdvjehrrsgkg
Upload files to local dir
>>> init logging <<<
2026-07-15 08:09:06,549 - SL - DEBUG - 4455 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

- The banner block `* Serving Flask app "server" (lazy loading)` / `* Environment: production` / `* Debug mode: on` is produced by `app.run(debug=True, port=7777)` at [server.py:L588] (Flask's own CLI banner).
- `>>> init logging <<<` is the `print` at import time — [app/log.py:L67].
- **The banner appears twice** because `debug=True` enables Werkzeug's stat reloader, which re-executes the module in a child process (parent pid `4448`, reloaded child pid `4455`; the shell `$!` reported the launcher pid `4447`). This is expected dev-server behavior and directly explains the two `>>> init logging <<<` prints.
- **Cause → effect on the missing "Running on" line:** the usual `* Running on http://127.0.0.1:7777/` is absent precisely because the Werkzeug logger is disabled at [app/log.py:L70-L71]. Readiness is instead confirmed by the `Server:` header (below) and the SL `after_request` log.

> **Per-boot-variable fields (annotated for reproducibility).** The `SERVER_PID` / parent / child pids (`4447 / 4448 / 4455` here), the `GNUPGHOME /tmp/<random>` paths, and the timestamps are assigned fresh on **every** boot and will differ on a reader's run — this capture is one representative boot of the current investigation (`2026-07-15`). The **invariant, reproducible** readiness signals are: the `>>> init logging <<<` line printed **exactly twice** (the reloader double-boot), the `Serving Flask app "server"` / `Environment: production` / `Debug mode: on` banner, and the **absence** of the `* Running on …` line (F-disabled Werkzeug logger, [app/log.py:L70-L71]). The remaining sections were captured during this same `2026-07-15` investigation across a few process restarts (the readiness banners here were re-captured last, so this boot's `08:0x` timestamps are *later* than the `06:xx`–`07:xx` stamps in §Q2–§Q4); consequently their pids and wall-clock times differ from this boot's and from each other's. Each section is internally consistent, and it is the invariant signals — statuses, log messages, state transitions — not the pids or timestamps, that every claim rests on.

The `Server:` header confirms the Werkzeug/Python versions:

```bash
docker exec sl-canonical bash -c "curl -sSI http://localhost:7777/ | grep -i '^Server:'"
```

```
Server: Werkzeug/1.0.1 Python/3.10.18
```

### Q1.2 — Unauthenticated `GET /` → `302` → `/auth/login`, then `200 OK`

This is the key readiness proof that authentication is wired: the root path bounces an anonymous visitor to the login page. The **redirect decision** is made by the `index()` view at [server.py:L250-L255] (`return redirect(url_for("auth.login"))` at [server.py:L255], taken because `current_user.is_authenticated` is `False`); the request is **logged** separately by the `after_request` hook at [server.py:L284].

```bash
docker exec sl-canonical bash -c 'curl -sSi http://localhost:7777/'
```

```
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
Vary: Cookie
Set-Cookie: slapp=<redacted>; Expires=Wed, 22-Jul-2026 08:09:37 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 15 Jul 2026 08:09:37 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to target URL: <a href="/auth/login">/auth/login</a>.  If not click the link.
```

The login page then returns `200 OK` and renders `templates/auth/login.html`:

```bash
docker exec sl-canonical bash -c 'curl -sSi http://localhost:7777/auth/login | sed -n "1,9p"'
docker exec sl-canonical bash -c 'curl -sS http://localhost:7777/auth/login | grep -oiE "name=\"password\"|Log in|forgot my password" | sort -u'
```

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 347790
Vary: Cookie
Set-Cookie: slapp=<redacted>; Expires=Wed, 22-Jul-2026 08:09:38 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 15 Jul 2026 08:09:38 GMT

Log in
forgot my password
name="password"
```

> **Note on the login-page `Content-Length` (dynamic value).** The `347790` above is a per-render snapshot, **not** a fixed constant. The login page is a full HTML document rendered by the Flask **debug** server (`app.run(debug=True, port=7777)` at [server.py:L588]), and its serialized byte size varies from render to render and across environments — re-issuing the same `GET /auth/login` during this investigation produced bodies of, e.g., `220498` and `283794` bytes. What is **stable and the actual readiness signal** is the `200 OK` status plus the `Log in` / `name="password"` / `forgot my password` markers and the `Server: Werkzeug/1.0.1 Python/3.10.18` header. The same dynamic-render caveat applies to the other full rendered HTML pages captured in this document (the registration form in §Q2.1 and the *"Activation Email Sent"* waiting page in §Q2.2, whose `Content-Length` values are likewise per-render snapshots). By contrast, the **small fixed-payload** responses are byte-stable across renders: the `229`-byte `GET /` redirect body (§Q1.2), the `3`-byte `/git` body, and the `4`-byte `/live` body (§Q1.3).

The redirect and the login render are both logged by `after_request` at [server.py:L284]:

```bash
docker exec sl-canonical bash -c 'grep -E "server.py:284.*(GET / |GET /auth/login )" /tmp/obs/server.log | sed -n "1,3p"'
```

```
2026-07-15 08:09:37,907 - SL - DEBUG - 4455 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.00101470947265625
2026-07-15 08:09:38,099 - SL - DEBUG - 4455 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10455107688903809
```

The `200 OK` body markers (`Log in` button, `name="password"` field, `I forgot my password` link) confirm the login template rendered — i.e., the blueprints are wired and the UI is served.

### Q1.3 — Monitor health endpoints (`/git`, `/live`, `/exception`)

These come from the `monitor_bp` blueprint (`url_prefix="/"` at [app/monitor/base.py:L3]).

**`/git`** returns the build SHA1 (HTTP 200) — `git_sha1()` returns `SHA1` at [app/monitor/views.py:L5-L7], where `SHA1 = "dev"` at [app/build_info.py:L1]:

```bash
docker exec sl-canonical bash -c 'curl -sSi http://localhost:7777/git'
```

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 3
Vary: Cookie
Set-Cookie: slapp=<redacted>; Expires=Wed, 22-Jul-2026 08:09:52 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 15 Jul 2026 08:09:52 GMT

dev
```

**`/live`** returns the literal `live` (HTTP 200) — `live()` returns `"live"` at [app/monitor/views.py:L10-L12]:

```bash
docker exec sl-canonical bash -c 'curl -sSi http://localhost:7777/live'
```

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 4
Vary: Cookie
Set-Cookie: slapp=<redacted>; Expires=Wed, 22-Jul-2026 08:09:52 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
Date: Wed, 15 Jul 2026 08:09:52 GMT

live
```

**`/exception`** deliberately raises to exercise error reporting (Sentry) — `test_exception()` does `raise Exception("to make sure sentry works")` at [app/monitor/views.py:L15-L17] — and therefore returns `500` **by design**:

```bash
docker exec sl-canonical bash -c 'curl -sSi http://localhost:7777/exception | sed -n "1,3p"'
```

```
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 5748
```

**Coverage note — `/git` is intentionally excluded from the request log** while `/live` and `/exception` are logged. The exclusion is the guard `and not request.path.startswith("/git")` at [server.py:L279]:

```bash
docker exec sl-canonical bash -c 'sed -n "279p" /app/server.py'
docker exec sl-canonical bash -c 'grep -E "server.py:284.*(GET /live|GET /exception|GET /git)" /tmp/obs/server.log | tail -2'
```

```
            and not request.path.startswith("/git")
2026-07-15 08:09:52,933 - SL - DEBUG - 4455 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /live ImmutableMultiDict([]) 200, takes 0.0005757808685302734
2026-07-15 08:09:53,039 - SL - DEBUG - 4455 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /exception ImmutableMultiDict([]) 500, takes 0.015216827392578125
```

(No `GET /git` line appears in the grep because it is filtered by the guard at [server.py:L279] — the absence is itself the observed proof.)

### Q1.4 — `job_runner.py` worker readiness

The background-job worker was started through its real entry point with its PID captured; it logs the load-config/`init logging` sequence and then enters its silent 10-second poll loop:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && \
  CONFIG=/root/sl.env nohup python job_runner.py > /tmp/obs/job_runner.log 2>&1 & echo "JOBRUNNER_PID=$!"'
sleep 10
docker exec sl-canonical bash -c 'cat /tmp/obs/job_runner.log'
```

```
JOBRUNNER_PID=4538
load config file /root/sl.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/nnhhfixgobbwhqmnewrc
Upload files to local dir
>>> init logging <<<
2026-07-15 08:09:53,775 - SL - DEBUG - 4539 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

The `__main__` poll loop is at [job_runner.py:L329-L347]; with no jobs pending, the loop `time.sleep(10)` at [job_runner.py:L347] produces no further output — the readiness signal here is the clean import / `init logging` start. §Q3.2 drives **real** jobs through this same running worker process and captures the `Take job` line at [job_runner.py:L334], the `ready → taken → done` state transition, and the measured ~10-second poll cadence.

### Q1.5 — `email_handler.py` SMTP controller banners (live)

The email-forwarding engine was started through its real entry point and **booted live** in this image, binding its aiosmtpd controller:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && \
  CONFIG=/root/sl.env nohup python email_handler.py > /tmp/obs/email_handler.log 2>&1 & echo "EMAILHANDLER_PID=$!"'
sleep 10
docker exec sl-canonical bash -c 'cat /tmp/obs/email_handler.log'
```

```
EMAILHANDLER_PID=4558
load config file /root/sl.env
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/vaqafhzhyhwmmtyioupm
Upload files to local dir
>>> init logging <<<
2026-07-15 08:10:04,149 - SL - DEBUG - 4559 - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
2026-07-15 08:10:04,613 - SL - INFO - 4559 - "/app/email_handler.py:2403" - <module>() -  - Listen for port 20381
2026-07-15 08:10:04,615 - SL - DEBUG - 4559 - "/app/email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 20381
```

- `Listen for port 20381` — `LOG.i("Listen for port %s", args.port)` at [email_handler.py:L2403] (default port `20381`).
- `Start mail controller 0.0.0.0 20381` — `LOG.d("Start mail controller %s %s", controller.hostname, controller.port)` at [email_handler.py:L2386], inside `def main(port: int)` [email_handler.py:L2381], after `controller = Controller(MailHandler(), hostname="0.0.0.0", port=port)` at [email_handler.py:L2383] and `controller.start()` at [email_handler.py:L2385].

The port is confirmed open:

```bash
docker exec sl-canonical bash -c 'python3 -c "import socket; s=socket.socket(); print(\"20381 OPEN\" if s.connect_ex((\"127.0.0.1\",20381))==0 else \"20381 CLOSED\"); s.close()"'
```

```
20381 OPEN
```

> **Canonical vs. non-canonical labeling.** These banners are **live-observed** in this image. `email_handler.py` boots because the container's `app/spamassassin_utils.py` is image-patched to `import re` (stdlib) rather than the canonical `import re2 as re` at [app/spamassassin_utils.py:L8] (§0.7). In the pristine canonical container `email_handler.py` boots the same way via `pyre2`, emitting identical banners. The banner text and the `file:line` sites are the canonical ones; only the underlying RE2 binding differs.

---


## §Q2 — The new-user product experience: register → verify → log in → dashboard

**Direct answer.** The typical new-user flow works end-to-end, and each of the three steps has a distinct, observable signal that confirms it succeeded and forwards the user toward the dashboard:

1. **Register** (`POST /auth/register`) returns **`200 OK`** rendering the *"Activation Email Sent"* waiting page, creates the `users` row with `activated = f`, mints a **30-character single-use `ActivationCode`**, and (because `NOT_SEND_EMAIL=true`) **logs** the activation email instead of sending it.
2. **Verify** (`GET /auth/activate?code=…`) flips `users.activated` **`f` → `t`**, **deletes** the single-use code (`1` row → `0` rows), logs the user in, and **`302`-redirects into `/dashboard/`**.
3. **Log in** (`POST /auth/login`) authenticates the credentials and, via `after_login()`, **`302`-redirects into `/dashboard/`**; `GET /dashboard/` then returns **`200 OK`** rendering the alias/mailbox UI.

All observations below were captured against a **temporary probe user** `blitzyprobe@gmail.com` (database `id = 3`) created **solely for this investigation**; it and every artifact it produced are removed in **§Q4** (cleanup), restoring the database to its seeded baseline. Every session cookie in the captured HTTP output is shown as `slapp=<redacted-session-cookie>` — the real value is a signed Flask session and is intentionally not reproduced.

### Q2.1 — Register: the form is served (`GET /auth/register` → `200`)

The registration page is served by `register()` at [app/auth/views/register.py:L31] (`@auth_bp.route("/register", methods=["GET", "POST"])`). The `GET` returns `200 OK` with the WTForms/Flask-WTF form containing a CSRF token, an email field, and a password field:

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/probe_jar.txt; rm -f "$JAR"
curl -sSi -c "$JAR" http://localhost:7777/auth/register | sed -n "1,6p"
echo "--- form markers ---"
curl -sS http://localhost:7777/auth/register | grep -oE "Create Account|name=\"csrf_token\"|name=\"email\"|name=\"password\"" | sort -u'
```

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 213221
Vary: Cookie
Set-Cookie: slapp=<redacted-session-cookie>; Expires=Wed, 22-Jul-2026 07:22:21 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
--- form markers ---
Create Account
name="csrf_token"
name="email"
name="password"
```

The three field markers correspond to the `RegisterForm` (`email`, `password`) and the Flask-WTF CSRF hidden field; `Create Account` is the submit button rendered by `templates/auth/register.html`.

> As with every local HTML page, the register page's exact `Content-Length` varies run-to-run (observed 213221 / 276441 across replays) because the debug build injects the Flask Debug Toolbar; the reproducible, invariant signals are the `200` status, the `csrf_token` / `email` / `password` field markers, and the `Create Account` submit button.

### Q2.2 — Register: successful submission (`POST /auth/register` → `200` waiting page)

Submitting a valid email + password returns `200 OK` rendering `register_waiting_activation.html`. The **visible confirmation** is the *"Activation Email Sent … An email to validate your email is on its way. Please check your inbox"* page, rendered by `render_template("auth/register_waiting_activation.html")` at [app/auth/views/register.py:L104]. The handler logs `create user …` at [app/auth/views/register.py:L85] before `User.create(...)`:

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/probe_jar.txt
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/register | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sS -b "$JAR" -c "$JAR" -D - -o /tmp/obs/register_body.html \
  --data-urlencode "csrf_token=$CSRF" \
  --data-urlencode "email=blitzyprobe@gmail.com" \
  --data-urlencode "password=BlitzyProbe#2026" \
  http://localhost:7777/auth/register | sed -n "1,6p"
echo "--- waiting-page markers ---"
grep -oE "Activation Email Sent|An email to validate your email is on its way.|Please check your inbox" /tmp/obs/register_body.html | head -3'
```

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 740825
Vary: Cookie
Set-Cookie: slapp=<redacted-session-cookie>; Expires=Wed, 22-Jul-2026 07:04:50 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
--- waiting-page markers ---
Activation Email Sent
An email to validate your email is on its way.
Please check your inbox
```

> As with every full rendered HTML page (see the §Q1.2 note), the `Content-Length: 740825` of this debug-toolbar-inflated waiting page is a **per-render snapshot** and varies run-to-run; the reproducible, invariant signals are the `200 OK` status and the three *"Activation Email Sent" / "An email to validate your email is on its way." / "Please check your inbox"* markers.

Server log for the same request — the `create user` line at [app/auth/views/register.py:L85] and the `after_request` completion line at [server.py:L284]:

```bash
docker exec sl-canonical bash -c 'grep -nE "register.py:85|POST /auth/register" /tmp/obs/server.log | tail -2'
```

```
2026-07-15 07:04:50,242 - SL - DEBUG - 270 - "/app/app/auth/views/register.py:85" - register() -  - create user blitzyprobe@gmail.com
2026-07-15 07:04:50,629 - SL - DEBUG - 270 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.4364745616912842
```

### Q2.3 — Register: error / boundary paths

Registration is guarded by two visible error branches, both returning `200` and re-rendering the form with a flashed error:

- **Duplicate email** — when the email already exists, the handler flashes *"Email … already used"* at [app/auth/views/register.py:L82]:

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/dup_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/register | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sS -b "$JAR" -c "$JAR" -o /tmp/obs/dup_body.html -w "http_code=%{http_code}\n" \
  --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=blitzyprobe@gmail.com" \
  --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/register
grep -oiE "Email blitzyprobe@gmail.com already used" /tmp/obs/dup_body.html | head -1'
```

```
http_code=200
Email blitzyprobe@gmail.com already used
```

- **Disallowed / invalid personal email** — an address rejected by the personal-inbox validation flashes *"You cannot use this email address as your personal inbox."* at [app/auth/views/register.py:L75]:

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/inv_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/register | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sS -b "$JAR" -c "$JAR" -o /tmp/obs/inv_body.html -w "http_code=%{http_code}\n" \
  --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=blitzyprobe@sl.local" \
  --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/register
grep -oiE "You cannot use this email address as your personal inbox." /tmp/obs/inv_body.html | head -1'
```

```
http_code=200
You cannot use this email address as your personal inbox.
```

(`blitzyprobe@sl.local` is rejected because `sl.local` is the service's own `EMAIL_DOMAIN` — a user may not register their personal inbox on a SimpleLogin-managed domain.)


### Q2.4 — The activation email: minted code + logged (not sent) under `NOT_SEND_EMAIL`

Registration mints a **30-character single-use activation code**. `send_activation_email()` calls `ActivationCode.create(user_id=user.id, code=random_string(30))` at [app/auth/views/register.py:L120]. Reading the `users`/`activation_code` rows for the probe user directly from PostgreSQL confirms the freshly created, still-inactive account and the exact code length:

```bash
docker exec sl-canonical su postgres -c "psql -d simplelogin -c \"
SELECT u.id, u.email, u.activated, a.code, length(a.code) AS code_len
FROM users u JOIN activation_code a ON a.user_id = u.id
WHERE u.email = 'blitzyprobe@gmail.com';\""
```

```
 id |         email         | activated |              code              | code_len
----+-----------------------+-----------+--------------------------------+----------
  3 | blitzyprobe@gmail.com | f         | rvsyrqprwkbusdnlgwjyagrtobxynj |       30
(1 row)
```

Because `NOT_SEND_EMAIL=true` (§0.4), the mailer **logs** the activation email rather than delivering it. Two lines prove the full transactional send path fired: `send_email()` at [app/email_utils.py:L303] and, one step deeper, `MailSender.send()` at [app/mail_sender.py:L131] (the `NOT_SEND_EMAIL` short-circuit branch at [app/mail_sender.py:L130-L136]):

```bash
docker exec sl-canonical bash -c 'grep -nE "email_utils.py:303|mail_sender.py:131" /tmp/obs/server.log | grep "blitzyprobe@gmail.com" | tail -2'
```

```
2026-07-15 07:04:50,538 - SL - DEBUG - 270 - "/app/app/email_utils.py:303" - send_email() -  - send email to blitzyprobe@gmail.com, subject 'Just one more step to join SimpleLogin'
2026-07-15 07:04:50,540 - SL - DEBUG - 270 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'blitzyprobe@gmail.com'
```

This is the observable proof that identity-verification email is wired and executing; the `subject 'Just one more step to join SimpleLogin'` is the activation email a real deployment would deliver. (The deeper mail-path semantics — transactional `send_email` vs. forwarding `sl_sendmail` — are dissected in **§Q3**.)

### Q2.5 — Verify: `GET /auth/activate?code=…` flips `activated f → t`, consumes the single-use code, and `302` → `/dashboard/`

**Direct answer.** Following the activation link is the exact boundary that confirms address verification: the `users.activated` flag transitions **`f` → `t`**, the `ActivationCode` row is **deleted** (single-use: `1` → `0`), the user is logged in, and the response is a **`302` redirect to `/dashboard/`**. The handler is `activate()` at [app/auth/views/activate.py:L13]; on success it sets `user.activated = True` at [app/auth/views/activate.py:L49], `login_user(user)` at [app/auth/views/activate.py:L50], `ActivationCode.delete(...)` at [app/auth/views/activate.py:L53], and redirects with the `redirect user to dashboard` log at [app/auth/views/activate.py:L66].

The capture below **resolves the pending activation code dynamically from the database** (no hard-coded value — the 30-char code differs every run), reads the state **before**, issues the real `GET /auth/activate?code=$CODE`, and reads the state **after** in a single sequence. The resolved code is also persisted to a scratch file so the single-use "reused code" test in §Q2.6 can replay the *exact* code that was just consumed:

```bash
docker exec sl-canonical bash -c '
# Resolve the pending activation code dynamically (minted in §Q2.2/§Q2.4) — no hard-coded value:
CODE=$(su postgres -c "psql -d simplelogin -tAc \"SELECT a.code FROM activation_code a JOIN users u ON a.user_id=u.id WHERE u.email='\''blitzyprobe@gmail.com'\'';\"")
echo "resolved activation code: $CODE (length ${#CODE})"
echo "$CODE" > /tmp/obs/probe_activation_code.txt   # persisted so the reused-code test in §Q2.6 can replay it
echo "=== BEFORE (activated, code_rows) ==="
su postgres -c "psql -d simplelogin -c \"SELECT u.activated, count(a.*) AS code_rows FROM users u LEFT JOIN activation_code a ON a.user_id=u.id WHERE u.email='\''blitzyprobe@gmail.com'\'' GROUP BY u.activated;\""
echo "=== GET /auth/activate?code=\$CODE ==="
JAR=/tmp/obs/activate_jar.txt; rm -f "$JAR"
curl -sSi -c "$JAR" "http://localhost:7777/auth/activate?code=$CODE" | sed -n "1,6p"
echo "=== AFTER (activated, code_rows) ==="
su postgres -c "psql -d simplelogin -c \"SELECT u.activated, count(a.*) AS code_rows FROM users u LEFT JOIN activation_code a ON a.user_id=u.id WHERE u.email='\''blitzyprobe@gmail.com'\'' GROUP BY u.activated;\""'
```

```
resolved activation code: rvsyrqprwkbusdnlgwjyagrtobxynj (length 30)
=== BEFORE (activated, code_rows) ===
 activated | code_rows
-----------+-----------
 f         |         1
(1 row)

=== GET /auth/activate?code=$CODE ===
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
Vary: Cookie
Set-Cookie: slapp=<redacted-session-cookie>; Expires=Wed, 22-Jul-2026 07:05:12 GMT; HttpOnly; Path=/; SameSite=Lax
=== AFTER (activated, code_rows) ===
 activated | code_rows
-----------+-----------
 t         |         0
(1 row)
```

The `Location: …/dashboard/` header is the forward-into-dashboard signal. Server log for the same request — the `redirect user to dashboard` line at [app/auth/views/activate.py:L66] and the `after_request` line at [server.py:L284] (note the `code` query arg is echoed by the logger):

```bash
docker exec sl-canonical bash -c 'grep -nE "activate.py:66|GET /auth/activate" /tmp/obs/server.log | tail -2'
```

```
2026-07-15 07:05:12,989 - SL - DEBUG - 270 - "/app/app/auth/views/activate.py:66" - activate() -  - redirect user to dashboard
2026-07-15 07:05:12,990 - SL - DEBUG - 270 - "/app/server.py:284" - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code', 'rvsyrqprwkbusdnlgwjyagrtobxynj')]) 302, takes 0.05293393135070801
```

Two further activation side-effects are observable in the same request. First, `activate()` flashes *"Your account has been activated"* at [app/auth/views/activate.py:L56] (shown on the dashboard after the redirect). Second, it calls `email_utils.send_welcome_email(user)` at [app/auth/views/activate.py:L58], which — under `NOT_SEND_EMAIL` — logs a **welcome email** addressed to the user's **auto-created default alias** `simplelogin-newsletter.word444@sl.local` (proving the default alias was provisioned at registration; in this run the alias has database `id = 20` — the random word and id differ every run):

```bash
docker exec sl-canonical bash -c 'grep -nE "Welcome to SimpleLogin" /tmp/obs/server.log | tail -2'
```

```
2026-07-15 07:05:12,988 - SL - DEBUG - 270 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.word444@sl.local, subject 'Welcome to SimpleLogin'
2026-07-15 07:05:12,989 - SL - DEBUG - 270 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.word444@sl.local'
```

### Q2.6 — Verify: error / boundary paths (bad code, reused code, expired code)

Activation has three visible failure branches, all returning **`400`** with a flashed error:

- **Unknown code** → *"Activation code cannot be found"* at [app/auth/views/activate.py:L28-L36] (the `if not activation_code:` branch):

```bash
docker exec sl-canonical bash -c '
curl -sS -o /tmp/obs/badcode.html -w "http_code=%{http_code}\n" "http://localhost:7777/auth/activate?code=NON_EXISTENT_CODE_123456789012"
grep -oiE "Activation code cannot be found" /tmp/obs/badcode.html | head -1'
```

```
http_code=400
Activation code cannot be found
```

- **Reused code** (single-use proof) → re-requesting the **same** code that succeeded in §Q2.5 now returns `400` with the same message, because the row was deleted at [app/auth/views/activate.py:L53]:

```bash
docker exec sl-canonical bash -c '
CODE=$(cat /tmp/obs/probe_activation_code.txt)   # the exact code consumed in §Q2.5
echo "replaying the now-consumed code: $CODE"
curl -sS -o /tmp/obs/reused.html -w "http_code=%{http_code}\n" "http://localhost:7777/auth/activate?code=$CODE"
grep -oiE "Activation code cannot be found" /tmp/obs/reused.html | head -1'
```

```
replaying the now-consumed code: rvsyrqprwkbusdnlgwjyagrtobxynj
http_code=400
Activation code cannot be found
```

- **Expired code** → *"Activation code was expired"* at [app/auth/views/activate.py:L38-L46] (the `if activation_code.is_expired():` branch — a plain `if` that functions as an effective else-if because the preceding not-found block at [app/auth/views/activate.py:L28-L36] `return`s first — where `is_expired()` — defined at [app/models.py:L1214] — evaluates the comparison `self.expired < arrow.now()` at [app/models.py:L1215]). To exercise this branch we inserted a temporary `ActivationCode` whose `expired` timestamp is in the past (this row is removed in §Q4):

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 - <<PY 2>/dev/null | grep -E "created expired code"
import arrow
from app.models import ActivationCode, User
from app.db import Session
u = User.get_by(email="blitzyprobe@gmail.com")          # resolve the probe id dynamically
ac = ActivationCode.create(user_id=u.id, code="blitzyexpiredcode000000000000x",
                           expired=arrow.now().shift(hours=-2), flush=True)
Session.commit()
print("created expired code id=%s user_id=%s is_expired=%s" % (ac.id, u.id, ac.is_expired()))
PY'
docker exec sl-canonical bash -c '
curl -sS -o /tmp/obs/expired.html -w "http_code=%{http_code}\n" "http://localhost:7777/auth/activate?code=blitzyexpiredcode000000000000x"
grep -oiE "Activation code was expired" /tmp/obs/expired.html | head -1'
```

```
created expired code id=2 user_id=3 is_expired=True
http_code=400
Activation code was expired
```


### Q2.7 — Log in: successful authentication → `302` → `/dashboard/`

**Direct answer.** A valid login returns **`302 FOUND`** with `Location: …/dashboard/`, forwarding the user into the dashboard. The endpoint is `login()` at [app/auth/views/login.py:L21]; on the success branch (`else:` at [app/auth/views/login.py:L70]) it calls `after_login(user, next_url)` at [app/auth/views/login.py:L72], which logs `log user … in` at [app/auth/views/login_utils.py:L35], calls `login_user(user)` at [app/auth/views/login_utils.py:L36], and redirects to `dashboard.index` at [app/auth/views/login_utils.py:L45].

This is demonstrated with the seeded admin user `john@wick.com / password` (from `flask dummy-data`, §0.5) — a known-good, already-activated account. (The just-verified probe user from §Q2.5 was *auto-logged-in* by `activate()`'s own `login_user()` call; an explicit login of any activated user takes the identical `after_login` path shown here.) Flask-Login sessions are **signed client-side cookies**, so a login creates **no database row** — the only artifact is the cookie jar, removed in §Q4:

```bash
docker exec sl-canonical bash -c '
JJAR=/tmp/obs/john_jar.txt; rm -f "$JJAR"
CSRF=$(curl -sS -c "$JJAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sSi -b "$JJAR" -c "$JJAR" \
  --data-urlencode "csrf_token=$CSRF" \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" \
  http://localhost:7777/auth/login | sed -n "1,4p"'
```

```
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/dashboard/
```

Server log for the same request — the two `after_login()` lines at [app/auth/views/login_utils.py:L35] and [app/auth/views/login_utils.py:L44], then the `after_request` line at [server.py:L284]:

```bash
docker exec sl-canonical bash -c 'grep -nE "login_utils.py:35|login_utils.py:44|POST /auth/login" /tmp/obs/server.log | tail -3'
```

```
2026-07-15 07:10:39,330 - SL - DEBUG - 270 - "/app/app/auth/views/login_utils.py:35" - after_login() -  - log user <User 1 John Wick john@wick.com> in
2026-07-15 07:10:39,330 - SL - DEBUG - 270 - "/app/app/auth/views/login_utils.py:44" - after_login() -  - redirect user to dashboard
2026-07-15 07:10:39,331 - SL - DEBUG - 270 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.2405257225036621
```

### Q2.8 — Log in: error / boundary paths (bad credentials, unactivated, disabled, rate-limit)

The `login()` handler has a cascade of guarded branches [app/auth/views/login.py:L45-L72]. All four failure signals below were observed live. For the *unactivated* and *disabled* branches (which are only reachable **after** the password check passes) we toggled the corresponding flag on the probe user to make the branch manifest, then restored it — the application genuinely evaluates `user.disabled` / `not user.activated`; only the flag value was set up.

- **Bad credentials** → `200` re-render with *"Email or password incorrect"* (`if not user or not user.check_password(...)` at [app/auth/views/login.py:L45], flash at [app/auth/views/login.py:L49]; this branch also sets `g.deduct_limit = True` at [app/auth/views/login.py:L47], which is what the rate-limiter counts):

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/badlogin_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sS -b "$JAR" -c "$JAR" -o /tmp/obs/badlogin.html -w "http_code=%{http_code}\n" \
  --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=WRONGPASSWORD" http://localhost:7777/auth/login
grep -oiE "Email or password incorrect" /tmp/obs/badlogin.html | head -1'
```

```
http_code=200
Email or password incorrect
```

- **Unactivated account** → `200` re-render with *"Please check your inbox for the activation email…"* (`elif not user.activated:` at [app/auth/views/login.py:L63], flash at [app/auth/views/login.py:L65-L68]). This branch sets `show_resend_activation = True` at [app/auth/views/login.py:L64], which renders a **resend button** — it does **not** auto-resend the email (verified: no `send email …` log line was emitted at this request's timestamp):

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.activated=False; Session.commit(); print(\"probe activated=\",u.activated)" 2>/dev/null | grep "probe activated="'
docker exec sl-canonical bash -c '
JAR=/tmp/obs/unact_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sS -b "$JAR" -c "$JAR" -o /tmp/obs/unact.html -w "http_code=%{http_code}\n" \
  --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=blitzyprobe@gmail.com" \
  --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/login
grep -oiE "Please check your inbox for the activation email" /tmp/obs/unact.html | head -1'
```

```
probe activated= False
http_code=200
Please check your inbox for the activation email
```

- **Disabled account** → `200` re-render with *"Your account is disabled. Please contact SimpleLogin team…"* (`elif user.disabled:` at [app/auth/views/login.py:L51], flash at [app/auth/views/login.py:L52-L55]):

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.activated=True; u.disabled=True; Session.commit(); print(\"probe activated=\",u.activated,\"disabled=\",u.disabled)" 2>/dev/null | grep "probe activated="'
docker exec sl-canonical bash -c '
JAR=/tmp/obs/dis_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sS -b "$JAR" -c "$JAR" -o /tmp/obs/dis.html -w "http_code=%{http_code}\n" \
  --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=blitzyprobe@gmail.com" \
  --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/login
grep -oiE "Your account is disabled. Please contact SimpleLogin team" /tmp/obs/dis.html | head -1
# restore the flag
cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.disabled=False; Session.commit()" >/dev/null 2>&1'
```

```
probe activated= True disabled= True
http_code=200
Your account is disabled. Please contact SimpleLogin team
```

- **Rate-limit** → the endpoint is decorated `@limiter.limit("10/minute", deduct_when=lambda r: hasattr(g, "deduct_limit") and g.deduct_limit)` at [app/auth/views/login.py:L22-L24]. Because `deduct_limit` is set **only** on the bad-credentials branch [app/auth/views/login.py:L47], only failed logins count against the budget. Firing consecutive bad logins, the first ten return `200` and the eleventh flips to **`429 TOO MANY REQUESTS`**:

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/rl_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
# The budget is a ROLLING 60s window keyed by client IP (ip:127.0.0.1 here), and ONLY failed
# logins deduct (deduct_when → g.deduct_limit, set on the bad-credentials branch). Any failed
# login in the preceding minute — e.g. the bad-credentials test above — would consume slots and
# shift the boundary earlier. Drain the window first (61s with ZERO failed logins) so the count
# starts clean and the boundary lands deterministically at attempt 11:
sleep 61
for i in $(seq 1 12); do
  CODE=$(curl -sS -b "$JAR" -c "$JAR" -o /dev/null -w "%{http_code}" \
    --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=john@wick.com" \
    --data-urlencode "password=WRONGPASS" http://localhost:7777/auth/login)
  echo "attempt $i -> HTTP $CODE"
done'
```

```
attempt 1 -> HTTP 200
attempt 2 -> HTTP 200
attempt 3 -> HTTP 200
attempt 4 -> HTTP 200
attempt 5 -> HTTP 200
attempt 6 -> HTTP 200
attempt 7 -> HTTP 200
attempt 8 -> HTTP 200
attempt 9 -> HTTP 200
attempt 10 -> HTTP 200
attempt 11 -> HTTP 429
attempt 12 -> HTTP 429
```

The `429` status line and its `after_request` log line (rate-limit rejections are still logged at [server.py:L284]):

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/rl_jar.txt
curl -sSi -b "$JAR" --data-urlencode "csrf_token=x" --data-urlencode "email=john@wick.com" \
  --data-urlencode "password=WRONGPASS" http://localhost:7777/auth/login | sed -n "1,3p"
grep -E "POST /auth/login.* 429" /tmp/obs/server.log | tail -1'
```

```
HTTP/1.0 429 TOO MANY REQUESTS
Content-Type: text/html; charset=utf-8
Content-Length: 5622
2026-07-15 07:15:25,210 - SL - DEBUG - 270 - "/app/server.py:284" - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 429, takes 0.0011665821075439453
```

(The `10/minute` counter lives in the limiter's **in-memory** storage as a **rolling 60-second window** keyed by client IP — `__key_func` returns `ip:127.0.0.1` for unauthenticated requests at [app/extensions.py:L14-L20], and `limiter = Limiter(key_func=__key_func)` at [app/extensions.py:L23] uses the default in-memory backend — so it drains on its own and creates no persistent artifact, which is exactly why the `sleep 61` drain restores a clean budget. Because `deduct_when` only counts the bad-credentials branch, the *unactivated* and *disabled* tests above — both issued with the **correct** password — never deducted; only the one bad-credentials probe did. The probe's `activated`/`disabled` flags were restored to `t`/`f` immediately after the two branches above.)

### Q2.9 — Log in: MFA and scheduled-deletion branches (exercised live)

`after_login()` and `login()` contain three additional branches that a plain account does **not** traverse (our probe and `john` fall straight through to the dashboard). Each was **exercised live** by setting the single controlling flag on the probe user, observing the branch at runtime, then **restoring the flag** — the application genuinely evaluates `user.fido_enabled()` / `user.enable_otp` / `user.delete_on`; only the flag value was set up.

- **FIDO/WebAuthn 2FA** — `if user.fido_enabled():` at [app/auth/views/login_utils.py:L20] — true precisely when `fido_uuid is not None` at [app/models.py:L1033-L1036] — stashes `session[MFA_USER_ID] = user.id` at [app/auth/views/login_utils.py:L23] and redirects to `auth.fido` at [app/auth/views/login_utils.py:L25]. FIDO is the leading `if`, so it wins over OTP when both are set. Setting `fido_uuid` and logging in yields **`302 → /auth/fido`**:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.fido_uuid=\"blitzy-fido-uuid-0001\"; Session.commit(); print(\"set fido_uuid=\",u.fido_uuid,\"fido_enabled=\",u.fido_enabled())" 2>/dev/null | grep "set fido_uuid"'
docker exec sl-canonical bash -c '
JAR=/tmp/obs/fido_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sSi -b "$JAR" -c "$JAR" --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=blitzyprobe@gmail.com" --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/login | grep -iE "^HTTP|^Location:"
# restore the flag
cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.fido_uuid=None; Session.commit()" >/dev/null 2>&1'
```

```
set fido_uuid= blitzy-fido-uuid-0001 fido_enabled= True
HTTP/1.0 302 FOUND
Location: http://localhost:7777/auth/fido
```

- **TOTP/OTP 2FA** — `elif user.enable_otp:` at [app/auth/views/login_utils.py:L28] stashes the MFA user id at [app/auth/views/login_utils.py:L29] and redirects to `auth.mfa` at [app/auth/views/login_utils.py:L31]. Setting `enable_otp=True` (with `fido_uuid` unset) and logging in yields **`302 → /auth/mfa`**:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.enable_otp=True; Session.commit(); print(\"set enable_otp=\",u.enable_otp,\"fido_uuid=\",u.fido_uuid)" 2>/dev/null | grep "set enable_otp"'
docker exec sl-canonical bash -c '
JAR=/tmp/obs/otp_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sSi -b "$JAR" -c "$JAR" --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=blitzyprobe@gmail.com" --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/login | grep -iE "^HTTP|^Location:"
# restore the flag
cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.enable_otp=False; Session.commit()" >/dev/null 2>&1'
```

```
set enable_otp= True fido_uuid= None
HTTP/1.0 302 FOUND
Location: http://localhost:7777/auth/mfa
```

- **Scheduled deletion** — in `login()`, `elif user.delete_on is not None:` at [app/auth/views/login.py:L57] flashes *"Your account is scheduled to be deleted on …"* at [app/auth/views/login.py:L58-L61] and does **not** log the user in (a `200` re-render). Setting `delete_on` to a future date (here `now + 30 days`, so the exact timestamp is run-specific) and logging in with the correct password:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
import arrow
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.delete_on=arrow.now().shift(days=30); Session.commit(); print(\"set delete_on=\",u.delete_on)" 2>/dev/null | grep "set delete_on"'
docker exec sl-canonical bash -c '
JAR=/tmp/obs/sched_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
curl -sS -b "$JAR" -c "$JAR" -o /tmp/obs/sched.html -w "http_code=%{http_code}\n" --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=blitzyprobe@gmail.com" --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/login
grep -oiE "Your account is scheduled to be deleted on [^<\"]+" /tmp/obs/sched.html | head -1
# restore the flag
cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=User.get_by(email=\"blitzyprobe@gmail.com\"); u.delete_on=None; Session.commit()" >/dev/null 2>&1'
```

```
set delete_on= 2026-08-14T07:18:03.631128+00:00
http_code=200
Your account is scheduled to be deleted on 2026-08-14T07:18:03.631128+00:00
```

When none of these apply, control reaches the fall-through at [app/auth/views/login_utils.py:L35-L45] — `login_user()` then redirect to `dashboard.index` — which is exactly the path observed in §Q2.7.

### Q2.10 — Dashboard: the login-success landing (`GET /dashboard/`)

**Direct answer.** With a valid session, `GET /dashboard/` returns **`200 OK`** rendering the alias/mailbox management UI; without a session it **`302`-redirects to `/auth/login`** with a `next=` back-reference. The endpoint is `index()` at [app/dashboard/views/index.py:L67] (route `@dashboard_bp.route("/", …)`), which renders `dashboard/index.html` at [app/dashboard/views/index.py:L215].

Authorized (reusing the logged-in cookie jar) — `200` with alias-management UI markers:

```bash
docker exec sl-canonical bash -c '
curl -sSi -b /tmp/obs/john_jar.txt http://localhost:7777/dashboard/ | sed -n "1,3p"
echo "--- UI markers ---"
curl -sS -b /tmp/obs/john_jar.txt http://localhost:7777/dashboard/ | grep -oiE "New Custom Alias|Random Alias|create-custom-email" | sort -u'
```

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 652726
--- UI markers ---
New Custom Alias
Random Alias
create-custom-email
random alias
```

> The `200` body is large and its exact `Content-Length` varies run-to-run (observed 651210 / 652726 across replays) because the local debug build injects the Flask Debug Toolbar into every HTML page; the invariant, reproducible signals are the `200` status and the alias-management UI markers above (the case-insensitive `grep -oi` also surfaces the lowercase `random alias` label alongside the `Random Alias` button, hence four lines).

Unauthorized (no cookie) — `302` to `/auth/login` with `next=%2Fdashboard%2F…` (the `@login_required` guard on the dashboard blueprint):

```bash
docker exec sl-canonical bash -c 'curl -sSi http://localhost:7777/dashboard/ | sed -n "1,5p"'
```

```
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 277
Location: http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F
Vary: Cookie
```

This closes the happy-path loop: **register** (200 waiting page) → **verify** (`activated f→t`, single-use code consumed, `302 → /dashboard/`) → **log in** (`302 → /dashboard/`) → **dashboard** (`200`, alias/mailbox UI). Every step produced a distinct, observed signal, and the authenticated user is forwarded into `/dashboard/` exactly as the flow requires.

The remaining subsections exercise the **additional authentication-flow surfaces** a complete operator walkthrough must cover: logging out and the session boundary (§Q2.11), CSRF protection on the state-changing forms (§Q2.12), registration input boundaries including XSS- and SQL-injection-shaped input (§Q2.13), `next=` open-redirect safety (§Q2.14), and the HTTP method matrix (§Q2.15). Each is exercised live below.

### Q2.11 — Log out: the session boundary (`GET /auth/logout` → `302` → `/auth/login`, cookies expired)

**Direct answer.** `GET /auth/logout` tears down the session and returns **`302` → `/auth/login`**, expiring the `slapp` session cookie (plus `mfa` and `dark-mode`) via `Set-Cookie … Expires=Thu, 01-Jan-1970 … Max-Age=0`; a subsequent `GET /dashboard/` with the same jar is no longer authenticated and **`302`-redirects back to `/auth/login`**. The handler is `logout()` at [app/auth/views/logout.py:L9]: it calls `logout_session()` at [app/auth/views/logout.py:L10], flashes *"You are logged out"* at [app/auth/views/logout.py:L11], redirects to `auth.login` at [app/auth/views/logout.py:L12], and deletes the three cookies at [app/auth/views/logout.py:L13-L15] (`SESSION_COOKIE_NAME = "slapp"`, [app/config.py:L199]). Demonstrated end-to-end with the seeded `john@wick.com`:

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/logout_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
echo "=== 1) log in (establish session) ==="
curl -sS -b "$JAR" -c "$JAR" -o /dev/null -w "login http_code=%{http_code}\n" \
  --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=john@wick.com" --data-urlencode "password=password" http://localhost:7777/auth/login
echo "=== 2) GET /dashboard/ WITH session ==="
curl -sS -b "$JAR" -o /dev/null -w "dashboard(before logout) http_code=%{http_code}\n" http://localhost:7777/dashboard/
echo "=== 3) GET /auth/logout ==="
curl -sSi -b "$JAR" -c "$JAR" http://localhost:7777/auth/logout | sed -n "1,10p"
echo "=== 4) GET /dashboard/ AFTER logout (same jar) ==="
curl -sSi -b "$JAR" http://localhost:7777/dashboard/ | sed -n "1,4p"'
```

```
=== 1) log in (establish session) ===
login http_code=302
=== 2) GET /dashboard/ WITH session ===
dashboard(before logout) http_code=200
=== 3) GET /auth/logout ===
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 229
Location: http://localhost:7777/auth/login
Set-Cookie: slapp=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: mfa=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Set-Cookie: dark-mode=; Expires=Thu, 01-Jan-1970 00:00:00 GMT; Max-Age=0; Path=/
Vary: Cookie
Set-Cookie: slapp=<redacted-flash-cookie>; Expires=Wed, 22-Jul-2026 07:18:56 GMT; HttpOnly; Path=/; SameSite=Lax
Server: Werkzeug/1.0.1 Python/3.10.18
=== 4) GET /dashboard/ AFTER logout (same jar) ===
HTTP/1.0 302 FOUND
Content-Type: text/html; charset=utf-8
Content-Length: 277
Location: http://localhost:7777/auth/login?next=%2Fdashboard%2F%3F
```

**Cause → effect.** The three `Expires=Thu, 01-Jan-1970 … Max-Age=0` lines are the browser-side cookie-expiry instruction emitted by `response.delete_cookie(...)` at [app/auth/views/logout.py:L13-L15]. (Flask *also* sets a fresh short-lived `slapp` cookie on this same response to carry the *"You are logged out"* flash message — that cookie holds only the flash, not an authenticated identity, which is exactly why step 4 is unauthenticated.) Step 4 returning `302 → /auth/login?next=%2Fdashboard%2F%3F` is the definitive session-boundary signal: the dashboard's `@login_required` guard no longer sees a logged-in user.

### Q2.12 — CSRF protection on register and login

**Direct answer.** Both state-changing forms are guarded by Flask-WTF CSRF: a `POST` with a **missing or invalid** `csrf_token` fails `form.validate_on_submit()` and the handler falls through to a **`200` re-render of the form** — **no user row is created** (register) and **no session is granted** (login). In `register()` the entire creation path is nested under `if form.validate_on_submit():` at [app/auth/views/register.py:L45]; in `login()` under the same guard at [app/auth/views/login.py:L40]. Because CSRF validation is part of `validate_on_submit()`, the create/authenticate branches are never entered.

```bash
docker exec sl-canonical bash -c '
echo "=== register WITHOUT csrf_token ==="
curl -sS -o /tmp/obs/nocsrf_reg.html -w "register(no csrf) http_code=%{http_code}\n" \
  --data-urlencode "email=blitzycsrf@gmail.com" --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/register
echo -n "users row for blitzycsrf@gmail.com: "; su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM users WHERE email='\''blitzycsrf@gmail.com'\'';\""
echo "=== register WITH an INVALID csrf_token ==="
curl -sS -o /tmp/obs/badcsrf_reg.html -w "register(bad csrf) http_code=%{http_code}\n" \
  --data-urlencode "csrf_token=deadbeef-not-a-valid-token" \
  --data-urlencode "email=blitzycsrf2@gmail.com" --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/register
echo -n "users row for blitzycsrf2@gmail.com: "; su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM users WHERE email='\''blitzycsrf2@gmail.com'\'';\""
echo "=== login WITHOUT csrf_token (correct password) ==="
curl -sSi -o /tmp/obs/nocsrf_login.txt -w "login(no csrf) http_code=%{http_code}\n" \
  --data-urlencode "email=john@wick.com" --data-urlencode "password=password" http://localhost:7777/auth/login
echo -n "status line: "; sed -n "1p" /tmp/obs/nocsrf_login.txt
echo -n "redirects to dashboard? "; grep -ciE "Location: .*dashboard" /tmp/obs/nocsrf_login.txt'
```

```
=== register WITHOUT csrf_token ===
register(no csrf) http_code=200
users row for blitzycsrf@gmail.com: 0
=== register WITH an INVALID csrf_token ===
register(bad csrf) http_code=200
users row for blitzycsrf2@gmail.com: 0
=== login WITHOUT csrf_token (correct password) ===
login(no csrf) http_code=200
status line: HTTP/1.0 200 OK
redirects to dashboard? 0
```

**Cause → effect.** Every case returns `200` (a form re-render), **not** a `302` and **not** a persisted row: `users` count stays `0` for both throwaway register emails, and the correct-password login yields `HTTP/1.0 200 OK` with **zero** `Location: …/dashboard` headers (no authenticated redirect). The CSRF failure short-circuits `validate_on_submit()` before the password is ever checked — which is also why this case does **not** consume a rate-limit slot (the `g.deduct_limit` bad-credentials branch is never reached).

### Q2.13 — Registration input boundaries (empty, malformed, service-domain, XSS-shaped, SQL-shaped)

**Direct answer.** Registration validates its input and does not create rows for bad addresses: an **empty** email is caught by `validators.DataRequired()` on the form field [app/auth/views/register.py:L24] (*"This field is required"*); a **malformed**, **service-domain** (`@sl.local`), or **XSS-shaped** email is rejected by `email_can_be_used_as_mailbox()` at [app/auth/views/register.py:L74] (*"You cannot use this email address as your personal inbox."*). A **SQL-injection-shaped** email is neutralized differently and more fundamentally — it is stored as **inert literal data** by the parameterized ORM, so no injection executes (detailed below). All five are exercised with a **valid** CSRF token so that the *email* validator — not CSRF — is what rejects them:

```bash
docker exec sl-canonical bash -c '
JAR=/tmp/obs/boundary_jar.txt; rm -f "$JAR"
CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/register | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
echo "users BEFORE: $(su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM users;\"")"
run() { local label="$1"; local email="$2"
  local code=$(curl -sS -b "$JAR" -c "$JAR" -o /tmp/obs/bnd.html -w "%{http_code}" \
    --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=$email" \
    --data-urlencode "password=BlitzyProbe#2026" http://localhost:7777/auth/register)
  local msg=$(grep -oiE "You cannot use this email address as your personal inbox.|This field is required" /tmp/obs/bnd.html | head -1)
  local rows=$(su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM users WHERE email='\''$email'\'';\"")
  printf "%-16s http=%s  rows=%s  msg=%s\n" "$label" "$code" "$rows" "$msg"; }
run "empty"          ""
run "malformed"      "notanemail"
run "service-domain" "probe@sl.local"
run "xss-like"       "<script>alert(1)</script>@x.com"
echo -n "XSS reflected unescaped in body? matches="; grep -c "<script>alert(1)</script>" /tmp/obs/bnd.html
run "sql-like"       "x'\'' OR 1=1--@x.com"
echo "users AFTER:  $(su postgres -c "psql -d simplelogin -tAc \"SELECT count(*) FROM users;\"")"
echo "=== what the SQL-shaped input actually stored (safe ORM read) ==="
cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=Session.query(User).order_by(User.id.desc()).first(); print(\"newest user id=%s email=%r\" % (u.id, u.email))" 2>/dev/null | grep "newest user"
# self-clean: remove the throwaway SQL-shaped user (FK-safe ORM delete)
cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env python3 -c "
from app.models import User; from app.db import Session
u=Session.query(User).filter(User.email==\"x'\''or1=1--@x.com\").first()
if u: User.delete(u.id); Session.commit()
print(\"users after self-clean:\", Session.query(User).count())" 2>/dev/null | grep "users after"'
```

```
users BEFORE: 3
empty            http=200  rows=0  msg=This field is required
malformed        http=200  rows=0  msg=You cannot use this email address as your personal inbox.
service-domain   http=200  rows=0  msg=You cannot use this email address as your personal inbox.
xss-like         http=200  rows=0  msg=You cannot use this email address as your personal inbox.
XSS reflected unescaped in body? matches=0
sql-like         http=200  rows=4  msg=
users AFTER:  4
=== what the SQL-shaped input actually stored (safe ORM read) ===
newest user id=4 email="x'or1=1--@x.com"
users after self-clean: 3
```

**Cause → effect (read each result by name).**
- **empty** → `200`, `rows=0`, *"This field is required"* — `validators.DataRequired()` fails validation, so the create branch is never reached.
- **malformed / service-domain / xss-like** → `200`, `rows=0`, *"You cannot use this email address as your personal inbox."* — all three fail `email_can_be_used_as_mailbox()` at [app/auth/views/register.py:L74]. The XSS payload is additionally **not reflected**: `matches=0` for the raw `<script>…` string in the response body (Jinja2 autoescaping renders it inert).
- **sql-like** (`x' OR 1=1--@x.com`) → this is the subtle one, answered directly: the app **does** create a user (count `3 → 4`), but the injection **does not execute** — the metacharacters are stored as **inert literal data**, `email = "x'or1=1--@x.com"` (a harmless string). That is the parameterized-ORM defense (`User.create(...)` at [app/auth/views/register.py:L86] binds values, never string-concatenates SQL). The `rows=4` printed by the loop is **my own naive verification query being injected** — `WHERE email='x' OR 1=1--…'` collapses to `WHERE email='x' OR 1=1`, matching all four rows — which is precisely the contrast that shows *why* the application uses bound parameters (the safe way to confirm the stored value is the ORM read shown above, which reports the inert literal `x'or1=1--@x.com`). The throwaway row is removed inline via `User.delete(...)` (`users after self-clean: 3`); the alias tombstone it leaves behind is swept by the comprehensive cleanup in §Q4.

### Q2.14 — `next=` redirect safety (open-redirect neutralized, internal path honored)

**Direct answer.** The post-login redirect target is passed through `sanitize_next_url()` at [app/auth/views/login.py:L26], which delegates to `NextUrlSanitizer.sanitize()` at [app/utils.py:L105-L122]. An **external absolute** URL or a **protocol-relative** `//host` target is neutralized to `None` (so login falls back to `dashboard.index`), while an **internal path** beginning with a single `/` is honored. Exercised with `john` (correct password, no rate-limit deduction):

```bash
docker exec sl-canonical bash -c '
login_next() { local label="$1"; local nexturl="$2"
  local JAR=/tmp/obs/next_jar.txt; rm -f "$JAR"
  local CSRF=$(curl -sS -c "$JAR" http://localhost:7777/auth/login | grep -oE "name=\"csrf_token\"[^>]*value=\"[^\"]+\"" | grep -oE "value=\"[^\"]+\"" | sed "s/value=\"//;s/\"//")
  local loc=$(curl -sSi -b "$JAR" -c "$JAR" --data-urlencode "csrf_token=$CSRF" --data-urlencode "email=john@wick.com" --data-urlencode "password=password" "http://localhost:7777/auth/login?next=$nexturl" | grep -i "^Location:" | tr -d "\r")
  printf "%-18s next=%-34s -> %s\n" "$label" "$nexturl" "$loc"; }
login_next "external-absolute" "https://evil.example.com/phish"
login_next "protocol-relative" "//evil.example.com"
login_next "internal-path"     "%2Fdashboard%2Fsetting%2F"'
```

```
external-absolute  next=https://evil.example.com/phish     -> Location: http://localhost:7777/dashboard/
protocol-relative  next=//evil.example.com                 -> Location: http://localhost:7777/dashboard/
internal-path      next=%2Fdashboard%2Fsetting%2F          -> Location: http://localhost:7777/dashboard/setting/
```

**Cause → effect.** The **external absolute** URL has a hostname not in `ALLOWED_REDIRECT_DOMAINS`, so `sanitize()` returns `None` at [app/utils.py:L113-L116] and `after_login` falls back to `dashboard.index` — the browser is sent to `/dashboard/`, **not** to `evil.example.com`. The **protocol-relative** `//evil.example.com` is caught by the explicit `not result.path.startswith("//")` guard at [app/utils.py:L117] → `None` → `/dashboard/`. The **internal path** `/dashboard/setting/` has no hostname and begins with a single `/`, so it is returned verbatim at [app/utils.py:L117-L120] and honored. This is the open-redirect defense: attacker-controlled `next=` cannot bounce a freshly-authenticated user off-site.

### Q2.15 — HTTP method matrix (unsupported method → `405`)

**Direct answer.** Each auth route declares its allowed methods, and an unsupported verb is rejected by Werkzeug's router with **`405 METHOD NOT ALLOWED`** before the view (or its rate-limiter) runs. `/auth/login` and `/auth/register` are `methods=["GET","POST"]` ([app/auth/views/login.py:L21], [app/auth/views/register.py:L31]); `/auth/logout` is GET-only ([app/auth/views/logout.py:L8]); `/auth/activate` is `methods=["GET","POST"]` ([app/auth/views/activate.py:L13]) — so `POST /auth/activate` is *allowed* and returns `400` (missing `code`), not `405`:

```bash
docker exec sl-canonical bash -c '
probe() { local method="$1"; local path="$2"
  local status=$(curl -sSi -X "$method" "http://localhost:7777$path" | sed -n "1p" | tr -d "\r")
  printf "%-6s %-16s -> %s\n" "$method" "$path" "$status"; }
probe DELETE /auth/login
probe PUT    /auth/register
probe POST   /auth/logout
probe POST   /auth/activate
echo "--- supported verbs still work ---"
probe GET    /auth/login
probe GET    /auth/register'
```

```
DELETE /auth/login      -> HTTP/1.0 405 METHOD NOT ALLOWED
PUT    /auth/register   -> HTTP/1.0 405 METHOD NOT ALLOWED
POST   /auth/logout     -> HTTP/1.0 405 METHOD NOT ALLOWED
POST   /auth/activate   -> HTTP/1.0 400 BAD REQUEST
--- supported verbs still work ---
GET    /auth/login      -> HTTP/1.0 200 OK
GET    /auth/register   -> HTTP/1.0 200 OK
```

**Cause → effect.** `DELETE /auth/login`, `PUT /auth/register`, and `POST /auth/logout` all hit routes that do not declare those verbs → `405`. `POST /auth/activate` returns `400` (not `405`) precisely because `activate` *does* declare `POST` at [app/auth/views/activate.py:L13] but the handler needs a `code` argument. The supported `GET` verbs still return `200`. (Werkzeug 1.0.1 does not emit an `Allow` header on these `405` responses — the body is a standard error page, `Content-Length ≈ 5668` — so none is shown.) These `405`s are raised by the URL router ahead of the view function, so they never reach the login rate-limiter.

---


## §Q3 — Behind the scenes: background jobs and internal services

**Direct answer.** While a user registers, verifies, and logs in, five internal pieces work behind the scenes, and each is observable at runtime:

1. **Identity-verification email** is produced by the **transactional** mail path `send_email()` → `MailSender.send()` — the activation and welcome emails observed in §Q2.4/§Q2.5.
2. **Background jobs** are executed by the **`job_runner.py`** worker, which polls every 10 seconds and moves each job `ready → taken → done`.
3. **Internal services communicate** over a **PostgreSQL `LISTEN`/`NOTIFY`** bus on the channel `simplelogin_sync_events` — an event **publisher** and the **`event_listener.py`** consumer talking through Postgres.
4. **Email forwarding** is handled by the **`email_handler.py`** SMTP controller (aiosmtpd on port 20381), whose forward/reply paths use a **different** send entry point — `sl_sendmail()` — than the transactional path.
5. **Scheduled maintenance** is run by **`cron.py`** under yacron.

A key correction over a naive reading: **there are two distinct mail entry points**, and forwarding does **not** go through `send_email()`. Both are detailed below with live evidence.

### Q3.1 — The two mail paths: transactional `send_email()` vs forwarding/reply `sl_sendmail()` (both converge at `MailSender.send()`)

**Direct answer.** SimpleLogin sends mail through **two different entry points** that converge on the same low-level sender:

- **Transactional mail** (activation, welcome, onboarding tips, notifications) enters through `send_email()` at [app/email_utils.py:L303], which calls `MailSender.send()` at [app/mail_sender.py:L126].
- **Forwarding and reply mail** (the alias engine) enters through `sl_sendmail()` — a module-level function at [app/mail_sender.py:L270] — invoked from `email_handler.py` on the **forward** path at [email_handler.py:L903] and on the **reply** path at [email_handler.py:L1224] and [email_handler.py:L1289]. `sl_sendmail()` then calls the same `MailSender.send()` at [app/mail_sender.py:L289].

Both paths converge at `MailSender.send()` [app/mail_sender.py:L126]; under `NOT_SEND_EMAIL=true` that method logs the message and returns without contacting an MTA — the `if config.NOT_SEND_EMAIL:` guard at [app/mail_sender.py:L130], the log at [app/mail_sender.py:L131], and `return True` at [app/mail_sender.py:L137]. The **entry point differs** (`send_email` vs `sl_sendmail`), which is why forwarding is *not* observable as a `send_email()` log line.

**Live evidence — transactional path.** The transactional entry point is proven live wherever an activation, welcome, or onboarding email is produced: each logs the tell-tale `email_utils.py:303 → mail_sender.py:131` pair. The onboarding email emitted by the real `onboarding-1` job in §Q3.2 (captured this run) is a clean, self-contained example of that signature:

```
2026-07-15 06:13:42,675 - SL - DEBUG - 278 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.test996@sl.local, subject 'SimpleLogin Tip: Send emails from your alias'
2026-07-15 06:13:42,676 - SL - DEBUG - 278 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'SimpleLogin Tip: Send emails from your alias', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.test996@sl.local'
```

The activation email (§Q2.4) and welcome email (§Q2.5) travel the identical `send_email() → MailSender.send()` path and log the same pair; all three are transactional and therefore *do* appear as `send_email()` lines.

**Live evidence — forwarding/reply path (`sl_sendmail`).** This path is exercised **live** in §Q3.4 — it is *not* merely code-mapped. A real inbound message delivered over SMTP directly to `email_handler.py` (port 20381, §Q1.5) addressed to an enabled alias is accepted with `250 Message accepted for delivery` and forwarded to the mailbox, logging its send at the same `mail_sender.py:131` — but reached through `sl_sendmail()` at [app/mail_sender.py:L270], invoked from `email_handler.py` at [email_handler.py:L903], **not** through `send_email()`. The forward call chain is `handle_forward()` at [email_handler.py:L536] → `forward_email_to_mailbox()` at [email_handler.py:L679] → `sl_sendmail(...)` at [email_handler.py:L903]; the reply chain is `handle_reply()` at [email_handler.py:L966] → `sl_sendmail(...)` at [email_handler.py:L1224]. This is the exact distinction the answer must preserve: **forwarding is an `sl_sendmail` path, not a `send_email` path** — and, per §Q3.4, it is observable at runtime.

### Q3.2 — Background jobs: the `job_runner.py` worker (10-second poll, `ready → taken → done`)

**Direct answer.** Background work is executed by `job_runner.py`, whose `__main__` loop at [job_runner.py:L329] repeatedly calls `get_jobs_to_run()` at [job_runner.py:L307], logs `Take job …` at [job_runner.py:L334], marks the row `taken` (`job.taken = True` at [job_runner.py:L337], `job.state = JobState.taken.value` at [job_runner.py:L339]), runs `process_job(job)` at [job_runner.py:L342], marks it `done` (`job.state = JobState.done.value` at [job_runner.py:L344]), and then `time.sleep(10)` at [job_runner.py:L347].

**(a) The 10-second poll cadence and the `ready → taken → done` transition, observed live.** A self-contained probe creates safe no-op jobs — an unknown job name falls through `process_job()` to the `LOG.e("Unknown job name %s", job.name)` branch at [job_runner.py:L303-L304], so no side effects occur — and measures the running worker. Part **[A]** reads a freshly-created job's state the instant after `commit` and again after the worker's next poll (proving the boundary transition); parts **[B]/[C]** create three sequential jobs plus one independent rerun and measure the wall-clock gap between consecutive `Take job` events:

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env timeout 130 python3 - <<PY 2>/dev/null | grep -vE "load config|>>> URL|MAX_NB|Paddle|GNUPGHOME|Upload files|init logging|utils.py:17"
import time, arrow
from app import config
from app.models import Job, JobState
from app.db import Session
def create_job():
    j = Job.create(name="blitzy-probe-cadence", payload={}, run_at=arrow.now(), state=JobState.ready.value, flush=True)
    Session.commit(); return j.id
def read(jid):
    Session.expire_all()
    return Session.execute("SELECT state, attempts, taken, taken_at FROM job WHERE id=:i", {"i":jid}).fetchone()
def wait_taken(jid, timeout=20):
    t0=time.time()
    while time.time()-t0 < timeout:
        r=read(jid)
        if r and r[2]: return r
        time.sleep(0.2)
    return read(jid)
jid=create_job(); r0=read(jid)
print("[A] immediately after create: state=%s attempts=%s taken=%s taken_at=%s" % tuple(r0))
t0=time.time()
while time.time()-t0<20:
    r=read(jid)
    if r and r[0]==JobState.done.value: break
    time.sleep(0.2)
print("[A] after worker poll:        state=%s attempts=%s taken=%s taken_at=%s" % tuple(r))
tt=[]
for i in range(3):
    j=create_job(); r=wait_taken(j); tt.append(r[3])
    print("[B] job%d id=%d taken_at=%s" % (i+1, j, r[3]))
iv=[(tt[i+1]-tt[i]).total_seconds() for i in range(len(tt)-1)]
print("[B] consecutive Take-job intervals = %s seconds" % ["%.3f"%x for x in iv])
j1=create_job(); a=wait_taken(j1)[3]
j2=create_job(); b=wait_taken(j2)[3]
print("[C] rerun interval = %.3f seconds" % (b-a).total_seconds())
Session.execute("DELETE FROM job WHERE name='"'"'blitzy-probe-cadence'"'"'")
Session.commit()
print("self-clean: job count now =", Session.execute("SELECT count(*) FROM job").scalar())
PY'
```

```
[A] immediately after create: state=0 attempts=0 taken=False taken_at=None
[A] after worker poll:        state=2 attempts=1 taken=True taken_at=2026-07-15 06:03:21.834271
[B] job1 id=15 taken_at=2026-07-15 06:03:31.850631
[B] job2 id=16 taken_at=2026-07-15 06:03:41.867590
[B] job3 id=17 taken_at=2026-07-15 06:03:51.884999
[B] consecutive Take-job intervals = ['10.017', '10.017'] seconds
[C] rerun interval = 10.016 seconds
self-clean: job count now = 0
```

**Reading the evidence (cause → effect).** In **[A]** the job is born `state=0` (`JobState.ready`, enum at [app/models.py:L253-L257]) with `taken=False`; the running `job_runner.py` is asleep in `time.sleep(10)` at [job_runner.py:L347], so the row simply *waits* until the next poll. On that poll `get_jobs_to_run()` [job_runner.py:L307] returns it, `Take job …` is logged [job_runner.py:L334], the row is stamped `taken=True` / `taken_at` / `state=taken` [job_runner.py:L337-L339] and — after `process_job()` [job_runner.py:L342] — `state=done` [job_runner.py:L344], which is exactly the observed `state=2 attempts=1 taken=True`. In **[B]/[C]** each new job is created immediately after the previous one is taken, so it is picked up on the worker's *next* cycle — the measured `10.017 s` (twice) and the independent `10.016 s` rerun are precisely the `time.sleep(10)` period [job_runner.py:L347], stable run-to-run. This is the "the worker waits, then advances the job through its states" signal Q3 asks about; the probe rows are deleted, leaving `job` at its baseline count of `0`.

**(b) A real job routing through `process_job`, observed live.** Beyond the synthetic probe, a real `onboarding-1` job (`config.JOB_ONBOARDING_1` at [app/config.py:L301]) created for the seeded `john@wick.com` (`id=1`) is routed by `process_job()` [job_runner.py:L188] to its onboarding branch [job_runner.py:L196], which calls `onboarding_send_from_alias()` [job_runner.py:L27] — itself a **transactional** `send_email()` (tying to §Q3.1):

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate
MARK=$(wc -l < /tmp/obs/job_runner.log 2>/dev/null || echo 0)
CONFIG=/root/sl.env python3 - <<PY 2>/dev/null | grep -vE "load config|>>> URL|MAX_NB|Paddle|GNUPGHOME|Upload files|init logging|utils.py:17"
import arrow
from app import config
from app.models import Job, JobState
from app.db import Session
j = Job.create(name=config.JOB_ONBOARDING_1, payload={"user_id": 1}, run_at=arrow.now(), state=JobState.ready.value, flush=True)
Session.commit()
print("created job id=%s name=%s state=%s (ready=%s)" % (j.id, j.name, j.state, JobState.ready.value))
PY
sleep 12
echo "--- job_runner.log (this run) ---"
tail -n +$((MARK+1)) /tmp/obs/job_runner.log | grep -E "job_runner.py:334|job_runner.py:196|email_utils.py:303|mail_sender.py:131" | head -4
CONFIG=/root/sl.env python3 - <<PY 2>/dev/null | grep -vE "load config|>>> URL|MAX_NB|Paddle|GNUPGHOME|Upload files|init logging|utils.py:17"
from app.db import Session
r = Session.execute("SELECT id,name,state,attempts,taken FROM job ORDER BY id DESC LIMIT 1").fetchone()
print("terminal job row:", tuple(r))
Session.execute("DELETE FROM job WHERE name='"'"'onboarding-1'"'"'"); Session.commit()
print("self-clean: job count now =", Session.execute("SELECT count(*) FROM job").scalar())
PY'
```

```
created job id=20 name=onboarding-1 state=0 (ready=0)
--- job_runner.log (this run) ---
2026-07-15 06:13:42,645 - SL - DEBUG - 278 - "/app/job_runner.py:334" - <module>() -  - Take job <Job 20 onboarding-1 {'user_id': 1}>
2026-07-15 06:13:42,651 - SL - DEBUG - 278 - "/app/job_runner.py:196" - process_job() -  - send onboarding send-from-alias email to user <User 1 John Wick john@wick.com>
2026-07-15 06:13:42,675 - SL - DEBUG - 278 - "/app/app/email_utils.py:303" - send_email() -  - send email to simplelogin-newsletter.test996@sl.local, subject 'SimpleLogin Tip: Send emails from your alias'
2026-07-15 06:13:42,676 - SL - DEBUG - 278 - "/app/app/mail_sender.py:131" - send() -  - send email with subject 'SimpleLogin Tip: Send emails from your alias', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.test996@sl.local'
terminal job row: (20, 'onboarding-1', 2, 1, True)
self-clean: job count now = 0
```

The real job confirms the same terminal `ready → taken → done` (`state=2` done, `attempts=1`, `taken=t`) and shows `process_job` dispatching to the onboarding branch [job_runner.py:L196] → `send_email()` [app/email_utils.py:L303] → `MailSender.send()` [app/mail_sender.py:L131] — a transactional send, exactly as §Q3.1 predicts. Both the probe jobs and this real job are deleted, so `job` returns to its baseline count of `0` (verified in §Q4).


### Q3.3 — Internal services communicating: the PostgreSQL `LISTEN`/`NOTIFY` event bus

**Direct answer.** SimpleLogin's separate processes talk to each other through a PostgreSQL `LISTEN`/`NOTIFY` channel named **`simplelogin_sync_events`** (the constant `NOTIFICATION_CHANNEL = "simplelogin_sync_events"` at [app/events/event_dispatcher.py:L14]). A **publisher** writes a `SyncEvent` row and issues `NOTIFY`; the **`event_listener.py`** consumer, blocked on `LISTEN`, receives the notification, processes the event through a sink, and deletes the row. This is observable end-to-end.

**(a) Default-config behavior — the high-level publisher skips when no webhook is set.** Under the canonical local config `EVENT_WEBHOOK` is unset (`EVENT_WEBHOOK = os.environ.get("EVENT_WEBHOOK", None)` at [app/config.py:L612]). The high-level API `EventDispatcher.send_event()` (a `@staticmethod` at [app/events/event_dispatcher.py:L48-L49]) therefore short-circuits at the branch `if not config.EVENT_WEBHOOK and skip_if_webhook_missing:` [app/events/event_dispatcher.py:L61], logging and returning [app/events/event_dispatcher.py:L62-L65]. Creating a random alias through the **real dashboard HTTP route** (`POST /dashboard/` with `form-name=create-random-email`, whose handler `index()` calls `Alias.create_new_random` and — via the model at [app/models.py:L1687] — `EventDispatcher.send_event`) triggers exactly this skip on the canonical server (`server.py`, PID 270):

```bash
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate
MARK=$(wc -l < /tmp/obs/server.log 2>/dev/null || echo 0)
JAR=$(mktemp /tmp/obs/jar.XXXXXX)
csrf(){ grep -oP "name=\"csrf_token\"[^>]*value=\"\K[^\"]+" | head -1; }
LT=$(curl -s -c "$JAR" http://localhost:7777/auth/login | csrf)
curl -s -b "$JAR" -c "$JAR" -o /dev/null http://localhost:7777/auth/login -d email=john@wick.com -d password=password -d csrf_token=$LT
DT=$(curl -s -b "$JAR" -c "$JAR" http://localhost:7777/dashboard/ | csrf)
RED=$(curl -s -b "$JAR" -c "$JAR" -o /dev/null -w "%{redirect_url}" http://localhost:7777/dashboard/ -d form-name=create-random-email -d csrf_token=$DT)
NID=$(echo "$RED" | grep -oE "highlight_alias_id=[0-9]+" | cut -d= -f2); rm -f "$JAR"
echo "created random alias via real HTTP on :7777, id=$NID (EVENT_WEBHOOK unset = canonical default)"
sleep 1
tail -n +$((MARK+1)) /tmp/obs/server.log | grep -E "event_dispatcher.py:62|index.py:110" | head
CONFIG=/root/sl.env python3 - "$NID" <<PY 2>/dev/null | grep -vE "load config|>>> URL|MAX_NB|Paddle|GNUPGHOME|Upload files|init logging|utils.py:17"
import sys
from app.db import Session
nid=int(sys.argv[1])
Session.execute("DELETE FROM alias_audit_log WHERE alias_id=:i", {"i":nid})
Session.execute("DELETE FROM alias WHERE id=:i", {"i":nid})
Session.execute("UPDATE daily_metric SET nb_alias=(SELECT count(*) FROM alias) WHERE date=CURRENT_DATE")
Session.commit()
print("self-clean: alias=%d daily_metric.nb_alias=%d" % (Session.execute("SELECT count(*) FROM alias").scalar(), Session.execute("SELECT nb_alias FROM daily_metric ORDER BY id DESC LIMIT 1").scalar()))
PY'
```

```
created random alias via real HTTP on :7777, id=16 (EVENT_WEBHOOK unset = canonical default)
2026-07-15 06:06:28,301 - SL - INFO - 270 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-15 06:06:28,304 - SL - DEBUG - 270 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 16 test_word357@sl.local> for user <User 1 John Wick john@wick.com>
self-clean: alias=11 daily_metric.nb_alias=11
```

This proves the dispatcher is wired and evaluating configuration on the **real request path**: the alias is created (`index.py:110`) and, in the same web request, `send_event()` reaches its `EVENT_WEBHOOK`-missing branch and logs the skip at [app/events/event_dispatcher.py:L62]. (The probe alias and its audit-log row are deleted, and `daily_metric.nb_alias` — bumped in place at [app/models.py:L1661] — is restored, so nothing is left behind.)

**(b) With a webhook configured — the full publisher → Postgres `NOTIFY` → listener → sink round trip, live.** When `EVENT_WEBHOOK` **is** set and the user has a `PartnerUser` row (the dispatcher's other guard, `if not partner_user:` at [app/events/event_dispatcher.py:L67-L70]), `send_event()` serializes the event and calls the low-level `PostgresDispatcher.send()` [app/events/event_dispatcher.py:L79-L80], logging `Sent event to the dispatcher` at [app/events/event_dispatcher.py:L84]. `PostgresDispatcher.send()` [app/events/event_dispatcher.py:L24] writes a `SyncEvent` row (`SyncEvent.create(..., flush=True)` [app/events/event_dispatcher.py:L25]) and issues `NOTIFY simplelogin_sync_events, '<id>'` [app/events/event_dispatcher.py:L26]. The separate `event_listener.py listener` process, blocked on `LISTEN` at [events/event_source.py:L47], receives it, POSTs to the webhook via `HttpEventSink`, and deletes the row.

The self-contained block below wires all four pieces — a disposable `PartnerUser`, a trivial always-`200` webhook, an `event_listener.py listener`, and a **second** app server on port 7778 started with `EVENT_WEBHOOK` set (an ephemeral, clearly **non-canonical** helper used only to make the *producing* branch fire; the canonical server on 7777 keeps the default unset config from part (a)) — then creates one alias through real HTTP and captures both ends before tearing everything down:

```bash
docker exec sl-canonical bash -c '
cd /app && source /app/venv/bin/activate
WEBHOOK="http://localhost:7799/blitzy-event"
CONFIG=/root/sl.env python3 - <<PY 2>/dev/null | grep -vE "load config|>>> URL|MAX_NB|Paddle|GNUPGHOME|Upload files|init logging|utils.py:17"
from app.models import PartnerUser
from app.db import Session
if not PartnerUser.get_by(user_id=1, partner_id=1):
    PartnerUser.create(user_id=1, partner_id=1, external_user_id="blitzy-partner-probe", partner_email="john@wick.com", commit=True)
print("disposable PartnerUser present: user_id=1 partner_id=1 (Proton)")
PY
cat > /tmp/obs/webhook200.py <<WPY
from http.server import BaseHTTPRequestHandler, HTTPServer
class H(BaseHTTPRequestHandler):
    def _ok(self):
        self.send_response(200); self.send_header("Content-Length","2"); self.end_headers(); self.wfile.write(b"ok")
    def do_POST(self): self._ok()
    def do_GET(self): self._ok()
    def log_message(self, *a): pass
HTTPServer(("127.0.0.1",7799),H).serve_forever()
WPY
nohup python /tmp/obs/webhook200.py >/dev/null 2>&1 & WH=$!
CONFIG=/root/sl.env EVENT_WEBHOOK="$WEBHOOK" nohup python event_listener.py listener >/tmp/obs/event_listener_b.log 2>&1 & EL=$!
CONFIG=/root/sl.env EVENT_WEBHOOK="$WEBHOOK" FLASK_APP=server:create_app nohup flask run --host 127.0.0.1 --port 7778 >/tmp/obs/server7778.log 2>&1 & SV=$!
for i in $(seq 1 40); do curl -s -o /dev/null http://localhost:7778/ 2>/dev/null && break; sleep 0.5; done; sleep 1
JAR=$(mktemp /tmp/obs/jar.XXXXXX)
csrf(){ grep -oP "name=\"csrf_token\"[^>]*value=\"\K[^\"]+" | head -1; }
SVMARK=$(wc -l < /tmp/obs/server7778.log); ELMARK=$(wc -l < /tmp/obs/event_listener_b.log)
LT=$(curl -s -c "$JAR" http://localhost:7778/auth/login | csrf)
curl -s -b "$JAR" -c "$JAR" -o /dev/null http://localhost:7778/auth/login -d email=john@wick.com -d password=password -d csrf_token=$LT
DT=$(curl -s -b "$JAR" -c "$JAR" http://localhost:7778/dashboard/ | csrf)
RED=$(curl -s -b "$JAR" -c "$JAR" -o /dev/null -w "%{redirect_url}" http://localhost:7778/dashboard/ -d form-name=create-random-email -d csrf_token=$DT)
NID=$(echo "$RED" | grep -oE "highlight_alias_id=[0-9]+" | cut -d= -f2); rm -f "$JAR"
sleep 3
echo "=== [PRODUCER] real HTTP alias id=$NID (EVENT_WEBHOOK set + PartnerUser) -> EventDispatcher.send_event ==="
tail -n +$((SVMARK+1)) /tmp/obs/server7778.log | grep -E "event_dispatcher.py:84|index.py:110" | head
echo "=== [LISTENER] NOTIFY -> sink -> done ==="
tail -n +$((ELMARK+1)) /tmp/obs/event_listener_b.log | grep -E "event_source.py:55|event_sink.py:22|event_sink.py:39|runner.py:28" | head
kill $SV $EL $WH 2>/dev/null; sleep 1
CONFIG=/root/sl.env python3 - "$NID" <<PY 2>/dev/null | grep -vE "load config|>>> URL|MAX_NB|Paddle|GNUPGHOME|Upload files|init logging|utils.py:17"
import sys
from app.db import Session
nid=int(sys.argv[1])
Session.execute("DELETE FROM alias_audit_log WHERE alias_id=:i", {"i":nid})
Session.execute("DELETE FROM alias WHERE id=:i", {"i":nid})
Session.execute("DELETE FROM sync_event")
Session.execute("DELETE FROM partner_user WHERE external_user_id=:e", {"e":"blitzy-partner-probe"})
Session.execute("UPDATE daily_metric SET nb_alias=(SELECT count(*) FROM alias) WHERE date=CURRENT_DATE")
Session.commit()
print("self-clean: alias=%d sync_event=%d partner_user=%d nb_alias=%d" % (
  Session.execute("SELECT count(*) FROM alias").scalar(),
  Session.execute("SELECT count(*) FROM sync_event").scalar(),
  Session.execute("SELECT count(*) FROM partner_user").scalar(),
  Session.execute("SELECT nb_alias FROM daily_metric ORDER BY id DESC LIMIT 1").scalar()))
PY
rm -f /tmp/obs/webhook200.py /tmp/obs/server7778.log /tmp/obs/event_listener_b.log
echo "state (b): processes stopped, artifacts cleaned"
'
```

```
disposable PartnerUser present: user_id=1 partner_id=1 (Proton)
=== [PRODUCER] real HTTP alias id=17 (EVENT_WEBHOOK set + PartnerUser) -> EventDispatcher.send_event ===
2026-07-15 06:16:41,370 - SL - INFO - 1551 - "/app/app/events/event_dispatcher.py:84" - send_event() -  - Sent event to the dispatcher
2026-07-15 06:16:41,373 - SL - DEBUG - 1551 - "/app/app/dashboard/views/index.py:110" - index() -  - create new random alias <Alias 17 test_list146@sl.local> for user <User 1 John Wick john@wick.com>
=== [LISTENER] NOTIFY -> sink -> done ===
2026-07-15 06:16:41,373 - SL - DEBUG - 1550 - "/app/events/event_source.py:55" - __listen() -  - Got NOTIFY: pid=1559 channel=simplelogin_sync_events payload=3
2026-07-15 06:16:41,442 - SL - INFO - 1550 - "/app/events/event_sink.py:22" - process() -  - Sending event 3 to http://localhost:7799/blitzy-event
2026-07-15 06:16:41,444 - SL - INFO - 1550 - "/app/events/event_sink.py:39" - process() -  - Event 3 sent successfully to webhook
2026-07-15 06:16:41,445 - SL - INFO - 1550 - "/app/events/runner.py:28" - __on_event() -  - Marked 3 as done
self-clean: alias=11 sync_event=0 partner_user=0 nb_alias=11
state (b): processes stopped, artifacts cleaned
```

**Reading the evidence (cause → effect).** The producing web request created `Alias 17` (`index.py:110`) and, because `EVENT_WEBHOOK` was set and a `PartnerUser` existed, `send_event()` passed both guards and logged `Sent event to the dispatcher` at [app/events/event_dispatcher.py:L84]; internally it wrote a `SyncEvent` and issued `NOTIFY simplelogin_sync_events` carrying the row id (`payload=3`). The independent listener process received `Got NOTIFY: … channel=simplelogin_sync_events payload=3` at [events/event_source.py:L55], the `HttpEventSink` POSTed it (`Sending event 3 to http://localhost:7799/blitzy-event` [events/event_sink.py:L22]) and confirmed `Event 3 sent successfully to webhook` [events/event_sink.py:L39], and the runner deleted the row logging `Marked 3 as done` [events/runner.py:L28]. This is the literal "internal services talking to each other": two **separate processes** coordinating through the PostgreSQL `LISTEN`/`NOTIFY` channel, with the row consumed and removed so `sync_event` returns to `0`. The helper `PartnerUser`, second server, listener, and webhook are all torn down and the database restored to baseline.

### Q3.4 — The email-forwarding engine: `email_handler.py` SMTP controller (aiosmtpd)

**Direct answer.** The forwarding engine is a standalone SMTP server built on aiosmtpd. `main(port)` at [email_handler.py:L2381] constructs `Controller(MailHandler(), hostname="0.0.0.0", port=port)` at [email_handler.py:L2383] and calls `controller.start()` at [email_handler.py:L2385]; its live startup on port 20381 was captured in §Q1.5. Every inbound message is dispatched by the router `handle()` at [email_handler.py:L1945], which classifies the message and calls one of:

- `handle_forward()` at [email_handler.py:L536] → `forward_email_to_mailbox()` at [email_handler.py:L679] → `sl_sendmail()` at [email_handler.py:L903] — the **alias → mailbox forward**.
- `handle_reply()` at [email_handler.py:L966] → `sl_sendmail()` at [email_handler.py:L1224] — the **mailbox → external reply**.
- `handle_bounce()` at [email_handler.py:L1851] — bounce handling.

Because forwarding uses `sl_sendmail()` (not `send_email()`), it is the `email_handler` process — not the web app — that carries alias mail, and its send lines appear under the `sl_sendmail → MailSender.send` path described in §Q3.1.

The engine is exercised **live** here by delivering real inbound messages over SMTP directly to `email_handler.py` on port 20381 (the same controller whose startup was captured in §Q1.5) and observing the return codes plus database side effects. Three cases cover the primary path and its two most important variants — an **enabled** alias (forwarded, `250`), a **nonexistent** alias (rejected `550 SL E515`), and a **disabled** alias (accepted `250` but *not* forwarded). The recipient aliases are discovered dynamically and every probe `Contact`/`EmailLog` is deleted afterward:

```bash
docker exec sl-canonical bash -c '
cd /app && source /app/venv/bin/activate
MARK=$(wc -l < /tmp/obs/email_handler.log 2>/dev/null || echo 0)
CONFIG=/root/sl.env python3 - <<PY 2>/dev/null | grep -vE "load config|>>> URL|MAX_NB|Paddle|GNUPGHOME|Upload files|init logging|utils.py:17"
import smtplib
from email.message import EmailMessage
from app.db import Session
from app.models import Alias
PF="blitzy-smtp-probe@example.com"
enabled  = Alias.filter_by(user_id=1, enabled=True).first()
disabled = Alias.filter_by(enabled=False).first()
real_addr=enabled.email; disabled_addr=(disabled.email if disabled else None)
nonexist="no-such-alias-blitzy-zzz@sl.local"
m=EmailMessage(); m["From"]=PF; m["Subject"]="blitzy smtp probe"; m.set_content("probe body")
def build(r):
    (m.replace_header("To",r) if m.get("To") else m.__setitem__("To",r)); return m.as_bytes()
def counts():
    return (Session.execute("SELECT count(*) FROM contact").scalar(), Session.execute("SELECT count(*) FROM email_log").scalar())
def send(r):
    s=smtplib.SMTP("127.0.0.1",20381,timeout=25)
    try:
        s.ehlo("blitzy.local"); s.mail(PF); rc,rr=s.rcpt(r)
        if rc>=400: return rc,(rr.decode() if isinstance(rr,bytes) else rr)
        code,resp=s.data(build(r)); return code,(resp.decode() if isinstance(resp,bytes) else resp)
    finally:
        try:s.quit()
        except:pass
print("baseline contact/email_log = %d/%d" % counts())
for label,rcpt in [("REAL(enabled)",real_addr),("NONEXISTENT",nonexist),("DISABLED",disabled_addr)]:
    if rcpt is None: continue
    c0,e0=counts(); code,resp=send(rcpt); c1,e1=counts()
    print("%-14s %-40s -> (%s, %r)  contact %d->%d  email_log %d->%d" % (label,rcpt,code,resp,c0,c1,e0,e1))
Session.execute("DELETE FROM email_log WHERE contact_id IN (SELECT id FROM contact WHERE website_email=:f)", {"f":PF})
Session.execute("DELETE FROM contact WHERE website_email=:f", {"f":PF}); Session.commit()
print("self-clean contact/email_log = %d/%d" % counts())
PY
echo "--- email_handler.py forward trace (same run) ---"
tail -n +$((MARK+1)) /tmp/obs/email_handler.log | grep -E "New message, mail from|Forward phase|Create or get contact|Created contact|Forward <Contact|Create <EmailLog|Forward mail from|mail_sender.py:131.*send email|cannot be created on-the-fly|with return code" | sed -E "s/ - [0-9a-f]{8}-[0-9a-f-]{27} - / - /"
'
```

```
baseline contact/email_log = 1/1
REAL(enabled)  simplelogin-newsletter.test996@sl.local  -> (250, 'Message accepted for delivery')  contact 1->2  email_log 1->2
NONEXISTENT    no-such-alias-blitzy-zzz@sl.local        -> (550, 'SL E515 Email not exist')  contact 2->2  email_log 2->2
DISABLED       e0@sl.local                              -> (250, 'Message accepted for delivery')  contact 2->3  email_log 2->3
self-clean contact/email_log = 1/1
--- email_handler.py forward trace (same run) ---
2026-07-15 06:11:40,659 - SL - INFO - 298 - "/app/email_handler.py:2343" - _handle() - New message, mail from blitzy-smtp-probe@example.com, rctp tos ['simplelogin-newsletter.test996@sl.local'] 
2026-07-15 06:11:40,664 - SL - DEBUG - 298 - "/app/email_handler.py:2202" - handle() - Forward phase blitzy-smtp-probe@example.com(blitzy-smtp-probe@example.com) -> simplelogin-newsletter.test996@sl.local
2026-07-15 06:11:40,671 - SL - DEBUG - 298 - "/app/email_handler.py:580" - handle_forward() - Create or get contact for from_header:blitzy-smtp-probe@example.com
2026-07-15 06:11:40,689 - SL - DEBUG - 298 - "/app/app/contact_utils.py:110" - create_contact() - Created contact <Contact 14 blitzy-smtp-probe@example.com 1> for alias <Alias 1 simplelogin-newsletter.test996@sl.local> with email blitzy-smtp-probe@example.com invalid_email=False
2026-07-15 06:11:40,695 - SL - DEBUG - 298 - "/app/email_handler.py:688" - forward_email_to_mailbox() - Forward <Contact 14 blitzy-smtp-probe@example.com 1> -> <Alias 1 simplelogin-newsletter.test996@sl.local> -> <Mailbox 1 john@wick.com>
2026-07-15 06:11:40,697 - SL - DEBUG - 298 - "/app/email_handler.py:740" - forward_email_to_mailbox() - Create <EmailLog 14> for <Contact 14 blitzy-smtp-probe@example.com 1>, <User 1 John Wick john@wick.com>, <Mailbox 1 john@wick.com>
2026-07-15 06:11:40,703 - SL - DEBUG - 298 - "/app/email_handler.py:893" - forward_email_to_mailbox() - Forward mail from blitzy-smtp-probe@example.com to john@wick.com, mail_options:[], rcpt_options:[] 
2026-07-15 06:11:40,703 - SL - DEBUG - 298 - "/app/app/mail_sender.py:131" - send() - send email with subject 'blitzy smtp probe', from '"blitzy-smtp-probe at example.com" <blitzy-smtp-probe_at_example_com_efwnzqmx@sl.local>' to 'simplelogin-newsletter.test996@sl.local'
2026-07-15 06:11:40,703 - SL - INFO - 298 - "/app/email_handler.py:2367" - _handle() - Finish mail_from blitzy-smtp-probe@example.com, rcpt_tos ['simplelogin-newsletter.test996@sl.local'], takes 0.04443049430847168 seconds with return code '250 Message accepted for delivery'<<===
2026-07-15 06:11:40,707 - SL - INFO - 298 - "/app/email_handler.py:2343" - _handle() - New message, mail from blitzy-smtp-probe@example.com, rctp tos ['no-such-alias-blitzy-zzz@sl.local'] 
2026-07-15 06:11:40,711 - SL - DEBUG - 298 - "/app/email_handler.py:2202" - handle() - Forward phase blitzy-smtp-probe@example.com(blitzy-smtp-probe@example.com) -> no-such-alias-blitzy-zzz@sl.local
2026-07-15 06:11:40,720 - SL - DEBUG - 298 - "/app/email_handler.py:551" - handle_forward() - alias no-such-alias-blitzy-zzz@sl.local cannot be created on-the-fly, return 550
2026-07-15 06:11:40,721 - SL - INFO - 298 - "/app/email_handler.py:2367" - _handle() - Finish mail_from blitzy-smtp-probe@example.com, rcpt_tos ['no-such-alias-blitzy-zzz@sl.local'], takes 0.014554500579833984 seconds with return code '550 SL E515 Email not exist'<<===
2026-07-15 06:11:40,724 - SL - INFO - 298 - "/app/email_handler.py:2343" - _handle() - New message, mail from blitzy-smtp-probe@example.com, rctp tos ['e0@sl.local'] 
2026-07-15 06:11:40,729 - SL - DEBUG - 298 - "/app/email_handler.py:2202" - handle() - Forward phase blitzy-smtp-probe@example.com(blitzy-smtp-probe@example.com) -> e0@sl.local
2026-07-15 06:11:40,736 - SL - DEBUG - 298 - "/app/email_handler.py:580" - handle_forward() - Create or get contact for from_header:blitzy-smtp-probe@example.com
2026-07-15 06:11:40,752 - SL - DEBUG - 298 - "/app/app/contact_utils.py:110" - create_contact() - Created contact <Contact 15 blitzy-smtp-probe@example.com 4> for alias <Alias 4 e0@sl.local> with email blitzy-smtp-probe@example.com invalid_email=False
2026-07-15 06:11:40,755 - SL - INFO - 298 - "/app/email_handler.py:2367" - _handle() - Finish mail_from blitzy-smtp-probe@example.com, rcpt_tos ['e0@sl.local'], takes 0.03142881393432617 seconds with return code '250 Message accepted for delivery'<<===
```

**Reading the evidence (cause → effect).**

- **Enabled alias → `250` + forwarded.** `handle()` classifies it as the *Forward phase* [email_handler.py:L2202], `handle_forward()` [email_handler.py:L536] creates/gets the sender `Contact` (`contact_utils.create_contact` [app/contact_utils.py:L110]), and `forward_email_to_mailbox()` [email_handler.py:L679] logs `Forward <Contact> -> <Alias> -> <Mailbox 1 john@wick.com>` [email_handler.py:L688], creates the `EmailLog` [email_handler.py:L740], and delivers via `sl_sendmail()` [email_handler.py:L903] → `MailSender.send()` [app/mail_sender.py:L131]. Net effect: `contact +1`, `email_log +1`, and a `250 Message accepted for delivery` (`status.E200` at [app/handler/status.py:L2]).
- **Nonexistent alias → `550`.** `handle_forward()` finds no alias, tries auto-create, fails, and returns at [email_handler.py:L551] (`alias … cannot be created on-the-fly, return 550`); the controller replies `550 SL E515 Email not exist` (`status.E515` at [app/handler/status.py:L51]). No rows are created (`Δ0/Δ0`).
- **Disabled alias → `250` but not forwarded.** Because the sender `Contact` is created *before* the enabled check (`handle_forward()` "Create or get contact" at [email_handler.py:L580] precedes the `if not alias.enabled …` guard at [email_handler.py:L596]), a blocked `EmailLog` (`blocked=True`) is written and the controller still returns `250` (`res_status = status.E200` at [email_handler.py:L608-L612]) — so a sender is never told the alias is disabled. Net effect: `contact +1`, `email_log +1`, yet **no delivery** (note the trace stops after `Create or get contact` — there is no `Forward mail from …` line for `e0@sl.local`).

Under `NOT_SEND_EMAIL=true` the final `MailSender.send()` logs rather than contacting an MTA (§Q3.1); the **internal** forwarding decision, contact creation, `EmailLog` write, and the `250`/`550` status codes are all real and observed above. Every probe `Contact`/`EmailLog` is then deleted, restoring `contact`/`email_log` to their seeded baseline of `1/1`.

### Q3.5 — Scheduled maintenance: `cron.py` under yacron

**Direct answer.** Periodic tasks are executed by `cron.py`, scheduled by **yacron** via `crontab.yml`. Every entry in that file runs the command `python /code/cron.py -j <job>` (e.g. the `stats` job at schedule `0 0 * * *`; the argparse dispatch is at [cron.py:L1262] and the `stats` branch at [cron.py:L1274-L1276]). **Reported honestly per the Run-First discipline: in the canonical image the *scheduled* command fails**, because the path `/code` does not exist there — the repository is checked out at `/app`, not `/code`. yacron parses the config and *spawns* the job correctly, but the child process `python /code/cron.py …` cannot open the file, exits `2`, and yacron reports a permanent failure. Crucially, the **canonical code entry point itself is healthy**: invoked at its real path (`python cron.py -j stats` from `/app`) it exits `0`. The single combined run below shows all three facts — the missing path **[A]**, the real yacron spawning the shipped command and failing **[B]**, and the healthy direct entry point **[C]**:

```bash
docker exec sl-canonical bash -c '
source /app/venv/bin/activate
echo "=== [A] /code missing; /app/cron.py present ==="
ls /code 2>&1 | head -1
ls -la /app/cron.py | awk "{print \$5\" bytes\", \$NF}"
echo "=== [B] REAL yacron runs the crontab.yml command (schedule accelerated to every-minute; command python /code/cron.py UNCHANGED) ==="
sed "0,/schedule: \"0 0 \* \* \*\"/s//schedule: \"* * * * *\"/" /app/crontab.yml > /tmp/blitzy_yacron.yml
grep -m1 "command:" /tmp/blitzy_yacron.yml
cd /app
CONFIG=/root/sl.env timeout 75 yacron -c /tmp/blitzy_yacron.yml 2>&1 | grep -E "Starting job|stderr|exit code|permanent failure" | head -6
rm -f /tmp/blitzy_yacron.yml
echo "=== [C] contrast: canonical direct entry point python cron.py -j stats exits 0 ==="
CONFIG=/root/sl.env python cron.py -j stats > /tmp/blitzy_cron.out 2>&1; echo "direct cron.py exit=$?"
grep -E "cron.py:1263|cron.py:1275|cron.py:540" /tmp/blitzy_cron.out; rm -f /tmp/blitzy_cron.out
'
```

```
=== [A] /code missing; /app/cron.py present ===
ls: cannot access '/code': No such file or directory
46424 bytes /app/cron.py
=== [B] REAL yacron runs the crontab.yml command (schedule accelerated to every-minute; command python /code/cron.py UNCHANGED) ===
    command: python /code/cron.py -j stats
INFO:yacron:Starting job SimpleLogin growth stats
INFO:yacron:Starting job SimpleLogin send unsent emails
[SimpleLogin send unsent emails stderr] python: can't open file '/code/cron.py': [Errno 2] No such file or directory
[SimpleLogin growth stats stderr] python: can't open file '/code/cron.py': [Errno 2] No such file or directory
INFO:yacron:Job SimpleLogin growth stats exit code 2; has stdout: false, has stderr: true; fail_reason: 'failsWhen=nonzeroReturn and retcode=2'
INFO:yacron:Cron job SimpleLogin growth stats: reporting permanent failure
=== [C] contrast: canonical direct entry point python cron.py -j stats exits 0 ===
direct cron.py exit=0
2026-07-15 06:06:27,493 - SL - DEBUG - 1401 - "/app/cron.py:1263" - <module>() -  - Start running cronjob
2026-07-15 06:06:27,494 - SL - DEBUG - 1401 - "/app/cron.py:1275" - <module>() -  - Compute growth and daily monitoring stats
2026-07-15 06:06:27,494 - SL - WARNING - 1401 - "/app/cron.py:540" - stats() -  - ADMIN_EMAIL not set, nothing to do
```

**Reading the evidence (cause → effect).** In **[A]** `ls /code` returns `No such file or directory` while `/app/cron.py` (46,424 bytes) exists — the scheduler's hard-coded path is simply wrong for this checkout. In **[B]** the shipped `crontab.yml` command is confirmed as `python /code/cron.py -j stats`; the real yacron *parses and spawns* it (`Starting job …`), so the scheduler configuration is valid, but the child prints `python: can't open file '/code/cron.py': [Errno 2] No such file or directory`, exits `2`, and yacron records `fail_reason: 'failsWhen=nonzeroReturn and retcode=2'` → `reporting permanent failure`. Because **every** entry in `crontab.yml` uses the same `/code` path, all of them fail identically (two jobs happened to fire in the window; both failed the same way). In **[C]**, running the *same* job through the canonical code path (`python cron.py` from `/app`) exits `0` and logs `Start running cronjob` [cron.py:L1263] → `Compute growth and daily monitoring stats` [cron.py:L1275] → the safe no-op `ADMIN_EMAIL not set, nothing to do` [cron.py:L540]. **Conclusion:** the cron *code* is healthy; the failure is purely the `/code` vs `/app` path mismatch in the container's scheduler config — a container-packaging detail, not a defect in `cron.py`. The accelerated-schedule copy is written to `/tmp` and deleted; no repository file is touched and no rows are created.

### Q3.6 — How the behind-the-scenes pieces support the new-user flow

Tying §Q3 back to the observed §Q2 flow, as cause → effect:

- **Identity verification** is supported by the **transactional mail path** (§Q3.1): registration's `send_activation_email()` and activation's `send_welcome_email()` both flow `send_email() → MailSender.send()` and were observed logging under `NOT_SEND_EMAIL` in §Q2.4 and §Q2.5. In a production deployment these are the emails that actually verify a new address.
- **Background jobs** (§Q3.2) run onboarding and alias-related work asynchronously via `job_runner.py`, decoupled from the request that created them — observed as a real `ready → taken → done` transition.
- **Internal-service communication** (§Q3.3) lets the web app, `job_runner`, and `email_handler` coordinate through the `simplelogin_sync_events` `LISTEN`/`NOTIFY` bus without direct coupling — observed as a live publish→consume→delete round trip.
- **Email forwarding** (§Q3.4) is the product's core function, carried by the `email_handler` SMTP controller on port 20381 through the `sl_sendmail` path — observed **live**: a real inbound message to an enabled alias was accepted `250` and forwarded (creating a new `Contact`/`EmailLog`), a nonexistent alias was rejected `550 SL E515`, and a disabled alias was accepted `250` but not delivered.
- **Scheduled maintenance** (§Q3.5) is run by `cron.py` under yacron. The `cron.py` code entry point is healthy (exits `0` at its real `/app` path), but — reported honestly per Run-First — the *scheduled* `python /code/cron.py` command fails in the canonical image because `/code` does not exist, so yacron reports a permanent failure until that path is corrected.

---


## §Q4 — Constraints honored: read-only source and temporary-artifact cleanup

**Direct answer.** Both constraints are satisfied and proven: (1) **no source file was modified** — the entire investigation added exactly **one** file to the destination repository (this document), with all configuration kept out-of-repo and the runtime confined to a separate container checkout; and (2) **every temporary artifact was removed** — the probe user, its alias/mailbox, the expired activation code, the test job, and the event were all deleted, restoring the database to its exact seeded baseline, and the running services were stopped by targeted PID.

### Q4.1 — Read-only source (proof)

The runtime lived in a **separate checkout inside the Docker container** (`/app`), and configuration was a copy of `example.env` placed **outside any repository** at `/root/sl.env` (loaded via SimpleLogin's absolute-path `CONFIG` passthrough at [app/config.py:L14-L20]). Consequently the destination repository's working tree shows only the one intended change — the answer document:

```bash
cd /tmp/blitzy/app/blitzy-3f604de9-c2f2-40ab-b951-3bc5004e14ae_37f383
git status --short
```

```
M  blitzy/documentation/app_2cd6ee777f8c.md
D  blitzy/screenshots/simplelogin_dashboard_after_login.png
D  blitzy/screenshots/simplelogin_login_page.png
```

The two `D` (deleted) entries are the removal of two screenshot PNGs that a prior revision had committed; deleting them brings the net change versus the project baseline to a single added file (see §Q4.5). No application source, template, migration, config, or dependency file appears — none was touched.

> **Historical snapshot.** The `git status --short` output above was captured *during* remediation — while the rewritten answer document was staged (`M`) and the two obsolete screenshot PNGs were being removed (`D`). It is retained here to explain how the tree reached its final shape. On the **current committed tree** the same command yields **no output at all** (a clean working tree), because those changes are now committed; the net change versus the project baseline `2cd6ee77` is exactly the single added answer document. The committed-state proof is in §Q4.5.

### Q4.2 — Temporary-artifact cleanup and exact baseline restoration (proof)

**What was created during the investigation (all temporary).** Exercising the flows above leaves a footprint that must be removed to honor the AAP's "temporary users and aliases … must be removed" constraint: one probe user (`blitzyprobe@gmail.com` — dynamically, the only `users.id > 2`) with its auto-created default alias and mailbox; one deliberately-expired `ActivationCode` (§Q2.6); the append-only `alias_audit_log` / `user_audit_log` rows and `deleted_alias` tombstones that every create/verify/login/delete cycle produces; an **in-place** `daily_metric` value drift; and — on the filesystem — the per-process `GNUPGHOME` temp directories and the `/tmp/obs` scratch. **Nothing is hard-coded:** the cleanup discovers its targets by `id`/pattern so it works on any fresh run (the auto-generated alias word — e.g. `word444` this run — differs every run and must never be hard-coded).

The seeded baseline (captured immediately after `flask dummy-data`, before any probe activity) is the restoration target. Its non-empty tracked tables and the **value-bearing** `daily_metric` row are:

```
users=2  alias=11  activation_code=0  deleted_alias=0  alias_audit_log=11  user_audit_log=0  mailbox=4  contact=1(SEED)  email_log=1(SEED)  job=0  sync_event=0
daily_metric: nb_alias=11  nb_new_web_non_proton_user=0
```

Restoration is proved **table-by-table** (not just for the handful named above) with a full 77-table snapshot helper: `/tmp/snap_before.txt` is captured with the observation drift still present (at the start of cleanup) and `/tmp/snap_after.txt` after cleanup, then the two are `diff`ed (Step 5). A reader reproducing from a clean bring-up instead snapshots the pristine baseline first, and the post-cleanup `diff` is then empty.

**Step 1 — discover the footprint dynamically.** The seeded users are `id` 1 (`john@wick.com`) and 2 (`winston@continental.com`); any `users.id > 2` is a probe:

```bash
docker exec sl-canonical su postgres -c "psql -d simplelogin -tA -c \"SELECT id, email FROM users WHERE id > 2 ORDER BY id;\""
```

```
3|blitzyprobe@gmail.com
```

**Step 2 — delete the probe user(s) through the canonical ORM.** `User.delete()` at [app/models.py:L671] is not a single-row delete: it emits a `UserDeleted` event ([app/models.py:L677]) and routes every owned alias through `delete_alias()` at [app/alias_utils.py:L336], which **moves the alias to a global trash** — a `DeletedAlias` tombstone at [app/alias_utils.py:L364-L365] — and logs an `alias_audit_log` entry ([app/alias_utils.py:L346]). It is driven from the discovered `id > 2` set (no hard-coded id):

```bash
docker exec -i sl-canonical bash -c 'cat > /tmp/orm_delete.py' <<'PY'
from app.models import User
from app.db import Session
ids = [u.id for u in Session.query(User).filter(User.id > 2).order_by(User.id).all()]
print("PROBE_USER_IDS=%s" % ids)
for uid in ids:
    print("DELETING id=%d email=%s" % (uid, User.get(uid).email))
    User.delete(uid)
Session.commit()
print("remaining_users=%d" % Session.query(User).count())
PY
docker exec sl-canonical bash -c 'cd /app && CONFIG=/root/sl.env /app/venv/bin/python /tmp/orm_delete.py 2>&1' \
  | grep -E "PROBE_USER_IDS|DELETING|has deleted alias|webhook is not configured|remaining_users"
```

```
PROBE_USER_IDS=[3]
DELETING id=3 email=blitzyprobe@gmail.com
2026-07-15 07:37:11,721 - SL - INFO - 3608 - "/app/app/events/event_dispatcher.py:62" - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
2026-07-15 07:37:11,748 - SL - INFO - 3608 - "/app/app/alias_utils.py:346" - delete_alias() -  - User <User 3 blitzyprobe@gmail.com blitzyprobe@gmail.com> has deleted alias <Alias 20 simplelogin-newsletter.word444@sl.local>
remaining_users=2
```

The two `send_event() … webhook is not configured` lines are the `UserDeleted` (and the alias-deletion) events being emitted and then skipped locally (the same default-configuration skip explained in §Q3.3), and the `delete_alias()` line is the tombstone being written. The random alias word (`word444` here) is exactly why the cleanup must never hard-code it.

**Step 3 — observe the residue the cascade leaves** (the crux of the finding: a count-only, users-table-only check would wrongly declare victory here):

```bash
docker exec -i sl-canonical /app/venv/bin/python - <<'PY' 2>/dev/null
import psycopg2
c = psycopg2.connect("postgresql://myuser:mypassword@localhost:5432/simplelogin").cursor()
for t in ["users","alias","mailbox","activation_code","alias_audit_log","user_audit_log","deleted_alias"]:
    c.execute('SELECT count(*) FROM "%s"' % t); print("%s=%d" % (t, c.fetchone()[0]))
c.execute("SELECT nb_alias, nb_new_web_non_proton_user FROM daily_metric WHERE id=1"); r = c.fetchone()
print("daily_metric nb_alias=%d nb_new_web_non_proton_user=%d" % (r[0], r[1]))
PY
```

```
users=2
alias=11
mailbox=4
activation_code=0
alias_audit_log=19
user_audit_log=14
deleted_alias=4
daily_metric nb_alias=15 nb_new_web_non_proton_user=4
```

`users` / `alias` / `mailbox` / `activation_code` are back to baseline, **but `alias_audit_log` is 19 (baseline 11), `user_audit_log` is 14 (baseline 0), `deleted_alias` is 4 (baseline 0), and `daily_metric` still reads `15 / 4` (baseline `11 / 0`)** — the audit trail, the alias tombstones, and the *in-place* metric drift the cascade does not touch. `alias_audit_log` even rose by one (18 → 19) because the deletion itself is audited.

**Step 4 — remove the residue explicitly** (dynamic by ownership / pattern — never by the run-specific alias word):

```bash
docker exec -i sl-canonical bash -c 'cat > /tmp/residual.sql' <<'SQL'
BEGIN;
DELETE FROM alias_audit_log WHERE user_id > 2;                                   -- probe + orphaned-throwaway rows
DELETE FROM user_audit_log;                                                      -- baseline is 0 (all rows are observation trail)
DELETE FROM deleted_alias WHERE email LIKE 'simplelogin-newsletter.%@sl.local';  -- probe default-alias tombstones (any run word)
UPDATE daily_metric SET nb_alias = 11, nb_new_web_non_proton_user = 0 WHERE id = 1;  -- restore the drifted VALUES
COMMIT;
SQL
docker exec sl-canonical su postgres -c "psql -d simplelogin -f /tmp/residual.sql"
```

```
BEGIN
DELETE 8
DELETE 14
DELETE 4
UPDATE 1
COMMIT
```

**Step 5 — prove exact restoration across ALL 77 tables + the metric VALUES.** A full snapshot is taken after cleanup and `diff`ed against the pre-cleanup (drifted) snapshot; the only rows that differ are the seven the probe touched, and every other table is identical:

```bash
docker exec -i sl-canonical /app/venv/bin/python - <<'PY' > /tmp/snap_after.txt 2>/dev/null
import psycopg2
c = psycopg2.connect("postgresql://myuser:mypassword@localhost:5432/simplelogin").cursor()
c.execute("SELECT table_name FROM information_schema.tables WHERE table_schema='public' ORDER BY table_name")
for t in [r[0] for r in c.fetchall()]:
    c.execute('SELECT count(*) FROM "%s"' % t); print("%s=%d" % (t, c.fetchone()[0]))
PY
diff <(sort /tmp/snap_before.txt) <(sort /tmp/snap_after.txt) | grep -E '^[<>]'
docker exec sl-canonical su postgres -c "psql -d simplelogin -tA -c \"SELECT 'daily_metric nb_alias='||nb_alias||' nb_new_web_non_proton_user='||nb_new_web_non_proton_user FROM daily_metric WHERE id=1;\""
```

```
< activation_code=1
> activation_code=0
< alias=12
< alias_audit_log=18
> alias=11
> alias_audit_log=11
< deleted_alias=3
> deleted_alias=0
< mailbox=5
> mailbox=4
< user_audit_log=14
< users=3
> user_audit_log=0
> users=2
daily_metric nb_alias=11 nb_new_web_non_proton_user=0
```

The `< …` lines are the drifted (pre-cleanup) values and the `> …` lines the restored ones; the restored values are exactly the seeded baseline (`activation_code=0`, `alias=11`, `alias_audit_log=11`, `deleted_alias=0`, `mailbox=4`, `user_audit_log=0`, `users=2`), and the remaining 70 tables produce **no diff line at all** — so the database is back to the seeded baseline exactly, including the value-bearing `daily_metric` row (`nb_alias 15 → 11`, `nb_new_web_non_proton_user 4 → 0`) that a count-only comparison (`daily_metric` stays at one row throughout) would have missed.

**Redis.** The local rate-limiter is in-memory (Flask-Limiter's default store) and sessions are signed cookies, so Redis never accumulated observation state; `DBSIZE` reads `0` throughout, and a `FLUSHALL` confirms nothing transient remains:

```bash
docker exec sl-canonical bash -c "redis-cli DBSIZE; redis-cli FLUSHALL; redis-cli DBSIZE"
```

```
(integer) 0
OK
(integer) 0
```

The remaining ephemeral artifacts — the running services, the per-process `GNUPGHOME` temp directories, and the `/tmp/obs` scratch — are process-scoped and are torn down in §Q4.3.

### Q4.3 — Stopping the services by targeted PID (no broad signals)

The service processes are discovered **dynamically** — the `[s]` character-class trick makes the pattern skip the `grep` line itself, and the match catches the launcher, the `server.py` parent, and its Werkzeug reloader child — then torn down by their **exact** PIDs (never `pkill -f`, never a broad signal). A tiny portable TCP probe confirms the ports are open before and closed after:

```bash
# one-time: a portable TCP probe (a real connect(), not a substring match in /proc)
docker exec -i sl-canonical bash -c 'cat > /tmp/portcheck.sh' <<'SH'
#!/bin/bash
for p in 7777 20381; do
  /app/venv/bin/python -c "import socket; s=socket.socket(); r=s.connect_ex(('127.0.0.1',$p)); s.close(); print('port $p', 'OPEN' if r==0 else 'CLOSED')"
done
SH

docker exec sl-canonical bash /tmp/portcheck.sh          # ports before stop
docker exec sl-canonical bash -c '
PIDS=$(ps -eo pid,cmd | grep -E "[s]erver\.py|[j]ob_runner\.py|[e]mail_handler\.py" | awk "{print \$1}" | sort -n | tr "\n" " ")
echo "discovered service PIDs: $PIDS"
kill $PIDS 2>/dev/null; sleep 3
S=$(ps -eo pid,cmd | grep -E "[s]erver\.py|[j]ob_runner\.py|[e]mail_handler\.py" | awk "{print \$1}" | tr "\n" " ")
[ -n "$S" ] && { kill -9 $S 2>/dev/null; sleep 2; }
echo "remaining SL service procs: $(ps -eo pid,cmd | grep -E "[s]erver\.py|[j]ob_runner\.py|[e]mail_handler\.py" | grep -v grep | wc -l)"
'
docker exec sl-canonical bash /tmp/portcheck.sh          # ports after stop
```

```
port 7777 OPEN
port 20381 OPEN
discovered service PIDs: 3805 3806 3807 3837
remaining SL service procs: 0
port 7777 CLOSED
port 20381 CLOSED
```

The PIDs are **run-specific** (every start assigns fresh ones — `3805 3806 3807 3837` here, different on the next run), which is exactly why they are discovered rather than hard-coded; the teardown kills only the set it just discovered, and `connect_ex` then returns non-zero (`CLOSED`) for both ports because nothing is listening.

**Filesystem: the per-process `GNUPGHOME` temp directories.** Every SL process — and every one-off app-context invocation used above — logs `Use a temp directory for GNUPGHOME /tmp/<random>` at startup and leaves that directory behind, so they accumulate over a session (60 were present at the end of the main investigation). With the services stopped they are safe to remove, together with the `/tmp/obs` scratch:

```bash
docker exec sl-canonical bash -c '
echo "GNUPGHOME dirs before: $(ls -d /tmp/*/ 2>/dev/null | grep -cE "/tmp/[a-z]{20}/$")"
ls -d /tmp/*/ 2>/dev/null | grep -E "/tmp/[a-z]{20}/$" | xargs -r rm -rf
rm -rf /tmp/obs
echo "GNUPGHOME dirs after:  $(ls -d /tmp/*/ 2>/dev/null | grep -cE "/tmp/[a-z]{20}/$")"
[ -e /tmp/obs ] && echo "/tmp/obs still present" || echo "/tmp/obs removed"
'
```

```
GNUPGHOME dirs before: 4
GNUPGHOME dirs after:  0
/tmp/obs removed
```

The `before` count is whatever the current session accumulated (`4` for the freshly-restarted stack shown here, `60` after the full investigation); `after` is always `0`. The out-of-repo config `/root/sl.env` may also be removed at this point — because it lived outside the repository from the start (§Q4.1), its presence or absence never affects the repository's cleanliness.


### Q4.4 — Reproduce the whole observation from scratch (executable)

The following is a complete, copy-pasteable sequence that provisions a fresh instance, seeds it, starts the four entry points with their PIDs captured to a file, and tears everything down idempotently. It contains no placeholders — every value is concrete, and the role/database creation is guarded so re-runs are safe.

```bash
# --- 0. Start the canonical container (maps the web port) ---
docker run -dit --name sl-canonical -p 7777:7777 --shm-size=1g \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0

# --- 1. PostgreSQL + role/db (idempotent; no sudo in image -> use su postgres) ---
docker exec sl-canonical bash -c 'su postgres -c "pg_ctlcluster 15 main start"'
docker exec sl-canonical su postgres -c "psql -tAc \"SELECT 1 FROM pg_roles WHERE rolname='myuser'\" | grep -q 1 \
  || psql -c \"CREATE ROLE myuser LOGIN PASSWORD 'mypassword' SUPERUSER;\""
docker exec sl-canonical su postgres -c "psql -tAc \"SELECT 1 FROM pg_database WHERE datname='simplelogin'\" | grep -q 1 \
  || psql -c \"CREATE DATABASE simplelogin OWNER myuser;\""

# --- 2. Redis ---
docker exec sl-canonical bash -c 'redis-server --daemonize yes --port 6379'

# --- 3. Out-of-repo config (keeps the repo read-only) ---
docker exec sl-canonical bash -c 'cp /app/example.env /root/sl.env'

# --- 4. Schema + seed data ---
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env alembic upgrade head'
docker exec sl-canonical bash -c 'cd /app && source /app/venv/bin/activate && CONFIG=/root/sl.env flask dummy-data'

# --- 5. Start the four entry points; capture each PID to /tmp/obs/pids ---
docker exec sl-canonical bash -c 'mkdir -p /tmp/obs; cd /app && source /app/venv/bin/activate
  CONFIG=/root/sl.env nohup python server.py        > /tmp/obs/server.log        2>&1 & echo "server $!"        >> /tmp/obs/pids
  CONFIG=/root/sl.env nohup python job_runner.py    > /tmp/obs/job_runner.log    2>&1 & echo "job_runner $!"    >> /tmp/obs/pids
  CONFIG=/root/sl.env nohup python email_handler.py > /tmp/obs/email_handler.log 2>&1 & echo "email_handler $!" >> /tmp/obs/pids
  CONFIG=/root/sl.env nohup python event_listener.py listener --dry-run > /tmp/obs/event_listener.log 2>&1 & echo "event_listener $!" >> /tmp/obs/pids'

# --- 5a. Wait (bounded) for the web app to accept connections, then confirm the core readiness signals ---
docker exec sl-canonical bash -c 'timeout 40 bash -c "until curl -s -o /dev/null http://localhost:7777/live; do sleep 1; done"'
docker exec sl-canonical bash -c 'curl -s -o /dev/null -w "GET /    -> %{http_code}\n" http://localhost:7777/'         # 302 -> /auth/login
docker exec sl-canonical bash -c 'curl -s -o /dev/null -w "GET /live -> %{http_code}\n" http://localhost:7777/live'    # 200
docker exec sl-canonical bash -c 'echo -n "init-logging lines: "; grep -c ">>> init logging <<<" /tmp/obs/server.log' # 2 (reloader)
docker exec sl-canonical bash -c 'grep -m1 "Listen for port 20381" /tmp/obs/email_handler.log'                        # SMTP controller up

# --- 5b. Reproduce the documented flows (each is a self-contained, copy-pasteable block above) ---
#   * register -> verify -> login -> dashboard ............... Q2.1 - Q2.10
#   * logout / CSRF / input boundaries / MFA / next / methods . Q2.11 - Q2.15
#   * job cadence, live SMTP forwarding, event dispatch ...... Q3.2 - Q3.5
#   * cleanup + exact baseline restoration ................... Q4.2
# Running those blocks in order reproduces every signal in this document.

# --- 6. Idempotent teardown: kill ONLY the captured PIDs, then remove scratch + config ---
docker exec sl-canonical bash -c 'while read name pid; do kill "$pid" 2>/dev/null && echo "stopped $name ($pid)"; done < /tmp/obs/pids'
docker exec sl-canonical bash -c 'rm -f /root/sl.env; rm -rf /tmp/obs'

# --- 7. Remove the whole ephemeral instance ---
docker rm -f sl-canonical
```

Because the child PIDs of `server.py`'s reloader are re-parented under the captured launcher PID, killing the captured PID (or, as shown in §Q4.3, the specific reloader child) stops the worker; step 7 then discards the entire container so no state can survive.

### Q4.5 — Git proof: exactly one file changes versus the project baseline

On the **current committed tree**, the net change versus the project baseline commit `2cd6ee77` is **exactly one added file** — the answer document — and nothing else:

```bash
cd /tmp/blitzy/app/blitzy-3f604de9-c2f2-40ab-b951-3bc5004e14ae_37f383
git diff --name-status 2cd6ee77..HEAD
```

```
A	blitzy/documentation/app_2cd6ee777f8c.md
```

No application source, template, migration, config, or dependency file appears — none was touched — which is the concrete proof of the MainRule read-only scope. While the rewritten document is being edited but not yet committed, `git status --short` additionally shows a single modified entry and nothing else:

```bash
git status --short
```

```
 M blitzy/documentation/app_2cd6ee777f8c.md
```

Once that edit is committed, the working tree is clean and the single-file `git diff` result above still holds against the new `HEAD` (see §Q4.1).

> **Historical snapshot (pre-remediation).** An earlier revision of the branch also carried two screenshot PNGs, so `git diff --name-status 2cd6ee77..HEAD` then listed **three** added files (`…/app_2cd6ee777f8c.md` plus `…/simplelogin_dashboard_after_login.png` and `…/simplelogin_login_page.png`), and they were removed with `git rm …` before the document was re-staged. Those removals are **already committed**, so on the current tree that `git rm` must **not** be re-run — it fails honestly:
> ```
> $ git rm blitzy/screenshots/simplelogin_login_page.png
> fatal: pathspec 'blitzy/screenshots/simplelogin_login_page.png' did not match any files
> ```
> because the files no longer exist. The single-file `git diff` above is the correct, reproducible proof for the current tree; the three-file view is retained only to explain how the tree reached its final shape.

---

## Coverage pass

Rule 4 asks for a closing coverage pass. The tables below map every item named in the four questions — including each item introduced by "e.g." / "such as" — to the section that answers it and the runtime signal that grounds it. Each row is a pointer to evidence already shown earlier in this document; it adds no new claim.

**Q1 — Readiness signals (in logs or the UI):**

| Named item | Answered in | Observed signal |
|------------|-------------|-----------------|
| Flask dev-server startup banner | Q1.1 | `* Serving Flask app "server" (lazy loading)` / `* Environment: production` / `* Debug mode: on` (the classic `* Running on …` line is suppressed because the Werkzeug logger is disabled at [app/log.py:L70-L71]; Werkzeug/1.0.1 confirmed via the `Server:` header — see §Q1.1) |
| `>>> init logging <<<` (SL logger init) | Q1.1 | emitted once per app import via `app/log.py` (twice total under the debug reloader — see §Q1.1) |
| Unauthenticated `GET /` redirect | Q1.2 | `302` → `Location: /auth/login`, then `200` on the login page |
| `/git` health endpoint | Q1.3 | `200` returning build `SHA1` (`dev`) |
| `/live` health endpoint | Q1.3 | `200` returning `live` |
| `/exception` health endpoint | Q1.3 | `500` by design (deliberate `raise` for Sentry) |
| `job_runner.py` worker readiness | Q1.4 | load-config + `init logging`, then silent 10-second poll |
| `email_handler.py` SMTP controller | Q1.5 | aiosmtpd controller bound; `Listen for port 20381` |

**Q2 — New-user flow (register → verify → log in → dashboard):**

| Named item | Answered in | Observed signal |
|------------|-------------|-----------------|
| Register form served | Q2.1 | `GET /auth/register` → `200` (CSRF token present in form) |
| Register success | Q2.2 | `POST /auth/register` → `200` "Activation Email Sent" waiting page |
| Register error / boundary paths | Q2.3 | duplicate-email and invalid-email flash messages |
| Activation email minted + logged | Q2.4 | 30-char single-use `ActivationCode`; logged under `NOT_SEND_EMAIL=true` |
| Verify flips `activated f → t` | Q2.5 | `GET /auth/activate?code=…` → `302` → `/dashboard/`; code consumed (`1` → `0`) |
| Verify error / boundary paths | Q2.6 | bad / reused / expired code → `400` |
| Login success | Q2.7 | `after_login()` → `302` → `dashboard.index` |
| Login error / boundary paths | Q2.8 | bad creds, unactivated, disabled, and rate-limit `429` |
| Login MFA + scheduled-deletion branches | Q2.9 | FIDO → `302` `/auth/fido`; OTP → `302` `/auth/mfa`; `delete_on` set → `200` scheduled-deletion warning, no login (exercised live) |
| Dashboard landing | Q2.10 | `GET /dashboard/` → `200` rendering `dashboard/index.html` |
| Log out (session boundary) | Q2.11 | `GET /auth/logout` → `302` → `/auth/login`; session cookies expired; post-logout `/dashboard/` → `302` |
| CSRF protection (register + login) | Q2.12 | missing / invalid CSRF → `200` re-render, no row created, no auth granted |
| Registration input boundaries | Q2.13 | empty / malformed / service-domain / XSS-shaped rejected; SQL-shaped stored inert (parameterized ORM) |
| `next=` redirect safety | Q2.14 | external + protocol-relative neutralized → `/dashboard/`; internal path honored |
| HTTP method matrix | Q2.15 | unsupported method → `405`; `GET` forms → `200` |

**Q3 — Behind the scenes (background jobs + internal services for email forwarding & identity verification):**

| Named item | Answered in | Observed signal |
|------------|-------------|-----------------|
| Mail path `send_email()` (transactional) | Q3.1 | converges at `MailSender.send()`; logged under `NOT_SEND_EMAIL` |
| Mail path `sl_sendmail()` (forward / reply) | Q3.1 | same convergence at `MailSender.send()` |
| Background jobs (`job_runner.py`) | Q3.2 | ≥3 real jobs `ready → taken → done`; measured ~10-second poll cadence across consecutive jobs + an independent repeat run |
| Internal services communicating | Q3.3 | real app `EventDispatcher.send_event()` (`Sent event to the dispatcher`) + Postgres `LISTEN`/`NOTIFY` on `simplelogin_sync_events`, end-to-end (listener consumes then deletes the `SyncEvent`) |
| Email-forwarding engine | Q3.4 | `email_handler.py` aiosmtpd controller (live in Q1.5); **live forwarding** to port 20381: real alias → `250` + `Contact`/`EmailLog` +1, nonexistent alias → `550 SL E515`, disabled alias → `250` but not forwarded |
| Scheduled maintenance (cron / yacron) | Q3.5 | direct `cron.py -j stats` entry point runs (exit `0`); the **bundled yacron schedule is broken in this image** — it invokes `/code/cron.py` while the repo is `/app`, so the child exits `2` (`can't open file '/code/cron.py'`), reported honestly |
| Support for the new-user flow | Q3.6 | identity-verification mail path + forwarding readiness |

**Q4 — Constraints (temporary artifacts removed; no source changes):**

| Named item | Answered in | Observed signal |
|------------|-------------|-----------------|
| Read-only source | Q4.1 | no tracked source file modified |
| Temporary-artifact cleanup + baseline restoration | Q4.2 | probe deleted via ORM `User.delete()` cascade + explicit residue cleanup; all **77 tables** and the value-bearing `daily_metric` row restored (77-table `diff` reduces to the seven probe-touched tables, then empty) |
| Stop services by targeted PID | Q4.3 | dynamically-discovered PIDs signalled (no broad `pkill`); ports `7777`/`20381` confirmed `CLOSED` via a socket probe |
| Reproduce from scratch | Q4.4 | executable bring-up + bounded-wait readiness + idempotent teardown block (no placeholders) |
| Git proof | Q4.5 | current staged tree vs baseline `2cd6ee77` = exactly one added file |
