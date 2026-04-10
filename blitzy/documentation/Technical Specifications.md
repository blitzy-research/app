# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a series of operational and behavioral questions about the SimpleLogin self-hosted email aliasing application. The user is new to SimpleLogin and wants to understand the observable runtime behavior of three core components — the **web server**, the **email handler**, and the **job runner** — after a local deployment.

- **Documentation Category:** Create new documentation
- **Documentation Type:** Operational Q&A / Runtime Behavior Guide
- **Target Output:** A single Markdown file named `app_2cd6ee777f8c.md` placed in the `blitzy/documentation/` directory

The user's requirements decompose into the following specific documentation needs:

- **Requirement 1 — Component Health Verification:** How can a user confirm the web server (Flask/Gunicorn on port 7777), email handler (aiosmtpd on port 20381), and job runner (polling loop) are up and responding after local startup? What log messages and health endpoints confirm liveness?
- **Requirement 2 — Dashboard UI Confirmation:** What should a user see in the web dashboard that confirms sign-in and alias management capabilities are functional? What statistics, views, and indicators appear?
- **Requirement 3 — User Action Walkthrough:** What happens at the code level when a new account is created, an alias is created, and that alias receives an email? What runtime artifacts (log entries, database records, UI changes) confirm these actions succeeded?
- **Requirement 4 — Background Component Behavior:** Do the email handler and job runner automatically come online to support email activity and data handling? What behavior confirms these background processes are functioning correctly?
- **Implicit Requirement — Code-Grounded Answers:** All answers must be grounded in the actual source code with reasoning and rationale provided. No assumptions are permitted.

### 0.1.2 Special Instructions and Constraints

- **Do not modify the source code.** The project rule explicitly states no existing files in the source repository may be changed.
- **Output file name and location:** The document must be named `app_2cd6ee777f8c.md` (matching the source branch name) and placed in the `blitzy/documentation/` directory.
- **Rationale required:** Every answer must include the thinking and rationale behind it, traced to specific source code files and line numbers.
- **Code-as-truth principle:** Answers must be derived from the codebase, not from assumptions or external documentation.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document health verification**, we will extract endpoint definitions from `server.py` (the `/health` route at line 213-215), startup log messages from `email_handler.py` (the `main()` function at lines 2381-2404), and the job runner's polling loop from `job_runner.py` (lines 329-347).
- To **document dashboard UI confirmation**, we will analyze `app/dashboard/views/index.py` (the `get_stats()` function and `index()` route), the auth login flow in `app/auth/views/login.py`, and the root route redirect logic in `server.py` (lines 250-255).
- To **document user action behavior**, we will trace account creation through `app/auth/views/register.py`, alias creation through `app/dashboard/views/index.py` and `app/alias_utils.py`, and email reception through the `handle_forward()` function in `email_handler.py` (lines 536-676).
- To **document background component behavior**, we will analyze the aiosmtpd Controller startup in `email_handler.py` (line 2383-2386), the job runner's infinite loop with `get_jobs_to_run()` in `job_runner.py` (lines 307-347), and the cron scheduling in `crontab.yml`.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, additional documentation subjects that must be addressed include:

- **Log format explanation:** The logging format defined in `app/log.py` (line 13-16) uses a structured format with `message_id` correlation that is critical for understanding runtime output.
- **SMTP status codes:** The status codes in `app/email/status.py` (E200 through E525) are returned by the email handler and appear in logs, requiring explanation.
- **Flask app factory pattern:** The `create_app()` vs `create_light_app()` distinction in `server.py` affects which components are available at runtime.
- **Database seeding:** The `flask dummy-data` command in `server.py` (lines 490-497) and `app/fake_data.py` (lines 40-55) seed a test user (`john@wick.com` / `password`) that the user will encounter during local development.


## 0.2 Documentation Discovery and Analysis


### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Markdown-based documentation structure** with moderate coverage of deployment and operations topics, but no existing document addressing the user's specific questions about runtime verification and behavioral walkthrough.

