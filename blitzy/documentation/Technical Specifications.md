# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a series of hands-on, exploratory questions about the SimpleLogin local development environment startup process. The user is performing a first-time stack bring-up and wants an authoritative reference document that captures the exact runtime behavior observed in three specific scenarios:

- **Category:** Create new documentation
- **Documentation Type:** Q&A / Exploratory behavior guide (placed in `blitzy/documentation/`)
- **Target File:** `blitzy/documentation/app_2cd6ee777f8c.md`

The specific documentation requirements are:

- **Requirement 1 — Database-not-migrated Error:** With PostgreSQL running and an empty database (no tables, no migrations executed), start the Flask web server with `python server.py`, then navigate to the login page in a browser. Document the full Python exception that is raised, explaining precisely where in the code it triggers and why.
- **Requirement 2 — Full Service Startup Verification:** After running database migrations (`alembic upgrade head`) and the initialization script (`python init_app.py`), identify every Python service required for the system to function. Start each service and capture the exact stdout/stderr log output that confirms each service is running and ready to accept connections, verifying port binding.
- **Requirement 3 — Missing init_app.py Domain Behavior:** With migrations applied but `init_app.py` NOT executed (leaving the `public_domain` / SLDomain table empty), start the email handler and simulate an incoming email to any `@sl.local` address. Document the exact SMTP status code returned to the sender and the log output explaining the rejection.

Inferred documentation needs based on code analysis:

- The document must include the thinking and rationale behind each answer, grounded strictly in the source code.
- The document must not modify any existing source files; only temporary config files or small test scripts are permissible for observation.
- The `server.py` web server, `email_handler.py` SMTP handler, and `job_runner.py` background worker are the three core services that make up the running stack (as documented in `CONTRIBUTING.md` and `README.md`).
- The `init_app.py` script seeds SLDomain records (from `ALIAS_DOMAINS` config) into the `public_domain` table and loads PGP keys — its absence directly affects domain resolution during email handling.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL Rule — No Source File Modifications:** The user explicitly states: "I'm not changing or deleting any source files here." The implementation rule further mandates: "Do not modify any existing files in the source repository." All answers must be derived from reading the code and observing behavior.
- **Temporary Files Permitted:** "Temporary config files or small test scripts are fine if needed" — this covers `.env` files and transient test helpers.
- **Implementation Rule — Output Format:** Per the `SWE-AtlasQnA-Repo` rule, the output must be a single markdown document named `app_2cd6ee777f8c.md` placed in `blitzy/documentation/`, providing thinking/rationale behind each answer, with all conclusions grounded in the source code as the sole truth.
- **Evidence-Based Answers Only:** Every claim must reference specific source files and line ranges, not assumptions or typical patterns.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the database-not-migrated error, we will **create** `blitzy/documentation/app_2cd6ee777f8c.md` with a section tracing the Flask request lifecycle through `server.py:create_app()` → `set_index_page()` → `auth/views/login.py:login()`, identifying that a GET request to `/login` renders without DB access, while a POST triggers `User.get_by()` in `app/models.py` which queries the nonexistent `users` table, producing `sqlalchemy.exc.ProgrammingError`.
- To document the service startup, we will enumerate the three required services (`server.py`, `email_handler.py`, `job_runner.py`), describe their startup sequences based on their `if __name__ == "__main__"` blocks, and document the expected log output format per `app/log.py`.
- To document the missing-init_app domain behavior, we will trace the email handler's `handle()` → `handle_forward()` → `try_auto_create()` path in `email_handler.py` and `app/alias_utils.py`, showing that with an empty `public_domain` table, emails to `@sl.local` resolve to SMTP status `550 SL E515 Email not exist` (defined in `app/email/status.py`).

### 0.1.4 Inferred Documentation Needs

