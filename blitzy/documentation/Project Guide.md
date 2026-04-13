# Blitzy Project Guide

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a comprehensive operational verification and behavioral walkthrough document for the SimpleLogin open-source email aliasing platform. The deliverable is a single markdown file (`blitzy/documentation/app_2cd6ee777f8c.md`) that answers three questions: how to confirm the platform is working (startup indicators, health endpoints, log messages), the complete new-user product experience (registration → activation → login → dashboard), and what background services are doing during that flow (job runner, event system, email handler, cron jobs, monitoring). All claims are evidence-based with file:line citations from 30+ source files. No source code was modified.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status
    "Completed (AI)" : 27
    "Remaining" : 3
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 30 |
| **Completed Hours (AI)** | 27 |
| **Remaining Hours** | 3 |
| **Completion Percentage** | 90.0% |

**Calculation:** 27 completed hours / (27 + 3 remaining hours) = 27/30 = 90.0% complete

### 1.3 Key Accomplishments

- [x] Created comprehensive 852-line markdown document (`blitzy/documentation/app_2cd6ee777f8c.md`) covering all three AAP-specified questions
- [x] Analyzed 30+ source files to derive evidence-based answers with specific file:line citations
- [x] Documented complete application startup sequence (13 subsections covering Flask init, blueprints, health endpoint, database, Redis, WSGI, logging)
- [x] Traced end-to-end new-user flow across 9 steps (register → User.create() → activation email → activate → login → MFA decision tree → dashboard)
- [x] Documented all 8 background service categories (job_runner, event system, event_listener, email_handler, monitoring, cron jobs, local dev email, domain seeding)
- [x] Verified all code references against actual source files for accuracy
- [x] Applied 2 follow-up fix commits to correct line-number references and handler mappings
- [x] All 639 existing tests pass (100% pass rate) — no regressions introduced
- [x] Zero compilation errors across all Python modules
- [x] Application runtime validated: health endpoint returns `success` with HTTP 200
- [x] Included Quick Reference Commands appendix with copy-pasteable verification commands

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Document line-number references may drift if upstream code is modified | Low — document remains conceptually accurate but specific line references may become stale | Human Developer | Ongoing maintenance |

### 1.5 Access Issues

No access issues identified. The project is a documentation-only deliverable that required only read access to the existing source repository and write access to the `blitzy/documentation/` directory. All required access was available and functional throughout the development process.

### 1.6 Recommended Next Steps

1. **[High]** Human review of `blitzy/documentation/app_2cd6ee777f8c.md` for accuracy — verify a sample of file:line references against current source code
2. **[Medium]** Proofread document for clarity, grammar, and completeness of explanations
3. **[Medium]** Validate Quick Reference Commands appendix by executing commands against a running SimpleLogin instance
4. **[Low]** Consider adding a version/date stamp to the document header for future maintenance tracking

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Source Code Analysis | 8 | Deep analysis of 30+ source files (server.py, app/models.py, auth views, email_handler.py, job_runner.py, monitoring.py, event_listener.py, etc.) to extract evidence-based answers about startup, auth flows, and background services |
| Document Composition | 10 | Writing 852 lines of structured markdown covering 4 main sections: Application Readiness (13 subsections), New-User Flow (9 steps), Background Services (8 subsections), Ephemeral Data Notes, and Quick Reference Appendix |
| Code Reference Verification | 3 | Systematic verification of all file:line citations in the document against actual source code — confirming function names, line numbers, log messages, and code behavior |
| Document Refinement | 2 | Two follow-up fix commits correcting Job.create() line ranges, blueprint URL prefixes, onboarding handler mappings, db.py references, and login.py line numbers |
| Test Execution & Validation | 3 | Running 639 tests across 104 test files (100% pass rate), fixing 5 IPv6-related test failures in test_mail_sender.py, verifying zero compilation errors |
| Runtime Validation | 1 | Flask application startup verification, health endpoint testing (GET /health → success/200), startup log message confirmation |
| **Total Completed** | **27** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human Review — Verify code reference accuracy | 1.5 | Medium |
| Proofreading and formatting polish | 1 | Low |
| Command validation against live instance | 0.5 | Low |
| **Total Remaining** | **3** | |

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| Unit & Integration | pytest 7.x | 639 | 639 | 0 | N/A | All tests across 104 test files in tests/ directory |

**Details:**
- **Test execution time:** 163.52 seconds (2 minutes 43 seconds)
- **Warnings:** 51 non-blocking warnings (SQLAlchemy deprecation, Flask thread context warnings)
- **Fix applied:** IPv6 loopback enabled via `sysctl net.ipv6.conf.lo.disable_ipv6=0` to unblock 5 tests in `tests/test_mail_sender.py` that require IPv6 socket support
- **Skipped:** 0 tests
- **Blocked:** 0 tests
- **Test environment:** Python 3.10.20, PostgreSQL on port 15432 (user: test, db: test), Redis on port 6379, CONFIG=tests/test.env