- **Documentation framework:** Plain Markdown files with no documentation generator (no mkdocs.yml, docusaurus.config.js, or sphinx conf.py detected)
- **Documentation directory:** `docs/` folder containing 13 files covering operational topics
- **API documentation:** `docs/api.md` — comprehensive REST API reference
- **OAuth documentation:** `docs/oauth.md` — OAuth/OIDC flow documentation
- **Troubleshooting:** `docs/troubleshooting.md` — covers email delivery diagnosis using `swaks`, `telnet`, and `docker logs`
- **Self-hosting guide:** `README.md` — end-to-end Docker-based self-hosting instructions
- **Contributing guide:** `CONTRIBUTING.md` — developer setup and local run instructions including the `alembic upgrade head && flask dummy-data && python3 server.py` workflow
- **Diagram tools detected:** No Mermaid or PlantUML configurations, but SVG illustrations exist in `docs/` (hero.svg, postfix PNGs)

Existing documentation files and their relevance to the user's questions:

| File | Content | Relevance |
|------|---------|-----------|
| `README.md` | Self-hosting guide (Docker, DNS, Postfix, Nginx) | High — describes how to start sl-app, sl-email, sl-job-runner containers |
| `CONTRIBUTING.md` | Developer setup, local run instructions | High — describes `python3 server.py`, `python email_handler.py`, `python job_runner.py` |
| `docs/troubleshooting.md` | Diagnosing email delivery failures | Medium — explains `docker logs sl-email` for checking email handler |
| `docs/api.md` | REST API reference | Low — API endpoints, not runtime behavior |
| `docs/upgrade.md` | Version upgrade procedures | Low — container restart ordering |
| `docs/ssl.md` | TLS/HTTPS configuration | None |
| `docs/ses.md` | Amazon SES relay setup | None |

### 0.2.2 Repository Code Analysis for Documentation

The following search patterns and directories were examined to build the documentation:

- **Entry points analyzed:** `server.py`, `email_handler.py`, `job_runner.py`, `cron.py`, `wsgi.py`, `init_app.py`, `monitoring.py`
- **Core application modules:** `app/config.py`, `app/log.py`, `app/db.py`, `app/models.py`, `app/fake_data.py`, `app/email/status.py`
- **Auth flow:** `app/auth/base.py`, `app/auth/views/` (login.py, register.py, activate.py, logout.py)
- **Dashboard flow:** `app/dashboard/base.py`, `app/dashboard/views/index.py` (alias statistics, creation)
- **API layer:** `app/api/base.py`, `app/api/serializer.py`
- **Email processing:** `email_handler.py` (handle_forward, handle_reply, MailHandler class)
- **Job processing:** `job_runner.py` (process_job, get_jobs_to_run)
- **Deployment:** `Dockerfile`, `example.env`, `crontab.yml`
- **Dependency manifest:** `pyproject.toml`

### 0.2.3 Web Search Research Conducted

No web search was required for this task. All answers are grounded in the source code per the project rule: "Do not make assumptions, base your answers on the code as the truth." The codebase provides complete answers to every question posed by the user.


## 0.3 Documentation Scope Analysis


### 0.3.1 Code-to-Documentation Mapping

The documentation must trace runtime behavior from source code. The following modules require analysis and documentation:

- **Module: `server.py` (Flask Web Server)**
  - Public APIs / Functions: `create_app()`, `create_light_app()`, `local_main()`, `healthcheck()`, `set_index_page()`, `register_blueprints()`
  - Current documentation: Partially covered in `CONTRIBUTING.md` (how to start locally)
  - Documentation needed: Health endpoint behavior, startup log messages, request logging format, route registration confirmation, dashboard redirect logic

- **Module: `email_handler.py` (SMTP Email Handler)**
  - Public APIs / Functions: `main()`, `MailHandler.handle_DATA()`, `MailHandler._handle()`, `handle()`, `handle_forward()`, `handle_reply()`
  - Current documentation: Partially covered in `CONTRIBUTING.md` (swaks test flow), `docs/troubleshooting.md` (docker logs)
  - Documentation needed: aiosmtpd Controller startup, per-email processing log lines, SMTP status codes returned, forward vs reply phase identification, contact/EmailLog creation behavior