- Based on code analysis: `app/db.py` connects to PostgreSQL at import time via `engine.connect()`, which succeeds against an empty database (no tables needed for connection).
- Based on structure: The three services (`server.py`, `email_handler.py`, `job_runner.py`) share `create_light_app()` from `server.py` for application context, meaning they all require the same environment configuration (`.env` file, `DB_URI`, `FLASK_SECRET`, etc.).
- Based on the email handler flow: The `handle_forward()` function in `email_handler.py` (line 536) first checks `Alias.get_by(email=...)`, then falls back to `try_auto_create()` which checks custom domains and directories — neither of which match `@sl.local` without SLDomain seeding.
- Based on the config loading: `app/config.py` derives `ALIAS_DOMAINS` from `EMAIL_DOMAIN` (default: `sl.local` per `example.env`), and `init_app.py:add_sl_domains()` iterates over `ALIAS_DOMAINS` to populate the `public_domain` table.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a documentation structure centered on operational self-hosting guides with significant gaps in local development environment behavior documentation.

**Existing documentation files discovered:**

| File | Type | Coverage Status |
|------|------|----------------|
| `README.md` | Self-hosting guide | Comprehensive Docker-based deployment; no local `python server.py` development flow |
| `CONTRIBUTING.md` | Developer guide | Lists local setup steps but does not document error scenarios, startup ordering, or behavior when steps are skipped |
| `SECURITY.md` | Security disclosure | Vulnerability reporting policy only |
| `docs/troubleshooting.md` | Troubleshooting guide | Covers Docker container connectivity issues; no coverage of raw Python service startup |
| `docs/api.md` | API reference | Full REST API documentation |
| `docs/oauth.md` | OAuth/OIDC reference | OAuth flows and local test URLs |
| `docs/ssl.md` | SSL/TLS guide | Certbot, Postfix TLS, HSTS, MTA-STS |
| `docs/upgrade.md` | Upgrade runbook | Version-to-version container upgrade steps |
| `docs/build-image.md` | Docker build guide | Multi-architecture image building |
| `docs/ses.md` | SES integration | Amazon SES relay configuration |
| `docs/gmail-relay.md` | Gmail relay | Gmail SMTP relay setup |
| `docs/enforce-spf.md` | SPF enforcement | Postfix PCRE rules for SPF |
| `docs/postfix-tls.md` | Postfix TLS | Submission TLS configuration |
| `docs/ufw.md` | Firewall setup | UFW port rules |
| `docs/code-structure.md` | Code structure note | Minimal; mostly a TODO about local_data/ |

