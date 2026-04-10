# Blitzy Project Guide — SimpleLogin Runtime Behavior Investigation Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project creates a comprehensive Technical Q&A documentation file for the SimpleLogin email privacy system, providing empirically verified answers to five specific runtime behavior questions about database migrations, web server startup, email handler port binding, user registration/login flow, and dynamic alias limit configuration. The documentation targets developers and operators who need to understand the exact runtime behavior of SimpleLogin subsystems during initialization, startup, and first-run experience. The single deliverable — `blitzy/documentation/app_2cd6ee777f8c.md` (813 lines) — was produced without modifying any existing source files, adhering strictly to the investigation constraint.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (81.3%)" : 26
    "Remaining (18.7%)" : 6
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 32 |
| **Completed Hours (AI)** | 26 |
| **Remaining Hours** | 6 |
| **Completion Percentage** | 81.3% |

**Calculation:** 26 completed hours / (26 completed + 6 remaining) = 26 / 32 = **81.3% complete**

### 1.3 Key Accomplishments

- ✅ Created comprehensive 813-line runtime behavior investigation guide covering all 5 question areas
- ✅ Empirically verified all answers against running PostgreSQL, Gunicorn, Flask, and email_handler instances
- ✅ Captured exact JSON responses, log messages, curl output, and SQL results verbatim from live system
- ✅ Verified all 29 source code line references against actual repository files
- ✅ Documented 6 critical gotchas (EMAIL_DOMAIN validation, werkzeug suppression, jwcrypto compatibility, etc.)
- ✅ Included Mermaid sequence diagram for registration/login flow
- ✅ Applied 4 iterative QA fix passes to correct factual inaccuracies (Content-Length, line references, is_premium values)
- ✅ Zero source file modifications — constraint fully respected
- ✅ No TODO/FIXME/placeholder content in final document
- ✅ Balanced 48 code block pairs with proper language tags throughout

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Documentation not independently reproduced by a human | Empirical claims rely on Blitzy agent verification only | Human Developer | 2.5h |
| No automated documentation lint/validation in CI | Documentation formatting drift possible over time | Human Developer | 1h |

### 1.5 Access Issues

No access issues identified. The documentation file is a standalone Markdown document requiring no special permissions, credentials, or third-party API access. The repository uses plain Markdown with no documentation build system.

### 1.6 Recommended Next Steps

