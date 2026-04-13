# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **produce a comprehensive runtime-verification and behavioral-analysis document** for the SimpleLogin open-source email alias application. The user is new to the project and needs authoritative, code-grounded answers to the following questions:

- **Startup Verification:** After launching SimpleLogin locally, what observable evidence (log output, HTTP responses, dashboard UI state) confirms that the three core processes — the Flask web server, the aiosmtpd email handler, and the polling job runner — are alive, healthy, and capable of serving requests?
- **User-Facing Workflow Verification:** Once the application is running, what does the system produce (in logs, database records, and the dashboard UI) when a user signs in, manages aliases, and interacts with the alias management surface?
- **End-to-End Email Flow Verification:** When a newly created alias receives an inbound email, what runtime artifacts (log lines, database rows in `email_log`, `contact`, and `notification` tables, SMTP status codes) confirm that the email handler processed the message correctly?
- **Background Component Behavior:** Do the email handler (`email_handler.py`) and job runner (`job_runner.py`) start automatically alongside the web server, or must they be launched as independent processes? What runtime behavior and log patterns confirm they are functioning correctly across different operational scenarios?

The deliverable is a single Markdown document named `app_2cd6ee777f8c.md` placed in the `blitzy/documentation` directory that comprehensively answers all of the above questions with rationale grounded in the actual source code.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: No Source Code Modification.** The user has explicitly stated: *"Do not modify the source code."* This means no changes to any existing `.py`, `.html`, `.env`, `.toml`, or any other repository file. The only artifact produced is the new Markdown documentation file.
- **Implementation Rule `SWE-AtlasQnA-Repo`:** The implementation rule mandates:
  - Create a new markdown document named `<source_branch_name>.md` (i.e., `app_2cd6ee777f8c.md`)
  - Provide thinking and rationale behind the answers
  - Base all answers on the code as the single source of truth — no assumptions
  - Do not modify any existing files in the source repository
  - Do not add any other code besides the requested document
  - Place the document in the `blitzy/documentation` directory
- **Temporary Test Data Cleanup:** The user permits creating temporary testing data (accounts, aliases) but requires cleanup when done. Since the task is a documentation exercise grounded in code analysis rather than runtime testing, this constraint is acknowledged but the deliverable is a documentation artifact.
- **Architectural Preservation:** The existing repository conventions — Flask blueprints, SQLAlchemy ORM patterns, aiosmtpd SMTP handling, 10-second job polling — must be described accurately without suggesting modifications.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer the startup verification questions**, we will analyze `server.py` (the Flask bootstrap with the `/health` endpoint at line 213), `email_handler.py` (the `main()` function at line 2381 that starts the aiosmtpd Controller and logs `"Start mail controller"` at line 2386), and `job_runner.py` (the `while True` polling loop at line 330 that logs `"Take job"` at line 334) to identify all startup log signatures and health check mechanisms.
- To **answer the user workflow questions**, we will trace the auth flow through `app/auth/views/login.py` (the `/login` route), `app/auth/views/register.py` (the `/register` route), and `app/dashboard/views/index.py` (the dashboard landing page that computes `Stats` including `nb_alias`, `nb_forward`, `nb_reply`, `nb_block`), documenting what the UI presents and what logs are emitted at each step.
- To **answer the email flow questions**, we will trace the `MailHandler.handle_DATA()` method (line 2289), the `handle()` dispatch function (line 1945), and `handle_forward()` (line 536), documenting every log line, database record creation (`Contact`, `EmailLog`), and SMTP status code returned.
- To **answer the background component questions**, we will document that each process (`server.py`, `email_handler.py`, `job_runner.py`) is an independent entry point that must be launched separately, as evidenced by the Docker deployment in `README.md` (three separate `docker run` commands for `sl-app`, `sl-email`, and `sl-job-runner` containers).
- To **produce the deliverable**, we will create a single file at `blitzy/documentation/app_2cd6ee777f8c.md` containing the complete analysis with code-referenced rationale.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The following files and directories were identified as relevant to answering the user's runtime-verification and behavioral-analysis questions. Each file was inspected via `read_file` or `get_source_folder_contents`.

**Core Application Entry Points (startup and health behavior)**

