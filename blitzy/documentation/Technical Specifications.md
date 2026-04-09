# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create a new investigative analysis document** answering a series of focused, runtime-oriented questions about how the SimpleLogin application handles server-side session deserialization, particularly in normal operation and under adversarial corruption of Redis-stored session data.

- **Category:** Create new documentation
- **Documentation Type:** Technical Q&A / Security Analysis Document
- **Output File:** `blitzy/documentation/app_2cd6ee777f8c.md` — as specified by the implementation rule "SWE-AtlasQnA-Repo"

The user's questions distill into five interrelated investigation areas:

- **Normal session lifecycle behavior:** What does a typical session look like at runtime as users log in (with or without MFA) and log out? What data is stored, how is it serialized, and what does the cookie contain?
- **Corruption and malformation handling:** If the pickled session bytes in Redis are replaced with corrupted, truncated, or otherwise malformed data, what happens on the next HTTP request? Does the application crash, silently reset, return an error, or log the event?
- **Observable runtime evidence:** What appears in HTTP responses, Flask logs, and Redis state when corruption occurs? Is the failure visible or completely silent?
- **Deserialization risk boundary:** Where is the line between a benign session reset (corrupted data that simply fails `pickle.loads()`) and a genuine deserialization attack (crafted pickle payload that triggers `__reduce__`-based code execution)?
- **Threat model nuance:** If an attacker can tamper with the raw session bytes stored in Redis but cannot forge the HMAC-signed session ID cookie, does that meaningfully change the risk profile, and what runtime evidence supports that conclusion?

### 0.1.2 Special Instructions and Constraints

- **No repository modifications:** The source repository must remain unchanged. Temporary observation scripts may be used during investigation but must be cleaned up afterward.
- **Code-as-truth:** All answers must be grounded in the actual codebase, not assumptions or theoretical best practices. The implementation rule explicitly states: "Do not make assumptions, base your answers on the code as the truth."
- **Rationale required:** The document must include thinking and rationale behind every answer.
- **Existing files untouched:** The implementation rule states: "Do not modify any existing files in the source repository."
- **Output location:** The generated document must be placed in `blitzy/documentation/` in the destination repository.
- **Document name:** `app_2cd6ee777f8c.md` (derived from the source branch name).

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To **document normal session lifecycle**, we will trace the code path through `app/session.py` (`RedisSessionStore.open_session` and `save_session`), `app/auth/views/login.py`, `app/auth/views/login_utils.py` (`after_login`), `app/auth/views/mfa.py`, `app/auth/views/logout.py`, and `app/session.py` (`logout_session`), documenting what session keys are written, how they are serialized via `pickle.dumps()`, and how the signed cookie is constructed via `itsdangerous.Signer`.
- To **document corruption handling**, we will analyze the `open_session()` method's `try/except Exception: pass` block (lines 74–79 of `app/session.py`) that catches all deserialization failures and silently returns a new empty `ServerSession`.
- To **document observable runtime evidence**, we will examine `app/log.py` and the `server.py` `after_request` handler to determine what gets logged (and what does not) during a corrupted-session recovery.
- To **document the deserialization risk boundary**, we will analyze how `pickle.loads()` processes both malformed bytes (which raise exceptions caught by the bare `except`) and crafted `__reduce__` payloads (which execute before any exception can be raised).
- To **document the threat model nuance**, we will trace the cookie→Redis→pickle pipeline to show that HMAC-signed cookies protect against client-side session ID forgery, while the unsigned Redis-stored pickle data is vulnerable if an attacker gains direct Redis write access.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **Session data schema documentation:** The session stores multiple keys (`_user_id`, `mfa_user_id`, `sudo_time`, `oauth_state`, `fido_challenge`, `csrf_token`, `slref`) that are not documented in any existing file. The Q&A document should enumerate these.
- **Fallback behavior when `MEM_STORE_URI` is unset:** When Redis is not configured, the application falls back to Flask's default cookie-based session (no `RedisSessionStore` is installed). This materially changes the deserialization threat model and should be noted.
- **Flask-Login `session_protection = "strong"` interaction:** The session protection mode in `app/extensions.py` (line 8) adds an IP/user-agent fingerprint check that interacts with session resets — this needs documentation for completeness.
- **TTL differentiation:** Authenticated sessions (7 days) vs. unauthenticated sessions (300 seconds) have different exposure windows and should be called out.
- **Silent failure pattern:** The deliberate choice to swallow all deserialization errors without logging is itself a significant design decision that warrants explicit documentation and security commentary.

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a documentation structure concentrated in the `docs/` directory with 12 Markdown files and 1 SVG asset, plus three top-level files (`README.md`, `CONTRIBUTING.md`, `SECURITY.md`). The documentation is operational/deployment-focused — there is **no existing document covering session internals, deserialization behavior, or the Redis session store architecture**.

