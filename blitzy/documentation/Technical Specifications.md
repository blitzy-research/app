# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new investigative documentation** that comprehensively answers a set of empirical questions about SimpleLogin's runtime behavior. The user seeks a detailed Q&A-style technical document that examines authentication mechanisms, session management internals, email forwarding header handling, alias creation token expiration, API key usage tracking, and failed login diagnostics — all derived from deep code analysis of the SimpleLogin open-source repository.

**Category:** Create new documentation
**Documentation type:** Technical investigation / Q&A reference document

The documentation requirements, restated with enhanced clarity:

- **API Session-Based Authentication Behavior** — When a user is logged into the SimpleLogin web interface and makes API calls without an `Authentication` header, document the exact HTTP status code and response body structure that the system returns. The answer must trace the code path in `app/api/base.py` through the `authorize_request()` function.
- **Privileged Operation Access with Browser Session** — When attempting to access `require_api_sudo()`-protected endpoints using only a browser session (no API key), document the specific HTTP status code and exact error message text returned. The user has observed that this is not a standard authorization error — the code analysis must explain why.
- **Redis Session Data Structure** — Capture and document the raw serialized session data format stored in Redis, including the serialization protocol (pickle), the keys present in a deserialized authenticated session (e.g., `_user_id`, `csrf_token`, `sudo_time`), and the structure of the Redis key itself (prefix, signed session ID format).
- **Session Identifier Regeneration During Login** — Determine whether the session identifier (cookie value) changes during the login flow. Trace the code path through `open_session()`, `login_user()`, and Flask-Login's `session_protection = "strong"` to establish whether the session ID is preserved or regenerated.
- **Email Header Forwarding Behavior** — For an inbound email containing a custom X-header, a `Received` header, and a `Reply-To` header, document which headers survive forwarding and which are stripped. Trace the `delete_all_headers_except()` call in `handle_forward()` and the explicit `headers_to_keep` allowlist.
- **Alias Creation Token Expiration Window** — Identify the exact time window after which a signed alias suffix token expires. Trace the `itsdangerous.TimestampSigner` usage in `app/alias_suffix.py` and the `max_age` parameter in `check_suffix_signature()`.
- **API Key Usage Statistics Tracking** — Document the exact database fields updated when API calls are made with a valid API key, including `last_used` and `times` on the `ApiKey` model, and describe the values observed after multiple calls.
- **Failed Login Diagnostics** — Document the specific log messages, HTTP response status codes, and response body content for failed login attempts with incorrect credentials, covering both the web form (`app/auth/views/login.py`) and the API endpoint (`app/api/views/auth.py`).

### 0.1.2 Special Instructions and Constraints

**CRITICAL directives from the user:**

- **No source file modifications:** "Don't modify any source files in the repository" — all findings must be derived from code reading, not code changes.
- **Evidence-based analysis:** "Show me evidence from actually running the system rather than just reading the code" — while the environment does not support running the full stack (PostgreSQL, Redis, Postfix), the documentation must provide the deepest possible code-path-based evidence, citing exact file paths, line numbers, and function traces to predict exact runtime behavior.
- **Test script allowance:** "You can create test scripts to observe behavior - just clean them up afterward" — temporary analysis scripts are permitted but must be removed.

**Implementation rule directives:**

- Create a new markdown document named `app_2cd6ee777f8c.md` in the `blitzy/documentation` directory
- Provide thinking / rationale behind all answers
- Do not make assumptions — base answers on the code as the source of truth
- Do not modify any existing files in the source repository

**Style preferences:**

- The document should be structured as a series of investigated questions with detailed answers
- Each answer should include code path traces with file:line references
- Diagrams (Mermaid) should illustrate complex flows where appropriate
- Actual values (status codes, error strings, data formats) must be cited precisely

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document API session-based authentication, we will create a new section in `blitzy/documentation/app_2cd6ee777f8c.md` tracing the `authorize_request()` function in `app/api/base.py` through both the API-key path and the Flask-Login session fallback path, citing exact response tuples.
- To document privileged operation access, we will trace the `require_api_sudo()` decorator and `check_sudo_mode_is_active()` function, identifying the non-standard HTTP 440 status code and the `AttributeError` path when `g.api_key` is `None`.
- To document session data structure, we will analyze `app/session.py`'s `RedisSessionStore`, the pickle serialization in `save_session()`, the Redis key format `session:{uuid}`, and the `itsdangerous.Signer` cookie signing mechanism.
- To document session ID regeneration, we will trace Flask-Login's `session_protection = "strong"` setting in `app/extensions.py` and its interaction with the `open_session()`/`save_session()` lifecycle.
- To document email header behavior, we will analyze the `headers_to_keep` allowlist in `email_handler.py` (lines 793–810) and the `delete_all_headers_except()` function in `app/email_utils.py`.
- To document token expiration, we will analyze `check_suffix_signature()` in `app/alias_suffix.py` with `max_age=600`.
- To document API key stats, we will trace the `authorize_request()` function's update of `api_key.last_used` and `api_key.times` fields.
- To document failed login diagnostics, we will trace both `app/auth/views/login.py` and `app/api/views/auth.py` for their respective error handling paths.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following additional documentation needs are inferred:

- **`app/api/base.py` line 42 sets `g.api_key = api_key` where `api_key` can be `None`** — this creates a downstream `AttributeError` in sudo-protected endpoints when accessed via browser session, which should be documented as a discovered behavioral quirk.
- **`app/session.py` uses `cPickle`/`pickle` for session serialization** — the security implications of pickle deserialization and the exact binary format should be documented.
- **Flask-Login `session_protection = "strong"` in `app/extensions.py` line 8** — this setting causes session regeneration on certain conditions (user-agent/IP changes), which affects the session ID behavior question.
- **The `HEADER_ALLOW_API_COOKIES` constant (`X-Sl-Allowcookies`)** — commented out in production `app/api/base.py` (lines 22–23) but injected by test infrastructure in `tests/conftest.py`, meaning cookie-based API auth works unconditionally in production.
- **Non-standard HTTP 440 status code** — used for sudo mode expiration in `app/api/base.py` line 70, this is a Microsoft IIS-specific code not part of the IANA HTTP status code registry.
- **Session TTL differentiation** — `app/session.py` lines 95–96 set a 300-second TTL for unauthenticated sessions vs. the full `permanent_session_lifetime` (7 days per `server.py` line 207) for authenticated sessions.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **flat documentation structure** with operational focus and partial API coverage. The project has no documentation generator framework (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx/conf.py` detected). All documentation exists as standalone Markdown files.

**Documentation files discovered:**

| File Path | Type | Coverage Status |
|-----------|------|-----------------|
| `README.md` | Self-hosting guide | Comprehensive for deployment; no runtime behavior analysis |
| `CONTRIBUTING.md` | Developer guide | Covers dev setup, testing, code structure; no auth/session internals |
| `SECURITY.md` | Vulnerability disclosure | Minimal; policy only |
| `docs/api.md` | API reference | Comprehensive endpoint listing; no session-auth behavior docs |
| `docs/oauth.md` | OAuth reference | Covers OAuth/OIDC flows; no session management details |
| `docs/troubleshooting.md` | Ops runbook | Email delivery debugging; no auth troubleshooting |
| `docs/ssl.md` | TLS/certificate guide | Infrastructure-focused |
| `docs/ses.md` | SES relay setup | Infrastructure-focused |
| `docs/gmail-relay.md` | Gmail relay setup | Infrastructure-focused |
| `docs/enforce-spf.md` | SPF hardening | Infrastructure-focused |
| `docs/postfix-tls.md` | Postfix TLS | Infrastructure-focused |
| `docs/upgrade.md` | Version upgrade guide | Ops-focused |
| `docs/build-image.md` | Docker image build | Ops-focused |
| `docs/ufw.md` | Firewall setup | Minimal |
| `docs/code-structure.md` | Code structure notes | TODO-style, minimal |

**Documentation tools detected:** None. The project uses raw Markdown without a build pipeline.

**API documentation tools in use:** None detected — `docs/api.md` is hand-written Markdown, not auto-generated from code annotations. No JSDoc, Sphinx, or similar tooling is present.

**Diagram tools detected:** None configured, though the tech spec uses Mermaid extensively. The `CONTRIBUTING.md` references an architecture image at `docs/archi.png`.

**Key finding:** No existing documentation covers the runtime behavioral questions the user is asking. The `docs/api.md` documents endpoint contracts (expected inputs/outputs) but does not document edge-case behaviors such as session-based API access, sudo mode with missing API keys, or header stripping during email forwarding.

### 0.2.2 Repository Code Analysis for Documentation

The following code modules were analyzed to build the source material for the Q&A document:

**Authentication and API Layer:**

| Source File | Lines | Key Content for Documentation |
|-------------|-------|-------------------------------|
| `app/api/base.py` | 74 | `authorize_request()`, `require_api_auth()`, `require_api_sudo()`, `check_sudo_mode_is_active()` |
| `app/api/views/auth.py` | 80+ | API login endpoint, error responses for failed auth |
| `app/api/views/sudo.py` | 27 | Sudo mode activation endpoint |
| `app/auth/views/login.py` | 83 | Web login form, flash messages, rate limiter integration |
| `app/auth/views/login_utils.py` | 69 | `after_login()`, MFA routing, `login_user()` call |
| `app/extensions.py` | 37 | `session_protection = "strong"`, rate limiter setup |

**Session Management:**

| Source File | Lines | Key Content for Documentation |
|-------------|-------|-------------------------------|
| `app/session.py` | 122 | `RedisSessionStore`, `ServerSession`, pickle serialization, HMAC signing |
| `app/redis_services.py` | 25 | Redis/Sentinel initialization, session store wiring |
| `app/config.py` | Lines 199–201 | `SESSION_COOKIE_NAME = "slapp"`, `CUSTOM_ALIAS_SECRET` derivation |
| `server.py` | Lines 195–207 | `permanent_session_lifetime = 7 days`, `make_session_permanent()` |

**Email Forwarding:**

| Source File | Lines | Key Content for Documentation |
|-------------|-------|-------------------------------|
| `email_handler.py` | Lines 536–873 | `handle_forward()`, header stripping, Reply-To rewriting |
| `app/email/headers.py` | 60 | Header constant definitions including `SL_DIRECTION`, `RECEIVED`, `REPLY_TO` |
| `app/email_utils.py` | Lines 505–545 | `delete_all_headers_except()`, `delete_header()`, `add_or_replace_header()` |

**Alias Token and API Key Tracking:**

| Source File | Lines | Key Content for Documentation |
|-------------|-------|-------------------------------|
| `app/alias_suffix.py` | 193 | `TimestampSigner`, `check_suffix_signature(max_age=600)`, suffix generation |
| `app/models.py` | Lines 2350–2370 | `ApiKey` model: `code`, `last_used`, `times`, `sudo_mode_at` |
| `app/api/views/new_custom_alias.py` | 236 | Alias creation with signed suffix validation, HTTP 412 on expiry |

**Error Handling:**

| Source File | Lines | Key Content for Documentation |
|-------------|-------|-------------------------------|
| `server.py` | Lines 339–395 | `setup_error_page()`: handlers for 400, 401, 403, 404, 405, 429, and generic `Exception` |
| `app/constants.py` | 1 | `HEADER_ALLOW_API_COOKIES = "X-Sl-Allowcookies"` |
| `tests/conftest.py` | 78 | Test infrastructure setup, `CustomTestClient` header injection |

### 0.2.3 Web Search Research Conducted

No web search was required for this documentation task. All questions posed by the user are answerable through direct code analysis of the SimpleLogin repository. The codebase serves as the definitive source of truth per the implementation rules ("Do not make assumptions, base your answers on the code as the truth").

The following standard knowledge was applied without web search:

- Python `pickle` serialization format (protocol 2+)
- `itsdangerous` library `TimestampSigner` behavior with `max_age`
- Flask-Login `session_protection = "strong"` behavior (session identifier validation)
- HTTP status code registry (440 is non-standard / Microsoft IIS-specific)


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation coverage to answer the user's eight investigative questions:

**Module: `app/api/base.py` — API Authentication and Authorization Layer**
- Public APIs: `authorize_request()`, `check_sudo_mode_is_active()`, `require_api_auth()`, `require_api_sudo()`
- Current documentation: `docs/api.md` mentions API key usage via `Authentication` header but does not document the session fallback path or sudo mode behavior
- Documentation needed: Detailed code-path trace for session-based API access, sudo mode with/without API key, exact HTTP status codes and error bodies

**Module: `app/session.py` — Redis-Backed Session Store**
- Public APIs: `RedisSessionStore.open_session()`, `save_session()`, `purge_session()`, `extract_and_validate_session_id()`
- Current documentation: No existing documentation
- Documentation needed: Session data format, Redis key structure, pickle serialization details, session ID signing mechanism, TTL policies

**Module: `app/auth/views/login.py` + `login_utils.py` — Login Flow**
- Public APIs: `login()` route handler, `after_login()` utility
- Current documentation: `docs/api.md` covers the API login endpoint; no documentation for web login form behavior
- Documentation needed: Session ID behavior during login, MFA routing, failed login diagnostics

**Module: `email_handler.py` — Email Forwarding Engine**
- Public APIs: `handle_forward()`, header rewriting logic
- Current documentation: `CONTRIBUTING.md` describes testing with `swaks`; no header handling documentation exists
- Documentation needed: Exact header allowlist, which headers survive forwarding, Reply-To rewriting behavior

**Module: `app/alias_suffix.py` — Alias Token Management**
- Public APIs: `check_suffix_signature()`, `get_alias_suffixes()`, `verify_prefix_suffix()`
- Current documentation: No existing documentation
- Documentation needed: Token expiration window (600 seconds), `TimestampSigner` mechanics

**Module: `app/models.py` (ApiKey class) — API Key Statistics**
- Public APIs: `ApiKey.create()`, fields `last_used`, `times`, `sudo_mode_at`
- Current documentation: No existing documentation on usage tracking
- Documentation needed: Which fields are updated per API call, exact update mechanics

**Module: `app/extensions.py` — Flask-Login Configuration**
- Public APIs: `login_manager.session_protection = "strong"`
- Current documentation: No existing documentation
- Documentation needed: Impact on session identifier behavior during authentication

**Module: `server.py` — Flask App Bootstrap and Error Handling**
- Public APIs: `create_app()`, `setup_error_page()`, `make_session_permanent()`
- Current documentation: `CONTRIBUTING.md` references `server.py` for local development
- Documentation needed: Global exception handler behavior (500 response for unhandled errors on API paths)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

**Undocumented behavioral patterns (all gaps — no existing coverage):**
- Session-based API authentication fallback in `app/api/base.py` (the commented-out `HEADER_ALLOW_API_COOKIES` check at lines 22–23 reveals this was once gated but is now open)
- Non-standard HTTP 440 status code for sudo mode expiration
- `g.api_key = None` downstream failure when session auth is used with sudo-protected endpoints
- Redis session data internal format and structure
- Session ID lifecycle during authentication (regeneration behavior)
- Email header stripping during the forward phase (the allowlist pattern)
- `TimestampSigner` `max_age=600` for alias creation tokens
- API key `last_used` and `times` counter update mechanics
- Failed login flash messages vs. API error bodies

**Missing cross-cutting documentation:**
- No documentation links authentication behavior to session management internals
- No documentation traces the complete code path from HTTP request to response for edge cases
- No documentation covers the interaction between Flask-Login's `session_protection = "strong"` and the custom `RedisSessionStore`

**Outdated documentation:**
- `docs/api.md` states "All following endpoint return `401` status code if the API Key is incorrect" — this is incomplete because session-based auth can bypass the API key requirement entirely, and sudo endpoints return 440 (not 401 or 403)

### 0.3.3 Configuration Options Requiring Documentation

| Config Key | Source | Current Doc Status | Documentation Needed |
|------------|--------|-------------------|---------------------|
| `SESSION_COOKIE_NAME` ("slapp") | `app/config.py:199` | Undocumented | Session cookie identification |
| `CUSTOM_ALIAS_SECRET` | `app/config.py:201` | Undocumented | Alias token signing key derivation |
| `MEM_STORE_URI` | `app/config.py` / `example.env` | Partially documented | Redis session backend connection |
| `FLASK_SECRET` | `app/config.py:196` | Partially documented | Session signing master key |
| `DISABLE_RATE_LIMIT` | `app/config.py:602` | Undocumented | Rate limiter bypass flag |
| `permanent_session_lifetime` (7 days) | `server.py:207` | Undocumented | Authenticated session TTL |


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document will be a single comprehensive Markdown file placed in the `blitzy/documentation/` directory per the implementation rules. The structure follows the user's investigative questions, with each section providing a complete answer backed by code-path evidence.

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
        ├── Introduction (investigation scope and methodology)
        ├── Q1: API Session-Based Authentication Response
        │   ├── Code Path Trace
        │   ├── HTTP Status Code and Response Body
        │   └── Rationale
        ├── Q2: Privileged Operations with Browser Session
        │   ├── Code Path Trace (require_api_sudo → check_sudo_mode_is_active)
        │   ├── Non-Standard HTTP 440 Status Code
        │   ├── AttributeError Path for Session-Only Access
        │   └── Rationale
        ├── Q3: Session Data Structure in Redis
        │   ├── Redis Key Format
        │   ├── Serialization Format (pickle)
        │   ├── Deserialized Session Keys
        │   ├── Session Cookie Signing Mechanism
        │   └── Rationale
        ├── Q4: Session Identifier Behavior During Login
        │   ├── Before-Login Session ID
        │   ├── Flask-Login session_protection="strong"
        │   ├── After-Login Session ID
        │   └── Rationale
        ├── Q5: Email Header Forwarding Behavior
        │   ├── Headers Allowlist (headers_to_keep)
        │   ├── Custom X-Header: Stripped
        │   ├── Received Header: Stripped
        │   ├── Reply-To Header: Replaced with Reverse Alias
        │   └── Rationale
        ├── Q6: Alias Creation Token Expiration Window
        │   ├── TimestampSigner Configuration
        │   ├── max_age=600 (10 minutes)
        │   ├── HTTP 412 Response on Expiry
        │   └── Rationale
        ├── Q7: API Key Usage Statistics
        │   ├── ApiKey Model Fields
        │   ├── Update Mechanics (last_used, times)
        │   └── Rationale
        ├── Q8: Failed Login Diagnostics
        │   ├── Web Login (flash messages, rate limiter)
        │   ├── API Login (JSON error, status 400)
        │   ├── Log Output (LoginEvent)
        │   └── Rationale
        └── Summary of Findings
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract API authentication behavior from `app/api/base.py` by tracing `authorize_request()` line-by-line with both the API-key-present and API-key-absent code paths
- Extract session data format from `app/session.py` by analyzing `save_session()` (pickle serialization), `open_session()` (pickle deserialization), and `_get_key()` (Redis key format)
- Extract header forwarding behavior from `email_handler.py` by analyzing the `headers_to_keep` list definition (lines 793–810) and `delete_all_headers_except()` invocation
- Extract token expiration from `app/alias_suffix.py` by analyzing `signer.unsign(signed_suffix, max_age=600)` at line 40
- Extract API key statistics from `app/api/base.py` lines 30–32 (`api_key.last_used`, `api_key.times += 1`)
- Extract failed login responses from both `app/auth/views/login.py` (line 49: flash) and `app/api/views/auth.py` (line 66: jsonify)

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagrams for code flow traces (authentication pipeline, header stripping)
- Code snippets using fenced blocks with syntax highlighting (Python)
- Source citations as inline references: `Source: /path/to/file.py:LineNumber`
- Tables for structured data (status codes, headers, model fields)

### 0.4.3 Diagram and Visual Strategy

The following Mermaid diagrams will be created within the output document:

- **API Authentication Decision Tree** — flowchart showing the `authorize_request()` branching between API key auth, session fallback, and error responses
- **Sudo Mode Access Flow** — flowchart showing the `require_api_sudo()` path with the `None` API key failure case
- **Session Lifecycle During Login** — sequence diagram showing session ID behavior before/during/after `login_user()`
- **Email Header Filtering Pipeline** — flowchart showing inbound headers passing through `delete_all_headers_except()` and which survive
- **Alias Token Signing and Verification** — sequence diagram showing `TimestampSigner.sign()` → cookie → `TimestampSigner.unsign(max_age=600)`


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `app/api/base.py`, `app/session.py`, `app/auth/views/login.py`, `app/auth/views/login_utils.py`, `app/extensions.py`, `email_handler.py`, `app/email/headers.py`, `app/email_utils.py`, `app/alias_suffix.py`, `app/models.py`, `app/api/views/auth.py`, `app/api/views/sudo.py`, `app/api/views/new_custom_alias.py`, `app/config.py`, `app/redis_services.py`, `server.py`, `app/constants.py` | Comprehensive Q&A document answering all eight investigative questions about SimpleLogin runtime behavior, with code-path traces, exact status codes, error messages, data formats, and Mermaid diagrams |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Technical Investigation / Q&A Reference
Source Code: 17 source files across app/, email_handler.py, and server.py
Sections:
    - Introduction (investigation methodology, code-as-truth rationale)
    - Q1: Session-based API Auth (from app/api/base.py:16-43)
    - Q2: Sudo Mode with Browser Session (from app/api/base.py:46-73, server.py:388-395)
    - Q3: Redis Session Data Structure (from app/session.py:21-114)
    - Q4: Session ID During Login (from app/extensions.py:8, app/auth/views/login_utils.py:36)
    - Q5: Email Header Forwarding (from email_handler.py:793-873, app/email/headers.py)
    - Q6: Alias Token Expiration (from app/alias_suffix.py:37-42)
    - Q7: API Key Usage Stats (from app/api/base.py:30-32, app/models.py:2350-2370)
    - Q8: Failed Login Diagnostics (from app/auth/views/login.py:45-50, app/api/views/auth.py:64-66)
    - Summary of Findings
Diagrams:
    - Authentication pipeline flowchart
    - Sudo mode access flow
    - Session lifecycle sequence diagram
    - Email header filtering pipeline
    - Alias token signing/verification flow
Key Citations:
    - app/api/base.py (authentication and authorization core)
    - app/session.py (session management)
    - email_handler.py (email forwarding)
    - app/alias_suffix.py (token management)
    - app/models.py (ApiKey model)
    - server.py (error handling, session configuration)
    - app/extensions.py (Flask-Login configuration)
    - app/email/headers.py (header constants)
    - app/email_utils.py (header manipulation utilities)
    - app/config.py (configuration constants)
    - app/auth/views/login.py (web login)
    - app/auth/views/login_utils.py (post-login routing)
    - app/api/views/auth.py (API login)
    - app/api/views/sudo.py (sudo activation)
    - app/api/views/new_custom_alias.py (alias creation with token)
    - app/redis_services.py (Redis/Sentinel initialization)
    - app/constants.py (HEADER_ALLOW_API_COOKIES)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are required. The project has no documentation generator framework (`mkdocs.yml`, `docusaurus.config.js`, or `sphinx/conf.py`). The new file is placed in the `blitzy/documentation/` directory as specified by the implementation rule.

### 0.5.4 Cross-Documentation Dependencies

- **Shared content:** The Q&A document references the same authentication flow documented in `docs/api.md` but provides deeper behavioral analysis not covered there
- **Navigation links:** No automatic navigation updates needed (no docs framework)
- **No index/glossary updates required:** The project has no centralized documentation index
- **No table of contents updates required:** The `README.md` does not link to the `blitzy/documentation/` directory


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

No additional documentation tools or packages need to be installed for this task. The output is a single Markdown file with embedded Mermaid diagrams, requiring no build pipeline. However, the following project dependencies are directly relevant to the documented behaviors and must be understood for accurate answers:

| Registry | Package Name | Version | Purpose (Relevant to Documentation) |
|----------|--------------|---------|--------------------------------------|
| pip (Poetry) | flask | ^1.1.2 | Web framework; session interface, request context, error handlers |
| pip (Poetry) | flask_login | ^0.5.0 | Session authentication; `session_protection = "strong"` behavior |
| pip (Poetry) | itsdangerous | (transitive via Flask) | Session cookie HMAC signing; alias suffix `TimestampSigner` |
| pip (Poetry) | redis | ^4.5.3 | Session storage backend; session key format |
| pip (Poetry) | arrow | ^0.16.0 | Timestamp handling for `ApiKey.last_used` updates |
| pip (Poetry) | bcrypt | ^3.2.0 | Password verification in failed login diagnostics |
| pip (Poetry) | aiosmtpd | ^1.2 | SMTP inbound handler for email forwarding |
| pip (Poetry) | flask-cors | ^3.0.9 | CORS wildcard on `/api/*` routes |
| pip (Poetry) | Flask-Limiter | ^1.4 | Rate limiting on login and API endpoints |
| pip (Poetry) | SQLAlchemy | 1.3.24 | ORM for ApiKey model and Session.commit() |
| pip (Poetry) | sentry_sdk | ^2.16.0 | Error tracking (Sentry integration in error handler) |
| pip (Poetry) | python | ^3.10 | Runtime; pickle protocol for session serialization |

### 0.6.2 Documentation Reference Updates

No documentation link updates are required. The new file `blitzy/documentation/app_2cd6ee777f8c.md` is a self-contained document that does not reference or depend on existing documentation links. Existing documentation files (`README.md`, `docs/api.md`, etc.) are not modified per the implementation rules.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (behavioral documentation):**
- Investigative questions documented: 0/8 (0%) — no existing documentation covers any of the user's questions
- Authentication edge cases documented: 0/3 (session fallback, sudo+session, failed login) — `docs/api.md` covers only happy-path API auth
- Session internals documented: 0/4 (data format, key structure, TTL, regeneration) — entirely undocumented
- Email header handling documented: 0/1 — no header stripping documentation exists
- Token expiration documented: 0/1 — alias suffix token behavior undocumented
- API key stats documented: 0/1 — usage tracking fields undocumented

**Target coverage after this task:**
- Investigative questions documented: 8/8 (100%)
- All eight behavioral questions fully answered with code-path evidence

**Coverage gaps addressed by this document:**

| Topic | Current Coverage | Target Coverage | Focus Areas |
|-------|-----------------|-----------------|-------------|
| API session-based auth | 0% | 100% | `authorize_request()` session fallback path |
| Sudo mode edge cases | 0% | 100% | HTTP 440, `NoneType` AttributeError on `g.api_key` |
| Session data format | 0% | 100% | Redis key, pickle, `_user_id`, `csrf_token`, TTL |
| Session ID regeneration | 0% | 100% | `session_protection="strong"`, `login_user()` |
| Header forwarding | 0% | 100% | `headers_to_keep` allowlist, X-header/Received/Reply-To |
| Token expiration | 0% | 100% | `max_age=600`, HTTP 412 on expiry |
| API key stats | 0% | 100% | `last_used`, `times`, `Session.commit()` |
| Failed login diagnostics | 0% | 100% | Web flash messages, API JSON errors, LoginEvent |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every answer must cite exact file paths and line numbers from the source code
- Every HTTP status code and error message must be quoted verbatim from the code
- Every data structure (session keys, Redis format) must be described precisely
- Code path traces must cover all branching conditions, not just the happy path

**Accuracy validation:**
- All code citations verified by direct `read_file` retrieval of source files
- All function signatures confirmed against actual source code
- All status codes cross-referenced between endpoint handlers and `setup_error_page()` in `server.py`
- No assumptions made — every claim traceable to a specific code location

**Clarity standards:**
- Each question answered with a clear, direct statement before the detailed trace
- Mermaid diagrams provided for complex multi-step flows
- Tables used for structured comparisons (headers kept vs. stripped, status codes)
- Technical language balanced with explanatory context

**Maintainability:**
- Source citations included for every finding (file:line format)
- Each answer self-contained to enable independent future updates
- Clear separation between "what the code says" and "inferred runtime behavior"

### 0.7.3 Example and Diagram Requirements

- Minimum diagrams: 5 (authentication pipeline, sudo access flow, session lifecycle, header filtering, token signing)
- Code snippet examples: At least one per question showing the relevant source code extract (2–3 lines each)
- Tables: Status code summary table, header survival table, session key table, API key field update table
- Expected response body examples: Formatted JSON showing exact `{"error": "..."}` structures


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the sole deliverable document

**Source code modules analyzed for documentation content:**
- `app/api/base.py` — API authentication pipeline, session fallback, sudo mode
- `app/api/views/auth.py` — API login endpoint, failed login response
- `app/api/views/sudo.py` — Sudo mode activation
- `app/api/views/new_custom_alias.py` — Alias creation with token validation
- `app/auth/views/login.py` — Web login form, flash messages
- `app/auth/views/login_utils.py` — Post-login routing, `login_user()`, MFA
- `app/session.py` — Redis session store, pickle serialization, HMAC signing
- `app/redis_services.py` — Redis/Sentinel initialization
- `app/extensions.py` — Flask-Login configuration, rate limiter
- `app/config.py` — Configuration constants (secrets, session cookie name)
- `app/constants.py` — `HEADER_ALLOW_API_COOKIES`
- `app/models.py` — `ApiKey` model (fields: `code`, `last_used`, `times`, `sudo_mode_at`)
- `app/alias_suffix.py` — `TimestampSigner`, `check_suffix_signature()`, suffix generation
- `app/email/headers.py` — Header constant definitions
- `app/email_utils.py` — `delete_all_headers_except()`, `delete_header()`, `add_or_replace_header()`
- `email_handler.py` — `handle_forward()`, header allowlist, Reply-To handling
- `server.py` — Flask app bootstrap, error handlers, session lifetime configuration

**Investigation topics covered:**
- HTTP response codes and error bodies for session-based API auth
- Non-standard HTTP 440 for sudo mode, and 500 for session-only sudo access
- Redis session data format (pickle, key structure, session keys, TTL)
- Session identifier behavior during authentication flow
- Email header survival through forwarding (X-headers, Received, Reply-To)
- Alias creation token 600-second expiration window
- API key `last_used` and `times` counter updates
- Failed login flash messages, API error JSON, and `LoginEvent` emission

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — No files in the repository will be modified (per user instruction and implementation rules)
- **Test file modifications** — No test files will be altered
- **Existing documentation updates** — `README.md`, `docs/api.md`, and other existing docs are not modified
- **Feature additions or code refactoring** — This is a documentation-only task
- **Deployment configuration changes** — No Docker, Postfix, or Nginx changes
- **Running the full system** — The environment lacks PostgreSQL, Redis, and Postfix; all answers are derived from code analysis
- **Email delivery testing** — Sending actual test emails via SMTP is not feasible without the full infrastructure stack; header behavior is documented from code analysis
- **Database inspection** — Direct database queries are not possible; API key stats are documented from the ORM update logic in `app/api/base.py`
- **Performance or load testing** — Not relevant to the user's behavioral questions
- **PGP encryption behavior** — Not asked by the user
- **OAuth/OIDC flow behavior** — Not asked by the user
- **Custom domain DNS verification** — Not asked by the user
- **Cron job behavior** — Not asked by the user
- **Payment/subscription behavior** — Not asked by the user


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** N/A — the output is a standalone Markdown file with no build pipeline
- **Documentation preview command:** Any Markdown renderer (e.g., VS Code Markdown preview, GitHub rendering)
- **Diagram generation command:** Mermaid diagrams are embedded inline within fenced code blocks; no external generation step required. Renderers such as GitHub, GitLab, and Mermaid Live Editor natively render `mermaid` blocks
- **Documentation deployment command:** N/A — the file is committed to the repository
- **Default format:** Markdown with embedded Mermaid diagrams
- **Citation requirement:** Every finding must reference the source file and line number using the format `Source: path/to/file.py:LineNumber`
- **Style guide:** Follow the investigative Q&A format with clear question headings, direct answers, code-path traces, and rationale sections
- **Documentation validation:** Manual review of all code citations against source files; verify that every quoted status code and error string matches the actual code

### 0.9.2 Output File Specification

| Attribute | Value |
|-----------|-------|
| File path | `blitzy/documentation/app_2cd6ee777f8c.md` |
| Format | Markdown |
| Naming convention | `<source_branch_name>.md` per implementation rule |
| Branch name | `app_2cd6ee777f8c` |
| Diagrams | Mermaid (embedded in fenced blocks) |
| Code examples | Python (fenced with `python` language tag) |
| Estimated sections | 10 (introduction + 8 questions + summary) |
| Source citations | Inline format: `Source: file.py:line` |


## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository** — all output must be contained in the new `blitzy/documentation/app_2cd6ee777f8c.md` file
- **Base all answers on the code as the source of truth** — do not make assumptions or speculate about behavior not evidenced in the source code
- **Provide thinking and rationale behind every answer** — each question must include a "Rationale" section explaining the reasoning chain from code to conclusion
- **Show evidence from the code, not just surface-level descriptions** — answers must include exact file paths, line numbers, function names, variable values, and quoted code strings
- **Create the output document in the `blitzy/documentation` directory** — this directory must be created if it does not exist
- **Name the output file `app_2cd6ee777f8c.md`** — matching the `<source_branch_name>` convention
- **Cite exact HTTP status codes and error message strings** — all status codes must be the literal integer values from the source code; all error messages must be quoted verbatim
- **Include Mermaid diagrams for complex flows** — authentication pipeline, sudo access, session lifecycle, header filtering, and token verification flows
- **Distinguish between code-predicted behavior and confirmed runtime behavior** — since the full system cannot be run in this environment, clearly label findings as "code-path analysis" rather than "observed runtime behavior"
- **Temporary test scripts are permitted but must be cleaned up** — if any temporary scripts are created for analysis, they must be removed before task completion
- **Do not document unrelated features** — focus exclusively on the eight investigative questions posed by the user


## 0.11 References

### 0.11.1 Source Files and Folders Searched

**Authentication and Authorization Layer:**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `app/api/base.py` | Core API authentication pipeline: `authorize_request()`, `require_api_auth()`, `require_api_sudo()`, `check_sudo_mode_is_active()`. Identified session fallback (line 21), `g.api_key = None` (line 42), HTTP 401/403 responses, and non-standard HTTP 440 for sudo mode (line 70) |
| `app/api/views/auth.py` | API login endpoint: identified `jsonify(error="Email or password incorrect"), 400` response for failed auth (line 66), disabled/deleted/unactivated account error bodies |
| `app/api/views/sudo.py` | Sudo activation endpoint: confirmed password re-verification via `user.check_password()`, `jsonify(error="Invalid password"), 403` response |
| `app/api/views/new_custom_alias.py` | Alias creation: confirmed `check_suffix_signature()` call (line 70), HTTP 412 response on token expiry (line 73), "Alias creation time is expired, please retry" error message |
| `app/auth/views/login.py` | Web login: identified `flash("Email or password incorrect", "error")` (line 49), `LoginEvent(LoginEvent.ActionType.failed).send()` (line 50), rate limiter deduction via `g.deduct_limit = True` (line 47) |
| `app/auth/views/login_utils.py` | Post-login routing: confirmed `login_user(user)` (line 36), `session["sudo_time"] = int(time())` (line 37), MFA routing via `session[MFA_USER_ID]` |
| `app/extensions.py` | Flask-Login config: identified `login_manager.session_protection = "strong"` (line 8) |
| `app/constants.py` | Header constants: identified `HEADER_ALLOW_API_COOKIES = "X-Sl-Allowcookies"` (line 1) |

**Session Management:**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `app/session.py` | Complete session implementation: `RedisSessionStore` class, `ServerSession` class, `SESSION_PREFIX = "session"` (line 18), `_get_key()` format `session:{sessionId}` (line 44-45), HMAC signing via `itsdangerous.Signer` with salt="session" (lines 38-41), pickle serialization (line 91), `_user_id` TTL check (line 95-96), 300s unauthenticated TTL, `uuid.uuid4()` session ID generation, `purge_session()` (lines 61-66), `logout_session()` (lines 117-121) |
| `app/redis_services.py` | Redis initialization: confirmed `RedisSessionStore` wiring with Redis or Sentinel backends (lines 12, 17-18) |
| `app/config.py` | Configuration: identified `SESSION_COOKIE_NAME = "slapp"` (line 199), `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` (line 201), `FLASK_SECRET` (line 196) |
| `server.py` | App bootstrap: identified `app.config["SESSION_COOKIE_NAME"] = SESSION_COOKIE_NAME` (line 167), `permanent_session_lifetime = timedelta(days=7)` (line 207), `SESSION_COOKIE_SAMESITE = "Lax"` (line 169), `SESSION_COOKIE_SECURE` when HTTPS (line 168), error handlers (lines 339-395) including `@app.errorhandler(Exception)` returning `jsonify(error="Internal error"), 500` for API paths |

**Email Forwarding:**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `email_handler.py` | Email forwarding: `handle_forward()` (line 536), `headers_to_keep` list (lines 793-810) including FROM, TO, CC, SUBJECT, DATE, MESSAGE_ID, REFERENCES, IN_REPLY_TO, SL_QUEUE_ID, LIST_UNSUBSCRIBE, LIST_UNSUBSCRIBE_POST + MIME_HEADERS, `delete_all_headers_except(msg, headers_to_keep)` (line 810), Reply-To handling (lines 586-594, 869-873), `SL_DIRECTION = "Forward"` header addition (line 843) |
| `app/email/headers.py` | Header definitions: `RECEIVED = "Received"` (line 14), `REPLY_TO = "Reply-To"` (line 13), `SL_DIRECTION = "X-SimpleLogin-Type"` (line 34), MIME_HEADERS list (lines 44-51) |
| `app/email_utils.py` | Header utilities: `delete_all_headers_except()` (lines 536-541) removes all headers not in the lowercased allowlist, `add_or_replace_header()` (lines 505-511), `delete_header()` (lines 513-519) |

**Alias Token and API Key:**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `app/alias_suffix.py` | Token management: `signer = itsdangerous.TimestampSigner(config.CUSTOM_ALIAS_SECRET)` (line 11), `check_suffix_signature()` with `max_age=600` (line 40), comment "hypothesis: user will click on the button in the 600 secs" (line 38) |
| `app/models.py` | ApiKey model (lines 2350-2370): `code = sa.Column(sa.String(128), unique=True)`, `last_used = sa.Column(ArrowType, default=None)`, `times = sa.Column(sa.Integer, default=0)`, `sudo_mode_at = sa.Column(ArrowType, default=None)` |

**Test Infrastructure (read-only analysis):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `tests/conftest.py` | Test setup: identified `CustomTestClient` injecting `HEADER_ALLOW_API_COOKIES` header (lines 50-56), CSRF disabled (line 25), rate limit disabled (line 65), transactional rollback (lines 74-77) |
| `tests/test.env` | Test configuration: `FLASK_SECRET=secret`, `DB_URI`, `MEM_STORE_URI=redis://localhost` (line 78) |

**Existing Documentation (analyzed for gap identification):**

| File Path | Purpose in Analysis |
|-----------|---------------------|
| `README.md` | Self-hosting guide: confirmed no runtime behavior documentation exists |
| `CONTRIBUTING.md` | Developer guide: confirmed Python 3.10 requirement, no auth/session internals |
| `SECURITY.md` | Vulnerability policy only |
| `docs/api.md` | API reference: confirmed "All following endpoint return 401 status code if the API Key is incorrect" (incomplete; does not cover session fallback or 440) |
| `docs/oauth.md` | OAuth reference: not relevant to user's questions |
| `docs/troubleshooting.md` | Email diagnostics only |
| `Dockerfile` | Build spec: confirmed Python 3.10 base image, poetry dependencies |
| `pyproject.toml` | Dependency manifest: confirmed all package versions (Flask ^1.1.2, flask_login ^0.5.0, redis ^4.5.3, itsdangerous transitive, etc.) |

**Folders Explored:**

| Folder Path | Depth | Purpose |
|-------------|-------|---------|
| (root) | 0 | Repository structure overview |
| `app/` | 1 | Main application package: identified 16 subpackages and 40+ modules |
| `app/api/` | 2 | API layer: `base.py`, `serializer.py`, `views/` |
| `app/api/views/` | 3 | API endpoints: `auth.py`, `sudo.py`, `new_custom_alias.py` and 14 others |
| `app/auth/` | 2 | Auth package: `base.py`, `views/` |
| `app/auth/views/` | 3 | Auth views: `login.py`, `login_utils.py` and 17 others |
| `app/handler/` | 2 | Email handlers: `dmarc.py`, `unsubscribe_*.py` |
| `app/email/` | 2 | Email constants: `headers.py`, `status.py` |
| `docs/` | 1 | Documentation: 13 files (Markdown + SVG) |
| `tests/` | 1 | Test suite: `conftest.py`, `test.env`, 30+ test files and 14 subpackages |

### 0.11.2 Attachments

No attachments were provided for this project. No Figma URLs or design files are applicable.

### 0.11.3 Tech Spec Sections Referenced

| Section | Content Used |
|---------|-------------|
| 1.1 Executive Summary | Project overview, stakeholder identification, system description |
| 1.3 Scope | In-scope features, implementation boundaries, technical requirements |
| 4.4 Authentication Workflows | Login flow diagrams, MFA challenge resolution, session state markers |
| 6.4 Security Architecture | Session management details, API auth/authz, token architecture, HMAC signing, rate limiting |