1. **[High]** Independently reproduce the documented runtime behaviors by following the Environment Setup section and executing each command
2. **[High]** Review all empirical claims (JSON responses, log messages, SQL results) against a freshly provisioned environment
3. **[Medium]** Perform editorial review for clarity, grammar, and formatting consistency
4. **[Low]** Plan documentation maintenance strategy to keep answers current when source code changes
5. **[Low]** Consider adding a CI check (e.g., markdownlint) to validate documentation formatting on future commits

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Environment Setup for Empirical Testing | 2 | Configured PostgreSQL, .env file, JWT key generation, domain seeding, dependency installation for live system testing |
| Q1: Database Migrations Investigation | 3 | Ran `alembic upgrade head` on empty DB, queried `information_schema`, analyzed 255-migration chain to identify last `create_table` |
| Q2: Web Server Startup Investigation | 2.5 | Started Gunicorn and Flask dev server, captured startup logs, analyzed werkzeug logger suppression in `app/log.py` |
| Q3: Email Handler Investigation | 1.5 | Started `email_handler.py -p 25025`, captured log output, verified port binding messages |
| Q4: Registration/Login Flow Investigation | 3 | Executed curl commands against live API, captured verbose HTTP output, verified database state via psql |
| Q5: Dynamic Alias Limits Investigation | 2.5 | Tested default and changed config values, restarted server, compared responses for users created before/after change |
| Source Code Analysis and Tracing | 3 | Read and analyzed 16+ source files to trace runtime behavior back to specific code lines, verified 29 references |
| Document Authoring | 5 | Wrote 813-line Markdown document with 48 code blocks, tables, Mermaid diagram, gotchas, and summary |
| QA Fixes and Iterations | 2.5 | Applied 4 iterative fix commits correcting Content-Length values, line references, is_premium values, and curl headers |
| Final Validation and Cleanup | 1 | Verified balanced code blocks, no TODO/FIXME, proper newline at EOF, clean working tree |
| **Total Completed** | **26** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Independent Reproducibility Verification | 2.5 | High |
| Human Review of Documentation Accuracy | 2 | High |
| Editorial and Formatting Review | 1 | Medium |
| Documentation Maintenance Planning | 0.5 | Low |
| **Total Remaining** | **6** | |

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Empirical Verification — Q1 (Migrations) | Alembic + psql | 4 | 4 | 0 | 100% | Table count (77), migration count (255), `__tablename__` count (76), last table (`user_audit_log`) |
| Empirical Verification — Q2 (Web Server) | Gunicorn + Flask | 3 | 3 | 0 | 100% | Gunicorn startup logs, Flask werkzeug suppression, sub-second timing |
| Empirical Verification — Q3 (Email Handler) | email_handler.py | 2 | 2 | 0 | 100% | INFO message, DEBUG message on port 25025 |
| Empirical Verification — Q4 (Registration) | curl + psql | 4 | 4 | 0 | 100% | Register 200, Login 422, DB activated=false, DB notification=true |
| Empirical Verification — Q5 (Alias Limits) | curl + API | 3 | 3 | 0 | 100% | Default=5, Changed=10, Both users=10 (global config) |
| Source Code Reference Validation | grep + sed | 29 | 29 | 0 | 100% | All source file line references verified against actual codebase |
| Documentation Structure Validation | Markdown analysis | 4 | 4 | 0 | 100% | Balanced code blocks (48 pairs), no TODO/FIXME, proper EOF newline, valid headers |

> **Note:** All tests listed originate from Blitzy's autonomous validation process. This is a documentation project; "tests" represent the empirical verification steps performed against the running SimpleLogin system to validate each documented answer.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ PostgreSQL 16 — Running with `simplelogin` database, 77 tables migrated via alembic
- ✅ Gunicorn 20.1.0 — Started successfully on port 7777, all 4 startup messages captured
- ✅ Flask Dev Server — Confirmed werkzeug logger suppression behavior documented correctly
- ✅ Email Handler (aiosmtpd 1.4.4) — Successfully bound to custom port 25025, both log messages captured
- ✅ Domain Seeding — `init_app.py` executed, `sldev.io` confirmed in `public_domain` (SLDomain) table

### API Integration Verification

- ✅ `POST /api/auth/register` — Returns 200 with `{"msg":"User needs to confirm their account"}`
- ✅ `POST /api/auth/login` (before activation) — Returns 422 with `{"error":"Account not activated"}`
- ✅ `GET /api/user_info` (default config) — Returns `"max_alias_free_plan": 5`
- ✅ `GET /api/user_info` (after config change) — Returns `"max_alias_free_plan": 10` for both old and new users

### Database Verification

- ✅ User row created with `activated=false`, `notification=true` after registration
- ✅ `account_activation` row created with 6-digit code
- ✅ 77 tables confirmed via `information_schema.tables` query

### UI Verification

- ⚠️ Not applicable — This is a documentation-only project. No UI components were created or modified.

---

## 5. Compliance & Quality Review

