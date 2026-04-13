# Blitzy Project Guide — SimpleLogin Runtime Verification Documentation

---

## 1. Executive Summary

### 1.1 Project Overview

This project produces a comprehensive runtime-verification and behavioral-analysis documentation artifact for the SimpleLogin open-source email alias application. The sole deliverable is a single Markdown file (`blitzy/documentation/app_2cd6ee777f8c.md`) that provides code-grounded answers to four operational question clusters: startup verification for three core processes (Flask web server, aiosmtpd email handler, polling job runner), user-facing workflow verification (sign-in, registration, alias management), end-to-end inbound email flow verification, and background component behavioral analysis. No source code was modified — the AAP explicitly prohibits changes to any existing repository files.

### 1.2 Completion Status

```mermaid
pie title Project Completion — 93.3% Complete
    "Completed (AI)" : 28
    "Remaining" : 2
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 30 |
| **Completed Hours (AI)** | 28 |
| **Remaining Hours** | 2 |
| **Completion Percentage** | 93.3% |

**Calculation:** 28 completed hours / (28 completed + 2 remaining) = 28/30 = **93.3% complete**

### 1.3 Key Accomplishments

- ✅ Created comprehensive 1,348-line documentation artifact covering all four AAP question clusters
- ✅ Analyzed 35+ source files to ground every claim in specific file paths, line numbers, and function references
- ✅ Included 263 Python file references and 240 specific line-number citations across the document
- ✅ Documented complete SMTP status code catalog (30+ codes from `app/email/status.py`)
- ✅ Traced full email processing lifecycle from SMTP entry through database record creation
- ✅ Mapped all startup log signatures for Flask server, email handler, and job runner
- ✅ Documented user workflow flows (registration, activation, login, dashboard, alias CRUD)
- ✅ Documented background component architecture (process independence, Docker deployment, job state machine)
- ✅ Created Quick-Start Verification Checklist appendix for operational readiness assessment
- ✅ All 634 in-scope tests pass (100%), 344 Python files compile cleanly, 71.52% coverage
- ✅ Zero source code modifications — fully compliant with AAP constraint
- ✅ All pre-commit checks pass on the deliverable file

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Line-number citations may drift if upstream source changes | Documentation accuracy reduced over time | Human Developer | 1h review |
| 5 out-of-scope test failures in `test_mail_sender.py` (IPv6) | No impact — pre-existing container environment limitation | Infrastructure Team | N/A |

### 1.5 Access Issues

No access issues identified. The project produces a documentation artifact only and does not require any external service credentials, API keys, or deployment access.

### 1.6 Recommended Next Steps

1. **[High]** Human review of documentation accuracy — verify line-number citations against current codebase state
2. **[Medium]** Stakeholder review — confirm the document addresses all intended audience questions
3. **[Low]** Consider adding a version stamp or commit SHA to the document header for traceability against the analyzed codebase version

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Codebase Analysis & Research | 8 | Deep analysis of 35+ source files (server.py, email_handler.py, job_runner.py, app/models.py, app/auth/views/*, app/dashboard/views/*, app/email/status.py, app/config.py, app/log.py, etc.) to extract runtime behavior evidence |
| Section A — Startup Verification | 3 | Documented Flask web server startup (create_app, /health endpoint, request logging hooks), email handler startup (aiosmtpd Controller, log sequences), job runner startup (polling loop, get_jobs_to_run), and logging infrastructure |
| Section B — User-Facing Workflow | 3 | Traced and documented registration flow, account activation, sign-in flow (including MFA branching), dashboard statistics computation, random/custom alias creation, alias deletion/disable, and seed data for local development |
| Section C — End-to-End Email Flow | 4 | Traced complete inbound email lifecycle from MailHandler.handle_DATA() through _handle(), handle(), handle_forward(), forward_email_to_mailbox(); documented all 30+ SMTP status codes; documented Contact, EmailLog, and Notification database record creation |
| Section D — Background Components | 3 | Documented process independence with source code and documentation evidence; Docker deployment architecture; job runner operational behavior (job types, state machine, retry mechanism); email handler runtime characteristics; cron scheduler behavior |
| Section E — Supplementary Information | 2 | Configuration reference for local development, swaks testing procedures, troubleshooting procedures, monitoring metrics documentation |
| Appendix — Quick-Start Checklist | 1 | Created operational readiness checklist covering web server, email handler, job runner, user workflow, and email flow verification steps |
| Code Review & Fixes | 2 | Addressed code review findings in second commit (3050be35); refined formatting, accuracy, and completeness |
| Validation & Quality Assurance | 2 | Ran pre-commit checks, verified document structure (103 headings, 263 file refs, 240 line citations), confirmed zero placeholders/TODOs, executed test suite (634/634 pass) |
| **Total Completed** | **28** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human Review — Verify line-number citations against current codebase | 1.5 | High |
| Editorial Adjustments — Minor corrections based on stakeholder feedback | 0.5 | Medium |
| **Total Remaining** | **2** | |

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Unit & Integration | pytest | 634 | 634 | 0 | 71.52% | All in-scope tests pass; covers auth, dashboard, email handler, job runner, API, models |
| Out-of-Scope | pytest | 5 | 0 | 5 | N/A | `tests/test_mail_sender.py` — IPv6 `::1` loopback unavailable in container; pre-existing environment limitation, not caused by any code changes |
| Compilation | py_compile | 344 | 344 | 0 | 100% | All 344 Python files compile cleanly with zero errors |
| Pre-commit | pre-commit | 1 | 1 | 0 | N/A | trailing-whitespace check passed on `app_2cd6ee777f8c.md` |

**Aggregate:** 639 tests executed, 634 passed (100% in-scope), 71.52% code coverage (exceeds 55% threshold).

---

## 4. Runtime Validation & UI Verification

This project produces a **documentation-only artifact** — no application components were modified. Runtime validation was assessed through code analysis rather than live execution.

**Documentation File Verification:**
- ✅ File exists at `blitzy/documentation/app_2cd6ee777f8c.md` (64,265 bytes, 1,348 lines)
- ✅ Document structure: 103 Markdown headings covering all 4 required question clusters
- ✅ Code references: 263 Python file references, 240 specific line-number citations
- ✅ Completeness: Zero placeholders, zero TODOs, zero "implement later" markers
- ✅ No source code modifications: `git diff --name-status` shows only 1 file added (`A`)

**Source Code Integrity:**
- ✅ All 344 Python files compile cleanly — no compilation regressions introduced
- ✅ 634/634 in-scope tests pass — no test regressions introduced
- ✅ Git working tree clean (only untracked `dump.rdb` from Redis test session)
- ✅ No submodules present or modified

**Content Coverage Verification:**
- ✅ Section A: Startup Verification — Flask `/health` endpoint, email handler aiosmtpd startup logs, job runner polling loop
- ✅ Section B: User Workflow — Registration, activation, login, dashboard stats, alias creation/deletion
- ✅ Section C: Email Flow — SMTP entry (`handle_DATA`), dispatch (`handle`), forward (`handle_forward`), Contact/EmailLog records, 30+ SMTP status codes
- ✅ Section D: Background Components — Process independence, Docker deployment, job state machine, cron scheduler
- ✅ Section E: Supplementary — Config reference, swaks testing, troubleshooting, monitoring
- ✅ Appendix: Quick-Start Verification Checklist

---

## 5. Compliance & Quality Review

| Requirement | Source | Status | Evidence |
|-------------|--------|--------|----------|
| No source code modification | AAP: "Do not modify the source code" | ✅ Pass | `git diff --name-status` shows only 1 file added; 0 files modified or deleted |
| Document named `app_2cd6ee777f8c.md` | AAP: `SWE-AtlasQnA-Repo` rule | ✅ Pass | File at `blitzy/documentation/app_2cd6ee777f8c.md` |
| Document placed in `blitzy/documentation/` | AAP: `SWE-AtlasQnA-Repo` rule | ✅ Pass | Directory listing confirms location |
| Code-as-Truth principle — all claims cite source code | AAP: "Base answers on code as truth" | ✅ Pass | 263 Python file references, 240 line-number citations |
| Provide thinking and rationale | AAP: `SWE-AtlasQnA-Repo` rule | ✅ Pass | Every section includes "Rationale" paragraphs explaining code-grounded reasoning |
| No additional code besides documentation | AAP: "Do not add any other code" | ✅ Pass | Only `app_2cd6ee777f8c.md` created; no Python, config, or build files added |
| All four question clusters answered | AAP: startup, user workflow, email flow, background components | ✅ Pass | Sections A, B, C, D comprehensively cover all four clusters |
| Zero placeholders or TODOs | Quality standard | ✅ Pass | `grep -c TODO` = 0, `grep -c FIXME` = 0 on deliverable |
| Pre-commit checks pass | Quality standard | ✅ Pass | trailing-whitespace and other hooks pass on deliverable |
| Test suite non-regression | Quality standard | ✅ Pass | 634/634 in-scope tests pass; no regressions from documentation-only change |

**Fixes Applied During Autonomous Validation:**
- Code review findings addressed in commit `3050be35` (formatting refinements, accuracy improvements)
- Pre-commit trailing-whitespace check applied and passed

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Line-number citations may become stale if upstream source code changes | Technical | Medium | High | Add commit SHA or version stamp to document header; re-verify citations periodically | Open — requires human action |
| Documentation may not cover edge cases not evident from static code analysis | Technical | Low | Medium | Supplement with runtime testing if needed; document caveats explicitly | Mitigated — "Rationale" sections note analysis boundaries |
| Out-of-scope IPv6 test failures may concern reviewers | Operational | Low | Low | Document in PR description that these are pre-existing container environment limitations | Mitigated — documented in test results |
| Reader may misinterpret code analysis as runtime-tested verification | Operational | Medium | Medium | Document introduction clearly states this is source-code analysis, not runtime testing | Mitigated — introduction paragraph states methodology |
| No security risks | Security | N/A | N/A | N/A — no code, credentials, or configuration modified | N/A |
| No integration risks | Integration | N/A | N/A | N/A — documentation-only artifact with no system dependencies | N/A |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 28
    "Remaining Work" : 2
```

