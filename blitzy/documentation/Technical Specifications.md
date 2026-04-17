# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **conduct a series of live behavioral investigations against the running SimpleLogin application** and document the observed results in a comprehensive markdown report. The requirement is explicitly empirical — the user wants evidence from actually executing the system, not static code analysis. The following discrete investigation areas have been identified:

- **API Authentication with Browser Session**: Confirm the HTTP status code and response body structure returned when an authenticated browser session (cookie) is used to make API calls without an `Authentication` header carrying an API key.
- **Privileged Operation Access via Session Cookie**: Determine the specific HTTP status code and exact error message text returned when a browser-session-authenticated user (with no API key) attempts to access an endpoint protected by the `require_api_sudo` decorator.
- **Session Storage Inspection**: Capture raw bytes from the Redis session backend, identify the serialization format, enumerate all keys present in an authenticated session's deserialized data, and document how the session key (Redis key) is structured.
- **Session ID Behavior During Login**: Capture the session identifier from the cookie before login, perform the login flow, and compare whether the session ID changes or persists through authentication.
- **Email Header Forwarding Behavior**: Determine which of three specific headers (a custom `X-` header, a `Received` header, and a `Reply-To` header) survive the email forwarding pipeline and which are stripped.
- **Alias Creation Token Expiration**: Experimentally verify the time window (max_age) after which alias creation suffix tokens become invalid.
- **API Key Usage Statistics Tracking**: Make several API calls with the same API key and observe which database fields get updated and their actual values.
- **Failed Login Response and Logging**: Document the specific HTTP response details and server log messages produced when a login attempt fails with incorrect credentials.

### 0.1.2 Special Instructions and Constraints

- **CRITICAL: Read-Only Source Constraint**: The user explicitly states: "Don't modify any source files in the repository." No existing file in the source repository may be altered.
- **Test Script Allowance**: The user permits creation of temporary test scripts to observe behavior, with the stipulation: "just clean them up afterward."
- **Output Artifact**: Per the `SWE-AtlasQnA-Repo` implementation rule, the deliverable is a markdown document named `<source_branch_name>.md` placed in the `blitzy/documentation` directory. The source branch name is `app_2cd6ee777f8c`.
- **Evidence-Based Answers**: The user requires that all answers be derived from "actually running the system rather than just reading the code." Code reading may inform experimental design, but all conclusions must be supported by runtime observations.
- **Empirical Methodology**: Each question must be answered by running the system (Flask server, PostgreSQL database, Redis session store) and collecting actual HTTP responses, log output, and database state — not by reasoning from source code alone.

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- To **verify API session-cookie authentication behavior**, we will start the Flask server with Redis sessions enabled, create a test user, log in via the browser auth flow to obtain a valid session cookie, and then issue API requests with that cookie but without the `Authentication` header — capturing the exact HTTP status code and JSON response body.
- To **determine the privileged operation response**, we will issue a `DELETE /api/user` request (which is protected by `require_api_sudo`) using only the browser session cookie and capture the resulting HTTP status code and error JSON payload.
- To **inspect session storage**, we will query Redis directly for session keys, retrieve the raw bytes, identify the pickle serialization protocol, and deserialize the data to enumerate all present keys and their types.
- To **test session ID fixation behavior**, we will capture the session ID from the `slapp` cookie before and after the login POST, comparing the UUID portion to determine if Flask-Login or the session store regenerates it.
- To **test email header forwarding**, we will programmatically construct an email `Message` object with `X-Custom-Header`, `Received`, and `Reply-To` headers, then invoke the `delete_all_headers_except` function used in `forward_email_to_mailbox` with the same `headers_to_keep` list, and observe which headers survive.
- To **verify token expiration**, we will sign a test suffix using the `itsdangerous.TimestampSigner` with the `CUSTOM_ALIAS_SECRET`, then call `check_suffix_signature` which enforces `max_age=600` and confirm the boundary.
- To **track API key usage statistics**, we will record the `times` and `last_used` fields of an `ApiKey` row before and after multiple API calls, observing the incremented counter and updated timestamp.
- To **capture failed login behavior**, we will submit an incorrect password via the login form and capture both the HTTP response (status code, flash message in HTML) and the server log output.