**Documentation generator:** None detected. All documentation is plain Markdown with no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx/conf.py` configuration files present.

**Diagram tools:** No Mermaid or PlantUML configurations detected in the repository. The `docs/` folder contains static images (`hero.svg`, `postfix-installation.png`, etc.) used by `README.md`.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used for code-to-document mapping:

- **Entry points analyzed:** `server.py` (Flask web app), `email_handler.py` (SMTP handler), `job_runner.py` (background job processor), `init_app.py` (data seeder), `cron.py` (scheduled tasks), `event_listener.py` (event processing)
- **Core application modules:** `app/config.py` (environment configuration), `app/db.py` (database connection), `app/models.py` (SQLAlchemy ORM models including `User`, `Alias`, `SLDomain`), `app/log.py` (logging infrastructure)
- **Authentication flow:** `app/auth/views/login.py` (login view), `app/auth/base.py` (auth blueprint)
- **Email processing:** `app/email/status.py` (SMTP status constants), `app/alias_utils.py` (alias auto-creation logic), `app/email_utils.py` (email utility functions including `is_valid_alias_address_domain`)
- **Database migrations:** `alembic.ini` (Alembic config), `migrations/env.py` (migration environment), `migrations/versions/` (revision history)
- **Configuration:** `example.env` (all available environment variables), `pyproject.toml` (dependencies and tooling)

Key directories examined: `app/`, `docs/`, `migrations/`, `scripts/`, `templates/auth/`

Related documentation found that provides context but does not answer the user's questions:
- `CONTRIBUTING.md` mentions `alembic upgrade head && flask dummy-data && python3 server.py` for local run but does not document what happens when these steps are omitted or run out of order.
- `README.md` documents the Docker-based startup order (migration → init_app → webapp → email handler → job runner) but doesn't describe behavior when steps are skipped.

### 0.2.3 Web Search Research Conducted

No web search is required for this documentation task. All answers are derived directly from the source code, which the implementation rules specify as the sole source of truth. The questions are project-specific behavioral inquiries that can only be answered by code analysis, not external best practices.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following source modules are directly relevant to the three documentation scenarios the user has described:

**Module: `server.py` (Flask web app bootstrap)**
- Public APIs: `create_app()`, `create_light_app()`, `local_main()`
- Current documentation: Mentioned briefly in `CONTRIBUTING.md` ("To run the server: `alembic upgrade head && flask dummy-data && python3 server.py`") but no error-scenario documentation exists
- Documentation needed: Startup behavior with empty database, exact exception traceback when DB tables are missing, login page GET vs POST behavior

**Module: `app/db.py` (Database connection)**
- Public APIs: `engine`, `connection`, `Session` (scoped session)
- Current documentation: None
- Documentation needed: Import-time behavior (connects to DB at module load), behavior when database exists but has no tables (connection succeeds, queries fail)

**Module: `app/auth/views/login.py` (Login endpoint)**
- Public APIs: `login()` route handler at `/login`
- Current documentation: None
- Documentation needed: GET request flow (renders template without DB queries), POST request flow (calls `User.get_by()` which triggers `ProgrammingError` on empty DB)

**Module: `app/models.py` (ORM models)**
- Public APIs: `ModelMixin.get_by()` (line 83), `User` class, `SLDomain` class (line 3116, table `public_domain`)
- Current documentation: None
- Documentation needed: How `get_by()` translates to `Session.query(cls).filter_by(**kw).first()`, which issues SQL against the missing table

**Module: `email_handler.py` (SMTP handler)**
- Public APIs: `main()`, `MailHandler.handle_DATA()`, `handle()`, `handle_forward()`
- Current documentation: Mentioned in `CONTRIBUTING.md` for test email workflow
- Documentation needed: Startup log output, behavior when `SLDomain` table is empty, SMTP status codes returned

**Module: `init_app.py` (Data seeder)**
- Public APIs: `add_sl_domains()`, `load_pgp_public_keys()`, `add_proton_partner()`
- Current documentation: Mentioned in `README.md` Docker init step
- Documentation needed: What data it seeds (`ALIAS_DOMAINS` → `public_domain` table), what breaks when it's skipped

**Module: `job_runner.py` (Background worker)**
- Public APIs: `process_job()`, `get_jobs_to_run()`
- Current documentation: Mentioned in `CONTRIBUTING.md` ("Some features require a job handler")
- Documentation needed: Startup log output, service readiness confirmation

**Module: `app/email/status.py` (SMTP status constants)**
- Public APIs: `E515` = `"550 SL E515 Email not exist"`, plus all other status constants
- Current documentation: None
- Documentation needed: Specific status code returned in the missing-init_app scenario

**Module: `app/alias_utils.py` (Alias auto-creation)**
- Public APIs: `try_auto_create()`, `try_auto_create_via_domain()`, `try_auto_create_directory()`
- Current documentation: None
- Documentation needed: Why auto-creation fails when no domains or directories exist

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented startup ordering:** No existing documentation describes what happens when setup steps are executed out of order or partially.
- **No error-scenario documentation:** The `docs/troubleshooting.md` covers Docker networking issues only; there is no documentation for Python-level exceptions caused by missing database schema.
- **No service readiness reference:** No existing document catalogs the expected startup log output for each service or how to verify they are bound to their ports.
- **No init_app.py impact documentation:** The effect of skipping `init_app.py` on the email handler's domain resolution is completely undocumented.
- **Missing development-mode behavior docs:** The distinction between `create_app()` (full app with debug toolbar) and `create_light_app()` (minimal app context for background services) is undocumented.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document follows the `SWE-AtlasQnA-Repo` implementation rule: a single markdown file named after the source branch, placed in `blitzy/documentation/`.

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
        ├── Introduction (context and approach)
        ├── Question 1: Database-Not-Migrated Error
        │   ├── Setup Context
        │   ├── Server Startup Behavior
        │   ├── Login Page GET Request Analysis
        │   ├── Login Page POST Request — The Exception
        │   ├── Full Exception Traceback
        │   └── Rationale / Code Trace
        ├── Question 2: Required Services and Startup Verification
        │   ├── Service Inventory
        │   ├── Service 1: Flask Web Server (server.py)
        │   ├── Service 2: Email Handler (email_handler.py)
        │   ├── Service 3: Job Runner (job_runner.py)
        │   └── Port Binding Verification
        ├── Question 3: Missing init_app.py — Empty SLDomain Behavior
        │   ├── What init_app.py Seeds
        │   ├── Email Handler Flow with Empty SLDomain
        │   ├── SMTP Status Code Returned
        │   ├── Log Output Explaining Rejection
        │   └── Rationale / Code Trace
        └── Source References
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the Flask app creation flow from `server.py:create_app()` (lines 139–217), tracing each setup step to identify whether it queries the database.
- Extract the login route handler from `app/auth/views/login.py:login()` (lines 25–82), distinguishing GET-path (no DB access) from POST-path (triggers `User.get_by()`).
- Extract the `ModelMixin.get_by()` implementation from `app/models.py` (line 83–84), showing it calls `Session.query(cls).filter_by(**kw).first()`.
- Extract the email handler main loop from `email_handler.py:main()` (lines 2381–2393) and the `handle_forward()` function (lines 536–676) for the domain resolution trace.
- Extract `init_app.py:add_sl_domains()` (lines 39–56) to document what data is seeded.
- Extract SMTP status constants from `app/email/status.py` (specifically `E515` at line 51).
- Generate examples by analyzing the `CONTRIBUTING.md` setup instructions and the `example.env` configuration template.

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Code examples using fenced code blocks with `python`, `bash`, or `text` syntax highlighting
- Source citations as inline references: `Source: /path/to/file.py:LineNumber`
- Thinking/rationale sections after each answer, as mandated by the implementation rules
- All conclusions grounded strictly in the source code — no assumptions

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create within `app_2cd6ee777f8c.md`:**

- **Sequence diagram:** Login page request lifecycle showing GET (success) vs POST (ProgrammingError) paths through `server.py` → `login.py` → `app/models.py` → `app/db.py`
- **Flowchart:** Email handler forward-phase decision tree from `handle()` → `handle_forward()` → `try_auto_create()` → `E515` response
- **Component diagram:** Three required services and their relationships (shared `create_light_app()`, shared DB, port bindings)


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `server.py`, `email_handler.py`, `init_app.py`, `job_runner.py`, `app/db.py`, `app/config.py`, `app/models.py`, `app/auth/views/login.py`, `app/email/status.py`, `app/alias_utils.py`, `app/log.py`, `example.env`, `CONTRIBUTING.md` | Complete Q&A document answering all three behavioral scenarios with full code traces, exception details, startup logs, SMTP status codes, and rationale |

**Transformation Mode: CREATE** — This is the only documentation file being produced. No existing files are modified or deleted.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Q&A / Behavioral Exploration Guide
Source Code References:
  - server.py (Flask app bootstrap, local_main, create_app, create_light_app)
  - email_handler.py (SMTP handler, handle, handle_forward, main, MailHandler)
  - init_app.py (add_sl_domains, load_pgp_public_keys)
  - job_runner.py (main loop, process_job, get_jobs_to_run)
  - app/db.py (engine creation, Session)
  - app/config.py (DB_URI, EMAIL_DOMAIN, ALIAS_DOMAINS, FLASK_SECRET)
  - app/models.py (ModelMixin.get_by, User class, SLDomain class)
  - app/auth/views/login.py (login route, LoginForm)
  - app/email/status.py (E515 and all status constants)
  - app/alias_utils.py (try_auto_create, try_auto_create_via_domain, try_auto_create_directory)
  - app/email_utils.py (is_valid_alias_address_domain)
  - app/log.py (LOG, log format, EmailHandlerFilter)
  - example.env (default configuration values)
  - CONTRIBUTING.md (development setup reference)
Sections:
  - Introduction: Context for the exploration exercise
  - Q1: Database-not-migrated error on login page
    - Server startup analysis (server.py:local_main, app/db.py import-time behavior)
    - GET /login flow (no DB access, page renders)
    - POST /login flow (User.get_by triggers ProgrammingError)
    - Full exception traceback (sqlalchemy.exc.ProgrammingError: UndefinedTable)
    - Rationale tracing through app/models.py:ModelMixin.get_by → Session.query
  - Q2: Required services and startup log output
    - Service 1: server.py on port 7777 (Flask dev server output)
    - Service 2: email_handler.py on port 20381 (aiosmtpd controller log)
    - Service 3: job_runner.py (polling loop, no port)
    - Port binding verification methods
  - Q3: Missing init_app.py — email rejection behavior
    - What init_app.py:add_sl_domains() seeds (ALIAS_DOMAINS → public_domain table)
    - Email handler forward-phase flow (handle → handle_forward → try_auto_create → E515)
    - SMTP 550 status code and E515 constant
    - Log output: "alias X not exist" → "cannot be created on-the-fly, return 550"
  - Source References: Comprehensive file list with line ranges
Diagrams:
  - Sequence diagram: Login page request lifecycle (GET vs POST)
  - Flowchart: Email handler forward-phase decision tree
  - Component diagram: Three services and their relationships
Key Citations: All source files listed above with specific line ranges
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are needed. The repository has no documentation generator (no `mkdocs.yml`, `docusaurus.config.js`, or similar). The output is a standalone Markdown file.

### 0.5.4 Cross-Documentation Dependencies

- The new document references behavior described in `CONTRIBUTING.md` (development setup steps) and `README.md` (Docker deployment order) for context, but does not modify them.
- No navigation links, table of contents updates, or index modifications are required.
- The `blitzy/documentation/` directory may need to be created if it does not already exist.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages from the project's `pyproject.toml` are directly relevant to this documentation exercise because the answers involve tracing their runtime behavior:

| Registry | Package Name | Version | Purpose |
|----------|-------------|---------|---------|
| pip (poetry) | python | ^3.10 | Runtime language; target version `py310` per `pyproject.toml` `[tool.black]` |
| pip (poetry) | flask | ^1.1.2 | Web framework; `create_app()` in `server.py` builds the Flask application |
| pip (poetry) | flask_login | ^0.5.0 | Authentication; `current_user.is_authenticated` check on login page |
| pip (poetry) | SQLAlchemy | 1.3.24 | ORM; `Session.query(User).filter_by()` raises `ProgrammingError` on missing tables |
| pip (poetry) | psycopg2-binary | ^2.9.3 | PostgreSQL driver; underlying `psycopg2.errors.UndefinedTable` exception |
| pip (poetry) | aiosmtpd | ^1.2 | SMTP server; `Controller` in `email_handler.py:main()` binds to port 20381 |
| pip (poetry) | alembic (via Flask-Migrate) | ^2.5.3 | Database migration tool; `alembic upgrade head` creates tables |
| pip (poetry) | arrow | ^0.16.0 | Datetime handling; used in log timestamps and model fields |
| pip (poetry) | coloredlogs | ^14.0 | Log formatting; `app/log.py` uses for colored console output when `COLOR_LOG` is set |
| pip (poetry) | sentry_sdk | ^2.16.0 | Error tracking; integrated in `server.py` if `SENTRY_DSN` is set |
| pip (poetry) | gunicorn | ^20.0.4 | Production WSGI server; not used in local dev mode (`server.py` uses Flask's built-in server) |
| pip (poetry) | flask-debugtoolbar | ^0.11.0 | Debug toolbar; enabled in `local_main()` for development mode |
| pip (poetry) | python-dotenv | ^0.14.0 | Environment loading; `app/config.py` calls `load_dotenv()` for `.env` file |

No documentation-specific tools (mkdocs, sphinx, typedoc, etc.) are required. The output is a standalone Markdown file written directly.

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file `blitzy/documentation/app_2cd6ee777f8c.md` is a standalone document that does not participate in any existing navigation structure or link network.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis:**

- Public APIs documented: The core entry-point scripts (`server.py`, `email_handler.py`, `job_runner.py`, `init_app.py`) are mentioned in `CONTRIBUTING.md` and `README.md` but their runtime behavior, startup sequences, and error modes are not documented.
- User-facing features documented: The login page has template documentation (`templates/auth/login.html`) but no behavioral documentation for error states.
- Configuration options documented: `example.env` covers all environment variables comprehensively, but the consequences of missing configuration or skipped setup steps are not documented.
- Error scenarios documented: 0/3 of the user's requested scenarios are covered by existing docs.

**Target coverage for this task:**

- 3/3 behavioral scenarios fully documented with code traces and exact outputs
- Every answer must cite specific source files and line ranges
- Each answer must include thinking/rationale grounded in the code

**Coverage gaps to address:**

- **Scenario 1 (Empty DB + Login Page):** Currently 0% documented. Target: complete trace from `server.py` startup through `login.py` GET/POST to the `ProgrammingError` exception.
- **Scenario 2 (Service Startup Logs):** Currently 0% documented. Target: exact stdout/stderr for all three services with port binding confirmation.
- **Scenario 3 (Missing init_app.py + Email):** Currently 0% documented. Target: full SMTP flow from `email_handler.py:handle()` through `handle_forward()` → `try_auto_create()` → `E515` with log output.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- All three questions must have definitive answers with full exception tracebacks or log output
- Every code path cited must include the file path and line numbers
- Mermaid diagrams must illustrate the decision flows for scenarios 1 and 3

**Accuracy validation:**

- All conclusions must be derivable from reading the source code alone (per the implementation rule: "base your answers on the code as the truth")
- Exception class names, SMTP status strings, and log format strings must exactly match the source
- The `ProgrammingError` exception must reference the correct table name (`users` from `User.__tablename__` or SQLAlchemy's generated name)
- The `E515` status code must be quoted exactly as `"550 SL E515 Email not exist"` per `app/email/status.py:51`

**Clarity standards:**

- Technical accuracy with developer-level language appropriate for someone running the stack locally
- Progressive disclosure: start with the observable behavior, then trace into the code
- Code snippets must be short (2-3 lines) to illustrate the critical path, not reproduce entire files

**Maintainability:**

- Source citations for every claim (e.g., `Source: email_handler.py:543-555`)
- Clear linkage between questions asked and answers provided
- No assumptions or typical-pattern references; only evidence from this codebase

### 0.7.3 Example and Diagram Requirements

- Minimum examples per scenario: 1 code snippet showing the critical path, 1 output block showing the observable result
- Diagram types required: sequence diagram (login flow), flowchart (email handler forward path), component diagram (service topology)
- Code example testing: Examples are read-only traces through existing code; no executable examples need verification
- Visual content: Mermaid diagrams embedded in the Markdown document


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the sole deliverable

**Source files to analyze for documentation content (read-only):**
- `server.py` — Flask app factory, `local_main()`, `create_app()`, `create_light_app()`
- `email_handler.py` — SMTP handler, `handle()`, `handle_forward()`, `main()`, `MailHandler`
- `init_app.py` — `add_sl_domains()`, `load_pgp_public_keys()`, `add_proton_partner()`
- `job_runner.py` — main polling loop, `process_job()`, `get_jobs_to_run()`
- `app/db.py` — `engine`, `connection`, `Session`
- `app/config.py` — `DB_URI`, `EMAIL_DOMAIN`, `ALIAS_DOMAINS`, `FLASK_SECRET`, `LOAD_PGP_EMAIL_HANDLER`
- `app/models.py` — `ModelMixin.get_by()`, `User`, `SLDomain` (table `public_domain`), `Alias`, `Base`
- `app/auth/views/login.py` — `login()` route handler, `LoginForm`
- `app/email/status.py` — all SMTP status constants, specifically `E515`
- `app/alias_utils.py` — `try_auto_create()`, `try_auto_create_via_domain()`, `try_auto_create_directory()`, `check_if_alias_can_be_auto_created_for_custom_domain()`
- `app/email_utils.py` — `is_valid_alias_address_domain()`
- `app/log.py` — `LOG`, log format string, `EmailHandlerFilter`
- `app/extensions.py` — `login_manager`, `limiter`
- `example.env` — default configuration values
- `alembic.ini` — migration configuration
- `migrations/env.py` — Alembic environment bootstrap
- `CONTRIBUTING.md` — development setup reference
- `README.md` — self-hosting deployment order reference
- `docs/troubleshooting.md` — existing troubleshooting context

**Temporary files permitted:**
- `.env` configuration file (copied from `example.env`)
- Small test scripts for simulating SMTP connections (if needed)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No changes to any existing Python, HTML, or configuration files in the repository. The implementation rule explicitly states: "Do not modify any existing files in the source repository."
- **Test file modifications:** No changes to `tests/` directory files.
- **Feature additions or code refactoring:** This is a read-only observation and documentation exercise.
- **Deployment configuration changes:** No Docker, Nginx, or Postfix configuration changes.
- **Documentation not specified by user:** No updates to `README.md`, `CONTRIBUTING.md`, `docs/troubleshooting.md`, or any other existing documentation file.
- **Database data creation:** No permanent data seeding beyond what's needed for observation (the user allows temporary config files and test scripts only).
- **Production environment concerns:** All observations are limited to local development mode (`python server.py` with `debug=True`, port 7777).
- **Redis/Memcached behavior:** The `MEM_STORE_URI` is not set in the basic local setup; in-memory storage is used for rate limiting.
- **PGP key loading:** The `LOAD_PGP_EMAIL_HANDLER` environment variable is not set by default, so PGP key loading in the email handler is skipped.


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file with no build step.
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/app_2cd6ee777f8c.md` in terminal.
- **Diagram generation command:** Mermaid diagrams are embedded directly in the Markdown using fenced mermaid code blocks; they render natively in GitHub, GitLab, and most Markdown viewers.
- **Documentation deployment command:** Not applicable — the file is committed to the repository.
- **Default format:** Markdown with embedded Mermaid diagrams.
- **Citation requirement:** Every answer section must reference specific source files and line numbers as inline citations.
- **Style guide:** Follow the existing repository documentation style (Markdown with code blocks, as seen in `CONTRIBUTING.md` and `README.md`).
- **Documentation validation:** Manual review to confirm all file paths and line numbers match the current codebase.

