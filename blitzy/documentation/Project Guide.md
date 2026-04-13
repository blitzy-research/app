# Blitzy Project Guide — SimpleLogin First-Run Initialization & Runtime Behavior Investigation

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive empirical investigation of the SimpleLogin email privacy and alias system's first-run initialization and runtime behavior. The investigation answers five specific questions about database migration output, web server startup logging, email handler port binding, registration/login flow behavior, and dynamic alias limit configuration — all verified through actual runtime execution against a freshly provisioned environment. The primary deliverable is a 687-line markdown document (`blitzy/documentation/app_2cd6ee777f8c.md`) containing exact log output, curl command results, SQL query results, and code path analysis. No source code files were modified per the explicit project constraint.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (88.9%)" : 24
    "Remaining (11.1%)" : 3
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | **27** |
| **Completed Hours (AI)** | **24** |
| **Remaining Hours** | **3** |
| **Completion Percentage** | **88.9%** (24 / 27 × 100) |

### 1.3 Key Accomplishments

- ✅ Provisioned full runtime environment: PostgreSQL 16, Redis, Python 3.10 venv with 152 packages
- ✅ Executed 255 Alembic migrations against empty database — verified 77 tables (76 app + alembic_version)
- ✅ Identified last table created by migration order: `user_audit_log` (revision `7d7b84779837`)
- ✅ Discovered and documented Werkzeug logger suppression — `* Running on` message hidden by `app/log.py`
- ✅ Captured exact email handler startup log messages for port 25025 binding
- ✅ Performed full registration → pre-activation login test with HTTP 422 rejection verified
- ✅ Verified database column values: `activated=False`, `notification=True` for new users
- ✅ Confirmed `MAX_NB_EMAIL_FREE_PLAN` change affects ALL users (existing and new) dynamically
- ✅ Delivered 687-line comprehensive investigation document with empirical evidence
- ✅ Zero source files modified — adhered to explicit no-modification constraint
- ✅ All 9 key Python source files compile without errors

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Documentation accuracy review pending | Low — all findings empirically verified but human confirmation recommended | Human Developer | 1–2 hours |
| Test database contains investigation artifacts | Minimal — test users and data from investigation remain in local DB | Human Developer | 0.5 hours |

### 1.5 Access Issues

No access issues identified. All required infrastructure (PostgreSQL, Redis, Python packages) was provisioned and configured successfully within the local environment.

### 1.6 Recommended Next Steps

1. **[High]** Review the investigation document (`blitzy/documentation/app_2cd6ee777f8c.md`) for accuracy and completeness against the original questions
2. **[High]** Merge this PR to preserve the investigation findings on the target branch
3. **[Medium]** Cross-reference findings with the specific SimpleLogin version/commit to ensure version accuracy
4. **[Low]** Clean up test users and data from the local PostgreSQL database
5. **[Low]** Remove the temporary `.env` file if not needed for future experimentation

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Environment Infrastructure Setup | 3 | Provisioned PostgreSQL 16, Redis, Python 3.10 venv with 152 packages, configured `.env` with 18 required variables, generated OpenID keys and word list files |
| Database Migration Analysis | 4 | Executed 255 Alembic migrations, counted 77 tables via SQL, analyzed migration history for create/drop/rename operations, identified `user_audit_log` as last created table |
| Web Server Startup Investigation | 2.5 | Captured Flask dev server and Gunicorn startup logs, discovered Werkzeug logger suppression root cause in `app/log.py`, measured 1,635ms startup timing |
| Email Handler Port Verification | 1.5 | Launched `email_handler.py` with port 25025, captured INFO and DEBUG log messages, traced code path through argparse and aiosmtpd Controller |
| Registration & Login Flow Testing | 3 | Executed registration and pre-activation login via curl, captured full HTTP output, verified database column values via SQL, traced code paths through auth.py and models.py |
| Dynamic Alias Limit Investigation | 3.5 | Tested baseline with MAX_NB_EMAIL_FREE_PLAN=5, changed to 10 with restart, verified both existing and new users affected, documented FLAG_FREE_OLD_ALIAS_LIMIT exception |
| Documentation Authoring & Refinement | 5 | Authored 687-line investigation document with 6 sections, code blocks, tables, sequence diagrams, and 3 rounds of accuracy refinement across 4 commits |
| Validation & Quality Assurance | 1.5 | Compiled all 9 key Python source files, cross-validated empirical findings against source code, verified git status cleanliness |
| **Total Completed** | **24** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human Review of Documentation Accuracy | 1.5 | High |
| PR Review, Approval & Merge | 1 | High |
| Environment Cleanup (test data, temp .env) | 0.5 | Low |
| **Total Remaining** | **3** | |