## 0.2 Repository Scope Discovery

### 0.2.1 Comprehensive File Analysis

The SimpleLogin repository is a production-grade Python/Flask monolith for email aliasing and privacy. The investigation areas touch the following existing source files and modules, all of which were inspected but will **not** be modified:

**Authentication and API Authorization Files**

| File Path | Relevance | Key Findings |
|-----------|-----------|--------------|
| `app/api/base.py` | Core API auth logic | Contains `authorize_request()`, `require_api_auth`, `require_api_sudo` decorators; fallback from API key to `current_user.is_authenticated`; usage tracking (`last_used`, `times`); sudo returns HTTP 440 "Need sudo" |
| `app/auth/views/login.py` | Login form handler | Validates credentials, flashes "Email or password incorrect" on failure, emits `LoginEvent` |
| `app/auth/views/login_utils.py` | Post-login routing | `after_login()` calls `login_user()`, sets `session["sudo_time"]`, routes through MFA if enabled |
| `app/extensions.py` | Flask-Login config | `login_manager.session_protection = "strong"` |
| `server.py` | App bootstrap | Session cookie config (`slapp`), Redis session init, error handlers (500 for API exceptions), `@login_manager.user_loader` |

**Session Management Files**

| File Path | Relevance | Key Findings |
|-----------|-----------|--------------|
| `app/session.py` | Redis session store | `RedisSessionStore` uses `pickle` serialization, HMAC-signed session IDs via `itsdangerous.Signer`, 7-day TTL for authenticated sessions, 300s TTL for unauthenticated |
| `app/redis_services.py` | Redis initialization | Configures `RedisSessionStore` with read/write Redis from `MEM_STORE_URI` |
| `app/config.py` | Configuration | `SESSION_COOKIE_NAME = "slapp"`, `FLASK_SECRET`, `MEM_STORE_URI`, `CUSTOM_ALIAS_SECRET` |

**Email Forwarding and Header Processing Files**

| File Path | Relevance | Key Findings |
|-----------|-----------|--------------|
| `email_handler.py` | SMTP inbound handler | `forward_email_to_mailbox()` calls `delete_all_headers_except()` with a defined keep-list; Reply-To is stripped then optionally re-added as reverse-alias |
| `app/email_utils.py` | Email utility functions | `delete_all_headers_except()` implementation; `add_or_replace_header()` |
| `app/email/headers.py` | Header constants | Defines `REPLY_TO`, `RECEIVED`, `MIME_HEADERS`, `SL_DIRECTION`, etc. |

**Alias Token and API Key Model Files**

| File Path | Relevance | Key Findings |
|-----------|-----------|--------------|
| `app/alias_suffix.py` | Alias suffix signing | `itsdangerous.TimestampSigner` with `CUSTOM_ALIAS_SECRET`; `check_suffix_signature()` uses `max_age=600` |
| `app/models.py` | Data models | `ApiKey` model with `code`, `last_used` (ArrowType), `times` (Integer), `sudo_mode_at`; `User` model with `alternative_id` for Flask-Login |
| `app/pw_models.py` | Password mixin | `PasswordOracle` with bcrypt hashing via `bcrypt.hashpw()` / `bcrypt.checkpw()` |

**Event and Logging Files**

| File Path | Relevance | Key Findings |
|-----------|-----------|--------------|
| `app/events/auth_event.py` | Login event tracking | `LoginEvent` with `ActionType.failed` sends custom New Relic event |
| `app/log.py` | Logging setup | Configures `coloredlogs` with structured log format |

### 0.2.2 Integration Point Discovery

