# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new investigative documentation** that comprehensively traces and documents the custom alias creation validation pathway in the SimpleLogin application, answering specific behavioral questions about HTTP responses, log output, rate-limiting headers, and quota enforcement logic — all derived from reading the actual source code, without modifying any existing repository files.

**Category:** Create new documentation
**Documentation Type:** Technical investigation / Q&A document based on source code analysis

The requirements decompose into the following concrete documentation tasks:

- **Signed suffix validation behavior:** Document the exact HTTP status codes and JSON error messages returned by the API when an invalid or expired signed suffix is submitted during custom alias creation, citing the responsible code paths in `app/alias_suffix.py` and `app/api/views/new_custom_alias.py`
- **Server-side log entries for validation failures:** Identify and document the exact `LOG.w()` and `LOG.e()` statements emitted to the server console when signed suffix verification fails, including the log format defined in `app/log.py`
- **Rate-limiting header presence and values:** Determine whether Flask-Limiter injects rate-limiting response headers (e.g., `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`) and what their actual values are for alias creation endpoints, based on the `ALIAS_LIMIT` configuration
- **Quota verification flow for successful creation:** Trace and document the quota-checking logic that runs before alias creation succeeds, specifically the `User.can_create_new_alias()` method, `User.max_alias_for_free_account()`, and the subscription/tier checks in `app/models.py`
- **Logged values during quota checks:** Document what specific values (user ID, alias count, plan tier, quota limit) appear in server logs when the system verifies whether a user can create more aliases
- **Component responsibility mapping:** Identify which exact modules and functions validate signed suffixes (`app/alias_suffix.check_suffix_signature`) and enforce creation limits (`app/extensions.limiter`, `app/parallel_limiter`, `User.can_create_new_alias`), and what conditions trigger rejection at each stage

**Inferred Documentation Needs:**

- The execution path traverses multiple decorator layers (`@limiter.limit`, `@require_api_auth`, `@parallel_limiter.lock`) before reaching the endpoint handler — the document must trace each layer's contribution
- The `itsdangerous.TimestampSigner` used for suffix signing has a 600-second (`max_age`) expiry window that is a key parameter to document
- The distinction between HTTP 412 (expired suffix) and HTTP 400 (tampered suffix) is critical for diagnosing intermittent failures
- Both v2 and v3 API endpoints share the same validation logic but differ in mailbox handling — both need coverage

### 0.1.2 Special Instructions and Constraints

- **No repository code modification:** The user explicitly states: "Don't modify the repository code — temporary test scripts or API calls are fine but keep the codebase unchanged." The output document must be purely investigative, based on source code analysis.
- **Implementation rule (SWE-AtlasQnA-Repo):** Create a new markdown document named `app_2cd6ee777f8c.md` (matching the source branch name) that comprehensively answers all questions posed. Provide thinking/rationale behind answers. Do not make assumptions — base answers on the code as truth. Do not modify any existing files. Place the document in the `blitzy/documentation` directory.
- **Answers must be evidence-based:** Every claim must cite specific source files, line numbers, and code references
- **Thinking and rationale required:** The document must not merely list facts but explain the reasoning and code flow behind each answer

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document signed suffix validation behavior, we will **create** `blitzy/documentation/app_2cd6ee777f8c.md` by tracing the execution path through `app/alias_suffix.py:check_suffix_signature()` (lines 37–42), the `itsdangerous.TimestampSigner.unsign()` call with `max_age=600`, and the error handling in `app/api/views/new_custom_alias.py` (lines 69–76 for v2, lines 184–191 for v3)
- To document server console log entries, we will extract the `LOG.w()` and `LOG.d()` calls from both `new_custom_alias.py` and `alias_suffix.py`, correlating them with the log format string defined in `app/log.py` (line 13): `"%(asctime)s - %(name)s - %(levelname)s - %(process)d - \"%(pathname)s:%(lineno)d\" - %(funcName)s() - %(message_id)s - %(message)s"`
- To document rate-limiting headers, we will analyze the Flask-Limiter configuration in `app/extensions.py`, the `ALIAS_LIMIT` value from `app/config.py` (line 448: `"100/day;50/hour;5/minute"`), and Flask-Limiter's default header injection behavior
- To document quota verification, we will trace `User.can_create_new_alias()` in `app/models.py` (lines 867–884), which chains through `is_active()`, `disabled`, `lifetime_or_active_subscription()`, `Alias.filter_by(user_id=self.id).count()`, and `max_alias_for_free_account()`

### 0.1.4 Inferred Documentation Needs