---

## 4. Runtime Validation & UI Verification

### Application Startup
- ✅ Flask application creates successfully via `create_app()`
- ✅ Startup log messages confirmed:
  - `"load config file /.../tests/test.env"` — config loaded
  - `">>> URL: http://localhost"` — URL configured
  - `">>> init logging <<<"` — logger initialized
  - `"load words file: /.../local_data/test_words.txt"` — word list loaded

### Health Endpoint
- ✅ `GET /health` returns HTTP 200 with body `"success"`

### Database
- ✅ PostgreSQL running on port 15432 (user: test, db: test)
- ✅ All Alembic migrations applied successfully
- ✅ SQLAlchemy engine and scoped session established at module import

### Redis
- ✅ Redis running on port 6379

### Compilation
- ✅ Zero compilation errors across all Python source files:
  - Root: server.py, wsgi.py, email_handler.py, job_runner.py, cron.py, monitoring.py, event_listener.py, init_app.py
  - App package: All files under app/ directory compile cleanly

### Deliverable File
- ✅ `blitzy/documentation/app_2cd6ee777f8c.md` — 852 lines, well-structured markdown
- ✅ All code references verified against actual source files
- ✅ No source code modifications — only the documentation file was added

---

## 5. Compliance & Quality Review

| Requirement | Status | Evidence |
|-------------|--------|----------|
| Create `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | File exists at specified path, 852 lines |
| Document answers Application Readiness question | ✅ Pass | Section 1 with 13 subsections covering all startup indicators |
| Document answers New-User Flow question | ✅ Pass | Section 2 with 9 sequential steps from registration to dashboard |
| Document answers Background Services question | ✅ Pass | Section 3 with 8 subsections covering all background services |
| Evidence-based answers with file:line citations | ✅ Pass | 387 inline code references throughout the document |
| No source code modifications | ✅ Pass | `git diff --name-status` shows only 1 file added (A blitzy/documentation/app_2cd6ee777f8c.md) |
| No assumptions — code as truth | ✅ Pass | All claims reference specific file paths, line numbers, and function names |
| Document placed in `blitzy/documentation/` directory | ✅ Pass | File path: `blitzy/documentation/app_2cd6ee777f8c.md` |
| Document named `app_2cd6ee777f8c.md` | ✅ Pass | Matches source branch naming convention |
| Ephemeral test data policy documented | ✅ Pass | Section 4 covers test data lifecycle, cleanup commands, and important reminders |
| Quick reference commands included | ✅ Pass | Appendix with 9 copy-pasteable verification commands |
| Zero test regressions | ✅ Pass | 639/639 tests pass (100%) |
| Zero compilation errors | ✅ Pass | All Python files compile cleanly |

**Autonomous Validation Fixes Applied:**
1. Corrected Job.create() line ranges and blueprint URL prefixes in documentation (commit `7bb97285`)
2. Corrected onboarding handler mappings, db.py and login.py line references (commit `7cd6f67b`)

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Line-number references become stale after upstream code changes | Technical | Low | Medium | Include function names alongside line numbers for resilience; consider automated reference-checking script | Open — inherent to documentation referencing specific lines |
| Document may not cover edge cases in social login or payment flows | Technical | Low | Low | AAP explicitly scopes these out; document focuses on email-based registration and standard login | Accepted — by design per AAP scope |
| Test environment differs from production | Operational | Low | Low | Document notes `NOT_SEND_EMAIL=true` and `CONFIG=tests/test.env` differences from production | Mitigated — documented in Section 5 of deliverable |
| No automated verification of document accuracy | Technical | Medium | Low | Human review step included in remaining work; sample verification of file:line references recommended | Open — requires human action |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 27
    "Remaining Work" : 3
```

**Summary:** 27 hours of AAP-scoped work completed out of 30 total hours = 90.0% complete. Remaining 3 hours consist of human review, proofreading, and command validation tasks.

---

## 8. Summary & Recommendations

### Achievement Summary

The project is **90.0% complete** (27 of 30 total hours). Blitzy agents successfully delivered the primary AAP deliverable — a comprehensive 852-line markdown document (`blitzy/documentation/app_2cd6ee777f8c.md`) that provides evidence-based answers to three questions about SimpleLogin's runtime behavior. The document was created through deep analysis of 30+ source files, with all code references verified for accuracy. Two refinement commits were applied to correct line-number references. The existing test suite of 639 tests continues to pass at 100%, zero compilation errors were found, and the application runtime was validated (health endpoint returns success/200).

