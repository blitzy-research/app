# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the user's requirement is to perform a comprehensive **operational verification and behavioral walkthrough** of a locally-running SimpleLogin instance. This is a **read-only, observational exercise** — not a code-modification task — that produces a detailed reference document answering three interconnected questions about the platform's runtime behavior:

- **Application Readiness Verification:** Identify the specific log messages, UI responses, and system indicators that confirm SimpleLogin's Flask web application, PostgreSQL database, Redis session store, Postfix mail transport, and background workers are all initialized and ready to handle user authentication and alias-based email activity.

- **End-to-End New-User Flow Walkthrough:** Trace the complete product experience from the perspective of a first-time user — registration at `/auth/register`, email activation via the `ActivationCode` model, login at `/auth/login` with credential validation and MFA decision routing, and the subsequent redirect into the `/dashboard/` alias management interface — documenting the visible UI behavior and HTTP lifecycle at each step.

- **Background Service Runtime Observability:** Surface the behind-the-scenes indicators that prove internal services — the `job_runner.py` onboarding job scheduler, the `email_handler.py` SMTP inbound processor, the `event_listener.py` event-sourcing pipeline, the `cron.py` scheduled maintenance tasks, and the `monitoring.py` metric exporter — are active, polling, and communicating with one another correctly during the new-user registration flow.

The deliverable is a single markdown document placed in `blitzy/documentation/app_2cd6ee777f8c.md` that comprehensively answers these questions with evidence drawn directly from the source code.

### 0.1.2 Implicit Requirements Detected

- The user expects answers grounded in **actual code behavior** (e.g., `server.py:create_app()`, `app/auth/views/register.py`, `app/models.py:User.create()`), not generic descriptions.
- The phrase "confirm that the platform is actually working" implies the answer must identify **concrete, observable signals** — specific log format strings from `app/log.py`, HTTP status codes, Flask flash messages, database row creation, and SMTP log entries.
- "Walk through the typical product experience" implies a **sequential, step-by-step narrative** following the Flask blueprint routing from `auth_bp` through `dashboard_bp`.
- "Behind the scenes during that flow" requires tracing the **Job model scheduling** (onboarding-1 through onboarding-4 jobs created in `app/models.py` lines 651–665), the event dispatcher pathway (`app/events/event_dispatcher.py`), and the `RegisterEvent`/`LoginEvent` telemetry hooks (`app/events/auth_event.py`).
- The directive "create anything temporary that you need for testing" paired with "remove them when you are done" means the document should describe the **ephemeral nature of test data** and the cleanup expectations.
- "Do not make any changes to the source code" is an absolute constraint that must be honored.

### 0.1.3 Special Instructions and Constraints

- **Implementation Rule — SWE-AtlasQnA-Repo:** Create a new markdown document named `app_2cd6ee777f8c.md` in the `blitzy/documentation` directory that comprehensively answers the user's questions.
- **No Source Code Modifications:** The document must not recommend or perform any changes to existing repository files.
- **Evidence-Based Answers:** All claims must be grounded in specific files and code paths observed in the repository.
- **Temporary Data:** Any references to test users or aliases must note that they are ephemeral and should be cleaned up.

### 0.1.4 Technical Interpretation

These requirements translate to the following technical implementation strategy:

- To **answer the application readiness question**, we will analyze `server.py:create_app()` (Flask app factory), `app/config.py` (environment variable loading with print statements like `>>> URL:`), `app/log.py` (logger initialization with `>>> init logging <<<`), `app/db.py` (SQLAlchemy engine creation), `app/extensions.py` (Flask-Login and Limiter setup), `app/redis_services.py` (Redis/Sentinel initialization), and the `/health` endpoint (returns `"success", 200`).

- To **trace the new-user registration flow**, we will follow `app/auth/views/register.py` (form validation → `User.create()` → `send_activation_email()` → render `register_waiting_activation.html`), then `app/auth/views/activate.py` (code validation → `user.activated = True` → `login_user()` → `send_welcome_email()` → redirect to `dashboard.index`), and then `app/auth/views/login.py` (credential check → `after_login()` decision tree → MFA routing or direct dashboard redirect).

- To **document background service indicators**, we will examine `job_runner.py` (10-second polling loop, job state machine with `JobState.ready` → `taken` → `done`), `cron.py` and `crontab.yml` (yacron-scheduled maintenance), `email_handler.py:main()` (aiosmtpd Controller on port 20381), `event_listener.py` (PostgreSQL LISTEN/NOTIFY on `simplelogin_sync_events`), and `monitoring.py` (60-second metric export loop).

- To **produce the deliverable**, we will create `blitzy/documentation/app_2cd6ee777f8c.md` containing the comprehensive answers.

## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

