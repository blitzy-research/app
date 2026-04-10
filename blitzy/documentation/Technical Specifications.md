# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers three interlocking questions about the SimpleLogin self-hosted platform's runtime behavior:

- **Category:** Create new documentation
- **Documentation type:** Operational Q&A guide / Verification runbook

The user is setting up SimpleLogin for the first time in a local environment and needs a detailed, code-grounded document that explains:

- **Startup Readiness Verification:** What observable evidence (log output, UI behavior, health endpoints) confirms that the Flask web application, SMTP email handler, job runner, and supporting infrastructure (PostgreSQL, Redis, Postfix) are fully initialized and ready to serve requests.
- **New-User Walkthrough (Registration → Verification → Login → Dashboard):** A step-by-step narrative tracing the complete product experience from the perspective of a first-time user — covering `POST /auth/register`, activation-email dispatch via `send_activation_email()`, `GET /auth/activate?code=`, session establishment via `login_user()`, MFA decision routing in `after_login()`, and the final redirect to `dashboard.index`.
- **Behind-the-Scenes Runtime Indicators:** What background jobs (`job_runner.py`), scheduled tasks (`cron.py` / yacron), event dispatching (`event_listener.py`, `app/events/event_dispatcher.py`), and monitoring processes (`monitoring.py`) should be active and how to confirm they are operational.

The user explicitly permits creating temporary test artifacts (users, aliases) for testing purposes but requires they be cleaned up afterward. The user also explicitly prohibits any modifications to source code.

### 0.1.2 Special Instructions and Constraints

- **No source code modifications**: The user stated "please do not make any changes to the source code." The deliverable is purely a documentation artifact.
- **Implementation rule from project configuration**: A markdown document named `app_2cd6ee777f8c.md` must be created in the `blitzy/documentation` directory, containing all answers with rationale grounded in the codebase.
- **Answer grounding**: All answers must be based on what the code actually does, not assumptions. Every claim must reference specific source files.
- **Temporary artifacts**: The user permits creating temporary users or aliases for testing but insists they be removed when done. The document must note this as guidance.
- **Style**: Q&A format with thinking/rationale behind each answer, following the SWE-AtlasQnA-Repo rule.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document startup readiness, we will **create** a comprehensive section in `blitzy/documentation/app_2cd6ee777f8c.md` analyzing `server.py:create_app()` (Flask initialization, blueprint registration, health endpoint at `/health`), `email_handler.py:main()` (aiosmtpd Controller startup on port 20381), `job_runner.py` (10-second polling loop), and `Dockerfile` (Gunicorn launch on port 7777).
- To document the user registration-to-dashboard flow, we will **create** a walkthrough section tracing the code path through `app/auth/views/register.py` → `app/auth/views/activate.py` → `app/auth/views/login.py` → `app/auth/views/login_utils.py:after_login()` → `app/dashboard/views/index.py`, referencing the exact log messages, flash notifications, HTTP redirects, and database state transitions.
- To document background runtime indicators, we will **create** a section covering `job_runner.py:process_job()` (Job table polling), `cron.py` scheduled functions, `event_listener.py` (PostgreSQL LISTEN/NOTIFY), `monitoring.py` (Postfix queue metrics and DB connection counts), and the `NOT_SEND_EMAIL` flag in `app/config.py` that controls email dispatch behavior during local development.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **`NOT_SEND_EMAIL` mode behavior:** When running locally with `example.env`, the `NOT_SEND_EMAIL=true` flag is set, meaning `app/mail_sender.py:send()` will log email details to stdout instead of delivering via SMTP. The documentation must explain that activation emails will appear in logs, not in an actual inbox, unless this flag is removed and a local MTA (e.g., MailHog) is configured.
- **`flask dummy-data` command:** The standard local development startup sequence is `alembic upgrade head && flask dummy-data && python3 server.py` (per `CONTRIBUTING.md`), which seeds a demo user `john@wick.com` / `password` via `app/fake_data.py`. The document should note this built-in test user.
- **Health endpoint:** `server.py` registers `GET /health` returning `"success", 200` — a key readiness indicator not mentioned in the user's prompt but essential for verification.
- **Redis dependency:** `app/config.py:MEM_STORE_URI` triggers `initialize_redis_services()` for session management and rate limiting. When Redis is absent, sessions fall back to cookie-only mode. The document should note this.
- **Onboarding job scheduling:** Upon user creation, `job_runner.py` processes onboarding email jobs (`JOB_ONBOARDING_1` through `JOB_ONBOARDING_4`) at scheduled intervals — relevant "behind the scenes" activity the user is asking about.


## 0.2 Documentation Discovery and Analysis


### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a focused documentation structure with operational runbooks and developer guides, but no formal Q&A verification or startup-confirmation documentation.

**Existing documentation inventory:**

| File | Purpose | Relevance to Task |
|------|---------|-------------------|
| `README.md` | Self-hosting deployment guide (DNS, Docker, Postfix, Nginx) | High — describes Docker container startup sequence (sl-db, sl-app, sl-email, sl-job-runner) |
| `CONTRIBUTING.md` | Developer setup, local run instructions, test guide | High — documents `alembic upgrade head && flask dummy-data && python3 server.py` startup |
| `SECURITY.md` | Vulnerability disclosure policy | Low |
| `docs/troubleshooting.md` | Diagnostic steps for welcome-email and alias-forwarding failures | High — directly relevant to verifying email flow |
| `docs/api.md` | Full REST API reference with endpoint documentation | Medium — provides API context for alias operations |
| `docs/oauth.md` | OAuth2/OIDC flow documentation | Low |
| `docs/build-image.md` | Docker multi-arch image build instructions | Low |
| `docs/upgrade.md` | Version upgrade runbook | Low |
| `docs/ssl.md` | TLS/HTTPS and certificate setup | Low |
| `docs/ses.md` | Amazon SES relay configuration | Low |
| `docs/gmail-relay.md` | Gmail SMTP relay setup | Low |
| `docs/enforce-spf.md` | SPF enforcement with Postfix PCRE rules | Low |
| `docs/postfix-tls.md` | Postfix TLS submission setup | Low |
| `docs/ufw.md` | Firewall port configuration | Low |
| `docs/code-structure.md` | Brief note on local_data directory and JWT keys | Low |
| `example.env` | All available configuration options with comments | High — documents environment variables controlling runtime behavior |

**Current documentation framework:** No dedicated documentation generator (mkdocs, Sphinx, Docusaurus) is in use. Documentation consists of raw Markdown files in the `docs/` directory and the project root.

**API documentation tools:** No automated API doc generation (JSDoc, Sphinx autodoc) is configured. `docs/api.md` is manually maintained.

**Diagram tools:** No Mermaid or PlantUML tooling is configured; existing docs use PNG images (`docs/archi.png`, `docs/hero.svg`).

**Gap identified:** There is no existing document that answers "how do I verify SimpleLogin is working after local startup" or "what does the new-user registration flow look like at runtime." This is the exact gap the user's request targets.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to identify code relevant to the user's three questions:

- **Startup and initialization:** `server.py:create_app()`, `server.py:local_main()`, `wsgi.py`, `Dockerfile`, `init_app.py`, `app/config.py`, `app/log.py`, `app/extensions.py`, `app/redis_services.py`
- **Authentication flow:** `app/auth/views/register.py`, `app/auth/views/activate.py`, `app/auth/views/login.py`, `app/auth/views/login_utils.py`, `app/auth/views/logout.py`, `app/auth/views/mfa.py`, `app/auth/views/fido.py`
- **Dashboard entry:** `app/dashboard/views/index.py`, `app/dashboard/base.py`
- **Email dispatch:** `app/email_utils.py:send_activation_email()`, `app/email_utils.py:send_welcome_email()`, `app/mail_sender.py:send()`
- **Background processing:** `job_runner.py`, `cron.py`, `crontab.yml`, `event_listener.py`, `events/event_source.py`, `events/runner.py`
- **Monitoring:** `monitoring.py`, `monitor/metric_exporter.py`, `app/events/auth_event.py`
- **Email handler:** `email_handler.py:MailHandler`, `email_handler.py:main()`, `app/handler/dmarc.py`, `app/handler/unsubscribe_handler.py`
- **Demo data:** `app/fake_data.py`

Key directories examined: `app/auth/`, `app/auth/views/`, `app/dashboard/`, `app/dashboard/views/`, `app/handler/`, `app/events/`, `app/jobs/`, `events/`, `monitor/`, `docs/`, `templates/`

### 0.2.3 Web Search Research Conducted

No external web search was required for this documentation task. All questions posed by the user can be answered comprehensively by direct analysis of the source code. The codebase is self-documenting with detailed log messages, clear function names, and well-organized module structure.


## 0.3 Documentation Scope Analysis


### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation analysis to answer the user's three questions:

**Question 1 — Startup Readiness Verification:**

- Module: `server.py`
  - Public APIs: `create_app()`, `create_light_app()`, `local_main()`, `register_blueprints()`, `set_index_page()`, `init_extensions()`, `init_admin()`
  - Current documentation: `CONTRIBUTING.md` describes the run command; no documentation of observable startup behavior
  - Documentation needed: Log messages emitted during startup, health endpoint, blueprint registration confirmation

