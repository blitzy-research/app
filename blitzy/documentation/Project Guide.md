# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project creates a comprehensive technical investigation document for the SimpleLogin open-source email aliasing platform. The deliverable is a single Markdown file (`blitzy/documentation/app_2cd6ee777f8c.md`) that answers eight empirical questions about SimpleLogin's runtime behavior — covering API authentication mechanics, session management internals, email header forwarding, alias token expiration, API key usage tracking, and failed login diagnostics. All findings are derived from deep code-path analysis of 17 source files across the repository, with exact file:line citations, Mermaid diagrams, and structured tables. No existing source code is modified. The target audience is developers and security engineers who need to understand SimpleLogin's internal behavioral patterns beyond what the existing API documentation covers.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (31h)" : 31
    "Remaining (6h)" : 6
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | **37** |
| **Completed Hours (AI)** | **31** |
| **Remaining Hours** | **6** |
| **Completion Percentage** | **83.8%** |

**Calculation:** 31 completed hours / (31 + 6) total hours = 83.8% complete.

### 1.3 Key Accomplishments

- ✅ Created comprehensive 1,147-line Q&A investigation document answering all 8 empirical questions
- ✅ Performed deep code-path analysis of 17 source files across authentication, session management, email forwarding, and token management modules
- ✅ Included 49 source citations with exact file:line references for every finding
- ✅ Created 5 Mermaid diagrams illustrating authentication pipeline, sudo mode flow, session lifecycle, header filtering, and token signing
- ✅ Produced 44 Python code snippets with syntax highlighting
- ✅ Built 96+ structured table rows comparing behaviors across scenarios
- ✅ Added 8 Rationale sections with complete reasoning chains from code to conclusion
- ✅ Verified all 17 referenced source files exist and citations are accurate
- ✅ Maintained zero modifications to existing source code (documentation-only)
- ✅ Cleaned up all temporary analysis artifacts — working tree is clean
- ✅ Committed across 3 progressive commits with descriptive messages

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Runtime behavior verification pending | Findings are code-path analysis predictions, not observed runtime behavior; edge cases may differ in actual execution | Human Developer | 3 hours |
| Mermaid diagram rendering untested | Diagrams validated syntactically but not rendered in target platform (GitHub/GitLab) | Human Developer | 0.5 hours |

### 1.5 Access Issues

No access issues identified. This is a documentation-only task that requires only read access to the source repository. All source files referenced in the document are publicly available in the SimpleLogin open-source repository.

### 1.6 Recommended Next Steps

