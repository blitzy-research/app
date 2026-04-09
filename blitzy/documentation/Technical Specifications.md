# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that captures the **actual runtime behavior** of the SimpleLogin codebase for developer onboarding purposes.

**Category:** Create new documentation
**Documentation Type:** Runtime behavior guide / Developer onboarding reference

The user is onboarding into the SimpleLogin codebase and requires a single, comprehensive reference document that answers specific empirical questions about how the application behaves when executed. The requirements translate into the following documented behaviors:

- **Flask application port binding** — Identify the exact port the Flask/Gunicorn server binds to and what configuration controls it, citing `server.py`, `Dockerfile`, and `example.env`.
- **Startup log output** — Capture and document the actual console output produced when the application initializes under both the development server (`python server.py`) and the production server (`gunicorn wsgi:app`).
- **Health check endpoint behavior** — Document the exact HTTP response body, status code, headers, and content type returned by the `/health` endpoint.
- **Alias creation API response** — Document the exact JSON response returned when a user creates an alias through the API (`POST /api/alias/random/new` or `POST /api/v3/alias/custom/new`), including all fields and their types.
- **Database effects of alias creation** — Identify which table (`alias`), what columns are populated, and what values are written when an alias is created.
- **PostgreSQL failure behavior** — Document the exact error output and failure point when the application attempts to start without a running PostgreSQL instance.

**Inferred Documentation Needs:**
- Based on code analysis: the `app/db.py` module performs an eager `engine.connect()` call at import time, making PostgreSQL availability a hard startup dependency — this behavior should be documented.
- Based on structure: the alias creation spans `app/api/views/new_custom_alias.py`, `app/api/views/new_random_alias.py`, `app/api/serializer.py`, and `app/models.py` — consolidated documentation is needed.
- Based on configuration: the port `7777` is hardcoded in `server.py:588` (dev mode) and `Dockerfile:47` (production) but can be configured via Gunicorn's `-b` flag — this nuance should be captured.
- Based on user journey: understanding startup, health check, and failure behavior forms a critical developer onboarding path that currently lacks dedicated documentation.

### 0.1.2 Special Instructions and Constraints