Since this is a read-only Q&A exercise that produces a documentation artifact, the "affected files" are those that must be **analyzed** to derive accurate answers, plus the single new file that will be created. No existing files are modified.

#### Files Analyzed for Application Startup and Readiness

| File | Purpose in Analysis | Key Evidence |
|------|---------------------|--------------|
| `server.py` | Flask app factory, blueprint registration, health endpoint | `create_app()` wires all extensions, `register_blueprints()` attaches auth/dashboard/api, `/health` returns `"success", 200` |
| `wsgi.py` | Production WSGI entry point | `app = create_app()` — single-line Gunicorn target |
| `app/config.py` | Environment variable loading with startup print statements | `print(">>> URL:", URL)` at line 80, `MAX_NB_EMAIL_FREE_PLAN` fallback print at line 123 |
| `app/log.py` | Logger initialization | `print(">>> init logging <<<")` at line 67, `LOG = _get_logger("SL")` at line 79 |
| `app/db.py` | SQLAlchemy engine and scoped session creation | `engine = create_engine(config.DB_URI, ...)` at line 9, `Session = scoped_session(...)` at line 14 |
| `app/extensions.py` | Flask-Login and Flask-Limiter bootstrap | `login_manager = LoginManager()` with `session_protection = "strong"`, `limiter = Limiter(key_func=__key_func)` |
| `app/redis_services.py` | Redis/Sentinel connection initialization | Called conditionally from `server.py` line 165 when `MEM_STORE_URI` is set |
| `example.env` | Reference configuration for local development | `URL=http://localhost:7777`, `NOT_SEND_EMAIL=true`, `DB_URI=postgresql://...`, `EMAIL_DOMAIN=sl.local` |
| `Dockerfile` | Container build showing Python 3.10, Poetry install, Gunicorn CMD | Exposes port 7777, uses `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2` |

#### Files Analyzed for Registration → Activation → Login Flow

| File | Purpose in Analysis | Key Evidence |
|------|---------------------|--------------|
| `app/auth/views/register.py` | Registration endpoint and `send_activation_email()` | `RegisterForm` with email + password (min 8 chars), `User.create()` call at line 86, `ActivationCode.create()` at line 120, `render("register_waiting_activation.html")` at line 104 |
| `app/auth/views/activate.py` | Activation code validation and user login | `user.activated = True` at line 49, `login_user(user)` at line 50, `send_welcome_email(user)` at line 58, redirect to `dashboard.index` at line 67 |
| `app/auth/views/login.py` | Login endpoint with credential checking | `LoginForm`, `User.get_by(email=...)` lookup, `user.check_password()` at line 45, `after_login()` call at line 72 |
| `app/auth/views/login_utils.py` | Post-login MFA decision tree and session setup | `after_login()` checks `user.fido_enabled()` → `user.enable_otp` → direct `login_user()`, sets `session["sudo_time"]` at line 37 |
| `app/models.py` (User class) | User model with `create()` classmethod | Lines 601–668: creates User → creates Mailbox → creates first alias (`simplelogin-newsletter`) → schedules onboarding Jobs (1–4) |
| `app/models.py` (ActivationCode) | Activation code model with 1-hour expiry | Lines 1202–1215: `code` column (30-char random string), `is_expired()` check |
| `app/email_utils.py` | Email sending functions | `send_activation_email()` at line 125, `send_welcome_email()` at line 97, `send_email()` at line 289 |
| `app/pw_models.py` | Password hashing with bcrypt | `PasswordOracle` mixin used by User class |
| `app/events/auth_event.py` | Registration/login telemetry events | `RegisterEvent` and `LoginEvent` with New Relic custom event emission |

#### Files Analyzed for Background Services and Runtime Observability

| File | Purpose in Analysis | Key Evidence |
|------|---------------------|--------------|
| `job_runner.py` | Continuous job polling and processing loop | 10-second sleep interval, `get_jobs_to_run()` with state machine, `process_job()` dispatch for onboarding/delete/batch-import jobs |
| `cron.py` | Scheduled maintenance functions | `stats`, `delete_old_monitoring`, `check_custom_domain`, `check_hibp`, `notify_trial_end`, `send_undelivered_mails`, and more |
| `crontab.yml` | yacron job schedule definitions | 15 scheduled jobs including 5-minute unsent-mail retry, daily stats, hourly audit log cleanup |
| `email_handler.py` | SMTP inbound mail processor | `MailHandler` class with `handle_DATA()`, aiosmtpd `Controller` on port 20381, `handle()` routing function at line 1945 |
| `event_listener.py` | Event sourcing CLI with PostgreSQL LISTEN/NOTIFY | `PostgresEventSource` and `DeadLetterEventSource` modes, `Runner` orchestration |
| `monitoring.py` | Postfix queue and DB connection metric export | `log_postfix_metrics()`, `log_nb_db_connection()`, `log_pending_to_process_events()`, 60-second loop |
| `init_app.py` | Domain seeding and PGP key loading | `add_sl_domains()`, `load_pgp_public_keys()`, `add_proton_partner()` |
| `app/mail_sender.py` | SMTP delivery with `NOT_SEND_EMAIL` short-circuit | `MailSender.send()` logs subject/from/to when `config.NOT_SEND_EMAIL` is True (line 131) |
| `app/events/event_dispatcher.py` | Protobuf event dispatch to PostgreSQL `sync_event` | `PostgresDispatcher` with `NOTIFY simplelogin_sync_events`, `EventDispatcher.send_event()` |
| `app/fake_data.py` | Demo data seeding (user `john@wick.com`, aliases, contacts) | `fake_data()` creates admin user, aliases, contacts, email logs, OAuth clients |

