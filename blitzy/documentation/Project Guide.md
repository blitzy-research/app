# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project creates a **Q&A behavioral exploration guide** for the SimpleLogin local development environment. The deliverable is a single Markdown document (`blitzy/documentation/app_2cd6ee777f8c.md`, 773 lines) answering three targeted questions about startup behavior: (1) the exception raised when the database has no tables, (2) the stdout/stderr output of all required services, and (3) the SMTP rejection when `init_app.py` is skipped. Every answer is grounded in source code traces across 15+ files with 43 inline citations and 70+ verified line references. No existing source files were modified — this is a read-only documentation exercise.

### 1.2 Completion Status

```mermaid
pie title Project Completion
    "Completed (16h)" : 16
    "Remaining (2h)" : 2
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 18 |
| **Completed Hours (AI)** | 16 |
| **Remaining Hours** | 2 |
| **Completion Percentage** | 88.9% |

**Calculation:** 16 completed hours / (16 + 2) total hours = 88.9% complete.

### 1.3 Key Accomplishments

- ✅ Created `blitzy/documentation/app_2cd6ee777f8c.md` (773 lines) answering all 3 AAP scenarios
- ✅ Q1: Full exception trace from `server.py` → `login.py` → `models.py` → `sqlalchemy.exc.ProgrammingError` documented with GET vs POST path analysis
- ✅ Q2: All 3 services (`server.py:7777`, `email_handler.py:20381`, `job_runner.py` poll loop) documented with expected startup logs and port verification
- ✅ Q3: Complete SMTP flow trace through 6 functions to `E515` ("550 SL E515 Email not exist") with broader impact analysis
- ✅ 3 Mermaid diagrams created (sequence diagram, component topology, flowchart decision tree)
- ✅ 43 source citations and 70+ line number references verified correct across 15+ source files
- ✅ 3 validation issues identified and fixed (Flask app name, Werkzeug logger note, Q2 rationale heading)
- ✅ Zero source file modifications — AAP constraint fully honored
- ✅ All 4 production-readiness gates passed

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Documentation accuracy not verified against live runtime | Claims are code-derived but not runtime-tested | Human Developer | 1h |
| Line number references may drift with future code changes | Source citations become stale after upstream merges | Human Developer | As needed |

### 1.5 Access Issues

No access issues identified. The deliverable is a standalone Markdown file committed to the repository. No external service credentials, API keys, or deployment access are required.

### 1.6 Recommended Next Steps

1. **[High]** Review documentation accuracy by running the 3 scenarios in a live local SimpleLogin development environment
2. **[Medium]** Confirm all Mermaid diagrams render correctly in the target Markdown viewer (GitHub, GitLab, etc.)
3. **[Low]** Consider integrating `blitzy/documentation/` into a documentation generator (mkdocs, etc.) if the project adopts one in the future
4. **[Low]** Periodically re-verify line number references after upstream code changes

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Repository analysis and code discovery | 3 | Reading 15+ source files (10,314 LOC across `server.py`, `email_handler.py`, `app/models.py`, `app/auth/views/login.py`, `app/db.py`, `app/log.py`, `app/email/status.py`, `app/alias_utils.py`, `app/email_utils.py`, `app/config.py`, `init_app.py`, `job_runner.py`, `app/extensions.py`, `example.env`, `CONTRIBUTING.md`); tracing 3 distinct execution paths |
| Q1: Database-not-migrated error documentation | 2.5 | Flask lifecycle trace (`server.py:local_main` → `create_app`), `app/db.py` import-time behavior, GET vs POST differentiation, `sqlalchemy.exc.ProgrammingError` identification, sequence diagram |
| Q2: Service startup verification | 2.5 | 3 service startup sequences (`server.py`, `email_handler.py`, `job_runner.py`), `app/log.py` format analysis, port binding verification commands, component topology diagram |
| Q3: Missing init_app.py behavior | 3 | SMTP flow trace through `MailHandler.handle_DATA` → `_handle` → `handle` → `handle_forward` → `try_auto_create` → `E515`; broader impact analysis of empty `public_domain` table; flowchart diagram |
| Mermaid diagrams (3) | 1 | Sequence diagram (login lifecycle), component diagram (service topology), flowchart (email handler forward-phase decision tree) |
| Source references and document formatting | 1 | Comprehensive 60-row references table, introduction section, prerequisite environment table, document structure |
| Validation and bug fixes | 3 | Line number verification across 15+ files, 3 bug fixes (Flask app name correction, Werkzeug logger note rewrite, Q2 rationale heading addition), structural integrity checks (28 balanced code block pairs, 3 Mermaid diagrams confirmed) |
| **Total** | **16** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human review of documentation accuracy against live runtime | 1 | High |
| Runtime verification of 3 documented scenarios | 1 | Medium |
| **Total** | **2** | |

### 2.3 Hours Validation

- Section 2.1 total (Completed): **16h**
- Section 2.2 total (Remaining): **2h**
- Sum: 16 + 2 = **18h** = Total Project Hours in Section 1.2 ✓
- Completion: 16/18 = **88.9%** ✓

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Structural Validation — Code Block Balance | Custom (Blitzy Validator) | 1 | 1 | 0 | 100% | 56 code fence markers = 28 balanced pairs |
| Structural Validation — Mermaid Diagrams | Custom (Blitzy Validator) | 1 | 1 | 0 | 100% | 3 Mermaid diagrams (sequence, component, flowchart) verified |
| Structural Validation — Required Sections | Custom (Blitzy Validator) | 1 | 1 | 0 | 100% | 4 required sections present (Intro, Q1, Q2, Q3) |
| Source Citation Verification | Custom (Blitzy Validator) | 1 | 1 | 0 | 100% | 43 source citations verified, 70+ line references checked against actual files |
| Trailing Whitespace Check | Custom (Blitzy Validator) | 1 | 1 | 0 | 100% | Zero trailing whitespace across 773 lines |
| Line Reference Accuracy — `server.py` | Custom (Blitzy Validator) | 1 | 1 | 0 | 100% | Lines 76, 83, 127-136, 139-217, 218-228, 249-255, 388-394, 572-588, 598-599 verified |
| Line Reference Accuracy — `email_handler.py` | Custom (Blitzy Validator) | 1 | 1 | 0 | 100% | Lines 536, 541-555, 1945, 2195-2211, 2227-2233, 2288-2289, 2335-2378, 2381-2393, 2396-2404 verified |
| Line Reference Accuracy — Other files (13) | Custom (Blitzy Validator) | 1 | 1 | 0 | 100% | `app/models.py`, `app/auth/views/login.py`, `app/db.py`, `app/log.py`, `app/email/status.py`, `app/alias_utils.py`, `app/email_utils.py`, `app/config.py`, `app/extensions.py`, `init_app.py`, `job_runner.py`, `example.env`, `CONTRIBUTING.md` — all verified |
| **Total** | | **8** | **8** | **0** | **100%** | All tests from Blitzy autonomous validation |

All tests originate from Blitzy's autonomous validation logs for this project. No external test suites exist for standalone documentation files.

---

## 4. Runtime Validation & UI Verification

This project delivers a standalone Markdown documentation file. No application runtime, server startup, or UI interaction is involved in the deliverable itself.

**Documentation File Verification:**

- ✅ File exists: `blitzy/documentation/app_2cd6ee777f8c.md` (773 lines, 36,481 bytes)
- ✅ File committed: 3 commits on branch `blitzy-5627ec3b-943a-4b4d-b9ab-20c690812d4b`
- ✅ Working tree clean: No untracked, unstaged, or stashed changes
- ✅ Branch ahead of origin by 1 commit (latest fix commit)
- ✅ Code block balance: 28 matched pairs (56 markers)
- ✅ Mermaid diagrams: 3 embedded diagrams
- ✅ Source citations: 43 inline references
- ✅ Section structure: Introduction + 3 Questions + Source References

**Runtime Scenarios (documented, not executed):**

- ⚠️ Q1 scenario (empty DB + login POST): Code-traced, not live-tested
- ⚠️ Q2 scenario (3 service startup): Code-traced, not live-tested
- ⚠️ Q3 scenario (missing init_app.py + email): Code-traced, not live-tested

Human runtime verification is recommended to confirm documented behaviors match actual output.

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence | Notes |
|----------------|--------|----------|-------|
| Create `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | File exists, 773 lines, committed | Sole deliverable |
| Q1: Database-not-migrated error on login page | ✅ Pass | Lines 38–206: server startup, GET/POST analysis, exception trace, rationale, sequence diagram | Complete with `ProgrammingError` identification |
| Q2: Required services and startup verification | ✅ Pass | Lines 209–433: 3 services, startup logs, port binding, rationale, component diagram | All 3 services documented |
| Q3: Missing init_app.py email rejection behavior | ✅ Pass | Lines 437–707: seeding analysis, SMTP flow, E515 status, log output, rationale, flowchart | Complete with `"550 SL E515 Email not exist"` |
| Source references section | ✅ Pass | Lines 711–773: 60-row table with file paths and line ranges | All 15+ files referenced |
| No source file modifications | ✅ Pass | `git diff --name-status` shows only 1 added file, 0 modified | AAP constraint satisfied |
| Code-trace-backed reasoning | ✅ Pass | 43 `Source:` citations, 70+ line references verified correct | Every claim cites specific files |
| Mermaid diagrams (3 required) | ✅ Pass | Sequence (login flow), Component (service topology), Flowchart (email decision tree) | All 3 render in standard Mermaid |
| Document formatting per repository style | ✅ Pass | Markdown with headers, code blocks, tables matching `CONTRIBUTING.md` style | Consistent with existing docs |
| Thinking/rationale for each answer | ✅ Pass | Rationale subsections in Q1 (line 169), Q2 (line 390), Q3 (line 666) | Progressive disclosure structure |
| Evidence-based answers only | ✅ Pass | All conclusions grounded in source code; no assumptions | Per AAP rule: "code as the truth" |