1. **[High]** Human review of all 8 technical answers for accuracy — verify code-path analysis predictions match actual runtime behavior, especially Q2 (sudo mode AttributeError) and Q4 (session ID non-regeneration)
2. **[High]** Runtime verification with full stack (PostgreSQL + Redis + Flask) — spin up the SimpleLogin development environment and confirm each finding against actual HTTP responses and session data
3. **[Medium]** Verify Mermaid diagrams render correctly in the target platform (GitHub, GitLab, or documentation site)
4. **[Low]** Consider filing upstream issues for documented behavioral quirks (g.api_key=None on sudo endpoints, session fixation, API key plaintext storage)
5. **[Low]** Monitor for source code changes that may invalidate file:line citations; update document accordingly

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Repository analysis & documentation planning | 2 | Analyzed repository structure, identified 17 key source files, mapped documentation gaps, designed document structure with 10 sections |
| Deep code analysis of 17 source files | 8 | Read and traced code paths in `app/api/base.py`, `app/session.py`, `email_handler.py`, `app/alias_suffix.py`, `app/models.py`, `app/auth/views/login.py`, `app/api/views/auth.py`, `app/extensions.py`, `server.py`, and 8 additional supporting files |
| Q1: API Session-Based Authentication | 1.5 | Traced `authorize_request()` function through API-key and session-fallback paths; documented HTTP responses, `g.api_key = None` behavior, and commented-out cookie gate |
| Q2: Privileged Operation Access | 1.5 | Traced `require_api_sudo()` → `check_sudo_mode_is_active()` with None API key; identified AttributeError → HTTP 500 path; documented non-standard HTTP 440 |
| Q3: Redis Session Data Structure | 2 | Analyzed `RedisSessionStore`, pickle serialization, HMAC signing, Redis key format, session cookie format, TTL policies, and deserialized session keys |
| Q4: Session ID Regeneration | 1.5 | Traced `open_session()` → `login_user()` → `save_session()` lifecycle; documented that session ID is NOT regenerated during login; analyzed `purge_session()` |
| Q5: Email Header Forwarding | 2 | Analyzed `headers_to_keep` allowlist, `delete_all_headers_except()`, Reply-To special handling with reverse-alias; built comprehensive header survival table |
| Q6: Alias Token Expiration | 1 | Traced `TimestampSigner` with `max_age=600`, `check_suffix_signature()`, and HTTP 412 response on expiry |
| Q7: API Key Usage Statistics | 1 | Traced `api_key.last_used` and `api_key.times` update mechanics, `Session.commit()`, and ApiKey model fields |
| Q8: Failed Login Diagnostics | 1.5 | Traced web login (flash + HTTP 200) and API login (JSON + HTTP 400) paths; documented rate limiting differences and LoginEvent analytics |
| 5 Mermaid diagram creation | 2 | Created authentication pipeline flowchart, sudo mode access flow, session lifecycle sequence diagram, email header filtering pipeline, alias token signing sequence |
| Summary & architectural observations | 1 | Wrote key findings table, 6 architectural observations, and configuration constants reference table |
| QA review & fixes (3 commits) | 2 | Addressed code review findings: added SESSION_COOKIE_HTTPONLY, CORS security context, CSRF protection analysis, and API key plaintext storage note |
| Source citation cross-verification | 1.5 | Verified all 49 source citations against actual source files; confirmed line numbers, function signatures, and quoted strings match |
| Markdown structure validation | 0.5 | Validated heading hierarchy, code fence pairing (100 markers, 50 pairs), table formatting, and trailing whitespace |
| **Total Completed** | **31** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human review of technical accuracy | 2 | High |
| Runtime behavior verification (full stack testing) | 3 | High |
| Mermaid diagram rendering verification | 0.5 | Medium |
| Minor editorial polish & publishing | 0.5 | Low |
| **Total Remaining** | **6** | |

**Integrity Check:** Section 2.1 total (31h) + Section 2.2 total (6h) = 37h = Total Project Hours in Section 1.2 ✅

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|--------------|-----------|-------------|--------|--------|------------|-------|
| Document Structure Validation | Custom Python script | 5 | 5 | 0 | 100% | Validated heading hierarchy (H1: 9, H2: 9, H3: 52), code fence pairing (50 pairs), and Markdown structure |
| Source File Existence Verification | Python os.path.exists | 17 | 17 | 0 | 100% | All 17 referenced source files confirmed to exist with correct line counts |
| Source Citation Cross-Reference | Manual + grep verification | 49 | 49 | 0 | 100% | All 49 `Source: file:line` citations verified against actual source code |
| Code Block Syntax Validation | Regex counting | 50 | 50 | 0 | 100% | 100 triple-backtick markers form 50 properly paired code blocks (44 Python + 5 Mermaid + 1 generic) |
| Deliverable Completeness Check | Manual AAP mapping | 8 | 8 | 0 | 100% | All 8 investigative questions answered with code-path traces, answers, and rationale sections |
| No-Source-Modification Constraint | git diff verification | 1 | 1 | 0 | 100% | `git diff --stat -- ':!blitzy/'` confirms zero modifications to existing files |