#### Files Analyzed for Dashboard Behavior

| File | Purpose in Analysis | Key Evidence |
|------|---------------------|--------------|
| `app/dashboard/views/index.py` | Dashboard landing page with alias stats | `get_stats()` computes `nb_alias`, `nb_forward`, `nb_reply`, `nb_block`; alias creation and deletion handlers |
| `app/dashboard/base.py` | Dashboard blueprint with `/dashboard` URL prefix | `dashboard_bp = Blueprint("dashboard", ...)` |

### 0.2.2 New File Requirements

A single new markdown document will be created as the deliverable:

| File Path | Purpose |
|-----------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | Comprehensive Q&A document answering user's questions about SimpleLogin's runtime behavior, authentication flow walkthrough, and background service observability |

No new source files, test files, configuration files, or migration files are required. This is purely a documentation exercise.

### 0.2.3 Web Search Research Conducted

No external web research is required for this task. All answers are derived directly from the source code analysis of the SimpleLogin repository. The codebase is self-contained and provides all necessary evidence for:
- Application startup sequences and log output patterns
- Authentication flow mechanics (registration, activation, login)
- Background job scheduling and event processing architecture
- Email forwarding pipeline and SMTP handler behavior

## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

Since this task produces only a documentation artifact and does not modify code or add dependencies, the dependency inventory focuses on the packages **relevant to the user's questions** — those that underpin the startup, authentication, email processing, and background job systems being documented.

| Registry | Package | Version | Purpose in Context |
|----------|---------|---------|-------------------|
| PyPI | python | ^3.10 | Runtime language (Dockerfile uses `python:3.10` base image) |
| PyPI | flask | ^1.1.2 | Web framework powering all auth/dashboard routes |
| PyPI | flask_login | ^0.5.0 | Session management, `login_user()`, `current_user`, `@login_required` |
| PyPI | gunicorn | ^20.0.4 | Production WSGI server (Dockerfile CMD: `gunicorn wsgi:app -b 0.0.0.0:7777`) |
| PyPI | SQLAlchemy | 1.3.24 | ORM layer for User, Alias, ActivationCode, Job, Mailbox models |
| PyPI | psycopg2-binary | ^2.9.3 | PostgreSQL database driver |
| PyPI | Flask-Migrate | ^2.5.3 | Alembic-based database migration management |
| PyPI | bcrypt | ^3.2.0 | Password hashing via `pw_models.py:PasswordOracle` |
| PyPI | aiosmtpd | ^1.2 | Async SMTP server for `email_handler.py` inbound mail processing |
| PyPI | arrow | ^0.16.0 | Timestamp handling for job scheduling, activation code expiry |
| PyPI | sentry_sdk | ^2.16.0 | Error tracking with Flask and SQLAlchemy integrations |
| PyPI | newrelic | 8.8.0 | APM for custom metrics (login/register events, email handler timing) |
| PyPI | redis | ^4.5.3 | Session storage and rate limiting backend (when `MEM_STORE_URI` set) |
| PyPI | flask-cors | ^3.0.9 | CORS on `/api/*` endpoints |
| PyPI | Flask-Limiter | ^1.4 | Rate limiting on auth and dashboard routes |
| PyPI | yacron | ^0.11.1 | Cron job scheduler reading `crontab.yml` |
| PyPI | dkimpy | ^1.0.5 | DKIM signature generation for outbound emails |
| PyPI | flask_admin | ^1.5.6 | Admin panel interface |
| PyPI | Flask-WTF | ^0.14.3 | CSRF protection and form validation for registration/login forms |
| PyPI | pyotp | ^2.4.0 | TOTP MFA code generation and verification |
| PyPI | webauthn | ^0.4.7 | FIDO/WebAuthn hardware key authentication |
| PyPI | coloredlogs | ^14.0 | Colored log output for local development (when `COLOR_LOG=true`) |
| PyPI | python-dotenv | ^0.14.0 | `.env` file loading in `app/config.py` |
| PyPI | email_validator | ^1.1.1 | Email address validation for registration |
| npm | (static assets) | via `package.json` | Frontend JS/CSS assets (built in Dockerfile npm stage) |