**Validation Fixes Applied:**

| Fix | Commit | Description |
|-----|--------|-------------|
| Flask app name correction | `49cb55c7` | Changed `"app"` to `"server"` in expected Flask startup banner |
| Werkzeug logger suppression note | `49cb55c7` | Rewrote to distinguish `click.echo()` (banner) from `_log()` (internal) |
| Q2 Rationale heading + code block identifiers | `580072c8` | Added missing `### Rationale` heading and `text` language identifiers |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Line number references become stale after upstream code changes | Technical | Medium | Medium | Pin references to commit hash or periodically re-verify; source references table enables targeted updates | Open |
| Documented behaviors not verified against live runtime | Technical | Medium | Low | Run all 3 scenarios in a local SimpleLogin environment and compare output to documentation | Open |
| Mermaid diagrams may not render in all Markdown viewers | Technical | Low | Low | Mermaid is supported by GitHub, GitLab, and most modern Markdown viewers; fallback is code block text | Accepted |
| `blitzy/documentation/` directory not integrated with any documentation system | Operational | Low | Low | No documentation generator exists in the repository; integrate if one is adopted | Accepted |
| No automated regression test for documentation accuracy | Operational | Low | Medium | Create a CI script that parses source citations and verifies line contents match | Open |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 16
    "Remaining Work" : 2
