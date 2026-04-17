# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **perform a comprehensive runtime exploration and behavioral verification of the SimpleLogin self-hosted application** — specifically producing a Q&A knowledge document that answers the user's operational questions based on real observed behavior. The user is new to SimpleLogin and needs to understand how to confirm that the three core subsystems (web server, email handler, and job runner) are functioning correctly, what runtime signals indicate healthy operation, and how typical user flows (registration, alias creation, email receipt) manifest at each layer of the system.

- **Requirement 1 — Startup Health Verification:** Determine how to confirm that the web server (Flask/Gunicorn on port 7777), the email handler (`email_handler.py` via aiosmtpd on port 20381), and the job runner (`job_runner.py` with 10-second polling) are actually up and responding after launch.
- **Requirement 2 — Log and Dashboard Readiness Signals:** Identify specific log messages and dashboard UI states that confirm users can sign in and manage their aliases.
- **Requirement 3 — User Action Walkthrough:** Perform basic user actions (create account, create alias, receive email to alias) and document observable system behavior at each stage.
- **Requirement 4 — Background Component Auto-Startup:** Clarify whether the email handler and job runner automatically start alongside the web server or require separate process launch, and what runtime evidence confirms they are functioning across different scenarios.
- **Implicit Requirement — No Source Code Modification:** The user explicitly requires that no source code be modified. Only temporary test data (accounts, aliases) may be created and must be cleaned up when done.
- **Implicit Requirement — Documentation Deliverable:** Per the `SWE-AtlasQnA-Repo` rule, a new markdown document named `<source_branch_name>.md` must be created in the `blitzy/documentation` directory answering all posed questions based on observed code behavior.

### 0.1.2 Special Instructions and Constraints

- **No Code Modification:** The source repository must remain untouched. Only a documentation file is produced.
- **Evidence-Based Answers:** All answers must be grounded in the actual source code and/or observed runtime behavior — no assumptions.
- **Cleanup Required:** Any temporary test data (users, aliases, contacts, email logs) created during runtime exploration must be deleted before completion.
- **Docker Container Environment:** The project uses the Docker image `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` from `ghcr.io/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`.
- **Architectural Convention:** SimpleLogin is a full-stack Python/Flask monolith deployed within a single Docker container but with **five distinct process entry points** that must be started independently: web server (`server.py`/`wsgi.py`), email handler (`email_handler.py`), job runner (`job_runner.py`), cron scheduler (`cron.py` + yacron), and event listener (`event_listener.py`).

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **answer startup health verification questions**, we will inspect `server.py` (the Flask `create_app()` factory and `/health` endpoint), `email_handler.py` (the `main()` function and `MailHandler` class using aiosmtpd `Controller`), and `job_runner.py` (the infinite polling loop with `create_light_app()` context). We will run each process and capture the log output that confirms successful startup.
- To **identify log and dashboard readiness signals**, we will examine `app/log.py` (the `LOG` logger configuration with `set_message_id` lifecycle tracking), the `after_request` logging in `server.py` that prints HTTP method/path/status/timing, the dashboard index view at `app/dashboard/views/index.py` (which displays alias counts, forward/reply/block stats), and the admin panel configured in `server.py` via Flask-Admin.
- To **perform and document user action walkthroughs**, we will exercise the registration flow (`app/auth/views/register.py`), alias creation (via API at `app/api/views/new_random_alias.py` or dashboard at `app/dashboard/views/custom_alias.py`), and email forwarding (SMTP delivery to `email_handler.py` on port 20381), then capture the database records and log outputs at each step.
- To **clarify background component behavior**, we will analyze the process architecture (five independent entry points) and demonstrate that the email handler and job runner are separate `python3` processes that must be launched independently, verifying that the job runner polls the `job` table every 10 seconds and the email handler binds an SMTP socket.


## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

Since this is a Q&A documentation exercise (no code modifications), the scope focuses on reading and analyzing the existing files that answer the user's questions. The following files were inspected to derive conclusions about startup behavior, health signals, user flows, and background component operations.

**Entry Point and Bootstrap Files (Critical to all answers):**