| File | Relevance | Key Observations |
|------|-----------|------------------|
| `server.py` | Flask app factory, health endpoint, startup logging | Defines `create_app()` with `/health` → `"success", 200` (line 213-215); `local_main()` starts on port 7777 (line 588); `after_request` logs every non-static request with method, path, status, and elapsed time (lines 272-296) |
| `wsgi.py` | Production WSGI entry point | Single-line `app = create_app()`, used by Gunicorn to serve the Flask app |
| `email_handler.py` | SMTP inbound handler | `main()` at line 2381 starts aiosmtpd `Controller` on port 20381; logs `"Start mail controller"` (line 2386) and `"Listen for port"` (line 2403); `MailHandler.handle_DATA()` logs `"New message, mail from..."` for each inbound email (line 2344) |
| `job_runner.py` | Background job processor | `while True` loop at line 330 polls every 10 seconds; `get_jobs_to_run()` queries `Job` table for ready/stale jobs; logs `"Take job"` when processing (line 334); transitions jobs through `ready → taken → done` states |
| `cron.py` | Scheduled maintenance tasks | Driven by `crontab.yml` via yacron; 16+ scheduled jobs including stats, cleanup, HIBP checks, subscription notifications |
| `Dockerfile` | Container build and runtime command | Gunicorn CMD: `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`; exposes port 7777 |

**Authentication and User Management (sign-in and account creation verification)**

| File | Relevance | Key Observations |
|------|-----------|------------------|
| `app/auth/views/login.py` | Login route `/auth/login` | Validates credentials, emits `LoginEvent`, calls `after_login()` on success; logs redirect behavior; flash messages for disabled/unactivated accounts |
| `app/auth/views/register.py` | Registration route `/auth/register` | Creates `User` record, generates `ActivationCode`, sends activation email; checks `DISABLE_REGISTRATION`; logs `"create user"` |
| `app/auth/views/activate.py` | Account activation `/auth/activate` | Validates activation code, sets `user.activated = True`, sends welcome email |
| `app/auth/views/login_utils.py` | Post-login flow | `after_login()` decides MFA challenge or direct dashboard redirect |
| `app/auth/base.py` | Auth blueprint definition | Creates `auth_bp` with prefix `/auth` |
| `app/fake_data.py` | Seed data for local development | Creates `john@wick.com` / `password` user with admin, aliases, contacts, subscriptions, and directories; run via `flask dummy-data` |

**Dashboard and Alias Management (alias creation and management verification)**

| File | Relevance | Key Observations |
|------|-----------|------------------|
| `app/dashboard/views/index.py` | Dashboard landing page `/dashboard/` | Computes `Stats(nb_alias, nb_forward, nb_reply, nb_block)` from `Alias` and `EmailLog` tables; handles random alias creation via `Alias.create_new_random()`; logs `"create new random alias..."` |
| `app/dashboard/views/custom_alias.py` | Custom alias creation `/dashboard/custom_alias` | Validates prefix, suffix, mailbox ownership; creates `Alias` and `AliasMailbox` records |
| `app/dashboard/base.py` | Dashboard blueprint | Creates `dashboard_bp` with prefix `/dashboard` |
| `app/alias_utils.py` | Alias lifecycle helpers | `try_auto_create()` for on-the-fly alias creation; alias deletion, transfer, status changes |

**Email Processing (inbound email flow verification)**

| File | Relevance | Key Observations |
|------|-----------|------------------|
| `email_handler.py` (full) | Complete SMTP inbound flow | `handle()` at line 1945 dispatches to `handle_forward()` (line 536) or `handle_reply()`; creates `Contact` and `EmailLog` records; returns SMTP status codes from `app/email/status.py` |
| `app/email/status.py` | SMTP response code catalog | Defines all status codes: `E200` = "250 Message accepted for delivery", `E515` = "550 SL E515 Email not exist", etc. |
| `app/contact_utils.py` | Contact creation/lookup | `create_contact()` creates or finds contacts for incoming emails |
| `app/email_utils.py` | Email composition and delivery | `send_email()`, DKIM signing, header manipulation, VERP handling |
| `app/handler/dmarc.py` | DMARC policy enforcement | `apply_dmarc_policy_for_forward_phase()` evaluates DMARC for incoming mail |