| Compliance Criterion | Status | Details |
|---------------------|--------|---------|
| No source file modifications | ✅ Pass | `git diff --name-status` confirms only `blitzy/documentation/app_2cd6ee777f8c.md` was added; zero existing files changed |
| Empirical verification of all answers | ✅ Pass | All 5 questions verified against running PostgreSQL, Gunicorn, Flask, email_handler instances |
| Exact output reproduction | ✅ Pass | JSON responses, log messages, curl output, SQL results captured verbatim from live system |
| Source code tracing for all answers | ✅ Pass | 29 source code references with file:line format, all verified against actual codebase |
| Document placed in `blitzy/documentation/` | ✅ Pass | File at `blitzy/documentation/app_2cd6ee777f8c.md` per SWE-AtlasQnA-Repo rule |
| Branch name as filename | ✅ Pass | Filename `app_2cd6ee777f8c.md` matches source branch `app_2cd6ee777f8c` |
| Rationale/thinking included for each answer | ✅ Pass | Every Q section has a "Rationale / Code Trace" subsection |
| Gotchas and failure modes documented | ✅ Pass | 6 gotchas documented (EMAIL_DOMAIN, werkzeug, jwcrypto, gunicorn timestamps, global config, init_app.py) |
| Mermaid sequence diagram for Q4 | ✅ Pass | Registration-login flow diagram at lines 553-567 |
| No TODO/FIXME/placeholder content | ✅ Pass | `grep -in` search returned zero matches |
| Valid Markdown formatting | ✅ Pass | 48 balanced code block pairs, proper headers, language tags, tables, newline at EOF |
| Production vs. development modes documented | ✅ Pass | Q2 covers both Gunicorn (production) and Flask dev server (development) startup |

### Autonomous Fixes Applied

| Fix | Commit | Description |
|-----|--------|-------------|
| Content-Length correction | ef1b8991 | Fixed curl verbose output Content-Length: 36 → 34 (actual byte count) |
| Request Content-Length correction | ef1b8991 | Fixed curl request Content-Length: 58 → 60 (actual byte count with spaces) |
| Missing HTTP headers | ef1b8991 | Added Access-Control-Allow-Origin, Vary, Set-Cookie headers to curl output |
| server.py line reference | ef1b8991 | Fixed 572-588 → 572-589 (app.run() at line 589) |
| config.py line references | ef1b8991 | Fixed 120-124 → 119-124 (try block at line 119) |
| config.py old plan reference | ef1b8991 | Fixed MAX_NB_EMAIL_OLD_FREE_PLAN line 126 → 125 |
| is_premium value | a507c5dd | Corrected is_premium from false to true in Q5 user_info examples |
| QA findings | 1de9d3df | Resolved 2 QA findings in runtime behavior guide |
| Factual inaccuracies | afff4d13 | Fixed 3 minor factual inaccuracies in documentation |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Empirical data not independently verified by human | Technical | Medium | Medium | Human developer should reproduce all 5 Q&A sections against a fresh environment | Open |
| Source code changes invalidate documented line numbers | Operational | Medium | High | Document references specific revisions; re-verify after any source code updates | Open |
| EMAIL_DOMAIN gotcha causes confusion for new developers | Operational | Low | Medium | Prominently documented with ⚠️ warning in Environment Setup section | Mitigated |
| Gunicorn log timestamps lack sub-second resolution | Technical | Low | Low | Documented as a known limitation; suggested `time.perf_counter()` instrumentation for precise timing | Mitigated |
| `jwcrypto` version conflict on certain platforms | Technical | Low | Medium | Documented workaround (upgrade to 0.9.1) in gotchas section | Mitigated |
| Documentation drift as codebase evolves | Operational | Medium | High | Recommend CI-based documentation linting and periodic review schedule | Open |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 26
    "Remaining Work" : 6