| File | Purpose | Relevance |
|------|---------|-----------|
| `server.py` | Flask application factory (`create_app()`), health endpoint, blueprint registration, admin panel init, request logging | Web server startup confirmation, `/health` endpoint, dashboard routing, login/session setup |
| `wsgi.py` | WSGI entry point exposing the Flask `app` object for Gunicorn | Production web server launch mechanism |
| `email_handler.py` | SMTP inbound processor using aiosmtpd `Controller`, `MailHandler.handle_DATA()`, forward/reply/bounce logic | Email handler startup logs, SMTP socket binding, email forwarding lifecycle |
| `job_runner.py` | Background job polling loop (10s interval), processes onboarding, deletion, batch imports, exports | Job runner startup, job processing behavior, polling frequency |
| `cron.py` | Scheduled maintenance tasks (stats, cleanup, HIBP, subscription checks) | Cron scheduler background processing |
| `event_listener.py` | PostgreSQL LISTEN/NOTIFY consumer for Proton event synchronization | Event-driven background processing |
| `init_app.py` | Seeds SL domains and PGP keys, `add_sl_domains()`, `add_proton_partner()` | Application initialization and data seeding |
| `monitoring.py` | Postfix queue monitoring, DB connection counting, New Relic metric export | Infrastructure health monitoring |

**Authentication and User Management Files:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `app/auth/views/login.py` | Login form, email/password verification, flash messages, rate limiting | Login flow behavior and UI responses |
| `app/auth/views/register.py` | Registration form, user creation, activation email dispatch | Account creation flow |
| `app/auth/views/login_utils.py` | Post-login redirect logic, referral tracking | Dashboard redirect behavior |
| `app/auth/base.py` | Auth blueprint definition (`/auth` prefix) | URL routing for auth endpoints |

**Dashboard and Alias Management Files:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `app/dashboard/views/index.py` | Dashboard homepage with alias listing, stats (forwards/replies/blocks), alias creation | Dashboard UI content after login |
| `app/dashboard/views/custom_alias.py` | Custom alias creation form with suffix selection and mailbox assignment | Alias creation flow via web UI |
| `app/api/views/new_random_alias.py` | API endpoint for random alias generation (`POST /api/alias/random/new`) | Alias creation via API |
| `app/api/views/alias.py` | Alias listing, toggle, and management API endpoints | Alias activity queries |
| `app/api/base.py` | API authentication via `Authentication` header and `ApiKey` model | API key validation |

**Core Application Infrastructure Files:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `app/config.py` | Environment variable loading, configuration constants (DB_URI, EMAIL_DOMAIN, etc.) | Application configuration behavior |
| `app/models.py` | SQLAlchemy ORM models (User, Alias, Contact, EmailLog, Job, Mailbox, etc.) | Data model structure for all operations |
| `app/db.py` | SQLAlchemy engine, session management | Database connectivity |
| `app/log.py` | Logger configuration, `EmailHandlerFilter` for message-id tracking | Log format and output structure |
| `app/fake_data.py` | Seed data function (demo users, aliases, subscriptions, API keys) | Test data setup mechanism |
| `app/extensions.py` | Flask-Login and Flask-Limiter initialization | Extension bootstrapping |

**Email Processing Files:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `app/email_utils.py` | Email rendering, header manipulation, DKIM, VERP, delivery helpers | Email composition and delivery |
| `app/mail_sender.py` | SMTP delivery, unsent-message persistence/retry, `sl_sendmail()` | Outbound email sending |
| `app/handler/dmarc.py` | DMARC policy enforcement for forward and reply phases | Email security processing |
| `app/handler/unsubscribe_handler.py` | Unsubscribe request processing | Email management actions |
| `app/contact_utils.py` | Contact creation/update when emails arrive | Contact record creation during forwarding |
| `app/email/spam.py` | SpamAssassin/Rspamd integration | Spam scoring during email handling |

**Configuration and Deployment Files:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `example.env` | Reference for all environment variables with documented defaults | Configuration reference |
| `Dockerfile` | Multi-stage build (Node + Python), exposes port 7777, Gunicorn CMD | Container build and runtime setup |
| `pyproject.toml` | Poetry dependency manifest (Python 3.10, Flask 1.x, SQLAlchemy 1.3.24, etc.) | Dependency version reference |
| `alembic.ini` | Alembic migration configuration pointing to `migrations/` | Database migration setup |
| `crontab.yml` | Yacron scheduled job definitions (16+ tasks) | Cron scheduling reference |

### 0.2.2 Web Search Research Conducted