- **No repository modifications:** The user explicitly requires that no existing files in the source repository be modified. Temporary test scripts may be created but must be cleaned up afterward.
- **Runtime output, not code analysis alone:** The user specifically requests "actual runtime output, not just what the code says should happen." Documentation must include real observed output from running the application.
- **Implementation rule — `SWE-AtlasQnA-Repo`:** The output must be a new markdown document named `<source_branch_name>.md` placed in `blitzy/documentation/`. The branch name is `app_2cd6ee777f8c`, so the file will be `blitzy/documentation/app_2cd6ee777f8c.md`.
- **Answers must be code-grounded:** Do not make assumptions; base all answers on the code as the source of truth.
- **Provide rationale:** Include thinking and rationale behind each answer.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **port binding behavior**, we will **create** a section in `blitzy/documentation/app_2cd6ee777f8c.md` that cites `server.py:588` (`app.run(debug=True, port=7777)`), `Dockerfile:44` (`EXPOSE 7777`), and `Dockerfile:47` (Gunicorn CMD binding to `0.0.0.0:7777`), supplemented by actual observed Gunicorn log output showing `Listening at: http://0.0.0.0:7777`.
- To document **startup logs**, we will **create** a section capturing the actual console output from both `python server.py` (Flask dev server) and `gunicorn wsgi:app -b 0.0.0.0:7777` (production), including config initialization messages, warnings, and worker boot lines.
- To document the **health check**, we will **create** a section showing the actual `curl -sv` output against `/health`, including HTTP status `200`, body `success`, content type `text/html; charset=utf-8`, and relevant headers.
- To document **alias creation API response**, we will **create** a section citing the serializer in `app/api/serializer.py:55-93` (`serialize_alias_info_v2`) and the route handlers in `app/api/views/new_random_alias.py` and `app/api/views/new_custom_alias.py`, including the full JSON response schema.
- To document **database effects**, we will **create** a section citing the `alias` table schema from `app/models.py:1469-1574` and the `Alias.create` method at line 1627, showing column names, types, and defaults.
- To document **PostgreSQL failure**, we will **create** a section showing the actual `psycopg2.OperationalError` and `sqlalchemy.exc.OperationalError` traceback produced when `app/db.py:12` (`connection = engine.connect()`) fails due to an unavailable PostgreSQL server.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **flat, ad-hoc documentation structure** concentrated in a top-level `docs/` directory with no documentation generator framework (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx/conf.py` present). Documentation is authored as standalone Markdown files.

**Current documentation inventory:**

| File | Type | Coverage |
|------|------|----------|
| `README.md` | Self-hosting guide | Comprehensive Docker-based deployment walkthrough |
| `CONTRIBUTING.md` | Contributor guide | Development setup and contribution workflow |
| `SECURITY.md` | Security disclosure | Vulnerability reporting policy |
| `docs/api.md` | API reference | Complete REST API documentation (1108 lines) |
| `docs/oauth.md` | OAuth reference | OAuth2/OIDC flow documentation |
| `docs/troubleshooting.md` | Troubleshooting | Welcome email and alias forwarding diagnostics |
| `docs/code-structure.md` | Code structure | Minimal TODO stub (9 lines, only covers `local_data/`) |
| `docs/build-image.md` | Ops runbook | Docker multi-arch image building |
| `docs/upgrade.md` | Ops runbook | Version upgrade procedures |
| `docs/ssl.md` | Ops guide | TLS/SSL certificate configuration |
| `docs/ufw.md` | Ops guide | Firewall port configuration |
| `docs/ses.md` | Integration guide | Amazon SES relay setup |
| `docs/gmail-relay.md` | Integration guide | Gmail SMTP relay setup |
| `docs/postfix-tls.md` | Integration guide | Postfix TLS submission setup |
| `docs/enforce-spf.md` | Integration guide | SPF enforcement setup |

**Documentation gaps identified for this task:**
- No documentation covering runtime behavior or "what happens when you run the app"
- No developer onboarding guide that explains observed behavior
- `docs/code-structure.md` is essentially empty (a TODO placeholder)
- No documentation explaining Flask app initialization sequence, health check behavior, or database schema runtime effects
- No documentation about failure modes (e.g., missing PostgreSQL)

**Documentation tools and frameworks:**
- No documentation site generator is configured
- API documentation (`docs/api.md`) is handwritten Markdown
- No diagram tool integration detected (no Mermaid configs, PlantUML, etc.)
- No documentation build scripts in `pyproject.toml` or any `Makefile`

### 0.2.2 Repository Code Analysis for Documentation

The following code paths were examined to extract runtime behavior information:

**Application bootstrap chain:**
- `server.py` — Flask app factory (`create_app()`), local development entry (`local_main()`), health check route, blueprint registration, error handlers
- `wsgi.py` — Production WSGI entry point (imports `create_app` from `server`)
- `app/config.py` — Environment variable loading, prints `>>> URL:` and config diagnostics during import
- `app/db.py` — SQLAlchemy engine creation and eager `connection = engine.connect()` at module scope
- `app/log.py` — Logger initialization, prints `>>> init logging <<<` during import
- `app/utils.py` — Word file loading, emits debug log during import

**API alias creation chain:**
- `app/api/views/new_random_alias.py` — Random alias creation route handler
- `app/api/views/new_custom_alias.py` — Custom alias creation route handlers (v2, v3)
- `app/api/serializer.py` — `serialize_alias_info_v2()` function defining JSON response shape
- `app/api/base.py` — API blueprint definition, `require_api_auth` decorator, authentication via `Authentication` header

**Database model chain:**
- `app/models.py` — `Alias` class (line 1469), `ModelMixin` base (line 62), `Alias.create()` override (line 1627)
- `Dockerfile` — `EXPOSE 7777`, `CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777","-w","2","--timeout","15"]`
- `example.env` — Default configuration including `URL=http://localhost:7777`, `DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin`

### 0.2.3 Web Search Research Conducted

No external web search was required for this task. All answers are derived directly from source code analysis and actual runtime execution of the codebase, consistent with the user's directive to base answers on the code as truth.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Module: `server.py` — Flask App Factory and Development Server**
- Public APIs: `create_app()`, `create_light_app()`, `local_main()`, inline `/health` route
- Current documentation: No runtime behavior documentation exists
- Documentation needed: Port binding behavior, startup sequence, health check response, development vs. production entry points

**Module: `wsgi.py` — Production WSGI Entry Point**
- Public APIs: Exposes `app` object for Gunicorn
- Current documentation: Mentioned briefly in `Dockerfile` CMD
- Documentation needed: How Gunicorn invokes this, observed startup log output

**Module: `app/db.py` — Database Connection Bootstrap**
- Public APIs: `engine`, `connection`, `Session`
- Current documentation: None
- Documentation needed: Eager connection behavior at import time, failure mode when PostgreSQL is unavailable

**Module: `app/config.py` — Configuration Loading**
- Public APIs: All exported constants (`DB_URI`, `URL`, `FLASK_SECRET`, etc.)
- Current documentation: `example.env` documents available variables
- Documentation needed: Startup print statements (`>>> URL:`, `Paddle param not set`, etc.) that appear in logs

**Module: `app/api/views/new_random_alias.py` — Random Alias Creation**
- Public APIs: `POST /api/alias/random/new`
- Current documentation: `docs/api.md` describes inputs/outputs in prose form
- Documentation needed: Exact JSON response body with all fields, database row effects

**Module: `app/api/views/new_custom_alias.py` — Custom Alias Creation**
- Public APIs: `POST /api/v2/alias/custom/new`, `POST /api/v3/alias/custom/new`
- Current documentation: `docs/api.md` describes inputs/outputs
- Documentation needed: Exact JSON response body, database row effects, v2 vs v3 differences

**Module: `app/api/serializer.py` — API Response Serialization**
- Public APIs: `serialize_alias_info_v2()`, `AliasInfo` dataclass
- Current documentation: None
- Documentation needed: Complete field listing with types for the alias creation response

**Module: `app/models.py` — Alias ORM Model**
- Public APIs: `Alias` class, `Alias.create()`, `Alias.create_new()`
- Current documentation: None for runtime behavior
- Documentation needed: Table name, column schema, `create()` side effects (rate limiting, event dispatch, audit logging)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No runtime behavior documentation exists** — The entire category of "what does the app actually do when running" is undocumented
- **Startup sequence undocumented** — The chain of module imports, config loading, print statements, and server binding is not captured anywhere
- **Health check response undocumented** — While the route exists in `server.py:213-215`, no documentation describes the exact HTTP response
- **Alias creation JSON schema not explicitly documented** — `docs/api.md` says "Use the same format as in GET /api/aliases/:alias_id" but doesn't repeat the full schema on the creation endpoints
- **Database schema effects undocumented** — No documentation maps API actions to database table changes
- **Failure modes undocumented** — `docs/troubleshooting.md` covers email delivery issues but not application startup failures
- **`docs/code-structure.md` is empty** — Only contains a TODO marker and JWT key generation instructions


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

Per the implementation rule `SWE-AtlasQnA-Repo`, the output is a single Markdown document placed at `blitzy/documentation/app_2cd6ee777f8c.md`. The document structure will be organized around the user's questions, with each section providing both the observed runtime evidence and the code-level rationale:

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
        ├── Overview (purpose and scope of this document)
        ├── 1. Flask App Port Binding
        │   ├── Development server (server.py)
        │   ├── Production server (Gunicorn/Dockerfile)
        │   └── Configuration sources
        ├── 2. Startup Log Output
        │   ├── Gunicorn (production) startup logs
        │   ├── Flask dev server startup logs
        │   └── Config initialization messages
        ├── 3. Health Check Endpoint
        │   ├── Route definition
        │   ├── Response body, status code, headers
        │   └── Observed curl output
        ├── 4. Alias Creation API Response
        │   ├── Random alias (POST /api/alias/random/new)
        │   ├── Custom alias (POST /api/v3/alias/custom/new)
        │   ├── Full JSON response schema
        │   └── Field descriptions with types
        ├── 5. Database Effects of Alias Creation
        │   ├── Target table: alias
        │   ├── Column schema and values
        │   ├── Side-effect tables (alias_used_on, alias_mailbox, daily_metric)
        │   └── Alias.create() method behavior
        ├── 6. PostgreSQL Failure Behavior
        │   ├── Where the failure occurs (app/db.py)
        │   ├── Exact error output
        │   └── Root cause explanation
        └── Source Citations
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract port configuration from `server.py:588`, `Dockerfile:44,47`, and `example.env:6`
- Extract health check definition from `server.py:213-215`
- Extract API response schema from `app/api/serializer.py:55-93` (`serialize_alias_info_v2`)
- Extract database schema from `app/models.py:1469-1574` (Alias class) and observed `\d alias` output
- Extract startup output from actual Gunicorn and Flask dev server execution captures
- Extract PostgreSQL failure output from actual execution with PostgreSQL stopped

**Documentation Standards Applied:**
- Markdown formatting with proper headers
- Code blocks with syntax highlighting for JSON, Python, and shell output
- Source citations as inline references (e.g., `Source: server.py:588`)
- Tables for structured data (column schemas, response fields)
- All content grounded in code evidence and observed runtime output

### 0.4.3 Diagram and Visual Strategy

No diagrams are required for this document. The user's questions are all about concrete, observable runtime output (logs, HTTP responses, database rows, error messages). The documentation will use code blocks and tables to present this information clearly rather than architectural diagrams.


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `server.py`, `wsgi.py`, `Dockerfile`, `app/db.py`, `app/config.py`, `app/log.py`, `app/api/views/new_random_alias.py`, `app/api/views/new_custom_alias.py`, `app/api/serializer.py`, `app/models.py`, `example.env`, `docs/api.md` | Comprehensive runtime behavior reference answering all six user questions with observed output and code citations |

### 0.5.2 New Documentation Files Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Developer onboarding / Runtime behavior reference
Source Code:
  - server.py (lines 127-217, 572-599)
  - wsgi.py (lines 1-3)
  - Dockerfile (lines 44, 47)
  - app/db.py (lines 1-18)
  - app/config.py (lines 65-80, 192-198)
  - app/log.py (lines 67-79)
  - app/api/views/new_random_alias.py (lines 21-117)
  - app/api/views/new_custom_alias.py (lines 28-113, 115-235)
  - app/api/serializer.py (lines 55-93, 252-298)
  - app/api/base.py (lines 11, 16-43)
  - app/models.py (lines 62-65, 1469-1574, 1627-1692)
  - example.env (lines 6, 75)
Sections:
  - Overview: Document purpose and scope
  - Flask App Port Binding: Port 7777 from server.py:588, Dockerfile:44,47
  - Startup Log Output: Gunicorn and Flask dev server captured output
  - Health Check Endpoint: /health route from server.py:213-215, observed curl response
  - Alias Creation API Response: JSON schema from serializer.py:55-93
  - Database Effects: alias table schema from models.py:1469-1574
  - PostgreSQL Failure: Error from db.py:12 when PG unavailable
  - Source Citations: All files referenced
Key Citations:
  server.py, wsgi.py, Dockerfile, app/db.py, app/config.py,
  app/log.py, app/api/serializer.py, app/api/views/new_random_alias.py,
  app/api/views/new_custom_alias.py, app/models.py, example.env, docs/api.md
```

### 0.5.3 Documentation Files to Update Detail

No existing documentation files are updated. Per the user instruction "don't modify the repository," and per the `SWE-AtlasQnA-Repo` rule, only a new file is created in `blitzy/documentation/`.

### 0.5.4 Documentation Configuration Updates

No documentation configuration updates are required. The repository has no documentation site generator (`mkdocs.yml`, `docusaurus.config.js`, etc.) to configure. The new file is a standalone Markdown document.

### 0.5.5 Cross-Documentation Dependencies

- The new document references `docs/api.md` for existing API documentation context (particularly the alias response format documented under `GET /api/v2/aliases` and `GET /api/aliases/:alias_id`)
- The new document references `example.env` for environment variable documentation
- No navigation links, table of contents, or index updates are required since there is no documentation generator


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No external documentation tools or packages are required for this task. The output is a standalone Markdown file that does not require any build step, generator, or rendering pipeline.

The following **runtime dependencies** are relevant because the documentation describes their behavior:

| Registry | Package Name | Version | Relevance to Documentation |
|----------|--------------|---------|---------------------------|
| pip | flask | ^1.1.2 | App factory, health check route, development server |
| pip | gunicorn | ^20.0.4 | Production WSGI server, startup log format |
| pip | sqlalchemy | 1.3.24 | ORM engine, eager connection, Alias model |
| pip | psycopg2-binary | ^2.9.3 | PostgreSQL driver, failure error messages |
| pip | arrow | ^0.16.0 | Timestamp formatting in Alias model and serializer |
| pip | coloredlogs | ^14.0 | Colored log output in development mode |
| pip | sentry-sdk | ^2.16.0 | Optional Sentry integration during startup |
| pip | flask-cors | ^3.0.9 | CORS on /api/* endpoints |
| pip | flask-limiter | ^1.4 | Rate limiting on alias creation endpoints |
| pip | python-dotenv | ^0.14.0 | .env file loading during config initialization |

All versions are as declared in `pyproject.toml`.

### 0.6.2 Documentation Reference Updates

No documentation reference or link updates are required. The new document is self-contained and does not alter any existing links or navigation structures.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (for the user's six questions):**
- Flask port binding documented: 0% — No existing documentation covers observed port behavior
- Startup logs documented: 0% — No existing documentation shows initialization output
- Health check response documented: 0% — Route exists in code but response is not documented anywhere
- Alias API response documented: ~60% — `docs/api.md` describes the format for `GET /api/aliases/:alias_id` but creation endpoints say only "Use the same format"
- Database effects documented: 0% — No documentation maps API actions to table/column writes
- PostgreSQL failure behavior documented: 0% — `docs/troubleshooting.md` covers email delivery issues only

**Target coverage:** 100% of the user's six questions fully answered with:
- Observed runtime output (not just code references)
- Code citations linking each answer to its source
- Rationale explaining why the behavior occurs

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every question posed by the user is answered with both observed evidence and code-level explanation
- JSON response schemas include all fields with their types
- Database column listings include column name, type, nullability, and default value
- Startup logs include the complete sequence from process launch to ready state
- Error output includes the full traceback with the originating file and line

**Accuracy validation:**
- All startup log content was captured from actual execution of `gunicorn wsgi:app -b 0.0.0.0:7777` and `python server.py`
- Health check response was captured from actual `curl -sv http://localhost:7777/health`
- PostgreSQL failure was captured by stopping PostgreSQL and importing `app.db`
- Alias response schema was derived from `serialize_alias_info_v2()` source code in `app/api/serializer.py:55-93`
- Database schema was verified against actual `\d alias` PostgreSQL introspection

**Clarity standards:**
- Each section opens with a direct answer to the user's question
- Technical details follow the answer with code citations
- Observed output is presented in code blocks with appropriate syntax highlighting
- The document uses progressive disclosure — answer first, then rationale

### 0.7.3 Example and Diagram Requirements

- Minimum 1 complete code block per question showing observed runtime output
- JSON examples for the alias creation API response use realistic field values
- Database schema presented as a table with all columns
- No diagrams required — the document focuses on concrete observed output


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — The sole deliverable

**Source files analyzed for content (read-only):**
- `server.py` — Flask app factory, health check route, port binding, development entry point
- `wsgi.py` — Production WSGI entry point
- `Dockerfile` — Port exposure and Gunicorn CMD
- `app/db.py` — SQLAlchemy engine and connection creation
- `app/config.py` — Environment variable loading, startup print diagnostics
- `app/log.py` — Logger initialization and format
- `app/utils.py` — Word file loading
- `app/api/base.py` — API blueprint and authentication
- `app/api/serializer.py` — Alias response serialization
- `app/api/views/new_random_alias.py` — Random alias creation endpoint
- `app/api/views/new_custom_alias.py` — Custom alias creation endpoints (v2, v3)
- `app/models.py` — Alias model, ModelMixin, User model, Alias.create()
- `example.env` — Default configuration values
- `docs/api.md` — Existing API reference (for context)
- `docs/troubleshooting.md` — Existing troubleshooting docs (for gap analysis)
- `docs/code-structure.md` — Existing code structure stub (for gap analysis)
- `pyproject.toml` — Dependency versions and Python version constraint

**Runtime behaviors documented:**
- Application startup (Gunicorn and Flask dev server)
- Health check endpoint (`GET /health`)
- Alias creation API (`POST /api/alias/random/new`, `POST /api/v3/alias/custom/new`)
- Database `alias` table schema and write effects
- PostgreSQL unavailability failure mode

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No existing repository files are modified (per user instruction and `SWE-AtlasQnA-Repo` rule)
- **Test file modifications** — No test files are created or modified in the repository
- **Feature additions or code refactoring** — No code changes of any kind
- **Email handling behavior** — The SMTP inbound processor (`email_handler.py`) is not covered
- **OAuth/OIDC flow behavior** — OAuth endpoints are not part of the user's questions
- **Admin panel behavior** — Flask-Admin interface is not in scope
- **Background job runtime** — `job_runner.py`, `cron.py` behavior is not covered
- **Deployment configuration** — Docker Compose, Postfix, or DNS setup is not covered
- **Existing documentation updates** — `README.md`, `docs/api.md`, and other existing docs are not modified
- **Documentation build/deploy infrastructure** — No documentation generator setup


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** N/A — The output is a standalone Markdown file requiring no build step
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/app_2cd6ee777f8c.md`
- **Diagram generation command:** N/A — No diagrams are included
- **Default format:** Markdown with fenced code blocks for JSON, Python, and shell output
- **Citation requirement:** Every section must reference specific source files with line numbers
- **Style guide:** Follow the user's directive to provide thinking/rationale behind answers, base all claims on code as truth, and present observed runtime output
- **Output location:** `blitzy/documentation/app_2cd6ee777f8c.md` (per `SWE-AtlasQnA-Repo` rule)
- **File naming:** `<source_branch_name>.md` where branch is `app_2cd6ee777f8c`

### 0.9.2 Environment Configuration Used for Runtime Observation

The following environment configuration was used to capture actual runtime output. These values are derived from `example.env` and the project's default configuration:

| Variable | Value Used | Source |
|----------|-----------|--------|
| `URL` | `http://localhost:7777` | `example.env:6` |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:22` |
| `SUPPORT_EMAIL` | `support@sl.local` | `example.env:40` |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:75` |
| `FLASK_SECRET` | `secret` | `example.env:77` |
| `NOT_SEND_EMAIL` | `true` | `example.env:19` |
| `LOCAL_FILE_UPLOAD` | `true` | `example.env:136` |
| `DISABLE_ONBOARDING` | `true` | `example.env:150` |

### 0.9.3 Validation Criteria

The generated document will be considered complete when:
- All six user questions are answered with observed runtime evidence
- Each answer includes code citations to the originating source file(s)
- Each answer includes rationale explaining why the behavior occurs
- No existing repository files have been modified
- The document is placed at `blitzy/documentation/app_2cd6ee777f8c.md`


## 0.10 Rules for Documentation

The following rules are explicitly specified by the user and the project's implementation rules:

- **Do not modify any existing files in the source repository.** Temporary test scripts may be created for runtime observation but must be cleaned up. The deliverable is a new file only.
- **Create a new markdown document named `app_2cd6ee777f8c.md`** placed in the `blitzy/documentation/` directory.
- **Provide thinking and rationale behind all answers.** Each section must explain not just what happens but why it happens, citing the specific code responsible.
- **Do not make assumptions; base answers on the code as the truth.** Every claim must be traceable to a specific file and line number in the repository.
- **Document actual runtime output, not just what the code says should happen.** Include captured console output, HTTP response bodies, error messages, and database schema from actual execution.
- **You can create temporary test scripts, but don't modify the repository and clean up afterward.** Any scripts used to capture runtime behavior must be removed after use.


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files were directly read and analyzed to derive the documentation plan:

| File Path | Purpose in Analysis |
|-----------|-------------------|
| `server.py` | Flask app factory, health check route, port binding (7777), local_main() dev server, blueprint registration, error handlers |
| `wsgi.py` | Production WSGI entry point (2 lines: imports `create_app`, creates `app`) |
| `Dockerfile` | Port exposure (`EXPOSE 7777`), Gunicorn CMD (`-b 0.0.0.0:7777`), Python 3.10 base image |
| `app/db.py` | SQLAlchemy engine/connection/session creation; eager `connection = engine.connect()` at module scope |
| `app/config.py` | Environment variable loading; `DB_URI`, `URL`, `FLASK_SECRET` extraction; startup print diagnostics |
| `app/log.py` | Logger initialization; `>>> init logging <<<` print; log format definition; `LOG` singleton |
| `app/api/base.py` | API blueprint (`/api` prefix); `require_api_auth` decorator; `Authentication` header parsing |
| `app/api/serializer.py` | `serialize_alias_info_v2()` defining full JSON response schema; `AliasInfo` dataclass; `get_alias_info_v2()` |
| `app/api/views/new_random_alias.py` | `POST /api/alias/random/new` handler; response format `jsonify(alias=alias.email, **serialize_alias_info_v2(...))` |
| `app/api/views/new_custom_alias.py` | `POST /api/v2/alias/custom/new` and `POST /api/v3/alias/custom/new` handlers; input validation; mailbox assignment |
| `app/models.py` | `Alias` class (line 1469, `__tablename__ = "alias"`); all column definitions; `Alias.create()` override (line 1627) with rate limiting, event dispatch, audit logging; `ModelMixin` (line 62) with `id`, `created_at`, `updated_at` base columns |
| `pyproject.toml` | Python `^3.10` constraint; all dependency versions; project metadata |
| `example.env` | Default configuration values: `URL=http://localhost:7777`, `DB_URI`, `FLASK_SECRET`, `EMAIL_DOMAIN` |
| `docs/api.md` | Existing API reference (1108 lines); alias response JSON examples; authentication documentation |
| `docs/troubleshooting.md` | Existing troubleshooting guide (email delivery issues only) |
| `docs/code-structure.md` | Existing code structure doc (9 lines, TODO placeholder) |
| `README.md` | Self-hosting deployment guide |
| `tests/conftest.py` | Test fixture setup; demonstrates `create_app()` usage and database setup pattern |
| `tests/test.env` | Test environment configuration; `DB_URI=postgresql://test:test@localhost:15432/test` |

The following folders were explored for structural understanding:

| Folder Path | Purpose in Analysis |
|-------------|-------------------|
| `` (root) | Repository structure overview; identification of all top-level entry points and documentation files |
| `app/` | Application package structure; identification of all modules and subpackages |
| `app/api/` | API layer structure; identification of blueprint, serializer, and views package |
| `app/api/views/` | API endpoint modules; identification of all route handlers |
| `docs/` | Existing documentation inventory; gap analysis |

### 0.11.2 Attachments

No attachments were provided by the user. No Figma screens or external design documents are referenced.

### 0.11.3 Runtime Observations Captured

The following runtime observations were performed on the actual codebase and their output is incorporated into the documentation plan:

| Observation | Method | Key Finding |
|-------------|--------|-------------|
| Gunicorn startup | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 15 --log-level debug` | Logs show `Listening at: http://0.0.0.0:7777`, worker boot with PID, config initialization messages |
| Flask dev server startup | `python server.py` | Logs show `Serving Flask app "server"`, debug mode, reloader process spawn |
| Health check | `curl -sv http://localhost:7777/health` | Returns HTTP 200, body `success`, Content-Type `text/html; charset=utf-8`, Content-Length `7` |
| PostgreSQL failure | Import `app.db` with PostgreSQL stopped | Raises `sqlalchemy.exc.OperationalError` wrapping `psycopg2.OperationalError: connection to server at "localhost" (127.0.0.1), port 5432 failed: Connection refused` at `app/db.py:12` |
| Database schema | `\d alias` via `psql` after table creation | Confirmed 23 columns including `id`, `email`, `user_id`, `mailbox_id`, `enabled`, `note`, `name`, `flags`, `pinned`, `ts_vector`, etc. |


