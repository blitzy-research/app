# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides an empirical, code-grounded technical investigation of SimpleLogin's runtime behavior across three domains: development server startup, authenticated request handling, and background process architecture.

**Documentation Type:** Technical investigation / runtime behavior analysis document

**Category:** Create new documentation — specifically, a single comprehensive Markdown document placed in `blitzy/documentation/app_2cd6ee777f8c.md` answering a series of interrelated questions about what the SimpleLogin application *actually does* at runtime.

The user's questions decompose into four distinct documentation requirements:

- **Requirement 1 — Dev Startup Lifecycle:** Trace the exact sequence of operations that occur when `python server.py` is executed in a development environment, including configuration loading via `app/config.py` and `python-dotenv`, Flask app construction in `create_app()`, extension initialization, blueprint registration, and the final `app.run(debug=True, port=7777)` call. The user wants to know what is printed to the console and which log messages signal readiness.

- **Requirement 2 — Configuration Loading Mechanics:** Document how environment variables are ingested through `dotenv`, how `app/config.py` derives all runtime constants at module import time, and which print statements and fallback-default messages appear during this process.

- **Requirement 3 — Authenticated Request Flow:** Trace how a single HTTP request enters the Flask WSGI application, how user identity is resolved (via Flask-Login session cookie through `load_user()` for browser requests, or via the `Authentication` header and `ApiKey` lookup in `app/api/base.py` for API requests), and how the authentication context is propagated on Flask's `g` object and `current_user` proxy throughout the request lifecycle.

- **Requirement 4 — Background Processes:** Determine whether the `server.py` development startup launches any background jobs, schedulers, or workers automatically. The answer, based on code analysis, is that it does not — `job_runner.py`, `cron.py`, `email_handler.py`, `event_listener.py`, and `monitoring.py` are all separate process entry points that must be started independently.

**Inferred Documentation Needs:**

- Based on code analysis: The `app/config.py` module produces multiple `print()` statements during import (e.g., `">>> URL:"`, `"Paddle param not set"`, `"MAX_NB_EMAIL_FREE_PLAN is not set"`), which form part of the observable startup output.
- Based on structure: The `app/log.py` module prints `">>> init logging <<<"` at import time and disables the Werkzeug logger, which affects what the user sees in the console.
- Based on the authentication layer: Two distinct authentication paths exist — session-based (Flask-Login / browser) and API-key-based (`Authentication` header) — and both must be documented as they carry identity differently.
- Based on deployment architecture: The `Dockerfile` confirms the production entry point is Gunicorn on port 7777, while `local_main()` uses Flask's built-in development server on the same port.

### 0.1.2 Special Instructions and Constraints

**Critical Directives:**
- **No source file modifications:** The user explicitly stated: "Please don't modify any repository source files while investigating. Temporary scripts are fine if needed, but leave the codebase unchanged and clean up anything you create when you're done."
- **Empirical over theoretical:** The user emphasized "I'm not looking for what the code says should happen, I want to know what actually gets printed when you run it." This means the documentation must trace actual execution paths and observable outputs, grounded in the code as the single source of truth.
- **Clean investigation:** Any temporary scripts created for analysis must be removed afterward.

**Implementation Rule (SWE-AtlasQnA-Repo):**
- Create a new Markdown document named `app_2cd6ee777f8c.md`
- Provide thinking and rationale behind the answers
- Base all answers on the code as truth — do not make assumptions
- Do not modify any existing files in the source repository
- Place the generated document in the `blitzy/documentation` directory

**Style Preferences:**
- The document should read as a direct, clear technical narrative answering each question
- Provide rationale and code references for every claim
- Use code excerpts and file-path citations to substantiate all observations

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the dev startup lifecycle, we will **create** `blitzy/documentation/app_2cd6ee777f8c.md` with a section tracing execution from `server.py:local_main()` through `create_app()`, covering every initialization step including `ProxyFix`, SQLAlchemy, session configuration, `limiter.init_app()`, `setup_error_page()`, `init_extensions()`, `register_blueprints()`, `set_index_page()`, `init_admin()`, and the debug toolbar setup.

- To document configuration loading, we will trace `app/config.py` module-level execution, including `load_dotenv()` behavior, the `CONFIG` environment variable, all `print()` and `os.environ[]` calls, and the fallback defaults that produce console output.

- To document authenticated request handling, we will trace both the browser flow (Flask-Login session cookie → `load_user()` callback → `current_user` proxy) and the API flow (`Authentication` header → `authorize_request()` → `g.user` / `g.api_key`), referencing `app/extensions.py`, `app/api/base.py`, `app/session.py`, and the `before_request` / `after_request` hooks in `server.py`.

- To document background processes, we will enumerate all separate process entry points (`job_runner.py`, `cron.py`, `email_handler.py`, `event_listener.py`, `monitoring.py`) and confirm that none are started by `server.py` or `create_app()`.


## 0.2 Documentation Discovery and Analysis


### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **flat documentation structure** with moderate but topic-specific coverage concentrated in operational deployment guides, with no existing documentation addressing runtime startup behavior or request lifecycle internals.

**Documentation files discovered:**

| Path | Format | Topic | Relevance |
|------|--------|-------|-----------|
| `README.md` | Markdown | Self-hosting setup guide (DNS, Docker, Postfix, DKIM) | Medium — describes production deployment, not dev startup |
| `CONTRIBUTING.md` | Markdown | Developer setup, local run instructions, testing, code structure overview | High — contains the canonical `python3 server.py` command and local dev prerequisites |
| `SECURITY.md` | Markdown | Vulnerability disclosure policy | Low |
| `docs/api.md` | Markdown | REST API reference (endpoints, auth headers, request/response examples) | Medium — documents the `Authentication` header convention |
| `docs/oauth.md` | Markdown | OAuth2/OIDC provider flow documentation | Low |
| `docs/code-structure.md` | Markdown | Brief TODO-style note on `local_data/` and JWT key generation | Low |
| `docs/build-image.md` | Markdown | Docker Buildx multi-arch image creation | Low |
| `docs/upgrade.md` | Markdown | Version upgrade runbook | Low |
| `docs/troubleshooting.md` | Markdown | Diagnostic steps for email forwarding issues | Low |
| `docs/ssl.md` | Markdown | TLS/SSL certificate and HTTPS setup | Low |
| `docs/enforce-spf.md` | Markdown | SPF hardening via Postfix PCRE rules | Low |
| `docs/ses.md` | Markdown | Amazon SES relay configuration | Low |
| `docs/gmail-relay.md` | Markdown | Gmail SMTP relay setup | Low |
| `docs/postfix-tls.md` | Markdown | Postfix TLS submission configuration | Low |
| `docs/ufw.md` | Markdown | UFW firewall port rules | Low |
| `example.env` | Env file | Complete environment variable reference with comments | High — documents all configuration knobs |

**Documentation infrastructure findings:**
- **No documentation generator** (no `mkdocs.yml`, `docusaurus.config.js`, `sphinx/conf.py`, or `.readthedocs.yml` detected)
- **No automated API doc generation** (no JSDoc, Sphinx autodoc, or similar tool configuration)
- **No diagram tooling** configured in the repository
- **Documentation hosting/deployment:** None detected — docs are consumed directly from the repository

**Key gap identified:** There is no existing document that traces what happens at runtime when the application starts in development mode, how a request is authenticated and routed, or whether the server launches background processes. `CONTRIBUTING.md` describes *how to run* the server but not *what happens when you do*.

### 0.2.2 Repository Code Analysis for Documentation

**Source files central to answering the user's questions (search patterns applied):**

- **Application bootstrap:** `server.py` — the central Flask factory containing `create_app()`, `local_main()`, and `create_light_app()`
- **WSGI entry point:** `wsgi.py` — production WSGI exposure (`app = create_app()`)
- **Configuration module:** `app/config.py` — all environment-driven settings loaded at import time via `load_dotenv()`
- **Logging setup:** `app/log.py` — logger initialization, format definition, Werkzeug suppression
- **Database layer:** `app/db.py` — SQLAlchemy engine and scoped session creation
- **Flask extensions:** `app/extensions.py` — Flask-Login `LoginManager` and Flask-Limiter initialization
- **Session management:** `app/session.py` — Redis-backed session store implementation
- **Redis services:** `app/redis_services.py` — Redis/Sentinel initialization for sessions and rate limiting
- **API authentication:** `app/api/base.py` — `authorize_request()`, `require_api_auth` decorator, `g.user` assignment
- **Browser authentication:** `app/auth/views/login.py` and `app/auth/views/login_utils.py` — login form handling and `after_login()` MFA routing
- **Build metadata:** `app/build_info.py` — `SHA1` and `BUILD_TIME` constants
- **Background jobs:** `job_runner.py`, `cron.py`, `email_handler.py`, `event_listener.py`, `monitoring.py` — all separate process entry points
- **Cron schedules:** `crontab.yml`, `crontab-all-hosts.yml` — yacron job definitions
- **Dockerfile:** `Dockerfile` — production build and entry point (`gunicorn wsgi:app -b 0.0.0.0:7777`)

**Key directories examined:**
- `app/` — main Flask application package
- `app/api/` — API blueprint, authentication decorators, serializers, and view handlers
- `app/auth/` — browser-facing authentication blueprint and view handlers
- `app/dashboard/` — authenticated dashboard blueprint
- `docs/` — existing documentation files
- `scripts/` — development tooling scripts
- `events/` — event listener framework

### 0.2.3 Web Search Research Conducted

No external web search was required for this documentation task. All questions are answerable through direct code analysis of the repository, which is the user's explicit preference ("base your answers on the code as the truth").


## 0.3 Documentation Scope Analysis


### 0.3.1 Code-to-Documentation Mapping

The documentation to be created requires extracting runtime behavior knowledge from the following modules:

**Module: `server.py` (Application Bootstrap)**
- Public functions: `create_app()`, `create_light_app()`, `local_main()`, `load_user()`, `register_blueprints()`, `set_index_page()`, `setup_error_page()`, `init_extensions()`, `init_admin()`, `setup_openid_metadata()`, `jinja2_filter()`, `setup_do_not_track()`, `register_custom_commands()`
- Current documentation: `CONTRIBUTING.md` mentions running `python3 server.py` but does not explain what it does internally
- Documentation needed: Full startup trace, initialization sequence, blueprint registration, development-mode specifics

**Module: `app/config.py` (Configuration Loading)**
- Public surface: ~120 configuration constants derived from environment variables
- Current documentation: `example.env` lists all variables with comments; no documentation of the loading mechanism itself
- Documentation needed: `load_dotenv()` behavior, `CONFIG` env var, print-statement trace, fallback defaults, derived constants

**Module: `app/log.py` (Logging Initialization)**
- Public surface: `LOG` logger instance, `set_message_id()`, `EmailHandlerFilter`
- Current documentation: None
- Documentation needed: Logger format, Werkzeug suppression, `COLOR_LOG` behavior, the `">>> init logging <<<"` print

**Module: `app/extensions.py` (Flask-Login / Flask-Limiter)**
- Public surface: `login_manager`, `limiter`, `__key_func()`
- Current documentation: None
- Documentation needed: Session protection mode (`"strong"`), rate-limit key function, `load_user()` callback registration

**Module: `app/api/base.py` (API Authentication)**
- Public surface: `authorize_request()`, `require_api_auth`, `require_api_sudo`, `check_sudo_mode_is_active()`
- Current documentation: `docs/api.md` documents the `Authentication` header convention
- Documentation needed: Full request authentication trace — header extraction, `ApiKey` lookup, `g.user` assignment, fallback to `current_user`

**Module: `app/auth/views/login.py` and `login_utils.py` (Browser Login)**
- Public surface: `/auth/login` route, `after_login()`, `LoginForm`
- Current documentation: None (internal browser flow)
- Documentation needed: Credential validation, MFA routing, `login_user()` call, session establishment

**Module: `app/session.py` (Redis Session Store)**
- Public surface: `RedisSessionStore`, `ServerSession`, `logout_session()`
- Current documentation: None
- Documentation needed: Session cookie signing, Redis key structure, TTL behavior

**Module: `app/db.py` (Database Session)**
- Public surface: `engine`, `connection`, `Session` (scoped)
- Current documentation: None
- Documentation needed: Engine creation with `application_name`, scoped session lifecycle

**Modules: `job_runner.py`, `cron.py`, `email_handler.py`, `event_listener.py`, `monitoring.py` (Background Processes)**
- Current documentation: `CONTRIBUTING.md` briefly mentions `job_runner.py` and `email_handler.py`
- Documentation needed: Confirmation that none are started by `server.py`; enumeration of all separate processes and their roles

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, the specific documentation gap is:

- **No runtime behavior documentation exists.** All existing docs describe *what to do* (commands to run, configs to set) but not *what happens next* (initialization sequence, console output, request lifecycle).
- **No authenticated request lifecycle documentation.** `docs/api.md` documents the API surface (endpoints, payloads) but does not trace how a request is authenticated internally.
- **No startup sequence documentation.** The `create_app()` factory function performs ~15 distinct initialization steps, none of which are documented.
- **No background process architecture documentation.** `CONTRIBUTING.md` mentions `job_runner.py` and `email_handler.py` as things to run but does not explain the overall process architecture or confirm that the web server does not spawn them.
- **No configuration loading mechanics documentation.** `example.env` documents available variables but not how they are loaded, parsed, or what console output they produce.


## 0.4 Documentation Implementation Design


### 0.4.1 Documentation Structure Planning

The single output document `blitzy/documentation/app_2cd6ee777f8c.md` will be structured as a technical investigation with the following hierarchy:

```
blitzy/documentation/
└── app_2cd6ee777f8c.md
    ├── Introduction (context and methodology)
    ├── Part 1: Development Server Startup
    │   ├── Entry Point and Invocation
    │   ├── Configuration Loading Sequence
    │   ├── Flask App Construction (create_app())
    │   ├── Development-Mode Enhancements (local_main())
    │   ├── Expected Console Output
    │   └── Ports and Endpoints Exposed
    ├── Part 2: Authenticated Request Handling
    │   ├── Request Entry Point (WSGI / Flask)
    │   ├── Before-Request Hooks
    │   ├── Browser Authentication (Flask-Login Sessions)
    │   ├── API Authentication (Authentication Header)
    │   ├── Identity Propagation (g.user / current_user)
    │   └── After-Request Logging
    ├── Part 3: Background Processes and Schedulers
    │   ├── What server.py Does NOT Start
    │   ├── Separate Process Entry Points
    │   └── Process Architecture Summary
    └── Conclusion
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the startup sequence by tracing `server.py:local_main()` → `create_app()` call graph, documenting each function call in execution order
- Extract configuration loading behavior by reading `app/config.py` top-to-bottom, noting all `print()` calls, `os.environ[]` accesses, and exception-caught fallbacks
- Extract authentication flow by tracing `app/api/base.py:authorize_request()` for API requests and `server.py:load_user()` combined with Flask-Login internals for browser requests
- Extract background process architecture by enumerating all `if __name__ == "__main__"` entry points in root-level Python files
- Source all claims by citing specific file paths and line numbers

**Documentation Standards:**
- Markdown formatting with hierarchical headers (`#`, `##`, `###`)
- Mermaid diagrams for the startup sequence and authentication flow
- Code examples using fenced code blocks with syntax highlighting
- Source citations as inline references: `Source: server.py:572-588`
- Tables for endpoint and blueprint summaries

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create within the document:**