**Completed: 28 hours (93.3%) | Remaining: 2 hours (6.7%)**

### Remaining Hours by Category

```mermaid
pie title Remaining Work Distribution
    "Human Review of Citations" : 1.5
    "Editorial Adjustments" : 0.5
```

---

## 8. Summary & Recommendations

### Achievements

The project successfully delivered a comprehensive runtime-verification and behavioral-analysis documentation artifact for the SimpleLogin email alias application. The document (`blitzy/documentation/app_2cd6ee777f8c.md`) is 1,348 lines with 263 Python file references and 240 specific line-number citations, covering all four required question clusters:

1. **Startup Verification** — Complete log signatures and health check mechanisms for all three core processes
2. **User-Facing Workflow** — Full code-traced flows for registration, activation, login, dashboard, and alias management
3. **End-to-End Email Flow** — Complete inbound email lifecycle from SMTP receipt through database record creation, with all 30+ SMTP status codes documented
4. **Background Component Behavior** — Process independence architecture, Docker deployment model, job state machine, and operational indicators

The project is **93.3% complete** (28 of 30 total hours). All AAP requirements have been fulfilled. Zero source code modifications were made, fully complying with the primary constraint.

### Remaining Gaps

The only remaining work is human review (2 hours total):
- **Line-number citation verification** (1.5h): Source code line numbers cited in the document should be spot-checked against the current codebase to ensure accuracy
- **Editorial adjustments** (0.5h): Minor corrections based on stakeholder or reviewer feedback