- **Module: `job_runner.py` (Background Job Runner)**
  - Public APIs / Functions: `get_jobs_to_run()`, `process_job()`, `delete_mailbox_job()`, onboarding email functions
  - Current documentation: Briefly mentioned in `CONTRIBUTING.md` ("python job_runner.py")
  - Documentation needed: Polling loop mechanics (10-second interval), job state transitions (ready → taken → done), job types handled, log messages during job processing

- **Module: `app/log.py` (Logging Infrastructure)**
  - Public APIs: `LOG`, `set_message_id()`, `EmailHandlerFilter`
  - Current documentation: None
  - Documentation needed: Log format explanation, message_id correlation, log level shortcuts (d/i/w/e)

- **Module: `app/dashboard/views/index.py` (Dashboard Landing)**
  - Public APIs: `index()`, `get_stats()`
  - Current documentation: None
  - Documentation needed: Alias statistics display, alias creation workflow, what users see after login

- **Module: `app/auth/views/register.py` and `app/auth/views/login.py` (Auth Flow)**
  - Current documentation: None specific to runtime behavior
  - Documentation needed: Registration flow, account activation, login redirect behavior

- **Module: `app/fake_data.py` (Test Data Seeding)**
  - Public APIs: `fake_data()`
  - Current documentation: Referenced in `CONTRIBUTING.md` as `flask dummy-data`
  - Documentation needed: Default test user credentials, seeded aliases, contacts, and email logs

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No runtime verification guide exists.** There is no document explaining what log output or HTTP responses confirm that all three components (web server, email handler, job runner) are operational.
- **No user-journey walkthrough.** No existing documentation traces account creation → alias creation → email reception as a connected flow with expected runtime artifacts.
- **No background component lifecycle documentation.** The email handler and job runner startup mechanics, their polling/listening behavior, and their log signatures are not documented anywhere.
- **No log format reference.** The structured log format with message_id correlation is not explained in any existing document.
- **No SMTP status code reference.** The custom `E2xx`/`E4xx`/`E5xx` status codes in `app/email/status.py` are not documented outside the source code.


## 0.4 Documentation Implementation Design


### 0.4.1 Documentation Structure Planning

The output document `app_2cd6ee777f8c.md` will be structured as a comprehensive Q&A guide organized by the user's original questions, with code-grounded rationale for every answer:

```text
blitzy/documentation/
└── app_2cd6ee777f8c.md
    ├── Introduction (context and scope)
    ├── Q1: Confirming Components Are Up and Responding
    ├── Q2: Dashboard UI and Log Confirmation
    ├── Q3: User Action Walkthrough
    ├── Q4: Background Component Lifecycle
    └── Conclusion
```

Key subsections within Q1 will cover web server health verification (the `/health` endpoint defined in `server.py` at lines 213-215, startup log messages, and request logging in the `after_request` hook at lines 272-296), email handler health verification (the aiosmtpd Controller startup in `email_handler.py` at lines 2381-2386 with its "Start mail controller" log message on port 20381), and job runner health verification (the polling loop in `job_runner.py` at lines 329-347 with its "Take job" log messages and 10-second sleep interval).

Q2 addresses the dashboard landing page redirect from `server.py` lines 250-255, the alias statistics display from `index.py` `get_stats()`, and notification indicators. Q3 traces account creation through the registration and activation views, alias creation through the dashboard, and email reception through `handle_forward()`. Q4 covers persistent listener and poller lifecycle and cron scheduling.

### 0.4.2 Content Generation Strategy

- **Information Extraction Approach:**
  - Extract health endpoint definitions from `server.py` function `healthcheck()` at line 213
  - Extract startup behavior from `email_handler.py` function `main()` at lines 2381-2404
  - Extract polling mechanics from `job_runner.py` main block at lines 329-347
  - Extract dashboard statistics from `app/dashboard/views/index.py` function `get_stats()` at lines 32-52
  - Extract registration flow from `app/auth/views/register.py` and activation from `app/auth/views/activate.py`
  - Extract email forwarding logic from `email_handler.py` function `handle_forward()` at lines 536-676