**Notes:** This is a documentation-only project. No unit tests, integration tests, or runtime tests were executed because no source code was written or modified. The validation suite above represents documentation-specific quality checks performed by Blitzy's autonomous validation system.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ Git working tree is clean — no uncommitted changes
- ✅ Document file created at `blitzy/documentation/app_2cd6ee777f8c.md` (1,147 lines)
- ✅ 3 commits on branch `blitzy-861f39b1-45fc-46f3-b2c9-50be35b854f8`
- ✅ Only 1 file added to the repository (no deletions or modifications)

### Documentation Verification

- ✅ All 8 Q sections present with complete code-path traces
- ✅ All 8 Rationale subsections present with reasoning chains
- ✅ 5 Mermaid diagrams embedded (authentication pipeline, sudo flow, session lifecycle, header filtering, token signing)
- ✅ 49 source citations in `Source: file.py:line` format
- ✅ 96+ table rows across structured comparison tables
- ✅ Introduction section with methodology description
- ✅ Summary section with key findings table, architectural observations, and configuration constants

### Constraint Verification

- ✅ Zero source code files modified (verified via `git diff --stat -- ':!blitzy/'`)
- ✅ Zero temporary scripts remaining (verified via `find` and `git status`)
- ✅ All findings labeled as "code-path analysis" (not observed runtime behavior)
- ✅ Document placed in `blitzy/documentation/` directory per AAP specification
- ✅ Named `app_2cd6ee777f8c.md` per branch naming convention

### API Integration Verification

- ⚠️ **Not applicable** — This is a documentation-only project; no API endpoints were created, modified, or tested at runtime

### UI Verification

- ⚠️ **Pending** — Mermaid diagram rendering has not been verified in the target platform (GitHub/GitLab); diagrams are syntactically valid but visual rendering should be confirmed by human review

---

## 5. Compliance & Quality Review