No external web search was necessary for this task. All answers were derived from:
- Direct source code inspection using repository tools
- Runtime execution of the three core processes (web server, email handler, job runner)
- Database queries against the live PostgreSQL instance
- API calls to the running Flask application
- SMTP test messages to the running email handler

### 0.2.3 New File Requirements

Per the `SWE-AtlasQnA-Repo` implementation rule, a single new markdown document will be created:

- **`blitzy/documentation/<source_branch_name>.md`** — A comprehensive Q&A document answering all user questions about SimpleLogin startup verification, runtime behavior, user action walkthroughs, and background component operation. This document is based entirely on source code analysis and observed runtime behavior.

No other new source, test, or configuration files are required since the task explicitly forbids source code modification.


## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

The following packages are relevant to the runtime exploration and behavioral verification task. All versions are sourced from `pyproject.toml` (the Poetry dependency manifest) and verified against the installed environment.

| Registry | Package | Version | Purpose |
|----------|---------|---------|---------|
| PyPI | python | ^3.10 | Runtime language (Dockerfile uses `python:3.10` base image) |
| PyPI | flask | ^1.1.2 | Web framework for dashboard, API, auth, and admin |
| PyPI | gunicorn | ^20.0.4 | WSGI HTTP server (production entry point) |
| PyPI | SQLAlchemy | 1.3.24 (pinned) | ORM for all database operations |
| PyPI | psycopg2-binary | ^2.9.3 | PostgreSQL adapter |
| PyPI | aiosmtpd | ^1.2 | Async SMTP server for inbound email handling |
| PyPI | flask-login | ^0.5.0 | User session management and authentication |
| PyPI | flask-admin | ^1.5.6 | Admin panel for system management |
| PyPI | flask-cors | ^3.0.9 | CORS support for API endpoints |
| PyPI | Flask-Limiter | ^1.4 | Rate limiting for login, alias creation, and API |
| PyPI | Flask-Migrate | ^2.5.3 | Alembic migration integration with Flask |
| PyPI | redis | ^4.5.3 | Session store, rate limiting, and concurrency locks |
| PyPI | sentry-sdk | ^2.16.0 | Error tracking and performance monitoring |
| PyPI | newrelic | 8.8.0 (pinned) | APM agent for performance monitoring |
| PyPI | arrow | ^0.16.0 | Date/time handling throughout the application |
| PyPI | bcrypt | ^3.2.0 | Password hashing for user authentication |
| PyPI | dkimpy | ^1.0.5 | DKIM signature verification and signing |
| PyPI | pyotp | ^2.4.0 | TOTP/HOTP for MFA |
| PyPI | pyre2 | ^0.3.6 | Fast regex matching for email processing |
| PyPI | email-validator | ^1.1.1 | Email address validation |
| PyPI | yacron | ^0.11.1 | YAML-configured cron scheduler |
| PyPI | Flask-WTF | ^0.14.3 | CSRF protection and form handling |
| PyPI | python-gnupg | ^0.4.6 | GPG/PGP key management for email encryption |
| PyPI | PGPy | 0.5.4 (pinned) | PGP encryption library (compatible with cryptography 37.0.1) |
| PyPI | cryptography | 37.0.1 (pinned) | Cryptographic primitives (version pinned for PGPy compatibility) |
| PyPI | flanker | ^0.9.11 | Email address parsing library |
| PyPI | coloredlogs | ^14.0 | Colored log output for development mode |
| npm | (static assets) | per package-lock.json | Frontend CSS/JS assets (installed during Docker build) |

### 0.3.2 Dependency Updates

No dependency updates are required. This task is a read-only behavioral analysis that produces only a documentation file. All existing dependencies remain at their current versions as specified in `pyproject.toml` and `poetry.lock`.

### 0.3.3 External Reference Updates

No external references require updates. The existing `example.env`, `README.md`, and `Dockerfile` remain unchanged.


## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

Since this task produces only a documentation file and makes no source code modifications, the integration analysis focuses on the **read-only touchpoints** that were examined to derive behavioral answers.

**Web Server Integration Points (Examined for Startup and Health Signals):**
- `server.py` → `create_app()` (line 139): Flask application factory that wires all extensions, blueprints, error handlers, admin panel, and the `/health` endpoint (line 213-215)
- `wsgi.py` (line 1-3): Production WSGI entry point that calls `create_app()` and exposes the `app` object for Gunicorn
- `server.py` → `register_blueprints()` (line 233): Registers auth_bp, dashboard_bp, api_bp, oauth_bp, monitor_bp, developer_bp, phone_bp, onboarding_bp, discover_bp, internal_bp
- `server.py` → `set_index_page()` (line 249): Root `/` redirect logic — authenticated users go to dashboard, unauthenticated go to login
- `server.py` → `after_request()` (line 272): Request logging with method, path, args, status code, and response time