- **Startup Sequence Diagram:** A flowchart showing the execution path from `python server.py` through `local_main()` → `create_app()` → individual initialization steps → `app.run()`
- **Configuration Loading Flow:** A flowchart showing `load_dotenv()` → environment variable reads → `print()` outputs → constant derivation
- **Request Authentication Flow (Browser):** A sequence diagram showing HTTP request → Flask WSGI → `before_request` hooks → Flask-Login `load_user()` → `current_user` proxy → route handler → `after_request` logging
- **Request Authentication Flow (API):** A sequence diagram showing HTTP request → `require_api_auth` decorator → `authorize_request()` → `Authentication` header extraction → `ApiKey` lookup → `g.user` assignment → route handler
- **Process Architecture Diagram:** A component diagram showing the five independent process entry points and their relationships to the database


## 0.5 Documentation File Transformation Mapping


### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `server.py`, `app/config.py`, `app/log.py`, `app/db.py`, `app/extensions.py`, `app/api/base.py`, `app/auth/views/login.py`, `app/auth/views/login_utils.py`, `app/session.py`, `app/redis_services.py`, `app/build_info.py`, `job_runner.py`, `cron.py`, `email_handler.py`, `event_listener.py`, `monitoring.py`, `wsgi.py`, `Dockerfile`, `example.env`, `crontab.yml` | Complete technical investigation document answering all user questions about dev startup, authentication, and background processes with Mermaid diagrams, code citations, and rationale |

**No other files are to be created, updated, or deleted.** The user's implementation rule explicitly requires a single new Markdown document, and the user explicitly prohibits modification of existing repository files.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Technical Investigation / Runtime Behavior Analysis
Source Code References:
  - server.py (lines 1-599) — Flask app factory, local dev entry point, blueprint registration, request hooks, user loader
  - app/config.py (lines 1-667) — Environment variable loading, dotenv integration, all print() statements
  - app/log.py (lines 1-79) — Logger creation, format definition, Werkzeug suppression
  - app/db.py (lines 1-18) — SQLAlchemy engine and scoped session
  - app/extensions.py (lines 1-37) — Flask-Login and Flask-Limiter setup
  - app/api/base.py (lines 1-74) — API authentication decorator, authorize_request()
  - app/auth/views/login.py (lines 1-83) — Browser login route
  - app/auth/views/login_utils.py (lines 1-69) — after_login() MFA routing
  - app/session.py (lines 1-122) — Redis session store
  - app/redis_services.py (lines 1-25) — Redis/Sentinel initialization
  - app/build_info.py (lines 1-2) — SHA1 and BUILD_TIME constants
  - job_runner.py (lines 1-347) — Background job polling loop
  - cron.py (lines 1-60+) — Scheduled maintenance tasks
  - email_handler.py (lines 1-70+) — SMTP inbound handler
  - event_listener.py (lines 1-113) — Event source/sink runner
  - monitoring.py (lines 1-170) — Metrics export to New Relic
  - wsgi.py (lines 1-3) — Production WSGI entry point
  - Dockerfile (lines 1-47) — Container build and CMD
  - example.env (lines 1-198) — Environment variable reference
  - crontab.yml (lines 1-97) — Yacron job schedule definitions
Sections:
  - Introduction (purpose, methodology, scope)
  - Part 1: Development Server Startup
    - Entry Point: server.py __main__ → local_main()
    - Configuration: app/config.py load_dotenv() → env var reads → print outputs
    - App Construction: create_app() initialization sequence (15+ steps)
    - Dev Enhancements: COLOR_LOG, DebugToolbarExtension, debug=True
    - Console Output: Reconstructed expected terminal output
    - Endpoints: Port 7777, all registered blueprints and routes
  - Part 2: Authenticated Request Handling
    - Entry: WSGI → ProxyFix → Flask routing
    - Before-Request: session permanence, timing, referral tracking
    - Browser Auth: Flask-Login cookie → load_user(alternative_id) → User lookup
    - API Auth: Authentication header → ApiKey.get_by(code=) → g.user
    - Identity Propagation: g.user, current_user, g.api_key
    - After-Request: Logging with method, path, args, status, duration
  - Part 3: Background Processes
    - server.py does NOT start any background processes
    - job_runner.py: polls Job table every 10s
    - cron.py + crontab.yml: yacron scheduled tasks
    - email_handler.py: aiosmtpd SMTP server
    - event_listener.py: Postgres LISTEN/NOTIFY event runner
    - monitoring.py: New Relic metric export loop
  - Conclusion