| Compliance Area | AAP Requirement | Status | Evidence |
|----------------|-----------------|--------|----------|
| Document creation | Create `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | File exists, 1,147 lines, committed |
| Q1: Session-based API auth | Trace `authorize_request()` through session fallback path | ✅ Pass | Lines 15–134 trace complete code path with 7 source citations |
| Q2: Sudo with browser session | Trace `require_api_sudo()` → `check_sudo_mode_is_active()` with None | ✅ Pass | Lines 137–272 identify AttributeError → HTTP 500 path |
| Q3: Redis session data | Document Redis key, pickle serialization, session keys, cookie signing | ✅ Pass | Lines 275–423 cover all 5 required aspects (key format, serialization, cookie, keys, TTL) |
| Q4: Session ID regeneration | Trace `open_session()` → `login_user()` → `save_session()` | ✅ Pass | Lines 426–576 trace full lifecycle, conclude ID NOT regenerated |
| Q5: Email header forwarding | Document `headers_to_keep` allowlist and header survival | ✅ Pass | Lines 579–750 include 18-row header table with survival status |
| Q6: Alias token expiration | Identify `max_age=600` in `check_suffix_signature()` | ✅ Pass | Lines 753–861 document 600s (10 min) window with HTTP 412 |
| Q7: API key stats | Document `last_used` and `times` update mechanics | ✅ Pass | Lines 864–947 trace update logic with field detail table |
| Q8: Failed login diagnostics | Document web and API failure paths | ✅ Pass | Lines 950–1098 include comparison table with 6 scenarios |
| Mermaid diagrams | Minimum 5 diagrams for complex flows | ✅ Pass | 5 diagrams: auth pipeline, sudo flow, session lifecycle, header filtering, token signing |
| Source citations | File:line format for every finding | ✅ Pass | 49 citations, all verified against source files |
| Rationale sections | Reasoning chain for every answer | ✅ Pass | 8 rationale sections (one per question) |
| No source modifications | Zero changes to existing repository files | ✅ Pass | `git diff` confirms only `blitzy/documentation/app_2cd6ee777f8c.md` added |
| No temporary scripts | All analysis artifacts cleaned up | ✅ Pass | `git status` shows clean working tree |
| Code-path analysis labeling | Distinguish code predictions from runtime observations | ✅ Pass | Introduction states "code-path analysis"; labeled throughout document |
| Evidence-based analysis | Exact status codes, error strings, data formats | ✅ Pass | All HTTP codes (200, 400, 401, 403, 412, 422, 440, 500) cited with source lines |

### Fixes Applied During Validation

| Fix | Commit | Description |
|-----|--------|-------------|
| Code review finding #1 | `35f44395` | Addressed 3 code review findings in documentation |
| QA finding #1 — SESSION_COOKIE_HTTPONLY | `974a8fad` | Added Flask default HttpOnly cookie note to Q3 configuration table |
| QA finding #2 — CORS context | `974a8fad` | Added CORS wildcard security context to Q1 answer (prevents cross-origin session abuse) |
| QA finding #3 — CSRF context | `974a8fad` | Added CSRF protection analysis for API endpoints accessed via session cookies |
| QA finding #4 — API key plaintext | `974a8fad` | Added security note about unhashed API key storage in Q7 |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Code-path analysis predictions may not match actual runtime behavior | Technical | Medium | Low | Runtime verification with full stack (PostgreSQL + Redis + Flask) recommended before publishing | Open |
| Source file line numbers may shift with future code changes | Technical | Low | Medium | All citations use stable identifiers (function names + line numbers); update document after any source changes | Open |
| Mermaid diagrams may not render in all Markdown viewers | Technical | Low | Medium | Test rendering in target platform (GitHub, GitLab, or documentation site); provide fallback descriptions | Open |
| Document may be treated as authoritative without runtime verification disclaimer | Operational | Medium | Medium | Introduction clearly states "code-path analysis" methodology; all findings labeled accordingly | Mitigated |
| Documented security observations (session fixation, pickle deserialization, API key plaintext) may require upstream action | Security | Medium | Low | Observations are documented for awareness; filing upstream issues is recommended but out of scope for this task | Open |
| No automated link-checking for source citations | Technical | Low | Low | All 49 citations manually verified; consider adding CI-based citation checker if document is maintained long-term | Open |

---

## 7. Visual Project Status

### Project Hours Breakdown

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 31
    "Remaining Work" : 6
```

**Integrity Check:** Completed (31h) + Remaining (6h) = 37h total = Section 1.2 Total Hours ✅

### Remaining Work by Priority

| Priority | Hours | Categories |
|----------|-------|------------|
| High | 5 | Human accuracy review (2h), runtime verification (3h) |
| Medium | 0.5 | Mermaid rendering verification (0.5h) |
| Low | 0.5 | Editorial polish & publishing (0.5h) |
| **Total** | **6** | |

---

## 8. Summary & Recommendations

### Achievement Summary

The project has delivered a comprehensive 1,147-line technical investigation document that fully answers all 8 empirical questions about SimpleLogin's runtime behavior as defined in the Agent Action Plan. The document covers API authentication mechanics (session fallback, sudo mode edge cases), session management internals (Redis data format, cookie signing, TTL policies, ID regeneration behavior), email header forwarding (allowlist-based stripping), alias token expiration (600-second window), API key usage tracking (last_used, times counters), and failed login diagnostics (web vs. API response differences). Every finding is supported by exact source code citations across 17 analyzed files.

### Completion Assessment

The project is **83.8% complete** (31 hours completed out of 37 total hours). All AAP-specified deliverables have been created and committed. The remaining 6 hours consist of human review and runtime verification tasks that require either human judgment or infrastructure not available in the autonomous environment.

### Critical Path to Production