- **Documentation framework:** None detected (no `mkdocs.yml`, `docusaurus.config.js`, `sphinx.conf.py`, or `.readthedocs.yml` present). Documentation is plain Markdown served directly from the repository.
- **Documentation generator:** Not applicable. Files are hand-authored Markdown.
- **API documentation:** `docs/api.md` provides a comprehensive REST API reference covering authentication, aliases, mailboxes, custom domains, contacts, notifications, and settings.
- **Diagram tools:** No diagram generation tools are configured. The existing docs use inline code blocks and text-based instructions.
- **Documentation hosting:** GitHub repository (docs served via `README.md` and `docs/` directory).

**Existing documentation inventory:**

| File | Category | Session-Relevant |
|------|----------|-----------------|
| `README.md` | Self-hosting guide, setup instructions | No |
| `CONTRIBUTING.md` | Contributor workflow | No |
| `SECURITY.md` | Vulnerability disclosure policy | Tangentially (security posture) |
| `docs/api.md` | REST API reference | No |
| `docs/oauth.md` | OAuth2/OIDC flow documentation | No |
| `docs/code-structure.md` | TODO-style note about `local_data/` | No |
| `docs/ssl.md` | TLS/HTTPS/HSTS/MTA-STS setup | No |
| `docs/troubleshooting.md` | Diagnostic steps for email issues | No |
| `docs/upgrade.md` | Version upgrade runbook | No |
| `docs/build-image.md` | Docker image build instructions | No |
| `docs/enforce-spf.md` | SPF enforcement setup | No |
| `docs/ses.md` | Amazon SES relay configuration | No |
| `docs/gmail-relay.md` | Gmail SMTP relay configuration | No |
| `docs/postfix-tls.md` | Postfix TLS configuration | No |
| `docs/ufw.md` | Firewall port configuration | No |

**Key finding:** No document in the repository addresses session management internals, Redis session store behavior, pickle deserialization, or the security boundary between signed cookies and unsigned server-side session data.

### 0.2.2 Repository Code Analysis for Documentation

The following source files were examined to gather evidence for the documentation:

**Primary session infrastructure:**

| File | Purpose | Key Evidence |
|------|---------|-------------|
| `app/session.py` (122 lines) | Custom Redis session interface | `pickle.loads()` / `pickle.dumps()`, HMAC signer, bare `except Exception: pass`, TTL policy |
| `app/redis_services.py` (25 lines) | Redis initialization and session store wiring | `RedisSessionStore` instantiation, Sentinel support |
| `app/extensions.py` (37 lines) | Flask-Login and rate limiter setup | `session_protection = "strong"` |
| `app/config.py` (~580 lines) | Configuration constants | `FLASK_SECRET`, `SESSION_COOKIE_NAME = "slapp"`, `MEM_STORE_URI`, `MFA_USER_ID` |
| `server.py` (600 lines) | Flask app factory and session cookie config | `SESSION_COOKIE_SAMESITE = "Lax"`, `SESSION_COOKIE_SECURE`, 7-day permanent session lifetime |

**Authentication and session consumers:**

| File | Purpose | Session Keys Used |
|------|---------|------------------|
| `app/auth/views/login.py` | Login form and credential verification | None directly (delegates to `after_login`) |
| `app/auth/views/login_utils.py` | Post-credential MFA routing | `session[MFA_USER_ID]`, `session["sudo_time"]`, `session["slref"]` |
| `app/auth/views/mfa.py` | TOTP challenge handler | `session.get(MFA_USER_ID)`, `del session[MFA_USER_ID]` |
| `app/auth/views/fido.py` | WebAuthn challenge handler | `session[MFA_USER_ID]`, `session["fido_challenge"]`, `session["sudo_time"]` |
| `app/auth/views/logout.py` | Logout endpoint | Calls `logout_session()`, deletes cookie |
| `app/dashboard/views/enter_sudo.py` | Sudo mode re-authentication | `session["sudo_time"]` read/write, 120-second gap |

**Supporting infrastructure:**

| File | Purpose | Relevance |
|------|---------|-----------|
| `app/log.py` (79 lines) | Logging configuration | LOG format, no session-error-specific handlers |
| `pyproject.toml` | Dependency manifest | `flask = "^1.1.2"`, `redis = "^4.5.3"`, `python = "^3.10"` |
| `example.env` | Environment variable documentation | `FLASK_SECRET`, `MEM_STORE_URI` not listed (but present in `tests/test.env`) |
| `tests/conftest.py` | Test fixture setup | `WTF_CSRF_ENABLED = False`, no Redis in test fixtures |
| `tests/test.env` | Test environment config | `MEM_STORE_URI=redis://localhost` |

### 0.2.3 Web Search Research Conducted

