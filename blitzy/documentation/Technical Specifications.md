# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that comprehensively answers a set of investigative questions about SimpleLogin's alias-creation flow by tracing the complete runtime behavior — from frontend request initiation through backend processing, database mutations, event dispatching, background task triggers, and error-handling paths — through deep source code analysis.

- **Category:** Create new documentation
- **Documentation type:** Technical investigation / Q&A document — a detailed narrative analysis backed by code evidence
- **Target artifact:** A single Markdown file named `app_2cd6ee777f8c.md` placed in the `blitzy/documentation/` directory of the destination repository

The user asks the platform to answer the following questions through code-based analysis (no existing files will be modified):

- **Frontend → Backend request:** What HTTP request (method, URL, headers, body) does the frontend send when a user creates an alias from the dashboard?
- **Backend → Frontend response:** What does the backend return — status codes, JSON payloads, metadata — on both success and failure?
- **Database mutations:** Which tables are inserted into or updated during alias creation, how many tables are touched, and what related entities are created as side-effects?
- **Background tasks and events:** Does the system trigger any background jobs, emit follow-up events, or perform additional asynchronous work beyond the initial write?
- **Error handling:** How do validation failures, database errors, and network issues surface in the API response, logs, and the resulting database state?

### 0.1.2 Special Instructions and Constraints

- **CRITICAL — Read-only policy:** The user explicitly states: *"Don't modify any of the repository source files while investigating."* No existing file in the repository may be edited, deleted, or moved.
- **Temporary scripts allowed but must be cleaned up:** The user allows creation of temporary investigative scripts but requires all temporary artifacts to be removed when done.
- **Observation-based analysis:** The user's intent is to understand how the system behaves by reading the code as the authoritative source of truth — not by assumption or convention. Every claim must be grounded in specific code paths.
- **No assumptions:** Per the project rules, answers must be based on the code and not on assumptions.
- **Provide rationale:** The project rules require thinking and rationale behind all answers.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document the frontend request**, we will trace the Jinja2 template `templates/dashboard/custom_alias.html` form submission (POST to `/dashboard/custom_alias`) and the API endpoint `POST /api/v3/alias/custom/new` in `app/api/views/new_custom_alias.py`, documenting the exact field names, content types, CSRF tokens, and authentication headers.
- To **document the backend response**, we will analyze the return paths of both the dashboard view (`app/dashboard/views/custom_alias.py`) and the API handlers (`app/api/views/new_custom_alias.py`, `app/api/views/new_random_alias.py`), cataloging every status code (201, 400, 409, 412) and response shape.
- To **document database mutations**, we will trace the `Alias.create()` classmethod in `app/models.py` (lines 1628–1692), the `AliasMailbox.create()` in junction tables, the `DailyMetric` increment, the `AliasAuditLog` insertion, and the conditional `AliasUsedOn` and `SyncEvent` writes — identifying every table touched.
- To **document background tasks and events**, we will trace the `EventDispatcher.send_event()` call chain in `app/events/event_dispatcher.py`, the protobuf `AliasCreated` event emission, the PostgreSQL `NOTIFY` on channel `simplelogin_sync_events`, and the New Relic custom event recording.
- To **document error handling**, we will catalog the error branches in the API views (signature expiry, tampered suffix, duplicate alias, consecutive dots, free-plan quota exceeded, mailbox validation failures) and the exception classes `AliasInTrashError` and `IntegrityError`, tracing how each surfaces to the caller.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs were identified:

- **Rate limiting documentation:** Alias creation is gated by three layers of rate limiting — Flask-Limiter (`ALIAS_LIMIT = "100/day;50/hour;5/minute"`), Redis bucket rate limiting (`ALIAS_CREATE_RATE_LIMIT_FREE` / `ALIAS_CREATE_RATE_LIMIT_PAID`), and the `parallel_limiter` Redis distributed lock for concurrency control. These must be documented to fully explain the request flow.
- **Suffix signature and timing:** The alias creation process relies on `itsdangerous.TimestampSigner` with a 600-second expiry window (in `app/alias_suffix.py`), which is an essential part of the flow that affects what the frontend sends and how the backend validates.
- **Auto-creation paths:** Beyond the explicit dashboard/API flows, aliases can also be auto-created via custom domain catch-all rules or directory rules during email forwarding (in `app/alias_utils.py`). These alternate creation paths touch the same database tables and should be referenced for completeness.
- **Dashboard vs. API flow divergence:** The dashboard custom alias creation and the API v3 endpoint differ slightly in validation and response shape, which must be documented distinctly.