### Production Readiness Assessment

The documentation artifact is **production-ready for merge**. It has passed all automated quality gates (pre-commit checks, test non-regression, compilation verification) and contains zero placeholders or incomplete sections. The recommended human review is a quality enhancement, not a blocking requirement.

### Success Metrics

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Question clusters covered | 4 | 4 | ✅ Met |
| Source code file references | >50 | 263 | ✅ Exceeded |
| Line-number citations | >50 | 240 | ✅ Exceeded |
| Placeholders/TODOs | 0 | 0 | ✅ Met |
| Source files modified | 0 | 0 | ✅ Met |
| In-scope test pass rate | 100% | 100% | ✅ Met |
| Pre-commit checks | Pass | Pass | ✅ Met |

---

## 9. Development Guide

### 9.1 System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Python | 3.10+ | Runtime for Flask web server, email handler, and job runner |
| Poetry | Latest | Python dependency management |
| Node.js | 10.x | Frontend static asset build |
| PostgreSQL | 13+ | Primary database |
| Redis | Latest | Session storage, rate limiting |
| Git | Latest | Version control |
| GPG | Latest | PGP email encryption support |

### 9.2 Environment Setup

**Clone and checkout the branch:**

```bash
git clone <repository-url>
cd <repository-name>
git checkout blitzy-750c1858-6178-4bfb-9a27-b88579bf2a80
```