- **Python pickle deserialization security risks:** Confirmed that `pickle.loads()` allows arbitrary code execution via `__reduce__` method exploitation. Python's official documentation warns against unpickling untrusted data. This is classified as CWE-502 (Deserialization of Untrusted Data) and features in OWASP Top 10.
- **Flask-Login `session_protection = "strong"` behavior:** When set to `"strong"` on permanent sessions, Flask-Login marks the session as non-fresh (`_fresh = False`) rather than deleting it when the identifier (IP + user-agent hash) changes. For non-permanent sessions, the entire session is deleted.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The documentation effort centers on a single new Markdown document that must synthesize information from multiple code modules. The mapping below identifies every source module contributing evidence to each question area.

**Module: `app/session.py` — RedisSessionStore**
- Public APIs: `ServerSession.__init__()`, `RedisSessionStore.__init__()`, `RedisSessionStore.open_session()`, `RedisSessionStore.save_session()`, `RedisSessionStore.purge_session()`, `RedisSessionStore.extract_and_validate_session_id()`, `RedisSessionStore._get_signer()`, `RedisSessionStore._get_key()`, `logout_session()`
- Current documentation: None
- Documentation needed: Complete behavioral description covering serialization, deserialization, error handling, HMAC signing, TTL differentiation, and the cookie lifecycle

**Module: `app/redis_services.py` — Redis initialization**
- Public APIs: `initialize_redis_services()`
- Current documentation: None
- Documentation needed: How Redis session store is wired into the Flask app, Sentinel vs. standard Redis, read/write split

**Module: `app/extensions.py` — Flask-Login configuration**
- Public APIs: `login_manager` (configured with `session_protection = "strong"`)
- Current documentation: None
- Documentation needed: How Flask-Login's strong session protection interacts with Redis-backed sessions

**Module: `server.py` — Flask app factory**
- Public APIs: `create_app()`, `make_session_permanent()` before-request hook, `load_user()` user loader
- Current documentation: Partial (README covers deployment, not internals)
- Documentation needed: Session cookie configuration (name, SameSite, Secure, httponly), permanent session lifetime, the user loader callback

**Module: `app/auth/views/login_utils.py` — Post-login routing**
- Public APIs: `after_login()`
- Current documentation: None
- Documentation needed: MFA routing logic, session keys written during login

**Module: `app/auth/views/logout.py` — Logout endpoint**
- Public APIs: `logout()` route
- Current documentation: None
- Documentation needed: Session purge mechanics, cookie deletion

**Module: `app/config.py` — Configuration**
- Relevant constants: `FLASK_SECRET`, `SESSION_COOKIE_NAME`, `MEM_STORE_URI`, `MFA_USER_ID`
- Current documentation: Partial (`example.env` documents some, but not `MEM_STORE_URI`)
- Documentation needed: Security-relevant session configuration parameters

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented session architecture:** No existing document describes the `RedisSessionStore` custom session interface, the cookie-to-Redis-to-pickle pipeline, or the HMAC signing scheme.
- **Undocumented error handling behavior:** The bare `except Exception: pass` in `open_session()` is a critical design choice with security implications, and it is not documented anywhere.
- **Undocumented session data schema:** The set of session keys (`_user_id`, `mfa_user_id`, `sudo_time`, `oauth_state`, `fido_challenge`, `slref`, `_fresh`, CSRF token) is not cataloged in any documentation.
- **Undocumented security boundary analysis:** The distinction between HMAC-protected session IDs (in cookies) and unprotected session data (in Redis) is not documented, nor is the risk surface this creates.
- **Undocumented deserialization risk:** The use of `pickle.loads()` for session data and its implications for Redis-access-dependent RCE is not discussed in `SECURITY.md` or elsewhere.
- **Undocumented TTL differentiation:** The 7-day vs. 300-second TTL policy based on `_user_id` presence is only visible in the source code.
- **Undocumented fallback behavior:** What happens when `MEM_STORE_URI` is not set (no Redis, Flask uses default cookie-based sessions) is not documented.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output is a single comprehensive Markdown document placed at:

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
```

The document itself should be structured to directly address each question posed by the user, with code-grounded evidence and rationale for every conclusion.

**Recommended internal structure of `app_2cd6ee777f8c.md`:**

```
# Session Deserialization Behavior in SimpleLogin

#### Session Architecture Overview

  - Cookie-to-Redis-to-Pickle pipeline
  - HMAC-signed session IDs (itsdangerous.Signer)
  - RedisSessionStore class anatomy

#### Normal Session Lifecycle

  - Login (credential verification → login_user → session save)
  - MFA flow (MFA_USER_ID → challenge → login_user)
  - Authenticated request processing (cookie → Redis → pickle → session dict)
  - Logout (logout_session → Redis delete → cookie clear)
  - Session data keys and their purposes

#### Session Data Schema

  - Enumeration of all session keys with types and sources
  - TTL differentiation (7 days vs. 300 seconds)
  - What a pickled session looks like in Redis

#### Corruption and Malformation Handling

  - What open_session() does with corrupted pickle data
  - The bare except Exception: pass pattern
  - Observable behavior: no log, no error response, silent new session
  - CSRF token implications (token lost → form resubmission may fail)