- **API endpoints connecting to auth**: All endpoints under `app/api/views/*.py` use `require_api_auth` (approximately 30+ endpoints) or `require_api_sudo` (only `DELETE /api/user` in `app/api/views/user.py`)
- **Database models affected**: `ApiKey` (usage tracking fields), `User` (authentication fields), session data in Redis
- **Service classes**: `RedisSessionStore` for session management, `itsdangerous.TimestampSigner` for token signing
- **Error handlers**: `server.py` line 388 catches all `Exception` types for `/api/` paths and returns `{"error": "Internal error"}` with HTTP 500

### 0.2.3 New File Requirements

Since the implementation rule mandates creating a single markdown document and prohibits modifying existing repository files, the only new file is:

- **CREATE**: `blitzy/documentation/app_2cd6ee777f8c.md` — Comprehensive markdown document containing all experimental observations, rationale, and evidence-based answers to each investigation question

No new source files, test files, or configuration files will be created or left in the repository.


## 0.3 Dependency Inventory

### 0.3.1 Private and Public Packages

The following packages are directly relevant to the behavioral investigation, as extracted from `pyproject.toml` and `poetry.lock`:

| Registry | Package | Version (poetry.lock) | Purpose |
|----------|---------|----------------------|---------|
| PyPI | `flask` | 1.1.2 | Web framework; routes, sessions, request context |
| PyPI | `flask-login` | 0.5.0 | User session management; `current_user`, `login_user()` |
| PyPI | `werkzeug` | 1.0.1 | WSGI toolkit; HTTP response handling |
| PyPI | `itsdangerous` | 1.1.0 | Session ID HMAC signing; `TimestampSigner` for alias tokens |
| PyPI | `redis` | 4.6.0 | Redis client for session storage and rate limiting |
| PyPI | `sqlalchemy` | 1.3.24 | ORM for PostgreSQL; `ApiKey`, `User` model queries |
| PyPI | `psycopg2-binary` | 2.9.3+ | PostgreSQL driver |
| PyPI | `bcrypt` | 3.2.0 | Password hashing for login verification |
| PyPI | `arrow` | 0.16.0 | Timestamp handling for `ApiKey.last_used` |
| PyPI | `flask-wtf` | 0.14.3 | CSRF protection on login forms |
| PyPI | `flask-limiter` | 1.4 | Rate limiting on login endpoint (10/minute) |
| PyPI | `sentry-sdk` | 2.16.0+ | Error tracking; event logging |
| PyPI | `newrelic` | 8.8.0 | APM; `LoginEvent` custom event recording |
| PyPI | `aiosmtpd` | 1.4.2 | SMTP inbound handler for email forwarding |
| PyPI | `dkimpy` | 1.1.8 | DKIM signing for outbound emails |

### 0.3.2 Runtime Infrastructure

| Component | Version | Purpose |
|-----------|---------|---------|
| Python | ^3.10 (target: py310 in `pyproject.toml`) | Application runtime |
| PostgreSQL | 16 | Primary database for user, API key, and alias data |
| Redis | 7.0+ | Session storage, rate limiting, concurrency locks |

### 0.3.3 Dependency Updates

No dependency changes are required. The investigation is read-only against the existing codebase. All packages listed above are used as-is from the existing `poetry.lock` manifest.


## 0.4 Integration Analysis

### 0.4.1 Existing Code Touchpoints

The investigation exercises multiple integration points across the SimpleLogin architecture. No modifications are made, but the following touchpoints are actively tested and observed:

**API Authentication Pipeline** (`app/api/base.py`)

- `authorize_request()` (lines 16–42): The central function tested when making API calls with and without the `Authentication` header. When no API key is present and `current_user.is_authenticated` is True, the function sets `g.user = current_user` and `g.api_key = None`, allowing the request to proceed.
- `check_sudo_mode_is_active()` (lines 46–49): Tested indirectly when accessing the sudo-protected endpoint. When `g.api_key` is `None` (cookie-only auth), calling `api_key.sudo_mode_at` on `None` raises `AttributeError`, which is caught by Flask's generic exception handler.
- `require_api_sudo` decorator (lines 63–71): The decorator wrapping `DELETE /api/user` that triggers the 500 error path during cookie-only access.