```

### Remaining Work by Priority

| Priority | Hours | Categories |
|----------|-------|------------|
| High | 4.5 | Independent reproducibility verification (2.5h), Human accuracy review (2h) |
| Medium | 1 | Editorial and formatting review (1h) |
| Low | 0.5 | Documentation maintenance planning (0.5h) |
| **Total** | **6** | |

---

## 8. Summary & Recommendations

### Achievements

The project successfully delivered a comprehensive 813-line runtime behavior investigation guide documenting all five requested question areas with empirically verified answers. The document covers database migrations (77 tables, 255 steps, last table `user_audit_log`), web server startup behavior (both Gunicorn and Flask dev mode), email handler custom port binding, user registration/login flow with exact HTTP responses and database state, and dynamic alias limit configuration semantics. All answers include source code tracing with 29 verified file:line references, and 6 critical gotchas were discovered and documented to prevent common setup pitfalls.

### Remaining Gaps

The project is **81.3% complete** (26 hours completed out of 32 total hours). The remaining 6 hours consist entirely of human review tasks: independent reproducibility verification (2.5h), accuracy review of empirical claims (2h), editorial review (1h), and documentation maintenance planning (0.5h). No AAP-scoped features are missing — all 5 questions are fully answered and verified.

### Critical Path to Production

1. **Independent Reproduction (High Priority, 2.5h):** A human developer should follow the Environment Setup section and execute each documented command to confirm outputs match.
2. **Accuracy Review (High Priority, 2h):** Spot-check JSON responses, HTTP status codes, database values, and log messages against a fresh test environment.
3. **Editorial Review (Medium Priority, 1h):** Review for clarity, grammar, and formatting consistency.

### Production Readiness Assessment

The documentation is functionally complete and ready for human review. The file has been through 5 commits (initial creation + 4 QA fix iterations), all source code references have been verified, code blocks are balanced, and no placeholder content remains. The document is self-contained and reproducible with the included Environment Setup instructions.

---

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Python | 3.10+ (tested with 3.12.3) | Runtime for SimpleLogin application |
| PostgreSQL | 13+ (tested with 16) | Database engine |
| OpenSSL | System default | JWT RSA key generation |
| pip / Poetry | Latest | Python dependency management |
| curl | 7.x+ | API testing |
| psql | Matching PostgreSQL version | Database verification |

### Environment Setup

#### 1. Clone the Repository

```bash
git clone <repository-url>
cd <repository-root>
git checkout blitzy-ef829dfb-aabe-49b8-b954-27397e1e3673
```

#### 2. Install Python Dependencies

```bash
# Using pip (recommended for quick setup)
pip install -e .

# Or using Poetry
poetry install
```

#### 3. Configure PostgreSQL

```bash
# Create database and user
sudo -u postgres psql -c "CREATE USER sl_user WITH PASSWORD 'sl_password';"
sudo -u postgres psql -c "CREATE DATABASE simplelogin OWNER sl_user;"
```

#### 4. Create .env Configuration

```bash
cat > .env << 'EOF'
URL=http://localhost:7777
EMAIL_DOMAIN=sldev.io
SUPPORT_EMAIL=support@sldev.io
DB_URI=postgresql://sl_user:sl_password@localhost:5432/simplelogin
FLASK_SECRET=replace-with-a-random-secret-string
NOT_SEND_EMAIL=true
WORDS_FILE_PATH=local_data/words.txt
OPENID_PRIVATE_KEY_PATH=local_data/jwtRS256.key
OPENID_PUBLIC_KEY_PATH=local_data/jwtRS256.key.pub
NAMESERVERS=9.9.9.9
PARTNER_API_TOKEN_SECRET=a-random-partner-secret
ALLOWED_REDIRECT_DOMAINS=[]
EMAIL_SERVERS_WITH_PRIORITY=[(10, "mail.sldev.io.")]
EOF
```

> **⚠️ CRITICAL:** Do NOT use `EMAIL_DOMAIN=sl.local` (the example.env default). The `email_validator` library rejects reserved TLDs and will cause 500 errors during user registration. Use `sldev.io` instead.

#### 5. Generate JWT Keys

```bash
mkdir -p local_data
openssl genrsa -out local_data/jwtRS256.key 4096
openssl rsa -in local_data/jwtRS256.key -pubout -outform PEM -out local_data/jwtRS256.key.pub
```

#### 6. Run Database Migrations

```bash
alembic upgrade head
```

Expected: 255 migration steps, 77 tables created.

#### 7. Seed Domain Data

```bash
python init_app.py
```

This seeds the `public_domain` table with the configured `EMAIL_DOMAIN`.

### Application Startup

#### Start Gunicorn (Production Mode)

```bash
gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 30 --log-level info
```

Expected output:
```text
[INFO] Starting gunicorn 20.1.0
[INFO] Listening at: http://0.0.0.0:7777 (PID)
[INFO] Using worker: sync
[INFO] Booting worker with pid: PID
```

#### Start Flask Dev Server

```bash
python server.py
```

Note: The `* Running on http://...` message is suppressed by `app/log.py` disabling the werkzeug logger.