- Module: `email_handler.py`
  - Public APIs: `main(port)`, `MailHandler.handle_DATA()`, `handle()`
  - Current documentation: `CONTRIBUTING.md` mentions running `python email_handler.py`
  - Documentation needed: aiosmtpd Controller startup log (`"Start mail controller 0.0.0.0 20381"`), PGP key loading indicator

- Module: `job_runner.py`
  - Public APIs: `get_jobs_to_run()`, `process_job()`, main polling loop
  - Current documentation: `CONTRIBUTING.md` mentions running `python job_runner.py`
  - Documentation needed: Polling loop behavior (10-second interval), job pickup log messages

- Module: `app/config.py`
  - Key items: `URL`, `EMAIL_DOMAIN`, `NOT_SEND_EMAIL`, `DB_URI`, `FLASK_SECRET`, `MEM_STORE_URI`, `DISABLE_ONBOARDING`
  - Documentation needed: Startup prints (`">>> URL:"`, `">>> init logging <<<"`), configuration loading confirmation

**Question 2 — New User Registration Flow:**

- Module: `app/auth/views/register.py`
  - Public APIs: `register()` route handler, `RegisterForm`, `send_activation_email()`
  - Documentation needed: Form validation, user creation, activation code generation, email dispatch, redirect to waiting page

- Module: `app/auth/views/activate.py`
  - Public APIs: `activate()` route handler
  - Documentation needed: Code validation, user activation flag, `login_user()`, welcome email, redirect to dashboard

- Module: `app/auth/views/login.py`
  - Public APIs: `login()` route handler, `LoginForm`
  - Documentation needed: Credential validation, account state checks (disabled, deletion-pending, unactivated), `after_login()` dispatch

- Module: `app/auth/views/login_utils.py`
  - Public APIs: `after_login()`, `get_referral()`
  - Documentation needed: MFA routing decision (FIDO → TOTP → direct login), session setup, dashboard redirect

- Module: `app/dashboard/views/index.py`
  - Public APIs: `index()`, `get_stats()`
  - Documentation needed: Alias statistics display, alias creation/management interface

- Module: `app/email_utils.py`
  - Public APIs: `send_activation_email()`, `send_welcome_email()`
  - Documentation needed: Email subjects, template rendering, `NOT_SEND_EMAIL` log behavior

**Question 3 — Background Runtime Indicators:**

- Module: `job_runner.py`
  - Documentation needed: Onboarding job types (`JOB_ONBOARDING_1` through `JOB_ONBOARDING_4`), job state machine (ready → taken → done)

- Module: `cron.py` + `crontab.yml`
  - Documentation needed: Scheduled job inventory (stats, delete_old_monitoring, check_custom_domain, etc.), yacron scheduling

- Module: `event_listener.py` + `events/`
  - Documentation needed: PostgreSQL LISTEN/NOTIFY for `SyncEvent`, `EventDispatcher` sending protobuf events, dead-letter processing

- Module: `monitoring.py` + `monitor/`
  - Documentation needed: Postfix queue monitoring, DB connection tracking, pending event counting, New Relic metric export

- Module: `app/events/auth_event.py`
  - Documentation needed: `LoginEvent` and `RegisterEvent` New Relic custom event telemetry

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No startup verification guide:** Neither `README.md` nor `CONTRIBUTING.md` describes what log output or UI behavior confirms successful startup. The README jumps from Docker commands to "Enjoy!" without explaining what to look for.
- **No user-journey walkthrough:** No existing document traces the registration → activation → login → dashboard experience from a user's perspective with corresponding system behavior.
- **No background-services runtime guide:** `CONTRIBUTING.md` mentions running `job_runner.py` but doesn't explain what it does at runtime or how to confirm it's working.
- **Missing `NOT_SEND_EMAIL` documentation impact:** The `example.env` sets `NOT_SEND_EMAIL=true` but no document explains what this means for the user experience (emails logged instead of sent).
- **No health-endpoint documentation:** The `/health` endpoint in `server.py` is undocumented in any user-facing guide.


## 0.4 Documentation Implementation Design


### 0.4.1 Documentation Structure Planning