- Based on code analysis: The `app/api/views/new_custom_alias.py` module contains two endpoint versions (v2 at `/api/v2/alias/custom/new` and v3 at `/api/v3/alias/custom/new`) — both must be covered since they share validation logic but have different input requirements
- Based on structure: The validation pipeline spans five separate modules (`app/api/base.py`, `app/extensions.py`, `app/parallel_limiter.py`, `app/alias_suffix.py`, `app/models.py`) requiring consolidated documentation
- Based on the `app/dashboard/views/custom_alias.py` module: The dashboard route performs the same validation with `check_suffix_signature()` and `verify_prefix_suffix()` but with different error handling (flash messages vs JSON), providing additional context for the API behavior
- Based on the `server.py` error handler at line 362: The HTTP 429 rate-limit response is handled at the application level with a specific JSON format `{"error": "Rate limit exceeded"}`, which must be documented
- Based on test file `tests/api/test_new_custom_alias.py`: The test suite includes explicit test cases for quota exhaustion (`test_out_of_quota`), rate limiting (`test_too_many_requests`), and suffix tampering, providing concrete expected behavior to document

## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **flat documentation structure** within the `docs/` directory containing 12 markdown files and one SVG asset, with no documentation generator framework (no `mkdocs.yml`, `docusaurus.config.js`, or `sphinx/conf.py` detected). Documentation is written as standalone Markdown files with no automated build pipeline.

**Documentation files discovered:**

| File | Category | Relevance to Task |
|------|----------|--------------------|
| `docs/api.md` | API reference | **High** — contains existing alias creation endpoint docs |
| `docs/troubleshooting.md` | Operational runbook | **Medium** — diagnostic patterns for email issues |
| `docs/code-structure.md` | Code structure note | **Low** — minimal TODO-style note about `local_data/` |
| `docs/oauth.md` | Authentication reference | **Low** — OAuth flows, not alias creation |
| `docs/build-image.md` | Build guide | None |
| `docs/upgrade.md` | Upgrade runbook | None |
| `docs/ssl.md` | TLS/SSL guide | None |
| `docs/ses.md` | SES relay guide | None |
| `docs/gmail-relay.md` | Gmail relay guide | None |
| `docs/enforce-spf.md` | SPF hardening | None |
| `docs/postfix-tls.md` | Postfix TLS config | None |
| `docs/ufw.md` | Firewall guide | None |

**Top-level documentation:**

| File | Category | Relevance |
|------|----------|-----------| 
| `README.md` | Self-hosting guide | **Medium** — establishes documentation style |
| `CONTRIBUTING.md` | Contributor guide | **Low** — process documentation |
| `SECURITY.md` | Vulnerability disclosure | None |

- **Documentation framework:** None (plain Markdown files, no generator)
- **API documentation tool:** Manual Markdown in `docs/api.md` — no JSDoc, Sphinx, or automated generation
- **Diagram tools detected:** None in repository (Mermaid recommended for new documentation)
- **Documentation hosting:** GitHub-served Markdown (no deployment pipeline)

### 0.2.2 Repository Code Analysis for Documentation

**Search patterns used for code to document:**

- Alias creation endpoints: `app/api/views/new_custom_alias.py` — contains `new_custom_alias_v2()` and `new_custom_alias_v3()` handlers
- Signed suffix verification: `app/alias_suffix.py` — contains `check_suffix_signature()`, `verify_prefix_suffix()`, and `get_alias_suffixes()`
- Alias utility functions: `app/alias_utils.py` — contains `check_alias_prefix()` and auto-creation logic
- Rate limiting infrastructure: `app/extensions.py` (Flask-Limiter setup), `app/parallel_limiter.py` (concurrency locks), `app/rate_limiter.py` (bucket-based limiting)
- Quota enforcement: `app/models.py` — `User.can_create_new_alias()`, `User.is_premium()`, `User.max_alias_for_free_account()`
- API authentication: `app/api/base.py` — `authorize_request()`, `require_api_auth()` decorator
- Configuration constants: `app/config.py` — `ALIAS_LIMIT`, `MAX_NB_EMAIL_FREE_PLAN`, `CUSTOM_ALIAS_SECRET`
- Logging infrastructure: `app/log.py` — log format, `EmailHandlerFilter`, shortcuts
- Error handler: `server.py` — HTTP 429 rate-limit error handler (lines 362–372)
- Dashboard variant: `app/dashboard/views/custom_alias.py` — web UI alias creation with same validation
- Error definitions: `app/errors.py` — custom exception hierarchy

**Key directories examined:**

- `app/api/views/` — 17 API view modules (alias_options, new_custom_alias, new_random_alias, alias, auth, etc.)
- `app/api/` — base blueprint, serializer, views package
- `app/dashboard/views/` — web UI route handlers
- `docs/` — 12 documentation files plus 1 SVG asset
- `tests/api/` — 17 test modules including `test_new_custom_alias.py`

**Related documentation found:**

- `docs/api.md` (lines 381–400) — existing API reference for `POST /api/v3/alias/custom/new` with input/output specification but **no error response documentation** and **no validation behavior detail**
- `docs/api.md` (lines 80–92) — generic error response format description and `4**` status code convention
- `tests/api/test_new_custom_alias.py` — 10 test cases that exercise validation scenarios and serve as behavioral specifications

### 0.2.3 Web Search Research Conducted