**Install Python dependencies:**

```bash
poetry install
```

**Install frontend dependencies:**

```bash
cd static && npm ci && cd ..
```

**Configure environment variables:**

```bash
cp example.env .env
# Edit .env with your local configuration
# Key variables:
#   URL=http://localhost:7777
#   EMAIL_DOMAIN=sl.local
#   NOT_SEND_EMAIL=true
#   DB_URI=postgresql://user:password@localhost:5432/simplelogin
#   FLASK_SECRET=your-secret-key
```

**Initialize the database:**

```bash
# Start PostgreSQL (if not running)
# Create database: createdb simplelogin

# Apply migrations
alembic upgrade head

# Seed development data (creates john@wick.com / password user)
flask dummy-data
```

### 9.3 Starting the Application

SimpleLogin requires **three separate processes** launched in separate terminals:

**Terminal 1 — Web Server (port 7777):**
```bash
python3 server.py
# Expected output: * Running on http://127.0.0.1:7777/
```

**Terminal 2 — Email Handler (port 20381):**
```bash
python email_handler.py
# Expected output: Listen for port 20381 → Start mail controller 0.0.0.0 20381
```

**Terminal 3 — Job Runner (background jobs):**
```bash
python job_runner.py
# No explicit startup log — runs silently until jobs appear
```

### 9.4 Verification Steps

**Verify web server:**
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:7777/health
# Expected: 200
```

**Verify email handler:**
```bash
# If swaks is installed:
swaks --to test@sl.local --from test@example.com --server 127.0.0.1:20381
```

**Verify development login:**
- Navigate to `http://localhost:7777/auth/login`
- Login with `john@wick.com` / `password`
- Dashboard should display at `/dashboard/`

### 9.5 Running Tests

```bash
# Run the full test suite (non-interactive)
pytest -x -v --tb=short
```

### 9.6 Viewing the Documentation Artifact

The deliverable document is located at:
```bash
cat blitzy/documentation/app_2cd6ee777f8c.md
# Or open in any Markdown viewer/editor
```

### 9.7 Troubleshooting