## 0.2 Documentation Discovery and Analysis


### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **docs-folder-centric documentation structure** with operational runbook coverage but **no existing documentation covering the alias-creation runtime flow**.

**Search patterns employed:**
- `docs/**/*.md` — 12 Markdown files found in `docs/`
- `README.md`, `CONTRIBUTING.md`, `SECURITY.md` — 3 root-level documentation files
- `docs/api.md` — API reference (the closest existing documentation to the user's question)
- No documentation generators detected (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx/conf.py` found)
- No diagram tools configuration detected (Mermaid diagrams are used inline in the tech spec, not in the repo itself)

**Existing documentation inventory:**

| File | Type | Relevance to Alias Flow |
|------|------|------------------------|
| `docs/api.md` | API reference | High — documents `POST /api/v3/alias/custom/new` and `POST /api/alias/random/new` endpoints with input/output schema |
| `CONTRIBUTING.md` | Development guide | Medium — describes local dev setup (`flask dummy-data`, test user `john@wick.com / password`) |
| `README.md` | Self-hosting guide | Low — focuses on production Docker deployment, not runtime behavior |
| `docs/code-structure.md` | Code overview | Low — minimal TODO-style note about `local_data/` |
| `docs/troubleshooting.md` | Ops runbook | Low — focuses on email forwarding issues, not alias creation |
| `docs/oauth.md` | OAuth reference | None — covers OAuth flows, not alias creation |
| `docs/build-image.md` | Build guide | None |
| `docs/upgrade.md` | Upgrade runbook | None |
| `docs/ssl.md` | Security setup | None |
| `docs/ses.md` | SES relay config | None |
| `docs/gmail-relay.md` | Gmail relay config | None |
| `docs/enforce-spf.md` | SPF hardening | None |
| `docs/ufw.md` | Firewall setup | None |
| `docs/postfix-tls.md` | TLS config | None |
| `SECURITY.md` | Vulnerability disclosure | None |

**Current documentation framework:** None — documentation is plain Markdown without any static site generator
**API documentation tools:** None detected — `docs/api.md` is manually maintained
**Diagram tools:** None configured in repo (Mermaid diagrams are used in the tech spec but not rendered via tooling in the repo)

### 0.2.2 Repository Code Analysis for Documentation

The following source code files and directories were examined to understand the alias-creation flow:

**Core alias creation logic:**
- `app/api/views/new_custom_alias.py` — API endpoints `POST /api/v2/alias/custom/new` and `POST /api/v3/alias/custom/new`
- `app/api/views/new_random_alias.py` — API endpoint `POST /api/alias/random/new`
- `app/dashboard/views/custom_alias.py` — Dashboard view `POST /dashboard/custom_alias`
- `app/dashboard/views/index.py` — Dashboard random alias creation via form POST to `/dashboard/`
- `app/alias_utils.py` — Business logic for alias creation, deletion, auto-creation
- `app/alias_suffix.py` — Suffix generation, signing, and verification
- `app/models.py` — `Alias.create()` (line 1628), `Alias.create_new_random()` (line 1721), `AliasMailbox`, `AliasUsedOn`, `DeletedAlias`, `DomainDeletedAlias`, `DailyMetric`, `AliasAuditLog`, `SyncEvent`

**Event and audit infrastructure:**
- `app/events/event_dispatcher.py` — `EventDispatcher.send_event()` dispatching protobuf events
- `app/events/generated/event_pb2.pyi` — `AliasCreated` protobuf message definition
- `app/alias_audit_log_utils.py` — `emit_alias_audit_log()` with `AliasAuditLogAction.CreateAlias`

**Rate limiting and concurrency:**
- `app/rate_limiter.py` — Redis bucket rate limiter (`check_bucket_limit`)
- `app/parallel_limiter.py` — Redis distributed lock for `alias_creation`
- `app/extensions.py` — Flask-Limiter setup
- `app/config.py` — `ALIAS_LIMIT`, `ALIAS_CREATE_RATE_LIMIT_FREE`, `ALIAS_CREATE_RATE_LIMIT_PAID`

**Serialization and response shaping:**
- `app/api/serializer.py` — `serialize_alias_info_v2()`, `get_alias_info_v2()` response construction
- `app/api/views/alias_options.py` — `GET /api/v5/alias/options` suffix provisioning endpoint

**Templates:**
- `templates/dashboard/custom_alias.html` — Frontend form for custom alias creation
- `templates/dashboard/index.html` — Dashboard landing page with random alias creation forms

**Error handling:**
- `app/errors.py` — `AliasInTrashError`, `SLException` hierarchy

**Development environment:**
- `example.env` — Default configuration for local development
- `app/fake_data.py` — Test data seeding (user: `john@wick.com`, password: `password`)
- `CONTRIBUTING.md` — Local development setup instructions

### 0.2.3 Web Search Research Conducted

No external web search was necessary for this task. The user's questions are entirely answerable from the codebase itself, which serves as the sole authoritative source of truth per the project rules. The documentation will be derived exclusively from code analysis.


## 0.3 Documentation Scope Analysis


### 0.3.1 Code-to-Documentation Mapping

The alias-creation flow spans multiple modules, each contributing a distinct layer of behavior that must be documented:

**Module: `app/dashboard/views/custom_alias.py`**
- Public interface: `custom_alias()` — Flask route handler for `POST /dashboard/custom_alias`
- Current documentation: No dedicated documentation exists; only `docs/api.md` documents the API endpoints
- Documentation needed: Full request/response description for the dashboard flow, form fields, CSRF validation, flash messages, redirect behavior

**Module: `app/dashboard/views/index.py`**
- Public interface: `index()` — Flask route handler for `POST /dashboard/` with `form-name=create-random-email`
- Current documentation: None
- Documentation needed: Random alias creation trigger, generator scheme selection, redirect-with-highlight behavior

**Module: `app/api/views/new_custom_alias.py`**
- Public interface: `new_custom_alias_v2()` at `POST /api/v2/alias/custom/new`, `new_custom_alias_v3()` at `POST /api/v3/alias/custom/new`
- Current documentation: `docs/api.md` documents v3 input/output at a high level
- Documentation needed: Detailed request validation chain, exact error response bodies, status codes, rate limiting behavior

**Module: `app/api/views/new_random_alias.py`**
- Public interface: `new_random_alias()` at `POST /api/alias/random/new`
- Current documentation: `docs/api.md` documents input/output at a high level
- Documentation needed: Hostname-based suggestion logic, mode parameter, alias reuse behavior

**Module: `app/models.py` — `Alias.create()` (lines 1628–1692)**
- Public interface: Class method `Alias.create(**kw)` — the central alias persistence point
- Current documentation: None
- Documentation needed: Exact sequence of operations (rate limit check → trash check → custom domain detection → Session.add → DailyMetric increment → partner flag update → commit/flush → event dispatch → audit log)

**Module: `app/alias_suffix.py`**
- Public interfaces: `get_alias_suffixes()`, `check_suffix_signature()`, `verify_prefix_suffix()`
- Current documentation: None
- Documentation needed: Suffix generation logic, `itsdangerous` signing with 600-second TTL, verification rules

**Module: `app/events/event_dispatcher.py`**
- Public interface: `EventDispatcher.send_event(user, EventContent)`
- Current documentation: None
- Documentation needed: Partner-user gating, protobuf serialization, `SyncEvent` persistence, PostgreSQL `NOTIFY`, New Relic telemetry

**Module: `app/alias_audit_log_utils.py`**
- Public interface: `emit_alias_audit_log(alias, action, message)`
- Current documentation: None
- Documentation needed: Audit log record structure, action types, survivability design (no FK to alias/user)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **No runtime flow documentation exists** — There is no existing document that traces the alias-creation flow from request to response to database mutations end-to-end
- **No database side-effect documentation** — The tables touched during alias creation (`alias`, `daily_metric`, `alias_audit_log`, `sync_event`, `alias_mailbox`, `alias_used_on`) are not documented as a coherent set
- **No event system documentation for alias creation** — The protobuf `AliasCreated` event, the `EventDispatcher` conditional logic (partner-user gating, webhook configuration check), and the PostgreSQL `NOTIFY` mechanism are undocumented
- **No error taxonomy for alias creation** — The complete set of error conditions (quota exceeded, expired signature, tampered suffix, duplicate alias, consecutive dots, invalid prefix, unverified mailbox, alias in trash, DB integrity error) is not cataloged
- **No rate limiting documentation** — The three-tier rate limiting architecture (Flask-Limiter, Redis bucket, Redis distributed lock) is not documented for the alias-creation context
- **Existing `docs/api.md` is incomplete** — It documents the v3 custom alias endpoint input/output but omits validation logic, error response bodies, rate limiting specifics, and database-level side effects


## 0.4 Documentation Implementation Design


### 0.4.1 Documentation Structure Planning

Per the project rules (`SWE-AtlasQnA-Repo`), the output is a single comprehensive Markdown document placed at `blitzy/documentation/app_2cd6ee777f8c.md`. The document structure will follow the user's investigative questions as organizing sections:

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
        ├── 1. Overview — What is the alias-creation flow?
        ├── 2. Frontend Request — What does the frontend send?
        │   ├── 2.1 Dashboard Custom Alias (Web Form POST)
        │   ├── 2.2 Dashboard Random Alias (Web Form POST)
        │   ├── 2.3 API Custom Alias (JSON API)
        │   └── 2.4 API Random Alias (JSON API)
        ├── 3. Backend Response — What does the backend return?
        │   ├── 3.1 Success responses (status codes, payloads)
        │   └── 3.2 Error responses (validation, conflicts, quota)
        ├── 4. Database Changes — What records are created/updated?
        │   ├── 4.1 Tables touched and records inserted
        │   ├── 4.2 The Alias.create() sequence in detail
        │   └── 4.3 Conditional side-effect writes
        ├── 5. Background Tasks and Events
        │   ├── 5.1 EventDispatcher and AliasCreated protobuf event
        │   ├── 5.2 PostgreSQL NOTIFY and SyncEvent
        │   └── 5.3 New Relic telemetry
        ├── 6. Error Handling
        │   ├── 6.1 Validation errors and their API surface
        │   ├── 6.2 Database errors (IntegrityError, AliasInTrashError)
        │   └── 6.3 Rate limiting errors
        └── 7. Summary — End-to-end flow diagram
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract the HTTP request shape from `templates/dashboard/custom_alias.html` form fields and `app/api/views/new_custom_alias.py` request parsing
- Extract the response shape from `app/api/serializer.py` `serialize_alias_info_v2()` and the Flask `jsonify()` return statements
- Trace database writes by following `Alias.create()` in `app/models.py` (lines 1628–1692) through every `Session.add()`, `Session.flush()`, and `Session.commit()` call
- Map events by following `EventDispatcher.send_event()` in `app/events/event_dispatcher.py` through to `PostgresDispatcher.send()` and `Session.execute("NOTIFY ...")`
- Catalog errors by enumerating all `return jsonify(error=...)` and `flash(...)` statements in the alias creation handlers

**Documentation Standards:**
- All claims will include source citations in the format `Source: /path/to/file.py:LineNumber`
- Code examples will use properly fenced blocks with `python` syntax highlighting
- Mermaid diagrams will illustrate the end-to-end flow and the database mutation sequence
- Tables will document the exact HTTP request/response shapes and database table impacts

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created:

- **End-to-end alias creation sequence diagram** — Showing the flow from frontend form submission → Flask view → `Alias.create()` → database writes → event dispatch → response
- **Database mutation flowchart** — Showing every table touched during `Alias.create()` including conditional writes
- **Error handling decision tree** — Showing every validation checkpoint and its corresponding error response
- **Rate limiting layers diagram** — Showing how Flask-Limiter, Redis bucket limiter, and the parallel_limiter distributed lock are stacked


## 0.5 Documentation File Transformation Mapping


### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `app/api/views/new_custom_alias.py`, `app/api/views/new_random_alias.py`, `app/dashboard/views/custom_alias.py`, `app/dashboard/views/index.py`, `app/models.py`, `app/alias_utils.py`, `app/alias_suffix.py`, `app/events/event_dispatcher.py`, `app/alias_audit_log_utils.py`, `app/api/serializer.py`, `app/rate_limiter.py`, `app/parallel_limiter.py`, `app/errors.py`, `app/config.py`, `app/api/views/alias_options.py`, `app/events/generated/event_pb2.pyi`, `templates/dashboard/custom_alias.html`, `templates/dashboard/index.html`, `docs/api.md`, `CONTRIBUTING.md` | Comprehensive Q&A document answering all user questions about the alias-creation flow: frontend requests, backend responses, database mutations, background events, and error handling — with code citations, Mermaid diagrams, and rationale |

- **No existing files will be modified** — per the user's explicit instruction and the `SWE-AtlasQnA-Repo` rule: *"Do not modify any existing files in the source repository."*
- **No files will be deleted** — the task is purely additive.
- **No REFERENCE-mode files** — the output is a standalone Q&A document, not styled after an existing documentation template.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Technical Investigation / Q&A Document
Source Code:
  - app/api/views/new_custom_alias.py (v2 and v3 custom alias API endpoints)
  - app/api/views/new_random_alias.py (random alias API endpoint)
  - app/dashboard/views/custom_alias.py (dashboard custom alias web view)
  - app/dashboard/views/index.py (dashboard random alias creation)
  - app/models.py:1628-1692 (Alias.create classmethod)
  - app/models.py:1721-1760 (Alias.create_new_random)
  - app/models.py:2939-2953 (AliasMailbox model)
  - app/models.py:2331-2348 (AliasUsedOn model)
  - app/models.py:2286-2304 (DeletedAlias model)
  - app/models.py:2558-2590 (DomainDeletedAlias model)
  - app/models.py:3262-3287 (DailyMetric model)
  - app/models.py:3810-3826 (AliasAuditLog model)
  - app/models.py:3759-3807 (SyncEvent model)
  - app/alias_suffix.py (suffix signing, verification, generation)
  - app/alias_utils.py (auto-creation, prefix validation)
  - app/events/event_dispatcher.py (event dispatch to SyncEvent + NOTIFY)
  - app/events/generated/event_pb2.pyi (AliasCreated protobuf message)
  - app/alias_audit_log_utils.py (audit log emission)
  - app/api/serializer.py:39-100 (response serialization)
  - app/api/views/alias_options.py (suffix options endpoint)
  - app/rate_limiter.py (Redis bucket rate limiter)
  - app/parallel_limiter.py (distributed concurrency lock)
  - app/config.py (ALIAS_LIMIT, rate limit configs)
  - app/errors.py (AliasInTrashError, SLException)
  - templates/dashboard/custom_alias.html (frontend form)
  - templates/dashboard/index.html (dashboard landing with alias creation)
Sections:
  - Overview of alias creation flow
  - Frontend request documentation (dashboard web forms and API JSON endpoints)
  - Backend response documentation (success and error payloads with status codes)
  - Database mutation analysis (tables inserted/updated, sequence of operations)
  - Background tasks and event documentation (EventDispatcher, SyncEvent, NOTIFY)
  - Error handling taxonomy (all error conditions, their sources, and their surface)
  - End-to-end flow summary with Mermaid diagrams
Diagrams:
  - Sequence diagram: end-to-end alias creation flow
  - Flowchart: database mutations during Alias.create()
  - Decision tree: error handling paths
  - Layer diagram: rate limiting architecture
Key Citations:
  - app/models.py, app/api/views/new_custom_alias.py, app/api/views/new_random_alias.py
  - app/dashboard/views/custom_alias.py, app/dashboard/views/index.py
  - app/events/event_dispatcher.py, app/alias_audit_log_utils.py
  - app/alias_suffix.py, app/api/serializer.py
  - app/rate_limiter.py, app/parallel_limiter.py, app/config.py
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are needed. The repository does not use a documentation generator (no `mkdocs.yml`, `docusaurus.config.js`, or similar). The output file is a standalone Markdown document that requires no navigation, sidebar, or build configuration changes.


## 0.6 Dependency Inventory


### 0.6.1 Documentation Dependencies

This documentation task produces a standalone Markdown file and does not require any documentation tooling to be installed. However, the following project dependencies are directly relevant to the alias-creation flow being documented and must be understood to produce accurate documentation:

| Registry | Package Name | Version | Purpose in Alias-Creation Flow |
|----------|--------------|---------|-------------------------------|
| pip (Poetry) | flask | ^1.1.2 | Web framework — routes, request handling, jsonify responses, Blueprint registration |
| pip (Poetry) | flask_login | ^0.5.0 | `login_required` decorator, `current_user` proxy for dashboard views |
| pip (Poetry) | Flask-Limiter | ^1.4 | HTTP-level rate limiting via `ALIAS_LIMIT` (100/day; 50/hour; 5/minute) |
| pip (Poetry) | Flask-WTF | ^0.14.3 | CSRF validation via `CSRFValidationForm` in dashboard alias creation |
| pip (Poetry) | SQLAlchemy | 1.3.24 (pinned) | ORM — `Session.add()`, `Session.flush()`, `Session.commit()`, model definitions |
| pip (Poetry) | psycopg2-binary | ^2.9.3 | PostgreSQL driver for database writes |
| pip (Poetry) | Flask-Migrate | ^2.5.3 | Alembic integration for schema migrations |
| pip (Poetry) | redis | ^4.5.3 | Backing store for rate limiter, concurrency lock, sessions |
| pip (Poetry) | itsdangerous | (transitive via Flask) | `TimestampSigner` for alias suffix signing with 600-second TTL |
| pip (Poetry) | email_validator | ^1.1.1 | `validate_email()` for alias email validation in dashboard flow |
| pip (Poetry) | arrow | ^0.16.0 | Timestamp handling for `created_at`, `updated_at`, rate limit buckets |
| pip (Poetry) | newrelic | 8.8.0 (pinned) | `record_custom_event` for telemetry after event dispatch |
| pip (Poetry) | sentry_sdk | ^2.16.0 | Error tracking (not directly in alias creation but wraps all Flask requests) |
| pip (Poetry) | tldextract | ^3.1.2 | Hostname domain extraction in `new_random_alias.py` for prefix suggestions |
| pip (Poetry) | python | ^3.10 | Runtime — project requires Python 3.10+ |

### 0.6.2 Documentation Reference Updates

Not applicable. This task creates a new standalone Markdown file (`blitzy/documentation/app_2cd6ee777f8c.md`) and does not modify any existing documentation. No link updates are required.


## 0.7 Coverage and Quality Targets


### 0.7.1 Documentation Coverage Metrics

The documentation must comprehensively answer **every question posed by the user** with code-backed evidence. Coverage is measured against the five investigation dimensions:

| Investigation Dimension | Source Files to Cover | Target Coverage |
|------------------------|----------------------|-----------------|
| Frontend request shape | `templates/dashboard/custom_alias.html`, `templates/dashboard/index.html`, `app/api/views/new_custom_alias.py`, `app/api/views/new_random_alias.py`, `app/api/views/alias_options.py` | 100% — all request paths (dashboard web form + API JSON) |
| Backend response shape | `app/api/views/new_custom_alias.py`, `app/api/views/new_random_alias.py`, `app/dashboard/views/custom_alias.py`, `app/dashboard/views/index.py`, `app/api/serializer.py` | 100% — all success and error response structures |
| Database mutations | `app/models.py` (Alias.create, AliasMailbox, AliasUsedOn, DailyMetric, AliasAuditLog, SyncEvent), `app/alias_suffix.py` | 100% — every table touched, every record inserted/updated |
| Background tasks and events | `app/events/event_dispatcher.py`, `app/events/generated/event_pb2.pyi`, `app/alias_audit_log_utils.py` | 100% — EventDispatcher chain, protobuf event, NOTIFY, New Relic |
| Error handling | `app/api/views/new_custom_alias.py`, `app/api/views/new_random_alias.py`, `app/dashboard/views/custom_alias.py`, `app/errors.py`, `app/rate_limiter.py`, `app/parallel_limiter.py` | 100% — all validation checkpoints, error responses, exception types |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every claim must include a source citation in the form `Source: file_path:line_number` or `Source: file_path`
- Every HTTP status code returned by alias-creation endpoints must be enumerated
- Every database table written to during alias creation must be identified by table name and model class
- Every error condition must be documented with its trigger condition, error message, and HTTP status code
- The rationale behind each behavior must be explained (per project rule: *"Provide thinking / rationale behind the answers"*)

**Accuracy validation:**
- All code examples must be extracted directly from the repository source files — no fabricated code
- All table names, column names, and function signatures must match the current codebase exactly
- Database write sequences must reflect the actual execution order in `Alias.create()` (rate limit check → trash check → `Session.add` → DailyMetric → flag update → optional commit/flush → EventDispatcher → audit log)

**Clarity standards:**
- Technical accuracy with accessible language — explain Flask, SQLAlchemy, and protobuf concepts briefly for readers unfamiliar with the stack
- Progressive disclosure: start with the high-level flow, then drill into each component
- Consistent terminology: use "alias creation" (not "email generation"), "suffix" (not "domain part"), "dashboard view" (not "web page")

### 0.7.3 Example and Diagram Requirements

- **Minimum 1 Mermaid sequence diagram** showing the end-to-end flow from request to response
- **Minimum 1 Mermaid flowchart** showing the database mutation sequence within `Alias.create()`
- **Minimum 1 table** documenting all error conditions with status codes
- **Minimum 1 table** documenting all database tables touched
- **Short code snippets** (2–3 lines max) extracted from the source where they illuminate specific behavior


## 0.8 Scope Boundaries


### 0.8.1 Exhaustively In Scope

**New documentation file:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the sole deliverable

**Alias creation entry points to document:**
- Dashboard custom alias creation: `POST /dashboard/custom_alias` via `app/dashboard/views/custom_alias.py`
- Dashboard random alias creation: `POST /dashboard/` with `form-name=create-random-email` via `app/dashboard/views/index.py`
- API custom alias creation (v2): `POST /api/v2/alias/custom/new` via `app/api/views/new_custom_alias.py`
- API custom alias creation (v3): `POST /api/v3/alias/custom/new` via `app/api/views/new_custom_alias.py`
- API random alias creation: `POST /api/alias/random/new` via `app/api/views/new_random_alias.py`
- API alias options (suffix provisioning): `GET /api/v4/alias/options` and `GET /api/v5/alias/options` via `app/api/views/alias_options.py`

**Database tables in scope for mutation analysis:**
- `alias` — the primary record created
- `alias_mailbox` — additional mailbox associations (v3 and dashboard custom alias with multi-mailbox)
- `alias_used_on` — hostname tracking when `hostname` parameter is provided
- `daily_metric` — `nb_alias` counter incremented
- `alias_audit_log` — audit record with action `create`
- `sync_event` — protobuf-serialized `AliasCreated` event (conditional on partner-user configuration)

**Supporting code modules in scope:**
- `app/alias_suffix.py` — suffix generation, signing, verification
- `app/alias_utils.py` — `check_alias_prefix()`, auto-creation logic (for reference as an alternate creation path)
- `app/events/event_dispatcher.py` — event dispatch chain
- `app/events/generated/event_pb2.pyi` — `AliasCreated` message schema
- `app/alias_audit_log_utils.py` — audit log emission
- `app/api/serializer.py` — API response serialization
- `app/rate_limiter.py` — bucket-based rate limiting
- `app/parallel_limiter.py` — Redis distributed lock
- `app/errors.py` — `AliasInTrashError` exception
- `app/config.py` — `ALIAS_LIMIT`, `MAX_NB_EMAIL_FREE_PLAN`, rate limit configurations

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No existing repository file will be modified, per the user's explicit instruction: *"Don't modify any of the repository source files while investigating"*
- **Test file modifications** — No test files will be created or modified
- **Email forwarding flow** — The `email_handler.py` forward/reply phase is not part of alias creation (covered separately in the tech spec Section 4.2)
- **Alias deletion flow** — `delete_alias()` in `app/alias_utils.py` is not in scope (the user asks about creation only)
- **Alias update/toggle flow** — Enabling, disabling, or patching aliases after creation is not in scope
- **OAuth/social login** — Authentication flows are not part of alias creation
- **Subscription/billing** — While `can_create_new_alias()` checks subscription status, the billing logic itself is out of scope
- **PGP encryption** — Alias-level PGP settings are not involved in alias creation
- **Docker deployment** — Production deployment is not relevant to understanding the runtime flow
- **Auto-creation via email forwarding** — While `try_auto_create()` in `app/alias_utils.py` is mentioned for completeness, the detailed email handler flow is out of scope; the document focuses on user-initiated creation through the dashboard and API
- **All items not specified by the user** — The scope is strictly bounded to the questions asked


## 0.9 Execution Parameters


### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the output is a standalone Markdown file, no build step required
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/app_2cd6ee777f8c.md`
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown and render in any Mermaid-compatible viewer (GitHub, VS Code, etc.)
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files with path and line numbers
- **Style guide:** Code-as-truth investigative Q&A — no assumptions, only evidence-based claims
- **Documentation validation:** Manual review against the five investigation dimensions (frontend request, backend response, DB mutations, events, error handling)

### 0.9.2 Rules for Documentation

The following documentation-specific rules apply, derived from user instructions and project rules:

- **Do not modify any existing files in the source repository** — This is the most critical constraint, stated by both the user and the `SWE-AtlasQnA-Repo` rule
- **Do not make assumptions; base answers on the code as the truth** — Per the `SWE-AtlasQnA-Repo` rule, every claim must be traceable to a specific file and line
- **Provide thinking and rationale behind the answers** — Per the `SWE-AtlasQnA-Repo` rule, the document must explain *why* the system behaves as it does, not just *what* it does
- **Create a new Markdown document named `app_2cd6ee777f8c.md`** — The filename is derived from the source branch name per the rule: *"Create a new markdown document named `<source_branch_name>.md`"*
- **Place the document in `blitzy/documentation/`** — Per the `SWE-AtlasQnA-Repo` rule
- **Temporary scripts must be cleaned up** — If any temporary investigative scripts are created during the process, they must be removed before completion
- **Focus exclusively on alias-creation flow** — The user's questions are specific to alias creation; do not expand scope to unrelated features
- **Include source code citations for all technical details** — Every database table, function, status code, and error message must be attributed to its source file


## 0.10 References


### 0.10.1 Source Files and Folders Searched

The following files and folders were searched, retrieved, and analyzed to derive the conclusions in this Agent Action Plan:

**Core alias creation handlers (read in full):**
- `app/api/views/new_custom_alias.py` — API endpoints for custom alias creation (v2 and v3)
- `app/api/views/new_random_alias.py` — API endpoint for random alias creation
- `app/dashboard/views/custom_alias.py` — Dashboard custom alias creation web view
- `app/dashboard/views/index.py` — Dashboard landing page with random alias creation
- `app/alias_utils.py` — Alias business logic (creation, deletion, auto-creation, prefix validation)
- `app/alias_suffix.py` — Suffix generation, signing (itsdangerous), and verification

**Data models and ORM (read selectively):**
- `app/models.py` — Lines 62–140 (ModelMixin), 211–215 (AliasGeneratorEnum), 336–460 (User model), 867–920 (can_create_new_alias), 1469–1770 (Alias model with create/create_new/create_new_random), 2286–2330 (DeletedAlias), 2331–2348 (AliasUsedOn), 2558–2590 (DomainDeletedAlias), 2939–2965 (AliasMailbox), 3262–3290 (DailyMetric), 3759–3826 (SyncEvent, AliasAuditLog)

**Event and audit infrastructure (read in full):**
- `app/events/event_dispatcher.py` — EventDispatcher, PostgresDispatcher, GlobalDispatcher
- `app/events/generated/event_pb2.pyi` — Protobuf message definitions (AliasCreated, EventContent, Event)
- `app/alias_audit_log_utils.py` — AliasAuditLogAction enum and emit_alias_audit_log()
- `app/events/__init__.py` — Package marker (empty)

**API serialization and authentication (read selectively):**
- `app/api/serializer.py` — Lines 1–100 (AliasInfo, serialize_alias_info_v2)
- `app/api/views/alias_options.py` — Suffix options endpoints (v4, v5)
- `app/api/base.py` — (folder summary reviewed) Authentication header resolution, require_api_auth decorator

**Rate limiting and concurrency (read in full):**
- `app/rate_limiter.py` — Redis bucket rate limiter (check_bucket_limit)
- `app/parallel_limiter.py` — Redis distributed lock (_InnerLock)

**Configuration and errors (read selectively):**
- `app/config.py` — Grep for ALIAS_LIMIT, ALIAS_CREATE_RATE_LIMIT_FREE/PAID, MAX_NB_EMAIL_FREE_PLAN
- `app/errors.py` — Full file (AliasInTrashError, SLException hierarchy)
- `example.env` — Lines 1–60 (default configuration for local development)

**Documentation and project files (read selectively):**
- `docs/api.md` — Lines 1–80 (table of contents), 340–420 (alias creation endpoint docs)
- `README.md` — Lines 1–100 (self-hosting guide, project overview)
- `CONTRIBUTING.md` — Full file (local dev setup, test user, code structure)
- `pyproject.toml` — Full file (dependencies, Python version, tooling config)

**Templates (searched via grep):**
- `templates/dashboard/custom_alias.html` — Form field names (prefix, signed-alias-suffix, mailboxes)
- `templates/dashboard/index.html` — Form names (create-random-email, create-custom-email)

**Folders explored:**
- Root (`/`) — Full repository structure
- `app/` — Application package structure
- `app/api/` — API layer with views subpackage
- `app/api/views/` — All API view modules
- `app/dashboard/` — Dashboard package structure
- `app/dashboard/views/` — All dashboard view modules
- `app/events/` — Event subsystem structure
- `docs/` — Documentation folder (12 Markdown files)
- `templates/dashboard/` — Dashboard HTML templates (listed)

**Tech spec sections reviewed:**
- Section 1.1 Executive Summary — Project overview and stakeholder context
- Section 4.2 Core Email Processing Workflows — Forward/reply phase flow (for auto-creation reference)
- Section 6.2 Database Design — Complete schema documentation with entity relationships, indexing, and data management

### 0.10.2 Attachments

No attachments were provided for this project. No Figma screens or design files are relevant to this documentation task.