- **Documentation Standards:**
  - Markdown formatting with proper headers (# ## ###)
  - Brief code examples using fenced code blocks with syntax highlighting (2-3 lines max)
  - Source citations as inline references: `Source: /path/to/file.py:LineNumber`
  - Tables for status code reference and component summary
  - Clear reasoning/rationale sections explaining the "why" behind each answer

### 0.4.3 Diagram and Visual Strategy

- **Mermaid sequence diagram:** Email reception flow (Postfix → email_handler → mailbox)
- **Mermaid flowchart:** Component startup and health check verification steps
- **Mermaid flowchart:** Account creation → alias creation → email reception lifecycle


## 0.5 Documentation File Transformation Mapping


### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `server.py`, `email_handler.py`, `job_runner.py`, `app/dashboard/views/index.py`, `app/auth/views/register.py`, `app/auth/views/login.py`, `app/fake_data.py`, `app/log.py`, `app/email/status.py`, `cron.py`, `crontab.yml`, `init_app.py`, `monitoring.py`, `app/config.py` | Comprehensive Q&A document answering all user questions about runtime behavior, health verification, user action walkthroughs, and background component lifecycles — all grounded in code analysis with rationale |

### 0.5.2 New Documentation Files Detail

```text
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Operational Q&A / Runtime Behavior Guide
Source Code:
  - server.py (web server entry point, health check, routing)
  - email_handler.py (SMTP handler, forward/reply logic)
  - job_runner.py (background job polling loop)
  - app/dashboard/views/index.py (dashboard statistics, alias management)
  - app/auth/views/register.py (account registration)
  - app/auth/views/login.py (authentication flow)
  - app/auth/views/activate.py (account activation)
  - app/fake_data.py (test data seeding)
  - app/log.py (logging configuration and format)
  - app/email/status.py (SMTP status codes)
  - app/alias_utils.py (alias creation logic)
  - app/models.py (database models: User, Alias, Contact, EmailLog)
  - cron.py (scheduled maintenance tasks)
  - crontab.yml (cron schedule definitions)
  - init_app.py (SL domain seeding)
  - monitoring.py (metric export and postfix queue monitoring)
  - CONTRIBUTING.md (local development workflow)
  - example.env (environment variable reference)
Sections:
  - Introduction and Context
  - Confirming Web Server Is Up (health endpoint, logs, dashboard access)
  - Confirming Email Handler Is Up (aiosmtpd startup, port listening, log messages)
  - Confirming Job Runner Is Up (polling loop, job state transitions, log messages)
  - Dashboard UI Confirmation (login redirect, alias stats, notification panel)
  - Account Creation Walkthrough (register → activate → login → dashboard)
  - Alias Creation Walkthrough (dashboard form → alias_utils → DB persistence)
  - Email Reception Walkthrough (SMTP → handle_forward → Contact/EmailLog creation)
  - Background Component Lifecycle (email handler persistent listener, job runner persistent poller, cron scheduling)
  - Cross-Component Interaction Patterns
Diagrams:
  - Mermaid sequence diagram: Email forwarding lifecycle
  - Mermaid flowchart: Component health verification
Key Citations:
  - server.py:213-215 (health endpoint)
  - server.py:250-255 (root redirect)
  - server.py:272-296 (request logging)
  - email_handler.py:2381-2404 (SMTP server startup)
  - email_handler.py:1945-2233 (handle function)
  - email_handler.py:536-676 (handle_forward)
  - job_runner.py:307-347 (get_jobs_to_run, main loop)
  - app/dashboard/views/index.py:32-52 (get_stats)
  - app/log.py:13-16 (log format)
  - app/fake_data.py:40-55 (test user seeding)
  - app/email/status.py:1-64 (SMTP status codes)
```

### 0.5.3 Documentation Configuration Updates

No documentation generator configuration files need to be created or updated. The project uses plain Markdown files without a build system. The new file will be placed directly in the `blitzy/documentation/` directory, which must be created as it does not currently exist.

### 0.5.4 Cross-Documentation Dependencies

- The new document references several existing files for context but does not require updates to any of them:
  - `README.md` — referenced for Docker container startup commands
  - `CONTRIBUTING.md` — referenced for local development workflow
  - `docs/troubleshooting.md` — referenced for email delivery diagnosis context
- No navigation, table of contents, or index updates are required since there is no documentation generator system


## 0.6 Dependency Inventory


### 0.6.1 Documentation Dependencies

This is a documentation-only task that creates a Markdown file. No documentation tooling dependencies are required beyond a text editor. However, the following project runtime dependencies are relevant because the documentation references their behavior:

| Registry | Package Name | Version | Purpose (Relevant to Documentation) |
|----------|--------------|---------|--------------------------------------|
| pip (poetry) | flask | ^1.1.2 | Web application framework — the server runs on Flask with Jinja2 templates |
| pip (poetry) | flask_login | ^0.5.0 | Session management — controls the login/redirect flow documented |
| pip (poetry) | gunicorn | ^20.0.4 | Production WSGI server — launches the web app on port 7777 in Docker |
| pip (poetry) | aiosmtpd | ^1.2 | Async SMTP server — the email handler's listening framework |
| pip (poetry) | sqlalchemy | 1.3.24 | ORM — drives database queries for User, Alias, Contact, EmailLog models |
| pip (poetry) | psycopg2-binary | ^2.9.3 | PostgreSQL driver — required for DB_URI connections |
| pip (poetry) | arrow | ^0.16.0 | DateTime handling — used in job scheduling and log timestamps |
| pip (poetry) | sentry_sdk | ^2.16.0 | Error tracking — integrated in server.py for production monitoring |
| pip (poetry) | newrelic | 8.8.0 | APM monitoring — custom metrics recorded in email_handler and monitoring.py |
| pip (poetry) | coloredlogs | ^14.0 | Colored log output — enabled when COLOR_LOG=true for local development |
| pip (poetry) | flask_admin | ^1.5.6 | Admin panel — provides /admin interface for database management |
| pip (poetry) | yacron | ^0.11.1 | YAML-configured cron — drives scheduled tasks from crontab.yml |
| pip (poetry) | python | ^3.10 | Runtime — required Python version per pyproject.toml |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file is a standalone document with no cross-links to existing documentation files that need modification. All references to existing files are informational citations within the document body.


## 0.7 Coverage and Quality Targets


### 0.7.1 Documentation Coverage Metrics

- **User questions addressed:** 4/4 (100%) — all four question clusters from the user's prompt are covered
  - Q1: Component health verification — covered via server.py `/health`, email_handler.py `main()`, job_runner.py main loop
  - Q2: Dashboard UI and log confirmation — covered via dashboard index.py `get_stats()`, server.py root redirect
  - Q3: User action walkthrough (account, alias, email) — covered via auth register/activate, dashboard index, email_handler `handle_forward()`
  - Q4: Background component lifecycle — covered via email_handler persistent listener, job_runner persistent poller, crontab.yml
- **Source code modules documented:** 17/17 relevant modules analyzed and cited
- **Runtime behaviors documented:** All observable log messages, HTTP responses, and database state changes
- **Coverage target:** 100% of user questions answered with code-grounded rationale

### 0.7.2 Documentation Quality Criteria

- **Completeness requirements:**
  - Every answer traces to specific source file paths and line numbers
  - All three core components (web server, email handler, job runner) have dedicated verification sections
  - Expected log messages are quoted from actual LOG statements in the code
  - Expected HTTP responses are documented from actual route handler return values

- **Accuracy validation:**
  - All code references verified against actual source files retrieved during analysis
  - SMTP status codes match the values defined in `app/email/status.py`
  - Port numbers match the values in `server.py` (7777), `email_handler.py` (20381), and `Dockerfile`
  - Test user credentials match `app/fake_data.py` line 44-54 (`john@wick.com` / `password`)

- **Clarity standards:**
  - Technical accuracy with accessible language suitable for newcomers to SimpleLogin
  - Progressive disclosure: simple verification steps first, then deeper behavioral analysis
  - Clear distinction between "what you see" (observable output) and "why it happens" (code rationale)

- **Maintainability:**
  - Source citations for traceability (every claim cites a file path)
  - Self-contained document with no external dependencies

### 0.7.3 Example and Diagram Requirements

- **Minimum examples per component:** At least one expected log line and one verification command per component
- **Diagram types required:** Mermaid sequence diagram (email flow), Mermaid flowchart (verification steps)
- **Code example policy:** Brief inline snippets (2-3 lines) showing relevant code paths, not full function bodies


## 0.8 Scope Boundaries


### 0.8.1 Exhaustively In Scope

- **New documentation files:**
  - `blitzy/documentation/app_2cd6ee777f8c.md` — the sole deliverable document

- **Source code files analyzed for documentation content (read-only):**
  - `server.py` — web server entry point, health check, routing, request logging
  - `email_handler.py` — SMTP handler, forward/reply logic, MailHandler class, aiosmtpd startup
  - `job_runner.py` — background job polling loop, job state machine, process_job dispatch
  - `cron.py` — scheduled maintenance tasks (stats, log deletion, sanity checks)
  - `crontab.yml` — cron schedule definitions for yacron
  - `init_app.py` — SL domain and PGP key seeding
  - `monitoring.py` — postfix queue metrics, DB connection monitoring
  - `wsgi.py` — Gunicorn WSGI entry point
  - `Dockerfile` — container build and startup command
  - `example.env` — environment variable reference
  - `app/config.py` — configuration loading from environment
  - `app/log.py` — logging format, message_id correlation, LOG shortcuts
  - `app/db.py` — database session management
  - `app/models.py` — User, Alias, Contact, EmailLog, Job, Mailbox models
  - `app/fake_data.py` — test data seeding (john@wick.com user)
  - `app/email/status.py` — SMTP status codes (E200-E525)
  - `app/alias_utils.py` — alias creation, auto-create, status change logic
  - `app/dashboard/views/index.py` — dashboard landing page, alias statistics
  - `app/dashboard/base.py` — dashboard blueprint definition
  - `app/auth/views/register.py` — account registration
  - `app/auth/views/login.py` — authentication flow
  - `app/auth/views/activate.py` — account activation
  - `app/auth/base.py` — auth blueprint definition
  - `app/api/base.py` — API authentication and authorization
  - `README.md` — self-hosting guide reference
  - `CONTRIBUTING.md` — developer setup reference
  - `docs/troubleshooting.md` — diagnostic reference

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository will be changed, added, or deleted (per user rule: "Do not modify the source code")
- **Test file modifications:** No test files will be created or modified
- **Deployment configuration changes:** No Docker, Postfix, Nginx, or DNS configuration will be altered
- **Feature additions or code refactoring:** No logic changes of any kind
- **Existing documentation updates:** README.md, CONTRIBUTING.md, docs/*.md will NOT be modified
- **Runtime execution:** While the user mentions "try performing some basic user actions," this documentation task documents the expected behavior based on code analysis — it does not execute the application or create actual test data
- **Security analysis:** No security audit or vulnerability assessment
- **Performance analysis:** No benchmarking or optimization recommendations
- **Non-core components:** Payment integrations (Paddle, Coinbase, Apple), social login providers (GitHub, Google, Facebook), and PGP encryption details are out of scope unless they directly answer the user's questions


## 0.9 Execution Parameters


### 0.9.1 Documentation-Specific Instructions

- **Output file:** `blitzy/documentation/app_2cd6ee777f8c.md`
- **Output directory creation:** `mkdir -p blitzy/documentation` (directory does not yet exist in the repository)
- **Default format:** Markdown with Mermaid diagrams for architectural and workflow illustrations
- **Documentation build command:** Not applicable — this is a standalone Markdown document, not part of a documentation generator pipeline (no mkdocs, Sphinx, or Docusaurus detected in the repository)
- **Documentation preview command:** Any Markdown previewer (e.g., VS Code preview, GitHub rendering, `grip` CLI tool)
- **Diagram generation:** Mermaid diagrams are embedded inline in fenced code blocks tagged with `mermaid`; they render natively on GitHub and in most modern Markdown viewers
- **Documentation validation:** Visual review of Markdown rendering; verify all Mermaid blocks parse correctly by pasting into the [Mermaid Live Editor](https://mermaid.live)
- **Citation requirement:** Every technical claim must reference the specific source file and line number from the SimpleLogin repository (e.g., `Source: server.py:213-215`)
- **Style guide:** Follow the established style of the repository's existing documentation (README.md, CONTRIBUTING.md, docs/troubleshooting.md) — direct prose, clear headings, code snippets for commands and log output

### 0.9.2 Content Generation Approach

- **Information extraction:** All answers are derived exclusively from static analysis of the repository source code. No runtime execution is performed. The document treats the code as the single source of truth.
- **Question-answer structure:** The output document is organized around the four user questions, with each answered by tracing through the relevant code paths, identifying log statements, HTTP responses, database queries, and UI template renderings.
- **Code snippet inclusion:** Short, targeted code excerpts (2–3 lines) illustrate key log messages, status codes, and control flow. Full function bodies are not reproduced.
- **Diagram inclusion:** Mermaid sequence diagrams and flowcharts visualize component startup, email processing, and job runner lifecycle.
- **Rationale sections:** Each answer includes a "Thinking / Rationale" block explaining which files were examined and why the conclusions follow from the code evidence, per the SWE-AtlasQnA-Repo rule.

### 0.9.3 Constraints and Guardrails

- **Read-only:** No source files may be modified. The only file created is the output document in `blitzy/documentation/`.
- **No runtime side-effects:** The document describes expected runtime behavior from code analysis; it does not start services, create accounts, or send emails.
- **Cleanup:** No temporary testing data is created, so no cleanup is required.
- **No assumptions:** All conclusions are grounded in explicit code evidence with file paths and line numbers cited. Where the code is ambiguous, the document states what is observable versus what is inferred.


## 0.10 Rules for Documentation


### 0.10.1 User-Specified Rules

- **"Do not modify any existing files in the source repository."** — The output document is the only artifact created. No existing file (source code, configuration, documentation) is changed or deleted.
- **"Do not modify the source code. You can create temporary testing data such as accounts or aliases but clean them up when you are done."** — Since the task is a code-as-truth documentation exercise and no runtime execution occurs, no temporary data is created and no cleanup is required.
- **"Do not make assumptions, base your answers on the code as the truth."** — Every claim in the output document references a specific file and line range. Where code does not provide a definitive answer, the document explicitly states that the behavior is inferred and explains the reasoning.
- **"Provide thinking / rationale behind the answers."** — Each section of the output document includes a rationale block that explains which files were examined, what evidence was found, and how conclusions were drawn.
- **"Create a new markdown document named `<source_branch_name>.md`"** — The deliverable is `blitzy/documentation/app_2cd6ee777f8c.md`, matching the branch name.
- **"Place the generated document in the `blitzy/documentation` directory in the destination repo."** — The directory is created if it does not exist. No other files are placed anywhere.

### 0.10.2 Documentation Content Rules

- **Code-as-truth principle:** All answers are derived from static reading of the repository source code. No external assumptions, blog posts, or third-party documentation are treated as authoritative for describing SimpleLogin behavior.
- **Source citation format:** Every technical claim cites the originating file and line range using the pattern `Source: <file_path>:<start_line>-<end_line>` or `Source: <file_path>:<line>`.
- **Short code excerpts only:** Code snippets included in the document are kept to 2–3 lines to illustrate specific log messages, status codes, or control flow checks. Full function bodies are not reproduced to respect the repository's license and avoid unnecessary duplication.
- **Mermaid diagrams for workflows:** Complex multi-component interactions (startup sequences, email processing chains, job runner lifecycle) are illustrated with Mermaid sequence diagrams or flowcharts embedded in fenced code blocks.
- **Consistent terminology:** The document uses the same names found in the codebase — "email handler" (not "mail server"), "job runner" (not "task worker"), "alias" (not "email alias" or "forwarding address"), "contact" (not "sender"), "mailbox" (not "inbox").
- **Answer completeness:** Each of the four user questions receives a dedicated section with a direct answer, supporting evidence, and rationale. No question is deferred or partially answered.
- **No speculative content:** If a behavior cannot be confirmed from the code, the document states "Not determinable from code analysis" rather than guessing.


## 0.11 References


### 0.11.1 Repository Files and Folders Searched

The following files and folders were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Entry point and runtime files:**

| File Path | Purpose |
|-----------|---------|
| `server.py` | Flask web server: app factory, health endpoint, routing, request logging, local dev server |
| `email_handler.py` | SMTP email handler: aiosmtpd controller, forward/reply processing, DMARC/spam checks |
| `job_runner.py` | Background job runner: polling loop, job dispatch, onboarding/import/deletion/export processing |
| `cron.py` | Scheduled tasks: stats, log deletion, sanity checks, domain verification, HIBP scanning |
| `crontab.yml` | Cron schedule definitions for yacron (14 scheduled jobs) |
| `init_app.py` | Application initialization: SL domain seeding, PGP key loading, Proton partner creation |
| `monitoring.py` | Runtime monitoring: postfix queue metrics, DB connection counts, New Relic export |
| `wsgi.py` | Gunicorn WSGI entry point |
| `Dockerfile` | Container build: multi-stage (Node 10 + Python 3.10), exposes port 7777 |

**Configuration and environment files:**

| File Path | Purpose |
|-----------|---------|
| `example.env` | Environment variable reference (199 lines, all config knobs) |
| `app/config.py` | Configuration loading from environment via dotenv |
| `pyproject.toml` | Python project metadata and dependency manifest |

**Application core modules:**

| File Path | Purpose |
|-----------|---------|
| `app/log.py` | Logging configuration: format string, message_id correlation, LOG.d/i/w/e shortcuts |
| `app/db.py` | Database session management |
| `app/fake_data.py` | Test data seeding: john@wick.com user, bounced email, sample aliases |
| `app/email/status.py` | Custom SMTP status codes E200 through E525 |
| `app/alias_utils.py` | Alias creation, auto-create, transfer, and status change logic |

**Web interface modules:**

| File Path | Purpose |
|-----------|---------|
| `app/dashboard/views/index.py` | Dashboard landing: get_stats() alias/forward/reply/block counts, alias creation |
| `app/dashboard/base.py` | Dashboard blueprint registration |
| `app/auth/views/register.py` | Account registration flow |
| `app/auth/views/login.py` | Authentication and login |
| `app/auth/views/activate.py` | Account activation via email link |
| `app/auth/base.py` | Auth blueprint registration |
| `app/api/base.py` | API authentication and authorization |

**Existing documentation:**

| File Path | Purpose |
|-----------|---------|
| `README.md` | Project overview, self-hosting guide, Docker setup instructions |
| `CONTRIBUTING.md` | Developer setup, local dev workflow, testing with swaks |
| `docs/troubleshooting.md` | Diagnostic steps for email delivery issues |

**Folder structures inspected:**

| Folder Path | Purpose |
|-------------|---------|
| `/` (repository root) | Top-level layout and file inventory |
| `app/` | Main application package structure |
| `app/dashboard/` | Dashboard blueprint and views |
| `app/dashboard/views/` | Individual dashboard view modules |
| `app/auth/` | Authentication blueprint and views |
| `app/auth/views/` | Individual auth view modules |
| `app/api/` | API endpoint blueprint |
| `docs/` | Existing documentation directory |

### 0.11.2 User-Provided Attachments

No attachments were provided by the user for this project.

### 0.11.3 External URLs and Figma Screens

No Figma screens or external URLs were referenced in the user's requirements. No design assets were provided.