### 0.3.2 Dependency Updates

No dependency updates are required. This task creates a standalone markdown document and does not alter any imports, package manifests, or build configurations. All dependency versions listed above are sourced directly from `pyproject.toml` in the repository.

## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

No existing code is modified by this task. However, the following integration points are **documented and analyzed** in the deliverable markdown file to answer the user's questions:

#### Application Startup Integration Chain

The startup sequence involves a chain of module-level initializations that the document must trace:

- `server.py:create_app()` → calls `init_extensions(app)` → `login_manager.init_app(app)` (from `app/extensions.py`)
- `server.py:create_app()` → calls `register_blueprints(app)` → registers `auth_bp`, `dashboard_bp`, `api_bp`, `oauth_bp`, `onboarding_bp`, `monitor_bp`, `developer_bp`, `phone_bp`, `discover_bp`, `internal_bp`
- `server.py:create_app()` → calls `limiter.init_app(app)` → Flask-Limiter setup
- `server.py:create_app()` → conditionally calls `initialize_redis_services(app, MEM_STORE_URI)` when Redis is configured
- `app/config.py` (module-level) → `load_dotenv()` → prints `>>> URL:` to stdout
- `app/log.py` (module-level) → prints `>>> init logging <<<` → creates `LOG` singleton
- `app/db.py` (module-level) → `create_engine(config.DB_URI, ...)` → `scoped_session(sessionmaker(...))`

#### Registration-to-Dashboard Integration Chain

The new-user flow crosses multiple module boundaries:

- `app/auth/views/register.py:register()` → `User.create(email, password, ...)` (in `app/models.py` line 602)
- `User.create()` → `Mailbox.create(user_id=user.id, email=user.email, verified=True)` (line 611)
- `User.create()` → `Alias.create_new(user, prefix="simplelogin-newsletter", ...)` (line 634)
- `User.create()` → `Job.create(name="onboarding-1", ...)` through `Job.create(name="onboarding-4", ...)` (lines 651–665, unless `DISABLE_ONBOARDING` is set)
- `register.py:send_activation_email()` → `ActivationCode.create(user_id=user.id, code=random_string(30))` → `email_utils.send_activation_email(user, activation_link)`
- `email_utils.send_activation_email()` → `send_email()` → `sl_sendmail()` (which logs but does not send when `NOT_SEND_EMAIL=true`)
- `app/auth/views/activate.py:activate()` → `user.activated = True` → `login_user(user)` → `send_welcome_email(user)` → redirect to `dashboard.index`
- `app/auth/views/login.py:login()` → `after_login(user, next_url)` → checks FIDO → checks TOTP → `login_user(user)` → redirect to `dashboard.index`
- `app/dashboard/views/index.py:index()` → `get_stats(user)` → queries `Alias`, `EmailLog` for forward/reply/block counts

#### Background Service Integration Points

- `job_runner.py:main()` → `create_light_app().app_context()` → `get_jobs_to_run()` → `process_job(job)` → dispatches based on `job.name`
- `job_runner.py:process_job()` → handles `onboarding-1` through `onboarding-4` → sends tip emails via `send_email()`
- `email_handler.py:main()` → `Controller(MailHandler(), hostname="0.0.0.0", port=20381)` → `controller.start()`
- `email_handler.py:MailHandler.handle_DATA()` → `create_light_app().app_context()` → `handle(envelope, msg)` → routes to forward/reply/bounce handlers
- `event_listener.py:main()` → `PostgresEventSource(EVENT_LISTENER_DB_URI)` → listens on `simplelogin_sync_events` channel
- `monitoring.py:main()` → 60-second loop → `log_postfix_metrics()`, `log_nb_db_connection()`, `log_pending_to_process_events()`
- `cron.py` + `crontab.yml` → yacron scheduler → dispatches scheduled maintenance (stats, cleanup, HIBP checks, unsent mail retry)

### 0.4.2 Database/Schema Context

The document references the following database entities (all managed through Alembic migrations in `migrations/`):