### 2.3 Hours Verification

- Completed Hours: **24**
- Remaining Hours: **3**
- Total Project Hours: 24 + 3 = **27**
- Completion: 24 / 27 × 100 = **88.9%**

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Compilation Checks | py_compile | 9 | 9 | 0 | 100% | All key Python source files: server.py, wsgi.py, email_handler.py, app/config.py, app/models.py, app/db.py, app/log.py, app/api/views/auth.py, app/api/views/user_info.py |
| Runtime — Database Migrations | Alembic | 1 | 1 | 0 | N/A | 255 migrations applied against empty DB; 77 tables verified via SQL count |
| Runtime — Web Server (Gunicorn) | curl + gunicorn | 1 | 1 | 0 | N/A | Health endpoint `/health` returns `success` |
| Runtime — Web Server (Flask dev) | python server.py | 1 | 1 | 0 | N/A | `* Debug mode: on` confirmed; Werkzeug suppression documented |
| Runtime — Email Handler | aiosmtpd | 1 | 1 | 0 | N/A | Port 25025 binding confirmed with two log messages |
| Runtime — Registration API | curl | 1 | 1 | 0 | N/A | HTTP 200 with `{"msg":"User needs to confirm their account"}` |
| Runtime — Pre-Activation Login | curl | 1 | 1 | 0 | N/A | HTTP 422 with `{"error":"Account not activated"}` |
| Runtime — Dynamic Alias Limits | curl + config change | 1 | 1 | 0 | N/A | Config 5→10 affects all users; verified for existing and new users |
| **Totals** | | **16** | **16** | **0** | | All autonomous validation tests passed |

> **Note:** This is a documentation/investigation task with no new application code written. The existing pytest suite (`tests/`) was not in scope per the AAP. All tests above originate from Blitzy's autonomous runtime validation process.

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ **PostgreSQL 16** — Running on `localhost:5432`, accepting connections, 77 tables in `simplelogin` database
- ✅ **Redis** — Running on `localhost:6379`, responding to PING
- ✅ **Alembic Migrations** — All 255 migrations applied successfully; head revision `32f25cbf12f6`
- ✅ **Gunicorn Web Server** — Starts on port 7777 with 2 workers; `/health` returns `success`
- ✅ **Flask Dev Server** — Starts with `python server.py`; `* Debug mode: on` confirmed
- ✅ **Email Handler** — Binds to custom port 25025 via `python email_handler.py -p 25025`
- ✅ **Registration API** — `POST /api/auth/register` creates user and returns HTTP 200
- ✅ **Login Rejection** — `POST /api/auth/login` for inactive user returns HTTP 422
- ✅ **Config Reload** — `MAX_NB_EMAIL_FREE_PLAN` change with server restart affects all users

### API Endpoint Verification

- ✅ `POST /api/auth/register` — HTTP 200, `{"msg":"User needs to confirm their account"}`
- ✅ `POST /api/auth/login` (pre-activation) — HTTP 422, `{"error":"Account not activated"}`
- ✅ `GET /api/user_info` — Returns `max_alias_free_plan` dynamically from config
- ✅ `GET /health` — Returns `success` (plaintext)

### UI Verification