Diagrams:
  - Startup sequence flowchart (Mermaid)
  - Configuration loading flowchart (Mermaid)
  - Browser authentication sequence diagram (Mermaid)
  - API authentication sequence diagram (Mermaid)
  - Process architecture component diagram (Mermaid)
Key Citations: server.py, app/config.py, app/log.py, app/api/base.py, app/extensions.py, app/auth/views/login.py, app/auth/views/login_utils.py, app/session.py, app/db.py, job_runner.py, cron.py, email_handler.py, event_listener.py, monitoring.py
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The repository does not use a documentation generator (no `mkdocs.yml`, `docusaurus.config.js`, or similar). The new file is a standalone Markdown document placed in the `blitzy/documentation/` directory per the implementation rule.

### 0.5.4 Cross-Documentation Dependencies

- The new document does not depend on or link to any existing documentation files
- No navigation or table-of-contents updates are required
- No index or glossary updates are needed
- The document is self-contained with all source citations inline


## 0.6 Dependency Inventory


### 0.6.1 Documentation Dependencies

No documentation tooling packages are required for this task. The output is a single Markdown file with embedded Mermaid diagram syntax. Mermaid diagrams are rendered natively by GitHub, GitLab, and most modern Markdown viewers without any build step.

**Project runtime dependencies relevant to the documentation content** (these are the packages whose behavior is being documented, not tools needed to generate the docs):

| Registry | Package Name | Version | Relevance to Documentation |
|----------|--------------|---------|---------------------------|
| PyPI | flask | ^1.1.2 | Core web framework — app factory, routing, request lifecycle |
| PyPI | flask_login | ^0.5.0 | Session-based user authentication — `login_manager`, `load_user()`, `current_user` |
| PyPI | python-dotenv | ^0.14.0 | Configuration loading — `load_dotenv()` in `app/config.py` |
| PyPI | gunicorn | ^20.0.4 | Production WSGI server — Dockerfile CMD entry point |
| PyPI | SQLAlchemy | 1.3.24 | Database engine and scoped session — `app/db.py` |
| PyPI | psycopg2-binary | ^2.9.3 | PostgreSQL driver — used by SQLAlchemy engine |
| PyPI | flask-debugtoolbar | ^0.11.0 | Development-mode debug toolbar — initialized in `local_main()` |
| PyPI | flask-cors | ^3.0.9 | CORS on `/api/*` endpoints — initialized in `create_app()` |
| PyPI | Flask-Limiter | ^1.4 | Rate limiting — `limiter` in `app/extensions.py` |
| PyPI | flask_admin | ^1.5.6 | Admin panel — `init_admin()` in `server.py` |
| PyPI | flask_profiler | ^1.8.1 | Optional profiling — conditionally initialized in `create_app()` |
| PyPI | sentry_sdk | ^2.16.0 | Error tracking — conditionally initialized at module level in `server.py` |
| PyPI | coloredlogs | ^14.0 | Colored console logging — activated when `COLOR_LOG=true` |
| PyPI | arrow | ^0.16.0 | Timestamp handling — used in API key stats and session timing |
| PyPI | redis | ^4.5.3 | Session store and rate limiting backend — `app/redis_services.py` |
| PyPI | newrelic | 8.8.0 | APM integration — custom events in `after_request` hook |
| PyPI | aiosmtpd | ^1.2 | Async SMTP server — `email_handler.py` entry point |
| PyPI | yacron | ^0.11.1 | Cron scheduler — runs `crontab.yml` job definitions |
| PyPI | python | ^3.10 | Runtime — specified in `pyproject.toml` |

### 0.6.2 Documentation Reference Updates

No documentation reference or link updates are applicable. The new document is self-contained and does not require modifications to any existing file.


## 0.7 Coverage and Quality Targets


### 0.7.1 Documentation Coverage Metrics

**Coverage against user questions:**