**Email Handler Integration Points (Examined for SMTP Behavior):**
- `email_handler.py` → `main()` (line 2381): Creates aiosmtpd `Controller` on `0.0.0.0:20381` (default), starts the SMTP server, and enters an infinite `time.sleep(2)` loop
- `email_handler.py` → `MailHandler.handle_DATA()` (line 2289): Entry point for each inbound email
- `email_handler.py` → `_handle()` (line 2335): Creates a `create_light_app()` context, generates a unique `message_id`, invokes `handle()`, and logs the full processing lifecycle with timing
- `email_handler.py` → `handle()` (line ~1950): Main email routing logic — determines forward vs. reply vs. bounce, resolves aliases, creates contacts, applies DMARC/spam checks, and dispatches delivery

**Job Runner Integration Points (Examined for Background Processing):**
- `job_runner.py` → `__main__` (line 329-347): Infinite loop with `create_light_app().app_context()`, calls `get_jobs_to_run()`, marks jobs as taken, calls `process_job()`, marks done, then `time.sleep(10)`
- `job_runner.py` → `get_jobs_to_run()` (line 307): Queries the `job` table for jobs in `ready` state or `taken` state that have been stuck for more than `JOB_TAKEN_RETRY_WAIT_MINS` minutes with fewer than `JOB_MAX_ATTEMPTS` attempts
- `job_runner.py` → `process_job()` (line 188): Dispatches by job name — handles onboarding emails, batch imports, account/mailbox/domain deletion, data export, proton welcome, and alias creation events

**Database/Schema Integration Points (Examined for Data Verification):**
- `app/models.py` → `User`, `Alias`, `Contact`, `EmailLog`, `Job`, `Mailbox`, `ApiKey`, `SLDomain`: Core ORM models used to verify that user actions create expected database records
- `migrations/` → Alembic revision chain: Used to initialize the database schema via `alembic upgrade head`
- `app/db.py` → `Session`: Scoped session proxy used by all processes for database operations

### 0.4.2 Dependency Injections

The following service injection points were observed during runtime analysis:

- `server.py` → `init_extensions()` (line 437): Registers Flask-Login via `login_manager.init_app(app)`
- `server.py` → `limiter.init_app(app)` (line 167): Initializes Flask-Limiter for rate control
- `server.py` → `initialize_redis_services(app, MEM_STORE_URI)` (line 165): Wires Redis for sessions and rate limiting (only when `MEM_STORE_URI` is set)
- `server.py` → `init_admin(app)` (line 179): Registers all Flask-Admin model views (User, Alias, Mailbox, etc.)
- `server.py` → `setup_paddle_callback(app)` and `setup_coinbase_commerce(app)`: Payment webhook handlers

### 0.4.3 Database/Schema State

The database was initialized using `alembic upgrade head` which applied the full migration chain from the `migrations/` directory. Key tables verified during runtime:

| Table | Verified Content |
|-------|-----------------|
| `users` | Seed user `john@wick.com` (admin) and `winston@continental.com` created by `fake_data()` |
| `alias` | Multiple aliases on `sl.local` domain created by seed and API test |
| `contact` | Contact records created automatically when emails arrive at aliases |
| `email_log` | Forward/bounce records created during email handling |
| `job` | Empty during test (onboarding disabled); polled every 10s by job runner |
| `sl_domain` | `sl.local` domain registered by `init_app.py` → `add_sl_domains()` |
| `notification` | Seed notifications created by `fake_data()` for dashboard display |


## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since this task produces only a Q&A documentation file, the execution plan centers on a single deliverable. No existing files are modified.

**Group 1 — Documentation Deliverable:**
- **CREATE:** `blitzy/documentation/<source_branch_name>.md` — Comprehensive markdown document answering all user questions about SimpleLogin startup verification, runtime behavior, user workflows, and background component operation

The document content is derived entirely from source code analysis and observed runtime behavior as described in the implementation approach below.

### 0.5.2 Implementation Approach

The documentation file will be structured to answer each of the user's questions in sequence, using evidence gathered from the following runtime activities:

**Step 1 — Environment Setup and Application Launch:**
- Install PostgreSQL 16, Redis, and all Python dependencies from `pyproject.toml`
- Set environment variables per `example.env` (DB_URI, EMAIL_DOMAIN=sl.local, NOT_SEND_EMAIL=true, etc.)
- Run `alembic upgrade head` to initialize the database schema
- Seed demo data via `fake_data()` and `add_sl_domains()` from `init_app.py`

**Step 2 — Web Server Verification:**
- Start Gunicorn with `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`
- Confirm health via `curl http://localhost:7777/health` → expect `success` with HTTP 200
- Verify login page renders at `/auth/login`
- Test login with seeded user `john@wick.com` / `password` → expect 302 redirect to `/dashboard/`
- Capture and document the `after_request` log pattern: `{remote_addr} {method} {path} {args} {status_code}, takes {elapsed}`

**Step 3 — Email Handler Verification:**
- Start email handler with `python3 email_handler.py -p 20381`
- Confirm startup log: `"Listen for port 20381"` followed by `"Start mail controller 0.0.0.0 20381"`
- Send test email via SMTP to an alias address
- Capture the full forwarding log lifecycle: `"New message, mail from ... rctp tos ..."` → `"Forward phase"` → `"Create or get contact"` → `"Forward mail from ... to ..."` → `"250 Message accepted for delivery"`

**Step 4 — Job Runner Verification:**
- Start job runner with `python3 job_runner.py`
- Observe the 10-second polling loop (the runner silently polls the `job` table)
- Note that with `DISABLE_ONBOARDING=true`, new user creation does not enqueue onboarding jobs
- Verify job processing by examining the `job` table in PostgreSQL

**Step 5 — User Action Walkthrough:**
- Create a test user via the Python API (`User.create(email='testuser@example.com', ...)`)
- Create a random alias via `POST /api/alias/random/new` with an API key
- Send an email to the alias via SMTP
- Query alias activity via `GET /api/v2/aliases?page_id=0` to confirm `nb_forward` incremented and `latest_activity` populated
- Verify database records: `contact` row created, `email_log` row with `bounced=False`, `blocked=False`, `is_reply=False`

**Step 6 — Cleanup:**
- Delete all test user data (email_logs, contacts, aliases, API keys, user record)
- Verify cleanup via database queries

### 0.5.3 Key Observations for the Q&A Document

Based on the runtime exploration, the following key findings will be documented:

- **Web Server Health:** The `/health` endpoint at `server.py:213-215` returns `"success"` with HTTP 200. The `after_request` hook logs every non-static request with timing.
- **Dashboard UI Confirmation:** After login, the dashboard at `/dashboard/` shows the user's name, alias count, and activity statistics (forwards, replies, blocks). The `get_stats()` function in `app/dashboard/views/index.py` queries `EmailLog` for these counts.
- **Email Handler Startup:** The `main()` function in `email_handler.py:2381` creates an aiosmtpd `Controller` and logs `"Start mail controller 0.0.0.0 {port}"`. Each incoming email generates a unique `message_id` UUID and produces a detailed log trail from receipt to delivery.
- **Job Runner Behavior:** The job runner runs in an infinite loop with `time.sleep(10)` between iterations. It silently polls when no jobs are available. Jobs appear in the `job` table when user actions trigger them (e.g., account deletion, mailbox deletion, batch imports).
- **Process Independence:** The email handler and job runner are completely independent processes that must be started separately. They do not auto-start with the web server. Each creates its own Flask app context via `create_light_app()` for database access.
- **Email NOT_SEND_EMAIL Mode:** When `NOT_SEND_EMAIL=true`, the email handler processes messages through the full pipeline (contact creation, email log recording, header rewriting) but does not actually deliver via SMTP. This is the standard local development configuration.


## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Documentation deliverable:**
- `blitzy/documentation/<source_branch_name>.md` — The sole output artifact