#### Log and Response Observability

  - What the after_request handler logs
  - What Sentry captures (or does not)
  - What Flask-Login's session_protection does on fingerprint mismatch

#### Deserialization Risk Boundary Analysis

  - Benign failure: malformed bytes → exception → silent reset
  - Dangerous payload: crafted __reduce__ → code execution BEFORE exception
  - The critical distinction: pickle executes during loads(), not after
  - Why the except block provides no protection against RCE payloads

#### Threat Model: Signed Cookie vs. Unsigned Redis Data

  - What the HMAC signature protects (session ID integrity)
  - What the HMAC signature does NOT protect (Redis-stored bytes)
  - Attack surface with Redis access but without FLASK_SECRET
  - Risk assessment and runtime evidence

#### Conclusions and Risk Summary

```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**
- Extract session lifecycle behavior from `app/session.py` (lines 68–114) using direct code tracing of `open_session()` and `save_session()`
- Extract authentication flow from `app/auth/views/login_utils.py` `after_login()` and `app/auth/views/mfa.py`
- Extract error handling semantics from the bare `except Exception: pass` at `app/session.py` line 78–79
- Extract cookie configuration from `server.py` lines 151–165
- Extract logging behavior from `app/log.py` and `server.py` `after_request` handler (lines 272–296)

**Documentation Standards:**
- Markdown formatting with `#`, `##`, `###` heading hierarchy
- Mermaid diagrams for session lifecycle flows
- Inline code references using backtick notation: `Source: app/session.py:76`
- Tables for session key enumeration and threat model comparison
- All claims backed by specific file paths and line numbers

### 0.4.3 Diagram and Visual Strategy

The document should include the following Mermaid diagrams:

- **Session request lifecycle flowchart:** Cookie extraction → HMAC verification → Redis lookup → pickle deserialization → session available (or new session on failure)
- **Login/logout session state diagram:** Pre-auth → MFA-pending → Authenticated → Logged-out, showing which session keys are written/deleted at each transition
- **Threat model diagram:** Showing the cookie ↔ Redis ↔ pickle boundary and where HMAC protection applies vs. where it does not
- **Corruption handling decision tree:** Showing the three failure modes (bad signature, missing Redis key, bad pickle data) and their identical outcome (new empty session)

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `app/session.py`, `app/redis_services.py`, `app/extensions.py`, `app/config.py`, `server.py`, `app/auth/views/login.py`, `app/auth/views/login_utils.py`, `app/auth/views/mfa.py`, `app/auth/views/fido.py`, `app/auth/views/logout.py`, `app/dashboard/views/enter_sudo.py`, `app/log.py`, `app/api/views/user_info.py` | Comprehensive Q&A document covering session deserialization behavior, normal lifecycle, corruption handling, logging observability, risk boundary analysis, and threat model assessment with code-grounded evidence |

This is the sole documentation file to produce. No existing files are modified.

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Technical Q&A / Security Analysis
Source Code:
    Primary:
        - app/session.py (RedisSessionStore, ServerSession, logout_session)
        - app/redis_services.py (initialize_redis_services)
        - server.py (create_app, session cookie config, before_request hooks)
    Authentication flow:
        - app/auth/views/login.py (login route)
        - app/auth/views/login_utils.py (after_login, MFA routing)
        - app/auth/views/mfa.py (TOTP challenge handler)
        - app/auth/views/fido.py (WebAuthn challenge handler)
        - app/auth/views/logout.py (logout route)
        - app/auth/views/recovery.py (recovery code handler)
    Session consumers:
        - app/dashboard/views/enter_sudo.py (sudo_time read/write)
        - app/api/views/user_info.py (API logout)
        - app/auth/views/facebook.py (oauth_state, next_url)
        - app/auth/views/google.py (oauth_state, next_url)
        - app/auth/views/github.py (oauth_state)
        - app/auth/views/oidc.py (oauth state/next)
        - app/auth/views/proton.py (oauth action/next/scheme/mode)
    Configuration:
        - app/config.py (FLASK_SECRET, SESSION_COOKIE_NAME, MEM_STORE_URI, MFA_USER_ID)
        - app/extensions.py (session_protection = "strong")
    Infrastructure:
        - app/log.py (LOG configuration)
        - app/parallel_limiter.py (Redis lock infrastructure)
        - app/rate_limiter.py (Redis rate limiting)
Sections:
    - Session Architecture Overview
    - Normal Session Lifecycle (login, MFA, request handling, logout)
    - Session Data Schema (all keys, types, TTLs)
    - Corruption and Malformation Handling
    - Log and Response Observability
    - Deserialization Risk Boundary Analysis
    - Threat Model: Signed Cookie vs. Unsigned Redis Data
    - Conclusions and Risk Summary
Diagrams:
    - Session request lifecycle flowchart (Mermaid)
    - Login/logout session state diagram (Mermaid)
    - Threat model boundary diagram (Mermaid)
    - Corruption handling decision tree (Mermaid)