- ⚠ **Not applicable** — This investigation task does not involve UI modifications. The existing SimpleLogin web UI was not in scope.

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence |
|----------------|--------|----------|
| Database Migration Analysis — table count | ✅ Pass | SQL query returns 77 tables (76 app + alembic_version); matches 76 `__tablename__` in models.py |
| Database Migration Analysis — last table | ✅ Pass | `user_audit_log` identified via migration file chronological analysis (revision `7d7b84779837`) |
| Web Server Startup — readiness message | ✅ Pass | `* Debug mode: on` captured; Werkzeug suppression documented with root cause |
| Web Server Startup — elapsed timing | ✅ Pass | ~1,635ms from first log to last visible startup message |
| Email Handler — port 25025 binding | ✅ Pass | Two log messages captured: INFO `Listen for port 25025`, DEBUG `Start mail controller 0.0.0.0 25025` |
| Registration — exact JSON response | ✅ Pass | HTTP 200, `{"msg":"User needs to confirm their account"}` with full curl output |
| Pre-activation Login — exact error | ✅ Pass | HTTP 422, `{"error":"Account not activated"}` with full curl output |
| Database Column Values — activated | ✅ Pass | `activated=f` (False) confirmed via `SELECT` query |
| Database Column Values — notification | ✅ Pass | `notification=t` (True) confirmed via `SELECT` query |
| Dynamic Alias Limits — config change effect | ✅ Pass | Verified: affects ALL users (existing + new) after restart |
| Dynamic Alias Limits — code path documented | ✅ Pass | `User.max_alias_for_free_account()` → `config.MAX_NB_EMAIL_FREE_PLAN` traced and documented |
| No source files modified | ✅ Pass | `git diff --name-status` confirms only `blitzy/documentation/app_2cd6ee777f8c.md` added |
| Empirical runtime evidence (not static analysis) | ✅ Pass | All answers include actual log output, curl responses, and SQL results from live execution |
| Documentation in `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | 687-line comprehensive document created and committed |

### Autonomous Validation Fixes Applied

| Fix | Commit | Description |
|-----|--------|-------------|
| Documentation accuracy fix 1 | `3387ea65` | Addressed 3 code review findings in documentation formatting |
| Documentation accuracy fix 2 | `f4001801` | Fixed 3 documentation accuracy issues (precise line references, exact output formatting) |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Investigation findings may drift from future code changes | Technical | Low | Medium | Document references specific commit `2cd6ee77` and file line numbers; re-run investigation after major updates | Documented |
| Temporary `.env` file contains database credentials | Security | Low | Low | `.env` uses local-only credentials (`myuser:mypassword@localhost`); not committed to repository; clean up after investigation | Mitigated |
| Test data (users, aliases) persists in local database | Operational | Low | High | Run `DROP DATABASE simplelogin; CREATE DATABASE simplelogin;` to reset, or delete specific test records | Documented |
| PostgreSQL version mismatch (16 used vs 13 in docs) | Technical | Low | Low | Core migration behavior is identical across PG 13–16; no version-specific features used | Accepted |
| Redis not strictly required for investigation | Integration | Low | Low | Redis was provisioned for completeness (rate limiting, sessions) but investigation results do not depend on it | Mitigated |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 24
    "Remaining Work" : 3
```

### Remaining Work Distribution

| Category | Hours | Proportion |
|----------|-------|------------|
| Human Review of Documentation Accuracy | 1.5 | 50% |
| PR Review, Approval & Merge | 1 | 33.3% |
| Environment Cleanup | 0.5 | 16.7% |
| **Total Remaining** | **3** | **100%** |

---

## 8. Summary & Recommendations

### Achievement Summary

The project is **88.9% complete** (24 hours completed out of 27 total hours). All five investigation questions defined in the AAP have been fully answered with empirical runtime evidence. The primary deliverable — a 687-line comprehensive investigation document — has been authored, refined through 3 rounds of accuracy corrections, and committed across 4 commits. All 16 autonomous validation tests passed with zero failures.

### Key Technical Findings Delivered

1. **Migration Count:** 76 application tables + 1 alembic_version = 77 total from 255 migrations
2. **Last Table Created:** `user_audit_log` (revision `7d7b84779837`, October 2024)
3. **Werkzeug Suppression:** `app/log.py` disables the werkzeug logger, hiding `* Running on` — the last visible message is `* Debug mode: on`
4. **Email Handler Port:** Confirmed `Listen for port 25025` (INFO) and `Start mail controller 0.0.0.0 25025` (DEBUG)
5. **Pre-Activation Login:** Returns HTTP 422 with `{"error":"Account not activated"}`; DB shows `activated=f, notification=t`
6. **Dynamic Alias Limits:** `MAX_NB_EMAIL_FREE_PLAN` change affects ALL users at request time — not stored per-user

### Remaining Gaps

The remaining 3 hours (11.1%) consist exclusively of human review and operational cleanup tasks:
- Human review of the investigation document for accuracy (1.5h)
- PR review, approval, and merge (1h)
- Cleanup of test data and temporary configuration (0.5h)

### Production Readiness Assessment

This is a documentation-only deliverable with no code changes. The investigation document is complete and ready for human review. There are no blocking issues, no compilation failures, and no failing tests. The document can be merged as-is after review.

### Recommendations

1. **Merge promptly** — The investigation document is self-contained and fully validated
2. **Version-pin the findings** — Note that findings are specific to commit `2cd6ee77` on branch `app_2cd6ee777f8c`
3. **Re-run if code changes** — Key behaviors (Werkzeug suppression, alias limit computation) may change in future versions
4. **Clean up local environment** — Remove test users and reset the database if the environment will be reused

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Python | 3.10+ | Application runtime |
| PostgreSQL | 13+ (16 tested) | Primary database |
| Redis | 6+ | Session store, rate limiter backend |
| pip | Latest | Python package manager |
| curl | Any | API testing |
| git | Any | Version control |