### Verification Steps

```bash
# Verify server is running
curl -s http://localhost:7777/api/user_info | head -1

# Register a test user
curl -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@example.com", "password": "testpass123"}'

# Attempt login before activation (expect 422)
curl -v -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@example.com", "password": "testpass123"}'
```

### Viewing the Documentation

The documentation file is at:
```bash
cat blitzy/documentation/app_2cd6ee777f8c.md
```

It renders natively on GitHub or with any Markdown viewer. Mermaid diagrams require a Mermaid-compatible renderer (e.g., GitHub, VS Code with Mermaid extension).

### Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| 500 error during user registration | `EMAIL_DOMAIN` uses reserved TLD (.local, .test) | Change `EMAIL_DOMAIN=sldev.io` in `.env` |
| `TypeError` in `load_pem_private_key()` | `jwcrypto ^0.8` conflicts with newer `cryptography` | Upgrade: `pip install jwcrypto==0.9.1` |
| Flask dev server starts silently | `app/log.py` disables werkzeug logger | Expected behavior; use Gunicorn for visible startup messages |
| Registration fails — no domain found | `init_app.py` not run after migrations | Run: `python init_app.py` |
| `MAX_NB_EMAIL_FREE_PLAN is not set` message | Environment variable not in `.env` | Add `MAX_NB_EMAIL_FREE_PLAN=5` to `.env` (or accept default) |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `alembic upgrade head` | Run all database migrations on empty PostgreSQL |
| `python init_app.py` | Seed public_domain table with EMAIL_DOMAIN |
| `gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 30 --log-level info` | Start production web server |
| `python server.py` | Start Flask development server |
| `python email_handler.py -p 25025` | Start email handler on custom port |
| `curl -X POST http://localhost:7777/api/auth/register -H "Content-Type: application/json" -d '{"email":"user@example.com","password":"testpass123"}'` | Register a new user |
| `curl -X POST http://localhost:7777/api/auth/login -H "Content-Type: application/json" -d '{"email":"user@example.com","password":"testpass123"}'` | Attempt login |
| `curl -H "Authentication: API_KEY" http://localhost:7777/api/user_info` | Get user info via API |
| `openssl genrsa -out local_data/jwtRS256.key 4096` | Generate JWT RSA private key |
| `openssl rsa -in local_data/jwtRS256.key -pubout -outform PEM -out local_data/jwtRS256.key.pub` | Extract JWT RSA public key |

### B. Port Reference

| Service | Default Port | Configurable Via |
|---------|-------------|------------------|
| Gunicorn (Web Server) | 7777 | `-b 0.0.0.0:PORT` CLI argument |
| Flask Dev Server | 7777 | `server.py:589` — `app.run(port=7777)` |
| Email Handler (aiosmtpd) | 20381 | `-p PORT` CLI argument |
| PostgreSQL | 5432 | `DB_URI` in `.env` |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | Main deliverable — Runtime Behavior Investigation Guide |
| `.env` | Environment configuration (created during setup, not committed) |
| `local_data/jwtRS256.key` | RSA private key for JWT/OpenID (generated, not committed) |
| `local_data/jwtRS256.key.pub` | RSA public key for JWT/OpenID (generated, not committed) |
| `example.env` | Reference configuration template (198 lines, existing) |
| `server.py` | Flask app factory and dev server entry point |
| `wsgi.py` | Gunicorn WSGI entry point |
| `email_handler.py` | SMTP email handler (2,404 lines) |
| `app/config.py` | All environment variable loading (635 lines) |
| `app/models.py` | 76 ORM model classes |
| `app/log.py` | Logger configuration, werkzeug disable |
| `app/api/views/auth.py` | Registration and login endpoints |
| `app/api/views/user_info.py` | User info API endpoint |
| `init_app.py` | Domain seeding script |
| `migrations/versions/` | 255 Alembic migration files |