### Remaining Gaps

The remaining 3 hours of work are human-oriented tasks:
- **Document review** (1.5h): A human developer should verify a sample of file:line references against the current source code to confirm accuracy
- **Proofreading** (1h): Final polish for grammar, clarity, and formatting consistency
- **Command validation** (0.5h): Execute Quick Reference Commands appendix against a running instance

### Production Readiness Assessment

This deliverable is **ready for human review and merge**. Since this is a documentation-only change:
- No deployment or infrastructure changes are needed
- No source code was modified, so no regression risk exists
- The document is self-contained and does not affect application behavior
- All 639 tests pass, confirming no unintended side effects

### Recommendations

1. **Merge with confidence** — the single file addition carries zero regression risk
2. **Schedule periodic review** — as the codebase evolves, line-number references should be spot-checked
3. **Consider linking** from the main README.md to this document for discoverability
4. **Extend pattern** — this evidence-based documentation approach could be applied to other subsystems (payment flows, custom domains, PGP encryption) in future iterations

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Python | 3.10+ | Runtime language |
| PostgreSQL | 12+ | Primary database |
| Redis | 6+ | Session store and rate limiting |
| Poetry | 1.x | Python dependency management |
| Node.js | 10.17+ | Frontend asset compilation |
| Git | 2.x | Version control |

### Environment Setup

```bash
# 1. Clone the repository and navigate to it
cd /tmp/blitzy/app/blitzy-4607de44-4354-4352-a5b8-95144667c7e7_cecd63

# 2. Create and activate the Python virtual environment
python3.10 -m venv venv
source venv/bin/activate

# 3. Install Python dependencies via Poetry
pip install poetry
poetry install --no-interaction --no-ansi --no-root

# 4. Set up environment configuration
# For testing (uses tests/test.env automatically):
export CONFIG=tests/test.env

# For local development (copy and customize example.env):
cp example.env .env
# Edit .env with your PostgreSQL credentials, URL, etc.
```

### Database Setup

```bash
# Ensure PostgreSQL is running, then create the database:
createdb -U myuser simplelogin

# Run all migrations:
source venv/bin/activate
export CONFIG=tests/test.env  # or your .env path
flask db upgrade

# (Optional) Seed demo data:
flask fake-data
```

### Dependency Installation Verification

```bash
source venv/bin/activate

# Verify Python version
python -V
# Expected: Python 3.10.x

# Verify key packages
python -c "import flask; print(f'Flask {flask.__version__}')"
python -c "import sqlalchemy; print(f'SQLAlchemy {sqlalchemy.__version__}')"
```

### Application Startup

```bash
# Start the Flask application (development mode)
source venv/bin/activate
export CONFIG=tests/test.env
python server.py

# Expected startup output:
# load config file /.../tests/test.env
# >>> URL: http://localhost
# >>> init logging <<<

# Or start with Gunicorn (production mode):
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

### Verification Steps

```bash
# 1. Verify health endpoint
curl http://localhost:7777/health
# Expected: "success" with HTTP 200

# 2. Run the full test suite
source venv/bin/activate
export CONFIG=tests/test.env
python -m pytest tests/ --tb=short -q
# Expected: 639 passed

# 3. Verify the deliverable document exists
cat blitzy/documentation/app_2cd6ee777f8c.md | wc -l
# Expected: 852

# 4. Verify no source code was modified
git diff --name-status origin/app_2cd6ee777f8c...HEAD
# Expected: A  blitzy/documentation/app_2cd6ee777f8c.md
```

### Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|------------|
| `ModuleNotFoundError: No module named 'arrow'` | Virtual environment not activated | Run `source venv/bin/activate` |
| `KeyError: 'URL'` | Environment not configured | Set `export CONFIG=tests/test.env` or create `.env` |
| IPv6 test failures in `test_mail_sender.py` | IPv6 loopback disabled in container | Run `sysctl -w net.ipv6.conf.lo.disable_ipv6=0` |
| PostgreSQL connection refused | Database not running | Start PostgreSQL: `pg_ctlcluster 15 main start` or use Docker |
| Redis connection refused | Redis not running | Start Redis: `redis-server --daemonize yes` |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `source venv/bin/activate` | Activate Python virtual environment |
| `export CONFIG=tests/test.env` | Set test environment configuration |
| `python server.py` | Start Flask development server |
| `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` | Start production WSGI server |
| `python -m pytest tests/ --tb=short -q` | Run full test suite |
| `flask db upgrade` | Apply database migrations |
| `flask fake-data` | Seed demo data |
| `curl http://localhost:7777/health` | Check application health |
| `python job_runner.py` | Start background job processor |
| `python email_handler.py` | Start SMTP inbound handler |
| `python monitoring.py` | Start metric export loop |
| `python event_listener.py` | Start event sourcing listener |
| `python init_app.py` | Seed alias domains and PGP keys |