The deliverable is a single markdown document following the SWE-AtlasQnA-Repo project rule. The file will be placed at `blitzy/documentation/app_2cd6ee777f8c.md` and structured as follows:

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
        ├── Introduction (context and scope)
        ├── Q1: Startup Readiness Verification
        │   ├── Flask Web Application Startup
        │   ├── SMTP Email Handler Startup
        │   ├── Job Runner Startup
        │   ├── Health Endpoint Verification
        │   ├── UI Readiness Indicators
        │   └── Summary: Startup Checklist
        ├── Q2: New User Registration Walkthrough
        │   ├── Step 1: Registration (POST /auth/register)
        │   ├── Step 2: Activation Email Dispatch
        │   ├── Step 3: Email Verification (GET /auth/activate)
        │   ├── Step 4: Login (POST /auth/login)
        │   ├── Step 5: MFA Routing Decision
        │   ├── Step 6: Dashboard Landing
        │   └── Summary: Expected Behavior at Each Step
        ├── Q3: Background Services and Runtime Indicators
        │   ├── Job Runner Processing
        │   ├── Cron Scheduled Tasks
        │   ├── Event System (LISTEN/NOTIFY)
        │   ├── Monitoring and Metrics
        │   ├── Auth Telemetry Events
        │   └── Summary: How to Confirm Services Are Active
        └── Notes on Testing and Cleanup
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract startup log messages by tracing `LOG.d()`, `LOG.i()`, and `print()` calls in `server.py`, `app/config.py`, `app/log.py`, `email_handler.py`, and `job_runner.py`
- Extract registration flow behavior from `app/auth/views/register.py` route handler, following the code path through form validation, user creation, and email dispatch
- Extract background job behavior by analyzing `job_runner.py:process_job()` switch-case logic and `crontab.yml` scheduling definitions
- Extract email behavior from `app/mail_sender.py:send()` and the `NOT_SEND_EMAIL` flag path

**Template Application:**

The SWE-AtlasQnA-Repo rule specifies: answers with rationale, code-grounded (no assumptions), no modifications to source code, placed in `blitzy/documentation/`.

**Documentation Standards:**

- Markdown formatting with `#`, `##`, `###` headers
- Mermaid diagrams for the registration-to-dashboard flow
- Code references as inline citations: `Source: /path/to/file.py:LineNumber`
- Tables for configuration options and log-message inventories
- Consistent use of exact function names, file paths, and log message strings from the codebase

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the Q&A document:

- **Registration-to-Dashboard flowchart:** Tracing `register()` → `send_activation_email()` → `activate()` → `after_login()` → `dashboard.index` with decision points for MFA
- **Background services component diagram:** Showing the relationships between `job_runner.py`, `cron.py`/yacron, `event_listener.py`, `monitoring.py`, and their data stores (PostgreSQL, Redis, S3)
- **Startup sequence diagram:** Showing the initialization order of Flask app, blueprints, extensions, admin views, and the health endpoint


## 0.5 Documentation File Transformation Mapping


### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `server.py`, `email_handler.py`, `job_runner.py`, `app/auth/views/register.py`, `app/auth/views/activate.py`, `app/auth/views/login.py`, `app/auth/views/login_utils.py`, `app/dashboard/views/index.py`, `app/email_utils.py`, `app/mail_sender.py`, `app/config.py`, `app/log.py`, `app/fake_data.py`, `cron.py`, `crontab.yml`, `event_listener.py`, `monitoring.py`, `app/events/auth_event.py`, `app/events/event_dispatcher.py`, `example.env`, `Dockerfile`, `wsgi.py`, `init_app.py`, `CONTRIBUTING.md`, `README.md`, `docs/troubleshooting.md` | Comprehensive Q&A document answering all three user questions with code-grounded rationale, Mermaid diagrams, and source citations |

No existing files are modified, deleted, or used as templates. This is a pure CREATE operation for a single deliverable file.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Operational Q&A Guide / Verification Runbook
Source Code: See complete list in transformation table above

Sections:
  - Introduction (purpose, scope, and constraints)
  - Q1: Startup Readiness Verification
    - Flask web app initialization (from server.py:create_app())
    - SMTP handler startup (from email_handler.py:main())
    - Job runner startup (from job_runner.py main loop)
    - Health endpoint verification (from server.py /health route)
    - UI-level readiness indicators (login page rendering)
    - NOT_SEND_EMAIL mode explanation (from app/config.py, app/mail_sender.py)
  - Q2: New User Registration Walkthrough
    - Registration form submission (from app/auth/views/register.py)
    - Activation email dispatch (from app/email_utils.py:send_activation_email)
    - Email verification and account activation (from app/auth/views/activate.py)
    - Login and MFA routing (from app/auth/views/login.py, login_utils.py)
    - Dashboard landing (from app/dashboard/views/index.py)
  - Q3: Background Services and Runtime Indicators
    - Job runner internals (from job_runner.py:process_job)
    - Cron scheduled tasks (from cron.py, crontab.yml)
    - Event system (from event_listener.py, app/events/event_dispatcher.py)
    - Monitoring pipeline (from monitoring.py, monitor/)
    - Auth telemetry (from app/events/auth_event.py)
  - Notes on Testing and Cleanup
    - Demo user account (john@wick.com from app/fake_data.py)
    - Temporary artifact management guidance