**Source files analyzed for Q&A content (read-only):**
- `server.py` — Web server factory, health endpoint, request logging, blueprint registration
- `wsgi.py` — WSGI entry point for Gunicorn
- `email_handler.py` — SMTP handler startup, `MailHandler` class, forward/reply/bounce logic
- `job_runner.py` — Job polling loop, job state management, job type dispatching
- `init_app.py` — Domain seeding, PGP key loading
- `monitoring.py` — Infrastructure metric collection
- `cron.py` — Scheduled maintenance task definitions
- `crontab.yml` — Yacron schedule configuration
- `example.env` — Environment variable reference
- `Dockerfile` — Container build and runtime configuration
- `pyproject.toml` — Dependency manifest
- `alembic.ini` — Database migration configuration
- `app/config.py` — Configuration constant loading
- `app/log.py` — Logger setup and message-id tracking
- `app/db.py` — Database engine and session management
- `app/models.py` — ORM model definitions (User, Alias, Contact, EmailLog, Job, etc.)
- `app/fake_data.py` — Seed data generation
- `app/extensions.py` — Flask-Login and Flask-Limiter bootstrapping
- `app/auth/views/login.py` — Login form and authentication logic
- `app/auth/views/register.py` — Registration form and user creation
- `app/auth/views/login_utils.py` — Post-login redirect behavior
- `app/auth/base.py` — Auth blueprint definition
- `app/dashboard/views/index.py` — Dashboard homepage with stats and alias management
- `app/dashboard/views/custom_alias.py` — Custom alias creation
- `app/dashboard/base.py` — Dashboard blueprint definition
- `app/api/views/new_random_alias.py` — Random alias creation API
- `app/api/views/alias.py` — Alias listing and management API
- `app/api/base.py` — API authentication and authorization
- `app/api/serializer.py` — API response serialization and query helpers
- `app/email_utils.py` — Email rendering and delivery utilities
- `app/mail_sender.py` — SMTP delivery with persistence/retry
- `app/contact_utils.py` — Contact creation on email receipt
- `app/handler/dmarc.py` — DMARC policy enforcement
- `app/handler/unsubscribe_generator.py` — Unsubscribe header generation
- `app/email/spam.py` — Spam scoring integration

**Runtime activities performed (read-only, with cleanup):**
- PostgreSQL database initialization via `alembic upgrade head`
- Seed data loading via `fake_data()` and `add_sl_domains()`
- Gunicorn web server started on port 7777 and tested
- Email handler started on port 20381 and tested with SMTP messages
- Job runner started and observed polling behavior
- API calls for alias creation and alias activity verification
- SMTP test messages sent to verify email forwarding pipeline
- Test user and associated data created and cleaned up

### 0.6.2 Explicitly Out of Scope

- **Source code modifications** — No changes to any existing file in the repository
- **New feature implementation** — No new functionality added
- **Performance optimization** — No benchmarking or tuning
- **Cron scheduler testing** — The `cron.py` / yacron system was not fully exercised (documented from source only)
- **Event listener testing** — The `event_listener.py` PostgreSQL LISTEN/NOTIFY system was not exercised (requires Proton webhook configuration)
- **Payment integration testing** — Paddle, Coinbase, and Apple subscription flows were not tested
- **OAuth/OIDC provider testing** — Third-party social login flows were not exercised
- **PGP encryption testing** — Email PGP encryption was not tested (requires GPG key configuration)
- **SpamAssassin/Rspamd testing** — Spam scoring was not tested (requires SpamAssassin service)
- **Production deployment concerns** — Nginx reverse proxy, TLS certificates, DNS configuration
- **Refactoring of existing code** — No structural changes


## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The user has specified the following rules via the `SWE-AtlasQnA-Repo` implementation rule:

- **Create a new markdown document** named `<source_branch_name>.md` that comprehensively answers the question(s) posed in the prompt
- **Build and run the source code** to analyze the repository behavior as needed
- **Do not make assumptions** — base all answers on the code as the truth
- **Provide thinking / rationale** behind the answers
- **Do not modify any existing files** in the source repository
- **Do not add any other code** in the source repository (besides the above-requested document)
- **Place the generated document** in the `blitzy/documentation` directory in the destination repo

### 0.7.2 Operational Constraints

- **No code modification:** The `SWE-AtlasQnA-Repo` rule explicitly forbids modifying existing repository files or adding code files. Only the markdown Q&A document is permitted.
- **Evidence-based documentation:** Every answer in the Q&A document must reference specific source code locations (file paths, line numbers, function names) or observed runtime behavior (log output, HTTP responses, database records).
- **Temporary data cleanup:** Any test users, aliases, contacts, or email logs created during runtime exploration must be deleted before the task is considered complete.
- **Environment variable consistency:** All runtime testing must use environment variables consistent with `example.env` to ensure answers reflect the standard self-hosting configuration.
- **Process isolation awareness:** Answers must clearly explain that the web server, email handler, and job runner are independent processes requiring separate launch commands — this is a common point of confusion for new users.


