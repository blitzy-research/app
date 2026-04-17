# SimpleLogin — Local Deployment Observability & Verification Guide

**Branch:** `app_2cd6ee777f8c`
**Commit:** `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`
**Container image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`
**Runtime stack:** Python 3.10 · PostgreSQL 13+ · Redis 6+ · Flask 1.1.2 · SQLAlchemy 1.3.24 · Gunicorn 20.x

> **Read-only exercise.** Per the user's explicit rule, **no source code was modified** during this investigation. All evidence in this document is drawn from inspecting the source files on branch `app_2cd6ee777f8c` and from a live runtime walkthrough in which a single test user was registered, activated, logged in, and then **removed** from the PostgreSQL database in correct foreign-key order.
>
> **Scope.** This guide answers three interrelated questions about a fresh local deployment of SimpleLogin:
>
> 1. **Startup readiness** — what in the logs and web UI confirms the app is operational?
> 2. **New-user journey** — what happens, step by step, from registration to dashboard?
> 3. **Background-service observability** — what indicators show that jobs, events, and inter-process communication are working?
>
> Every behavioural claim in this document cites the specific source file and line where the code lives. A consolidated index appears in the **Appendix — Code References** at the end.

---

## Table of Contents

- [Local Environment Prerequisites](#local-environment-prerequisites)
- [1. Startup Readiness — What confirms SimpleLogin is fully operational?](#1-startup-readiness--what-confirms-simplelogin-is-fully-operational)
  - [1.1 The Bootstrap Chain (Code Walk-through)](#11-the-bootstrap-chain-code-walk-through)
  - [1.2 Observed Startup Log Output](#12-observed-startup-log-output)
  - [1.3 HTTP Readiness Probes](#13-http-readiness-probes)
  - [1.4 Database Readiness](#14-database-readiness)
  - [1.5 Summary Checklist — "SimpleLogin is Ready"](#15-summary-checklist--simplelogin-is-ready)
- [2. New User Journey — From Registration to Dashboard](#2-new-user-journey--from-registration-to-dashboard)
  - [2.1 Registration — `POST /auth/register`](#21-registration--post-authregister)
  - [2.2 Activation — `GET /auth/activate?code=...`](#22-activation--get-authactivatecode)
  - [2.3 Login — `POST /auth/login`](#23-login--post-authlogin)
  - [2.4 Dashboard — `GET /dashboard/`](#24-dashboard--get-dashboard)
  - [2.5 End-to-End Verification Timeline](#25-end-to-end-verification-timeline)
- [3. Background Service Observability](#3-background-service-observability)
  - [3.1 The Five-Process Architecture](#31-the-five-process-architecture)
  - [3.2 Job Runner — `job_runner.py`](#32-job-runner--job_runnerpy)
  - [3.3 Cron Scheduler — `cron.py` + `crontab.yml`](#33-cron-scheduler--cronpy--crontabyml)
  - [3.4 Event Dispatcher — `app/events/event_dispatcher.py`](#34-event-dispatcher--appeventsevent_dispatcherpy)
  - [3.5 Auth Events — `app/events/auth_event.py`](#35-auth-events--appeventsauth_eventpy)
  - [3.6 Email Subsystem — `app/mail_sender.py` and `NOT_SEND_EMAIL` Mode](#36-email-subsystem--appmail_senderpy-and-not_send_email-mode)
  - [3.7 Inbound Email — `email_handler.py`](#37-inbound-email--email_handlerpy)
  - [3.8 Event Listener — `event_listener.py`](#38-event-listener--event_listenerpy)
  - [3.9 Monitoring — `monitoring.py`](#39-monitoring--monitoringpy)
  - [3.10 Observability Matrix — "What's Happening Now?"](#310-observability-matrix--whats-happening-now)
  - [3.11 Quick Smoke-Test Commands](#311-quick-smoke-test-commands)
- [4. Post-Verification Cleanup](#4-post-verification-cleanup)
- [Appendix — Code References](#appendix--code-references)

---

## Local Environment Prerequisites

SimpleLogin is a multi-process Flask application whose behaviour is driven almost entirely by environment variables loaded through `python-dotenv` in `app/config.py` (see `app/config.py:9` — `from dotenv import load_dotenv` — and `app/config.py:64-71` where the file is actually loaded). The reference set of variables is `example.env` at the repository root.

### Required Services

| Component | Version | Role |
|---|---|---|
| **PostgreSQL** | 13+ (tested with v16) | Primary relational store; holds `users`, `mailbox`, `alias`, `job`, `activation_code`, `daily_metric`, `public_domain`, `sync_event`, etc. |
| **Redis** | 6+ (tested with v7) | Session store, Flask-Limiter backend, distributed locks. Only wired in when `MEM_STORE_URI` is set (see `server.py:163-165`). |
| **Node.js** | 10.17.0 in the Dockerfile (newer works too) | Builds the frontend assets under `static/` at image build time (Dockerfile Stage 1 `FROM node:10.17.0-alpine AS npm`). Not required to run the backend. |
| **Postfix / MTA** | optional | Required only when delivering real email. With `NOT_SEND_EMAIL=true` the Postfix dependency disappears. |

The database `simplelogin` must exist, owned by user `myuser` (or whatever role you use in `DB_URI`). Alembic will manage the schema.

### Required Environment Variables

All of these are drawn from `example.env`. The investigation used exactly this set:

```bash
URL=http://localhost:7777
EMAIL_DOMAIN=sl.local
SUPPORT_EMAIL=support@sl.local
SUPPORT_NAME=Son from SimpleLogin
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
FLASK_SECRET=secret
NOT_SEND_EMAIL=true            # critical: enables log-only email simulation
COLOR_LOG=true                 # enables coloredlogs (app/log.py:62); see note below about TTY detection
EMAIL_SERVERS_WITH_PRIORITY=[(10, "email.hostname.")]
```

Two of these deserve special attention because they govern observable behaviour:

- **`NOT_SEND_EMAIL=true`** (`example.env:19`) is the flag that tells `MailSender.send()` in `app/mail_sender.py:130` to log the email metadata and return `True` **without** opening an SMTP socket. It is the single most important variable for local verification — without it, every outbound send attempts a real SMTP handshake.
- **`COLOR_LOG=true`** activates `coloredlogs.install()` at `app/log.py:62`, but the `coloredlogs` library itself auto-detects whether stdout is a TTY. ANSI escape codes are therefore **only visible when the process writes to an interactive terminal**. If you redirect stdout to a file (e.g. `python server.py > /tmp/sl_startup.log 2>&1`), the log contents will be plain ASCII without color — this is expected library behaviour, not a misconfiguration.
- **`DISABLE_ONBOARDING=true`** is explicitly enabled in `example.env:150` (`# For self-hosted instance`). When this is truthy, `User.create()` in `app/models.py:646-647` takes an **early return** and **no onboarding jobs are scheduled**. If you want to see the three onboarding rows in the `job` table, you must **unset** or comment out `DISABLE_ONBOARDING` in your `.env`.

### Bootstrap Commands

After the environment variables are in place, run these in order **exactly once** per fresh database:

```bash
# 1. Apply all schema migrations (256 revisions under migrations/versions/).
alembic upgrade head

# 2. Seed the public_domain table and load PGP keys (idempotent).
python3 init_app.py

# 3. Launch the app. Two flavours:

# 3a. Production-style (matches Dockerfile CMD):
gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 15

# 3b. Development-style (debug toolbar, reloader, flask dev server):
python server.py
```

The Dockerfile's default CMD is `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` (see the last line of `Dockerfile`). The container exposes port **7777** (`EXPOSE 7777`).

`wsgi.py` is a three-line file:

```python
from server import create_app

app = create_app()
```

All real bootstrap happens in `server.py:create_app()` (line 139).

---

## 1. Startup Readiness — What confirms SimpleLogin is fully operational?

This section answers the question: **after the app is launched, which log lines and HTTP probes prove the system is fully initialised and ready to accept user traffic?**

### 1.1 The Bootstrap Chain (Code Walk-through)

The process of going from `gunicorn wsgi:app` (or `python server.py`) to a request-serving Flask application is a tightly-sequenced chain. The order matters — each step has an observable side-effect that shows up in logs.

#### Step 1 — `app/config.py` import (happens before any Flask code runs)

The very first side-effect on the console comes from **module-level** `print()` calls in `app/config.py`. When Python imports `server.py`, it imports `app.config`, which at import time:

1. Executes the `.env` loading branch at `app/config.py:64-71`: **if** `CONFIG` environment variable is set, `load_dotenv(get_abs_path(config_file))` is called for that explicit path (`app/config.py:69`); **else** `load_dotenv()` is called with no arguments, which auto-discovers the default `.env` (`app/config.py:71`). The two calls are mutually exclusive — only one runs per process.
2. Reads `URL = os.environ["URL"]` (`app/config.py:79`)
3. Emits `print(">>> URL:", URL)` (`app/config.py:80`)

So the **first line** you see on stdout is always the `URL:` echo. If this line is missing, `dotenv` didn't find your `.env` file and the process will eventually crash with `KeyError: "URL"`.

#### Step 2 — `app/log.py` import (custom logger init)

`server.py` imports `from app.log import LOG`. At the top of `app/log.py`:

- Line 5: `import coloredlogs`
- Line 48: `_get_logger(name)` is defined
- Line 62: `coloredlogs.install(level="DEBUG", logger=logger, ...)` — colourises output when `COLOR_LOG` is set
- Line 67: `print(">>> init logging <<<")` fires
- Line 79: `LOG = _get_logger("SL")` creates the project-wide logger

So the **second line** on stdout is `>>> init logging <<<`. These two print statements are your "Python side of the stack is alive" signal.

#### Step 3 — `server.py:create_app()` executes the full initialisation chain

The factory function at `server.py:139` (`def create_app() -> Flask:`) performs, in order:

| Step | Line | Action | Side-effect |
|---|---|---|---|
| 1 | 142 | `app.wsgi_app = ProxyFix(app.wsgi_app, ...)` | Honours `X-Forwarded-*` headers behind a reverse proxy |
| 2 | 159 | Session cookie: `SESSION_COOKIE_NAME`, `SameSite="Lax"`, `SESSION_COOKIE_SECURE=True` if URL starts with `https` | Cookie config locked in |
| 3 | 163-165 | `if MEM_STORE_URI: initialize_redis_services(app, MEM_STORE_URI)` | Redis session store + limiter storage attached |
| 4 | 167 | `limiter.init_app(app)` | Flask-Limiter bound to app |
| 5 | 169 | `setup_error_page(app)` | 400/401/404/405/500 handlers registered |
| 6 | 171 | `init_extensions(app)` → `login_manager.init_app(app)` | Flask-Login wired up |
| 7 | 172 | `register_blueprints(app)` | 10 distinct blueprints registered (see below) |
| 8 | 173 | `set_index_page(app)` | `/` route + `before_request` / `after_request` hooks |
| 9 | 176 | `setup_favicon_route(app)` | `/favicon.ico` handler |
| 10 | 177 | `setup_openid_metadata(app)` | `/.well-known/openid-configuration` |
| 11 | 179 | `init_admin(app)` | Flask-Admin mounted at `/admin` |
| 12 | 183 | `register_custom_commands(app)` | CLI commands (`dummy-data`, `send-newsletter`, `fill-up-email-log-alias`, etc.) |
| 13 | 200 | `CORS(app, resources={r"/api/*": {"origins": "*"}})` | CORS on API paths |
| 14 | 213-215 | `/health` route registered, returns `"success", 200` | Liveness endpoint |

Sentry SDK is initialised only when `SENTRY_DSN` is set (`server.py:111-114`). New Relic is imported lazily; actual APM instrumentation only kicks in if `NEW_RELIC_CONFIG_FILE` points to a valid INI file.

#### Step 4 — Blueprint registration (10 distinct, 11 calls)

`server.py:233-246` — `register_blueprints(app)` calls `app.register_blueprint(...)` eleven times across ten distinct blueprints:

```python
# (paraphrased for clarity; actual code at server.py:233-246)
app.register_blueprint(auth_bp)
app.register_blueprint(monitor_bp)
app.register_blueprint(dashboard_bp)
app.register_blueprint(developer_bp)
app.register_blueprint(phone_bp)
app.register_blueprint(oauth_bp,     url_prefix="/oauth")
app.register_blueprint(oauth_bp,     url_prefix="/oauth2")   # same bp, second prefix
app.register_blueprint(onboarding_bp)
app.register_blueprint(discover_bp)
app.register_blueprint(internal_bp)
app.register_blueprint(api_bp)
```

The `oauth_bp` **double-registration** at both `/oauth` and `/oauth2` is easy to miss when counting — it is there to support both the legacy and current OAuth spec URL conventions. If you grep `server.py` for `register_blueprint`, you should see **11 matches**. Note that **only `oauth_bp` carries an explicit `url_prefix=` at the registration call site**: every other blueprint — including `auth_bp` and `dashboard_bp` — is registered with no `url_prefix` argument because each blueprint defines its own prefix inside its module's `base.py` (e.g. `auth_bp = Blueprint("auth", __name__, url_prefix="/auth")` in `app/auth/base.py`, and the analogous declaration in `app/dashboard/base.py`). The effective URL prefixes (`/auth`, `/dashboard`, etc.) are therefore built into the blueprint objects themselves.

The blueprint objects come from `app/auth/base.py` (`auth_bp`), `app/dashboard/base.py` (`dashboard_bp`), and their respective `app/<module>/base.py` files for the rest.

#### Step 5 — Request-lifecycle hooks

Two hooks installed by `set_index_page(app)` at `server.py:249` govern every HTTP request:

- **`@app.before_request`** (`server.py:257-258`) — captures `g.start_time = time.time()` and handles the `slref` referral cookie. **Skipped** for `/static`, `/admin/static`, and `/_debug_toolbar` paths to keep static-asset loading fast.
- **`@app.after_request`** (`server.py:272-273`) — emits the per-request DEBUG log in the format
  ```text
  %s %s %s %s %s, takes %s
  ```
  where the substitutions are `remote_addr | method | path | args | status_code | elapsed_time`, and additionally records `newrelic.agent.record_custom_event("HttpResponseStatus", {"code": res.status_code})`. This is **your per-request audit trail** — every served request produces exactly one of these lines, **except** for the six excluded path prefixes filtered at `server.py:275-283`: `/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, and `/health`. Requests whose path starts with any of those prefixes produce **no** `@after_request` log line (so grepping logs for `/health` hits will always return zero matches — this is by design).

The index page itself is mounted at `server.py:249-254`: `/` redirects authenticated users to the dashboard and unauthenticated users to `auth.login`.

### 1.2 Observed Startup Log Output

When the app is launched in dev mode (`python server.py` → `local_main()` at `server.py:572`), the live investigation captured exactly this output (captured verbatim from a clean run with stdout redirected to a file):

```text
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/<random>
Upload files to local dir
>>> init logging <<<
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

Each line is traceable to a specific source location:

| Line | Source | Meaning |
|---|---|---|
| `>>> URL: http://localhost:7777` | `app/config.py:80` — `print(">>> URL:", URL)` | `.env` loaded; `URL` env var resolved |
| `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` | `app/config.py` — printed when the free-plan alias quota env var is unset | Informational; the default cap of 5 aliases for free-plan accounts is applied |
| `Paddle param not set` | `app/config.py` — printed when the Paddle (billing provider) credentials are missing | Informational; local dev has no billing integration |
| `WARNING: Use a temp directory for GNUPGHOME /tmp/<random>` | `app/pgp_utils.py` via `app/config.py` — `GNUPGHOME` env var unset, so a fresh temp directory is provisioned via `tempfile.mkdtemp()` | Warning only; PGP features still function against the temp keyring |
| `Upload files to local dir` | `app/config.py` — when `LOCAL_FILE_UPLOAD=true` (or equivalent local-dev switch) and `BUCKET` is unset | Uploads are persisted to `LOCAL_FILE_UPLOAD_DIRECTORY` instead of S3 |
| `>>> init logging <<<` | `app/log.py:67` — `print(">>> init logging <<<")` | `app.log` imported; `LOG = _get_logger("SL")` built (line 79); `coloredlogs` installed (line 62) when `COLOR_LOG` is set |
| `* Serving Flask app "server" (lazy loading)` | emitted by Flask's Werkzeug dev server at `app.run(...)` | Dev-mode only — fires from `local_main()` (see `server.py:572`) |
| `* Environment: production` | Werkzeug dev-server banner (Flask 1.1.x default when `FLASK_ENV` is unset) | Dev server IS running; the "production" label here refers to Werkzeug's own default environment flag, **not** to the `app.debug` state |
| `WARNING: This is a development server. Do not use it in a production deployment. / Use a production WSGI server instead.` | Werkzeug dev-server banner | Expected when launched via `python server.py`; Gunicorn runs do not print this |
| `* Debug mode: on` | from `app.run(debug=True, port=7777)` at `server.py:588` | `app.debug = True` is set at `server.py:581` |

Note that the four "bootstrap" lines between `>>> URL:` and `>>> init logging <<<` (`MAX_NB_EMAIL_FREE_PLAN`, `Paddle param`, `GNUPGHOME`, `Upload files`) originate from module-level `print()` / warning calls in `app/config.py` that fire as a side-effect of the first `import app.config`. They therefore appear under **both** dev-mode and Gunicorn startup paths.

**Under Gunicorn** the Werkzeug "Serving Flask app" / "Debug mode" lines are replaced with Gunicorn's own bootstrap output, which looks roughly like:

```text
[INFO] Starting gunicorn 20.1.0
[INFO] Listening at: http://0.0.0.0:7777 (PID)
[INFO] Using worker: sync
[INFO] Booting worker with pid: <N>
```

These Gunicorn lines come from Gunicorn itself (not from SimpleLogin source code), but the **`>>> URL:`** and **`>>> init logging <<<`** lines still fire under both run modes because they originate in `app/config.py` and `app/log.py` — files that any entry point (Gunicorn or dev) imports.

### 1.3 HTTP Readiness Probes

The most reliable way to confirm a fresh deployment is healthy is to issue a handful of HTTP requests:

| Check | Endpoint | Expected Response | Source reference |
|---|---|---|---|
| **Liveness** | `GET /health` | `200 "success"` | `server.py:213-215` |
| **Root redirect (unauthenticated)** | `GET /` | `302 → /auth/login` | `server.py:249-254` — redirects via `url_for("auth.login")` |
| **Login page** | `GET /auth/login` | `200` HTML | `app/auth/views/login.py:21` — `@auth_bp.route("/login", ...)` |
| **Register page** | `GET /auth/register` | `200` HTML | `app/auth/views/register.py:31` — `@auth_bp.route("/register", ...)` |
| **OpenID discovery** | `GET /.well-known/openid-configuration` | `200` JSON | `server.py:299` — `setup_openid_metadata(app)` |

A shell probe:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/health
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/register
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/.well-known/openid-configuration
```

Four of the five probes above produce a matching line in the web process stdout through the `@after_request` hook. **`/health` does not** — it is one of the six path prefixes explicitly excluded at `server.py:275-283` (`/static`, `/admin/static`, `/_debug_toolbar`, `/git`, `/favicon.ico`, `/health`). The format is `%s %s %s %s %s, takes %s` where `args` is Werkzeug's `ImmutableMultiDict`, which renders as `ImmutableMultiDict([])` when the request had no query parameters. Observed verbatim output (captured live):

```text
127.0.0.1 GET / ImmutableMultiDict([]) 302, takes 0.0006682872772216797
127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10135126113891602
127.0.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes 0.03267025947570801
```

Note: the fields are **space-separated** (no dash delimiters), and the `args` column is the literal `repr()` of `request.args` (not a Python dict). For a request carrying query parameters (e.g. `GET /auth/activate?code=xyz`), the same slot prints as `ImmutableMultiDict([('code', 'xyz')])`.

### 1.4 Database Readiness

Startup **does not** automatically apply schema migrations or seed domain data. Those are discrete operational steps.

**1. Apply migrations.** The repository ships **256 Alembic revisions** under `migrations/versions/` (confirmed via `ls migrations/versions/ | wc -l` → `256`). Applying them:

```bash
alembic upgrade head
```

After this the full schema exists. Key tables for the new-user flow:

```text
users            -- user accounts
mailbox          -- user mailboxes (verified destination addresses)
alias            -- alias email addresses
activation_code  -- short-lived codes for email verification
job              -- background-job queue
daily_metric     -- daily aggregate counters (e.g. registrations/day)
public_domain    -- SimpleLogin-managed alias domains (seeded by init_app.py)
sync_event       -- outbound event queue (consumed by event_listener.py)
```

**2. Seed SL domains and load PGP keys.** `init_app.py` (73 lines) is the seed step:

- `load_pgp_public_keys()` at `init_app.py:13` — iterates over users' configured PGP public keys and loads them into the GnuPG keyring used by the forwarding pipeline (`Mailbox.pgp_public_key` and `Contact.pgp_public_key`).
- `add_sl_domains()` at `init_app.py:39-56` — iterates over `config.ALIAS_DOMAINS` and `config.PREMIUM_ALIAS_DOMAINS` and creates a row in the `public_domain` table for each (via `SLDomain.create(...)`). The function is **idempotent**:
  - If the domain already exists → `LOG.d("%s is already a SL domain", domain)` (DEBUG).
  - Otherwise → `LOG.i("Add %s to SL domain", domain)` (INFO) before creating.
- `add_proton_partner()` is **defined** in `init_app.py:59` but is **not** invoked from `init_app.py`'s `__main__` block. It is called at web-app bootstrap time from `server.py` (inside the `dummy-data` CLI handler) and from the pytest fixture at `tests/conftest.py:39`. Running `python3 init_app.py` therefore does **not** create the Proton partner row; that row appears only when the main Flask app boots or when the test suite runs.

The `if __name__ == "__main__":` block at the end of `init_app.py` (lines 69-73) wraps the two seed calls in `create_light_app().app_context()` so they can run standalone without starting the web server. The call order is: **first** `load_pgp_public_keys()`, **then** `add_sl_domains()` — `add_proton_partner()` is **not** in this block.

**Note on terminology:** the SQLAlchemy model class is named `SLDomain` (defined in `app/models.py`), but its `__tablename__` is **`public_domain`** (see `app/models.py:3119`). Use `public_domain` in any `psql` queries:

```sql
SELECT domain FROM public_domain ORDER BY id;
```

After `init_app.py` you should see `sl.local` (the value of `EMAIL_DOMAIN`) in that table.

### 1.5 Summary Checklist — "SimpleLogin is Ready"

Tick the following off in order. If any item fails, stop and fix the corresponding stage before moving on.

- ✅ stdout contains `>>> URL: http://localhost:7777` (from `app/config.py:80`)
- ✅ stdout contains `>>> init logging <<<` (from `app/log.py:67`)
- ✅ stdout contains either Flask's `* Serving Flask app "server"` line (dev mode) **or** Gunicorn's `[INFO] Listening at:` line (prod mode)
- ✅ `curl http://localhost:7777/health` returns `200 "success"`
- ✅ `curl -I http://localhost:7777/` returns `302 Found` with `Location: /auth/login`
- ✅ `curl http://localhost:7777/auth/login` returns `200` (the login form renders)
- ✅ No `sqlalchemy.exc.OperationalError` / `psycopg2.OperationalError` stack traces in stdout (DB connection is live)
- ✅ `public_domain` table contains `EMAIL_DOMAIN` (e.g. `sl.local`)
- ✅ If Sentry configured: no `Failed to initialize Sentry` warnings
- ✅ If Redis configured (`MEM_STORE_URI` set): no `redis.exceptions.ConnectionError` in stdout

When all of these are green, the application is fully operational and ready to accept the new-user flow described in §2.

---

## 2. New User Journey — From Registration to Dashboard

This section answers: **what happens, step by step, when a brand-new user registers, verifies their email, and logs in for the first time? What visible behaviours confirm the system processed each phase correctly and forwarded the user to the dashboard?**

### Overview Flow

The end-to-end journey traverses four HTTP requests and touches six database tables. The following diagram (from AAP §0.4.1) shows the full chain:

```mermaid
flowchart TD
    A["POST /auth/register"] --> B["register.py: validate form"]
    B --> C["email_utils: email_can_be_used_as_mailbox()"]
    C --> D["models.py: User.create()"]
    D --> E["models.py: Mailbox.create()<br/>(verified=True, default)"]
    E --> F["models.py: Alias.create_new()<br/>(simplelogin-newsletter prefix)"]
    F --> G["config.py: Schedule 3 onboarding Jobs<br/>(JOB_ONBOARDING_1,2,4)"]
    G --> H["register.py: ActivationCode.create()"]
    H --> I["email_utils: send_activation_email()"]
    I --> J["mail_sender.py: MailSender.send()<br/>(logs only when NOT_SEND_EMAIL)"]
    J --> K["DailyMetric: increment nb_new_web_non_proton_user"]
    K --> L["auth_event.py: RegisterEvent.success → New Relic"]
    L --> M["Render: register_waiting_activation.html"]

    M --> N["GET /auth/activate?code=..."]
    N --> O["activate.py: validate code, set user.activated=True"]
    O --> P["Flask-Login: login_user()"]
    P --> Q["email_utils: send_welcome_email()"]
    Q --> R["302 Redirect → /dashboard/"]

    R --> S["GET /dashboard/"]
    S --> T["dashboard/index.py: compute stats"]
    T --> U["index.py: show_intro=True (first visit)"]
    U --> V["Render: dashboard/index.html with alias list"]
```

The subsections below walk through each of these four HTTP requests in the same order a browser would exercise them.

### 2.1 Registration — `POST /auth/register`

**Endpoint:** `app/auth/views/register.py:31` — `@auth_bp.route("/register", methods=["GET", "POST"])`

**Step-by-step flow inside the handler:**

1. **Guard: already authenticated.** Line 33 — if `current_user.is_authenticated` is true, 302-redirect to the dashboard. This prevents a logged-in user from registering a second account.
2. **Guard: registration disabled.** Line 38 — if `config.DISABLE_REGISTRATION` is truthy (`app/config.py:138` — `DISABLE_REGISTRATION = "DISABLE_REGISTRATION" in os.environ`), flash a warning and redirect to login.
3. **Form validation.** `RegisterForm` (class defined at line 23) declares `email` as a `StringField` with the `validators.DataRequired()` validator only (line 24) and `password` as a `StringField` with `DataRequired()` plus `validators.Length(min=8, max=100)` (line 25). Note that `password` is declared as `StringField` (not `PasswordField`); the HTML `type="password"` comes from the template layer (`templates/auth/register.html`), not from the form class. Dedicated email-shape validation is deferred to the handler body via `is_valid_email()`, `canonicalize_email()`, and `email_can_be_used_as_mailbox()`, which run after WTForms has confirmed the field is non-empty.
4. **hCaptcha verification.** Lines 47-70 — only runs when `config.HCAPTCHA_SECRET` is set. In a local deployment (where `HCAPTCHA_SECRET` is unset), this block is skipped entirely. A failed captcha fires `RegisterEvent(RegisterEvent.ActionType.catpcha_failed).send()` and re-renders the form.
5. **Email canonicalisation.** Line 73 — `email = canonicalize_email(email)`, then `email_can_be_used_as_mailbox(email)` (line 76) and `personal_email_already_used(email)` (line 83) are re-checked against both the canonical and original form.
6. **Log the creation.** Line 85:
   ```python
   LOG.d("create user %s", email)
   ```
   This produces the `create user testuser@example.com` DEBUG line visible in stdout.