| Table | Model Class | Role in Documented Flow |
|-------|-------------|------------------------|
| `users` | `User` | Created during registration, activated during email verification |
| `activation_code` | `ActivationCode` | Generated at registration, consumed at activation (1-hour TTL) |
| `mailbox` | `Mailbox` | Auto-created as default mailbox during `User.create()` |
| `alias` | `Alias` | First alias (`simplelogin-newsletter`) created during `User.create()` |
| `job` | `Job` | Onboarding jobs (1–4) scheduled during registration |
| `email_log` | `EmailLog` | Tracked for forward/reply/block statistics on dashboard |
| `contact` | `Contact` | Created when emails are forwarded or replied through aliases |
| `public_domain` / `SLDomain` | `SLDomain` | Alias domains seeded by `init_app.py:add_sl_domains()` |
| `daily_metric` | `DailyMetric` | Incremented during registration (`nb_new_web_non_proton_user`) |
| `sync_event` | `SyncEvent` | Event-sourcing table used by `event_listener.py` |
| `transactional_email` | `TransactionalEmail` | Tracked for every `send_email()` call |

## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

This task has a single deliverable file. No existing files are created or modified.

- **CREATE:** `blitzy/documentation/app_2cd6ee777f8c.md` — Comprehensive markdown Q&A document answering all three user questions about SimpleLogin's runtime behavior

The document's content structure is organized into three major answer sections aligned with the user's questions:

**Section 1 — Application Readiness Indicators** must cover:
- Console output during startup: `>>> init logging <<<` from `app/log.py:67`, `>>> URL: http://localhost:7777` from `app/config.py:80`, `MAX_NB_EMAIL_FREE_PLAN` fallback message from `app/config.py:123`
- Flask app initialization sequence: `create_app()` in `server.py:139` wiring extensions, blueprints, admin, error handlers, rate limiting, CORS, profiler
- Health check endpoint: `GET /health` returns `"success", 200` (defined at `server.py:213-215`)
- Database connectivity: SQLAlchemy engine creation in `app/db.py:9-14` with `application_name` from config
- Redis readiness (if configured): `initialize_redis_services()` called at `server.py:165`
- Blueprint registration confirmation: all 11 blueprints registered at `server.py:233-246` (`auth`, `monitor`, `dashboard`, `developer`, `phone`, `oauth` ×2 prefixes, `onboarding`, `discover`, `internal`, `api`)
- Gunicorn worker startup: `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15`
- Email handler readiness: `email_handler.py:2386` logs `"Start mail controller 0.0.0.0 20381"` and optionally loads PGP keys
- Job runner readiness: `job_runner.py:329-347` enters its 10-second polling loop
- Request logging: `server.py:284-292` after_request handler logs every request with method, path, args, status code, and elapsed time

**Section 2 — New User Flow Walkthrough** must cover:
- **Step 1 — Registration:** `GET /auth/register` renders form → `POST /auth/register` validates `RegisterForm` (email + password ≥8 chars) → optional hCaptcha → `canonicalize_email()` → `email_can_be_used_as_mailbox()` check → `personal_email_already_used()` check → `User.create()` → `Session.commit()` → `send_activation_email()` → renders `register_waiting_activation.html`
- **Step 2 — What User.create() Does:** Creates User row → creates default Mailbox (verified=True) → generates `alternative_id` (UUID) → creates first alias `simplelogin-newsletter@{FIRST_ALIAS_DOMAIN}` → schedules onboarding Jobs (1-day, 2-day, 3-day delays unless `DISABLE_ONBOARDING` is set) → increments `DailyMetric.nb_new_web_non_proton_user` → emits `RegisterEvent(success)` to New Relic
- **Step 3 — Activation Email:** `ActivationCode.create(user_id, code=random_string(30))` with 1-hour expiry → `email_utils.send_activation_email()` composes email with subject "Just one more step to join SimpleLogin" → when `NOT_SEND_EMAIL=true`, the `MailSender.send()` method at `app/mail_sender.py:131` logs the email details without SMTP transmission
- **Step 4 — Account Activation:** `GET /auth/activate?code=...` → validates code → checks expiry → `user.activated = True` → `login_user(user)` → deletes consumed `ActivationCode` → `Session.commit()` → flashes "Your account has been activated" → `send_welcome_email(user)` with subject "Welcome to SimpleLogin" → redirects to `dashboard.index`
- **Step 5 — Login (Subsequent):** `GET /auth/login` renders form → `POST /auth/login` validates `LoginForm` → `sanitize_email()` → `canonicalize_email()` → `User.get_by(email=...)` lookup → `user.check_password()` (bcrypt verify) → checks disabled/deletion/activated states → emits `LoginEvent(success)` → calls `after_login(user, next_url)`
- **Step 6 — after_login() Decision Tree:** If not from Proton: checks `user.fido_enabled()` → if yes, sets `session[MFA_USER_ID]` and redirects to `/auth/fido`; else checks `user.enable_otp` → if yes, redirects to `/auth/mfa`; else calls `login_user(user)`, sets `session["sudo_time"]`, redirects to `dashboard.index`
- **Step 7 — Dashboard Landing:** `app/dashboard/views/index.py:index()` computes `Stats(nb_alias, nb_forward, nb_reply, nb_block)` from `Alias` and `EmailLog` queries → renders dashboard template with alias list, pagination, and alias management controls