1. **Human technical review** (2h) — A developer familiar with SimpleLogin should review the 8 answers for accuracy, especially Q2 (sudo mode behavior with session auth) and Q4 (session fixation implications)
2. **Runtime verification** (3h) — Spin up the full SimpleLogin stack and test each of the 8 documented scenarios to confirm code-path predictions match actual behavior
3. **Rendering verification** (0.5h) — Confirm Mermaid diagrams render correctly in the deployment target

### Production Readiness Assessment

| Criterion | Status | Notes |
|-----------|--------|-------|
| Document completeness | ✅ Ready | All 8 questions answered, all diagrams present, all citations verified |
| Technical accuracy | ⚠️ Needs Review | Code-path analysis is thorough but runtime verification recommended |
| Formatting quality | ✅ Ready | Valid Markdown, matched code fences, clean heading hierarchy |
| Constraint compliance | ✅ Ready | Zero source modifications, no temporary artifacts, proper labeling |
| Publishing readiness | ⚠️ Needs Review | Mermaid rendering in target platform should be verified |

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Python | 3.10+ | SimpleLogin runtime (per `pyproject.toml` target) |
| Git | 2.x+ | Repository management |
| Markdown viewer | Any | Document viewing (VS Code, GitHub, GitLab) |
| Mermaid renderer | Any | Diagram viewing (GitHub native, Mermaid Live Editor, VS Code extension) |

### Environment Setup

This is a documentation-only project. No application environment setup is required to view or edit the document. However, if you wish to verify the source code references or run the full SimpleLogin stack for runtime verification, follow these steps:

**1. Clone the repository and switch to the feature branch:**

```bash
git clone <repository-url>
cd <repository-name>
git checkout blitzy-861f39b1-45fc-46f3-b2c9-50be35b854f8
```

**2. View the document:**

```bash
# Open in any text editor or Markdown viewer
cat blitzy/documentation/app_2cd6ee777f8c.md

# Or use VS Code with Markdown preview
code blitzy/documentation/app_2cd6ee777f8c.md
```

**3. Verify source citations (optional):**

```bash
# Example: Verify Q1 citation at app/api/base.py:17
sed -n '17p' app/api/base.py
# Expected: api_code = request.headers.get("Authentication")

# Example: Verify Q6 citation at app/alias_suffix.py:40
sed -n '40p' app/alias_suffix.py
# Expected: return signer.unsign(signed_suffix, max_age=600).decode()
```

### Running the Full SimpleLogin Stack (for Runtime Verification)

If you want to verify the documented runtime behaviors against the actual system:

**1. Install dependencies:**

```bash
# Install Poetry (if not installed)
pip install poetry

# Install project dependencies
poetry install
```

**2. Set up environment variables:**

```bash
cp example.env .env
# Edit .env and set at minimum:
# FLASK_SECRET=<random-secret>
# DB_URI=postgresql://user:pass@localhost:5432/simplelogin
# MEM_STORE_URI=redis://localhost:6379
```

**3. Start required services:**

```bash
# PostgreSQL and Redis must be running
# Using Docker:
docker run -d --name sl-postgres -e POSTGRES_DB=simplelogin -e POSTGRES_USER=user -e POSTGRES_PASSWORD=pass -p 5432:5432 postgres:13
docker run -d --name sl-redis -p 6379:6379 redis:7
```

**4. Initialize the database:**

```bash
flask db upgrade
python init_app.py
```

**5. Start the application:**

```bash
python server.py
# Server starts on http://localhost:7777
```

**6. Verify health endpoint:**

```bash
curl -s http://localhost:7777/health
# Expected: success
```

### Troubleshooting