7. **Call `User.create()`.** Lines 86-91 — invokes `User.create(email=email, name=name, password=password, from_partner=False, ...)` defined at `app/models.py:602`. See the sub-section below for what `User.create()` actually does.
8. **Send the activation email.** Line 95 — wrapped in `try/except` (the `try` opens at line 95); on failure a `RegisterEvent(ActionType.invalid_email)` is emitted (line 101) and the form is re-rendered with an error flash.
9. **Record the success.** Lines 96-97 — `RegisterEvent(RegisterEvent.ActionType.success).send()` (records a New Relic custom event with action="success", source="web") and `DailyMetric.get_or_create_today_metric().nb_new_web_non_proton_user += 1` (increments today's registration counter in the `daily_metric` table).
10. **Render the waiting page.** Line 104 — `render_template("auth/register_waiting_activation.html")` — the template is rendered with **no** keyword arguments; it does not need the submitted email because the page simply asks the user to check their inbox.

#### What `User.create()` does internally

The orchestration logic lives at `app/models.py:602` and is the single biggest side-effect of registration. Step by step:

| Line | Action | DB effect |
|---|---|---|
| 602 | `User.create(...)` insert into `users` | `users` +1 row with `activated=False` |
| 611 | `Mailbox.create(user_id=user.id, email=user.email, verified=True)` — the call takes no `commit=True` argument; the commit is performed later by the outer caller after all related rows are inserted. | `mailbox` +1 row, marked verified |
| 613 | `user.default_mailbox_id = mb.id` | Updates the `users` row |
| 616 | `if "alternative_id" not in kwargs: user.alternative_id = str(uuid.uuid4())` — only auto-generated when the caller did not supply an explicit `alternative_id` kwarg. | Updates the `users` row (conditional) |
| 634-641 | `Alias.create_new(user, prefix="simplelogin-newsletter", mailbox_id=mb.id, ...)` — the user's first alias | `alias` +1 row; the alias email is `simplelogin-newsletter.<random>@<FIRST_ALIAS_DOMAIN or EMAIL_DOMAIN>`. The alias record has a `note` populated with "This is your first alias..." |
| 643 | `user.newsletter_alias_id = alias.id` | Updates the `users` row |
| **646** | **`if config.DISABLE_ONBOARDING:`** (truthy by default per `example.env:150`) | **Early return** — no onboarding jobs are scheduled. `LOG.d("Disable onboarding emails")` is emitted on line 647. |
| 651-655 | (if `DISABLE_ONBOARDING` unset) `Job.create(name=config.JOB_ONBOARDING_1, payload={"user_id": user.id}, run_at=arrow.now().shift(days=1))` — no `commit=True` argument is passed; `Job.create` does not accept one at this call site. | `job` +1 row, `state=ready`, `run_at = now + 1 day` |
| 656-660 | `Job.create(name=config.JOB_ONBOARDING_2, ..., run_at=now + 2 days)` — likewise no `commit=True`. | `job` +1 row |
| 661-665 | `Job.create(name=config.JOB_ONBOARDING_4, ..., run_at=now + 3 days)` — likewise no `commit=True`. | `job` +1 row |

> **⚠️ `DISABLE_ONBOARDING` caveat.** `example.env` **turns this flag on** at line 150 with the comment `# For self-hosted instance`. With the default `.env` copy the three onboarding jobs **will not** be created and the `job` table will remain empty after registration. To see the three rows in `job`, delete or comment out `DISABLE_ONBOARDING=true` before launching. This is a common source of confusion for first-time operators who expect to see onboarding jobs.

The constants `JOB_ONBOARDING_1` / `JOB_ONBOARDING_2` / `JOB_ONBOARDING_3` / `JOB_ONBOARDING_4` are defined at `app/config.py:301-304`:

```python
JOB_ONBOARDING_1 = "onboarding-1"
JOB_ONBOARDING_2 = "onboarding-2"
JOB_ONBOARDING_3 = "onboarding-3"
JOB_ONBOARDING_4 = "onboarding-4"
```

Note that `JOB_ONBOARDING_3` exists as a constant but is **not scheduled** during registration — only `1`, `2`, and `4` are. (`onboarding-3` is a legacy slot; `process_job` in `job_runner.py` does not handle it either.)

#### Activation-code creation (inside `register.py`)

The private `send_activation_email(user, next_url)` helper at `app/auth/views/register.py:117` performs:

1. Deletes any prior `ActivationCode` rows for this user (forces a single outstanding code)
2. Creates a new `ActivationCode` row: `code=random_string(30)`, `user_id=user.id`, `expired=arrow.now().shift(hours=1)` (the default via `_expiration_1h` at `app/models.py:1186`)
3. Builds the activation link: `f"{URL}/auth/activate?code={activation.code}"` plus optional `&next=<next_url>`
4. Calls `email_utils.send_activation_email(user, activation_link)` at `app/email_utils.py:125` which composes a message with subject **"Just one more step to join SimpleLogin"** using the `transactional/activation.txt` and `transactional/activation.html` templates. These template paths are resolved **relative to the `templates/emails/` prefix** added by the internal `render()` helper at `app/email_utils.py:72`; on disk the files live at `templates/emails/transactional/activation.txt` and `templates/emails/transactional/activation.html`.

#### Observed logs for Step 1 (AAP §0.5.3)

During the live walkthrough these lines appeared in stdout (in order):

```text
create user testuser@example.com
send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'testuser@example.com'
Not sending events because webhook is not configured and allowed to be empty
```

Each line maps to a source location:

| Log line | Source | Notes |
|---|---|---|
| `create user testuser@example.com` | `app/auth/views/register.py:85` — `LOG.d("create user %s", email)` | Fires **before** `User.create()` so you see it even if `User.create()` fails |
| `send email with subject '...', from '...' to '...'` | `app/mail_sender.py:131-136` — inside the `if config.NOT_SEND_EMAIL:` short-circuit | Logs the composed message and returns `True` **without** opening an SMTP connection |
| `Not sending events because webhook is not configured and allowed to be empty` | `app/events/event_dispatcher.py:62-64` | Fires in `EventDispatcher.send_event()` when `EVENT_WEBHOOK` is unset and `skip_if_webhook_missing=True`. This is the expected message on a local dev setup. |

> **Note on the `from` address.** `email_utils.send_activation_email()` at `app/email_utils.py:125` calls `send_email(user.email, "Just one more step…", …)` **without** supplying `from_name` or `from_addr`. The `send_email()` helper at `app/email_utils.py:303-306` defaults both parameters to `config.NOREPLY`, which `app/config.py:435` resolves as `os.environ.get("NOREPLY", f"noreply@{EMAIL_DOMAIN}")` → `noreply@sl.local` in the local-dev env. The helper composes the header as `f'"{from_name}" <{from_addr}>'`, producing the literal string `"noreply@sl.local" <noreply@sl.local>`. It deliberately does **not** use `SUPPORT_EMAIL` / `SUPPORT_NAME` — those are reserved for transactional replies sent by `send_email_with_rate_control()` and similar helpers. The same `NOREPLY` defaulting applies to `send_welcome_email()` (Step 2) and most other automated mails; grepping for `noreply@sl.local` in the log is the reliable way to confirm the email subsystem is exercising the short-circuit path.

> **Note on the last log line.** The AAP §0.5.3 shorthand reads `"Not sending events because webhook is not configured"` but the actual emitted message (per `app/events/event_dispatcher.py:63`) is the **longer** `"Not sending events because webhook is not configured and allowed to be empty"`. This document uses the full literal string.

#### Database state after Step 1

After a single successful registration the database has been mutated as follows:

| Table | Rows added | Key columns |
|---|---|---|
| `users` | +1 | `email=<input>`, `activated=False`, `password=<bcrypt hash>`, `default_mailbox_id=<mb.id>`, `alternative_id=<uuid>`, `newsletter_alias_id=<alias.id>`, `notification=True` |
| `mailbox` | +1 | `user_id=<user.id>`, `email=<user.email>`, `verified=True` |
| `alias` | +1 | `user_id=<user.id>`, `email="simplelogin-newsletter.<random>@sl.local"`, `mailbox_id=<mb.id>`, `enabled=True`, `note="This is your first alias…"` |
| `activation_code` | +1 | `user_id=<user.id>`, `code=<30-char random>`, `expired ≈ now + 1h` |
| `job` | +3 (only if `DISABLE_ONBOARDING` unset) | `name` in `{onboarding-1, onboarding-2, onboarding-4}`, `run_at` at +1 / +2 / +3 days, `state=0` (`JobState.ready`), `attempts=0`, `payload={"user_id": ...}` |
| `daily_metric` | +1 (first reg of day) or updated | `date=CURRENT_DATE`, `nb_new_web_non_proton_user += 1` |

> **Note on `alias` columns.** The `alias` table has **no** `prefix` or `suffix` columns — the full address is stored in the single `email` column (varchar 128, unique, not null). `prefix="simplelogin-newsletter"`, `suffix=".<random>@sl.local"`, and the random middle component are keyword **arguments** passed to `Alias.create_new()` in `app/models.py` (around line 634); the factory concatenates them into the final `email` value before `INSERT`. Live `\d alias` from PostgreSQL shows 24 columns — `id, created_at, updated_at, user_id, email, enabled, custom_domain_id, automatic_creation, directory_id, note, mailbox_id, name, disable_pgp, cannot_be_disabled, disable_email_spoofing_check, batch_import_id, original_owner_id, pinned, transfer_token, transfer_token_expiration, hibp_last_check, ts_vector, last_email_log_id, flags` — none of which is `prefix` or `suffix`. A working inspection query is therefore `SELECT id, email, user_id, mailbox_id, enabled, note FROM alias WHERE user_id = <USER_ID>`.

> **Note on `job.state`.** `state` is an **integer** column (`sa.Integer`, `server_default='0'`, `default=JobState.ready.value`), not a string. The `JobState` enum at `app/models.py:253` maps `ready=0, taken=1, done=2, error=3`. So a freshly-scheduled onboarding job has `state=0`, not `state='ready'`. Filter queries must compare against integers — e.g. `SELECT id, name, state, run_at FROM job WHERE state = 0 ORDER BY id;` — otherwise PostgreSQL returns `operator does not exist: integer = unknown`.

Sample `psql` inspection commands:

```sql
-- Who got created?
SELECT id, email, activated, default_mailbox_id, newsletter_alias_id
  FROM users WHERE email = 'testuser@example.com';

-- Which mailbox was created?
SELECT id, email, verified FROM mailbox WHERE user_id = <USER_ID>;

-- Which alias was created?
SELECT email, note FROM alias WHERE user_id = <USER_ID>;

-- Are there pending activation codes?
SELECT code, expired FROM activation_code WHERE user_id = <USER_ID>;

-- Were onboarding jobs scheduled? (Empty if DISABLE_ONBOARDING=true)
SELECT id, name, run_at, state, attempts
  FROM job WHERE payload::text LIKE '%"user_id": <USER_ID>%'
  ORDER BY id;

-- Today's registration counter:
SELECT date, nb_new_web_non_proton_user FROM daily_metric WHERE date = CURRENT_DATE;
```

### 2.2 Activation — `GET /auth/activate?code=...`

**Endpoint:** `app/auth/views/activate.py:13` — `@auth_bp.route("/activate")`

**Rate limit:** `10/minute` via Flask-Limiter (rate-limit dings only on failed attempts via `g.deduct_limit = True` — a legitimate activation does not consume the quota).

**Step-by-step flow:**

1. Extract `code` from the query string.
2. `activation_code = ActivationCode.get_by(code=code)` at line 24. If `None` → return 400 with `g.deduct_limit = True`.
3. Check expiry via `activation_code.is_expired()` (`app/models.py:1214-1215` — compares `self.expired < arrow.now()`). The default expiry is **1 hour** (`_expiration_1h` at `app/models.py:1186` returns `arrow.now().shift(hours=1)`). Expired → 400.
4. **Success path (lines 48-51):**
   ```python
   user.activated = True
   login_user(user)
   ActivationCode.delete(activation_code.id)  # single-use
   Session.commit()
   ```
   - `login_user` is Flask-Login's function — it writes the `_user_id` key into the signed Flask session cookie.
   - The activation code is **deleted immediately**, so the same code cannot be reused.
5. `flash("Your account has been activated", "success")` — Bootstrap-style flash message for the next rendered page.
6. `email_utils.send_welcome_email(user)` at line 58.
7. Redirect:
   - If the request carried a `next` query parameter → `redirect(sanitize_next_url(next))` and `LOG.d("redirect user to %s", next_url)` at line 63.
   - Otherwise → `redirect(url_for("dashboard.index"))` and `LOG.d("redirect user to dashboard")` at line 66.

#### The welcome-email destination — a counter-intuitive detail

`email_utils.send_welcome_email(user)` at `app/email_utils.py:97` does **not** send to `user.email`. It calls `user.get_communication_email()` (defined at `app/models.py:1041`), which returns the **user's newsletter alias** (something like `simplelogin-newsletter.abc123@sl.local`) when all of the following hold:

- `user.notification == True` (default)
- `user.activated == True` (just set!)
- `user.disabled == False`
- `user.newsletter_alias_id` is set and the alias is still enabled

Only if those conditions fail does `get_communication_email()` fall back to `user.email`. So the observed log line

```text
send email with subject 'Welcome to SimpleLogin', from '...' to 'simplelogin-newsletter.xxx@sl.local'
```

is **correct and expected**. SimpleLogin prefers to route its own communications through the user's alias so the subscription stays within the platform's forwarding machinery (and the user can later disable the alias to unsubscribe).

#### Observed logs for Step 2 (AAP §0.5.3)

```text
send email with subject 'Welcome to SimpleLogin', from '...' to 'simplelogin-newsletter.xxx@sl.local'
redirect user to dashboard
```

| Log line | Source |
|---|---|
| `send email with subject 'Welcome to SimpleLogin', ...` | `app/mail_sender.py:131-136` (via `email_utils.py:97` call) |
| `redirect user to dashboard` | `app/auth/views/activate.py:66` — `LOG.d("redirect user to dashboard")` |

#### Database state after Step 2

| Table | Change |
|---|---|
| `users` | `activated = true` for this user |
| `activation_code` | Row **DELETED** — single-use (line 50 inside activate.py) |

Verification:

```sql
SELECT activated FROM users WHERE id = <USER_ID>;     -- -> t
SELECT count(*) FROM activation_code WHERE user_id = <USER_ID>;  -- -> 0
```

### 2.3 Login — `POST /auth/login`

**Endpoint:** `app/auth/views/login.py:21` — `@auth_bp.route("/login", methods=["GET", "POST"])`

**Rate limit:** `10/minute`.

Note that the activation flow in §2.2 calls `login_user()` itself, so the user is already logged in when they land on the dashboard the first time. The explicit `POST /auth/login` is exercised on **subsequent** visits (e.g. after logout or a fresh browser session).

**Step-by-step flow:**

1. If `current_user.is_authenticated` already → redirect to `next` or dashboard with a DEBUG log.
2. Validate the `LoginForm` (email + password).
3. Look up the user: `User.get_by(email=email)` with a fallback to the canonicalised form.
4. **Guard checks** (ordered — first match wins):

   | Condition | `LoginEvent.ActionType` | HTTP result |
   |---|---|---|
   | Missing user, or wrong password (line 50) | `failed` | Re-render login form with error flash |
   | `user.disabled` (line 56) | `disabled_login` | Re-render with "account disabled" flash |
   | `user.delete_on is not None` (line 62) | `scheduled_to_be_deleted` | Re-render with "pending deletion" message |
   | `not user.activated` (line 69) | `not_activated` | Re-render with the "resend activation" link |

   Each failure path calls `LoginEvent(LoginEvent.ActionType.<value>).send()` at `app/events/auth_event.py:23`, which records a New Relic custom event `"LoginEvent"` with `{"action": "<value>", "source": "web"}`.

5. **Success path (line 71):**
   ```python
   LoginEvent(LoginEvent.ActionType.success).send()
   return after_login(user, next_url)
   ```

**`after_login()`** lives at `app/auth/views/login_utils.py:12` and is the MFA decision tree:

| Priority | Condition | Behaviour |
|---|---|---|
| 1 | `user.fido_enabled()` (WebAuthn key registered) | Store `session[MFA_USER_ID] = user.id`, redirect to `/auth/fido` |
| 2 | `user.enable_otp` (TOTP enabled) | Store `session[MFA_USER_ID]`, redirect to `/auth/mfa` |
| 3 | Neither — plain login | `LOG.d("log user %s in", user)` (line 35) → `login_user(user)` → `session["sudo_time"] = int(time())` → redirect to `next` URL or dashboard |

For a freshly-registered test user without any MFA enrolment, the **third** path runs: the `log user <User X ...> in` log line, followed by `redirect user to dashboard` (line 44).

#### Observed logs for Step 3 (AAP §0.5.3)

```text
log user <User 3 testuser@example.com> in
redirect user to dashboard
```

Trace:

| Log line | Source |
|---|---|
| `log user <User 3 testuser@example.com> in` | `app/auth/views/login_utils.py:35` — `LOG.d("log user %s in", user)` |
| `redirect user to dashboard` | `app/auth/views/login_utils.py:44` — `LOG.d("redirect user to dashboard")` |

The `<User 3 testuser@example.com>` format comes from `User.__repr__` at `app/models.py:1182-1183`:

```python
def __repr__(self):
    return f"<User {self.id} {self.name} {self.email}>"
```

(When `self.name` is `None` or empty, Python's f-string renders it as an empty string, leaving two spaces between `<User 3` and the email — this is why the observed log shows `<User 3 testuser@example.com>` with just a single visible space.)

### 2.4 Dashboard — `GET /dashboard/`

**Endpoint:** `app/dashboard/views/index.py:55` — `@dashboard_bp.route("/", methods=["GET", "POST"])`, `@login_required`

**Rate limits:** Alias creation rate limit on POST; `10/minute` GET per `current_user.id`.

**Key operations on a first visit:**

1. `get_stats(user)` at `app/dashboard/views/index.py:32` computes the four dashboard counters:
   - `nb_alias` = `Alias.filter_by(user_id=user.id).count()` — for a freshly-activated user this is **1** (the newsletter alias).
   - `nb_forward`, `nb_reply`, `nb_block` = counts from `EmailLog` joined on the user's aliases — for a new user these are **0** because no mail has been forwarded yet.
2. `mailboxes = current_user.mailboxes()` — returns a list of the user's verified mailboxes, ordered. For a new user this is the single default mailbox created during registration.
3. **Intro tour logic.** Lines 170-177 (approximately):
   ```python
   show_intro = False
   if not current_user.intro_shown:
       LOG.d("Show intro to %s", current_user)
       show_intro = True
       current_user.intro_shown = True
       Session.commit()
   ```
   Because `intro_shown` is persisted to the `users` table, **the `Show intro to ...` log fires exactly once per account** — a perfect single-shot signal that the user has reached the dashboard for the first time.
4. `alias_infos = get_alias_infos_with_pagination_v3(current_user, page_id=page, ...)` — returns the current page of aliases (fetches `PAGE_LIMIT + 1` rows to detect whether a "next page" link is needed).
5. Renders `templates/dashboard/index.html` with `alias_infos`, `mailboxes`, `show_intro`, `stats`, and pagination variables.

#### The intro tour UI behaviour

The template wraps its `introJs().start()` invocation inside `{% if show_intro %}`. Additionally the client-side JS checks `window.innerWidth >= 1024` before kicking off the tour, so on narrow viewports (mobile) the highlight overlay is silently suppressed. The DEBUG log line still fires server-side regardless of viewport — the log is a **reliable** first-visit indicator; the UI overlay is not.

#### Observed logs for Step 4 (AAP §0.5.3)

```text
Show intro to <User 3 testuser@example.com>
```

Trace:

| Log line | Source |
|---|---|
| `Show intro to <User 3 ...>` | `app/dashboard/views/index.py:172` — `LOG.d("Show intro to %s", current_user)` |

There are of course also the per-request DEBUG lines from `server.py:272-297` (the `@after_request` hook), for example:

```text
127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.123
```

The exact format string in `server.py:285` is `"%s %s %s %s %s, takes %s"` populated with (`remote_addr`, `method`, `path`, `args`, `status_code`, `elapsed`) — there is **no** user/session field in this log. The four fields `remote_addr`, `method`, `path`, and `args` are space-separated with no delimiters; `args` is the Werkzeug `ImmutableMultiDict` repr, which is `ImmutableMultiDict([])` for an empty query string. (The identity of the signed-in user during the request is instead emitted separately by the view-level logs such as `Show intro to <User ...>` and `log user <User ...> in`.)

#### Visible UI elements on the dashboard

Per `templates/dashboard/index.html` and the rendered page, a first-visit dashboard shows:

- A stats strip near the top showing `nb_alias / nb_forward / nb_reply / nb_block` — all `0` except `nb_alias=1` for a brand-new user.
- The single alias row: `simplelogin-newsletter.<random>@sl.local`, labelled as the user's first alias.
- Two primary buttons above the alias list: **"Create random alias"** and **"Create custom alias"**.
- A sidebar/menu with links to *Mailboxes*, *Custom Domains*, *Settings*, and *API Keys*.
- An IntroJS guided-tour overlay (only when `show_intro=True` **and** viewport ≥ 1024 px).

### 2.5 End-to-End Verification Timeline

Putting all four requests together, a first-time user's session produces exactly this sequence of HTTP calls, DB mutations, and log lines:

| # | HTTP | Status | Redirect target | DB effect | Canonical log line |
|---|---|---|---|---|---|
| 1 | `GET /` | 302 | `/auth/login` | none | `127.0.0.1 GET / ImmutableMultiDict([]) 302, takes ...` |
| 2 | `GET /auth/register` | 200 | – | none | `127.0.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes ...` |
| 3 | `POST /auth/register` | 200 (renders waiting page) | – | `users +1`, `mailbox +1`, `alias +1`, `activation_code +1`, `job +3` (if `DISABLE_ONBOARDING` unset), `daily_metric +1` | `create user testuser@example.com` |
| 4 | `GET /auth/activate?code=...` | 302 | `/dashboard/` | `users.activated=true`, `activation_code -1` | `redirect user to dashboard` |
| 5 | `GET /dashboard/` (first time) | 200 | – | `users.intro_shown=true` | `Show intro to <User 3 ...>` |
| 6 | (Later) `POST /auth/login` | 302 | `/dashboard/` | session cookie issued | `log user <User 3 ...> in` |

**Success signals to watch for:**

- ✅ `POST /auth/register` returns `200` and the browser lands on the waiting-activation page.
- ✅ A row appears in `activation_code` with `expired > NOW()`.
- ✅ Two `send email with subject ...` log lines appear (activation + welcome) when `NOT_SEND_EMAIL=true`.
- ✅ After the activation link is opened, `users.activated` flips to `true` and the `activation_code` row disappears.
- ✅ The browser is redirected to `/dashboard/` with a valid session cookie.
- ✅ `Show intro to <User X ...>` appears exactly once in the logs (first dashboard hit).
- ✅ The `alias` table holds the `simplelogin-newsletter.<random>@sl.local` row and the `users.newsletter_alias_id` FK points at it.

---

## 3. Background Service Observability

This section answers: **during and after the new-user flow, what runtime indicators show that background jobs, internal services, and inter-process communication are active and correctly supporting email forwarding and identity verification?**

SimpleLogin is a multi-process system. The web worker is only one of six operational roles. This section enumerates every process, explains how to observe each one, and documents the specific logs, database tables, and metrics that signal health.

### 3.1 The Five-Process Architecture (plus Monitoring)

A fully-deployed SimpleLogin instance runs the following processes in parallel. All of them share the same PostgreSQL database and (optionally) Redis cache. They coordinate via DB rows, `NOTIFY`/`LISTEN` channels, and — for inbound mail — the local MTA queue.

| # | Process | Entrypoint | Role |
|---|---|---|---|
| 1 | **Web** | `gunicorn wsgi:app` → `server.py:create_app()` (line 139) | Serves HTTP: `/auth/*`, `/dashboard/*`, `/api/*`, `/oauth/*`, `/oauth2/*`, `/admin/*`, etc. |
| 2 | **Job Runner** | `python job_runner.py` | Polls the `job` table every 10 s and executes queued work (onboarding emails, account/mailbox/domain deletion, batch imports, user reports, etc.) |
| 3 | **Cron Scheduler** | `yacron -c crontab.yml` (primary host) + `yacron -c crontab-all-hosts.yml` (per-host) | Fires scheduled maintenance tasks (stats, HIBP checks, subscription notifications, cleanup) by shelling out to `python /code/cron.py -j <function_name>` |
| 4 | **Inbound SMTP Handler** | `python email_handler.py` (aiosmtpd) | Receives mail on an SMTP port, routes forwards and replies through aliases, handles bounces |
| 5 | **Event Listener** | `python event_listener.py listener` | PostgreSQL `LISTEN simplelogin_sync_events` consumer; pushes protobuf events to the Proton webhook when configured. (The CLI uses `argparse` subparsers, so the mode is passed as a **positional** sub-command — `listener`, `dead_letter`, `debug`, or `run` — **not** as a `--mode` flag.) |
| 6 | **Monitoring** | `python monitoring.py` | Every 60 s: Postfix queue depth, PG connection counts, `sync_event` backlog; pushes custom metrics to New Relic |

During a local single-user smoke test, **only the web process is strictly required** — the other five produce side-effects in the database that can be inspected even if those processes are not running. Running them in parallel (e.g. via `honcho` or separate terminals) surfaces additional observable log lines.

### 3.2 Job Runner — `job_runner.py`

**Entry:** `if __name__ == "__main__":` block at `job_runner.py:329`.

**Loop:** Infinite `while True`, each iteration:

1. Enter `create_light_app().app_context()` (imported from `server.py:127`). `create_light_app` creates a stripped-down Flask app with only DB access — no blueprints, no Flask-Login, no Limiter. This keeps the background worker memory-light.
2. Call `get_jobs_to_run()` at `job_runner.py:307`:
   ```sql
   -- Conceptually (remember `state` is an integer enum — JobState.ready=0, taken=1, done=2, error=3):
   SELECT * FROM job
    WHERE (state = 0 /* ready */ OR (state = 1 /* taken */ AND taken_at < NOW() - INTERVAL 'JOB_TAKEN_RETRY_WAIT_MINS minutes'))
      AND attempts < JOB_MAX_ATTEMPTS
      AND (run_at IS NULL OR run_at <= NOW() + INTERVAL '10 minutes')
   ```
3. For each returned `Job`: mark `state=1` (`JobState.taken`), `taken_at=now()`, `taken=True`, `attempts += 1`, commit, then call `process_job(job)` at line 188, then `state=2` (`JobState.done`), commit.
4. `time.sleep(10)` at `job_runner.py:347` — **10-second poll interval**.

**`process_job(job)`** (line 188) dispatches on `job.name`:

| `job.name` constant | Value | Handler |
|---|---|---|
| `JOB_ONBOARDING_1` | `"onboarding-1"` | `onboarding_send_from_alias(user)` — subject "SimpleLogin Tip: Send emails from your alias" |
| `JOB_ONBOARDING_2` | `"onboarding-2"` | `onboarding_mailbox(user)` — subject "SimpleLogin Tip: Manage Your mailbox" |
| `JOB_ONBOARDING_4` | `"onboarding-4"` | `onboarding_pgp(user)` — subject "SimpleLogin Tip: Secure your emails with PGP". Skipped if the user's only mailbox is a Proton mailbox. |
| `JOB_BATCH_IMPORT` | `"batch-import"` | Bulk-creates aliases from a CSV the user uploaded |
| `JOB_DELETE_ACCOUNT` | `"delete-account"` | Tears down a user and all owned records |
| `JOB_DELETE_MAILBOX` | `"delete-mailbox"` | Deletes a mailbox after the user confirms via email |
| `JOB_DELETE_DOMAIN` | `"delete-domain"` | Deletes a custom domain |
| `JOB_SEND_USER_REPORT` | `"send-user-report"` | Emails the user their monthly usage report |
| `JOB_SEND_PROTON_WELCOME_1` | `"proton-welcome-1"` | Sends the first welcome email to Proton-partnered users |

Every onboarding handler has three identical guard clauses: skip if `User.get(user_id)` returns `None` (user was deleted), skip if `user.notification=False` (user opted out), skip if `not user.activated`.

#### Observable evidence — **without** running the job runner

The `job` table itself is evidence of scheduling activity:

```sql
SELECT id, name, run_at, state, attempts
  FROM job
  WHERE name LIKE 'onboarding-%'
  ORDER BY id DESC LIMIT 5;
```

After a fresh registration (with `DISABLE_ONBOARDING` unset) you see three rows with `state=0` (`JobState.ready`; `state` is an integer column, **not** a string — see the note in §2.1), `attempts=0`, and `run_at` one/two/three days in the future.

#### Observable evidence — **with** the job runner running

The job runner produces these DEBUG lines per onboarding email:

```text
Take job <Job id=42 name=onboarding-1 run_at=...>
send onboarding send-from-alias email to user <User 3 ...>
send email with subject 'SimpleLogin Tip: Send emails from your alias', from '...' to 'simplelogin-newsletter.xxx@sl.local'
```

The first line is emitted from `job_runner.py:334` (`LOG.d("Take job %s", job)`). The second line pattern is at `job_runner.py:196` (`LOG.d("send onboarding send-from-alias email to user %s", user)`). The third line is the usual `MailSender.send()` NOT_SEND_EMAIL short-circuit.

### 3.3 Cron Scheduler — `cron.py` + `crontab.yml` + `crontab-all-hosts.yml`

**`crontab.yml`** (primary-host schedule, per the file observed at repo root) defines **15** jobs, each shelling out to `python /code/cron.py -j <function>`:

| Time (UTC) | Job | Function in `cron.py` | Purpose |
|---|---|---|---|
| `0 0 * * *` (00:00 daily) | `stats` | `stats` (line 537) | Aggregate platform-wide metrics into the `metric2` table |
| `15 1 * * *` | `delete_old_monitoring` | `delete_old_monitoring` (line 954) | Prune old `monitoring` table rows |
| `15 2 * * *` | `check_custom_domain` | `check_custom_domain` (line 902) | Verify DNS for user custom domains, mark verification state |
| `15 3 * * *` (Forbid) | `check_hibp` | `_hibp_check` / helpers | Cross-check alias emails against Have-I-Been-Pwned corpus |
| `15 4 * * *` (Forbid) | `notify_hibp` | `notify_hibp` (line 1171) | Email users whose aliases matched HIBP breaches |
| `15 5 * * *` | `delete_logs` | `delete_logs` (line 92) | Prune old `email_log` rows |
| `30 5 * * *` | `delete_old_data` | `delete_old_data` (line 1245) | Purge audit logs, deleted-user artefacts, expired tokens, etc. |
| `15 6 * * *` | `poll_apple_subscription` | `poll_apple_subscription` (line 287) | Reconcile Apple IAP subscription state |
| `15 8 * * *` | `notify_trial_end` | `notify_trial_end` (line 75) | Email users whose free trial is ending |
| `15 9 * * *` | `notify_manual_subscription_end` | `notify_manual_sub_end` (line 188) | Warn manual-subscription users their subscription is ending |
| `15 10 * * *` | `notify_premium_end` | `notify_premium_end` (line 156) | Warn Premium users about upcoming renewal / expiry |
| `15 11 * * *` (Forbid) | `delete_scheduled_users` | `clear_users_scheduled_to_be_deleted` (line 1223) | Finalise account deletions past the grace period |
| `*/5 * * * *` (Forbid) | `send_undelivered_mails` | `send_undelivered_mails` in `cron.py` | Retry messages persisted to the filesystem dead-letter directory by `mail_sender.py` when SMTP delivery failed |
| `0 * * * *` (hourly, Forbid) | `clear_alias_audit_log` | `clear_alias_audit_log` (line 1252) | Prune alias audit log entries older than retention |
| `0 * * * *` (hourly, Forbid) | `clear_user_audit_log` | `clear_user_audit_log` (line 1257) | Prune user audit log entries older than retention |

`concurrencyPolicy: Forbid` appears on `check_hibp`, `notify_hibp`, `delete_scheduled_users`, `send_undelivered_mails`, `clear_alias_audit_log`, and `clear_user_audit_log` — these tasks are long-running and/or idempotency-sensitive, so yacron refuses to start a new instance while a previous one is still active.

**`crontab-all-hosts.yml`** is the per-host variant, intended to be deployed on every SimpleLogin host in a multi-host deployment. In this repository it currently contains a single job — `send_undelivered_mails` (same `*/5 * * * *` schedule, `concurrencyPolicy: Forbid`) — so that every worker node independently re-tries its own local filesystem dead-letter queue.

**`cron.py`** exposes 25+ top-level `def` functions. A representative list (with line numbers verified against the source):

| Line | Function | Role |
|---|---|---|
| 75 | `notify_trial_end` | Email users whose free trial is ending |
| 92 | `delete_logs` | Prune `email_log` entries older than retention |
| 136 | `delete_refused_emails` | Clean up rejected mail artifacts |
| 156 | `notify_premium_end` | Warn Premium users about renewal |
| 188 | `notify_manual_sub_end` | Warn manual-subscription users |
| 287 | `poll_apple_subscription` | Apple IAP reconciliation |
| 304 | `compute_metric2` | Compute per-user stats |
| 401 | `bounce_report` | Daily bounce summary |
| 488 | `alias_creation_report` | Aggregate new-alias counts |
| 537 | `stats` | Platform-wide metric rollup |
| 724 | `sanity_check` | Self-consistency audits |
| 797 | `check_mailbox_valid_domain` | Mailbox-domain DNS check |
| 864 | `check_mailbox_valid_pgp_keys` | Mailbox PGP key validity |
| 902 | `check_custom_domain` | User custom-domain DNS check |
| 954 | `delete_old_monitoring` | Trim monitoring history |
| 964 | `delete_expired_tokens` | Purge expired auth tokens |
| 1171 | `notify_hibp` | Email HIBP matches |
| 1223 | `clear_users_scheduled_to_be_deleted` | Finalise account deletions past grace period |
| 1245 | `delete_old_data` | Aggregate cleanup |
| 1252 | `clear_alias_audit_log` | Prune alias audit log |
| 1257 | `clear_user_audit_log` | Prune user audit log |

Each task executes inside `create_light_app().app_context()` and emits DEBUG/INFO logs through the same `LOG` object, so output lands in the cron process's stdout/stderr. yacron is configured with `captureStderr: true` so its log tail also captures Python stack traces.

**Key task for email reliability:** `send_undelivered_mails` (in the all-hosts variant) retries messages that `app/mail_sender.py` persisted to the filesystem dead-letter directory when SMTP delivery failed in live (`NOT_SEND_EMAIL=false`) mode. During a local test with `NOT_SEND_EMAIL=true`, this task is a no-op because no messages are ever enqueued for retry.

### 3.4 Event Dispatcher — `app/events/event_dispatcher.py`

SimpleLogin uses the word "event" for two distinct subsystems. It is crucial to distinguish them:

1. **New Relic custom events** — pure analytics; recorded via `newrelic.agent.record_custom_event(name, attrs)`. Always emitted when the New Relic agent is active. Types: `LoginEvent`, `RegisterEvent`, `HttpResponseStatus` (per request), `EventStoredToDb` (per sync event).
2. **Proton sync events** — protobuf payloads shipped to the Proton ecosystem via a PostgreSQL `NOTIFY` channel consumed by the `event_listener.py` process, which then HTTP-POSTs them to `config.EVENT_WEBHOOK`. **These are the events the webhook-missing log line refers to.**

**`EventDispatcher.send_event(user, content, dispatcher, skip_if_webhook_missing=True)`** at `app/events/event_dispatcher.py:48` contains the local-dev short-circuit logic:

```python
# Approximate shape of the function:
if config.EVENT_WEBHOOK_DISABLE:
    LOG.i("Not sending events because webhook is disabled")
    return
if not config.EVENT_WEBHOOK and skip_if_webhook_missing:
    LOG.i("Not sending events because webhook is not configured and allowed to be empty")
    return
partner_user = get_partner_user(user)
if partner_user is None:
    LOG.i(f"Not sending events because there's no partner user for user {user}")
    return
event = event_pb2.Event(user_id=..., external_user_id=..., partner_id=..., content=content)
serialized = event.SerializeToString()
dispatcher.send(serialized)   # default PostgresDispatcher → NOTIFY simplelogin_sync_events, '<id>'
newrelic.agent.record_custom_event("EventStoredToDb", {...})
LOG.i("Sent event to the dispatcher")
```

(Line references: `app/events/event_dispatcher.py:57-58` for the disable branch; lines 61-64 for the missing-webhook branch; lines 67-69 for the no-partner-user branch.)

**`PostgresDispatcher.send()`** (lines 24-26): inserts a row into `sync_event` with the serialised protobuf, then executes:

```sql
NOTIFY simplelogin_sync_events, '<sync_event.id>';
```

The channel name is defined at `app/events/event_dispatcher.py:14` — `NOTIFICATION_CHANNEL = "simplelogin_sync_events"`.

#### Why the "not sending events" log is *good* news

When the local `.env` has no `EVENT_WEBHOOK` set, every call site (`register.py`, account deletions, plan changes, partner sync) flows into the second branch and logs:

```text
Not sending events because webhook is not configured and allowed to be empty
```

This confirms three things simultaneously: (a) `EventDispatcher.send_event()` was actually reached (the caller was wired up correctly), (b) `EVENT_WEBHOOK` is intentionally unset for local dev, (c) no noisy `NOTIFY` rows are being inserted into the database.

### 3.5 Auth Events — `app/events/auth_event.py`

These are the **New Relic** custom events for login/registration analytics. They are separate from the Proton sync events above.

**`LoginEvent`** (top of `app/events/auth_event.py`):

| `ActionType` | Emitted from |
|---|---|
| `success` | `login.py:71` — successful password login |
| `failed` | `login.py:50` — wrong password or missing user |
| `disabled_login` | `login.py:56` — user account is disabled |
| `not_activated` | `login.py:69` — user hasn't activated yet |
| `scheduled_to_be_deleted` | `login.py:62` — user is pending deletion |

`LoginEvent.send()` at line 23 calls:

```python
newrelic.agent.record_custom_event("LoginEvent", {"action": self.action.name, "source": self.source.name})
```

Because `ActionType` and `Source` are declared via Python's `enum.Enum`, `.name` returns the human-readable identifier (e.g. `"success"`, `"failed"`, `"catpcha_failed"`, `"web"`, `"api"`), whereas `.value` would return the underlying enum member value. The `.name` string is what appears on New Relic's dashboards — so operators filtering custom events search by `action = "success"` / `action = "catpcha_failed"` (note the verbatim typo preserved from the source enum), not by numeric value.

**`RegisterEvent`** (same file):

| `ActionType` | Emitted from |
|---|---|
| `success` | `register.py:96` — successful registration |
| `catpcha_failed` | `register.py:65` — failed hCaptcha (note: typo in source; kept verbatim) |
| `email_in_use` | `register.py:76,83` — email already registered |
| `invalid_email` | `register.py:101` — activation-email dispatch failed |
| `failed` | generic fallback |

`RegisterEvent.send()` at line 46 records `"RegisterEvent"` as the NR event name.

Both `Source` enum values are `web` and `api`; only `web` is emitted by the view handlers above.

### 3.6 Email Subsystem — `app/mail_sender.py` and `NOT_SEND_EMAIL` mode

The **single most important** observability feature for a local test is the `NOT_SEND_EMAIL` short-circuit at `app/mail_sender.py:126-138`. Relevant excerpt:

```python
def send(self, send_request: SendRequest, retries: int = 2) -> bool:
    ...
    if config.NOT_SEND_EMAIL:
        LOG.d(
            "send email with subject '%s', from '%s' to '%s'",
            send_request.msg[headers.SUBJECT],
            send_request.msg[headers.FROM],
            send_request.msg[headers.TO],
        )
        return True
    # ... real SMTP path below
```

(Line 130 is the `if config.NOT_SEND_EMAIL:` check; lines 131-135 are the multi-line `LOG.d` call.)

**Behaviour when `NOT_SEND_EMAIL=true`:**

- No SMTP connection is opened (no `smtplib.SMTP(...)` call).
- A single DEBUG line records the composed subject, from-address, and to-address.
- `send()` returns `True` so the caller believes delivery succeeded.

**Behaviour when `NOT_SEND_EMAIL=false`:**

- If a background `ThreadPoolExecutor` was configured via `enable_background_pool()`, the message is submitted asynchronously.
- Otherwise `_send_to_smtp(send_request, retries)` opens `smtplib.SMTP(config.POSTFIX_SERVER, config.POSTFIX_PORT, timeout=config.POSTFIX_TIMEOUT)` and calls `smtp.sendmail(...)`.
- On failure the message is persisted to a filesystem dead-letter directory; `cron.py`'s `send_undelivered_mails` task retries it later.
- New Relic custom metrics `Custom/smtp_connection_time` and `Custom/smtp_sending_time` are recorded for APM observability.

**Observable pattern during the new-user flow:**

Every attempted email produces exactly one `send email with subject '...', from '...' to '...'` DEBUG line in the web process log. During a single registration → activation cycle expect **two** such lines:

| # | Subject | From | To | Triggered by |
|---|---|---|---|---|
| 1 | `Just one more step to join SimpleLogin` | `"noreply@sl.local" <noreply@sl.local>` | `testuser@example.com` | `email_utils.send_activation_email()` at line 125 |
| 2 | `Welcome to SimpleLogin` | `"noreply@sl.local" <noreply@sl.local>` | `simplelogin-newsletter.<random>@sl.local` | `email_utils.send_welcome_email()` at line 97 |

Both mails use `config.NOREPLY` as both the display name and the SMTP-From address — the `send_email()` helper at `app/email_utils.py:303-306` defaults `from_name` and `from_addr` to `config.NOREPLY` when the caller does not override them, and `app/config.py:435` resolves `NOREPLY` to `noreply@{EMAIL_DOMAIN}` → `noreply@sl.local` for the local-dev env. Neither helper routes through `SUPPORT_EMAIL` / `SUPPORT_NAME`, so grepping for `support@sl.local` in the log will **not** locate either of these lines.

### 3.7 Inbound SMTP Handler — `email_handler.py`

Not strictly required for the new-user flow (registration and login do not receive any inbound mail), but documented here for completeness because email forwarding is SimpleLogin's core value proposition.

**Entry:** `python email_handler.py` — an aiosmtpd-based SMTP server. The listening port is taken from the `--port` argparse argument whose `default=20381` (the numeric default hard-coded in `email_handler.py`'s `argparse.ArgumentParser` definition). In a production deployment Postfix forwards mail to this port on the loopback interface, so the port is usually left at the default or overridden via CLI.

**Three actors** (per the docstring at the top of the file):
- **Contact** — the external sender who emailed `alias@sl.co`.
- **SL email handler** — this process.
- **User personal email** — the real mailbox behind the alias.

**Two phases:**
- **Forward.** `contact → alias@sl.co → SL handler → user's real mailbox`. The handler rewrites headers, signs with DKIM, and relays through Postfix.
- **Reply.** `user → reply+special@sl.co → SL handler → contact`. The handler matches a special reverse-alias token to the original sender, rewrites again, and relays.

**Observability:** stdout logs like `new message from ..., to ...`, `forward email from ... to ...`, bounce handling traces, and `mail sent from` summaries. During a registration-only test, this process sits idle waiting for connections.

### 3.8 Event Listener — `event_listener.py`

**Entry:** `python event_listener.py listener` (or `python event_listener.py dead_letter`, `python event_listener.py debug`, `python event_listener.py run`).

The CLI uses `argparse` sub-parsers, so the mode is passed as a **positional** sub-command — not a `--mode` flag.

**`listener` mode:**

1. Opens a psycopg2 connection using `config.EVENT_LISTENER_DB_URI` (falls back to `DB_URI` when unset).
2. Executes `LISTEN simplelogin_sync_events;`.
3. In a loop: receive `NOTIFY` payloads (each payload is a `sync_event.id`), fetch the corresponding `sync_event` row, deserialise the protobuf, pass it to the configured `EventSink`.
4. Sinks available: `HttpEventSink` (POST to `config.EVENT_WEBHOOK`) and `ConsoleEventSink` (dry-run — log to stdout).

**`dead_letter` mode:** Scans the `sync_event` table for rows with `retry_count < max_retries` and `taken_time` older than the retry window. Re-drives them through the same sink.

**Observability:** stdout logs `Received event <id>`, `Pushing event to webhook ...`, `HTTP 200 from webhook`, plus any connection errors. In a local dev setup without `EVENT_WEBHOOK`, this process typically isn't started because `EventDispatcher.send_event()` short-circuits before it would ever produce work.

### 3.9 Monitoring — `monitoring.py`

**Entry:** `if __name__ == "__main__":` block at `monitoring.py:157-171`.

**Loop:** `while True`, `sleep(60)` at line 171 — **60-second tick**.

**Per-tick actions (in order):**

| # | Function | What it measures | Log line |
|---|---|---|---|
| 1 | `log_postfix_metrics()` (line 39) | Postfix queue sizes: incoming / active / deferred; counts of smtp, smtpd, bounce, cleanup processes | `postfix queue sizes <inc> <act> <def>` (line 44) |
| 2 | `log_nb_db_connection()` | `SELECT count(*) FROM pg_stat_activity` | `number of db connections %s` |
| 3 | `log_pending_to_process_events()` | `SELECT count(*) FROM sync_event WHERE taken_time IS NULL` | `number of events pending to process %s` |
| 4 | `log_events_pending_dead_letter()` | Events > 10 min old without completion | `number of events pending dead letter %s` |
| 5 | `log_failed_events()` | `sync_event` rows with `retry_count >= 10` | `number of failed events %s` |
| 6 | `log_nb_db_connection_by_app_name()` | `pg_stat_activity` grouped by `application_name` starting with `sl-` | Per-app-name DEBUG line |
| 7 | `Session.close()` | Close DB session | — |
| 8 | `exporter.run()` | `MetricExporter(get_newrelic_license())` pushes buffered metrics to New Relic | — |

Each function records a `Custom/<metric_name>` counter into the New Relic APM agent: `Custom/postfix_incoming_queue`, `Custom/postfix_active_queue`, `Custom/postfix_deferred_queue`, `Custom/process_<name>_count`, `Custom/nb_db_connections`, `Custom/sync_events_pending_to_process`, `Custom/sync_events_pending_dead_letter`, `Custom/sync_events_failed`, and `Custom/nb_db_app_connection/<name>`.

**Resilience:** The module tracks a module-level `_nb_failed` counter (line ~25) with `_max_nb_fails=10`; if ten consecutive ticks raise exceptions the loop aborts (so that a stuck monitoring process doesn't silently hide a real outage).

**Safety cap:** `_max_incoming=50` — postfix incoming queues larger than this trigger warning log output.

### 3.10 Observability Matrix — "What's Happening Now?"

The following table is a one-page reference for the key runtime signals a local operator should be able to observe. Each signal is tied to a specific source file and line (or table column).

| Subsystem | Signal to look for | Location | Meaning |
|---|---|---|---|
| Logger init | `>>> init logging <<<` | stdout (process start) | `app/log.py:67` — logger initialised |
| Config load | `>>> URL: http://localhost:7777` | stdout (process start) | `app/config.py:80` — `dotenv` parsed, `URL` resolved |
| Per-request log | `<ip> <method> <path> <args> <status>, takes <sec>` | web stdout | `server.py:272-297` — `@after_request` fired; format string is `"%s %s %s %s %s, takes %s"` (`server.py:285`) populated with 6 positional fields — **no** user/session field |
| User create | `create user <email>` | web stdout | `app/auth/views/register.py:85` reached |
| Email composed (`NOT_SEND_EMAIL`) | `send email with subject '...', from '...' to '...'` | web stdout | `app/mail_sender.py:131-136` — email composed; no SMTP attempted |
| Event dispatch skip (local) | `Not sending events because webhook is not configured and allowed to be empty` | web stdout | `app/events/event_dispatcher.py:63` — expected in local dev |
| Login success | `log user <User X ...> in` | web stdout | `app/auth/views/login_utils.py:35` |
| Auth redirect | `redirect user to dashboard` | web stdout | `app/auth/views/login_utils.py:44` **or** `app/auth/views/activate.py:66` |
| Intro tour (first dashboard hit) | `Show intro to <User X ...>` | web stdout | `app/dashboard/views/index.py:172` — fires exactly once per account |
| Activation-code table | Row with fresh `expired ≈ NOW + 1h` | `activation_code` | `ActivationCode._expiration_1h` at `app/models.py:1186` |
| Job scheduled | Rows in `job` with `name IN ('onboarding-1','onboarding-2','onboarding-4')` | `job` table | `app/models.py:651-665` — requires `DISABLE_ONBOARDING` unset |
| Job pickup | `Take job <Job id=... name=... run_at=...>` | job_runner stdout | `job_runner.py:334` |
| Job send | `send onboarding send-from-alias email to user <User ...>` | job_runner stdout | `job_runner.py:196` |
| Cron tick | yacron task output on the configured schedule | cron stdout | `crontab.yml` + `cron.py` |
| Monitor tick | `postfix queue sizes <inc> <act> <def>`, `number of db connections N` | monitoring stdout | `monitoring.py:44` and related |
| Event listener received | `Received event <id>` / `Pushing event to webhook` | event_listener stdout | `event_listener.py` (only when `EVENT_WEBHOOK` is configured) |
| Inbound mail received | `new message from ..., to ...` | email_handler stdout | `email_handler.py` (only when mail is inbound) |
| PG `NOTIFY` channel | `NOTIFY simplelogin_sync_events, '<id>'` | PG server log | `app/events/event_dispatcher.py:14` |

### 3.11 Quick Smoke-Test Commands

A few read-only commands an operator can run to confirm background-service health without starting every process:

```bash
# 1. Confirm Gunicorn is listening
curl -sS -o /dev/null -w 'GET /health → %{http_code}\n' http://localhost:7777/health

# 2. Confirm the database has the expected SL domain seed
psql -U myuser -d simplelogin -c "SELECT domain FROM public_domain ORDER BY id;"

# 3. Confirm the latest registration created the expected rows
psql -U myuser -d simplelogin -c \
  "SELECT u.id, u.email, u.activated, m.email AS mb, a.email AS newsletter_alias
     FROM users u
     JOIN mailbox m ON m.id = u.default_mailbox_id
     LEFT JOIN alias a ON a.id = u.newsletter_alias_id
     WHERE u.email = 'testuser@example.com';"

# 4. Confirm the onboarding jobs (or their absence) — depends on DISABLE_ONBOARDING
psql -U myuser -d simplelogin -c \
  "SELECT id, name, run_at, state FROM job WHERE payload::text LIKE '%testuser%' ORDER BY id;"

# 5. Confirm today's registration counter
psql -U myuser -d simplelogin -c \
  "SELECT date, nb_new_web_non_proton_user FROM daily_metric WHERE date = CURRENT_DATE;"

# 6. Confirm no sync_events are stuck (should be 0 locally)
psql -U myuser -d simplelogin -c \
  "SELECT count(*) FILTER (WHERE taken_time IS NULL) AS pending,
          count(*) FILTER (WHERE retry_count >= 10) AS failed
     FROM sync_event;"
```

---

## 4. Post-Verification Cleanup

The user's explicit rule (AAP §0.7.1) requires that **any test users, aliases, or temporary data created during verification be removed after verification is complete**. This section documents the SQL used for cleanup, in a foreign-key-safe order.

> **FK-constraint reality (per `app/models.py`).** The cleanup order below is **not** driven by `ON DELETE RESTRICT` — in fact, no user-owned FK uses `RESTRICT`. The specific mix that this script has to cope with is:
>
> - `users.default_mailbox_id → mailbox.id` has **no** explicit `ondelete=` clause, so SQLAlchemy falls back to the SQL default (`NO ACTION`). **This** is why step 5 (`UPDATE users SET default_mailbox_id = NULL …`) is required before deleting the mailbox: without nulling this FK first, the mailbox `DELETE` would fail with a foreign-key violation.
> - `users.newsletter_alias_id → alias.id` is declared with `ondelete="SET NULL"`, so the database would null it automatically when the alias is removed. Step 3 nulls it explicitly anyway, for clarity and so the script does not depend on the server-side cascade evaluation order.
> - Every `*.user_id` FK (alias, mailbox, job, activation_code, etc.) is `ondelete="cascade"`. In principle a single `DELETE FROM users WHERE id = <id>` would cascade through most of these, but the step-by-step ordering in the script keeps every intermediate table auditable and makes failure diagnostics (which step rolled back?) trivial.

> **Preferred path: `User.delete()`.** In production SimpleLogin code, the canonical way to tear down a user is `User.delete(user_id)` at `app/models.py:671` — it cascades through owned records and emits a `UserDeleted` sync event via `EventDispatcher.send_event()`. That event path is unwanted in a pure observability cleanup (and would just log "Not sending events because webhook is not configured and allowed to be empty" anyway in local dev), so for a read-only smoke test the direct SQL below is both sufficient and auditable.

### FK-Safe Cleanup SQL

Replace `<TEST_USER_ID>` with the integer `id` of the test user from the `users` table (e.g. `SELECT id FROM users WHERE email='testuser@example.com';`).

```sql
BEGIN;

-- 1. Purge any remaining single-use activation codes
DELETE FROM activation_code WHERE user_id = <TEST_USER_ID>;

-- 2. Purge queued onboarding jobs (if DISABLE_ONBOARDING was unset when the user registered)
DELETE FROM job WHERE payload::text LIKE '%"user_id": <TEST_USER_ID>%';

-- 3. Break the newsletter_alias_id FK on users so the alias row can be removed
UPDATE users SET newsletter_alias_id = NULL WHERE id = <TEST_USER_ID>;

-- 4. Purge the user's aliases (includes the simplelogin-newsletter.xxx alias)
DELETE FROM alias WHERE user_id = <TEST_USER_ID>;

-- 5. Break the default_mailbox_id FK on users so the mailbox can be removed
UPDATE users SET default_mailbox_id = NULL WHERE id = <TEST_USER_ID>;

-- 6. Purge the user's mailboxes
DELETE FROM mailbox WHERE user_id = <TEST_USER_ID>;

-- 7. Optionally decrement today's registration counter so metrics stay honest
UPDATE daily_metric
   SET nb_new_web_non_proton_user = GREATEST(nb_new_web_non_proton_user - 1, 0)
 WHERE date = CURRENT_DATE;

-- 8. Remove the user
DELETE FROM users WHERE id = <TEST_USER_ID>;

COMMIT;
```

### Verification After Cleanup

```sql
-- All of these should return 0 rows:
SELECT count(*) FROM users        WHERE email = 'testuser@example.com';
SELECT count(*) FROM mailbox      WHERE email = 'testuser@example.com';
SELECT count(*) FROM alias        WHERE email LIKE 'simplelogin-newsletter.%@sl.local';
SELECT count(*) FROM activation_code WHERE user_id = <TEST_USER_ID>;
SELECT count(*) FROM job          WHERE payload::text LIKE '%"user_id": <TEST_USER_ID>%';

-- The SL domain seed should still be present (idempotent from init_app.py):
SELECT domain FROM public_domain;  -- should include 'sl.local'
```

### What *Remains* After Cleanup (by design)

- The `public_domain` table row for `EMAIL_DOMAIN` (e.g. `sl.local`) — seeded by `init_app.py:39-56`, idempotent, harmless, and required for the next registration. (The SQLAlchemy model class is `SLDomain`, but the underlying table name on disk is `public_domain` — see `app/models.py:3116-3119`.)
- All Alembic migration records in the `alembic_version` table.
- PGP keys loaded into the GnuPG keyring by `load_pgp_public_keys()` at `init_app.py:13` — these live in the GnuPG home directory on the filesystem, not in PostgreSQL.
- The Proton partner record created by `add_proton_partner()` — this helper is defined in `init_app.py:59` but is **not** invoked by `init_app.py`'s `if __name__ == "__main__":` block. It is invoked automatically on normal app bootstrap from `server.py:497` (inside the `dummy-data` CLI command) and from `tests/conftest.py:39` (during pytest setup), so it may or may not exist in your local database depending on which entry points you have run. It is infrastructure seed data, not user data, so it is left in place.

---

## Appendix — Code References

Every behavioural claim made in this document maps to a specific file and line in the `app_2cd6ee777f8c` source tree. This appendix consolidates every reference into a single grep-friendly list so the reader can verify any statement against the code in isolation.

### Configuration & Logging

- `app/config.py:9` — `from dotenv import load_dotenv`
- `app/config.py:64-71` — `if config_file: load_dotenv(get_abs_path(config_file)) else: load_dotenv()` — reads `.env` at import time; the two branches are mutually exclusive, exactly one runs per process
- `app/config.py:73` — `COLOR_LOG = "COLOR_LOG" in os.environ`
- `app/config.py:79` — `URL = os.environ["URL"]`
- `app/config.py:80` — `print(">>> URL:", URL)` — startup URL log line
- `app/config.py:91` — `NOT_SEND_EMAIL = "NOT_SEND_EMAIL" in os.environ`
- `app/config.py:138` — `DISABLE_REGISTRATION = "DISABLE_REGISTRATION" in os.environ`
- `app/config.py:157-161` — `ALIAS_DOMAINS` list (includes `EMAIL_DOMAIN`)
- `app/config.py:164-165` — `PREMIUM_ALIAS_DOMAINS`
- `app/config.py:168` — `FIRST_ALIAS_DOMAIN`
- `app/config.py:301-304` — `JOB_ONBOARDING_1` / `_2` / `_3` / `_4` string constants
- `app/config.py:401` — `DISABLE_ONBOARDING = "DISABLE_ONBOARDING" in os.environ`
- `app/config.py:568` — `MEM_STORE_URI` (Redis URI)
- `app/config.py:612` — `EVENT_WEBHOOK`
- `app/config.py:616` — `EVENT_WEBHOOK_DISABLE`
- `app/log.py:5` — `import coloredlogs`
- `app/log.py:48` — `def _get_logger(name)`
- `app/log.py:62` — `coloredlogs.install(level="DEBUG", ...)` — attaches coloured handler when `COLOR_LOG` is set
- `app/log.py:67` — `print(">>> init logging <<<")` — startup logger banner
- `app/log.py:79` — `LOG = _get_logger("SL")` — the module-level logger imported everywhere

### Application Factory & Middleware

- `wsgi.py:1-3` — `from server import create_app; app = create_app()`
- `server.py:111-114` — Sentry SDK init gated on `SENTRY_DSN`
- `server.py:127` — `create_light_app()` — lightweight app used by `job_runner.py` and `cron.py`
- `server.py:139` — `create_app()` — main Flask factory
- `server.py:142` — `app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_host=1)`
- `server.py:159` — `app.config["SESSION_COOKIE_NAME"] = ...`
- `server.py:163-165` — conditional `initialize_redis_services(app, MEM_STORE_URI)` when `MEM_STORE_URI` is set
- `server.py:167` — `limiter.init_app(app)` (Flask-Limiter)
- `server.py:169` — `setup_error_page(app)`
- `server.py:171` — `init_extensions(app)` — wires up Flask-Login
- `server.py:172` — `register_blueprints(app)`
- `server.py:173` — `set_index_page(app)`
- `server.py:176-177` — `setup_favicon_route(app)`, `setup_openid_metadata(app)`
- `server.py:179` — `init_admin(app)` — Flask-Admin at `/admin`
- `server.py:183` — `register_custom_commands(app)` — adds CLI commands
- `server.py:200` — `CORS(app, resources={r"/api/*": {"origins": "*"}})`
- `server.py:204-206` — `@app.before_request make_session_permanent()`
- `server.py:213-215` — `/health` route → `return "success", 200`
- `server.py:233-246` — `register_blueprints()`: `auth_bp`, `monitor_bp`, `dashboard_bp`, `developer_bp`, `phone_bp`, `oauth_bp` (×2 — `/oauth` and `/oauth2`), `onboarding_bp`, `discover_bp`, `internal_bp`, `api_bp` → **10 distinct blueprints, 11 `register_blueprint` calls**
- `server.py:249-254` — `set_index_page()` — `/` route: if authenticated → `dashboard.index`, else → `auth.login` (302)
- `server.py:257-258` — `@app.before_request` — captures `g.start_time`, extracts `slref` cookie for referrals; skipped for `/static`, `/admin/static`, `/_debug_toolbar`
- `server.py:272-297` — `@app.after_request` — DEBUG log `"%s %s %s %s %s, takes %s"` and `newrelic.agent.record_custom_event("HttpResponseStatus", {"code": res.status_code})`
- `server.py:299` — `setup_openid_metadata()` registers `/.well-known/openid-configuration`
- `server.py:437` — `init_extensions()`
- `server.py:441` — `init_admin()`
- `server.py:572` — `local_main()` — sets `COLOR_LOG=True`, instantiates app, attaches Flask-DebugToolbar
- `server.py:588` — `app.run(debug=True, port=7777)`
- `server.py:598-599` — `if __name__ == "__main__": local_main()`

### Authentication Views

- `app/auth/base.py` — defines `auth_bp = Blueprint("auth", __name__, url_prefix="/auth")`; the `/auth` prefix is baked into the blueprint itself, **not** passed at `register_blueprint()` time in `server.py`
- `app/auth/views/register.py:23` — `class RegisterForm(FlaskForm)` — WTForms definition
- `app/auth/views/register.py:24` — `email = StringField(..., validators=[validators.DataRequired()])` — **only** `DataRequired()` is declared at the form level; email-shape validation happens in the handler via `is_valid_email()` and `email_can_be_used_as_mailbox()`
- `app/auth/views/register.py:25` — `password = StringField("Password", validators=[validators.DataRequired(), validators.Length(min=8, max=100)])` — note the form declares `StringField` (not `PasswordField`); the HTML `type="password"` comes from the template layer (`templates/auth/register.html`), not from the form class
- `app/auth/views/register.py:31` — `@auth_bp.route("/register", methods=["GET","POST"])`
- `app/auth/views/register.py:33` — `if current_user.is_authenticated:` authenticated-guard redirect
- `app/auth/views/register.py:38` — `if config.DISABLE_REGISTRATION:` disabled-registration guard
- `app/auth/views/register.py:47-70` — hCaptcha verification block
- `app/auth/views/register.py:65` — `RegisterEvent(ActionType.catpcha_failed).send()` on captcha failure
- `app/auth/views/register.py:73` — `email = canonicalize_email(email)`
- `app/auth/views/register.py:76, 83` — `RegisterEvent(ActionType.email_in_use).send()`
- `app/auth/views/register.py:85` — `LOG.d("create user %s", email)` — emits `create user <email>` log
- `app/auth/views/register.py:86-91` — `User.create(...)` call
- `app/auth/views/register.py:95` — `send_activation_email(user, next_url)` in `try/except`
- `app/auth/views/register.py:96` — `RegisterEvent(ActionType.success).send()`
- `app/auth/views/register.py:97` — `DailyMetric.get_or_create_today_metric().nb_new_web_non_proton_user += 1`
- `app/auth/views/register.py:101` — `RegisterEvent(ActionType.invalid_email).send()` on email-dispatch failure
- `app/auth/views/register.py:104` — `render_template("auth/register_waiting_activation.html")` — **no** keyword arguments are passed; the template renders from a static context
- `app/auth/views/register.py:117` — `def send_activation_email(user, next_url)`: deletes prior `ActivationCode` rows, creates new one with `random_string(30)`, builds `{URL}/auth/activate?code=...` link, calls `email_utils.send_activation_email`
- `app/auth/views/activate.py:13` — `@auth_bp.route("/activate")`
- `app/auth/views/activate.py:14-15` — `@limiter.limit("10/minute", deduct_when=...)`
- `app/auth/views/activate.py:26` — `activation_code = ActivationCode.get_by(code=code)`
- `app/auth/views/activate.py:49-54` — success path: `user.activated = True` (line 49); `login_user(user)` (line 50); `ActivationCode.delete(activation_code.id)` (line 53); `Session.commit()` (line 54)
- `app/auth/views/activate.py:58` — `email_utils.send_welcome_email(user)`
- `app/auth/views/activate.py:63` — `LOG.d("redirect user to %s", next_url)`
- `app/auth/views/activate.py:66` — `LOG.d("redirect user to dashboard")` — emits `redirect user to dashboard` log
- `app/auth/views/login.py:21` — `@auth_bp.route("/login", methods=["GET","POST"])`
- `app/auth/views/login.py:50` — `LoginEvent(ActionType.failed).send()` — wrong password or missing user
- `app/auth/views/login.py:56` — `LoginEvent(ActionType.disabled_login).send()`
- `app/auth/views/login.py:62` — `LoginEvent(ActionType.scheduled_to_be_deleted).send()`
- `app/auth/views/login.py:69` — `LoginEvent(ActionType.not_activated).send()`
- `app/auth/views/login.py:71-72` — `LoginEvent(ActionType.success).send()` → `return after_login(user, next_url)`
- `app/auth/views/login_utils.py:12` — `def after_login(user, next_url)` — MFA decision tree
- `app/auth/views/login_utils.py:35` — `LOG.d("log user %s in", user)` — emits `log user <User ...> in` log
- `app/auth/views/login_utils.py:41` — `LOG.d("redirect user to %s", next_url)`
- `app/auth/views/login_utils.py:44` — `LOG.d("redirect user to dashboard")` — emits `redirect user to dashboard` log

### Dashboard Views

- `app/dashboard/base.py` — defines `dashboard_bp = Blueprint("dashboard", __name__, url_prefix="/dashboard")`; the `/dashboard` prefix is baked into the blueprint itself, **not** passed at `register_blueprint()` time in `server.py`
- `app/dashboard/views/index.py:25` — `class Stats` — small dataclass of alias counters returned by `get_stats`
- `app/dashboard/views/index.py:32` — `def get_stats(user) -> Stats` — computes `nb_alias`, `nb_forward`, `nb_reply`, `nb_block`
- `app/dashboard/views/index.py:55` — `@dashboard_bp.route("/", methods=["GET","POST"])` `@login_required`
- `app/dashboard/views/index.py:67` — `def index()` — handler body
- `app/dashboard/views/index.py:172` — `LOG.d("Show intro to %s", current_user)` — emits `Show intro to <User ...>` log (one-shot per account because `intro_shown` is persisted)

### Models

- `app/models.py:336` — `class User(Base, ModelMixin, UserMixin, PasswordOracle)`
- `app/models.py:408` — `default_mailbox_id` column — declared **without** an `ondelete=` clause on its `ForeignKey`, so the database falls back to `NO ACTION`; the cleanup SQL in §4 must null this column explicitly before deleting the mailbox
- `app/models.py:447-451` — `newsletter_alias_id` column — declared with `ondelete="SET NULL"`; when the alias is deleted the FK is nulled automatically by PostgreSQL
- `app/models.py:602` — `User.create(...)` classmethod — full registration orchestration
- `app/models.py:611` — `Mailbox.create(user_id=user.id, email=user.email, verified=True)` — no `commit=True` kwarg; the commit happens later in `User.create()`
- `app/models.py:613` — `user.default_mailbox_id = mb.id`
- `app/models.py:616` — `if "alternative_id" not in kwargs:` — conditional assignment: `user.alternative_id = str(uuid.uuid4())` is applied **only** when the caller has not already supplied an explicit `alternative_id` kwarg
- `app/models.py:634` — `alias = Alias.create_new(user, prefix="simplelogin-newsletter", mailbox_id=mb.id, note=...)` — creates the first newsletter alias
- `app/models.py:643` — `user.newsletter_alias_id = alias.id`
- `app/models.py:646` — `if config.DISABLE_ONBOARDING:` — early return, **no onboarding jobs** when flag set
- `app/models.py:647` — `LOG.d("Disable onboarding emails")`
- `app/models.py:651-655` — `Job.create(name=config.JOB_ONBOARDING_1, payload={"user_id": user.id}, run_at=arrow.now().shift(days=1))` — no `commit=True` kwarg (the surrounding `Session.commit()` at the end of `User.create()` is what persists everything atomically)
- `app/models.py:656-660` — `Job.create(name=config.JOB_ONBOARDING_2, ..., run_at=now + 2d)` — no `commit=True`
- `app/models.py:661-665` — `Job.create(name=config.JOB_ONBOARDING_4, ..., run_at=now + 3d)` — no `commit=True`
- `app/models.py:671` — `User.delete(obj_id)` classmethod — canonical user-deletion path
- `app/models.py:1041` — `User.get_communication_email()` — returns newsletter alias for activated/notifying users, else `user.email`
- `app/models.py:1182-1183` — `User.__repr__` → `f"<User {self.id} {self.name} {self.email}>"`
- `app/models.py:1186-1187` — `def _expiration_1h(): return arrow.now().shift(hours=1)` — regular function (not a lambda), used as the default factory for `ActivationCode.expired`
- `app/models.py:1202` — `class ActivationCode`
- `app/models.py:1212` — `expired = sa.Column(ArrowType, default=_expiration_1h, ...)`
- `app/models.py:1214-1215` — `def is_expired(self) -> bool: return self.expired < arrow.now()`
- `app/models.py:3116` — `class SLDomain(Base, ModelMixin)` — the SQLAlchemy model class
- `app/models.py:3119` — `__tablename__ = "public_domain"` — **the Python class is `SLDomain` but the underlying table name is `public_domain`**; every SQL example in this document uses the correct on-disk name
- `app/models.py:3262` — `class DailyMetric`
- `app/models.py:3280` — `DailyMetric.get_or_create_today_metric()` static method — upserts today's row

### Email Subsystem

- `app/email_utils.py:97` — `send_welcome_email(user)` — subject `"Welcome to SimpleLogin"`, recipient `user.get_communication_email()`
- `app/email_utils.py:125` — `send_activation_email(user, activation_link)` — subject `"Just one more step to join SimpleLogin"`, recipient `user.email`
- `app/mail_sender.py:126` — `def send(self, send_request: SendRequest, retries: int = 2) -> bool`
- `app/mail_sender.py:130` — `if config.NOT_SEND_EMAIL:` — short-circuit gate
- `app/mail_sender.py:131-136` — `LOG.d("send email with subject '%s', from '%s' to '%s'", ...)` — emits the `send email with subject ...` DEBUG line
- `app/mail_sender.py:137` — `return True` — pretends delivery succeeded in local dev

### Events

- `app/events/event_dispatcher.py:14` — `NOTIFICATION_CHANNEL = "simplelogin_sync_events"`
- `app/events/event_dispatcher.py:24-26` — `PostgresDispatcher.send()` inserts `sync_event` row and executes `NOTIFY simplelogin_sync_events, '<id>'`
- `app/events/event_dispatcher.py:48` — `EventDispatcher.send_event(user, content, dispatcher, skip_if_webhook_missing=True)`
- `app/events/event_dispatcher.py:57-58` — `if config.EVENT_WEBHOOK_DISABLE: LOG.i("Not sending events because webhook is disabled"); return`
- `app/events/event_dispatcher.py:61-64` — `if not config.EVENT_WEBHOOK and skip_if_webhook_missing: LOG.i("Not sending events because webhook is not configured and allowed to be empty"); return`
- `app/events/event_dispatcher.py:67-69` — `if partner_user is None: LOG.i(f"Not sending events because there's no partner user for user {user}"); return`
- `app/events/auth_event.py` — top: `LoginEvent` and `RegisterEvent` classes with `ActionType` and `Source` enums
- `app/events/auth_event.py:23-25` — `LoginEvent.send()` → `newrelic.agent.record_custom_event("LoginEvent", {"action": self.action.name, "source": self.source.name})` — emits the enum member **name** (human-readable string like `"success"` / `"catpcha_failed"` / `"web"`), **not** `.value`
- `app/events/auth_event.py:45-47` — `RegisterEvent.send()` → `newrelic.agent.record_custom_event("RegisterEvent", {"action": self.action.name, "source": self.source.name})` — same `.name` convention

### Background Processes

- `job_runner.py:27` — `def onboarding_send_from_alias(user)` — onboarding-1 handler body
- `job_runner.py:48` — `def onboarding_pgp(user)` — onboarding-4 handler body (PGP-encryption promo)
- `job_runner.py:65` — `def onboarding_browser_extension(user)` — browser-extension promo helper
- `job_runner.py:90` — `def onboarding_mailbox(user)` — onboarding-2 handler body (multiple-mailbox promo)
- `job_runner.py:107` — `def welcome_proton(user)` — welcome helper for Proton-partner users
- `job_runner.py:130` — `def delete_mailbox_job(job)` — processor for mailbox-deletion jobs
- `job_runner.py:188` — `def process_job(job)` — the dispatcher `if job.name == config.JOB_ONBOARDING_1: ...`
- `job_runner.py:196` — `LOG.d("send onboarding send-from-alias email to user %s", user)`
- `job_runner.py:307-327` — `def get_jobs_to_run()` — state/attempts/run_at filter query
- `job_runner.py:329` — `if __name__ == "__main__":`
- `job_runner.py:334` — `LOG.d("Take job %s", job)`
- `job_runner.py:347` — `time.sleep(10)` — 10-second poll interval
- `cron.py:75` — `def notify_trial_end()`
- `cron.py:92` — `def delete_logs()`
- `cron.py:136` — `def delete_refused_emails()`
- `cron.py:156` — `def notify_premium_end()`
- `cron.py:188` — `def notify_manual_sub_end()`
- `cron.py:287` — `def poll_apple_subscription()`
- `cron.py:304` — `def compute_metric2()`
- `cron.py:401` — `def bounce_report()`
- `cron.py:488` — `def alias_creation_report()`
- `cron.py:537` — `def stats()`
- `cron.py:724` — `def sanity_check()`
- `cron.py:797` — `def check_mailbox_valid_domain()`
- `cron.py:864` — `def check_mailbox_valid_pgp_keys()`
- `cron.py:902` — `def check_custom_domain()`
- `cron.py:954` — `def delete_old_monitoring()`
- `cron.py:964` — `def delete_expired_tokens()`
- `cron.py:1171` — `def notify_hibp()`
- `cron.py:1223` — `def clear_users_scheduled_to_be_deleted(dry_run=False)`
- `cron.py:1245` — `def delete_old_data()`
- `cron.py:1252` — `def clear_alias_audit_log()`
- `cron.py:1257` — `def clear_user_audit_log()`
- `crontab.yml` — yacron primary-host schedule: **15** jobs (`stats`, `delete_old_monitoring`, `check_custom_domain`, `check_hibp`, `notify_hibp`, `delete_logs`, `delete_old_data`, `poll_apple_subscription`, `notify_trial_end`, `notify_manual_subscription_end`, `notify_premium_end`, `delete_scheduled_users`, `send_undelivered_mails`, `clear_alias_audit_log`, `clear_user_audit_log`)
- `crontab-all-hosts.yml` — yacron multi-host schedule; in this repo it holds a single job (`send_undelivered_mails`, `*/5 * * * *`, `concurrencyPolicy: Forbid`) so every host re-tries its own local filesystem dead-letter queue
- `monitoring.py:39` — `def log_postfix_metrics()` — queue depth + process counts
- `monitoring.py:44` — `LOG.d("postfix queue sizes %s %s %s", ...)`
- `monitoring.py:157-171` — `if __name__ == "__main__":` — the 60-second tick loop
- `monitoring.py:171` — `sleep(60)`
- `event_listener.py` — PostgreSQL LISTEN/NOTIFY consumer; `argparse` sub-parsers expose four positional sub-commands: `listener`, `dead_letter`, `debug`, `run`. Invocation is `python event_listener.py listener` — there is **no** `--mode` flag.
- `email_handler.py` — aiosmtpd-based inbound mail handler (forward + reply phases); default listening port is **20381** (the `default=20381` value on the `--port` argparse argument)

### Infrastructure & Entrypoints

- `wsgi.py` — 3-line WSGI entry point exporting `app`
- `Dockerfile` — two-stage build: `node:10.17.0-alpine` (Stage 1 — `npm ci` for `static/`) → `python:3.10` (Stage 2 — Poetry install). `EXPOSE 7777`. `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]`.
- `alembic.ini` — Alembic configuration pointing at `migrations/`
- `migrations/versions/` — 256 migration revisions (as of `app_2cd6ee777f8c`)
- `init_app.py:13` — `load_pgp_public_keys()` — loads keys into the GnuPG keyring
- `init_app.py:39-56` — `add_sl_domains()` — seeds the `public_domain` table (Python class `SLDomain`, on-disk table name `public_domain`) via `SLDomain.create(...)`; idempotent (logs `"%s is already a SL domain"` DEBUG or `"Add %s to SL domain"` INFO)
- `init_app.py:59` — `add_proton_partner()` — defined here, but **not** invoked from `init_app.py`'s `__main__`. It runs on normal app bootstrap from `server.py:497` (inside the `dummy-data` CLI command) and from `tests/conftest.py:39`.
- `init_app.py:69-73` — `if __name__ == "__main__":` — calls, in order, `load_pgp_public_keys()` **first**, then `add_sl_domains()`. That is the complete set — `add_proton_partner()` is **not** called from this block.
- `pyproject.toml` — Poetry manifest. Key pins: `python ^3.10`, `flask ^1.1.2`, `flask_login ^0.5.0`, `gunicorn ^20.0.4`, `SQLAlchemy 1.3.24`, `psycopg2-binary ^2.9.3`, `redis ^4.5.3`, `bcrypt ^3.2.0`, `python-dotenv ^0.14.0`, `sentry_sdk ^2.16.0`, `newrelic 8.8.0`, `aiosmtpd ^1.2`, `yacron ^0.11.1`, `flask-debugtoolbar ^0.11.0`, `flask_admin ^1.5.6`, `flask-cors ^3.0.9`.
- `example.env:16` — `COLOR_LOG=true` (commented by default)
- `example.env:19` — `NOT_SEND_EMAIL=true`
- `example.env:22` — `EMAIL_DOMAIN=sl.local`
- `example.env:150` — `DISABLE_ONBOARDING=true` with comment `# For self-hosted instance` — **default for self-hosted** operators; suppresses onboarding job scheduling

### Templates

- `templates/auth/register.html` — registration form UI
- `templates/auth/register_waiting_activation.html` — post-registration waiting page (rendered after `POST /auth/register` succeeds)
- `templates/auth/activate.html` — activation page (rarely seen; most activation flows redirect immediately after the GET)
- `templates/auth/login.html` — login form UI
- `templates/dashboard/index.html` — main dashboard; wraps `introJs().start()` in `{% if show_intro %}` and a `window.innerWidth >= 1024` JS guard

---

## Key Insights Recap

The following eight insights were surfaced during the investigation and are worth calling out explicitly because they are common sources of surprise:

1. **`DISABLE_ONBOARDING=true` is the default in `example.env`.** With the shipped `example.env` copied verbatim to `.env`, the three onboarding jobs are **not** scheduled during registration. To observe them, delete or comment out the line at `example.env:150` (see `app/models.py:646`).
2. **Mail-sender log format is verbose.** The actual format string at `app/mail_sender.py:131-136` is `send email with subject '%s', from '%s' to '%s'` — not the shortened "send email to X, subject 'Y'" some notes use.
3. **EventDispatcher log message is verbose.** The full literal emitted at `app/events/event_dispatcher.py:63` is `"Not sending events because webhook is not configured and allowed to be empty"` — do not paraphrase.
4. **The SimpleLogin-managed-domain table is `public_domain`, not `sl_domain`.** The Python **class** is `SLDomain` (defined at `app/models.py:3116`), but its `__tablename__` is `"public_domain"` (`app/models.py:3119`) — so every `psql` query and every FK reference (e.g. `users.default_alias_public_domain_id` → `public_domain.id` at `app/models.py:383`) uses `public_domain`. `init_app.py:39-56`'s `add_sl_domains()` inserts into `public_domain`.
5. **Gunicorn vs Flask dev server produce different startup logs.** Under `gunicorn wsgi:app` you see `[INFO] Starting gunicorn 20.1.0`, `[INFO] Listening at: http://0.0.0.0:7777`, and `[INFO] Booting worker with pid: ...`. Under `python server.py → local_main()` you see `* Serving Flask app "server" (lazy loading)` and `* Debug mode: on`. The `>>> URL:` and `>>> init logging <<<` banners appear in **both** cases because they fire at import time in `app/config.py:80` and `app/log.py:67`.
6. **Welcome-email recipient is the newsletter alias, not the registration email.** `user.get_communication_email()` at `app/models.py:1041` returns `simplelogin-newsletter.<random>@sl.local` for an activated, opted-in user — so the welcome-email log line shows the alias as the recipient.
7. **`oauth_bp` is double-registered** at `/oauth` and `/oauth2` (`server.py:240-241`). The blueprint count is 10 distinct blueprints but 11 `register_blueprint` calls.
8. **`Show intro to ...` fires exactly once per account.** After the first dashboard visit, `current_user.intro_shown` is persisted `True`, so this log line is a reliable single-shot "first-visit" signal — great for verifying that an end-to-end test actually reached the dashboard as opposed to just the login page.

---

*End of document.*