```

**Completed: 16h | Remaining: 2h | Total: 18h | 88.9% Complete**

| Work Area | Completed Hours | Remaining Hours |
|-----------|----------------|-----------------|
| Code analysis and discovery | 3 | 0 |
| Q1 Documentation | 2.5 | 0 |
| Q2 Documentation | 2.5 | 0 |
| Q3 Documentation | 3 | 0 |
| Diagrams | 1 | 0 |
| Formatting and references | 1 | 0 |
| Validation and fixes | 3 | 0 |
| Human review | 0 | 1 |
| Runtime verification | 0 | 1 |
| **Total** | **16** | **2** |

---

## 8. Summary & Recommendations

### Achievement Summary

The project successfully delivered a comprehensive 773-line Q&A behavioral exploration guide for the SimpleLogin local development environment. All three AAP-scoped scenarios are fully documented with deep code traces, exact exception details, startup log output, SMTP status codes, and decision-flow diagrams. The documentation covers 15+ source files with 43 source citations and 70+ verified line number references. Three Mermaid diagrams (sequence, component, and flowchart) illustrate the key execution paths. All four production-readiness gates passed during autonomous validation, and three issues were identified and fixed.

### Completion Assessment

The project is **88.9% complete** (16 completed hours out of 18 total hours). All AAP-specified deliverables have been autonomously completed. The remaining 2 hours consist of human review tasks: verifying documentation accuracy against live runtime behavior (1h) and confirming the 3 documented scenarios produce the exact output described (1h).

### Critical Path to Production

1. **Human review** (1h): A developer familiar with SimpleLogin should read the document and verify technical accuracy of all code traces and claims.
2. **Runtime verification** (1h): Execute the 3 documented scenarios in a local SimpleLogin environment and compare actual output to the documentation.

### Production Readiness Assessment

The documentation file is **ready for merge** after human review. It is a standalone Markdown file with no build dependencies, no runtime requirements, and no impact on existing source code. The `blitzy/documentation/` directory is new and does not conflict with any existing project structure.

---

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Git | Any recent | Clone repository and inspect branch |
| Markdown viewer | Any (GitHub, VS Code, etc.) | View rendered documentation with Mermaid diagrams |
| Python | ^3.10 | Required only if reproducing documented scenarios |
| PostgreSQL | Any recent | Required only if reproducing documented scenarios |
| Poetry | Any recent | Required only if reproducing documented scenarios |

### Viewing the Documentation

```bash
# Clone the repository and switch to the feature branch
git clone <repository-url>
cd <repository-root>
git checkout blitzy-5627ec3b-943a-4b4d-b9ab-20c690812d4b