| Issue | Resolution |
|-------|-----------|
| Mermaid diagrams not rendering | Use GitHub web UI, install VS Code Mermaid extension, or paste into mermaid.live |
| Source citation line numbers don't match | Source code may have changed since document creation; use function name search as fallback |
| `poetry install` fails | Ensure Python 3.10+ is installed; try `poetry install --no-root` |
| Redis connection refused | Ensure Redis is running on port 6379; check `MEM_STORE_URI` in `.env` |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `cat blitzy/documentation/app_2cd6ee777f8c.md` | View the investigation document |
| `git diff origin/app_2cd6ee777f8c...HEAD --stat` | See all changes on this branch |
| `git log origin/app_2cd6ee777f8c...HEAD --oneline` | View commit history |
| `sed -n '<line>p' <file>` | Verify a specific source citation |
| `grep -n '<function>' <file>` | Find function definitions for citation verification |
| `wc -l blitzy/documentation/app_2cd6ee777f8c.md` | Count document lines (expected: 1147) |

### B. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Deliverable** — The investigation document (1,147 lines) |
| `app/api/base.py` | API authentication pipeline (`authorize_request()`, `require_api_sudo()`) |
| `app/session.py` | Redis-backed session store (`RedisSessionStore`, pickle serialization) |
| `app/auth/views/login.py` | Web login form handler |
| `app/api/views/auth.py` | API login endpoint |
| `email_handler.py` | Email forwarding with header stripping (`handle_forward()`) |
| `app/alias_suffix.py` | Alias token signing/verification (`TimestampSigner`, `max_age=600`) |
| `app/models.py` | Database models including `ApiKey` (`last_used`, `times`) |
| `app/extensions.py` | Flask-Login configuration (`session_protection = "strong"`) |
| `server.py` | Flask app bootstrap, error handlers, session lifetime configuration |
| `app/config.py` | Configuration constants (`SESSION_COOKIE_NAME`, `CUSTOM_ALIAS_SECRET`) |

### C. Technology Versions

| Technology | Version | Source |
|-----------|---------|--------|
| Python | ^3.10 | `pyproject.toml` (target), `CONTRIBUTING.md` |
| Flask | ^1.1.2 | `pyproject.toml` |
| Flask-Login | ^0.5.0 | `pyproject.toml` |
| itsdangerous | Transitive (via Flask) | Used for session signing and alias token signing |
| Redis | ^4.5.3 | `pyproject.toml` |
| SQLAlchemy | 1.3.24 | `pyproject.toml` |
| Arrow | ^0.16.0 | `pyproject.toml` |
| aiosmtpd | ^1.2 | `pyproject.toml` (SMTP inbound handler) |

### D. Environment Variable Reference

| Variable | Purpose | Example |
|----------|---------|---------|
| `FLASK_SECRET` | Master secret key for session signing and token derivation | `<random-string>` |
| `DB_URI` | PostgreSQL connection string | `postgresql://user:pass@localhost:5432/simplelogin` |
| `MEM_STORE_URI` | Redis connection string for session storage | `redis://localhost:6379` |
| `SESSION_COOKIE_NAME` | Session cookie name (hardcoded as `slapp`) | `slapp` |
| `DISABLE_RATE_LIMIT` | Bypass rate limiting (development only) | `true` |

### E. Glossary

| Term | Definition |
|------|-----------|
| Code-path analysis | Methodology of tracing execution paths through source code to predict runtime behavior without running the application |
| `authorize_request()` | Core function in `app/api/base.py` that authenticates API requests via API key or Flask-Login session |
| `require_api_sudo()` | Decorator requiring elevated privileges (sudo mode) for sensitive API operations |
| `RedisSessionStore` | Custom Flask session interface in `app/session.py` that stores session data in Redis using pickle serialization |
| `TimestampSigner` | `itsdangerous` class that signs values with embedded timestamps, enabling time-limited token validation |
| `headers_to_keep` | Allowlist in `email_handler.py` defining which email headers survive the forwarding process |
| Reverse alias | A dynamically generated email address that maps replies back through SimpleLogin to the original sender |
| HTTP 440 | Non-standard status code (Microsoft IIS "Login Timeout") used by SimpleLogin for sudo mode expiration |
| Session fixation | Security vulnerability where an attacker sets a session ID before authentication, which persists after login |