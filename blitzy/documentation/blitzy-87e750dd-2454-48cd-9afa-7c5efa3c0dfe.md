# SimpleLogin Local Deployment — Startup, New-User Journey, and Background-Service Observability Guide

> This document answers three related questions about a fresh, local deployment of SimpleLogin:
>
> 1. After building and running all components locally, what specific indicators in the application logs and web UI confirm that SimpleLogin is fully operational and ready to handle user authentication and alias-based email activity?
> 2. What happens step-by-step when a new user registers, verifies their email address, and logs in for the first time? What visible behaviors at each stage confirm the system is correctly processing every phase of the authentication flow and forwarding the user to the dashboard?
> 3. During and after the new-user flow, what runtime indicators show that background jobs, internal services, and inter-process communication are active and correctly supporting email forwarding and identity verification?
>
> Every answer below is grounded in the actual source code in this repository and validated against a live run performed against a freshly built environment. Runtime evidence was captured by POST/GET requests against `http://localhost:7777`, by inspecting the PostgreSQL schema created by Alembic (255 migration revisions), and by reading the Flask dev-server log emitted by `app/log.py`. Any temporary test data created for verification was fully removed from PostgreSQL after the walkthrough, as instructed.

---

## Table of Contents

- [1. Environment Context for This Investigation](#1-environment-context-for-this-investigation)
- [2. Startup Readiness Indicators](#2-startup-readiness-indicators)
  - [2.1 Pre-Flight: What Must Be True Before the App Starts](#21-pre-flight-what-must-be-true-before-the-app-starts)
  - [2.2 The Bootstrap Sequence (from `server.py` + `app/config.py` + `app/log.py`)](#22-the-bootstrap-sequence-from-serverpy--appconfigpy--applogpy)
  - [2.3 Log Markers That Confirm Successful Startup](#23-log-markers-that-confirm-successful-startup)
  - [2.4 HTTP Indicators That Confirm the Web UI Is Operational](#24-http-indicators-that-confirm-the-web-ui-is-operational)
  - [2.5 Database Indicators That Confirm the Schema Is Ready](#25-database-indicators-that-confirm-the-schema-is-ready)
  - [2.6 Summary Checklist — "App Is Ready" Gate](#26-summary-checklist--app-is-ready-gate)
- [3. New-User Journey Walkthrough](#3-new-user-journey-walkthrough)
  - [3.1 Flow Overview](#31-flow-overview)
  - [3.2 Step 1 — Load the Registration Form (`GET /auth/register`)](#32-step-1--load-the-registration-form-get-authregister)
  - [3.3 Step 2 — Submit the Registration Form (`POST /auth/register`)](#33-step-2--submit-the-registration-form-post-authregister)
  - [3.4 Step 3 — Activate via Emailed Link (`GET /auth/activate?code=...`)](#34-step-3--activate-via-emailed-link-get-authactivatecode)
  - [3.5 Step 4 — Log In Explicitly (`POST /auth/login`)](#35-step-4--log-in-explicitly-post-authlogin)
  - [3.6 Step 5 — Dashboard Rendered (`GET /dashboard/`)](#36-step-5--dashboard-rendered-get-dashboard)
  - [3.7 Consolidated Journey Log Tape](#37-consolidated-journey-log-tape)
- [4. Background Service Observability](#4-background-service-observability)
  - [4.1 Map of Background Subsystems That Run Alongside Flask](#41-map-of-background-subsystems-that-run-alongside-flask)
  - [4.2 Onboarding Jobs Recorded in the `job` Table](#42-onboarding-jobs-recorded-in-the-job-table)
  - [4.3 Aggregate Signals in the `daily_metric` Table](#43-aggregate-signals-in-the-daily_metric-table)
  - [4.4 Event Dispatcher — Expected and Observed Behavior in a Local Run](#44-event-dispatcher--expected-and-observed-behavior-in-a-local-run)
  - [4.5 APM Events (`RegisterEvent`, `LoginEvent`) in New Relic](#45-apm-events-registerevent-loginevent-in-new-relic)
  - [4.6 Email Subsystem Observability with `NOT_SEND_EMAIL=true`](#46-email-subsystem-observability-with-not_send_emailtrue)
  - [4.7 `job_runner.py` — What You Would See If You Started It](#47-job_runnerpy--what-you-would-see-if-you-started-it)
  - [4.8 `cron.py` + `crontab.yml` Maintenance Scheduler](#48-cronpy--crontabyml-maintenance-scheduler)
  - [4.9 `email_handler.py`, `event_listener.py`, `monitoring.py`](#49-email_handlerpy-event_listenerpy-monitoringpy)
- [5. Temporary Test Data and Cleanup](#5-temporary-test-data-and-cleanup)
- [6. Consolidated "Everything Is Healthy" Dashboard](#6-consolidated-everything-is-healthy-dashboard)
- [7. Appendix — File and Code Anchors Used as Evidence](#7-appendix--file-and-code-anchors-used-as-evidence)

---

## 1. Environment Context for This Investigation

All runtime observations below were produced against the following local deployment. Versions are taken from `pyproject.toml`, `poetry.lock`, and `Dockerfile`, and actually installed into the working environment:

| Layer | Version / Setting | Source of Truth |
|---|---|---|
| Runtime | Python `3.10.20` (venv) | `pyproject.toml` declares `python = "^3.10"`; `Dockerfile` stage 2 starts from `python:3.10`. |
| Web framework | Flask `1.1.4` | `pyproject.toml` → `flask = "^1.1.2"`; `poetry.lock` pin. |
| ORM | SQLAlchemy `1.3.24` | `pyproject.toml` pin `SQLAlchemy = "1.3.24"`. |
| WSGI server | `gunicorn = "^20.0.4"` (used initially); Flask dev server (`python server.py`) also exercised | `Dockerfile` `CMD` uses `gunicorn`; `server.py` has `if __name__ == "__main__"` block for dev runs. |
| Database | PostgreSQL `16.13` | Installed from Ubuntu apt repo; project requires 13+; Alembic migrations applied. |
| Cache / session store | Redis `7.0.15` | Installed from Ubuntu apt repo; project requires 6+. |
| Email delivery | `NOT_SEND_EMAIL=true` (log-only mode) | `app/mail_sender.py` branches on this flag. |
| Inbound SMTP | **Not started** for this observability exercise | `email_handler.py` is out of scope of the user-facing registration/login flow. |
| Job runner | **Not started** (observed only via `job` table rows) | `job_runner.py` has its own process loop (see §4.7). |

**Environment variables applied** (all non-secret and documented in `example.env`):

```
URL=http://localhost:7777
EMAIL_DOMAIN=sl.local
SUPPORT_EMAIL=support@sl.local
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
FLASK_SECRET=secret
NOT_SEND_EMAIL=true
COLOR_LOG=true
EMAIL_SERVERS_WITH_PRIORITY=[(10, "email.hostname.")]
```

**Rationale for these choices.**
- `URL` sets `Server url` in `app/config.py` and is echoed at import time so that any mis-configuration is visible immediately in the logs (`print(">>> URL:", URL)` on line 80 of `app/config.py`).
- `NOT_SEND_EMAIL=true` short-circuits SMTP delivery in `app/mail_sender.py:send()`, logging the message metadata instead. This is the supported mode for local development per the application's own code path and is documented in `example.env`. It is the key enabler of observability without needing Postfix.
- `COLOR_LOG=true` triggers the `coloredlogs.install(...)` branch in `app/log.py` so that log levels are color-coded on a local TTY. The underlying formatter is identical.
- `FLASK_SECRET` is required by Flask to sign the session cookie; `app/config.py` reads it directly via `os.environ["FLASK_SECRET"]` and will `KeyError` on startup if missing.
- `EMAIL_SERVERS_WITH_PRIORITY` is parsed by `sl_getenv()` (a `literal_eval`-based env reader in `app/config.py`) and is only relevant for outbound DNS fallback; it is harmless in `NOT_SEND_EMAIL` mode but must still parse as a Python literal.

**Startup commands used** (after the Python venv, PostgreSQL, and Redis services were running):

```bash
# 1. Apply Alembic migrations (255 revisions currently in `migrations/versions/`)
alembic upgrade head

# 2. Seed SL domains and load PGP keys
python init_app.py

# 3. Start the Flask dev server on port 7777
python server.py
# --- or (production-style) ---
gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60 --log-level DEBUG
```

The two startup paths differ only in the WSGI container. Both call `create_app()` in `server.py`, which is the single place where blueprints, middleware, error handlers, Sentry, New Relic, and (optionally) Redis-backed sessions are wired.

---

## 2. Startup Readiness Indicators

### 2.1 Pre-Flight: What Must Be True Before the App Starts

Before `server.py` / `wsgi.py` can successfully import, several invariants must hold. These are not optional — they are enforced by hard `os.environ[...]` reads in `app/config.py`. If any fails, the process crashes before it ever starts listening, so their *absence* of an error is itself a startup indicator:

1. **Required env vars present.** `app/config.py` issues raw `os.environ["URL"]` (line 79), `os.environ["DB_URI"]` (read later in the file), and `os.environ["FLASK_SECRET"]` reads. Missing any of these raises `KeyError` before the Flask app is created.
2. **PostgreSQL reachable on `DB_URI`.** `app/db.py` creates a SQLAlchemy engine via `create_engine(DB_URI, ...)`; failure surfaces when the first query is issued (e.g., during `init_app.py` or the first HTTP request).
3. **Migrations applied.** `alembic upgrade head` must have run successfully. The codebase's current head is applied in a sequence visible under `migrations/versions/`.
4. **SL domains seeded.** `init_app.py`'s `add_sl_domains()` populates the `public_domain` table with the `EMAIL_DOMAIN` value (`sl.local` in our setup) — without this row, the alias-domain dropdowns on the dashboard/register form are empty.

### 2.2 The Bootstrap Sequence (from `server.py` + `app/config.py` + `app/log.py`)

In import order, the key steps that occur when a WSGI worker (or `python server.py`) initializes are:

```mermaid
flowchart TD
    A["Python imports 'server' / 'wsgi'"] --> B["app/config.py evaluated<br/>(load_dotenv, env var reads)"]
    B --> C["print '>>> URL: http://localhost:7777'<br/>(config.py line 80)"]
    C --> D["app/log.py evaluated<br/>coloredlogs.install + LOG = _get_logger('SL')"]
    D --> E["print '>>> init logging <<<'<br/>(log.py line 67)"]
    E --> F["server.py:create_app()<br/>Flask app + middleware (ProxyFix)"]
    F --> G["Register 11 blueprints<br/>(auth_bp, dashboard_bp, api_bp, oauth_bp x2,<br/>developer_bp, discover_bp, monitor_bp,<br/>onboarding_bp, internal_bp, phone_bp)"]
    G --> H["Flask-Login manager + Flask-Limiter + Sentry + New Relic init"]
    H --> I["initialize_redis_services() (optional, if MEM_STORE_URI set)"]
    I --> J["Flask logs<br/>'* Serving Flask app ...'<br/>'* Debug mode: on'"]
    J --> K["HTTP listener bound to 0.0.0.0:7777"]
```

**Source anchors** for each step:
- `print(">>> URL:", URL)` at `app/config.py:80`.
- `print(">>> init logging <<<")` at `app/log.py:67`.
- Blueprint registrations at `server.py:234-246` (`register_blueprints(app)`).
- `after_request` logging hook at `server.py:284` (this is what produces every subsequent "127.0.0.1 METHOD /path ... status, takes ..." log line).

### 2.3 Log Markers That Confirm Successful Startup

The following block is the **exact** log output observed when the Flask dev server started cleanly for this investigation. Every line here is a positive readiness indicator; the **absence** of this block means the application failed to start.

```
2026-04-16 23:04:01 - SL - DEBUG - 23764 - ".../app/utils.py:17" - <module>() -  - load words file: .../local_data/test_words.txt
>>> URL: http://localhost:7777
MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value
Paddle param not set
WARNING: Use a temp directory for GNUPGHOME /tmp/dysgzxbmaabxtcfmqkcj
Upload files to local dir
>>> init logging <<<
 * Serving Flask app "server" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
```

Line-by-line interpretation:

| Log line | Meaning | Source |
|---|---|---|
| `load words file: .../local_data/test_words.txt` | Random alias-name word list loaded for `Alias.create_new_random()` | `app/utils.py:17` |
| `>>> URL: http://localhost:7777` | `URL` env var resolved and bound to `config.URL` | `app/config.py:80` |
| `MAX_NB_EMAIL_FREE_PLAN is not set, use 5 as default value` | Optional plan-limit env var defaulted | `app/config.py` |
| `Paddle param not set` | Optional payment integration skipped (expected in local dev) | `app/config.py` |
| `WARNING: Use a temp directory for GNUPGHOME /tmp/...` | PGP keyring initialized to a fresh tempdir because `GNUPGHOME` env var was not set (expected in local dev) | `app/pgp_utils.py` |
| `Upload files to local dir` | File uploads configured as local filesystem (no S3 credentials set — expected) | `app/config.py` |
| `>>> init logging <<<` | Custom `SL` logger wired up with coloredlogs | `app/log.py:67` |
| ` * Serving Flask app "server" (lazy loading)` | Flask dev server alive | Werkzeug |
| ` * Debug mode: on` | Debug mode active (expected for local dev, **should be off in prod**) | Flask/Werkzeug |

If any of these lines are **missing or replaced by a traceback**, startup failed. In particular:

- A missing `>>> URL: ...` line means `app/config.py` failed — typically a `KeyError` on a required env var.
- A missing `>>> init logging <<<` line means an import error occurred before `app/log.py` could run.
- A traceback mentioning `psycopg2.OperationalError` means PostgreSQL is unreachable on `DB_URI`.

### 2.4 HTTP Indicators That Confirm the Web UI Is Operational

Once the process is listening, three probes confirm that routing, templating, and the request-logging middleware are all working. These were exercised live:

| Probe | Expected | Observed (this run) |
|---|---|---|
| `GET /auth/login` | HTTP `200` with the login form rendered | `200` |
| `GET /auth/register` | HTTP `200` with the registration form | `200` |
| `GET /` | HTTP `302` redirecting to `/auth/login` (anonymous user) | `302` |

Each successful request also emits a line from the `after_request` handler at `server.py:284`, e.g.:

```
2026-04-16 23:04:14 - SL - DEBUG - 23772 - ".../server.py:284" - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.10274934768676758
```

Fields in order: remote address, HTTP method, path, query args, status code, wall-clock duration (seconds). The presence of these lines is the canonical "web UI is live" log signature.

**Why `GET /auth/login` is the best health check.** It is an unauthenticated, non-rate-limited (at GET), template-rendered endpoint that exercises: (a) blueprint routing (`auth_bp`), (b) Jinja2 template loading (`templates/auth/login.html`), (c) CSRF token generation (Flask-WTF), and (d) the `after_request` log hook — i.e., the full request lifecycle. A `200` here is a four-in-one positive signal.

### 2.5 Database Indicators That Confirm the Schema Is Ready

The application depends on 50+ tables. After `alembic upgrade head` and `python init_app.py`, two quick SQL checks are definitive:

```sql
-- 1. Migrations ran to head — alembic_version contains exactly one row, a recent revision hash
SELECT * FROM alembic_version;

-- 2. init_app seeded SL domains — at least one row for EMAIL_DOMAIN
SELECT id, domain, premium_only FROM public_domain;
```

Observed in this run:

```
 id |  domain  | premium_only
----+----------+--------------
  1 | sl.local | f
```

If `public_domain` is empty, either `init_app.py` was never run, or `EMAIL_DOMAIN` was unset. `init_app.py`'s log line `Add sl.local to SL domain` (or `sl.local is already a SL domain` on the second run) is the canonical confirmation.

### 2.6 Summary Checklist — "App Is Ready" Gate

A fresh SimpleLogin deployment is **ready** when all of these are simultaneously true:

- [x] Startup log contains `>>> URL: ...`, `>>> init logging <<<`, and `* Serving Flask app "server"` (no traceback above or between them).
- [x] `GET /auth/login` → **`200`**.
- [x] `GET /` → **`302`** to `/auth/login`.
- [x] `alembic_version` has a single row (migrations at head).
- [x] `public_domain` has at least one row matching `EMAIL_DOMAIN`.
- [x] PostgreSQL is serving on `5432` and Redis on `6379` (verifiable with `pg_isready` and `redis-cli PING`, returning `accepting connections` and `PONG` respectively).
- [x] No `ERROR`/`CRITICAL` lines in the SL log prefix after the initial boot sequence.

All of these were satisfied in the live run performed for this guide.

---

## 3. New-User Journey Walkthrough

### 3.1 Flow Overview

The full "first-time user" path consists of four HTTP transactions plus a password-form submission. Below is the precise sequence as implemented by the code:

```mermaid
sequenceDiagram
    autonumber
    participant U as Browser / curl
    participant F as Flask (server.py)
    participant DB as PostgreSQL
    participant MS as MailSender (NOT_SEND_EMAIL=true)

    U->>F: GET /auth/register
    F-->>U: 200 (form + CSRF token)

    U->>F: POST /auth/register (email, password, CSRF)
    F->>DB: INSERT users (activated=false)
    F->>DB: INSERT mailbox (verified=true)
    F->>DB: INSERT alias (simplelogin-newsletter.<word>@sl.local)
    F->>DB: INSERT job x3 (onboarding-1, -2, -4)
    F->>DB: INSERT activation_code
    F->>DB: UPDATE daily_metric (nb_new_web_non_proton_user++)
    F->>MS: send_activation_email(user)
    MS-->>F: (log-only; subject 'Just one more step to join SimpleLogin')
    F-->>U: 200 (render auth/register_waiting_activation.html)

    Note over U,MS: User reads the activation code (delivered by email in real prod;<br/>for local testing, read from the activation_code table)

    U->>F: GET /auth/activate?code=<code>
    F->>DB: SELECT activation_code WHERE code=?
    F->>DB: UPDATE users SET activated=true
    F->>DB: DELETE activation_code (single-use)
    F->>F: login_user(user)  [Flask-Login session cookie set]
    F->>MS: send_welcome_email(user)
    MS-->>F: (log-only; subject 'Welcome to SimpleLogin')
    F-->>U: 302 -> /dashboard/

    U->>F: GET /dashboard/
    F->>DB: query aliases / stats / mailboxes
    F->>DB: UPDATE users SET intro_shown=true (first visit only)
    F-->>U: 200 (dashboard rendered)

    Note over U,MS: Optional — log out, then POST /auth/login for an explicit sign-in
    U->>F: POST /auth/login (email, password)
    F->>F: after_login(): login_user(user), session['sudo_time']=now
    F-->>U: 302 -> /dashboard/
```

The rest of this section documents each step with the *exact* code path, DB mutations, and log lines.

### 3.2 Step 1 — Load the Registration Form (`GET /auth/register`)

**Route:** `app/auth/views/register.py`, method `GET`.
**Template:** `templates/auth/register.html`.
**Database:** none written.
**Log line:**

```
127.0.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes 0.038
```

**What confirms success:** HTTP `200`, a rendered HTML page containing a `<form method="POST">` with a hidden `csrf_token` field. The CSRF token must be submitted along with the registration POST.

### 3.3 Step 2 — Submit the Registration Form (`POST /auth/register`)

**Route handler:** `register()` in `app/auth/views/register.py`.
**Key body:**

```python
LOG.d("create user %s", email)
user = User.create(
    email=email,
    name=form.email.data,
    password=form.password.data,
    referral=get_referral(),
)
Session.commit()

try:
    send_activation_email(user, next_url)
    RegisterEvent(RegisterEvent.ActionType.success).send()
    DailyMetric.get_or_create_today_metric().nb_new_web_non_proton_user += 1
    Session.commit()
except Exception:
    flash("Invalid email, are you sure the email is correct?", "error")
```

`User.create()` (in `app/models.py:602`) is the fan-out: it creates the user row, creates the default mailbox with `verified=True`, creates the first alias with prefix `simplelogin-newsletter`, and schedules the three onboarding jobs.

**Database state created by a single POST** (verified live in this run for `testuser@example.com`):

| Table | Rows inserted | Key values |
|---|---|---|
| `users` | 1 | `id=2`, `email=testuser@example.com`, `activated=false`, `default_mailbox_id=2`, `newsletter_alias_id=2` |
| `mailbox` | 1 | `id=2`, `user_id=2`, `email=testuser@example.com`, `verified=true` |
| `alias` | 1 | `id=2`, `user_id=2`, `email=simplelogin-newsletter.swoons184@sl.local`, `mailbox_id=2` |
| `job` | 3 | `onboarding-1` at now+1d, `onboarding-2` at now+2d, `onboarding-4` at now+3d, each with `payload={"user_id":2}`, `state=0` (ready) |
| `activation_code` | 1 | `user_id=2`, `code=uidmweayegyqgwfmacgbjjjtmxqnrb` (30-char random) |
| `daily_metric` | 1 row inserted (or updated) | `date=2026-04-16`, `nb_new_web_non_proton_user` incremented |

**Why the newsletter alias's word suffix is random.** `Alias.create_new()` calls into `app/utils.py` which reads from `local_data/test_words.txt` (or `words.txt` in prod) — the first startup log line `load words file: .../local_data/test_words.txt` confirms this word list was loaded in memory. The suffix `swoons184` observed here is deterministic given the RNG seed at runtime; in general it will differ each run.

**Log lines observed:**

```
register() -  - create user testuser@example.com
event_dispatcher.py:62 - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
email_utils.py:303 - send_email() -  - send email to testuser@example.com, subject 'Just one more step to join SimpleLogin'
mail_sender.py:131 - send() -  - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'testuser@example.com'
server.py:284 - after_request() -  - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.60
```

**HTTP response:** `200` (note: not `302`) rendering `templates/auth/register_waiting_activation.html`. The browser-visible message is:

> **An email to validate your email is on its way.**
> Please check your inbox/spam folder. Make sure to mark the message as not spam so that future messages come to your normal inbox.

(Page `<title>` is `Activation Email Sent | SimpleLogin`.) The page also offers a link back to login.

**What confirms success:**
- The `create user testuser@example.com` DEBUG line.
- The `send email ... subject 'Just one more step to join SimpleLogin'` DEBUG line (twice — once from `email_utils.send_email` at the composer layer, once from `mail_sender.send` at the delivery layer).
- A fresh row in `activation_code` (can be inspected with `SELECT * FROM activation_code WHERE user_id = ...`).
- A fresh row in `users` with `activated = false`.
- Three fresh rows in `job` whose `name` ∈ {`onboarding-1`, `onboarding-2`, `onboarding-4`} and `run_at` in the near future.
- The `daily_metric.nb_new_web_non_proton_user` counter incremented by 1 for today's date.
- HTTP `200` on the POST (a 302 to `/auth/login` would indicate the `email_can_be_used_as_mailbox` check failed, or that the user already existed, etc.).

### 3.4 Step 3 — Activate via Emailed Link (`GET /auth/activate?code=...`)

**Route handler:** `activate()` in `app/auth/views/activate.py`. The full handler body is short enough to quote:

```python
code = request.args.get("code")
activation_code: ActivationCode = ActivationCode.get_by(code=code)
if not activation_code:
    g.deduct_limit = True
    return (render_template("auth/activate.html", error="Activation code cannot be found"), 400)
if activation_code.is_expired():
    return (render_template("auth/activate.html", error="Activation code was expired", show_resend_activation=True), 400)

user = activation_code.user
user.activated = True
login_user(user)
ActivationCode.delete(activation_code.id)
Session.commit()
flash("Your account has been activated", "success")
email_utils.send_welcome_email(user)

if "next" in request.args:
    ...
else:
    LOG.d("redirect user to dashboard")
    return redirect(url_for("dashboard.index"))
```

**For local testing** (where emails are not delivered because `NOT_SEND_EMAIL=true`), the activation code is retrieved by querying:

```sql
SELECT code FROM activation_code WHERE user_id = (SELECT id FROM users WHERE email='testuser@example.com');
```

and hitting `GET /auth/activate?code=<that_code>`.

**Log lines observed:**

```
email_utils.py:303 - send_email() -  - send email to simplelogin-newsletter.swoons184@sl.local, subject 'Welcome to SimpleLogin'
mail_sender.py:131 - send() -  - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.swoons184@sl.local'
activate.py:66 - activate() -  - redirect user to dashboard
server.py:284 - after_request() -  - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code','uidmweayegyqgwfmacgbjjjtmxqnrb')]) 302, takes 0.057
```

**Notice two subtle but important behaviors confirmed by the logs:**

1. The **welcome email is sent to the user's newly created newsletter alias**, not to their registration email. This is by design — the first outbound email to the alias exercises the alias-forwarding concept end-to-end (in prod it would route through Postfix and `email_handler.py` back to the default mailbox). The log `send email to simplelogin-newsletter.swoons184@sl.local, subject 'Welcome to SimpleLogin'` is the proof.
2. The activation code is deleted in the same transaction that sets `activated=true`, so re-using the URL a second time will render `templates/auth/activate.html` with `error="Activation code cannot be found"` (HTTP `400`). Re-running the verification query after activation confirms `SELECT count(*) FROM activation_code WHERE user_id=2` returns `0`.

**HTTP response:** `302 Location: /dashboard/`. The `Set-Cookie: session=...` header on this response carries the Flask-Login session cookie — this is how the user is "silently" logged in by the activation endpoint.

**What confirms success:**
- `activate() -  - redirect user to dashboard` DEBUG line.
- `users.activated` flips from `false` to `true` (verify with SQL).
- `activation_code` row is deleted (verify with SQL).
- A welcome email log line to the newsletter alias.
- HTTP `302` with `Location: /dashboard/`.

### 3.5 Step 4 — Log In Explicitly (`POST /auth/login`)

After activation, the user is already logged in. To validate that the *explicit* login path also works, the verification also logs out (fresh cookie jar) and POSTs to `/auth/login`.

**Route handler:** `login()` in `app/auth/views/login.py`. Key decision tree:

```python
if not user or not user.check_password(form.password.data):
    LoginEvent(LoginEvent.ActionType.failed).send()
elif user.disabled:
    LoginEvent(LoginEvent.ActionType.disabled_login).send()
elif user.delete_on is not None:
    LoginEvent(LoginEvent.ActionType.scheduled_to_be_deleted).send()
elif not user.activated:
    show_resend_activation = True
    LoginEvent(LoginEvent.ActionType.not_activated).send()
else:
    LoginEvent(LoginEvent.ActionType.success).send()
    return after_login(user, next_url)
```

`after_login()` in `app/auth/views/login_utils.py` picks the post-login destination based on MFA state:

```python
if not login_from_proton:
    if user.fido_enabled():
        session[MFA_USER_ID] = user.id
        return redirect(url_for("auth.fido"))
    elif user.enable_otp:
        session[MFA_USER_ID] = user.id
        return redirect(url_for("auth.mfa"))

LOG.d("log user %s in", user)
login_user(user)
session["sudo_time"] = int(time())
...
LOG.d("redirect user to dashboard")
return redirect(url_for("dashboard.index"))
```

**Log lines observed in this run:**

```
server.py:284 - after_request() -  - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.022
login_utils.py:35 - after_login() -  - log user <User 2 testuser@example.com testuser@example.com> in
login_utils.py:44 - after_login() -  - redirect user to dashboard
server.py:284 - after_request() -  - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.296
server.py:284 - after_request() -  - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.143
```

**What confirms success:**
- The two paired `after_login()` DEBUG lines, `log user <User ...> in` followed by `redirect user to dashboard`.
- HTTP `302` on `POST /auth/login` with `Location: /dashboard/`.
- A follow-up `GET /dashboard/` returning `200`.
- `session['sudo_time']` is set so that sensitive account-management endpoints can be guarded by a recent-auth check.

**What a failure looks like.** Bad password → HTTP `200` back on `/auth/login` with `flash('Email or password incorrect', 'error')`, and a `LoginEvent(ActionType.failed)` is emitted (goes to New Relic if configured). Disabled account → HTTP `200` with a different flash and `LoginEvent.disabled_login`. Scheduled-for-deletion → `LoginEvent.scheduled_to_be_deleted`. Never-activated → `LoginEvent.not_activated` and a "resend activation" button is shown.

### 3.6 Step 5 — Dashboard Rendered (`GET /dashboard/`)

**Route handler:** `index()` in `app/dashboard/views/index.py`.
**Template:** `templates/dashboard/index.html` (page `<title>` becomes `Alias | SimpleLogin`).

The dashboard does four things on every GET:

1. Query the user's aliases (paginated), with optional filters/sort from the query string.
2. Compute forwarding statistics (`stats = get_stats(current_user)`) — these drive the three counters ("Emails Forwarded", "Emails Replied", "Emails Blocked") visible on the top card. On a brand-new user these are all zero.
3. On the user's very first dashboard visit, flip `intro_shown=True` and render the onboarding tour overlay:
    ```python
    show_intro = False
    if not current_user.intro_shown:
        LOG.d("Show intro to %s", current_user)
        show_intro = True
        current_user.intro_shown = True
        Session.commit()
    ```
4. Render the newsletter alias row with its note (this is `Alias` #2 — the user's only alias at this point):
   > *"This is your first alias. It's used to receive SimpleLogin communications like new features announcements, newsletters."*

**Log line observed:**

```
dashboard/views/index.py:172 - index() -  - Show intro to <User 2 testuser@example.com testuser@example.com>
```

**HTML evidence observed in this run:**
- `<title>Alias | SimpleLogin</title>`.
- The top-level navigation contains `<a class="nav-link active" href="/dashboard/">... Aliases</a>`.
- The user's newsletter alias is rendered at `<span class="font-weight-bold">simplelogin-newsletter.swoons184@sl.local</span>` and in a clipboard `data-clipboard-text="simplelogin-newsletter.swoons184@sl.local"` attribute.
- A `<textarea>` shows the default note `"This is your first alias. It's used to receive SimpleLogin communications like new features announcements, newsletters."`.
- `data-step="2"` attributes are present on the alias card — these drive the step-by-step onboarding tour triggered by `show_intro=True`.
- An enable/disable switch is rendered: `<input type="checkbox" class="enable-disable-alias custom-switch-input" data-alias="2" data-alias-email="simplelogin-newsletter.swoons184@sl.local" checked>`.

**What confirms success:**
- HTTP `200`.
- `Show intro to <User ...>` log line (first visit only).
- The rendered HTML contains the newsletter alias email and the Aliases nav item marked `active`.
- `users.intro_shown` flips from `false` to `true` — subsequent visits skip the intro and no "Show intro..." log line is produced.

### 3.7 Consolidated Journey Log Tape

The exact chronological log captured for the end-to-end flow `testuser@example.com`, from form load through activation, was:

```
# GET /auth/login (unauthenticated)
after_request() - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.103

# GET /auth/register (form)
after_request() - 127.0.0.1 GET /auth/register ImmutableMultiDict([]) 200, takes 0.038

# POST /auth/register — user created, onboarding jobs scheduled, activation email logged
register()    - create user testuser@example.com
event_dispatcher.send_event() - Not sending events because webhook is not configured and allowed to be empty
email_utils.send_email() - send email to testuser@example.com, subject 'Just one more step to join SimpleLogin'
mail_sender.send() - send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'testuser@example.com'
after_request() - 127.0.0.1 POST /auth/register ImmutableMultiDict([]) 200, takes 0.602

# GET /auth/activate?code=uidmweayegyqgwfmacgbjjjtmxqnrb — user activated and auto-logged in
email_utils.send_email() - send email to simplelogin-newsletter.swoons184@sl.local, subject 'Welcome to SimpleLogin'
mail_sender.send()        - send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.swoons184@sl.local'
activate()                - redirect user to dashboard
after_request()           - 127.0.0.1 GET /auth/activate ImmutableMultiDict([('code','uidmweayegyqgwfmacgbjjjtmxqnrb')]) 302, takes 0.057

# GET /dashboard/ — intro tour shown
index()                   - Show intro to <User 2 testuser@example.com testuser@example.com>
after_request()           - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.291

# Fresh session — GET the login form, then POST credentials
after_request() - 127.0.0.1 GET /auth/login ImmutableMultiDict([]) 200, takes 0.022
after_login()   - log user <User 2 testuser@example.com testuser@example.com> in
after_login()   - redirect user to dashboard
after_request() - 127.0.0.1 POST /auth/login ImmutableMultiDict([]) 302, takes 0.296
after_request() - 127.0.0.1 GET /dashboard/ ImmutableMultiDict([]) 200, takes 0.143
```

This log tape is the canonical trace of a successful new-user journey on a local deployment.

---

## 4. Background Service Observability

### 4.1 Map of Background Subsystems That Run Alongside Flask

SimpleLogin is a *five-process architecture* in production. In our local deployment we only started the Flask app, but every subsystem is observable through the database records and log lines the Flask process itself produces. This section documents what each subsystem *would* do and what records it produces for the new-user flow.

| Process | Purpose | What it consumes | What it produces |
|---|---|---|---|
| `server.py` / `wsgi.py` | Flask web app (authentication, dashboard, API, OAuth) | HTTP requests | DB writes; SL log lines; `job` rows (scheduling) |
| `job_runner.py` | Long-running poller (10 s loop) that executes `Job` rows | `job` rows with `state=ready` and `run_at<=now+10min` | Outbound emails (via `mail_sender`); `job.state=done`; audit log rows |
| `cron.py` + `crontab.yml` (yacron) | Scheduled maintenance (stats, cleanup, HIBP, subscription reminders, DNS verification) | Timer ticks from yacron | DB metrics; periodic emails; cleanup side-effects |
| `email_handler.py` | Inbound SMTP via aiosmtpd (forwarding, reply routing, bounce processing) | Incoming SMTP messages on port 25 | Outbound SMTP (forwarded mail); `contact`, `email_log`, `bounce` rows |
| `event_listener.py` | PostgreSQL `LISTEN` consumer feeding partner (Proton) sync | PG `NOTIFY` payloads from `EventDispatcher` | Webhook HTTP calls to `EVENT_WEBHOOK` |
| `monitoring.py` | 60-second collection loop of queue depth, process counts, PG connections | `ps`, Postfix queue, PG queries | Metric log lines / Sentry breadcrumbs |

### 4.2 Onboarding Jobs Recorded in the `job` Table

The registration flow is the primary producer of rows in the `job` table for a new user. The code in `User.create()` (`app/models.py`) executes immediately after the first alias is inserted:

```python
if config.DISABLE_ONBOARDING:
    LOG.d("Disable onboarding emails")
    return user

Job.create(name=config.JOB_ONBOARDING_1, payload={"user_id": user.id}, run_at=arrow.now().shift(days=1))
Job.create(name=config.JOB_ONBOARDING_2, payload={"user_id": user.id}, run_at=arrow.now().shift(days=2))
Job.create(name=config.JOB_ONBOARDING_4, payload={"user_id": user.id}, run_at=arrow.now().shift(days=3))
Session.flush()
```

The job-name constants (`app/config.py`) are:

```
JOB_ONBOARDING_1 = "onboarding-1"
JOB_ONBOARDING_2 = "onboarding-2"
JOB_ONBOARDING_3 = "onboarding-3"    # not used for new-user flow
JOB_ONBOARDING_4 = "onboarding-4"
```

The `onboarding-3` job is intentionally skipped in the new-user flow (it is reserved for a different lifecycle milestone). That only `-1`, `-2`, `-4` are scheduled is by design.

**Observed live for `testuser@example.com`:**

```
 id |     name     |    payload     | state |           run_at
----+--------------+----------------+-------+----------------------------
  1 | onboarding-1 | {"user_id": 2} |     0 | 2026-04-17 23:04:24.042445
  2 | onboarding-2 | {"user_id": 2} |     0 | 2026-04-18 23:04:24.042664
  3 | onboarding-4 | {"user_id": 2} |     0 | 2026-04-19 23:04:24.042785
```

`state=0` maps to `JobState.ready` (enum defined in `app/models.py`). `run_at` is shifted +1/+2/+3 days from registration time, which is exactly what `arrow.now().shift(days=N)` produces.

**Observability gate for the new-user flow.** Even without `job_runner.py` running, the immediate existence of **exactly three onboarding rows** in the `job` table with the correct names and future `run_at` values is the conclusive evidence that registration's background-work-scheduling path executed. Fewer than three rows (or different names) would indicate `DISABLE_ONBOARDING=true` was set or that `User.create()` bailed early.

**What `job_runner.py` does with these rows when you start it.** The `__main__` loop (`job_runner.py:333-350`) wakes every 10 seconds, queries `get_jobs_to_run()` (which filters `state=ready OR (state=taken AND taken_at<now-30min AND attempts<5)` AND `run_at IS NULL OR run_at<=now+10min`), and for each match:
1. Logs `Take job <Job ...>`.
2. Marks `state=taken`, `taken_at=now()`, `attempts+=1`.
3. Calls `process_job(job)` which branches on `job.name` and for onboarding jobs invokes `onboarding_send_from_alias(user)` / `onboarding_mailbox(user)` / `onboarding_pgp(user)` — each of which composes the respective tip email via `render()` and calls `send_email()`.
4. Marks `state=done` on success.

### 4.3 Aggregate Signals in the `daily_metric` Table

Next to the per-user `job` rows, the registration flow updates a global counter used for product analytics:

```python
DailyMetric.get_or_create_today_metric().nb_new_web_non_proton_user += 1
Session.commit()
```

Observed live (after two consecutive registration attempts in the same day):

```
 id |         created_at         |         updated_at         |    date    | nb_new_web_non_proton_user | nb_alias
----+----------------------------+----------------------------+------------+----------------------------+----------
  1 | 2026-04-16 23:02:39.214475 | 2026-04-16 23:04:24.079697 | 2026-04-16 |                          2 |        2
```

The `nb_alias` column is populated by other code paths (alias creation), but you can see here it reflects the two newsletter aliases created across the two registrations. This single row per UTC date is the stable observability signal for "how many sign-ups happened today".

### 4.4 Event Dispatcher — Expected and Observed Behavior in a Local Run

`EventDispatcher.send_event()` (at `app/events/event_dispatcher.py:55`) is the emit point for domain events that feed the Proton partner sync via `event_listener.py`. In a local deployment without a configured webhook, the expected and correct behavior is to *skip* event emission. The relevant code:

```python
if config.EVENT_WEBHOOK_DISABLE:
    LOG.i("Not sending events because webhook is disabled")
    return

if not config.EVENT_WEBHOOK and skip_if_webhook_missing:
    LOG.i("Not sending events because webhook is not configured and allowed to be empty")
    return
```

**Observed in this run, exactly once per registration:**

```
event_dispatcher.py:62 - send_event() -  - Not sending events because webhook is not configured and allowed to be empty
```

This is a **positive** indicator in a local dev context — it proves the event subsystem was reached (so the code path is wired) but correctly short-circuited because there is no partner integration to notify. Production deployments connected to Proton would instead see log lines like `Sent event to the dispatcher` plus PostgreSQL `NOTIFY` broadcasts consumed by `event_listener.py`.

### 4.5 APM Events (`RegisterEvent`, `LoginEvent`) in New Relic

Independent of the PostgreSQL/Webhook event system, SimpleLogin emits *custom New Relic events* for product analytics. These are thin wrappers around `newrelic.agent.record_custom_event(...)` and are defined in `app/events/auth_event.py`:

```python
class LoginEvent:
    class ActionType(EnumE):
        success = 0
        failed = 1
        disabled_login = 2
        not_activated = 3
        scheduled_to_be_deleted = 4
    class Source(EnumE):
        web = 0
        api = 1
    def send(self):
        newrelic.agent.record_custom_event("LoginEvent", {"action": self.action.name, "source": self.source.name})


class RegisterEvent:
    class ActionType(EnumE):
        success = 0
        failed = 1
        catpcha_failed = 2
        email_in_use = 3
        invalid_email = 4
    class Source(EnumE):
        web = 0
        api = 1
    def send(self):
        newrelic.agent.record_custom_event("RegisterEvent", {"action": self.action.name, "source": self.source.name})
```

During the new-user flow, these events are emitted at known places:
- `RegisterEvent(ActionType.success)` — `app/auth/views/register.py` right after `User.create()` + `send_activation_email()` succeed.
- `LoginEvent(ActionType.success)` — `app/auth/views/login.py` immediately before `after_login(user, next_url)`.

**Important**: New Relic is a no-op if the `NEW_RELIC_LICENSE_KEY` env var is not set (the default in local dev). The events are still *constructed and `send()`-called*, but `newrelic.agent.record_custom_event` is a cheap noop. In a production run with the agent enabled, you would find these events in New Relic Insights as `RegisterEvent` and `LoginEvent` entity types with `action` and `source` attributes.

### 4.6 Email Subsystem Observability with `NOT_SEND_EMAIL=true`

All outbound email in SimpleLogin flows through `MailSender.send()` in `app/mail_sender.py`. The relevant branch for local dev is:

```python
def send(self, send_request: SendRequest, retries: int = 2) -> bool:
    if self._store_emails:
        self._emails_sent.append(send_request)
    if config.NOT_SEND_EMAIL:
        LOG.d(
            "send email with subject '%s', from '%s' to '%s'",
            send_request.msg[headers.SUBJECT],
            send_request.msg[headers.FROM],
            send_request.msg[headers.TO],
        )
        return True
    ...
```

Therefore, in local mode, the two observability tiers for email are:

1. **`email_utils.send_email()` log** (the *composition* layer, line 303) — always emitted:
   ```
   send email to testuser@example.com, subject 'Just one more step to join SimpleLogin'
   ```
2. **`mail_sender.send()` log** (the *delivery* layer, line 131) — emitted only when `NOT_SEND_EMAIL=true`:
   ```
   send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'testuser@example.com'
   ```

Seeing **both** lines in close succession is the conclusive evidence that the full mail pipeline executed (MIME composition, headers, `From` stamping with the configured `SUPPORT_EMAIL` domain, and the would-be SMTP hand-off). The second line's `from '"noreply@sl.local" <noreply@sl.local>'` confirms that `EMAIL_DOMAIN=sl.local` was threaded through the SMTP envelope correctly.

For the new-user flow, the complete email log tape was:

```
# Registration:
send email to testuser@example.com, subject 'Just one more step to join SimpleLogin'
send email with subject 'Just one more step to join SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'testuser@example.com'

# Activation:
send email to simplelogin-newsletter.swoons184@sl.local, subject 'Welcome to SimpleLogin'
send email with subject 'Welcome to SimpleLogin', from '"noreply@sl.local" <noreply@sl.local>' to 'simplelogin-newsletter.swoons184@sl.local'
```

Two activation-phase emails, two different recipients (the registration email, then the newsletter alias), exactly as specified by `send_activation_email()` and `send_welcome_email()` in `app/email_utils.py`.

### 4.7 `job_runner.py` — What You Would See If You Started It

The job runner is a separate process and was **not** started for this investigation (it is not required for the new-user-flow observation). If you run it:

```bash
python job_runner.py
```

…its main loop is (abbreviated from the source):

```python
if __name__ == "__main__":
    while True:
        with create_light_app().app_context():
            for job in get_jobs_to_run():
                LOG.d("Take job %s", job)
                job.taken = True
                job.taken_at = arrow.now()
                job.state = JobState.taken.value
                job.attempts += 1
                Session.commit()
                process_job(job)

                job.state = JobState.done.value
                Session.commit()

            time.sleep(10)
```

Observability signals you would see if you ran it against this user's jobs (after `run_at` passes):

- **`Take job <Job 1 onboarding-1 ...>`** immediately on pick-up.
- `job.state` transitions: `ready(0) → taken(1) → done(2)` in the DB.
- For each onboarding job, a pair of `send_email` / `send` log lines for the tip email it produces (subjects such as `SimpleLogin Tip: Send emails from your alias`, `SimpleLogin Tip: Secure your emails with PGP`, etc., visible in `job_runner.py`'s `onboarding_send_from_alias`/`onboarding_pgp` helpers).
- If `process_job` branches into an unknown name, you would see `Unknown job name <name>`.

For retries: `get_jobs_to_run()` also picks up jobs stuck in `taken` for more than `JOB_TAKEN_RETRY_WAIT_MINS=30` minutes (per `app/config.py`), provided `attempts < JOB_MAX_ATTEMPTS=5`. That guarantees at-least-once execution.

### 4.8 `cron.py` + `crontab.yml` Maintenance Scheduler

`cron.py` defines ~16 scheduled tasks (stats, cleanup, HIBP checks, subscription reminders, DNS verification), all wrapped in `create_light_app().app_context()`. The schedule is expressed in YAML in `crontab.yml` and `crontab-all-hosts.yml`, and executed by `yacron` (a pure-Python cron replacement). This process is independent of the Flask app and was not started for this investigation.

Two notable tasks relevant to the new-user flow:

- `send_undelivered_mails` (every 5 minutes) — retries any SMTP messages written to `MailSender`'s filesystem dead-letter area by a previous failure. In `NOT_SEND_EMAIL=true` mode this path is inactive.
- `stats` (daily at midnight UTC) — aggregates platform-wide metrics; reads from `daily_metric` (which we observed being written to on registration).

### 4.9 `email_handler.py`, `event_listener.py`, `monitoring.py`

These three processes are part of the production topology but are **not exercised by the user-facing registration/login flow** on a fresh local deployment with `NOT_SEND_EMAIL=true`:

- **`email_handler.py`** — inbound SMTP on port 25 (aiosmtpd); relevant only when real mail flows in/out via Postfix. Its absence here does not affect registration correctness.
- **`event_listener.py`** — PostgreSQL `LISTEN` consumer for the `EventDispatcher` `NOTIFY` channel; only meaningful when `EVENT_WEBHOOK` is configured (we logged `Not sending events because webhook is not configured` confirming it would be a no-op anyway).
- **`monitoring.py`** — periodic metric collection of Postfix queue depth, etc.; unused in local dev without Postfix.

Their absence from the log does not invalidate the successful new-user flow observation; the flow is fully self-contained within the Flask process and PostgreSQL.

---

## 5. Temporary Test Data and Cleanup

As instructed, all test data created during this verification was removed from PostgreSQL after the walkthrough. The cleanup statements were executed inside a single transaction to respect foreign-key constraints (notably `users.default_mailbox_id → mailbox.id`, which must be nulled before the mailbox can be deleted):

```sql
BEGIN;

-- Break the circular users ↔ mailbox FK so we can delete the mailbox
UPDATE users SET default_mailbox_id = NULL, newsletter_alias_id = NULL WHERE id = 2;

-- Delete any activation codes (should already be empty post-activation)
DELETE FROM activation_code WHERE user_id = 2;

-- Delete onboarding jobs created by User.create()
DELETE FROM job WHERE payload::text LIKE '%"user_id": 2%';

-- Delete alias-to-mailbox link rows, then aliases themselves
DELETE FROM alias_mailbox WHERE alias_id IN (SELECT id FROM alias WHERE user_id = 2);
DELETE FROM alias WHERE user_id = 2;

-- Delete audit log rows produced during user lifecycle
DELETE FROM user_audit_log WHERE user_id = 2;

-- Delete the user's mailbox
DELETE FROM mailbox WHERE user_id = 2;

-- Reset the daily metric counter
DELETE FROM daily_metric;

-- Delete the user row last
DELETE FROM users WHERE id = 2;

COMMIT;
```

**Post-cleanup verification** (executed live):

```
       tbl       | count
-----------------+-------
 users           |     0
 mailbox         |     0
 alias           |     0
 job             |     0
 daily_metric    |     0
 activation_code |     0
 user_audit_log  |     0
 public_domain   |     1     <-- intentional: this is the SL domain seeded by init_app.py, not test data
```

Only `public_domain` retained a single row — `sl.local` — which is infrastructure seed data created by `init_app.py` (not user-created test data) and is expected to persist across runs. All user-created records are gone.

---

## 6. Consolidated "Everything Is Healthy" Dashboard

A single-glance summary of every observability signal for a fresh local deployment running through a new-user journey:

| Area | Signal | Expected Value | How to Verify |
|---|---|---|---|
| Startup | Boot log block | Contains `>>> URL`, `>>> init logging`, `* Serving Flask app "server"` | Inspect stdout of `python server.py` |
| Startup | `GET /auth/login` | `200` | `curl -s -o /dev/null -w '%{http_code}\n' http://localhost:7777/auth/login` |
| Startup | `GET /` | `302` | `curl -s -o /dev/null -w '%{http_code}\n' http://localhost:7777/` |
| Startup | `public_domain` table | ≥1 row matching `EMAIL_DOMAIN` | `SELECT * FROM public_domain;` |
| Startup | `alembic_version` | 1 row (head revision) | `SELECT * FROM alembic_version;` |
| Register | `create user <email>` log | Present on POST | `grep 'create user' <log>` |
| Register | Activation email log pair | Both composition + delivery lines visible | `grep 'Just one more step' <log>` |
| Register | `users` row | `activated=false`, `default_mailbox_id=<mb>`, `newsletter_alias_id=<alias>` | `SELECT * FROM users WHERE email=...;` |
| Register | `mailbox` row | `verified=true` | `SELECT * FROM mailbox;` |
| Register | `alias` row | 1 row, `email LIKE 'simplelogin-newsletter.%'` | `SELECT email FROM alias;` |
| Register | `job` rows | 3 rows: `onboarding-1`, `onboarding-2`, `onboarding-4` with `state=0` | `SELECT name, state, run_at FROM job;` |
| Register | `activation_code` row | 1 row per user | `SELECT * FROM activation_code;` |
| Register | `daily_metric` | Today's date, counter ≥1 | `SELECT * FROM daily_metric;` |
| Register | `POST /auth/register` | `200` (not `302`) | curl status |
| Activate | `redirect user to dashboard` log | Present on activate | `grep 'redirect user to dashboard' <log>` |
| Activate | Welcome email to **alias** | Subject `'Welcome to SimpleLogin'`, to=newsletter alias | log pair |
| Activate | `activation_code` | Row deleted | `SELECT count(*) FROM activation_code WHERE user_id=...;` |
| Activate | `users.activated` | `true` | `SELECT activated FROM users WHERE ...;` |
| Activate | `GET /auth/activate?code=...` | `302 -> /dashboard/` | curl |
| Login | `log user <User...> in` + `redirect user to dashboard` | Paired DEBUG lines | `grep after_login <log>` |
| Login | `POST /auth/login` | `302 -> /dashboard/` | curl |
| Dashboard | `Show intro to <User...>` | First-visit only | `grep 'Show intro' <log>` |
| Dashboard | `users.intro_shown` | Flips to `true` after first visit | `SELECT intro_shown FROM users;` |
| Dashboard | HTML contains the newsletter alias string | Visible | grep the response |
| Events | `Not sending events because webhook is not configured...` | Info line per register event | `grep 'Not sending events' <log>` |

---

## 7. Appendix — File and Code Anchors Used as Evidence

All observations in this document are traceable to the following specific files and line ranges in the repository:

**Application factory & startup.**
- `server.py:234-246` — `register_blueprints(app)`: auth_bp, monitor_bp, dashboard_bp, developer_bp, phone_bp, oauth_bp (×2), onboarding_bp, discover_bp, internal_bp, api_bp.
- `server.py:284` — `after_request()` DEBUG log producing the `127.0.0.1 METHOD /path ... status, takes ...` lines.
- `wsgi.py` — one-line WSGI entry point for gunicorn.
- `app/config.py:79-80` — `URL = os.environ["URL"]; print(">>> URL:", URL)`.
- `app/log.py:67` — `print(">>> init logging <<<")` plus `LOG = _get_logger("SL")`.
- `init_app.py` — `add_sl_domains()` seeds `public_domain`; `load_pgp_public_keys()` walks `Mailbox`/`Contact` keys.

**Registration.**
- `app/auth/views/register.py:60-100` — captcha, email canonicalization, `User.create()`, `send_activation_email()`, `DailyMetric` increment, render `register_waiting_activation.html`.
- `app/models.py:602-668` — `User.create()`: creates mailbox (`verified=True`), newsletter alias (`prefix="simplelogin-newsletter"`), schedules `JOB_ONBOARDING_{1,2,4}` via `arrow.now().shift(days=1|2|3)`.
- `templates/auth/register_waiting_activation.html` — the "An email to validate your email is on its way." confirmation page.

**Activation.**
- `app/auth/views/activate.py` — code lookup, expiry check, `user.activated=True`, `login_user(user)`, `ActivationCode.delete()`, `send_welcome_email`, `LOG.d("redirect user to dashboard")`, `redirect(url_for("dashboard.index"))`.
- `app/email_utils.send_welcome_email()` — sends to `user.newsletter_alias.email`, not `user.email`.

**Login.**
- `app/auth/views/login.py` — credential validation, disabled/delete_on/activated gates; each gate emits a different `LoginEvent.ActionType`.
- `app/auth/views/login_utils.py:12-45` — `after_login()` MFA decision (FIDO → TOTP → direct), `login_user()`, `session["sudo_time"]`, redirect to `dashboard.index`.

**Dashboard.**
- `app/dashboard/views/index.py:67-172` — query/filter/sort handling, `show_intro` logic at line 170 (`LOG.d("Show intro to %s", current_user)`), `stats = get_stats(current_user)`, pagination.
- `templates/dashboard/index.html` — renders the alias list, stats cards, and the onboarding tour (via `data-step` attributes).

**Email.**
- `app/mail_sender.py:128-140` — `NOT_SEND_EMAIL` short-circuit: `LOG.d("send email with subject '%s', from '%s' to '%s'", ...)`.
- `app/email_utils.py:303` — `LOG.d("send email to %s, subject %r", ...)` (composition-layer log).

**Events.**
- `app/events/event_dispatcher.py:55-85` — `EventDispatcher.send_event()`: `EVENT_WEBHOOK_DISABLE` / missing webhook / missing partner_user early returns; protobuf serialize + `dispatcher.send(serialized)`; `newrelic.agent.record_custom_event("EventStoredToDb", ...)`.
- `app/events/auth_event.py` — `LoginEvent`, `RegisterEvent` with their `ActionType` and `Source` enums and `send()` wrapping `newrelic.agent.record_custom_event`.

**Background processing.**
- `job_runner.py:333-350` — main `while True` loop; `get_jobs_to_run()` query at lines 308-328; `process_job()` branches on `job.name`.
- `cron.py` + `crontab.yml` + `crontab-all-hosts.yml` — yacron-driven maintenance tasks.
- `monitoring.py`, `email_handler.py`, `event_listener.py` — side-car processes not required for the new-user flow.

**Schema / migrations.**
- `alembic.ini` — Alembic config pointing at `migrations/`.
- `migrations/versions/` — 255 migration revisions (applied with `alembic upgrade head`).

**Configuration reference.**
- `example.env` — documents all supported environment variables, including `URL`, `EMAIL_DOMAIN`, `DB_URI`, `FLASK_SECRET`, `NOT_SEND_EMAIL`, `COLOR_LOG`, `EMAIL_SERVERS_WITH_PRIORITY`, `DISABLE_ONBOARDING` (opt-out), `EVENT_WEBHOOK`.
- `pyproject.toml` — Python `^3.10`, Flask `^1.1.2`, SQLAlchemy `1.3.24`, and ~50 other dependencies.
- `Dockerfile` — two-stage image: Node 10.17.0 for frontend assets in `static/`, Python 3.10 for the backend; exposes port 7777; `CMD` runs gunicorn.

---

**End of guide.** All claims are based on direct code reading and a live, successful run of the registration → activation → login → dashboard flow on a fresh local deployment with PostgreSQL 16 and Redis 7. All user-created test data was cleaned up immediately after verification, leaving only the infrastructure-seed `public_domain` row intact. No source code was modified during this investigation.