**Session Management Pipeline** (`app/session.py`)

- `open_session()` (lines 69–79): Validates the HMAC-signed session ID from the `slapp` cookie, retrieves pickle-serialized data from Redis via `session:{uuid}` key.
- `save_session()` (lines 81–109): Serializes session dict via `pickle.dumps()`, stores with appropriate TTL (7 days for authenticated, 300 seconds for unauthenticated), sets HMAC-signed cookie.
- `purge_session()` (lines 62–67): Only called during logout; NOT called during login — this is why the session ID persists through the login flow.

**Login Flow** (`app/auth/views/login.py` and `login_utils.py`)

- `login()` view (lines 21–76): Handles credential validation, emits `LoginEvent`, flashes error messages for failures.
- `after_login()` (lines 12–46 in `login_utils.py`): Calls `login_user(user)` which adds `_user_id` to the existing session without regenerating the session ID, then sets `session["sudo_time"]`.

**Email Forwarding Pipeline** (`email_handler.py`)

- `handle_forward()` (line 536): Entry point for forward processing; extracts `Reply-To` header before calling `forward_email_to_mailbox`.
- `forward_email_to_mailbox()` (line 679): Calls `delete_all_headers_except()` with a keep-list that explicitly omits `Reply-To`, `Received`, and all custom `X-` headers; then optionally re-adds a transformed `Reply-To` with the reverse-alias address.

**Alias Token Signing** (`app/alias_suffix.py`)

- `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)` (line 11): Creates the signer used for all alias suffix tokens.
- `check_suffix_signature()` (lines 37–40): Calls `signer.unsign(signed_suffix, max_age=600)` — the `max_age=600` parameter enforces the 10-minute expiry window.

**API Key Usage Tracking** (`app/api/base.py`, lines 28–31)

- On each authenticated API request with a valid key: `api_key.last_used = arrow.now()` and `api_key.times += 1`, followed by `Session.commit()`.

### 0.4.2 Error Handling Integration

- `server.py` (lines 388–393): The generic `@app.errorhandler(Exception)` handler catches the `AttributeError` from the sudo path and returns `{"error": "Internal error"}` with HTTP 500 for any `/api/` path.
- `server.py` (line 347): The `@app.errorhandler(401)` handler renders either JSON or HTML depending on the request path.

### 0.4.3 Database and Schema Touchpoints

- **`api_key` table**: Fields `times` (INTEGER, default 0) and `last_used` (ArrowType/TIMESTAMP, nullable) are updated on every authenticated API call.
- **`users` table**: Fields `email`, `password` (bcrypt hash), `activated`, `alternative_id` (UUID for Flask-Login `get_id()`) are read during login.
- **Redis `session:*` keys**: Pickle-serialized dictionaries stored with TTL; keys include `_permanent`, `_fresh`, `csrf_token`, `_user_id`, `_id`, `sudo_time`.


## 0.5 Technical Implementation

### 0.5.1 File-by-File Execution Plan

Since the implementation rule (`SWE-AtlasQnA-Repo`) mandates that no existing files be modified and only a markdown document be created, the execution plan centers on a single output file:

- **CREATE**: `blitzy/documentation/app_2cd6ee777f8c.md` — Contains all experimental results, organized by investigation question, with evidence from the running system

All experiments are conducted by starting the Flask server with PostgreSQL and Redis, creating a test user and API key via Python scripts, and exercising the system via `curl` and Python introspection. No test scripts are left in the repository after completion.

### 0.5.2 Implementation Approach — Experimental Protocol

The investigation follows a structured experimental protocol for each question:

**Experiment 1: API Authentication with Session Cookie**
- Start Flask server with `MEM_STORE_URI=redis://localhost:6379`
- Create test user and log in via `POST /auth/login` with CSRF token
- Issue `GET /api/user_info` with only the session cookie (no `Authentication` header)
- Record the HTTP status code and full JSON response body

**Experiment 2: Privileged Operation with Session Cookie**
- Using the authenticated session from Experiment 1
- Issue `DELETE /api/user` (protected by `require_api_sudo`) with only the session cookie
- Record the HTTP status code and JSON error payload
- Inspect server logs for the exception traceback

**Experiment 3: Session Storage Inspection**
- Use `redis-cli` and Python `redis` + `pickle` libraries to query `session:*` keys
- Capture raw bytes, identify pickle protocol version, deserialize and enumerate all keys
- Document the Redis key format (`session:<uuid4>`) and cookie structure (`<uuid>.<hmac-signature>`)

**Experiment 4: Session ID Across Login**
- GET the login page, capture the `slapp` cookie UUID portion
- POST login credentials, capture the `slapp` cookie UUID portion after redirect
- Compare before/after values

**Experiment 5: Email Header Forwarding**
- Construct a Python `email.message.Message` with `X-Custom-Header`, `Received`, and `Reply-To`
- Apply the same `delete_all_headers_except()` call with the identical `headers_to_keep` list used in `forward_email_to_mailbox()`
- Report which headers survived and which were removed

**Experiment 6: Alias Token Expiration**
- Sign a test suffix with `itsdangerous.TimestampSigner` using `CUSTOM_ALIAS_SECRET`
- Call `check_suffix_signature()` immediately (expect success)
- Inspect the `max_age=600` parameter to confirm the 10-minute window
- Verify with boundary tests at `max_age=0` and `max_age=1` (expect failure)

**Experiment 7: API Key Usage Stats**
- Query the `ApiKey` row for `times` and `last_used` before API calls
- Make 3 API calls with the key via `GET /api/user_info`
- Query the `ApiKey` row again and report the delta

**Experiment 8: Failed Login Response**
- GET the login page for a fresh CSRF token
- POST with correct email but wrong password
- Capture the HTTP status code, the flash message in the HTML response, and server log entries

### 0.5.3 Observed Results Summary

All experiments were executed against the running system. The key findings are:

| Investigation | Observed Result |
|---------------|----------------|
| API call with session cookie, no API key | HTTP 200, full JSON user info response |
| Privileged (sudo) endpoint with session cookie | HTTP 500, `{"error": "Internal error"}` |
| Session serialization format | Python pickle protocol 4 |
| Authenticated session keys | `_permanent`, `_fresh`, `csrf_token`, `_user_id`, `_id`, `sudo_time` |
| Session key structure | `session:<uuid4>` in Redis |
| Session ID across login | **Unchanged** — same UUID before and after |
| X-Custom-Header through forwarding | **Stripped** |
| Received header through forwarding | **Stripped** |
| Reply-To header through forwarding | **Stripped** (then re-added as reverse-alias if reply_to_contact exists) |
| Alias token expiry window | 600 seconds (10 minutes) |
| API key fields updated | `times` incremented by 1 per call, `last_used` set to current timestamp |
| Failed login HTTP code | HTTP 200 (re-renders login page) |
| Failed login error message | `toastr.error("Email or password incorrect")` via flash |
| Failed login server log | `POST /auth/login ... 200` (no explicit failure log line from Flask) |


## 0.6 Scope Boundaries

### 0.6.1 Exhaustively In Scope

**Source Files Read for Analysis (read-only)**