No external web searches are necessary for this task. All answers derive from the source code itself, per the user's instruction to base answers on the code as truth rather than assumptions. The documentation tools and frameworks in use are minimal (plain Markdown), requiring no external research.

## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

**Modules requiring documentation in the new Q&A document:**

- **Module: `app/api/views/new_custom_alias.py`**
  - Public APIs: `new_custom_alias_v2()` (line 32), `new_custom_alias_v3()` (line 119)
  - Current documentation: Partial in `docs/api.md` (input/output only, no error detail)
  - Documentation needed: Complete HTTP response matrix, error messages, validation conditions, decorator chain behavior

- **Module: `app/alias_suffix.py`**
  - Public APIs: `check_suffix_signature()` (line 37), `verify_prefix_suffix()` (line 45), `get_alias_suffixes()` (line 94)
  - Current documentation: None
  - Documentation needed: Signing mechanism, expiry window (600s), failure modes, LOG entries

- **Module: `app/models.py` (User class)**
  - Public APIs: `can_create_new_alias()` (line 867), `is_premium()` (line 787), `max_alias_for_free_account()` (line 858), `is_active()` (line 766), `lifetime_or_active_subscription()` (line 746)
  - Current documentation: None
  - Documentation needed: Quota logic flow, subscription tier checks, alias count evaluation, logged values

- **Module: `app/extensions.py`**
  - Public APIs: `limiter` (Flask-Limiter instance, line 23), `__key_func()` (line 14)
  - Current documentation: None
  - Documentation needed: Rate limit key resolution (user-based vs IP-based), header injection behavior

- **Module: `app/parallel_limiter.py`**
  - Public APIs: `lock()` (line 68), `_InnerLock` class (line 19)
  - Current documentation: None
  - Documentation needed: Redis-based concurrency lock, lock name format, TooManyRequests behavior

- **Module: `app/rate_limiter.py`**
  - Public APIs: `check_bucket_limit()` (line 19)
  - Current documentation: None
  - Documentation needed: Bucket-based rate limit logic, Redis INCR, New Relic event emission

- **Module: `app/api/base.py`**
  - Public APIs: `authorize_request()` (line 16), `require_api_auth()` (line 52)
  - Current documentation: Partial in `docs/api.md` (header convention only)
  - Documentation needed: Auth error codes (401, 403), account state checks

- **Module: `server.py`**
  - Relevant function: `rate_limited()` error handler (line 362)
  - Current documentation: None
  - Documentation needed: 429 response format for API routes

- **Module: `app/config.py`**
  - Relevant constants: `ALIAS_LIMIT` (line 448), `MAX_NB_EMAIL_FREE_PLAN` (line 121), `CUSTOM_ALIAS_SECRET` (line 201), `DISABLE_RATE_LIMIT` (line 602)
  - Current documentation: Partial in `example.env`
  - Documentation needed: Default values, impact on validation behavior

- **Module: `app/log.py`**
  - Relevant definitions: `_log_format` (line 13), `EmailHandlerFilter` (line 28), `LOG` shortcuts (lines 74–78)
  - Current documentation: None
  - Documentation needed: Log format template, log level shortcuts (`.d`, `.i`, `.w`, `.e`)

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented validation error responses:** The existing `docs/api.md` describes only the success path (201) and generic error format. It does not enumerate the specific error codes (400, 409, 412, 429) or their JSON payloads for alias creation validation failures.
- **No rate-limiting behavior documentation:** There is no documentation about the `ALIAS_LIMIT` rate of `100/day;50/hour;5/minute`, how Flask-Limiter resolves rate-limit keys, or whether response headers expose rate-limit state.
- **No signed suffix mechanics documentation:** The `itsdangerous.TimestampSigner` signing mechanism, 600-second expiry window, and `CUSTOM_ALIAS_SECRET` derivation (`FLASK_SECRET + "custom_alias"`) are entirely undocumented.
- **No quota enforcement documentation:** The `can_create_new_alias()` decision tree — spanning subscription checks, alias counts, and `MAX_NB_EMAIL_FREE_PLAN` thresholds — has no written documentation.
- **No logging behavior documentation:** The log format, log levels used during alias creation, and specific log entries emitted at each validation stage are not documented anywhere.
- **No decorator chain documentation:** The stacking of `@limiter.limit`, `@require_api_auth`, and `@parallel_limiter.lock` decorators and their order of execution is not documented.

## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The output document `blitzy/documentation/app_2cd6ee777f8c.md` will follow a Q&A investigation structure, organized by the user's specific questions, with code-traced evidence for each answer:

```
blitzy/
└── documentation/
    └── app_2cd6ee777f8c.md
        ├── Overview (investigation scope and summary)
        ├── Q1: HTTP Status Codes and Error Messages for Invalid/Expired Signed Suffixes
        │   ├── Expired suffix → HTTP 412 response
        │   ├── Tampered suffix → HTTP 400 response
        │   ├── Invalid prefix/suffix combination → HTTP 400 response
        │   └── Complete error response table
        ├── Q2: Validation Log Entries in the Server Console
        │   ├── Log format structure
        │   ├── Suffix expiry log entries
        │   ├── Suffix tampering log entries
        │   └── Prefix/suffix verification log entries
        ├── Q3: Rate-Limiting Headers in Responses
        │   ├── Flask-Limiter header behavior
        │   ├── ALIAS_LIMIT configuration values
        │   ├── Concurrency lock (parallel_limiter) behavior
        │   └── Header presence analysis
        ├── Q4: Quota-Related Checks for Successful Alias Creation
        │   ├── can_create_new_alias() decision tree
        │   ├── Subscription tier checks
        │   ├── Alias count evaluation
        │   └── Logged values during quota verification
        ├── Q5: Execution Path and Component Responsibility
        │   ├── Decorator chain execution order
        │   ├── Component-by-component validation trace
        │   ├── Rejection conditions summary
        │   └── Mermaid execution flow diagram
        └── References (source files cited)
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract HTTP response codes and error messages from `app/api/views/new_custom_alias.py` by tracing each `return jsonify(error=...), STATUS_CODE` statement
- Extract log entries from `LOG.w()`, `LOG.d()`, `LOG.e()` calls in `new_custom_alias.py`, `alias_suffix.py`, and `alias_utils.py`, correlating with the format string in `app/log.py`
- Determine rate-limiting header behavior from Flask-Limiter's documented defaults and the configuration in `app/extensions.py`
- Trace the `can_create_new_alias()` call chain through `app/models.py`, identifying each conditional branch and its logging
- Generate a Mermaid sequence/flowchart diagram mapping the complete request lifecycle from decorator chain entry to response

**Documentation Standards:**

- Markdown formatting with proper headers (# through ####)
- Code citations in the format: `Source: /path/to/file.py:LineNumber`
- Tables for error response matrices and configuration parameters
- Mermaid diagrams for execution flow visualization
- Inline code blocks for exact error message strings and log format templates

### 0.4.3 Diagram and Visual Strategy

**Mermaid diagrams to create:**

- **Execution flow diagram:** Flowchart tracing the complete request path from HTTP request through the decorator chain (`limiter.limit` → `require_api_auth` → `parallel_limiter.lock` → handler function) to response, with all rejection branches
- **Quota check decision tree:** Flowchart of `User.can_create_new_alias()` logic including `is_active()`, `disabled`, `lifetime_or_active_subscription()`, and alias count comparison
- **Signed suffix validation flow:** Flowchart of `check_suffix_signature()` → `verify_prefix_suffix()` with all failure modes

## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---------------------------|----------------|------------------|-----------------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | CREATE | `app/api/views/new_custom_alias.py`, `app/alias_suffix.py`, `app/alias_utils.py`, `app/models.py`, `app/extensions.py`, `app/parallel_limiter.py`, `app/rate_limiter.py`, `app/api/base.py`, `app/config.py`, `app/log.py`, `server.py`, `app/errors.py`, `app/dashboard/views/custom_alias.py` | Comprehensive Q&A document answering all questions about custom alias creation validation behavior, HTTP responses, log entries, rate-limiting headers, quota checks, and execution path tracing |

**Transformation Modes Used:**

- **CREATE** — One new file: `blitzy/documentation/app_2cd6ee777f8c.md`
- No UPDATE, DELETE, or REFERENCE transformations are needed since the implementation rules explicitly prohibit modifying existing files

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/app_2cd6ee777f8c.md
Type: Technical Investigation / Q&A Document
Source Code Files:
    - app/api/views/new_custom_alias.py (primary — endpoint handlers)
    - app/alias_suffix.py (signed suffix verification)
    - app/alias_utils.py (alias prefix validation)
    - app/models.py (User.can_create_new_alias, quota logic)
    - app/extensions.py (Flask-Limiter setup)
    - app/parallel_limiter.py (concurrency lock)
    - app/rate_limiter.py (bucket-based rate limiter)
    - app/api/base.py (API auth decorator)
    - app/config.py (ALIAS_LIMIT, MAX_NB_EMAIL_FREE_PLAN, CUSTOM_ALIAS_SECRET)
    - app/log.py (log format, LOG singleton)
    - server.py (429 error handler)
    - app/errors.py (exception hierarchy)
    - app/dashboard/views/custom_alias.py (dashboard variant)
Sections:
    - Overview (investigation scope, branch context, methodology)
    - Q1: HTTP Status Codes and Error Messages for Invalid/Expired Signed Suffixes
        - Expired suffix: HTTP 412, {"error": "Alias creation time is expired, please retry"}
        - Tampered suffix: HTTP 400, {"error": "Tampered suffix"}
        - Wrong prefix/suffix: HTTP 400, {"error": "wrong alias prefix or suffix"}
        - Complete error response table with all possible codes (400, 409, 412, 429, 401, 403)
    - Q2: Validation Log Entries in Server Console
        - Log format template from app/log.py
        - LOG.w entries for expired/tampered suffixes
        - LOG.d entries for quota checks
        - LOG.e entries for prefix/suffix verification failures
    - Q3: Rate-Limiting Headers in Responses
        - Flask-Limiter default header injection behavior
        - ALIAS_LIMIT = "100/day;50/hour;5/minute"
        - Rate limit key function (userid vs ip)
        - parallel_limiter 429 on contention
        - Analysis of Retry-After, X-RateLimit-* header presence
    - Q4: Quota-Related Checks for Successful Alias Creation
        - can_create_new_alias() decision tree traced through code
        - is_active(), disabled check, lifetime_or_active_subscription()
        - Alias.filter_by(user_id=self.id).count() < max_alias_for_free_account()
        - MAX_NB_EMAIL_FREE_PLAN (default 5) and MAX_NB_EMAIL_OLD_FREE_PLAN (default 15)
        - Specific LOG.d values during quota verification
    - Q5: Execution Path and Component Responsibility
        - Decorator chain: limiter.limit → require_api_auth → parallel_limiter.lock → handler
        - Component responsibility matrix
        - Rejection condition summary table
        - Mermaid execution flow diagram
    - References (all source files with line numbers)
Diagrams:
    - Execution flow diagram (Mermaid flowchart)
    - Quota check decision tree (Mermaid flowchart)
    - Signed suffix validation flow (Mermaid flowchart)
Key Citations:
    app/api/views/new_custom_alias.py (lines 28-113, 115-236)
    app/alias_suffix.py (lines 11, 37-42, 45-91)
    app/models.py (lines 766-769, 787-800, 858-884)
    app/extensions.py (lines 14-23)
    app/parallel_limiter.py (lines 30-34, 48-63)
    app/config.py (lines 121-124, 201, 448, 602)
    app/log.py (lines 13-16, 74-79)
    app/api/base.py (lines 16-43, 52-60)
    server.py (lines 362-372)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration updates are needed. The repository uses no documentation generator framework — all documentation is standalone Markdown served directly from GitHub. The `blitzy/documentation/` directory will be created as a new output directory per the implementation rules.

### 0.5.4 Cross-Documentation Dependencies

- The new document references `docs/api.md` for the existing API specification of `POST /api/v3/alias/custom/new` and `GET /api/v5/alias/options`
- The new document supplements (but does not modify) the existing error format documentation in `docs/api.md` (lines 80–92)
- No navigation links, TOC updates, or index changes are required since there is no documentation build system

## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The following packages from the project's dependency manifest (`pyproject.toml`) are directly relevant to the documentation task, as their behavior must be accurately described in the Q&A document:

| Registry | Package Name | Version | Purpose |
|----------|--------------|---------|---------|
| pip (PyPI) | itsdangerous | (transitive via flask ^1.1.2) | `TimestampSigner` used for signed suffix creation and verification in `app/alias_suffix.py` |
| pip (PyPI) | flask | ^1.1.2 | Web framework; provides `jsonify`, `request`, `g` context, and error handler registration |
| pip (PyPI) | Flask-Limiter | ^1.4 | HTTP rate limiting with `ALIAS_LIMIT` string; injects rate-limit response headers |
| pip (PyPI) | redis | ^4.5.3 | Backend storage for rate limiter, session store, and concurrency locks |
| pip (PyPI) | flask_login | ^0.5.0 | Session-based auth used as fallback when no API key header is present |
| pip (PyPI) | SQLAlchemy | 1.3.24 | ORM for `Alias.filter_by().count()` in quota checks |
| pip (PyPI) | arrow | ^0.16.0 | Timestamp handling in API key usage tracking |
| pip (PyPI) | coloredlogs | ^14.0 | Optional colored log output (when `COLOR_LOG` is set) |
| pip (PyPI) | werkzeug | (transitive via flask) | `exceptions.TooManyRequests` raised by concurrency lock |
| pip (PyPI) | limits | (transitive via Flask-Limiter) | `RedisStorage` used by `parallel_limiter` and `rate_limiter` |

No new documentation-tool dependencies are required. The output is a plain Markdown file with Mermaid diagram blocks (renderable by GitHub's built-in Mermaid support). No additional packages need to be installed.

### 0.6.2 Documentation Reference Updates

Not applicable. The implementation rules explicitly prohibit modifying existing files. No link updates are needed in existing documentation.

## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (for the specific topic of custom alias creation validation):**

- Public APIs documented in `docs/api.md`: 1/2 endpoints (v3 has input/output docs; v2 referenced but less detailed) — **50%**
- Error responses documented: 0/8 distinct error conditions — **0%**
- Rate-limiting behavior documented: 0/3 limiting mechanisms (Flask-Limiter, parallel_limiter, bucket-based) — **0%**
- Quota enforcement logic documented: 0/5 decision points in `can_create_new_alias()` — **0%**
- Server log entries documented: 0/8 distinct LOG calls in the validation path — **0%**
- Execution path components documented: 0/6 modules in the decorator/validation chain — **0%**

**Target coverage after documentation creation: 100%** — every question posed by the user must have a complete, evidence-based answer citing specific source code.

**Coverage gaps to address:**

| Area | Current | Target | Focus |
|------|---------|--------|-------|
| Error response codes | 0% | 100% | All HTTP codes (400, 401, 403, 409, 412, 429) with exact JSON payloads |
| Server log entries | 0% | 100% | All LOG.w, LOG.d, LOG.e calls in the validation path |
| Rate-limit headers | 0% | 100% | Flask-Limiter header injection, ALIAS_LIMIT values, Retry-After behavior |
| Quota logic | 0% | 100% | Complete can_create_new_alias() decision tree with subscription checks |
| Component mapping | 0% | 100% | Every module in the execution chain with responsibility and rejection conditions |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every question posed by the user has a dedicated section with a direct, unambiguous answer
- All HTTP status codes returned by the custom alias creation endpoints are enumerated with their exact JSON error bodies
- All LOG statements in the validation path are listed with their exact format string and argument placeholders
- The decorator chain execution order is fully traced from outermost to innermost
- The quota enforcement decision tree is documented with every branch and threshold value

**Accuracy validation:**

- Every claim cites a specific source file and line number (e.g., `Source: app/alias_suffix.py:40`)
- Error messages are quoted verbatim from the source code, not paraphrased
- Configuration default values are extracted from `app/config.py` with fallback logic documented
- The `itsdangerous` signing behavior is described per the actual `signer.unsign(signed_suffix, max_age=600)` call, not generalized

**Clarity standards:**

- Thinking and rationale are provided behind each answer, not just bare facts
- Complex logic (like the `can_create_new_alias()` chain) is visualized with Mermaid diagrams
- A comprehensive summary table consolidates all rejection conditions in one place
- Code references use consistent formatting throughout

### 0.7.3 Example and Diagram Requirements

- Minimum examples per error condition: 1 (exact JSON response body)
- Diagram types required: Mermaid flowcharts for execution path, quota logic, and suffix validation
- Code example format: Inline code blocks showing exact function calls and return statements from the source
- Log entry examples: Full log line with format template filled in using realistic placeholder values

## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**
- `blitzy/documentation/app_2cd6ee777f8c.md` — the sole deliverable document

**Source code files analyzed (read-only) for documentation content:**
- `app/api/views/new_custom_alias.py` — v2 and v3 custom alias creation endpoints
- `app/alias_suffix.py` — signed suffix signing, verification, and prefix/suffix validation
- `app/alias_utils.py` — alias prefix validation (`check_alias_prefix`)
- `app/models.py` — `User.can_create_new_alias()`, `User.is_premium()`, `User.is_active()`, `User.max_alias_for_free_account()`, `User.lifetime_or_active_subscription()`
- `app/extensions.py` — Flask-Limiter configuration and rate-limit key function
- `app/parallel_limiter.py` — Redis-backed concurrency lock mechanism
- `app/rate_limiter.py` — bucket-based rate limiting with Redis INCR
- `app/api/base.py` — API authentication decorator and account state checks
- `app/config.py` — `ALIAS_LIMIT`, `MAX_NB_EMAIL_FREE_PLAN`, `CUSTOM_ALIAS_SECRET`, `DISABLE_RATE_LIMIT`
- `app/log.py` — log format string, `EmailHandlerFilter`, `LOG` singleton
- `server.py` — HTTP 429 error handler for API routes
- `app/errors.py` — custom exception hierarchy (`AliasInTrashError`, etc.)
- `app/dashboard/views/custom_alias.py` — dashboard route (for cross-reference)
- `app/api/views/alias_options.py` — suffix option generation (v4/v5 endpoints)
- `app/api/views/new_random_alias.py` — random alias creation (for cross-reference)
- `app/constants.py` — HTTP header constants

**Reference documentation (read-only):**
- `docs/api.md` — existing API specification
- `tests/api/test_new_custom_alias.py` — behavioral test specifications

**Topics in scope:**
- HTTP status codes and JSON error payloads for all custom alias validation failures
- Server-side log entries (format, level, content) for validation events
- Rate-limiting response headers (presence, values, configuration)
- Quota enforcement logic (subscription checks, alias counts, thresholds)
- Complete execution path trace from request entry to response
- Component responsibility mapping for signed suffix validation and limit enforcement
- Conditions that trigger rejection at each validation stage

### 0.8.2 Explicitly Out of Scope

- **Source code modifications** — The user explicitly stated "Don't modify the repository code." No changes to any `.py`, `.html`, `.yml`, or other source files
- **Existing documentation updates** — Implementation rules (SWE-AtlasQnA-Repo) state "Do not modify any existing files in the source repository"
- **Test file modifications** — No test files will be created or modified
- **Email handler documentation** — The inbound email processing pipeline (`email_handler.py`) is not part of the custom alias creation flow and is excluded
- **Dashboard/UI flow documentation** — The web dashboard alias creation at `/dashboard/custom_alias` is referenced for cross-comparison only; the focus is on API behavior
- **Random alias creation** — `POST /api/alias/random/new` is excluded unless used for comparative context
- **OAuth, mailbox, and custom domain endpoints** — Other API endpoints are out of scope
- **Deployment, infrastructure, or configuration guide changes** — No changes to `README.md`, `docker-compose.yml`, or operational docs
- **Feature additions, bug fixes, or refactoring** — The task is strictly documentation
- **Running the application or executing test scripts** — The user mentioned temporary test scripts are acceptable, but the code tracing is done via static analysis of the source

## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — plain Markdown, no build system
- **Documentation preview command:** Not applicable — Markdown is renderable natively on GitHub
- **Diagram generation command:** Not applicable — Mermaid blocks are rendered by GitHub's built-in Mermaid support
- **Documentation deployment command:** Not applicable — the file is placed directly in `blitzy/documentation/`
- **Default format:** Markdown with Mermaid diagram blocks
- **Citation requirement:** Every answer section must reference source files with line numbers in the format `Source: path/to/file.py:LineNumber`
- **Style guide:** Follow the repository's existing Markdown conventions observed in `docs/api.md` and `README.md`:
  - Use `##` for major sections, `###` for subsections
  - Use fenced code blocks with language identifiers for python, json, and mermaid content
  - Use tables for structured data (error codes, configuration values)
  - Keep inline code in backticks for file paths, function names, and variable names