Key Citations:
    - app/session.py:10-12 (pickle import)
    - app/session.py:38-41 (_get_signer with HMAC)
    - app/session.py:68-80 (open_session with pickle.loads and bare except)
    - app/session.py:82-114 (save_session with pickle.dumps and TTL logic)
    - app/session.py:61-66 (purge_session)
    - app/session.py:117-121 (logout_session)
    - app/redis_services.py:9-25 (initialize_redis_services)
    - app/extensions.py:8 (session_protection = "strong")
    - server.py:151 (app.secret_key = FLASK_SECRET)
    - server.py:159-162 (session cookie config)
    - server.py:204-207 (make_session_permanent, 7-day lifetime)
    - server.py:220-230 (load_user callback)
    - app/config.py:196-199 (FLASK_SECRET, SESSION_COOKIE_NAME)
    - app/config.py:295 (MFA_USER_ID = "mfa_user_id")
    - app/config.py:568 (MEM_STORE_URI)
```

### 0.5.3 Documentation Configuration Updates

No documentation framework configuration files exist in the repository (no `mkdocs.yml`, `docusaurus.config.js`, etc.), so no configuration updates are required. The output file is a standalone Markdown document placed in a new `blitzy/documentation/` directory.

### 0.5.4 Cross-Documentation Dependencies

- **No shared content/includes:** The document is self-contained.
- **No navigation links to update:** No documentation framework to reconfigure.
- **No table of contents updates:** The document does not integrate into any existing documentation index.
- **Potential future reference:** The analysis in this document could be referenced by `SECURITY.md` if the project chooses to document session deserialization behavior publicly.

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The documentation task does not require any documentation generation tools, as the output is a standalone Markdown file. However, the following project dependencies are directly relevant to the session behavior being documented and must be accurately referenced in the output document:

| Registry | Package Name | Version (from `pyproject.toml`) | Purpose in Session Context |
|----------|-------------|------|---------|
| pip (Poetry) | flask | ^1.1.2 | Web framework; provides `SessionInterface` protocol, `SessionMixin`, `session` proxy, cookie configuration |
| pip (Poetry) | flask_login | ^0.5.0 | User session management; `login_user()`, `logout_user()`, `session_protection = "strong"`, `_user_id` session key |
| pip (Poetry) | redis | ^4.5.3 | Redis client used for session storage via `limits.storage.RedisStorage.storage` |
| pip (Poetry) | Flask-Limiter | ^1.4 | Rate limiter that shares Redis connection; provides `limits.storage.RedisStorage` and `limits.storage.RedisSentinelStorage` used by `RedisSessionStore` |
| stdlib | pickle / cPickle | (stdlib) | Session data serialization and deserialization — the core of the security analysis |
| pip (transitive) | itsdangerous | (Flask dependency) | `itsdangerous.Signer` for HMAC-signing session IDs; `itsdangerous.want_bytes` for encoding |
| pip (transitive) | werkzeug | (Flask dependency) | `CallbackDict` (base class for `ServerSession`), `SessionInterface` protocol |
| pip (Poetry) | Flask-WTF | ^0.14.3 | CSRF token stored in session; relevant to corruption impact analysis |
| pip (Poetry) | sentry_sdk | ^2.16.0 | Error tracking; relevant to observability of session errors |
| pip (Poetry) | python | ^3.10 | Runtime version; determines pickle protocol defaults |

### 0.6.2 Documentation Reference Updates

Not applicable. No existing documentation links reference session management, and no link transformation is required. The new document is standalone and requires no internal link updates in existing files.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (session-related documentation):**
- Session architecture documented: 0/1 (0%) — no existing document covers the `RedisSessionStore`
- Session lifecycle documented: 0/1 (0%) — login/logout/MFA session behavior not documented
- Session security analysis documented: 0/1 (0%) — pickle deserialization risk not documented
- Session data schema documented: 0/1 (0%) — session keys not cataloged

**Target coverage after this task: 100%** — all five user questions fully answered with code-grounded evidence.

**Coverage gaps to address:**

| Topic | Current State | Target |
|-------|---------------|--------|
| Session architecture (RedisSessionStore) | Undocumented | Full behavioral description with Mermaid diagrams |
| Normal session lifecycle (login/MFA/logout) | Undocumented | Step-by-step code trace with session key enumeration |
| Corruption handling behavior | Undocumented | Analysis of `open_session()` exception handling with observable outcomes |
| Logging and response observability | Undocumented | Enumeration of what is and is not logged/surfaced during session failures |
| Deserialization risk boundary | Undocumented | Technical analysis distinguishing benign reset from RCE-capable payloads |
| Threat model (signed cookie vs. unsigned Redis) | Undocumented | Risk assessment grounded in code-traced data flow |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**
- Every answer must cite specific source files with line numbers
- Every behavioral claim must be traceable to a code path in the repository
- Every security conclusion must distinguish between what the code does and what it does not do
- The fallback behavior (no Redis) must be noted

**Accuracy validation:**
- All code references must match the actual source at the cited lines
- Pickle behavior claims must align with Python 3.10 `pickle` module semantics
- HMAC signing claims must align with `itsdangerous` library behavior
- Flask-Login session protection behavior must align with documented `"strong"` mode semantics for permanent sessions

**Clarity standards:**
- Questions are answered directly before providing supporting evidence
- Technical rationale is provided for every conclusion
- Diagrams illustrate complex flows (session lifecycle, threat model)
- Tables summarize enumerative data (session keys, failure modes)

**Maintainability:**
- Source citations use `file:line` format for traceability
- Conclusions are clearly separated from evidence so they can be updated independently

### 0.7.3 Example and Diagram Requirements

- **Minimum diagrams:** 4 (session request lifecycle, login/logout state, threat model boundary, corruption decision tree)
- **Diagram format:** Mermaid (inline in Markdown)
- **Code snippet requirements:** Short inline excerpts from `app/session.py` (the `open_session` and `save_session` methods) to ground behavioral claims
- **Table requirements:** Session data key enumeration table, failure mode comparison table, threat model comparison table

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the sole deliverable

**Source code analyzed for documentation (read-only):**
- `app/session.py` — `RedisSessionStore`, `ServerSession`, `logout_session()`
- `app/redis_services.py` — `initialize_redis_services()`
- `app/extensions.py` — Flask-Login `session_protection = "strong"`
- `app/config.py` — `FLASK_SECRET`, `SESSION_COOKIE_NAME`, `MEM_STORE_URI`, `MFA_USER_ID`
- `server.py` — `create_app()`, session cookie configuration, `make_session_permanent()`, `load_user()`, error handlers, `after_request` logging
- `app/auth/views/login.py` — Login form handling and credential verification
- `app/auth/views/login_utils.py` — `after_login()` with MFA routing and `sudo_time` setting
- `app/auth/views/mfa.py` — TOTP challenge flow
- `app/auth/views/fido.py` — WebAuthn challenge flow
- `app/auth/views/recovery.py` — Recovery code flow
- `app/auth/views/logout.py` — Logout endpoint
- `app/auth/views/facebook.py` — Facebook OAuth session usage
- `app/auth/views/google.py` — Google OAuth session usage
- `app/auth/views/github.py` — GitHub OAuth session usage
- `app/auth/views/oidc.py` — Generic OIDC session usage
- `app/auth/views/proton.py` — Proton OAuth session usage
- `app/dashboard/views/enter_sudo.py` — Sudo mode session interaction
- `app/api/views/user_info.py` — API logout endpoint
- `app/log.py` — Logging infrastructure
- `app/parallel_limiter.py` — Redis-backed concurrency locks (shared Redis)
- `app/rate_limiter.py` — Redis-backed rate limiting (shared Redis)
- `pyproject.toml` — Dependency versions
- `example.env` — Environment variable documentation
- `tests/test.env` — Test environment configuration
- `tests/conftest.py` — Test fixture setup
- `README.md`, `SECURITY.md`, `CONTRIBUTING.md` — Top-level docs
- `docs/**/*.md` — All existing documentation files

**Analysis topics in scope:**
- Normal session creation, storage, retrieval, and deletion lifecycle
- Session data serialization format (pickle) and signing mechanism (itsdangerous HMAC)
- Behavior under corrupted, truncated, or malformed Redis session data
- Observable runtime artifacts in HTTP responses and application logs
- Deserialization risk boundary between benign failure and code-execution-capable payloads
- Threat model analysis of signed session IDs vs. unsigned Redis session data
- TTL policy differentiation (authenticated vs. unauthenticated)
- Flask-Login `session_protection = "strong"` interaction with Redis-backed sessions

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No files in the repository are to be modified (per implementation rule)
- **Test file modifications:** No test files are created or modified
- **Feature additions or code refactoring:** No changes to session handling code
- **Deployment configuration changes:** No changes to Docker, Nginx, or Postfix configuration
- **Other documentation areas:** Email handling, DNS configuration, OAuth provider setup, API reference, and all other documentation topics not directly related to session deserialization behavior
- **Live runtime testing:** While the user mentions temporary scripts for observation, this planning document focuses on the documentation deliverable; any temporary scripts used during investigation are outside the persistent deliverable scope and must be cleaned up
- **Remediation recommendations:** The document answers analytical questions about current behavior; it does not propose code changes (though the analysis may inform future decisions)
- **Non-session Redis usage:** Rate limiting (`app/rate_limiter.py`), concurrency locks (`app/parallel_limiter.py`), and other Redis consumers are only documented insofar as they share the same Redis connection and establish context for the session store's deployment topology

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — output is a standalone Markdown file requiring no build step
- **Documentation preview command:** Any Markdown previewer (e.g., `grip blitzy/documentation/app_2cd6ee777f8c.md` or IDE preview)
- **Diagram generation command:** Not applicable — Mermaid diagrams are inline in the Markdown and render natively on GitHub and most Markdown renderers
- **Documentation deployment command:** Not applicable — file is committed to the repository
- **Default format:** Markdown with Mermaid diagrams
- **Citation requirement:** Every section must reference source files using `Source: path/to/file.py:LineNumber` notation
- **Style guide:** Follow the user's implementation rule: base answers on the code as truth, provide thinking/rationale, do not modify existing files
- **Documentation validation:** Manual review for code reference accuracy; Mermaid syntax validation via GitHub rendering

### 0.9.2 Content Strategy for Question Answering

Each section of the output document must follow this pattern:

1. **Direct answer** — State the conclusion plainly at the start of each section
2. **Code evidence** — Cite the specific file, line, and logic that supports the conclusion
3. **Rationale** — Explain why the code behaves this way and what the design intent appears to be
4. **Edge cases** — Note any caveats, alternative code paths, or conditions that change the behavior
5. **Security implications** — Where relevant, state what the behavior means for the system's security posture

### 0.9.3 Repository Integrity Verification

Before finalizing the documentation, verify:
- No files in the source repository have been modified (per implementation rule)
- The `blitzy/documentation/` directory is created in the destination repository only
- Any temporary observation scripts used during investigation have been removed
- The document itself contains only analysis and documentation, not executable code or patches

## 0.10 Rules for Documentation

The following rules govern the production of the documentation deliverable, drawn from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The analysis document is the only artifact produced, and it is placed in `blitzy/documentation/` in the destination repo.
- **Do not make assumptions; base answers on the code as the truth.** Every behavioral claim must be traced to a specific code path in the repository. If the code does not provide evidence for a conclusion, state that explicitly rather than speculating.
- **Provide thinking and rationale behind the answers.** The document must not merely state conclusions — it must explain why the code behaves as described, what design intent is apparent, and where ambiguity exists.
- **Temporary scripts for observation only, with cleanup.** If temporary scripts are used during investigation to observe runtime behavior (e.g., a script that writes corrupted data to a test Redis instance), those scripts must be removed before finalization. They must not persist in the repository.
- **The repository itself should remain unchanged.** This is restated for emphasis: no source file, test file, configuration file, or documentation file in the existing repository is to be modified.
- **Place the generated document in `blitzy/documentation/`.** The output file must be named `app_2cd6ee777f8c.md` per the branch-name naming convention specified by the implementation rule.
- **Source citations for all technical details.** Every claim about session behavior, error handling, cookie configuration, or security properties must cite the specific file and line number(s) that provide the evidence.
- **Mermaid diagrams for complex flows.** Session lifecycle, threat model boundaries, and corruption handling decision trees should be visualized with Mermaid diagrams embedded in the Markdown.
- **No remediation code or patches.** The document answers analytical questions — it does not propose or implement code changes, even if the analysis reveals potential improvements.

## 0.11 References

### 0.11.1 Files and Folders Searched

**Primary session infrastructure (read in full):**

| File | Lines | Key Findings |
|------|-------|-------------|
| `app/session.py` | 1–122 | `RedisSessionStore` with `pickle.loads()` / `pickle.dumps()`, HMAC-signed session IDs via `itsdangerous.Signer`, bare `except Exception: pass` on deserialization, TTL differentiation (7 days auth / 300s unauth), `logout_session()` purge |
| `app/redis_services.py` | 1–25 | `initialize_redis_services()` wires Redis or Sentinel storage into session interface, rate limiter, and concurrency locks |
| `app/extensions.py` | 1–37 | `login_manager.session_protection = "strong"`, Flask-Limiter key function (user ID or IP) |
| `server.py` | 1–600 | `create_app()` app factory, `FLASK_SECRET` as `app.secret_key`, `SESSION_COOKIE_NAME = "slapp"`, `SESSION_COOKIE_SAMESITE = "Lax"`, `SESSION_COOKIE_SECURE` conditional on HTTPS, 7-day permanent session lifetime, `load_user()` callback, `after_request` logging, error handlers |
| `app/config.py` | 190–575 | `FLASK_SECRET` (required, env var), `SESSION_COOKIE_NAME = "slapp"`, `MEM_STORE_URI` (optional, env var), `MFA_USER_ID = "mfa_user_id"` |

**Authentication flow files (read in full):**

| File | Lines | Key Findings |
|------|-------|-------------|
| `app/auth/views/login.py` | 1–83 | Login route with credential verification, delegates to `after_login()` |
| `app/auth/views/login_utils.py` | 1–69 | `after_login()` MFA routing, writes `session[MFA_USER_ID]` and `session["sudo_time"]`, referral code from `session["slref"]` |
| `app/auth/views/mfa.py` | 1–108 | TOTP challenge, reads `session.get(MFA_USER_ID)`, deletes on success, calls `login_user()` |
| `app/auth/views/fido.py` | (partial grep) | WebAuthn challenge, `session["fido_challenge"]`, `session[MFA_USER_ID]`, `session["sudo_time"]` |
| `app/auth/views/logout.py` | 1–17 | Calls `logout_session()`, deletes `slapp`, `mfa`, `dark-mode` cookies |
| `app/auth/views/recovery.py` | (grep) | Recovery code handler, reads/deletes `session[MFA_USER_ID]` |
| `app/dashboard/views/enter_sudo.py` | 1–81 | `sudo_required` decorator reads `session["sudo_time"]` with 120-second gap |
| `app/api/views/user_info.py` | 130–145 | API logout calls `logout_session()`, deletes `SESSION_COOKIE_NAME` cookie |

**Session consumer files (grep-searched):**

| File | Session Keys Used |
|------|------------------|
| `app/auth/views/facebook.py` | `session["facebook_next_url"]`, `session["oauth_state"]` |
| `app/auth/views/google.py` | `session["google_next_url"]`, `session["oauth_state"]` |
| `app/auth/views/github.py` | `session["oauth_state"]` |
| `app/auth/views/oidc.py` | `session[SESSION_STATE_KEY]`, `session[SESSION_NEXT_KEY]` |
| `app/auth/views/proton.py` | `session[SESSION_ACTION_KEY]`, `session["oauth_next"]`, `session["oauth_scheme"]`, `session["oauth_mode"]` |
| `app/internal/exit_sudo.py` | `session["sudo_time"] = 0` |

**Supporting files (read or grep-searched):**

| File | Purpose |
|------|---------|
| `app/log.py` | Logging infrastructure — `LOG` logger at DEBUG level, format includes timestamp, process, pathname, lineno, funcName |
| `app/parallel_limiter.py` | Redis distributed locks — shares Redis connection with session store |
| `app/rate_limiter.py` | Redis rate limiting — shares Redis connection with session store |
| `pyproject.toml` | Dependency manifest — `python = "^3.10"`, `flask = "^1.1.2"`, `redis = "^4.5.3"`, `Flask-Limiter = "^1.4"` |
| `example.env` | Environment variable docs — `FLASK_SECRET=secret` (example value), `MEM_STORE_URI` not listed |
| `tests/test.env` | Test config — `MEM_STORE_URI=redis://localhost`, `FLASK_SECRET=secret` |
| `tests/conftest.py` | Test fixtures — `WTF_CSRF_ENABLED = False`, `create_app()` used in tests |
| `Dockerfile` | Build config — `FROM python:3.10`, port 7777 |
| `README.md` | Self-hosting guide — no session internals |
| `SECURITY.md` | Disclosure policy — `security@simplelogin.io` |
| `CONTRIBUTING.md` | Contributor workflow |
| `docs/code-structure.md` | TODO note about `local_data/` JWT keys |
| `docs/api.md` | REST API reference |
| `docs/` (all 12 .md files) | Scanned for session-related content — none found |