| File Path | Purpose in Investigation |
|-----------|------------------------|
| `app/api/base.py` | API authentication decorators, `authorize_request()`, `check_sudo_mode_is_active()`, `require_api_sudo` |
| `app/session.py` | Redis-backed session implementation: `SimpleLoginSessionInterface`, `open_session()`, `save_session()` |
| `app/auth/views/login.py` | Login form handler, CSRF validation, credential check, flash messages |
| `app/auth/views/login_utils.py` | `after_login()` redirect logic, `LoginEvent` dispatch |
| `app/email/headers.py` | Header constants: `SL_DIRECTION`, `SL_EMAIL_LOG_ID`, keep-list definition |
| `app/alias_suffix.py` | `get_alias_suffixes()`, `check_suffix_signature()`, `TimestampSigner` with `max_age=600` |
| `app/models.py` (lines 2350–2385) | `ApiKey` model: `times`, `last_used`, `code` field |
| `email_handler.py` (lines 679–900) | `forward_email_to_mailbox()`, `delete_all_headers_except()`, header keep-list |
| `app/email_utils.py` (lines 505–560) | `delete_all_headers_except()` utility function |
| `app/redis_services.py` | Redis connection utilities |
| `app/extensions.py` | Flask-Login `login_manager` initialization, `session_protection = "strong"` |
| `server.py` (lines 155–230, 339–395) | App factory, session config, `user_loader`, error handlers |
| `app/api/views/user.py` | `DELETE /api/user` endpoint with `require_api_sudo` decorator |
| `app/events/auth_event.py` | `LoginEvent` class with `ActionType.failed`, New Relic event dispatch |
| `app/errors.py` | Custom exception classes (`SLException`, `ErrAuthentication`, etc.) |
| `app/config.py` | `SESSION_COOKIE_NAME`, `FLASK_SECRET`, `MEM_STORE_URI`, `CUSTOM_ALIAS_SECRET` |

**Configuration Files Examined**

| File Path | Purpose |
|-----------|---------|
| `pyproject.toml` | Python version, all 50+ dependency declarations with version constraints |
| `example.env` | Default environment variable names and example values |
| `.env` (created) | Runtime config: `MEM_STORE_URI=redis://localhost:6379` |
| `alembic.ini` | Database migration configuration |

**Infrastructure Used**

| Component | Purpose |
|-----------|---------|
| PostgreSQL 16 | User, ApiKey, and session-related model storage |
| Redis 7.0 | Session backend (`session:<uuid>` keys, pickle serialization) |
| Flask dev server (port 7777) | HTTP endpoint for all API and auth experiments |

**Output Artifact**

| File Path | Purpose |
|-----------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | Final markdown document with all experimental findings |

### 0.6.2 Explicitly Out of Scope

- **Source file modifications** — Per the `SWE-AtlasQnA-Repo` rule, no existing repository files are modified
- **New features or bug fixes** — The investigation is observational; no code changes to fix the HTTP 500 sudo crash or session fixation behavior
- **Performance testing or load testing** — Only functional behavior is examined
- **MFA / WebAuthn / FIDO2 flows** — These authentication mechanisms exist in the codebase but are not part of the user's questions
- **OAuth / social login flows** — Google, Facebook, Proton login integrations are present but not investigated
- **SMTP relay / actual email delivery** — Header filtering was tested programmatically using the same utility function; no actual mail transfer occurred
- **PGP encryption / signing** — Present in the codebase but outside the scope of the user's questions
- **Custom domain / premium feature flows** — Not relevant to the specific behaviors under investigation
- **CI/CD pipeline, Docker deployment, or infrastructure configuration** — Not part of the behavioral investigation
- **Frontend JavaScript behavior** — Only the server-side HTTP responses and session storage are examined; client-side rendering is out of scope except for noting flash message content


## 0.7 Rules for Feature Addition

### 0.7.1 User-Specified Rules