**Section 3 — Background Service Indicators** must cover:
- **Job Runner:** Polls every 10 seconds for `Job` records where `state == ready` OR `(state == taken AND taken_at < now - 30 min AND attempts < 5)` AND `(run_at IS NULL OR run_at <= now + 10 min)`. After registration, three onboarding jobs (onboarding-1 at +1 day, onboarding-2 at +2 days, onboarding-4 at +3 days) appear in the queue.
- **Event System:** `RegisterEvent.send()` and `LoginEvent.send()` emit New Relic custom events. `User.delete()` dispatches `UserDeleted` protobuf events through `EventDispatcher.send_event()` → `PostgresDispatcher` → `SyncEvent` table → `NOTIFY simplelogin_sync_events`.
- **Email Handler:** `email_handler.py:main()` starts aiosmtpd on port 20381, logs `"Start mail controller"`, and processes inbound email through `MailHandler.handle_DATA()`. Each email gets a UUID `message_id` for lifecycle tracking.
- **Monitoring:** `monitoring.py` runs a 60-second loop exporting postfix queue sizes, DB connection counts, pending sync events, dead-letter events, and failed events as New Relic custom metrics.
- **Cron Jobs:** `crontab.yml` defines 15 scheduled jobs including `send_undelivered_mails` every 5 minutes, `stats` daily at midnight, `clear_alias_audit_log` and `clear_user_audit_log` hourly.
- **Email Sending in Local Dev:** When `NOT_SEND_EMAIL=true` (set in `example.env`), `MailSender.send()` logs message details (subject, from, to) at DEBUG level and returns True without SMTP transmission.

### 0.5.2 Implementation Approach

The approach for creating the deliverable document follows this sequence:

- Establish the answer framework by organizing content around the user's three questions
- Populate each section with evidence drawn from specific file paths and line numbers
- Include code references in the format `file.py:line_number` for traceability
- Describe observable behaviors (log messages, HTTP responses, database state changes) rather than abstract architectural descriptions
- Note the `NOT_SEND_EMAIL=true` behavior as a key local-development consideration — activation emails are logged, not sent, so the activation code must be retrieved from database or logs
- Document the ephemeral test data pattern: any test users/aliases created during verification would be removed after testing, but the document itself does not create or remove data — it describes what the user should observe

## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**New file to create:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the single deliverable document

**Source files analyzed to derive answers (read-only):**
- `server.py` — Flask app factory, blueprint registration, health endpoint, request lifecycle logging
- `wsgi.py` — Gunicorn WSGI entry point
- `app/config.py` — All environment variable loading, startup print statements, feature flags
- `app/log.py` — Logger initialization, log format, `LOG` singleton
- `app/db.py` — SQLAlchemy engine and session setup
- `app/extensions.py` — Flask-Login manager and rate limiter bootstrap
- `app/redis_services.py` — Redis/Sentinel connection setup
- `app/models.py` — `User`, `ActivationCode`, `ResetPasswordCode`, `Alias`, `Mailbox`, `Job`, `JobState`, `EmailLog`, `Contact`, `SLDomain`, `DailyMetric`, `TransactionalEmail` models
- `app/pw_models.py` — `PasswordOracle` bcrypt hashing mixin
- `app/auth/base.py` — Auth blueprint definition (`/auth` prefix)
- `app/auth/views/register.py` — Registration endpoint and `send_activation_email()`
- `app/auth/views/activate.py` — Activation code processing and user login
- `app/auth/views/login.py` — Login endpoint with credential validation
- `app/auth/views/login_utils.py` — `after_login()` MFA decision tree
- `app/auth/views/mfa.py` — TOTP challenge handler
- `app/auth/views/fido.py` — FIDO/WebAuthn challenge handler
- `app/auth/views/recovery.py` — Recovery code handler
- `app/auth/views/logout.py` — Session cleanup and cookie deletion
- `app/dashboard/base.py` — Dashboard blueprint definition (`/dashboard` prefix)
- `app/dashboard/views/index.py` — Dashboard landing page with stats computation
- `app/email_utils.py` — `send_activation_email()`, `send_welcome_email()`, `send_email()`, template rendering
- `app/mail_sender.py` — `MailSender` class, `NOT_SEND_EMAIL` short-circuit, SMTP delivery
- `app/events/auth_event.py` — `RegisterEvent` and `LoginEvent` telemetry classes
- `app/events/event_dispatcher.py` — `EventDispatcher`, `PostgresDispatcher`, event serialization
- `email_handler.py` — SMTP inbound processor, `MailHandler`, `handle()` routing, `main()` controller startup
- `job_runner.py` — Job polling loop, `process_job()`, onboarding/delete/batch dispatch
- `cron.py` — Scheduled maintenance functions
- `crontab.yml` — yacron job schedule definitions
- `monitoring.py` — Metric export loop (postfix queues, DB connections, sync events)
- `event_listener.py` — Event sourcing with PostgreSQL LISTEN/NOTIFY
- `init_app.py` — Domain seeding and PGP key loading
- `app/fake_data.py` — Demo data seeding reference
- `example.env` — Reference environment configuration
- `Dockerfile` — Build and runtime configuration
- `pyproject.toml` — Dependency manifest and tool configuration
- `README.md` — Self-hosting deployment guide