| User Question | Coverage Target | Source Files Required |
|---------------|----------------|---------------------|
| How does the backend start up in dev mode? | 100% — Full trace from `python server.py` through `app.run()` | `server.py`, `app/config.py`, `app/log.py`, `app/db.py` |
| How is configuration loaded? | 100% — Complete `load_dotenv()` flow and env var processing | `app/config.py`, `example.env` |
| Which log messages indicate readiness? | 100% — All `print()` calls during startup plus Flask's "Running on" message | `app/config.py`, `app/log.py`, `server.py` |
| What ports/endpoints are exposed? | 100% — Port 7777, all blueprints, `/health`, OpenID endpoints | `server.py`, `app/*/base.py` |
| How is an authenticated request handled? | 100% — Both browser and API authentication paths | `app/api/base.py`, `app/extensions.py`, `server.py` |
| Where does the request first enter? | 100% — WSGI → ProxyFix → Flask routing | `server.py`, `wsgi.py` |
| How is user identity determined? | 100% — `load_user()`, `authorize_request()`, cookie and header paths | `server.py`, `app/api/base.py`, `app/session.py` |
| How is auth context carried through request handling? | 100% — `g.user`, `current_user`, `g.api_key` propagation | `app/api/base.py`, `app/extensions.py` |
| Does the app start background jobs/schedulers? | 100% — Definitive "no" with enumeration of separate processes | `server.py`, `job_runner.py`, `cron.py`, `email_handler.py`, `event_listener.py`, `monitoring.py` |
| What actually gets printed when you run it? | 100% — Reconstructed console output based on code trace | `app/config.py`, `app/log.py`, `server.py` |

**Target coverage: 100%** for all user-specified questions. Every question maps to specific source files and will be answered with code-grounded evidence.

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every claim references a specific source file and line range
- All initialization steps in `create_app()` are documented in execution order
- Both authentication paths (browser session and API key) are fully traced
- All background process entry points are enumerated with their roles
- Expected console output is reconstructed from actual `print()` statements in the code

**Accuracy validation:**
- All code references are verified against the actual source files in the repository
- Function signatures and call sequences match the code
- Configuration variable names and defaults match `app/config.py` and `example.env`
- Blueprint URL prefixes match the actual `register_blueprints()` calls