### 0.9.2 Environment Configuration for Scenarios

To reproduce the documented behavior, the following environment setup is assumed (based on `example.env` and `CONTRIBUTING.md`):

- `.env` file copied from `example.env` with these key values:
  - `URL=http://localhost:7777`
  - `EMAIL_DOMAIN=sl.local`
  - `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin`
  - `FLASK_SECRET=secret`
  - `NOT_SEND_EMAIL=true`
  - `SUPPORT_EMAIL=support@sl.local`
- PostgreSQL running on `localhost:5432` with database `simplelogin` created
- Python version 3.10 with dependencies installed via `poetry install`
- No Redis required — the basic local setup does not require `MEM_STORE_URI`; Flask-Limiter falls back to in-memory storage

### 0.9.3 Service Startup Commands

The three services are started with these exact commands, each in a separate terminal:

- **Web server:** `python server.py` — binds to `0.0.0.0:7777`
- **Email handler:** `python email_handler.py` — binds to `0.0.0.0:20381`
- **Job runner:** `python job_runner.py` — no port binding; polls DB every 10 seconds


## 0.10 Rules for Documentation

The following rules govern the production of the output documentation file, derived from the user's explicit instructions and the project-level implementation rules.

### 0.10.1 User-Specified Behavioral Rules

- **No source modifications:** Do not modify or delete any existing source files in the repository. The documentation is observational — it describes the system's behavior without altering it. Temporary configuration files or small test scripts are acceptable only if needed to reproduce a scenario.
- **Exact observed output:** Every answer must report the actual output (Python exceptions, SMTP status codes, log messages) that the code would produce. Do not paraphrase or generalize — reproduce the exact string literals and exception class names from the source code.
- **Three-scenario structure:** The document must address exactly three scenarios in the order the user specified: (1) server start with unmigrated database, (2) service startup with correct setup, (3) email handler behavior without `init_app.py` seeding.
- **Code-trace-backed reasoning:** Each answer must explain the code path that produces the observed behavior, citing specific files and line numbers as evidence. Do not assert behavior without tracing the execution path through the source.