**Template files referenced in answers:**
- `templates/auth/register.html`
- `templates/auth/register_waiting_activation.html`
- `templates/auth/login.html`
- `templates/auth/activate.html`
- `templates/emails/transactional/activation.html`
- `templates/emails/transactional/activation.txt`
- `templates/emails/com/welcome.html`
- `templates/emails/com/welcome.txt`

### 0.6.2 Explicitly Out of Scope

- **No source code modifications** — the user explicitly stated "please do not make any changes to the source code"
- **No new application features** — this is a documentation/Q&A task
- **No test execution** — the document describes what the user should observe, not automated test results
- **No infrastructure changes** — no Docker, CI/CD, or deployment modifications
- **No dependency additions or upgrades** — no changes to `pyproject.toml` or `poetry.lock`
- **No database migrations** — no new Alembic revision files
- **Social login flows** (GitHub, Google, Facebook, Proton, OIDC) — user asked specifically about email-based registration and standard login
- **Payment and subscription flows** (Paddle, Coinbase, Apple) — not part of the initial setup verification
- **PGP encryption** — advanced feature not relevant to initial verification
- **Custom domain configuration** — out of scope for first-time setup walkthrough
- **Performance optimization or refactoring** — not requested
- **Admin panel operations** — not part of the end-user flow being documented

## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The following rules are explicitly stated by the user and must be strictly observed:

- **SWE-AtlasQnA-Repo Rule:** Create a new markdown document named `app_2cd6ee777f8c.md` that comprehensively answers the questions posed in the prompt. Provide thinking and rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Do not add any other code in the source repository besides the requested document. Place the generated document in the `blitzy/documentation` directory.

- **No Source Code Changes:** The user explicitly stated: "please do not make any changes to the source code." This means zero modifications to any existing `.py`, `.html`, `.yml`, `.toml`, `.env`, or any other file in the repository.

- **Temporary Data Policy:** The user stated: "You can create anything temporary that you need for testing like new users or aliases but remove them when you are done." In the context of a documentation deliverable, this means the document may describe the creation and cleanup of test data as part of the verification walkthrough, but the document itself does not execute these operations.

- **Evidence-Based Answers:** The rule "Do not make assumptions, base your answers on the code as the truth" means every claim in the document must reference a specific file path and, where possible, a line number or function name from the actual codebase.

### 0.7.2 Conventions to Follow

- **Document naming:** Must match the source branch name exactly: `app_2cd6ee777f8c.md`
- **Document location:** Must be placed in `blitzy/documentation/` directory
- **Markdown formatting:** Standard markdown with headers, code blocks, and tables for clarity
- **Code citations:** Use the format `file_path:function_name()` or `file_path:line_number` for traceability
- **No speculative content:** Every statement about application behavior must be traceable to observed code

## 0.8 References

### 0.8.1 Files and Folders Searched

The following files and folders were systematically explored to derive the conclusions in this Agent Action Plan:

**Root-level files:**
- `server.py` — Flask app factory, blueprint registration, health endpoint, local_main()
- `wsgi.py` — WSGI entry point for Gunicorn
- `pyproject.toml` — Poetry dependency manifest (Python ^3.10, all package versions)
- `example.env` — Reference environment configuration (URL, DB_URI, EMAIL_DOMAIN, NOT_SEND_EMAIL, etc.)
- `Dockerfile` — Multi-stage Docker build (Node 10.17 for npm, Python 3.10 for app)
- `email_handler.py` — SMTP inbound handler (2405 lines), MailHandler class, aiosmtpd Controller
- `job_runner.py` — Background job processing loop (348 lines)
- `cron.py` — Scheduled maintenance functions
- `crontab.yml` — yacron schedule definitions (15 jobs)
- `crontab-all-hosts.yml` — Additional host-level cron jobs
- `monitoring.py` — Metric export loop (172 lines)
- `event_listener.py` — Event sourcing CLI (114 lines)
- `init_app.py` — Domain and partner seeding (74 lines)
- `shell.py` — Admin shell utility
- `README.md` — Self-hosting documentation
- `CONTRIBUTING.md` — Contributor guidelines
- `SECURITY.md` — Vulnerability disclosure policy
- `alembic.ini` — Alembic migration configuration
- `.flake8`, `.pylintrc`, `.pre-commit-config.yaml` — Linting and code quality configuration