- **Documentation validation:** Manual review — verify that all quoted error messages match source code verbatim and all line numbers are accurate
- **Output location:** `blitzy/documentation/app_2cd6ee777f8c.md` (per SWE-AtlasQnA-Repo implementation rule, using the source branch name `app_2cd6ee777f8c`)

## 0.10 Rules for Documentation

### 0.10.1 User-Specified Rules

The following rules are explicitly mandated by the user and the implementation rule set:

- **Do not modify any existing files in the source repository.** The codebase must remain entirely unchanged. Only the new document in `blitzy/documentation/` is permitted as output.
- **Base all answers on the code as truth.** Do not make assumptions or rely on external documentation — every claim must be traceable to a specific source file and line number in the repository.
- **Provide thinking and rationale behind answers.** The document must explain the reasoning and code flow behind each finding, not merely list facts.
- **Create the document as `app_2cd6ee777f8c.md`** — the file name must match the source branch name exactly.
- **Place the document in the `blitzy/documentation` directory** in the destination repository.
- **Temporary test scripts or API calls are acceptable** for investigation purposes, but must not persist as modifications to the repository.
- **Comprehensively answer all questions posed** — no question should be left partially addressed or deferred.

### 0.10.2 Documentation-Specific Conventions

- All error messages quoted in the document must be verbatim copies from the source code (e.g., `"Alias creation time is expired, please retry"` from `app/api/views/new_custom_alias.py:73`)
- Configuration values must include their default fallback behavior (e.g., `MAX_NB_EMAIL_FREE_PLAN` defaults to `5` if the environment variable is not set)
- Log entries must be presented with the complete log format template from `app/log.py`, showing where each field appears
- Mermaid diagrams should be used for any flow that involves three or more decision branches
- Source citations should follow the pattern `Source: relative/path/to/file.py:LineNumber` throughout
- The document should be self-contained — a reader should not need to open source files to understand the answers, though source references are provided for verification