### B. Port Reference

| Port | Service | Description |
|------|---------|-------------|
| 7777 | Flask/Gunicorn | Main web application |
| 5432 (or 15432 in test) | PostgreSQL | Primary database |
| 6379 | Redis | Session store and rate limiting |
| 20381 | aiosmtpd | Inbound SMTP email handler |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Deliverable** — operational verification and walkthrough document |
| `server.py` | Flask app factory and development entry point |
| `wsgi.py` | Production WSGI entry point |
| `app/config.py` | Environment variable loading and configuration |
| `app/models.py` | All ORM models (User, Alias, Mailbox, Job, etc.) |
| `app/auth/views/register.py` | Registration endpoint |
| `app/auth/views/activate.py` | Account activation endpoint |
| `app/auth/views/login.py` | Login endpoint |
| `app/auth/views/login_utils.py` | Post-login MFA decision tree |
| `app/dashboard/views/index.py` | Dashboard landing page |
| `email_handler.py` | SMTP inbound mail processor |
| `job_runner.py` | Background job polling loop |
| `cron.py` | Scheduled maintenance functions |
| `crontab.yml` | Cron job schedule definitions |
| `monitoring.py` | Metric export loop |
| `event_listener.py` | Event sourcing CLI |
| `example.env` | Reference environment configuration |
| `tests/test.env` | Test environment configuration |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Python | ^3.10 | pyproject.toml |
| Flask | ^1.1.2 | pyproject.toml |
| SQLAlchemy | 1.3.24 | pyproject.toml |
| PostgreSQL | 12+ | Dockerfile / deployment docs |
| Redis | 4.5.3+ | pyproject.toml (redis client) |
| Gunicorn | ^20.0.4 | pyproject.toml |
| aiosmtpd | ^1.2 | pyproject.toml |
| pytest | ^7.0.0 | pyproject.toml (dev) |
| Node.js | 10.17 | Dockerfile (asset build stage) |

### E. Environment Variable Reference

| Variable | Default/Example | Purpose |
|----------|----------------|---------|
| `URL` | `http://localhost:7777` | Server public URL |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | PostgreSQL connection string |
| `EMAIL_DOMAIN` | `sl.local` | Primary alias domain |
| `NOT_SEND_EMAIL` | `true` | Suppress SMTP delivery (log only) |
| `FLASK_SECRET` | (required) | Flask session encryption key |
| `MEM_STORE_URI` | (optional) | Redis URI for sessions/rate limiting |
| `CONFIG` | (optional) | Path to dotenv configuration file |
| `COLOR_LOG` | (optional) | Enable colored console log output |
| `DISABLE_ONBOARDING` | `true` | Skip onboarding job scheduling |
| `SENTRY_DSN` | (optional) | Sentry error tracking DSN |
| `HCAPTCHA_SECRET` | (optional) | hCaptcha verification secret |

### F. Developer Tools Guide

| Tool | Configuration File | Purpose |
|------|-------------------|---------|
| Black | `pyproject.toml [tool.black]` | Python code formatter (target: py310) |
| Ruff | `pyproject.toml [tool.ruff]` | Fast Python linter |
| Flake8 | `.flake8` | Python style checker |
| Pylint | `.pylintrc` | Python static analysis |
| djLint | `pyproject.toml [tool.djlint]` | Jinja template linter |
| pre-commit | `.pre-commit-config.yaml` | Git pre-commit hooks |
| pytest | `pytest.ci.ini` / `coverage.ini` | Test runner and coverage configuration |
| Alembic | `alembic.ini` | Database migration tool |

### G. Glossary

| Term | Definition |
|------|------------|
| **Alias** | A generated email address (e.g., `abc123@sl.local`) that forwards to a user's real mailbox |
| **Mailbox** | A user's real email address that receives forwarded messages |
| **Activation Code** | A 30-character random string with 1-hour TTL used to verify email ownership during registration |
| **MFA** | Multi-Factor Authentication — supports TOTP (app-based) and FIDO/WebAuthn (hardware key) |
| **Job Runner** | Background process (`job_runner.py`) that polls the `job` table every 10 seconds for pending work |
| **Event Listener** | Process (`event_listener.py`) that uses PostgreSQL LISTEN/NOTIFY for event sourcing |
| **SLDomain** | A SimpleLogin-managed domain available for creating aliases (seeded by `init_app.py`) |
| **Blueprint** | A Flask organizational unit that groups related routes under a URL prefix |
| **NOT_SEND_EMAIL** | Configuration flag that, when true, logs email content without SMTP transmission |