**app/ package (core application):**
- `app/config.py` — Environment variable loading, feature flags, job name constants
- `app/log.py` — Logger initialization with `LOG` singleton
- `app/db.py` — SQLAlchemy engine, connection, and scoped session
- `app/models.py` — All ORM models (User, Alias, Mailbox, Contact, Job, EmailLog, ActivationCode, etc.)
- `app/extensions.py` — Flask-Login and Flask-Limiter bootstrap
- `app/redis_services.py` — Redis/Sentinel initialization
- `app/pw_models.py` — Password hashing oracle
- `app/email_utils.py` — Email rendering and sending functions
- `app/mail_sender.py` — SMTP delivery with NOT_SEND_EMAIL short-circuit
- `app/fake_data.py` — Demo data seeding
- `app/utils.py` — Random string generation, email sanitization
- `app/constants.py` — HTTP and DMARC constants

**app/auth/ (authentication subsystem):**
- `app/auth/__init__.py` — Package exports
- `app/auth/base.py` — Auth blueprint definition
- `app/auth/views/__init__.py` — View package marker
- `app/auth/views/register.py` — Registration endpoint
- `app/auth/views/activate.py` — Activation code processing
- `app/auth/views/login.py` — Login endpoint
- `app/auth/views/login_utils.py` — after_login() MFA routing
- `app/auth/views/mfa.py` — TOTP MFA challenge
- `app/auth/views/fido.py` — FIDO/WebAuthn challenge
- `app/auth/views/recovery.py` — Recovery code handler
- `app/auth/views/logout.py` — Session cleanup
- `app/auth/views/forgot_password.py` — Password reset initiation
- `app/auth/views/reset_password.py` — Password reset processing

**app/dashboard/ (dashboard subsystem):**
- `app/dashboard/base.py` — Dashboard blueprint definition
- `app/dashboard/__init__.py` — Package exports
- `app/dashboard/views/index.py` — Dashboard landing page with stats

**app/events/ (event subsystem):**
- `app/events/auth_event.py` — LoginEvent and RegisterEvent classes
- `app/events/event_dispatcher.py` — EventDispatcher with PostgresDispatcher

**app/handler/ (mail processing handlers):**
- `app/handler/dmarc.py` — DMARC policy engine
- `app/handler/spamd_result.py` — Rspamd result parsing
- `app/handler/unsubscribe_handler.py` — Unsubscribe processing
- `app/handler/unsubscribe_generator.py` — List-Unsubscribe header generation

**app/onboarding/ (onboarding subsystem):**
- `app/onboarding/base.py` — Onboarding blueprint
- `app/onboarding/utils.py` — Browser detection utilities

**app/jobs/ (background jobs):**
- `app/jobs/event_jobs.py` — Alias creation event emission
- `app/jobs/export_user_data_job.py` — User data export job

**app/api/ (REST API):**
- `app/api/base.py` — API blueprint and authentication decorator
- `app/api/serializer.py` — Alias/contact serialization

**events/ (event infrastructure):**
- `events/event_source.py` — PostgresEventSource and DeadLetterEventSource
- `events/event_sink.py` — HttpEventSink and ConsoleEventSink
- `events/runner.py` — Event runner orchestration

**docs/ (documentation):**
- `docs/api.md` — API reference
- `docs/oauth.md` — OAuth documentation
- `docs/troubleshooting.md` — Diagnostic guide

**Template directories examined:**
- `templates/auth/` — Authentication templates
- `templates/dashboard/` — Dashboard templates
- `templates/emails/transactional/` — Transactional email templates
- `templates/emails/com/` — Communication email templates

### 0.8.2 Attachments

No attachments were provided by the user for this task. No Figma URLs or external design assets are referenced.

### 0.8.3 Technical Specification Sections Referenced

The following tech spec sections were retrieved and cross-referenced to validate findings:

- **1.1 Executive Summary** — Confirmed project overview, AGPLv3 license, core business problem, and stakeholder analysis
- **4.4 Authentication Workflows** — Validated registration, login, MFA, and social auth flow diagrams and decision logic
- **4.2 Core Email Processing Workflows** — Confirmed inbound email routing, forward phase, reply phase, and bounce handling flows
- **4.6 Background Processing Workflows** — Validated job runner polling, cron job inventory, and scheduled maintenance operations