### Environment Setup

#### 1. Clone and Navigate to Repository

```bash
cd /tmp/blitzy/app/blitzy-af7f232a-a093-42ee-b590-6516afbec4de_487a15
```

#### 2. Create Python Virtual Environment

```bash
python3.10 -m venv venv
source venv/bin/activate
```

#### 3. Create `.env` Configuration File

```bash
cat > .env << 'EOF'
URL=http://localhost:7777
NOT_SEND_EMAIL=true
EMAIL_DOMAIN=sl.local
SUPPORT_EMAIL=support@sl.local
SUPPORT_NAME=Son from SimpleLogin
EMAIL_SERVERS_WITH_PRIORITY=[(10, "email.hostname.")]
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
FLASK_SECRET=secret
OPENID_PRIVATE_KEY_PATH=local_data/jwtRS256.key
OPENID_PUBLIC_KEY_PATH=local_data/jwtRS256.key.pub
WORDS_FILE_PATH=local_data/test_words.txt
DISABLE_ONBOARDING=true
LOCAL_FILE_UPLOAD=true
NAMESERVERS=1.1.1.1
PARTNER_API_TOKEN_SECRET=changeme
ALLOWED_REDIRECT_DOMAINS=[]
MAX_NB_EMAIL_FREE_PLAN=5
MEM_STORE_URI=redis://localhost:6379/0
EOF
```

#### 4. Generate OpenID Key Files (if not present)

```bash
openssl genrsa -out local_data/jwtRS256.key 2048
openssl rsa -in local_data/jwtRS256.key -pubout -out local_data/jwtRS256.key.pub
```

### Dependency Installation

```bash
source venv/bin/activate
pip install -r requirements.txt
# Or if using pyproject.toml directly:
pip install .
```

**Expected output:** 152 packages installed with no broken requirements.

**Verify installation:**

```bash
pip check
```

### Database Setup

#### 1. Start PostgreSQL (if not running)

```bash
sudo service postgresql start
```

#### 2. Create Database and User

```bash
sudo -u postgres psql -c "CREATE USER myuser WITH PASSWORD 'mypassword';"
sudo -u postgres psql -c "CREATE DATABASE simplelogin OWNER myuser;"
```

#### 3. Run Alembic Migrations

```bash
source venv/bin/activate
alembic upgrade head
```

**Expected output:** 255 migration upgrade messages, ending with revision `32f25cbf12f6`.

#### 4. Verify Table Count

```bash
psql -U myuser -d simplelogin -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
```

**Expected output:** `77` (76 app tables + 1 alembic_version)

### Application Startup

#### Option A: Gunicorn (Production-like)

```bash
source venv/bin/activate
gunicorn wsgi:app -b 0.0.0.0:7777 --workers 2 &
```

#### Option B: Flask Dev Server

```bash
source venv/bin/activate
python server.py &
```

#### Start Email Handler (Optional)

```bash
source venv/bin/activate
python email_handler.py -p 25025 &
```

### Verification Steps

#### Health Check

```bash
curl -s http://localhost:7777/health
# Expected: success
```

#### Test Registration

```bash
curl -s -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "testpass123"}'
# Expected: {"msg":"User needs to confirm their account"}
```

#### Verify Database

```bash
psql -U myuser -d simplelogin -c "SELECT email, activated, notification FROM users LIMIT 5;"
```

### Troubleshooting

| Issue | Solution |
|-------|----------|
| `ModuleNotFoundError` on import | Ensure virtual environment is activated: `source venv/bin/activate` |
| `psycopg2.OperationalError: connection refused` | Start PostgreSQL: `sudo service postgresql start` |
| `ConnectionError: Redis` | Start Redis: `redis-server --daemonize yes` |
| `FileNotFoundError: jwtRS256.key` | Generate keys: `openssl genrsa -out local_data/jwtRS256.key 2048` |
| Werkzeug `* Running on` not visible | Expected behavior — suppressed by `app/log.py` line 71 |
| Port 7777 already in use | Kill existing process: `lsof -ti:7777 | xargs kill` |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `alembic upgrade head` | Apply all database migrations |
| `python server.py` | Start Flask development server on port 7777 |
| `gunicorn wsgi:app -b 0.0.0.0:7777 --workers 2` | Start Gunicorn production server |
| `python email_handler.py -p 25025` | Start SMTP email handler on port 25025 |
| `python job_runner.py` | Start background job runner |
| `python shell.py` | Start interactive admin shell |
| `curl -s http://localhost:7777/health` | Health check endpoint |
| `psql -U myuser -d simplelogin` | Connect to PostgreSQL database |

