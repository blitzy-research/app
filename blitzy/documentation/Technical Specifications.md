# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **produce a comprehensive observability and verification guide for a fresh local deployment of SimpleLogin**, answering three interrelated questions about the system's runtime behavior:

- **Startup Readiness Verification:** After building and running all components locally, what specific indicators in the application logs and web UI confirm that SimpleLogin is fully operational and ready to handle user authentication and alias-based email activity?
- **New User Journey Walkthrough:** What happens step-by-step when a new user registers, verifies their email address, and logs in for the first time? What visible behaviors at each stage confirm the system is correctly processing every phase of the authentication flow and forwarding the user to the dashboard?
- **Background Service Observability:** During and after the new-user flow, what runtime indicators show that background jobs, internal services, and inter-process communication are active and correctly supporting email forwarding and identity verification?

The implicit requirements surfaced from this request include:

- The system must be built and started locally from source (not just analyzed statically)
- Temporary test data (users, aliases) may be created to validate the flow but must be cleaned up afterward
- **No modifications to the source code are permitted** — this is a read-only observability exercise
- The output is a documentation artifact (a markdown file) capturing findings, not code changes

### 0.1.2 Special Instructions and Constraints

- **No Source Code Modifications:** The user explicitly states "please do not make any changes to the source code." All testing must be done through the running application's existing interfaces (HTTP endpoints, database queries, log inspection)
- **Temporary Data Cleanup:** Any test users, aliases, or other records created during verification must be removed when testing is complete
- **Docker-Based Environment:** The environment is based on the container image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`, which provides the base runtime
- **Output Rule (SWE-AtlasQnA-Repo):** Create a new markdown document named `<source_branch_name>.md` placed in the `blitzy/documentation` directory that comprehensively answers the user's questions, with rationale grounded in the code

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **verify startup readiness**, we will start the Flask application via `server.py` or Gunicorn, observe the bootstrap log messages (URL echo, logging initialization, blueprint registration, database connection), and confirm the web UI returns HTTP 200 on `/auth/login`
- To **walk through the new-user journey**, we will programmatically POST to `/auth/register`, extract the activation code from the `activation_code` database table, GET `/auth/activate?code=...`, then POST to `/auth/login` with the test credentials — verifying HTTP status codes, redirect targets, database state changes, and log entries at each step
- To **observe background services**, we will inspect the `job` table for onboarding job records created during registration, check the `daily_metric` table for registration counters, review log entries showing `EventDispatcher` activity, and document the `mail_sender.py` output for NOT_SEND_EMAIL mode email simulation
- To **clean up**, we will delete the test user and all associated records (aliases, mailboxes, jobs, metrics) from PostgreSQL in the correct foreign-key order


## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

Since this task is a **read-only observability and documentation exercise** (no code changes permitted), the scope discovery focuses on identifying every file that participates in the startup, authentication, and background-processing flows that the user needs to understand. These files were inspected during the investigation.

**Application Bootstrap and Entry Points:**

| File | Role in Investigation |
|---|---|
| `server.py` | Central Flask app factory (`create_app()` and `create_light_app()`); blueprint registration, middleware wiring, error handlers, Sentry/New Relic init, debug toolbar, Gunicorn entry |
| `wsgi.py` | WSGI entry point exposing the Flask app object for production serving |
| `init_app.py` | Seeds SL domains (`add_sl_domains()`), loads PGP keys, creates Proton partner record |
| `Dockerfile` | Multi-stage build: Node.js for frontend assets, Python/Poetry for backend; exposes port 7777 |

**Authentication Flow Files:**

| File | Role in Investigation |
|---|---|
| `app/auth/views/register.py` | Registration endpoint: form validation, hCaptcha check, `User.create()`, activation code generation, activation email dispatch |
| `app/auth/views/activate.py` | Activation endpoint: code validation, user activation, `login_user()`, welcome email, redirect to dashboard |
| `app/auth/views/login.py` | Login endpoint: credential validation, account status checks (disabled, scheduled deletion, unactivated), rate limiting, `after_login()` dispatch |
| `app/auth/views/login_utils.py` | Post-login routing: MFA decision tree (FIDO → TOTP → direct login), session establishment, dashboard redirect |
| `app/auth/base.py` | Blueprint registration for `/auth` prefix |

**Core Application Models and Services:**

| File | Role in Investigation |
|---|---|
| `app/models.py` | SQLAlchemy model layer — `User.create()` orchestrates mailbox creation, first alias generation, onboarding job scheduling |
| `app/config.py` | Centralized environment-driven configuration: `DB_URI`, `EMAIL_DOMAIN`, `FLASK_SECRET`, `NOT_SEND_EMAIL`, job name constants |
| `app/db.py` | Database engine creation via `create_engine()`, scoped session management |
| `app/extensions.py` | Flask-Login manager, Flask-Limiter with user-aware key function |
| `app/fake_data.py` | Reference seed data generator (`flask dummy-data` command) |

**Background Processing and Jobs:**

| File | Role in Investigation |
|---|---|
| `job_runner.py` | Continuous 10-second poll loop processing `Job` table records: onboarding emails, account/mailbox/domain deletion, batch import, data export |
| `cron.py` | Sixteen-plus scheduled maintenance functions: stats, cleanup, HIBP checks, subscription notifications, DNS verification |
| `crontab.yml` | yacron schedule definitions for primary-host cron tasks |
| `crontab-all-hosts.yml` | yacron schedule definitions for multi-host distributed tasks |

**Email and Event Systems:**

| File | Role in Investigation |
|---|---|
| `app/mail_sender.py` | Outbound SMTP delivery with `NOT_SEND_EMAIL` log-only mode; multi-tier retry with filesystem dead-letter |
| `app/email_utils.py` | Email composition: activation emails, welcome emails, transactional templates |
| `email_handler.py` | Inbound SMTP handler (aiosmtpd): forwarding, reply routing, bounce processing |
| `app/events/event_dispatcher.py` | Application event emission: protobuf serialization, PostgreSQL NOTIFY, webhook dispatch |
| `app/events/auth_event.py` | `LoginEvent` and `RegisterEvent` classes emitting New Relic custom events |
| `event_listener.py` | PostgreSQL LISTEN/NOTIFY consumer for Proton ecosystem sync |

**Dashboard and UI:**

| File | Role in Investigation |
|---|---|
| `app/dashboard/views/index.py` | Dashboard main view: alias listing with pagination, stats computation (forward/reply/block counts), intro tour trigger |
| `app/dashboard/base.py` | Blueprint registration for `/dashboard` prefix |
| `templates/auth/register.html` | Registration form template |
| `templates/auth/register_waiting_activation.html` | Post-registration waiting page |
| `templates/auth/login.html` | Login form template |
| `templates/dashboard/index.html` | Main dashboard template with alias management |

**Monitoring and Observability:**

| File | Role in Investigation |
|---|---|
| `monitoring.py` | 60-second collection loop: Postfix queue depth, process counts, PostgreSQL connections, event backlog |
| `app/redis_services.py` | Redis initialization for sessions, rate limiting, and distributed locks |
| `app/session.py` | Redis-backed session store with HMAC-signed session IDs |

**Configuration and Environment:**

| File | Role in Investigation |
|---|---|
| `example.env` | Documents all available environment variables for self-hosting |
| `pyproject.toml` | Poetry dependency manifest: Python ^3.10, Flask ^1.1.2, SQLAlchemy 1.3.24, and 50+ dependencies |
| `poetry.lock` | Pinned dependency versions |
| `alembic.ini` | Alembic migration configuration pointing to `migrations/` |

### 0.2.2 Web Search Research Conducted

No external web search was required for this task. The investigation was conducted entirely through source code analysis and live runtime testing of the local deployment.

### 0.2.3 New File Requirements

Since the implementation rules specify creating a markdown document answering the user's questions:

- **CREATE:** `blitzy/documentation/<source_branch_name>.md` — Comprehensive verification guide documenting startup indicators, new-user journey walkthrough, and background service observability findings


## 0.3 Dependency Inventory

### 0.3.1 Key Packages

The following packages are directly relevant to the startup, authentication, and background processing flows under investigation. Versions are taken from `pyproject.toml` and `poetry.lock`.

| Registry | Package | Version (from manifest) | Purpose in This Investigation |
|---|---|---|---|
| PyPI | `flask` | ^1.1.2 (lock: 1.1.4) | Web framework; app factory, blueprint registration, request handling |
| PyPI | `flask-login` | ^0.5.0 | Session-based user authentication; `login_user()`, `current_user` |
| PyPI | `gunicorn` | ^20.0.4 (lock: 20.1.0) | WSGI server; production entry point on port 7777 |
| PyPI | `SQLAlchemy` | 1.3.24 | ORM for all database models; scoped sessions |
| PyPI | `psycopg2-binary` | ^2.9.3 | PostgreSQL driver |
| PyPI | `Flask-Migrate` | ^2.5.3 | Alembic migration integration for schema management |
| PyPI | `bcrypt` | ^3.2.0 | Password hashing for user credentials |
| PyPI | `redis` | ^4.5.3 | Session storage, rate limiting, distributed locks |
| PyPI | `Flask-Limiter` | ^1.4 | Rate limiting on login/register endpoints |
| PyPI | `sentry-sdk` | ^2.16.0 | Error tracking with Flask and SQLAlchemy integrations |
| PyPI | `newrelic` | 8.8.0 | APM; custom event recording for `LoginEvent`/`RegisterEvent` |
| PyPI | `arrow` | ^0.16.0 | Datetime handling for job scheduling and token expiry |
| PyPI | `aiosmtpd` | ^1.2 (lock: 1.4.6) | Inbound SMTP handler for email processing |
| PyPI | `dkimpy` | ^1.0.5 | DKIM signature handling for email security |
| PyPI | `email-validator` | ^1.1.1 | Email address validation during registration |
| PyPI | `yacron` | ^0.11.1 (lock: 0.11.2) | Cron scheduler for background maintenance tasks |
| PyPI | `coloredlogs` | ^14.0 | Colored log output for local development |
| PyPI | `python-dotenv` | ^0.14.0 | `.env` file loading for configuration |
| PyPI | `flask-debugtoolbar` | ^0.11.0 | Debug toolbar for local development mode |
| PyPI | `python` | ^3.10 | Runtime; Dockerfile uses `python:3.10` base image |

### 0.3.2 Infrastructure Dependencies

| Component | Version | Role |
|---|---|---|
| PostgreSQL | 13+ (compatible with 16 used in testing) | Primary relational database; stores users, aliases, jobs, events |
| Redis | v6+ | Session store, rate limiting backend, distributed concurrency locks |
| Postfix | (external MTA) | Outbound SMTP relay; not required when `NOT_SEND_EMAIL=true` |
| Node.js | 10.17.0 (Dockerfile Stage 1) | Frontend asset compilation (`npm ci` in `static/`) |

### 0.3.3 Dependency Updates

No dependency changes are required for this task. The investigation operates with the existing dependency set as-is. The only output artifact is a documentation file.


## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The user's questions span three interconnected subsystems. The integration analysis maps how these subsystems interact during the flows under investigation.

**Startup Initialization Chain:**

The application startup touches the following integration points in sequence:

- `app/config.py` — Loads 50+ environment variables via `python-dotenv`; validates required variables (`URL`, `EMAIL_DOMAIN`, `DB_URI`, `FLASK_SECRET`)
- `app/db.py` — Creates SQLAlchemy engine connected to PostgreSQL; establishes `scoped_session`
- `server.py:create_app()` — Registers 10 Flask blueprints (`auth_bp`, `dashboard_bp`, `api_bp`, `oauth_bp`, `developer_bp`, `discover_bp`, `monitor_bp`, `onboarding_bp`, `internal_bp`, `phone_bp`), initializes Flask-Login, Flask-Limiter, Sentry SDK, New Relic, and optionally Redis sessions via `initialize_redis_services()`
- `init_app.py` — Seeds `public_domain` table with `EMAIL_DOMAIN` and `PREMIUM_ALIAS_DOMAINS`, loads PGP keys into keyring

**Registration-to-Dashboard Flow Integration:**

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

**Database Touchpoints During Registration:**

| Table | Operation | Triggered By |
|---|---|---|
| `users` | INSERT | `User.create()` in `app/models.py` |
| `mailbox` | INSERT (verified=True) | `Mailbox.create()` during `User.create()` |
| `alias` | INSERT (first newsletter alias) | `Alias.create_new()` during `User.create()` |
| `job` | INSERT × 3 (onboarding-1, onboarding-2, onboarding-4) | `Job.create()` during `User.create()` |
| `activation_code` | INSERT then DELETE | `ActivationCode.create()` in register.py; `ActivationCode.delete()` in activate.py |
| `daily_metric` | INSERT or UPDATE | `DailyMetric.get_or_create_today_metric()` in register.py |

### 0.4.2 Background Service Integration Points

**Job Runner (`job_runner.py`):**

- Polls the `job` table every 10 seconds with `get_jobs_to_run()`
- Uses `create_light_app().app_context()` for database access
- Onboarding jobs (`onboarding-1`, `onboarding-2`, `onboarding-4`) are scheduled with 1-day, 2-day, and 3-day delays respectively via `run_at`
- Each job sends an onboarding email through `email_utils` → `mail_sender.py`

**Event Dispatcher (`app/events/event_dispatcher.py`):**

- `RegisterEvent` and `LoginEvent` emit New Relic custom events for observability
- `EventDispatcher.send_event()` serializes protobuf events and issues PostgreSQL NOTIFY — but only if `EVENT_WEBHOOK` is configured; otherwise logs: "Not sending events because webhook is not configured and allowed to be empty"

**Cron Scheduler (`cron.py` + `crontab.yml`):**

- `send_undelivered_mails` runs every 5 minutes to retry failed SMTP deliveries
- `stats` runs daily at midnight to compute platform-wide metrics
- All tasks execute within `create_light_app().app_context()`

### 0.4.3 Email Delivery Integration

When `NOT_SEND_EMAIL=true` is set (the standard local development configuration), `MailSender.send()` in `app/mail_sender.py` short-circuits actual SMTP delivery and instead logs the email subject, sender, and recipient at DEBUG level. This is the key indicator that the email subsystem is "working" in a local environment — the log messages confirm that the application composed and attempted to deliver the email, even though no SMTP server receives it.

Emails triggered during the new-user flow:

| Email | Subject | Triggered By | Recipient |
|---|---|---|---|
| Activation | "Just one more step to join SimpleLogin" | `send_activation_email()` in `register.py` | User's registration email |
| Welcome | "Welcome to SimpleLogin" | `send_welcome_email()` in `activate.py` | User's newsletter alias |


## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since no source code modifications are permitted, the execution plan centers on creating a single documentation artifact that captures the findings from the live investigation.

**Group 1 — Documentation Output:**

- **CREATE:** `blitzy/documentation/<source_branch_name>.md` — The comprehensive verification guide answering all three of the user's questions with evidence from live testing

**Group 2 — Environment Setup (Runtime-Only, No File Changes):**

- Install PostgreSQL 13+ and Redis 6+ locally
- Create database `simplelogin` with user `myuser`
- Set required environment variables: `URL`, `EMAIL_DOMAIN`, `SUPPORT_EMAIL`, `DB_URI`, `FLASK_SECRET`, `NOT_SEND_EMAIL`, `EMAIL_SERVERS_WITH_PRIORITY`
- Run Alembic migrations: `alembic upgrade head`
- Seed SL domains: `python3 init_app.py`
- Start application: `gunicorn wsgi:app -b 0.0.0.0:7777 -w 1`

**Group 3 — Verification Testing (Temporary Data):**

- Register test user via POST to `/auth/register`
- Extract activation code from `activation_code` table
- Activate user via GET to `/auth/activate?code=...`
- Login via POST to `/auth/login`
- Access dashboard via GET to `/dashboard/`
- Inspect database state: `users`, `mailbox`, `alias`, `job`, `daily_metric` tables
- Clean up all temporary data in correct FK order

### 0.5.2 Implementation Approach

The documentation file will be structured around three major sections corresponding to the user's three questions:

**Section 1 — Startup Readiness Indicators:**

- Document the exact log lines emitted during startup (URL echo, logging init, Flask serving message)
- Document the HTTP response codes for key endpoints (`/auth/login` → 200, `/` → 302)
- Document the database state after `init_app.py` (SL domains in `public_domain` table)

**Section 2 — New User Journey Walkthrough:**

- Document each HTTP request/response in the registration → activation → login → dashboard flow
- Document the database state changes at each step with exact table names and column values
- Document the log entries that confirm each step succeeded
- Document the UI elements visible on the dashboard (alias list, stats, intro tour)

**Section 3 — Background Service Observability:**

- Document the `job` table records created during registration (onboarding-1, onboarding-2, onboarding-4)
- Document the `daily_metric` table entries showing registration counters
- Document the `RegisterEvent` and `LoginEvent` New Relic custom event emissions in `auth_event.py`
- Document the `NOT_SEND_EMAIL` log output showing email composition without delivery
- Document the `EventDispatcher` log output showing event system status
- Document the `job_runner.py` polling behavior and how it would process the onboarding jobs

### 0.5.3 Key Runtime Observations from Live Testing

The following observations were captured during the actual live test run:

**Startup Logs Observed:**
```
>>> URL: http://localhost:7777
>>> init logging <<<
* Serving Flask app "server" (lazy loading)
* Debug mode: on
```

**Registration Log Entries:**
```
create user testuser@example.com
send email to testuser@example.com, subject 'Just one more step to join SimpleLogin'
Not sending events because webhook is not configured
```

**Activation Log Entries:**
```
send email to simplelogin-newsletter.xxx@sl.local, subject 'Welcome to SimpleLogin'
redirect user to dashboard
```

**Login Log Entries:**
```
log user <User 3 testuser@example.com> in
redirect user to dashboard
```

**Dashboard Log Entries:**
```
Show intro to <User 3 testuser@example.com>
```


## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Files Read and Analyzed:**
- `server.py` — Application factory, blueprint registration, middleware chain
- `wsgi.py` — WSGI entry point
- `init_app.py` — Domain seeding and PGP key loading
- `app/auth/views/register.py` — Registration flow
- `app/auth/views/activate.py` — Activation flow
- `app/auth/views/login.py` — Login flow
- `app/auth/views/login_utils.py` — Post-login MFA routing
- `app/auth/base.py` — Auth blueprint
- `app/dashboard/views/index.py` — Dashboard main view
- `app/models.py` — User, Alias, Mailbox, Job, ActivationCode, DailyMetric models
- `app/config.py` — Environment configuration
- `app/db.py` — Database session management
- `app/extensions.py` — Flask-Login, Flask-Limiter
- `app/mail_sender.py` — Email delivery with NOT_SEND_EMAIL mode
- `app/email_utils.py` — Email composition utilities
- `app/events/event_dispatcher.py` — Event emission system
- `app/events/auth_event.py` — Login/Register event recording
- `app/redis_services.py` — Redis session/rate-limit initialization
- `app/session.py` — Redis-backed session store
- `app/fake_data.py` — Seed data reference
- `job_runner.py` — Background job processor
- `cron.py` — Scheduled maintenance tasks
- `crontab.yml` — yacron scheduling configuration
- `monitoring.py` — Infrastructure metric collection
- `email_handler.py` — Inbound SMTP handler
- `event_listener.py` — PostgreSQL LISTEN/NOTIFY consumer
- `example.env` — Environment variable documentation
- `pyproject.toml` — Dependency manifest
- `Dockerfile` — Container build definition
- `templates/auth/**` — Authentication HTML templates
- `templates/dashboard/**` — Dashboard HTML templates

**Database Tables Inspected:**
- `users` — User records (created, activated, cleaned up)
- `mailbox` — Default mailbox (created with user, cleaned up)
- `alias` — First newsletter alias (created with user, cleaned up)
- `activation_code` — Activation codes (created on register, deleted on activate)
- `job` — Onboarding jobs (created on register, cleaned up)
- `daily_metric` — Registration counters (incremented on register, cleaned up)
- `public_domain` — SL domains (seeded by init_app.py)

**Runtime Flows Executed:**
- Full application startup via Gunicorn
- User registration via POST `/auth/register`
- Email activation via GET `/auth/activate?code=...`
- User login via POST `/auth/login`
- Dashboard access via GET `/dashboard/`
- Complete cleanup of all test data

**Output Artifact:**
- `blitzy/documentation/<source_branch_name>.md`

### 0.6.2 Explicitly Out of Scope

- **Source code modifications** — Explicitly forbidden by the user
- **SMTP email delivery testing** — `NOT_SEND_EMAIL=true` is the correct local configuration; actual Postfix integration is out of scope
- **MFA flow testing** (FIDO/TOTP) — Not part of the basic new-user journey
- **OAuth2/OIDC provider flow** — Not requested
- **Social login testing** (GitHub, Google, Facebook, Proton) — Not requested
- **Custom domain management** — Not part of the new-user flow
- **Payment/subscription testing** — Not requested
- **Admin panel operations** — Not requested
- **Browser extension integration** — Not applicable to server-side verification
- **Performance optimization** — Not requested
- **Security hardening** — Not requested
- **Production deployment** — User explicitly asks about local development setup only


## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

The following rules were explicitly provided by the user and must be strictly adhered to:

- **SWE-AtlasQnA-Repo Rule:** Create a new markdown document named `<source_branch_name>.md` that comprehensively answers the questions posed in the prompt
- **Build and Run Requirement:** Build and run the source code to analyze repository behavior as needed — do not make assumptions; base all answers on the code as the truth
- **Rationale Requirement:** Provide thinking and rationale behind the answers
- **No Existing File Modifications:** Do not modify any existing files in the source repository
- **No Additional Code:** Do not add any other code in the source repository besides the requested documentation
- **Document Placement:** Place the generated document in the `blitzy/documentation` directory in the destination repo
- **Temporary Data Policy:** Any test users, aliases, or temporary data created during testing must be removed after verification is complete
- **Source Code Integrity:** No changes to the source code are permitted — this is a read-only observation exercise

### 0.7.2 Conventions Observed from Codebase

- **Environment Configuration:** All runtime settings are driven by environment variables loaded through `python-dotenv` in `app/config.py`; the `example.env` file documents all available options
- **Local Development Mode:** `NOT_SEND_EMAIL=true` and `COLOR_LOG=true` are standard for local development per `example.env`
- **Database Migrations:** Managed by Alembic via `alembic.ini` and the `migrations/` directory; executed with `alembic upgrade head`
- **Domain Seeding:** `init_app.py` must be run after migrations to populate the `public_domain` table
- **Logging Convention:** The application uses a custom `LOG` object from `app/log.py` with colored output when `COLOR_LOG` is set, including DEBUG-level request logging with method, path, status code, and timing


## 0.8 References

### 0.8.1 Repository Files and Folders Searched

The following files and directories were directly inspected during the investigation:

**Root-Level Files:**
- `server.py` — Flask application factory and WSGI bootstrap
- `wsgi.py` — Gunicorn entry point
- `init_app.py` — Domain seeding and PGP initialization
- `job_runner.py` — Background job polling loop
- `cron.py` — Scheduled maintenance task definitions
- `crontab.yml` — yacron cron schedule configuration
- `monitoring.py` — Infrastructure metric collection
- `email_handler.py` — Inbound SMTP handler
- `event_listener.py` — PostgreSQL LISTEN/NOTIFY event consumer
- `example.env` — Environment variable documentation
- `pyproject.toml` — Poetry dependency manifest and tooling configuration
- `poetry.lock` — Pinned dependency versions
- `Dockerfile` — Multi-stage Docker build definition
- `alembic.ini` — Alembic migration configuration
- `shell.py` — Interactive admin shell
- `README.md` — Project documentation

**Application Module Files (`app/`):**
- `app/config.py` — Centralized environment configuration
- `app/models.py` — SQLAlchemy ORM model definitions
- `app/db.py` — Database engine and session management
- `app/extensions.py` — Flask-Login and Flask-Limiter setup
- `app/fake_data.py` — Seed data generator
- `app/mail_sender.py` — Outbound email delivery
- `app/email_utils.py` — Email composition and delivery utilities
- `app/email_validation.py` — Email address validation
- `app/redis_services.py` — Redis connection initialization
- `app/session.py` — Redis-backed session store

**Authentication Module (`app/auth/`):**
- `app/auth/base.py` — Auth blueprint definition
- `app/auth/views/register.py` — Registration endpoint
- `app/auth/views/activate.py` — Activation endpoint
- `app/auth/views/login.py` — Login endpoint
- `app/auth/views/login_utils.py` — Post-login routing logic

**Dashboard Module (`app/dashboard/`):**
- `app/dashboard/base.py` — Dashboard blueprint definition
- `app/dashboard/views/index.py` — Main dashboard view

**Events Module (`app/events/`):**
- `app/events/event_dispatcher.py` — Event emission API
- `app/events/auth_event.py` — Login/Register event classes

**API Module (`app/api/`):**
- `app/api/views/auth.py` — API authentication endpoints

**Template Directories:**
- `templates/auth/` — Authentication HTML templates (register, login, activate, etc.)
- `templates/dashboard/` — Dashboard HTML templates

**Folders Explored:**
- Root repository (`/`) — Full directory listing
- `app/` — All Python modules
- `app/auth/views/` — All authentication view files
- `app/dashboard/views/` — All dashboard view files
- `app/api/views/` — API endpoint modules
- `app/events/` — Event system infrastructure
- `migrations/` — Alembic migration revisions

### 0.8.2 Technical Specification Sections Referenced

- **Section 1.1 (Executive Summary)** — Project overview, stakeholder identification, business value
- **Section 4.4 (Authentication Workflows)** — Registration, login, MFA, and social auth flow diagrams
- **Section 4.6 (Background Processing Workflows)** — Job runner and cron scheduler documentation
- **Section 6.1 (Core Services Architecture)** — Five-process architecture, inter-service communication, scalability, resilience patterns

### 0.8.3 Attachments

No external attachments (Figma designs, API specifications, etc.) were provided for this task.

### 0.8.4 Environment Configuration

- **Container Image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (as specified by `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`)
- **Runtime:** Python 3.10 (as specified in `pyproject.toml` and `Dockerfile`)
- **Database:** PostgreSQL (tested with v16; project compatible with 13+)
- **Cache:** Redis (tested with v7; project compatible with v6+)
- **Key Environment Variables Used:**
  - `URL=http://localhost:7777`
  - `EMAIL_DOMAIN=sl.local`
  - `SUPPORT_EMAIL=support@sl.local`
  - `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin`
  - `FLASK_SECRET=secret`
  - `NOT_SEND_EMAIL=true`
  - `COLOR_LOG=true`
  - `EMAIL_SERVERS_WITH_PRIORITY=[(10, "email.hostname.")]`