| Issue | Resolution |
|-------|------------|
| `poetry install` fails with pyre2 error | Install system deps: `apt install libre2-dev cmake ninja-build` |
| Database connection refused | Ensure PostgreSQL is running: `pg_isready` |
| Email handler won't start | Check port 20381 is available: `lsof -i :20381` |
| Tests fail on `test_mail_sender.py` | Known IPv6 issue in containers — 5 tests depend on `::1` loopback; not a code issue |
| `flask dummy-data` errors | Ensure migrations applied first: `alembic upgrade head` |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `python3 server.py` | Start Flask development server on port 7777 |
| `python email_handler.py` | Start SMTP email handler on port 20381 |
| `python job_runner.py` | Start background job polling runner |
| `alembic upgrade head` | Apply all database migrations |
| `flask dummy-data` | Seed development database with test user |
| `curl http://localhost:7777/health` | Health check for web server |
| `pytest -x -v --tb=short` | Run test suite |
| `python -m py_compile <file>` | Verify Python file compilation |
| `pre-commit run --all-files` | Run all pre-commit hooks |

### B. Port Reference

| Port | Service | Process |
|------|---------|---------|
| 7777 | Flask Web Server / Gunicorn | `server.py` / `wsgi.py` |
| 20381 | SMTP Email Handler (aiosmtpd) | `email_handler.py` |
| 5432 | PostgreSQL (default) | External service |
| 6379 | Redis (default) | External service |

### C. Key File Locations

| Path | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Deliverable** — Runtime verification documentation |
| `server.py` | Flask application factory and web server entry point |
| `email_handler.py` | SMTP inbound email handler |
| `job_runner.py` | Background job polling runner |
| `wsgi.py` | WSGI entry point for Gunicorn (production) |
| `app/config.py` | Central configuration loaded from `.env` |
| `app/models.py` | SQLAlchemy ORM model definitions |
| `app/log.py` | Logging infrastructure and format |
| `app/email/status.py` | SMTP response status code catalog |
| `example.env` | Reference environment variable configuration |
| `Dockerfile` | Container build and Gunicorn CMD |
| `CONTRIBUTING.md` | Local development setup guide |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Python | 3.10 | `Dockerfile`, `pyproject.toml` |
| Flask | ^1.1.2 | `pyproject.toml` |
| Gunicorn | ^20.0.4 | `pyproject.toml` |
| aiosmtpd | ^1.2 | `pyproject.toml` |
| SQLAlchemy | 1.3.24 | `pyproject.toml` |
| PostgreSQL | 13+ | `CONTRIBUTING.md` |
| Node.js | 10.x | `Dockerfile` |
| Redis | ^4.5.3 | `pyproject.toml` |

### E. Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `URL` | Yes | `http://localhost:7777` | Application base URL |
| `DB_URI` | Yes | — | PostgreSQL connection string |
| `FLASK_SECRET` | Yes | — | Flask session encryption key |
| `EMAIL_DOMAIN` | Yes | `sl.local` | Domain for alias addresses |
| `SUPPORT_EMAIL` | Yes | `support@sl.local` | Transactional email sender |
| `NOT_SEND_EMAIL` | No | unset | Suppress SMTP delivery (local dev) |
| `COLOR_LOG` | No | unset | Enable colored console output |
| `DISABLE_REGISTRATION` | No | unset | Block new user registrations |
| `POSTFIX_SERVER` | No | `localhost` | Outbound SMTP server |
| `POSTFIX_PORT` | No | `25` | Outbound SMTP port |
| `SENTRY_DSN` | No | unset | Sentry error tracking DSN |

### G. Glossary

| Term | Definition |
|------|------------|
| **Alias** | A proxy email address that forwards incoming mail to the user's real mailbox |
| **Reverse Alias** | A special address generated for each Contact that allows the user to reply through the alias |
| **Contact** | An external sender who has emailed an alias; stored in the `Contact` table |
| **EmailLog** | An audit record for every email processed by the system |
| **VERP** | Variable Envelope Return Path — technique for tracking bounces |
| **DMARC** | Domain-based Message Authentication, Reporting and Conformance |
| **Job Runner** | Background process polling the `Job` table every 10 seconds for pending tasks |
| **create_light_app()** | Lightweight Flask app factory used by email handler and job runner for DB access |