## 0.11 References

### 0.11.1 Files and Folders Searched

**Source code files read and analyzed (with line ranges):**

| File Path | Lines Read | Key Findings |
|-----------|------------|--------------|
| `app/api/views/new_custom_alias.py` | 1–236 | v2 endpoint (lines 28–113) and v3 endpoint (lines 115–236); quota check at line 48/137; suffix validation at lines 70–76/184–191; error responses: 400 (multiple), 409, 412 |
| `app/alias_suffix.py` | 1–193 | `TimestampSigner` with `CUSTOM_ALIAS_SECRET` (line 11); `check_suffix_signature()` with `max_age=600` (lines 37–42); `verify_prefix_suffix()` with domain and prefix validation (lines 45–91); `get_alias_suffixes()` suffix generation (lines 94–193) |
| `app/alias_utils.py` | 1–599 | `check_alias_prefix()` regex validation (lines 414–425); `_ALIAS_PREFIX_PATTERN = r"[0-9a-z-_.]{1,}"` (line 415); auto-creation logic with quota checks |
| `app/models.py` | 710–920 | `can_create_new_alias()` (lines 867–884); `is_premium()` (lines 787–800); `max_alias_for_free_account()` (lines 858–865); `is_active()` (lines 766–769); `lifetime_or_active_subscription()` (lines 746–753) |
| `app/extensions.py` | 1–37 | Flask-Limiter setup with `__key_func()` (lines 14–19); `limiter = Limiter(key_func=__key_func)` (line 23); `DISABLE_RATE_LIMIT` filter (lines 26–28) |
| `app/parallel_limiter.py` | 1–74 | `_InnerLock` class (lines 19–65); Redis SET NX with TTL (lines 30–33); `TooManyRequests` on contention (line 34); lock name format `cl:{user_id}:{suffix}` (lines 56–58) |
| `app/rate_limiter.py` | 1–43 | `check_bucket_limit()` (lines 19–42); bucket ID calculation (lines 25–27); Redis INCR with expiry (line 31); New Relic custom event (lines 36–39) |
| `app/api/base.py` | 1–74 | `authorize_request()` (lines 16–43); API key lookup via `Authentication` header (lines 17–18); disabled → 403 (line 37); inactive → 401 (lines 39–40); `require_api_auth()` decorator (lines 52–60) |
| `app/config.py` | 115–210, 440–460, 590–620 | `MAX_NB_EMAIL_FREE_PLAN` default 5 (lines 121–124); `MAX_NB_EMAIL_OLD_FREE_PLAN` default 15 (line 126); `CUSTOM_ALIAS_SECRET = FLASK_SECRET + "custom_alias"` (line 201); `ALIAS_LIMIT = "100/day;50/hour;5/minute"` (line 448); `DISABLE_RATE_LIMIT` (line 602) |
| `app/log.py` | 1–80 | Log format: `"%(asctime)s - %(name)s - %(levelname)s - %(process)d - \"%(pathname)s:%(lineno)d\" - %(funcName)s() - %(message_id)s - %(message)s"` (lines 13–15); shortcuts: `.d=debug`, `.i=info`, `.w=warning`, `.e=exception` (lines 74–77); `LOG = _get_logger("SL")` (line 79) |
| `server.py` | 1–80, 355–380 | `rate_limited()` handler (lines 362–372): returns `{"error": "Rate limit exceeded"}` with 429 for API paths; LOG.w on rate limit hit (lines 364–367) |
| `app/errors.py` | 1–131 | `AliasInTrashError` (lines 11–14); custom exception hierarchy |
| `app/dashboard/views/custom_alias.py` | 1–174 | Dashboard variant of alias creation; same `check_suffix_signature()` and `verify_prefix_suffix()` calls; flash-based error messaging |
| `app/api/views/alias_options.py` | 1–154 | v4 and v5 alias options endpoints; returns `can_create` boolean and signed suffix list |
| `app/api/views/new_random_alias.py` | 1–118 | Random alias endpoint; same decorator chain and quota check pattern |
| `app/utils.py` | 1–80 | `convert_to_id()` (lines 50–56): lowercases, unidecodes, removes spaces, truncates to 256 chars |
| `app/constants.py` | 1–2 | `HEADER_ALLOW_API_COOKIES` and `DMARC_RECORD` constants |
| `docs/api.md` | 1–200, 340–440 | Existing API reference; alias creation endpoint docs (lines 381–400); error format (lines 80–92); alias options (lines 340–378) |
| `tests/api/test_new_custom_alias.py` | 1–306 | 10 test cases: `test_v2`, `test_minimal_payload`, `test_full_payload`, `test_custom_domain_alias`, `test_wrongly_formatted_payload`, `test_mailbox_ids_is_not_an_array`, `test_out_of_quota`, `test_cannot_create_alias_in_trash`, `test_too_many_requests`, `test_invalid_alias_2_consecutive_dots` |
| `tests/test_alias_suffixes.py` | 1–153 | 5 test cases for suffix generation and validation |
| `pyproject.toml` | 1–full | Python ^3.10, Flask ^1.1.2, Flask-Limiter ^1.4, itsdangerous (transitive), redis ^4.5.3, SQLAlchemy 1.3.24 |
| `README.md` | 1–80 | Documentation style reference |