### 0.10.2 Project Implementation Rules

- The output file must be named `app_2cd6ee777f8c.md` (matching the source branch name) and placed in the `blitzy/documentation/` directory.
- The document must comprehensively answer the questions posed in the prompt, including thinking and rationale behind the answers.
- All answers must be based on the code as the truth — no assumptions.
- No existing files in the source repository may be modified.

### 0.10.3 Documentation Style Rules

- Follow the existing repository documentation style observed in `README.md` and `CONTRIBUTING.md` — standard Markdown with code blocks and headings.
- Include Mermaid diagrams where they aid comprehension of multi-step code paths.
- Use inline source citations in the format `Source: path/to/file.py:LineNumber` for every technical claim.
- Code snippets should be short and focused (2–3 lines), showing only the critical lines from each code path.


## 0.11 References

### 0.11.1 Source Files Examined

The following source files were read and analyzed to derive the conclusions in this Agent Action Plan:

| File Path | Relevance |
|-----------|-----------|
| `server.py` | Flask app factory (`create_app`, `create_light_app`), root route redirect to login, error handlers, `local_main()` on port 7777 |
| `init_app.py` | `add_sl_domains()` seeding `SLDomain` records into `public_domain` table from `ALIAS_DOMAINS` config |
| `email_handler.py` | `handle()` main dispatch, `handle_forward()` alias lookup and auto-create, `main()` aiosmtpd controller on port 20381 |
| `job_runner.py` | Background job polling loop with `create_light_app()` context, 10-second poll interval |
| `app/db.py` | SQLAlchemy engine creation from `config.DB_URI`, scoped session binding |
| `app/config.py` | `DB_URI`, `EMAIL_DOMAIN`, `ALIAS_DOMAINS`, `FLASK_SECRET` environment variable loading |
| `app/models.py` | `Base = declarative_base()`, `ModelMixin.get_by()` → `Session.query().filter_by().first()`, `SLDomain` model |
| `app/auth/views/login.py` | `/login` route handler — GET renders template, POST calls `User.get_by(email=...)` triggering DB query |
| `app/email/status.py` | SMTP status code constants, including `E515 = "550 SL E515 Email not exist"` |
| `app/alias_utils.py` | `try_auto_create()`, `try_auto_create_via_domain()`, `try_auto_create_directory()` alias auto-creation logic |
| `app/email_utils.py` | `is_valid_alias_address_domain()` checking `SLDomain` and `CustomDomain`, `should_ignore_bounce()` |
| `app/log.py` | Logging configuration with timestamp, level, process, pathname:lineno, funcName, message format |
| `pyproject.toml` | Python ^3.10, Flask ^1.1.2, SQLAlchemy 1.3.24, aiosmtpd ^1.2, dependency versions |
| `example.env` | Default environment variable values for local development setup |
| `alembic.ini` | Alembic migration configuration, script location = `migrations` |
| `README.md` | Docker self-hosting guide, startup order: migrations → init_app → webapp → email handler → job runner |
| `CONTRIBUTING.md` | Local development instructions: `alembic upgrade head && flask dummy-data && python3 server.py` |