**Configuration and Logging (understanding runtime indicators)**

| File | Relevance | Key Observations |
|------|-----------|------------------|
| `app/config.py` | All environment-driven configuration | `NOT_SEND_EMAIL` flag for local dev (line 91); `DB_URI` for Postgres; job names; alert types; rate limits |
| `app/log.py` | Logging infrastructure | Creates `LOG` logger with format including timestamp, level, process ID, file path, line number, function name, and message ID; shortcuts `LOG.d`, `LOG.i`, `LOG.w`, `LOG.e` for debug/info/warning/exception |
| `example.env` | Reference environment configuration | Documents all configuration knobs with defaults for local development |
| `app/models.py` | ORM model definitions | `User`, `Alias`, `Contact`, `EmailLog`, `Job`, `JobState`, `Mailbox`, `Notification` — all tables relevant to verifying runtime behavior |

**Monitoring and Observability (metrics and health)**

| File | Relevance | Key Observations |
|------|-----------|------------------|
| `monitoring.py` | Operational metrics exporter | Logs Postfix queue sizes, DB connections, pending events; exports to New Relic |
| `app/monitor/base.py` | Monitor blueprint | Provides monitoring endpoints |

**Deployment and Infrastructure (how components are launched)**

| File | Relevance | Key Observations |
|------|-----------|------------------|
| `README.md` | Self-hosting guide | Shows three separate `docker run` commands for `sl-app`, `sl-email`, `sl-job-runner` |
| `CONTRIBUTING.md` | Local development guide | Documents `alembic upgrade head && flask dummy-data && python3 server.py`; login with `john@wick.com / password`; separate `python email_handler.py` and `python job_runner.py` commands |
| `crontab.yml` | Cron job schedule | 16+ scheduled tasks with yacron scheduling syntax |
| `init_app.py` | Database initialization | Seeds SL domains, PGP keys, Proton partner data |
| `scripts/run-test.sh` | Test execution | Starts Postgres, runs migrations, executes pytest |

**Test Infrastructure (understanding expected behavior)**

| File | Relevance | Key Observations |
|------|-----------|------------------|
| `tests/conftest.py` | Test fixtures | Creates Flask test app, seeds baseline data, provides `flask_client` fixture |
| `tests/test_email_handler.py` | Email handler tests | Tests for the email handling flow |
| `docs/troubleshooting.md` | Diagnostic procedures | Documents `swaks` testing, `docker logs` inspection, Postfix verification |
| `docs/api.md` | API reference | Full endpoint documentation for auth, alias, mailbox, contact, notification endpoints |

### 0.2.2 Web Search Research Conducted

No external web searches were required for this task. All answers are grounded exclusively in the source code repository, as mandated by the implementation rule: *"Do not make assumptions, base your answers on the code as the truth."*

### 0.2.3 New File Requirements

A single new file will be created:

| File Path | Purpose |
|-----------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | Comprehensive runtime-verification and behavioral-analysis document answering all user questions about startup verification, user workflow traces, email flow behavior, and background component operation — with code-referenced rationale |

No other files will be created or modified in the repository.

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