Diagrams:
  - Mermaid flowchart: Registration → Activation → Login → Dashboard
  - Mermaid component diagram: Background services architecture
  - Mermaid sequence: Startup initialization order

Key Citations:
  server.py, email_handler.py, job_runner.py, app/auth/views/register.py,
  app/auth/views/activate.py, app/auth/views/login.py,
  app/auth/views/login_utils.py, app/dashboard/views/index.py,
  app/email_utils.py, app/mail_sender.py, app/config.py,
  cron.py, crontab.yml, event_listener.py, monitoring.py,
  app/events/auth_event.py, app/events/event_dispatcher.py,
  example.env, Dockerfile, CONTRIBUTING.md, README.md,
  docs/troubleshooting.md, app/fake_data.py, init_app.py
```

### 0.5.3 Documentation Configuration Updates

No documentation generator configuration updates are required. The project does not use mkdocs, Docusaurus, Sphinx, or any other documentation build system. The `blitzy/documentation/` directory is a standalone output directory specified by the project's implementation rules.


## 0.6 Dependency Inventory


### 0.6.1 Documentation Dependencies

This is a pure documentation task producing a Markdown file. No documentation generator tools need to be installed or configured. However, the following runtime dependencies from the project are relevant to understanding and verifying the documented behavior:

| Registry | Package Name | Version | Purpose (Relevant to Documentation) |
|----------|--------------|---------|--------------------------------------|
| pip (Poetry) | flask | ^1.1.2 | Web framework powering the app; startup via `create_app()` |
| pip (Poetry) | flask_login | ^0.5.0 | Session management; `login_user()` and `@login_required` behavior |
| pip (Poetry) | gunicorn | ^20.0.4 | WSGI server launched by Dockerfile on port 7777 |
| pip (Poetry) | aiosmtpd | ^1.2 | Async SMTP server for `email_handler.py` inbound email |
| pip (Poetry) | SQLAlchemy | 1.3.24 | ORM for User, Alias, Job, ActivationCode models |
| pip (Poetry) | psycopg2-binary | ^2.9.3 | PostgreSQL driver for DB_URI connections |
| pip (Poetry) | redis | ^4.5.3 | Session store and rate limiter backend (MEM_STORE_URI) |
| pip (Poetry) | bcrypt | ^3.2.0 | Password hashing for user registration and login |
| pip (Poetry) | arrow | ^0.16.0 | Timestamp handling in job scheduling and cron |
| pip (Poetry) | sentry_sdk | ^2.16.0 | Error tracking initialization logged at startup |
| pip (Poetry) | newrelic | 8.8.0 | APM and custom metric recording (auth events, email metrics) |
| pip (Poetry) | yacron | ^0.11.1 | Cron scheduler consuming `crontab.yml` |
| pip (Poetry) | coloredlogs | ^14.0 | Colored log output when `COLOR_LOG` is set |
| pip (Poetry) | dkimpy | ^1.0.5 | DKIM signing for outbound email |
| pip (Poetry) | flask_admin | ^1.5.6 | Admin panel initialization logged at startup |
| pip (Poetry) | Flask-Limiter | ^1.4 | Rate limiting on auth endpoints |
| npm | (static assets) | per package-lock.json | Frontend assets built via Node 10.17.0 in Dockerfile |

### 0.6.2 Documentation Reference Updates

Not applicable. This task creates a new standalone document and does not modify any existing documentation files or their internal links.


## 0.7 Coverage and Quality Targets


### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (before this task):**

- Startup readiness verification documented: 0/1 (0%) — No existing guide covers observable startup behavior
- New-user walkthrough documented: 0/1 (0%) — No existing document traces registration → dashboard
- Background services runtime guide documented: 0/1 (0%) — `CONTRIBUTING.md` mentions `job_runner.py` but not its runtime indicators

**Target coverage (after this task):**

- Startup readiness verification: 1/1 (100%) — covering Flask app, email handler, job runner, health endpoint
- New-user walkthrough: 1/1 (100%) — covering register, activate, login, MFA routing, dashboard
- Background services runtime guide: 1/1 (100%) — covering job runner, cron, events, monitoring

**Coverage gaps to address:**

| Area | Current State | Target State | Key Sources |
|------|--------------|--------------|-------------|
| Startup log messages | Undocumented | Fully cataloged with file:line references | `server.py`, `app/config.py`, `app/log.py`, `email_handler.py` |
| Health endpoint | Undocumented | Described with curl verification command | `server.py:213-215` |
| Registration flow | Undocumented as user-facing walkthrough | Step-by-step with expected UI/log behavior | `app/auth/views/register.py`, `app/email_utils.py` |
| Activation flow | Undocumented as user-facing walkthrough | Step-by-step with redirect behavior | `app/auth/views/activate.py` |
| Login + MFA routing | Undocumented as user-facing walkthrough | Complete decision tree documented | `app/auth/views/login.py`, `login_utils.py` |
| Dashboard entry | Undocumented | Statistics display and alias interface described | `app/dashboard/views/index.py` |
| `NOT_SEND_EMAIL` behavior | Mentioned in `example.env` comment only | Fully explained with implications | `app/config.py:91`, `app/mail_sender.py:130-137` |
| Job runner operation | Mentioned as "run `python job_runner.py`" | Full runtime behavior documented | `job_runner.py` |
| Cron job inventory | Not documented outside `crontab.yml` | Tabulated with purposes and schedules | `cron.py`, `crontab.yml` |
| Event system | Not documented for operators | LISTEN/NOTIFY pattern explained | `event_listener.py`, `events/` |
| Monitoring pipeline | Not documented for operators | Metric export flow explained | `monitoring.py`, `monitor/` |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every claim in the Q&A document must reference a specific source file and, where relevant, a line number or function name
- All three user questions must be answered exhaustively with no deferred or incomplete sections
- Both the "what to observe" (user-visible) and "why it happens" (code-grounded) perspectives must be covered

**Accuracy validation:**
- All log message strings cited must match the actual `LOG.d()`, `LOG.i()`, or `print()` calls in the source code
- All URL paths must match the blueprint prefix + route decorator in the corresponding view module
- All configuration variable names must match `app/config.py` or `example.env`

**Clarity standards:**
- Progressive disclosure: start with what the user will see, then explain the underlying code
- Consistent terminology: use "activation" (not "verification"), "alias" (not "forward address"), "mailbox" (not "real email") to match SimpleLogin's own terminology
- Each section includes a summary checklist for quick reference

**Maintainability:**
- Source citations in every section enable future maintainers to trace claims back to code
- Mermaid diagrams can be updated if the code flow changes

### 0.7.3 Example and Diagram Requirements

- Minimum 1 Mermaid diagram per major question (3 total minimum)
- Code snippet examples limited to 2-3 lines showing key log outputs or curl commands
- Tables for structured information (log messages, cron schedules, job types)


## 0.8 Scope Boundaries


### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the sole deliverable

**Source code analyzed for documentation content (read-only):**
- `server.py` — Flask app creation, blueprint registration, health endpoint, startup logging
- `wsgi.py` — WSGI app object exposure for Gunicorn
- `Dockerfile` — Container build and Gunicorn launch command
- `email_handler.py` — SMTP handler startup (`main()`), `MailHandler` class, `handle()` routing
- `job_runner.py` — Job polling loop, `process_job()` dispatch, onboarding job types
- `cron.py` — Scheduled maintenance functions
- `crontab.yml` — yacron schedule definitions
- `event_listener.py` — Event system CLI entrypoint
- `monitoring.py` — Postfix queue and DB connection monitoring
- `init_app.py` — SL domain seeding and PGP key loading
- `app/config.py` — All configuration constants and startup print statements
- `app/log.py` — Logger initialization and format
- `app/extensions.py` — Flask-Login and Flask-Limiter initialization
- `app/redis_services.py` — Redis session and limiter setup
- `app/auth/views/register.py` — Registration route handler
- `app/auth/views/activate.py` — Activation route handler
- `app/auth/views/login.py` — Login route handler
- `app/auth/views/login_utils.py` — Post-login MFA routing logic
- `app/auth/views/logout.py` — Logout and session cleanup
- `app/dashboard/views/index.py` — Dashboard landing page and statistics
- `app/dashboard/base.py` — Dashboard blueprint definition
- `app/email_utils.py` — Activation and welcome email functions
- `app/mail_sender.py` — SMTP delivery with `NOT_SEND_EMAIL` short-circuit
- `app/fake_data.py` — Demo user seeding (`john@wick.com`)
- `app/events/auth_event.py` — Login/Register New Relic telemetry
- `app/events/event_dispatcher.py` — SyncEvent dispatch to PostgreSQL
- `events/event_source.py` — PostgresEventSource LISTEN/NOTIFY
- `events/runner.py` — Event processing orchestration
- `monitor/metric_exporter.py` — Upcloud-to-New Relic metric pipeline
- `example.env` — Configuration variable documentation
- `README.md` — Self-hosting Docker deployment guide
- `CONTRIBUTING.md` — Local development setup instructions
- `docs/troubleshooting.md` — Diagnostic steps for email issues

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** The user explicitly stated "do not make any changes to the source code." No `.py`, `.html`, `.js`, `.css`, or configuration files in the repository will be modified.
- **Test file modifications:** No test files will be created or changed.
- **Feature additions or refactoring:** No new functionality, routes, or code logic will be introduced.
- **Deployment configuration changes:** No changes to `Dockerfile`, `docker-compose.yml`, `nginx.conf`, or Postfix configuration.
- **Existing documentation updates:** `README.md`, `CONTRIBUTING.md`, `docs/*.md`, and other existing files will not be modified.
- **Documentation for features not asked about:** OAuth2/OIDC provider flows, Paddle/Coinbase payment integration, custom domain DNS verification, PGP encryption setup, and browser extension APIs are not in scope.
- **Automated testing of the described flows:** The document describes what to observe; it does not implement automated verification tests.
- **Production deployment guidance:** The document focuses on local/development environment verification, not production hardening.


## 0.9 Execution Parameters


### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a raw Markdown file requiring no build step.
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/app_2cd6ee777f8c.md` for text-based preview.
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown and render natively in GitHub, GitLab, and most Markdown previewers.
- **Documentation deployment command:** Not applicable — no documentation hosting is configured.
- **Default format:** Markdown with embedded Mermaid diagrams.
- **Citation requirement:** Every technical claim must reference a source file path and, where relevant, a function name or line number.
- **Style guide to follow:** SWE-AtlasQnA-Repo rule: provide thinking/rationale, base answers on code as truth, do not make assumptions.
- **Documentation validation:** Manual review of all file path references against the actual repository structure.

### 0.9.2 Local Development Environment Context

The document's answers assume the standard local development setup described in `CONTRIBUTING.md`:

- **Python:** 3.10 (per `pyproject.toml` target-version and `Dockerfile`)
- **Database:** PostgreSQL 13+ running via Docker on localhost
- **Package manager:** Poetry with `poetry sync` for dependency installation
- **Startup command:** `alembic upgrade head && flask dummy-data && python3 server.py`
- **Web app URL:** `http://localhost:7777` (per `example.env` and `server.py:local_main()`)
- **Email behavior:** `NOT_SEND_EMAIL=true` by default in `example.env`, meaning emails are logged, not sent
- **Demo user:** `john@wick.com` / `password` (created by `flask dummy-data` via `app/fake_data.py`)


## 0.10 Rules for Documentation


The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **SWE-AtlasQnA-Repo rule:** Create a new markdown document named `app_2cd6ee777f8c.md` that comprehensively answers the question(s) posed in the prompt. Provide thinking/rationale behind the answers. Do not make assumptions — base answers on the code as the truth. Do not modify any existing files in the source repository. Place the generated document in the `blitzy/documentation` directory.
- **No source code modifications:** The user explicitly stated: "please do not make any changes to the source code." This extends to all `.py`, `.html`, `.js`, `.env`, and configuration files in the repository.
- **Temporary artifact cleanup:** The user permits creating temporary test artifacts (users, aliases) for testing but requires: "remove them when you are done." The documentation should note this guidance.
- **Code-grounded answers:** All behavioral descriptions must be traceable to specific functions, log statements, or configuration values in the codebase. No inferences from "typical Flask behavior" or general web framework patterns are acceptable unless confirmed by the actual source.
- **Comprehensive coverage:** The document must address all three of the user's questions without deferring any part to future work:
  - Startup readiness indicators (logs, UI, health)
  - New user registration-to-dashboard walkthrough
  - Background service runtime indicators


## 0.11 References


### 0.11.1 Files and Folders Searched

The following files and folders were directly inspected to derive the conclusions in this Agent Action Plan:

**Root-level entry points and configuration:**
- `server.py` — Flask application factory, blueprint registration, health endpoint, startup hooks
- `wsgi.py` — WSGI app object for Gunicorn
- `Dockerfile` — Multi-stage Docker build, Gunicorn CMD on port 7777
- `email_handler.py` — aiosmtpd SMTP handler, `MailHandler` class, `main()` startup, `handle()` routing hub
- `job_runner.py` — Background job polling loop, `process_job()` dispatcher, job state management
- `cron.py` — Scheduled maintenance functions (stats, cleanup, health checks, notifications)
- `crontab.yml` — yacron schedule definitions for 15+ recurring jobs
- `event_listener.py` — Event system CLI with LISTEN/NOTIFY and dead-letter modes
- `monitoring.py` — Postfix queue metrics, DB connection counts, pending event tracking
- `init_app.py` — SL domain seeding, PGP key loading, Proton partner setup
- `example.env` — All available configuration options with documentation comments
- `pyproject.toml` — Python 3.10 target, all Poetry dependencies with exact versions
- `README.md` — Self-hosting deployment guide (Docker containers, Postfix, Nginx)
- `CONTRIBUTING.md` — Local development setup, test running, code structure overview
- `SECURITY.md` — Vulnerability disclosure policy
- `shell.py` — Administrative IPython shell
- `oauth_tester.py` — OAuth2 test harness

**Application core (`app/` package):**
- `app/config.py` — Environment-driven configuration loading with startup prints
- `app/log.py` — Logger initialization, `LOG` convenience object, `EmailHandlerFilter`
- `app/extensions.py` — Flask-Login manager, Flask-Limiter setup
- `app/redis_services.py` — Redis/Sentinel session and rate-limit initialization
- `app/mail_sender.py` — SMTP delivery with `NOT_SEND_EMAIL` bypass logging
- `app/email_utils.py` — `send_activation_email()`, `send_welcome_email()`, template rendering
- `app/fake_data.py` — Demo data seeding (user `john@wick.com`, aliases, contacts)
- `app/models.py` — ORM models (User, Alias, Job, ActivationCode, etc.)
- `app/pw_models.py` — bcrypt password hashing and verification

**Authentication subsystem (`app/auth/`):**
- `app/auth/base.py` — `auth_bp` blueprint definition at `/auth`
- `app/auth/__init__.py` — Package re-exports
- `app/auth/views/register.py` — `RegisterForm`, `register()` route, `send_activation_email()`
- `app/auth/views/activate.py` — `activate()` route, activation code validation, `login_user()`
- `app/auth/views/login.py` — `LoginForm`, `login()` route, credential/state validation
- `app/auth/views/login_utils.py` — `after_login()` MFA routing, `get_referral()`
- `app/auth/views/logout.py` — Session cleanup and cookie deletion
- `app/auth/views/mfa.py` — TOTP challenge handler
- `app/auth/views/fido.py` — WebAuthn/FIDO challenge handler
- `app/auth/views/recovery.py` — Recovery code handler

**Dashboard subsystem (`app/dashboard/`):**
- `app/dashboard/base.py` — `dashboard_bp` blueprint definition at `/dashboard`
- `app/dashboard/views/index.py` — Dashboard landing page, `Stats` dataclass, alias statistics

**Email handler subsystem (`app/handler/`):**
- `app/handler/dmarc.py` — DMARC policy engine
- `app/handler/unsubscribe_handler.py` — Unsubscribe request processing
- `app/handler/spamd_result.py` — SpamAssassin/Rspamd result parsing

**Event subsystem:**
- `app/events/auth_event.py` — `LoginEvent`, `RegisterEvent` with New Relic telemetry
- `app/events/event_dispatcher.py` — `EventDispatcher`, `PostgresDispatcher`, SyncEvent persistence
- `events/event_source.py` — `PostgresEventSource` (LISTEN/NOTIFY), `DeadLetterEventSource`
- `events/runner.py` — Event processing orchestration with source+sink pattern
- `events/event_sink.py` — `HttpEventSink`, `ConsoleEventSink` implementations

**Background jobs (`app/jobs/`):**
- `app/jobs/event_jobs.py` — Alias creation event emission in batches
- `app/jobs/export_user_data_job.py` — User data export to ZIP with email delivery

**Monitoring (`monitor/`):**
- `monitor/metric_exporter.py` — Upcloud-to-New Relic export orchestration
- `monitor/upcloud.py` — Upcloud API client for database metrics
- `monitor/newrelic.py` — New Relic telemetry SDK adapter
- `monitor/metric.py` — UpcloudRecord/UpcloudMetric dataclass definitions

**Documentation (`docs/`):**
- `docs/troubleshooting.md` — Diagnostic steps for email delivery issues
- `docs/api.md` — REST API reference
- `docs/oauth.md` — OAuth2/OIDC flow documentation
- `docs/code-structure.md` — JWT key pair notes

**Technical specification sections retrieved:**
- Section 1.2: System Overview — project context, architecture, tech stack
- Section 4.2: Core Email Processing Workflows — forward/reply/bounce flow diagrams
- Section 4.4: Authentication Workflows — registration, login, MFA, social auth diagrams
- Section 4.6: Background Processing Workflows — job runner and cron job inventory

### 0.11.2 Attachments

No attachments were provided by the user.

### 0.11.3 Figma Screens

No Figma URLs or design screens were provided.