# View the documentation file
cat blitzy/documentation/app_2cd6ee777f8c.md

# Or open in any Markdown viewer for rendered Mermaid diagrams
# GitHub and GitLab render Mermaid natively in .md files
```

### Reproducing the Documented Scenarios

To verify the documentation against actual runtime behavior, set up the SimpleLogin local development environment:

#### Step 1: Environment Setup

```bash
# Install Python dependencies
poetry install

# Copy environment template
cp example.env .env

# Ensure .env has these key values (defaults from example.env):
# URL=http://localhost:7777
# EMAIL_DOMAIN=sl.local
# DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
# FLASK_SECRET=secret
# NOT_SEND_EMAIL=true
```

#### Step 2: Database Setup

```bash
# Create PostgreSQL database
createdb -U myuser simplelogin

# For Q1 scenario: STOP HERE — do not run migrations
# For Q2/Q3 scenarios: Run migrations
alembic upgrade head
```

#### Step 3: Reproduce Q1 — Database-Not-Migrated Error

```bash
# Start server with empty database (no migrations)
python server.py

# In a browser, navigate to http://localhost:7777/
# GET /login should render successfully
# Submit the login form (POST /login) — should see 500 error
# Check terminal for sqlalchemy.exc.ProgrammingError traceback
```

#### Step 4: Reproduce Q2 — Service Startup Verification

```bash
# Terminal 1: Web server
python server.py
# Expected: ">>> init logging <<<" then Flask banner on port 7777

# Terminal 2: Email handler
python email_handler.py
# Expected: ">>> init logging <<<" then "Listen for port 20381"

# Terminal 3: Job runner
python job_runner.py
# Expected: ">>> init logging <<<" then silence (polling)

# Verify port binding
lsof -i :7777
lsof -i :20381
ps aux | grep job_runner.py
```

#### Step 5: Reproduce Q3 — Missing init_app.py Email Rejection

```bash
# Ensure migrations are applied but init_app.py is NOT run
# Start email handler
python email_handler.py

# Send test email (requires swaks: apt-get install swaks)
swaks --to test@sl.local --from sender@example.com --server 127.0.0.1:20381