The following packages are key to understanding the runtime behavior described in the documentation deliverable. All versions are sourced from `pyproject.toml` (the Poetry dependency manifest).

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| PyPI | `flask` | ^1.1.2 | Web framework powering the dashboard, auth, API, and admin UI |
| PyPI | `gunicorn` | ^20.0.4 | WSGI HTTP server that serves the Flask app on port 7777 |
| PyPI | `aiosmtpd` | ^1.2 | Async SMTP server library used by `email_handler.py` to receive inbound email on port 20381 |
| PyPI | `sqlalchemy` | 1.3.24 (pinned) | ORM layer for all database models (`User`, `Alias`, `Contact`, `EmailLog`, `Job`) |
| PyPI | `psycopg2-binary` | ^2.9.3 | PostgreSQL database adapter for SQLAlchemy |
| PyPI | `flask_login` | ^0.5.0 | Session-based user authentication for web dashboard |
| PyPI | `flask-migrate` (Alembic) | ^2.5.3 | Database schema migration management |
| PyPI | `flask_admin` | ^1.5.6 | Admin panel for managing users, aliases, domains |
| PyPI | `arrow` | ^0.16.0 | Date/time library used extensively for job scheduling and timestamp handling |
| PyPI | `coloredlogs` | ^14.0 | Colored console log output for local development |
| PyPI | `sentry_sdk` | ^2.16.0 | Error tracking and performance monitoring |
| PyPI | `newrelic` | 8.8.0 (pinned) | Application performance monitoring agent |
| PyPI | `redis` | ^4.5.3 | Session storage, rate limiting, and distributed concurrency locks |
| PyPI | `yacron` | ^0.11.1 | YAML-configured cron scheduler for `crontab.yml` |
| PyPI | `dkimpy` | ^1.0.5 | DKIM email signing for outbound messages |
| PyPI | `python-gnupg` | ^0.4.6 | PGP encryption for email content |
| PyPI | `PGPy` | 0.5.4 (pinned) | Alternative PGP implementation used as fallback |
| PyPI | `bcrypt` | ^3.2.0 | Password hashing for user authentication |
| PyPI | `pyotp` | ^2.4.0 | TOTP-based multi-factor authentication |
| PyPI | `webauthn` | ^0.4.7 | FIDO/WebAuthn hardware key authentication |
| PyPI | `flask-cors` | ^3.0.9 | CORS support for API endpoints (`/api/*`) |
| PyPI | `boto3` | ^1.15.9 | S3-compatible object storage (or local file fallback) |
| PyPI | `python-dotenv` | ^0.14.0 | Environment variable loading from `.env` files |
| npm | (static assets) | via `package.json` in `static/` | Frontend JavaScript/CSS dependencies (Node 10) |

### 0.3.2 Dependency Updates

**No dependency updates are required.** This task creates a documentation-only artifact (`app_2cd6ee777f8c.md`) and does not modify any source code, configuration, or dependency manifests. All existing dependencies remain as-is.

**Import Updates:** Not applicable — no Python files are being modified.

**External Reference Updates:** Not applicable — no configuration, build, or CI files are being modified.

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

Since this task produces a read-only documentation artifact, no code modifications are made. However, the following integration points were analyzed to produce accurate answers about runtime behavior:

**Web Server Integration Points**

- **`server.py` → `create_app()`**: The Flask application factory wires together all blueprints (`auth_bp`, `dashboard_bp`, `api_bp`, `oauth_bp`, etc.), initializes extensions (Flask-Login, Flask-Limiter), sets up error handlers, configures session management, and defines the `/health` health check endpoint. The `after_request` hook (line 272) logs every non-static HTTP request with remote address, method, path, arguments, status code, and elapsed time.
- **`server.py` → `local_main()`**: The local development entry point enables Flask debug mode, debug toolbar, and runs on port 7777. This is what developers interact with when running `python3 server.py`.
- **`wsgi.py` → `create_app()`**: Production deployment uses Gunicorn to load the WSGI app object via this module.

**Email Handler Integration Points**

- **`email_handler.py` → `MailHandler.handle_DATA()`** (line 2289): The async SMTP entry point that receives raw email bytes, parses them, and dispatches to `_handle()`.
- **`email_handler.py` → `_handle()`** (line 2335): Sets a unique `message_id` for lifecycle tracking, creates a `create_light_app()` context, calls `handle()`, logs the elapsed time and return status code.
- **`email_handler.py` → `handle()`** (line 1945): The central dispatch function that sanitizes addresses, determines if the email is a forward, reply, bounce, unsubscribe, complaint, or VERP message, and routes to the appropriate handler.
- **`email_handler.py` → `handle_forward()`** (line 536): Looks up or auto-creates the alias, creates a Contact record via `get_or_create_contact()`, creates an `EmailLog` entry, and forwards to the user's mailbox via Postfix.
- **`email_handler.py` → `main()`** (line 2381): Starts the aiosmtpd `Controller` on the configured port (default 20381) and enters an infinite `time.sleep(2)` loop.

**Job Runner Integration Points**