## 0.8 References

### 0.8.1 Files and Folders Searched

The following files and folders were comprehensively inspected to derive conclusions for the Agent Action Plan:

**Root-Level Files Inspected:**
- `server.py` — Flask application factory, health check, request logging, blueprint registration
- `wsgi.py` — WSGI entry point for Gunicorn
- `email_handler.py` — SMTP inbound handler (lines 1-2405 examined, key sections: 1-180 imports, 2288-2405 MailHandler and main)
- `job_runner.py` — Background job processing (full file, 348 lines)
- `init_app.py` — Application initialization and data seeding (full file, 74 lines)
- `monitoring.py` — Infrastructure monitoring and New Relic metrics (full file, 172 lines)
- `cron.py` — Cron job definitions (lines 1-60 examined)
- `crontab.yml` — Yacron schedule configuration (full file, 97 lines)
- `example.env` — Environment variable reference (full file, 198 lines)
- `Dockerfile` — Container build configuration (full file, 47 lines)
- `pyproject.toml` — Poetry dependency manifest (full file, 135 lines)
- `alembic.ini` — Database migration configuration (first 20 lines)

**Application Package Files Inspected:**
- `app/` folder structure — All children enumerated and categorized
- `app/config.py` — Configuration loading (lines 1-80)
- `app/log.py` — Logger configuration (full file, 80 lines)
- `app/fake_data.py` — Seed data generation (full file, 272 lines)
- `app/auth/` folder — Structure and summaries
- `app/auth/views/login.py` — Login view (full file, 83 lines)
- `app/auth/views/register.py` — Registration view (full file, 130 lines)
- `app/auth/base.py` — Auth blueprint definition
- `app/dashboard/` folder — Structure and summaries
- `app/dashboard/views/index.py` — Dashboard homepage (lines 32-100)
- `app/dashboard/views/custom_alias.py` — Custom alias creation (lines 1-60)
- `app/dashboard/base.py` — Dashboard blueprint definition
- `app/api/` folder — Structure and summaries
- `app/api/views/new_random_alias.py` — Random alias API (file reference identified)
- `app/api/views/alias.py` — Alias management API (file reference identified)
- `app/api/base.py` — API authentication (full summary examined)
- `app/api/serializer.py` — API serialization (full summary examined)

**Tech Spec Sections Retrieved:**
- Section 1.1 Executive Summary — Project overview, core business problem, stakeholders, value proposition
- Section 5.1 High-Level Architecture — System overview, core components, data flows, external integrations
- Section 4.1 System Workflow Overview — Entry points, request routing, core processing pipeline, timing constraints

### 0.8.2 Runtime Verification Activities

| Activity | Method | Result |
|----------|--------|--------|
| Web server health check | `curl http://localhost:7777/health` | HTTP 200 `success` |
| Login page rendering | `curl http://localhost:7777/auth/login` | HTTP 200, HTML login form with CSRF token |
| User login (john@wick.com) | POST to `/auth/login` with CSRF token | HTTP 302 redirect to `/dashboard/` |
| Dashboard access | `curl http://localhost:7777/dashboard/` with session cookie | HTTP 200, shows user name "John Wick" and alias list |
| API alias listing | `GET /api/v2/aliases?page_id=0` with `Authentication: code` header | HTTP 200, JSON array of aliases with metadata |
| Random alias creation | `POST /api/alias/random/new` with `Authentication: testkey123` header | HTTP 201, created `salute_iodide498@sl.local` |
| SMTP email delivery | Python `smtplib` to localhost:20381 | SMTP 250, email processed through full forwarding pipeline |
| Email handler log verification | Examined `/tmp/email_handler.log` | Full lifecycle logged: receipt → contact creation → header rewriting → delivery |
| Job runner startup | `python3 job_runner.py` as background process | Process started, silently polling every 10 seconds |
| Database record verification | PostgreSQL queries on `email_log`, `contact`, `alias`, `notification` tables | All expected records present |
| Test data cleanup | Python script deleting test user and all associated records | Cleanup verified via database queries |

### 0.8.3 Attachments

No attachments were provided by the user for this project. No Figma designs, wireframes, or external documentation files were referenced.