# Expected SMTP response: 550 SL E515 Email not exist
# Check email handler logs for the rejection trace
```

### Troubleshooting

| Issue | Resolution |
|-------|-----------|
| Mermaid diagrams not rendering | Use a Mermaid-compatible viewer (GitHub, GitLab, VS Code with Mermaid extension) |
| `poetry install` fails | Ensure Python ^3.10 is active; check `pyproject.toml` for dependency constraints |
| PostgreSQL connection refused | Verify PostgreSQL is running: `pg_isready -h localhost -p 5432` |
| `swaks` not found | Install: `apt-get install -y swaks` (Debian/Ubuntu) or `brew install swaks` (macOS) |
| Port 7777 already in use | Kill existing process: `lsof -ti :7777 | xargs kill` |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `cat blitzy/documentation/app_2cd6ee777f8c.md` | View the documentation file |
| `python server.py` | Start Flask web server on port 7777 |
| `python email_handler.py` | Start SMTP email handler on port 20381 |
| `python job_runner.py` | Start background job polling loop |
| `python init_app.py` | Seed SLDomain records into `public_domain` table |
| `alembic upgrade head` | Run all database migrations |
| `swaks --to test@sl.local --from sender@example.com --server 127.0.0.1:20381` | Send test email to SMTP handler |
| `lsof -i :7777` | Verify web server port binding |
| `lsof -i :20381` | Verify email handler port binding |
| `ps aux \| grep job_runner.py` | Verify job runner process is running |

### B. Port Reference

| Service | Port | Bind Address | Protocol |
|---------|------|--------------|----------|
| Flask Web Server (`server.py`) | 7777 | 127.0.0.1 | HTTP |
| Email Handler (`email_handler.py`) | 20381 | 0.0.0.0 | SMTP |
| Job Runner (`job_runner.py`) | N/A | N/A | DB polling |
| PostgreSQL | 5432 | localhost | PostgreSQL |

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Deliverable** — Q&A behavioral exploration guide |
| `server.py` | Flask web app bootstrap (599 lines) |
| `email_handler.py` | SMTP inbound handler (2,404 lines) |
| `job_runner.py` | Background job processor (347 lines) |
| `init_app.py` | Data seeder for SLDomain records (73 lines) |
| `app/models.py` | SQLAlchemy ORM models (3,843 lines) |
| `app/auth/views/login.py` | Login route handler (82 lines) |
| `app/db.py` | Database engine and session (18 lines) |
| `app/config.py` | Environment configuration (666 lines) |
| `app/email/status.py` | SMTP status code constants (64 lines) |
| `app/alias_utils.py` | Alias auto-creation logic (598 lines) |
| `app/log.py` | Logging infrastructure (79 lines) |
| `example.env` | Environment variable template |
| `CONTRIBUTING.md` | Developer setup guide |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Python | ^3.10 | `pyproject.toml` `[tool.black] target-version` |
| Flask | ^1.1.2 | `pyproject.toml` dependencies |
| SQLAlchemy | 1.3.24 | `pyproject.toml` dependencies (via flask-sqlalchemy) |
| psycopg2-binary | ^2.9.3 | `pyproject.toml` dependencies |
| aiosmtpd | ^1.2 | `pyproject.toml` dependencies |
| flask-login | ^0.5.0 | `pyproject.toml` dependencies |
| flask-debugtoolbar | ^0.11.0 | `pyproject.toml` dependencies |
| gunicorn | ^20.0.4 | `pyproject.toml` dependencies |
| sentry_sdk | ^2.16.0 | `pyproject.toml` dependencies |
| arrow | ^0.16.0 | `pyproject.toml` dependencies |

### E. Environment Variable Reference

| Variable | Default Value | Source | Purpose |
|----------|--------------|--------|---------|
| `URL` | `http://localhost:7777` | `example.env:6` | Base URL for the web application |
| `NOT_SEND_EMAIL` | `true` | `example.env:19` | Disable actual email sending in development |
| `EMAIL_DOMAIN` | `sl.local` | `example.env:22` | Primary email domain for alias generation |
| `SUPPORT_EMAIL` | `support@sl.local` | `example.env:40` | Support contact email address |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `example.env:75` | PostgreSQL connection string |
| `FLASK_SECRET` | `secret` | `example.env:77` | Flask session secret key |
| `ALIAS_DOMAINS` | Derived: `[EMAIL_DOMAIN]` | `app/config.py:157–161` | List of domains for alias creation |
| `LOAD_PGP_EMAIL_HANDLER` | Not set | `app/config.py:339` | Enable PGP key loading in email handler |

### G. Glossary

| Term | Definition |
|------|-----------|
| **SLDomain** | SQLAlchemy model (table `public_domain`) representing SimpleLogin-managed email domains |
| **ALIAS_DOMAINS** | Configuration list of domains on which aliases can be created; seeded into DB by `init_app.py` |
| **E515** | SMTP status code `"550 SL E515 Email not exist"` — permanent rejection for unknown addresses |
| **`create_app()`** | Full Flask application factory in `server.py` with all blueprints and extensions |
| **`create_light_app()`** | Minimal Flask application factory in `server.py` for background services (DB session only) |
| **`ModelMixin.get_by()`** | Convenience method at `app/models.py:82–84` that wraps `Session.query(cls).filter_by(**kw).first()` |
| **`try_auto_create()`** | Function at `app/alias_utils.py:202–224` that attempts on-the-fly alias creation via custom domains or directories |
| **aiosmtpd** | Python async SMTP server library used by `email_handler.py` to listen for inbound emails |
| **ProgrammingError** | SQLAlchemy exception wrapping `psycopg2.errors.UndefinedTable` when querying a non-existent table |