- **`job_runner.py` → `get_jobs_to_run()`** (line 307): Queries the `Job` table for records matching `state == ready` or stale `taken` jobs (older than 30 minutes with fewer than 5 attempts), and scheduled within a 10-minute run window.
- **`job_runner.py` → `process_job()`** (line 188): Dispatches to named handlers: `onboarding-1` through `onboarding-4`, `batch-import`, `delete-account`, `delete-mailbox`, `delete-domain`, `send-user-report`, `proton-welcome-1`, and `send-alias-creation-events`.
- **`job_runner.py` → main loop** (line 329-347): Creates a `create_light_app()` context, processes all available jobs, commits state transitions (`ready → taken → done`), then sleeps 10 seconds.

**Database Schema Touchpoints**

- **`app/models.py` → `User`** (line 336): Stores user credentials, activation status, subscription state, and notification preferences.
- **`app/models.py` → `Alias`** (line 1469): Core alias entity with email, user association, mailbox binding, enabled/disabled flag.
- **`app/models.py` → `Contact`** (line 1863): Represents external senders who have emailed an alias; stores reply email (reverse alias) address.
- **`app/models.py` → `EmailLog`** (line 2060): Audit trail for every email processed — tracks forwarded, replied, blocked, bounced status.
- **`app/models.py` → `Job`** (line 2683): Background job queue with name, payload, state (ready/taken/done/error), attempts count, and scheduling fields.
- **`app/models.py` → `Notification`** (line 3065): In-app notifications displayed to the user in the dashboard.

**Configuration Touchpoints**

- **`app/config.py`**: Central configuration loaded from `.env` via `python-dotenv`. Key flags for runtime verification include `NOT_SEND_EMAIL` (suppresses actual SMTP delivery), `COLOR_LOG` (enables colored console output), `DISABLE_REGISTRATION` (blocks new signups), `DB_URI` (PostgreSQL connection), and `POSTFIX_PORT` (outbound SMTP port).
- **`example.env`**: Reference configuration file documenting all available environment variables with local development defaults (e.g., `URL=http://localhost:7777`, `EMAIL_DOMAIN=sl.local`, `NOT_SEND_EMAIL=true`).

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since this task creates only a documentation artifact, there is a single file to produce. However, the execution plan details what content that file must contain, grounded in the exhaustive code analysis performed.

**Group 1 — Documentation Artifact:**

- **CREATE: `blitzy/documentation/app_2cd6ee777f8c.md`** — The comprehensive Q&A document that covers:

  **Section A — Startup Verification:**
  - Web server readiness: The `/health` endpoint in `server.py` returns `"success", 200`. The `after_request` hook logs every request. Gunicorn startup produces worker-ready messages.
  - Email handler readiness: `email_handler.py` logs `"Listen for port 20381"` at startup (line 2403), followed by `"Start mail controller 0.0.0.0 20381"` (line 2386). The aiosmtpd Controller begins accepting SMTP connections.
  - Job runner readiness: `job_runner.py` enters its polling loop silently (no explicit startup log), but logs `"Take job <Job>"` whenever it finds work. Absence of error logs and continuous 10-second polling confirms it is running.

  **Section B — User Workflow Verification:**
  - Sign-in flow: The login route at `/auth/login` validates credentials, emits `LoginEvent`, and redirects to `/dashboard/`. The dashboard index computes `Stats(nb_alias, nb_forward, nb_reply, nb_block)` and renders the alias list.
  - Account creation: The register route at `/auth/register` creates a `User` record and `ActivationCode`, sends an activation email (suppressed when `NOT_SEND_EMAIL=true`), then shows `register_waiting_activation.html`.
  - Alias management: Random aliases are created via `Alias.create_new_random()` on the dashboard; custom aliases via `/dashboard/custom_alias`. Both log creation events and flash success messages.

  **Section C — Email Flow Verification:**
  - Inbound email handling: The `handle()` function logs full mail headers including `mail_from`, `rcpt_tos`, `header_from`, `header_to`, `message_id`, and `client_ip` (line 1980-1994). `handle_forward()` creates `Contact` and `EmailLog` records, returns `E200` ("250 Message accepted for delivery") on success.
  - The `_handle()` wrapper logs completion: `"Finish mail_from ..., takes X seconds with return code 'Y'"` (line 2367-2373).
  - SMTP status codes from `app/email/status.py` indicate the outcome: `E200` for success, `E515` for non-existent alias, `E502` for disabled account.

  **Section D — Background Component Behavior:**
  - The email handler and job runner are **independent processes** that must be started separately. They do not auto-launch with the web server.
  - In Docker deployment: three separate containers (`sl-app`, `sl-email`, `sl-job-runner`) with `--restart always`.
  - In local development: three separate terminal commands (`python3 server.py`, `python email_handler.py`, `python job_runner.py`).