- **Read-Only Source Constraint**: "Don't modify any source files in the repository" — all existing files under the repository root must remain unaltered. No patches, no bug fixes, no configuration changes to committed files.
- **Test Script Cleanup**: "You can create test scripts to observe behavior — just clean them up afterward" — any temporary Python scripts, curl invocations, or helper files used during experimentation must be removed before final delivery.
- **Evidence-Based Answers**: "Show me evidence from actually running the system rather than just reading the code" — every answer must include observed HTTP status codes, response bodies, raw data captures, or log output from the live system. Static code analysis alone is insufficient.
- **Output Document Rule** (`SWE-AtlasQnA-Repo`): Create a new markdown document named `app_2cd6ee777f8c.md` that comprehensively answers the questions posed in the prompt. Place it in `blitzy/documentation/` in the destination repo.

### 0.7.2 Behavioral Investigation Standards

- **Actual vs. Documented Behavior**: The user explicitly distinguishes between "what the code says should happen" and "what actually happens when you try it." Every finding must be grounded in observed runtime behavior — HTTP responses captured with `curl -v`, Redis data retrieved with `redis-cli` or Python `redis` library, database queries executed against the live PostgreSQL instance.
- **Specificity Requirement**: The user requests "specific status code and exact error message text," "actual before and after values," "raw bytes from the storage backend," and "actual logged output." Answers must include verbatim captured data, not paraphrased summaries.
- **Boundary Testing**: For the alias token expiration investigation, the user wants to "experimentally verify what that window is by testing whether a token works immediately, then waiting and testing if it still works, until you find the boundary where it stops working." This demands iterative testing at the expiration boundary.

### 0.7.3 Technical Conventions Followed

- **Server Configuration**: Flask dev server running on port 7777 with `FLASK_ENV=development`, `SESSION_COOKIE_NAME=slapp`, `FLASK_SECRET` set to a test value, `MEM_STORE_URI=redis://localhost:6379` for Redis-backed sessions
- **Test User Isolation**: Single test user (`test@test.com`) created via direct SQL to avoid domain validation constraints. API key generated for API-key-authenticated experiments. All experiments use this single user to ensure consistency.
- **Session Format Awareness**: The session cookie uses `UUID.HMAC-signature` format (via `itsdangerous.Signer`), Redis keys are `session:<uuid>`, and session data is serialized with Python pickle protocol 4
- **No Persistent Side Effects**: Experiments that create database records (test user, API key) are part of the investigation infrastructure, not modifications to the application source


## 0.8 References

### 0.8.1 Repository Files Searched and Analyzed

**Core Application Files**

| File Path | Lines Read | Relevance |
|-----------|-----------|-----------|
| `app/api/base.py` | 1–74 (full) | API auth decorators, `authorize_request()`, `check_sudo_mode_is_active()`, `require_api_sudo`, `require_api_auth` |
| `app/session.py` | 1–122 (full) | `SimpleLoginSessionInterface`, `open_session()`, `save_session()`, TTL logic, pickle serialization |
| `app/auth/views/login.py` | 1–end (full) | Login form handler, credential validation, CSRF check, flash error messages, rate limiting |
| `app/auth/views/login_utils.py` | 1–end (full) | `after_login()` redirect, `LoginEvent.send()` dispatch |
| `app/email/headers.py` | 1–end (full) | Header constants (`SL_DIRECTION`, `SL_EMAIL_LOG_ID`), header keep-list definition |
| `app/alias_suffix.py` | 1–end (full) | `get_alias_suffixes()`, `check_suffix_signature()`, `TimestampSigner` with `max_age=600` |
| `app/models.py` | 2350–2385 | `ApiKey` model fields: `code`, `times`, `last_used`, `sudo_mode_at` |
| `email_handler.py` | 679–900 | `forward_email_to_mailbox()`, `delete_all_headers_except()` call, `headers_to_keep` list |
| `app/email_utils.py` | 505–560 | `delete_all_headers_except()` utility implementation |
| `app/redis_services.py` | 1–end (full) | Redis connection helper, used for session storage |
| `app/extensions.py` | 1–end (full) | Flask-Login `login_manager`, `session_protection = "strong"` |
| `server.py` | 155–230, 339–395 | App factory `create_app()`, session config, `user_loader` by `alternative_id`, generic error handler |
| `app/api/views/user.py` | 1–end (full) | `DELETE /api/user` with `require_api_sudo` decorator |
| `app/events/auth_event.py` | 1–end (full) | `LoginEvent` class, `ActionType.failed`, New Relic custom event |
| `app/errors.py` | 1–end (full) | `SLException` base, `ErrAuthentication`, error hierarchy |
| `app/config.py` | selected lines | `SESSION_COOKIE_NAME`, `FLASK_SECRET`, `MEM_STORE_URI`, `CUSTOM_ALIAS_SECRET` derivation |