**Clarity standards:**
- Technical accuracy with direct, conversational language (matching the user's tone)
- Answers are structured to address each question directly
- Progressive disclosure: overview first, then detailed traces
- Mermaid diagrams provide visual reinforcement of textual explanations

**Maintainability:**
- Source citations with file paths enable future verification against code changes
- Modular section structure allows updating individual answers independently
- No dependency on external tools or build steps

### 0.7.3 Example and Diagram Requirements

- **Minimum code citations per answer:** Every answer must reference at least one specific source file with line context
- **Diagram types required:** Flowchart (startup), sequence diagram (authentication), component diagram (process architecture)
- **Expected console output:** Reconstructed from `print()` statements in `app/config.py` and `app/log.py`, plus Flask's built-in dev server output
- **Code example testing:** Not applicable — this is a read-only investigation with no executable code modifications


## 0.8 Scope Boundaries


### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the single output document

**Source files analyzed for documentation content (read-only):**
- `server.py` — application bootstrap, Flask factory, request hooks, user loader
- `wsgi.py` — production WSGI entry point
- `app/config.py` — configuration loading and environment variable processing
- `app/log.py` — logging initialization and format
- `app/db.py` — SQLAlchemy engine and session
- `app/extensions.py` — Flask-Login and Flask-Limiter
- `app/session.py` — Redis-backed session store
- `app/redis_services.py` — Redis/Sentinel initialization
- `app/build_info.py` — build metadata constants
- `app/api/base.py` — API authentication decorators
- `app/api/views/auth.py` — API login endpoint
- `app/auth/base.py` — auth blueprint definition
- `app/auth/views/login.py` — browser login route
- `app/auth/views/login_utils.py` — post-login MFA routing
- `app/dashboard/base.py` — dashboard blueprint definition
- `app/developer/base.py` — developer blueprint definition
- `app/discover/base.py` — discover blueprint definition
- `app/internal/base.py` — internal blueprint definition
- `app/monitor/base.py` — monitor blueprint definition
- `app/oauth/base.py` — OAuth blueprint definition
- `app/onboarding/base.py` — onboarding blueprint definition
- `app/phone/base.py` — phone blueprint definition
- `app/payments/paddle.py` — Paddle callback setup
- `app/payments/coinbase.py` — Coinbase Commerce setup
- `job_runner.py` — background job runner
- `cron.py` — scheduled maintenance tasks
- `email_handler.py` — SMTP email handler
- `event_listener.py` — event processing CLI
- `monitoring.py` — metrics export
- `crontab.yml` — yacron job definitions
- `example.env` — environment variable documentation
- `Dockerfile` — container build and entry point
- `CONTRIBUTING.md` — developer setup reference
- `pyproject.toml` — dependency manifest and Python version

**Topics in scope:**
- Development server startup sequence and initialization order
- Configuration loading mechanism (`load_dotenv`, env vars, `print()` output)
- Console output reconstruction (what actually gets printed)
- Ports and endpoints exposed in development mode
- Browser-based authentication flow (Flask-Login sessions)
- API-based authentication flow (`Authentication` header / `ApiKey`)
- Identity propagation through request context (`g.user`, `current_user`)
- Request hooks (`before_request`, `after_request`)
- Background process enumeration and confirmation that they are NOT auto-started
- Process architecture overview (webapp, email handler, job runner, cron, event listener, monitoring)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing repository files will be modified, as per user instruction
- **Test file modifications:** No test files will be created or changed
- **Feature additions or code refactoring:** This is a read-only documentation exercise
- **Deployment configuration changes:** No Docker, Postfix, or infrastructure changes
- **Email handler internals:** The SMTP forwarding/reply logic in `email_handler.py` is out of scope beyond confirming it is a separate process
- **Cron job implementation details:** Individual cron task logic (e.g., HIBP scanning, subscription notifications) is out of scope beyond listing them
- **OAuth provider flow internals:** The OAuth2/OIDC provider implementation is out of scope beyond confirming the `/oauth` and `/oauth2` blueprints are registered
- **Database schema and model details:** ORM model internals are out of scope beyond their role in authentication (e.g., `User.get_by()`, `ApiKey.get_by()`)
- **Frontend/template layer:** Jinja templates and static assets are out of scope
- **Production deployment specifics:** Gunicorn configuration, NGINX proxy, Docker Compose orchestration are out of scope except as contrast to the dev server
- **Unrelated documentation:** SSL setup, SPF enforcement, SES relay, Gmail relay, UFW firewall, upgrade procedures — these existing docs are not affected


## 0.9 Execution Parameters


### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file with no build step required.
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/app_2cd6ee777f8c.md`. Mermaid diagrams render natively on GitHub/GitLab.
- **Diagram generation command:** Not applicable — Mermaid diagrams are embedded inline in the Markdown and rendered by the viewing platform.
- **Documentation deployment command:** Not applicable — the document is committed directly to the repository.
- **Default format:** Markdown with embedded Mermaid diagrams.
- **Citation requirement:** Every section references source files with path and line context. Format: `Source: <file_path>:<line_range>`.
- **Style guide:** The document uses a direct, investigative tone matching the user's conversational style. Technical precision with accessible explanations. Code excerpts are kept brief (2-3 lines) with full-path references for deeper reading.
- **Documentation validation:** Manual review — verify that every claim traces to an actual line of code in the referenced file. No automated linting or link checking is applicable for this standalone document.

### 0.9.2 File Placement and Naming Convention

Per the implementation rule `SWE-AtlasQnA-Repo`:
- **Directory:** `blitzy/documentation/` (to be created if it does not exist)
- **File name:** `app_2cd6ee777f8c.md` (derived from the source branch name `app_2cd6ee777f8c`)
- **Full path:** `blitzy/documentation/app_2cd6ee777f8c.md`


## 0.10 Rules for Documentation


The following rules are derived from the user's explicit instructions and the implementation rule:

- **Do not modify any existing files in the source repository.** The investigation is read-only. Only the new Markdown file in `blitzy/documentation/` is created.
- **Base all answers on the code as the single source of truth.** Do not make assumptions. Every claim must trace to a specific line of code or configuration file.
- **Provide thinking and rationale behind the answers.** The document should explain *why* the code behaves as it does, not merely *what* it does.
- **Temporary scripts are acceptable for investigation but must be cleaned up.** Any helper scripts created during analysis must be removed before completion.
- **Focus on what actually gets printed, not what the code says should happen.** Trace the actual execution path and reconstruct observable console output from `print()` statements and Flask/Werkzeug log output.
- **The document must be placed in `blitzy/documentation/` with the filename `app_2cd6ee777f8c.md`.** This is the naming convention specified by the `SWE-AtlasQnA-Repo` implementation rule.
- **Use Mermaid diagrams for all major flows.** Startup sequence, authentication paths, and process architecture should be visually illustrated.
- **Include source code citations for all technical details.** Format: `Source: <file>:<line_range>` or inline code references.
- **Keep code excerpts brief** (2-3 lines maximum) — point readers to the source file for full context.


## 0.11 References


### 0.11.1 Files and Folders Searched

The following files and folders were systematically searched and analyzed to derive the conclusions in this Agent Action Plan:

**Root-level entry points and configuration:**

| File | Purpose | Key Findings |
|------|---------|-------------|
| `server.py` | Flask app factory and dev server entry point | `local_main()` sets `COLOR_LOG=True`, enables debug toolbar, runs on port 7777; `create_app()` performs 15+ initialization steps; `load_user()` resolves users by `alternative_id` |
| `wsgi.py` | Production WSGI entry point | Single line: `app = create_app()` — exposes WSGI app for Gunicorn |
| `example.env` | Environment variable reference | 198 lines documenting all configuration knobs; `URL=http://localhost:7777` default |
| `pyproject.toml` | Dependency manifest | Python ^3.10, Flask ^1.1.2, SQLAlchemy 1.3.24, Poetry-managed |
| `Dockerfile` | Container build definition | Multi-stage build, Python 3.10, exposes port 7777, CMD is Gunicorn |
| `job_runner.py` | Background job processor | Polls `Job` table every 10 seconds in infinite loop; separate process |
| `cron.py` | Scheduled maintenance tasks | Stats, cleanup, HIBP checks, subscription notifications; separate process |
| `crontab.yml` | Yacron job schedule | 16 scheduled jobs with cron expressions |
| `email_handler.py` | SMTP inbound email handler | aiosmtpd-based; handles forward and reply phases; separate process |
| `event_listener.py` | Event processing CLI | PostgresEventSource and HttpEventSink; separate process |
| `monitoring.py` | Metrics export to New Relic | Logs Postfix queues, DB connections, event counts; 60-second loop; separate process |
| `init_app.py` | Database seeding | Loads PGP keys, adds SL domains; uses `create_light_app()` context |
| `shell.py` | Interactive admin shell | IPython embed with model imports |
| `CONTRIBUTING.md` | Developer setup guide | Contains `python3 server.py` command, prerequisites, code structure overview |
| `README.md` | Self-hosting guide | Production deployment instructions |

**App package — core modules:**

| File | Purpose | Key Findings |
|------|---------|-------------|
| `app/config.py` | Configuration constants | `load_dotenv()` at import time; ~120 constants from env vars; multiple `print()` calls during loading |
| `app/log.py` | Logging initialization | Creates `LOG` logger with custom format; prints `">>> init logging <<<"` at import; disables Werkzeug logger |
| `app/db.py` | SQLAlchemy engine/session | Creates engine with `application_name` from `DB_CONN_NAME`; scoped session |
| `app/extensions.py` | Flask-Login and Limiter | `login_manager.session_protection = "strong"`; rate-limit key uses `current_user.id` or IP |
| `app/session.py` | Redis session store | `RedisSessionStore` with signed cookies, pickle serialization, TTL management |
| `app/redis_services.py` | Redis initialization | Supports both standalone Redis and Sentinel; initializes sessions, parallel limiter, rate limiter |
| `app/build_info.py` | Build metadata | `SHA1 = "dev"`, `BUILD_TIME = "1652365083"` |
| `app/rate_limiter.py` | Bucket-based rate limiting | Redis-backed bucket rate limits with configurable thresholds |

**App package — authentication and API:**

| File | Purpose | Key Findings |
|------|---------|-------------|
| `app/api/base.py` | API authentication | `authorize_request()` reads `Authentication` header, looks up `ApiKey`, assigns `g.user` and `g.api_key` |
| `app/api/views/auth.py` | API login endpoint | `/api/auth/login` validates credentials, creates `ApiKey`, returns JSON |
| `app/auth/base.py` | Auth blueprint | URL prefix `/auth`, template folder `templates` |
| `app/auth/views/login.py` | Browser login route | `/auth/login`, rate-limited at 10/min, delegates to `after_login()` |
| `app/auth/views/login_utils.py` | Post-login routing | `after_login()` checks FIDO, TOTP, then calls `login_user()` and sets `sudo_time` |

**App package — blueprints examined:**

| File | Blueprint Name | URL Prefix |
|------|---------------|------------|
| `app/auth/base.py` | `auth` | `/auth` |
| `app/dashboard/base.py` | `dashboard` | `/dashboard` |
| `app/api/base.py` | `api` | `/api` |
| `app/internal/base.py` | `internal` | `/internal` |
| `app/oauth/base.py` | `oauth` | `/oauth` and `/oauth2` |
| `app/onboarding/base.py` | `onboarding` | (registered without explicit prefix in `register_blueprints()`) |
| `app/developer/base.py` | `developer` | (registered without explicit prefix in `register_blueprints()`) |
| `app/discover/base.py` | `discover` | (registered without explicit prefix in `register_blueprints()`) |
| `app/monitor/base.py` | `monitor` | (registered without explicit prefix in `register_blueprints()`) |
| `app/phone/base.py` | `phone` | (registered without explicit prefix in `register_blueprints()`) |

**Folders searched:**

| Folder | Depth | Findings |
|--------|-------|----------|
| `` (root) | Level 0 | 28 files + 14 subdirectories identified |
| `app/` | Level 1 | 48 modules + 17 subpackages; central Flask application code |
| `app/api/` | Level 2 | Blueprint base, serializer, and views subpackage |
| `app/api/views/` | Level 3 | 17 view modules covering all API endpoints |
| `app/auth/` | Level 2 | Blueprint base and views subpackage |
| `app/auth/views/` | Level 3 | 20 view modules for all auth flows |
| `app/dashboard/` | Level 2 | Blueprint base and views subpackage |
| `docs/` | Level 1 | 13 documentation files; no startup or auth lifecycle docs found |
| `scripts/` | Level 1 | 6 shell scripts for build, migration, testing |

### 0.11.2 Attachments

No attachments were provided by the user for this project.

### 0.11.3 Figma Screens

No Figma screens were referenced or provided for this project.

### 0.11.4 Tech Spec Sections Referenced

| Section | Purpose |
|---------|---------|
| 1.1 Executive Summary | Project overview and stakeholder context |
| 4.4 Authentication Workflows | Detailed auth flow diagrams and MFA routing logic |