**Folders explored:**

| Folder | Depth | Findings |
|--------|-------|----------|
| Root (`""`) | Level 0 | Full project structure — 14 top-level files, 14 subdirectories |
| `app/` | Level 1 | 47 Python modules, 17 subpackages — session.py, redis_services.py, extensions.py, config.py identified as primary targets |
| `app/auth/views/` | Level 2 | 19 authentication view modules — login, logout, MFA, FIDO, recovery, social providers all examined |
| `app/dashboard/views/` | Level 2 | enter_sudo.py identified as session consumer |
| `app/api/views/` | Level 2 | user_info.py logout endpoint identified |
| `docs/` | Level 1 | 12 Markdown files + 1 SVG — no session-related documentation |
| `tests/` | Level 1 | conftest.py and test.env examined for test session config |

### 0.11.2 External Research

| Topic | Source | Key Finding |
|-------|--------|-------------|
| Python pickle deserialization RCE | Semgrep docs, GitHub advisories (GHSA-g8c6-8fjj-2r4m), PayloadsAllTheThings | `pickle.loads()` executes arbitrary code via `__reduce__` during deserialization; classified as CWE-502; Python docs explicitly warn against untrusted data |
| Flask-Login session_protection strong | Flask-Login 0.7.0 docs, GitHub issues #297, #231 | In `"strong"` mode with permanent sessions, Flask-Login marks session as non-fresh (`_fresh = False`) rather than deleting it; for non-permanent sessions, session is deleted on identifier mismatch |
| Flask session cookie security | Miguel Grinberg blog | Standard Flask sessions are signed but not encrypted; server-side sessions (Redis) protect data confidentiality but shift trust to the Redis store's integrity |

### 0.11.3 Tech Spec Sections Referenced

| Section | Content Used |
|---------|-------------|
| 6.4 Security Architecture | Session management documentation (6.4.1.4), authentication framework (6.4.1.1–6.4.1.3), token architecture (6.4.1.5), security control matrix (6.4.6) |

### 0.11.4 Attachments

No attachments were provided by the user. No Figma URLs or design files are referenced in this task.