**Configuration and Dependency Files**

| File Path | Purpose |
|-----------|---------|
| `pyproject.toml` | Python version constraint (^3.10), all dependency declarations |
| `poetry.lock` | Pinned dependency versions, exported to `requirements.txt` |
| `example.env` | Environment variable reference: `DB_URI`, `FLASK_SECRET`, `EMAIL_DOMAIN`, `MEM_STORE_URI` |
| `alembic.ini` | Database migration configuration |

**Directory Structure Explored**

| Path | Contents |
|------|----------|
| `/` (repository root) | `server.py`, `email_handler.py`, `pyproject.toml`, `poetry.lock`, `alembic.ini` |
| `app/` | Core application: `models.py`, `config.py`, `extensions.py`, `session.py`, `errors.py`, `email_utils.py` |
| `app/api/` | API layer: `base.py` (auth decorators), `views/` (endpoint handlers) |
| `app/api/views/` | Endpoint modules: `user.py`, `alias.py`, `mailbox.py`, `export.py`, etc. |
| `app/auth/views/` | Auth views: `login.py`, `login_utils.py`, `register.py`, `mfa.py` |
| `app/email/` | Email processing: `headers.py`, header constants |
| `app/events/` | Event dispatch: `auth_event.py` (LoginEvent) |
| `migrations/versions/` | Alembic migration scripts |

### 0.8.2 Tech Spec Sections Retrieved

| Section | Key Information Used |
|---------|---------------------|
| 6.4 Security Architecture | API authentication pipeline, session management design, CSRF protection, rate limiting, MFA framework |
| 4.4 Authentication Workflows | Login flow sequence, session creation, API key auth, MFA integration, password verification steps |

### 0.8.3 Experimental Evidence Sources

| Experiment | Method | Key Evidence |
|------------|--------|-------------|
| API with session cookie | `curl -v` GET `/api/user_info` with `slapp` cookie | HTTP 200, JSON body with user fields |
| Privileged sudo endpoint | `curl -v` DELETE `/api/user` with `slapp` cookie | HTTP 500, `{"error": "Internal error"}`, `AttributeError` in server log |
| Session storage inspection | `redis-cli KEYS`, Python `redis.get()` + `pickle.loads()` | Pickle protocol 4, 300 bytes, 6 keys in deserialized dict |
| Session ID across login | `curl -c` / `curl -b` capturing cookies pre/post login | Same UUID before and after login |
| Email header filtering | Python `email.message.Message` + `delete_all_headers_except()` | X-Custom-Header, Received, Reply-To all stripped |
| Alias token expiration | `TimestampSigner.unsign()` with varying `max_age` | Valid at 0s, expired at `max_age=0` and `max_age=1`; boundary is 600s |
| API key usage tracking | SQL `SELECT times, last_used` before/after 3 API calls | `times` incremented 1→4, `last_used` updated to current timestamp |
| Failed login | `curl -v` POST `/auth/login` with wrong password | HTTP 200, `toastr.error("Email or password incorrect")` in HTML |

### 0.8.4 Attachments and External Resources

- **No Figma URLs** were provided for this investigation
- **No external attachments** were provided
- **Docker image reference**: `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (container `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`)
- **Source branch name**: `app_2cd6ee777f8c`
- **Output document path**: `blitzy/documentation/app_2cd6ee777f8c.md`