**Folders explored:**

| Folder Path | Depth | Key Contents Discovered |
|-------------|-------|------------------------|
| `` (root) | 0 | Project structure: Flask monolith with `app/`, `docs/`, `tests/`, `scripts/` |
| `app/` | 1 | 47 modules including alias_suffix.py, alias_utils.py, models.py, config.py, extensions.py |
| `app/api/` | 2 | base.py (auth), serializer.py, views/ package |
| `app/api/views/` | 3 | 17 view modules including new_custom_alias.py, alias_options.py |
| `docs/` | 1 | 12 markdown files + 1 SVG; no doc generator framework |
| `tests/` | 1 | conftest.py, 35+ test modules, 14 subpackages |
| `tests/api/` | 2 | 17 API test modules including test_new_custom_alias.py |
| `app/dashboard/` | 2 | base.py, views/ with custom_alias.py |

### 0.11.2 Tech Spec Sections Referenced

| Section | Relevance |
|---------|-----------|
| 6.4 Security Architecture | Rate limiting architecture (6.4.5), API authentication flow (6.4.2.2), signed alias suffix token architecture (6.4.1.5) |
| 4.2 Core Email Processing Workflows | Auto-create alias logic referenced in alias_utils.py context |

### 0.11.3 Attachments

No attachments were provided for this project. No Figma URLs or external design references apply to this documentation task.