### 0.5.2 Implementation Approach per File

The documentation file will be structured as follows:

- **Establish the runtime verification foundation** by tracing each entry point's startup sequence through the source code, identifying every log statement, status endpoint, and observable side effect.
- **Map user workflows to code paths** by following the Flask route definitions in `app/auth/views/` and `app/dashboard/views/`, documenting the exact log lines, database queries, flash messages, and template renderings that occur at each step.
- **Trace the complete email lifecycle** from SMTP arrival in `MailHandler.handle_DATA()` through `handle()` dispatching to `handle_forward()` or `handle_reply()`, documenting every database record created (Contact, EmailLog), every log line emitted, and every SMTP status code returned.
- **Document the background process architecture** by analyzing `README.md` (Docker deployment), `CONTRIBUTING.md` (local development), and the source code of each entry point to explain why each process is independent and what behavior confirms it is operational.
- **Provide code-referenced rationale** for every claim by citing specific file paths, line numbers, function names, and log message patterns from the analyzed source code.

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Documentation Artifact:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — The sole deliverable

**Source Files Analyzed (read-only, no modifications):**
- Entry points: `server.py`, `email_handler.py`, `job_runner.py`, `cron.py`, `wsgi.py`, `init_app.py`
- Configuration: `app/config.py`, `example.env`, `crontab.yml`
- Auth module: `app/auth/views/login.py`, `app/auth/views/register.py`, `app/auth/views/activate.py`, `app/auth/views/login_utils.py`, `app/auth/base.py`
- Dashboard module: `app/dashboard/views/index.py`, `app/dashboard/views/custom_alias.py`, `app/dashboard/base.py`
- Email processing: `app/email/status.py`, `app/contact_utils.py`, `app/email_utils.py`, `app/handler/dmarc.py`
- Data models: `app/models.py` (User, Alias, Contact, EmailLog, Job, JobState, Mailbox, Notification)
- Logging: `app/log.py`
- Seed data: `app/fake_data.py`
- Infrastructure: `Dockerfile`, `README.md`, `CONTRIBUTING.md`
- Monitoring: `monitoring.py`
- Troubleshooting: `docs/troubleshooting.md`
- API reference: `docs/api.md`
- Scripts: `scripts/run-test.sh`, `scripts/reset_local_db.sh`

**Topics Covered in the Document:**
- Web server (Flask/Gunicorn) startup and health verification
- Email handler (aiosmtpd) startup and readiness verification
- Job runner startup and operational verification
- User sign-in and registration flow behavior
- Alias creation and management flow behavior
- Inbound email reception and forwarding flow behavior
- Background process architecture and lifecycle
- Log patterns and SMTP status codes that confirm correct operation
- Dashboard UI indicators (alias counts, notifications, stats)
- Database records created during normal operations

### 0.6.2 Explicitly Out of Scope

- **Source code modifications** — No changes to any `.py`, `.html`, `.env`, `.toml`, `.yml`, `.json`, or any other existing repository file
- **New code additions** — No new Python scripts, test files, configuration files, or build artifacts beyond the single Markdown document
- **Runtime testing or execution** — The deliverable is a documentation artifact based on code analysis; no actual process launching, database seeding, or SMTP testing is performed
- **Performance optimization** — No profiling, benchmarking, or performance tuning recommendations
- **Security hardening** — No security audit, penetration testing, or vulnerability remediation
- **Deployment configuration changes** — No modifications to Dockerfile, docker-compose, nginx, or Postfix configuration
- **Dependency upgrades** — No changes to `pyproject.toml`, `poetry.lock`, or `package.json`
- **Feature additions unrelated to the user's questions** — No new endpoints, models, or business logic
- **CI/CD pipeline modifications** — No changes to `.github/workflows/` or test infrastructure
- **External service integrations** — No Sentry, New Relic, Paddle, or other third-party service configuration

## 0.7 Rules for Feature Addition