### B. Port Reference

| Port | Service | Protocol |
|------|---------|----------|
| 7777 | Flask/Gunicorn web server | HTTP |
| 25025 | Email handler (custom, via `-p` flag) | SMTP |
| 20381 | Email handler (default) | SMTP |
| 5432 | PostgreSQL | TCP |
| 6379 | Redis | TCP |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | Primary investigation deliverable (687 lines) |
| `.env` | Runtime configuration (18 environment variables) |
| `server.py` | Flask web server bootstrap (599 lines) |
| `email_handler.py` | SMTP handler with port argument (2,404 lines) |
| `app/config.py` | Environment variable loading (666 lines) |
| `app/models.py` | 76 SQLAlchemy ORM models (3,843 lines) |
| `app/api/views/auth.py` | Registration/login endpoints (392 lines) |
| `app/api/views/user_info.py` | User info endpoint (163 lines) |
| `app/log.py` | Logging configuration with Werkzeug suppression |
| `local_data/jwtRS256.key` | OpenID RSA private key |
| `local_data/jwtRS256.key.pub` | OpenID RSA public key |
| `local_data/test_words.txt` | Word list for alias generation |
| `migrations/versions/` | 256 Alembic migration revision files |
| `example.env` | Reference for all available environment variables |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Python | 3.10+ (3.10.20 in venv) | `pyproject.toml` |
| Flask | 1.1.2 | `pip show flask` |
| Gunicorn | 20.0.4 | `pip show gunicorn` |
| SQLAlchemy | 1.3.24 | `pip show sqlalchemy` |
| Alembic | 1.4.3 | `pip show alembic` |
| psycopg2-binary | 2.9.3 | `pip show psycopg2-binary` |
| aiosmtpd | 1.4.2 | `pip show aiosmtpd` |
| Redis (Python client) | 4.6.0 | `pip show redis` |
| PostgreSQL | 16 (13+ compatible) | `pg_isready` |
| Redis (server) | 6+ | `redis-cli info server` |

### E. Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DB_URI` | Yes | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | PostgreSQL connection string |
| `FLASK_SECRET` | Yes | `secret` | Flask session signing key |
| `URL` | Yes | `http://localhost:7777` | Public URL for the server |
| `EMAIL_DOMAIN` | Yes | `sl.local` | Domain for alias creation |
| `SUPPORT_EMAIL` | Yes | `support@sl.local` | Support contact email |
| `EMAIL_SERVERS_WITH_PRIORITY` | Yes | N/A | MX server configuration |
| `NOT_SEND_EMAIL` | No | `false` | Suppress email delivery (set `true` for local dev) |
| `MAX_NB_EMAIL_FREE_PLAN` | No | `5` | Free plan alias limit |
| `WORDS_FILE_PATH` | No | `local_data/words.txt` | Path to word list for aliases |
| `OPENID_PRIVATE_KEY_PATH` | No | `local_data/jwtRS256.key` | RSA private key for JWT signing |
| `OPENID_PUBLIC_KEY_PATH` | No | `local_data/jwtRS256.key.pub` | RSA public key for JWKS |
| `DISABLE_ONBOARDING` | No | Not set | Disable onboarding emails |
| `LOCAL_FILE_UPLOAD` | No | Not set | Use local filesystem for uploads |
| `PARTNER_API_TOKEN_SECRET` | Yes | `changeme` | Partner API authentication secret |
| `MEM_STORE_URI` | No | Not set | Redis connection string for sessions/rate limiting |

### G. Glossary

| Term | Definition |
|------|------------|
| AAP | Agent Action Plan — the project requirements specification |
| Alembic | Database migration tool for SQLAlchemy |
| aiosmtpd | Asynchronous SMTP server library for Python |
| `MAX_NB_EMAIL_FREE_PLAN` | Configuration value controlling the maximum number of aliases on the free plan |
| `FLAG_FREE_OLD_ALIAS_LIMIT` | Per-user flag that overrides alias limit to use the legacy higher value (15) |
| Werkzeug | Python WSGI utility library used by Flask for the development server |
| SimpleLogin | Open-source email aliasing and privacy service |