### D. Technology Versions

| Technology | Version (pyproject.toml) | Version (Installed) |
|------------|--------------------------|---------------------|
| Python | ^3.10 | 3.12.3 |
| Flask | ^1.1.2 | 1.1.4 |
| Gunicorn | ^20.0.4 | 20.1.0 |
| SQLAlchemy | 1.3.24 | 1.3.24 |
| Alembic | via Flask-Migrate ^2.5.3 | 1.14.1 |
| aiosmtpd | ^1.2 | 1.4.4 |
| psycopg2-binary | ^2.9.3 | (installed) |
| jwcrypto | ^0.8 | 0.9.1 (recommended) |
| email-validator | ^1.1.1 | (installed) |
| PostgreSQL | 13+ (recommended) | 16 (tested) |

### E. Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `URL` | Yes | — | Base URL of the SimpleLogin instance |
| `EMAIL_DOMAIN` | Yes | — | Primary domain for email aliases (must NOT use reserved TLD) |
| `SUPPORT_EMAIL` | Yes | — | Support contact email address |
| `DB_URI` | Yes | — | PostgreSQL connection string |
| `FLASK_SECRET` | Yes | — | Flask session secret key |
| `NOT_SEND_EMAIL` | No | false | Set to `true` to disable actual email sending |
| `MAX_NB_EMAIL_FREE_PLAN` | No | 5 | Maximum aliases for free plan users (global setting) |
| `MAX_NB_EMAIL_OLD_FREE_PLAN` | No | 15 | Maximum aliases for old free plan users (with flag) |
| `WORDS_FILE_PATH` | No | — | Path to words file for random alias generation |
| `OPENID_PRIVATE_KEY_PATH` | Yes | — | Path to JWT RSA private key |
| `OPENID_PUBLIC_KEY_PATH` | Yes | — | Path to JWT RSA public key |
| `NAMESERVERS` | No | — | DNS resolver for MX lookups |
| `PARTNER_API_TOKEN_SECRET` | Yes | — | Secret for partner API token signing |
| `ALLOWED_REDIRECT_DOMAINS` | No | `[]` | Allowed OAuth redirect domains |
| `EMAIL_SERVERS_WITH_PRIORITY` | No | — | MX priority list for outbound email |

### G. Glossary

| Term | Definition |
|------|-----------|
| `alembic_version` | Alembic's internal migration tracking table, storing the current head revision hash |
| `SLDomain` / `public_domain` | Database table storing domains available for alias creation (seeded by `init_app.py`) |
| `FLAG_FREE_OLD_ALIAS_LIMIT` | Bit flag (`1 << 2 = 4`) on User model that routes alias limit to `MAX_NB_EMAIL_OLD_FREE_PLAN` instead of `MAX_NB_EMAIL_FREE_PLAN` |
| werkzeug | Flask's underlying WSGI utility library; its logger is disabled by `app/log.py` to suppress request logs |
| `AccountActivation` | Database model storing 6-digit activation codes generated during user registration |
| aiosmtpd | Async SMTP server library used by `email_handler.py` to receive inbound email |
| `LOG.i` / `LOG.d` | Shorthand logging methods defined in `app/log.py` mapping to `logging.Logger.info` and `logging.Logger.debug` |