The following rules and constraints are explicitly emphasized by the user and the implementation configuration:

- **No Source Code Modification:** The user explicitly stated: *"Do not modify the source code."* This is an absolute constraint. Only the documentation file `blitzy/documentation/app_2cd6ee777f8c.md` may be created.
- **Implementation Rule `SWE-AtlasQnA-Repo`:**
  - The output document must be named `<source_branch_name>.md` — in this case `app_2cd6ee777f8c.md`
  - The document must be placed in the `blitzy/documentation` directory
  - All answers must provide thinking and rationale behind them
  - All answers must be grounded in the actual source code as the single source of truth — no assumptions
  - No existing files in the source repository may be modified
  - No other code may be added to the repository besides the requested document
- **Code-as-Truth Principle:** Every claim in the document must cite a specific file, function, line number, or code pattern from the repository. Speculation or "typical behavior" statements are not permitted.
- **Temporary Data Cleanup:** The user permits creating temporary testing data but requires cleanup. Since this is a documentation-only exercise, this constraint is acknowledged but not exercised.
- **Comprehensive Coverage:** All four question clusters from the user's prompt must be fully answered:
  - Startup verification (web server, email handler, job runner)
  - User workflow verification (sign-in, alias management)
  - Email flow verification (alias receiving email, runtime confirmation)
  - Background component behavior (auto-start, operational indicators)

## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and directories were inspected to derive all conclusions:

| Category | Path | Purpose of Inspection |
|----------|------|-----------------------|
| Entry Point | `server.py` | Web server bootstrap, `/health` endpoint, request logging |
| Entry Point | `email_handler.py` | SMTP handler startup, `handle_forward()`, `handle()` dispatch, status codes |
| Entry Point | `job_runner.py` | Job polling loop, job types, state transitions |
| Entry Point | `cron.py` | Scheduled task definitions |
| Entry Point | `wsgi.py` | Gunicorn WSGI entry |
| Configuration | `pyproject.toml` | Dependencies and versions |
| Configuration | `example.env` | Environment variable reference |
| Configuration | `app/config.py` | Runtime configuration constants |
| Configuration | `Dockerfile` | Container build and Gunicorn CMD |
| Configuration | `crontab.yml` | Cron schedule definitions |
| Authentication | `app/auth/views/login.py` | Login form, credential flow, flash messages |
| Authentication | `app/auth/views/register.py` | Registration form, user creation, activation |
| Dashboard | `app/dashboard/views/index.py` | Dashboard stats, random alias creation |
| Dashboard | `app/dashboard/views/custom_alias.py` | Custom alias creation with validation |
| Models | `app/models.py` | User, Alias, EmailLog, Job, Mailbox, Contact, Notification models |
| Logging | `app/log.py` | Log format, LOG helper |
| Email Status | `app/email/status.py` | SMTP response codes catalog |
| Seed Data | `app/fake_data.py` | Dev seed user and alias data |
| Init | `init_app.py` | SL domain seeding, PGP key loading |
| Monitoring | `monitoring.py` | Postfix queue and DB metrics |
| API | `app/api/` | API endpoint structure and auth |
| Tests | `tests/` | Test directory structure |
| Scripts | `scripts/` | Utility script directory structure |
| Documentation | `README.md` | Self-hosting guide, Docker instructions |
| Documentation | `CONTRIBUTING.md` | Local dev setup, test commands |
| Documentation | `docs/troubleshooting.md` | Diagnostic procedures, swaks testing |
| Documentation | `docs/api.md` | API endpoint reference |
| Folder | `app/auth/views/` | All auth view modules |
| Folder | `app/dashboard/views/` | All dashboard view modules |
| Folder | `app/` | Full application package structure |
| Folder | `migrations/` | Alembic migration directory |
| Folder | `static/` | Static asset directory |
| Folder | `templates/` | Jinja2 template directory |

### 0.8.2 Tech Spec Sections Retrieved

| Section | Purpose |
|---------|---------|
| 1.1 Executive Summary | High-level project context |
| 5.1 High-Level Architecture | Architectural overview and process model |

### 0.8.3 Attachments and External Resources

- **Attachments provided:** None
- **Figma URLs provided:** None
- **Environment files provided:** None
- **Setup instructions provided:** None