### 0.11.2 Folders Explored

| Folder Path | Purpose |
|-------------|---------|
| `/` (root) | Repository top-level structure discovery |
| `app/` | Core Python application package — models, config, auth, email handling |
| `app/auth/views/` | Authentication blueprints including login route |
| `app/email/` | Email processing utilities and SMTP status codes |
| `docs/` | Existing documentation directory (API, contribution guides, SSL, troubleshooting) |
| `migrations/` | Alembic database migration scripts and version history |
| `scripts/` | Utility scripts for various operations |

### 0.11.3 Existing Documentation Reviewed

| Documentation File | Content Summary |
|-------------------|-----------------|
| `README.md` | Project overview, Docker self-hosting guide, feature list, architecture diagram link |
| `CONTRIBUTING.md` | Local development setup (prerequisites, running, testing), code style, PR workflow |
| `docs/api.md` | REST API documentation for alias management, mailbox, custom domain, contact endpoints |
| `docs/ssl.md` | SSL/TLS setup instructions for Nginx reverse proxy |
| `docs/troubleshooting.md` | Common issues and solutions for self-hosted deployments |
| `docs/contributor-code-of-conduct.md` | Contributor code of conduct |

### 0.11.4 User Attachments

No file attachments were provided by the user.

### 0.11.5 External URLs

No Figma URLs or external design references were provided. No web searches were required — all answers are derived from direct source code analysis.


