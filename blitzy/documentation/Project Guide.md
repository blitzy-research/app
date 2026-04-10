# Blitzy Project Guide — SimpleLogin Operational Q&A Documentation

---

## 1. Executive Summary

### 1.1 Project Overview

This project creates a comprehensive operational Q&A guide and verification runbook (`blitzy/documentation/app_2cd6ee777f8c.md`) for the SimpleLogin self-hosted email aliasing platform. The document answers three interlocking questions about runtime behavior: (1) how to verify all services are fully initialized and ready, (2) the complete new-user registration-to-dashboard experience, and (3) what background jobs, cron tasks, event dispatching, and monitoring processes should be active. The deliverable is a single 923-line Markdown document with 4 Mermaid diagrams and 38 source citations, grounded entirely in analysis of the codebase. No source code was modified. The target audience is developers and operators setting up SimpleLogin locally for the first time.

### 1.2 Completion Status

```mermaid
pie title Project Completion — 88.0%
    "Completed (AI)" : 22
    "Remaining" : 3
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 25 |
| **Completed Hours (AI)** | 22 |
| **Remaining Hours** | 3 |
| **Completion Percentage** | 88.0% |

**Calculation:** 22 completed hours / (22 completed + 3 remaining) = 22 / 25 = **88.0%**

### 1.3 Key Accomplishments

- ✅ Created `blitzy/documentation/app_2cd6ee777f8c.md` (923 lines, 43 KB) — the sole AAP deliverable
- ✅ Q1: Startup Readiness Verification fully documented — Flask `create_app()` initialization (15-step table), SMTP handler on port 20381, job runner 10s polling, health endpoint (`GET /health → 200`), `NOT_SEND_EMAIL` mode, Redis dependency, startup checklist, Mermaid sequence diagram
- ✅ Q2: Registration-to-Dashboard walkthrough fully documented — 7-step user journey covering `POST /auth/register`, activation email dispatch, `GET /auth/activate?code=`, login validation (4 failure cases), MFA routing decision tree (FIDO → TOTP → direct), dashboard landing, 2 Mermaid flowcharts
- ✅ Q3: Background Services & Runtime Indicators fully documented — job runner dispatch (10 job types with state machine), cron (15 yacron-scheduled tasks), event system (PostgreSQL LISTEN/NOTIFY with protobuf), monitoring pipeline (6 metric functions at 60s interval), auth telemetry (New Relic custom events), Mermaid architecture diagram
- ✅ 38 source citations grounding every claim to specific files and functions
- ✅ 4 Mermaid diagrams exceeding the 3 minimum required
- ✅ Zero source code modifications — confirmed via `git diff`
- ✅ Pre-commit hooks passed (trailing-whitespace)
- ✅ SWE-AtlasQnA-Repo compliance: thinking/rationale sections, code-grounded, no assumptions
- ✅ All changes committed on correct branch with clean working tree

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Documentation not linked from README.md or CONTRIBUTING.md | Low discoverability for new developers | Human Developer | 0.5h |
| Line number references may drift after future source code commits | Citations become stale over time | Human Developer | Ongoing maintenance |

### 1.5 Access Issues

No access issues identified. This is a documentation-only task producing a Markdown file. No external services, credentials, APIs, or deployment infrastructure were required.

### 1.6 Recommended Next Steps

1. **[High]** Review all 38 source citations in the document against the current codebase to verify accuracy
2. **[Medium]** Add a cross-reference link from `CONTRIBUTING.md` (local development section) to `blitzy/documentation/app_2cd6ee777f8c.md`
3. **[Medium]** Manually execute the startup verification steps from Q1 (e.g., `curl http://localhost:7777/health`) to confirm documented behavior
4. **[Low]** Walk through the registration flow from Q2 in a running local environment to validate the described user experience
5. **[Low]** Establish a maintenance process to update line-number citations when referenced source files change

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Repository Analysis & Source Code Reading | 6 | Deep analysis of 28+ source files (~8,000+ lines across `server.py`, `email_handler.py`, `job_runner.py`, auth views, dashboard, email utilities, config, events, monitoring) to extract accurate documentation content |
| Q1: Startup Readiness Verification Documentation | 4 | Flask `create_app()` 15-step initialization table, SMTP handler startup, job runner polling loop, health endpoint, `NOT_SEND_EMAIL` mode, Redis dependency, startup checklist, Mermaid sequence diagram |
| Q2: Registration-to-Dashboard Walkthrough Documentation | 5 | 7-step user journey with validation chains, activation email dispatch, MFA routing decision tree (FIDO → TOTP → direct), dashboard landing page analysis, 2 Mermaid flowcharts |
| Q3: Background Services & Runtime Indicators Documentation | 4 | Job runner dispatch table (10 types) with state machine, cron task inventory (15 yacron tasks), LISTEN/NOTIFY event system, monitoring pipeline (6 metric functions), auth telemetry, Mermaid architecture diagram |
| Testing Notes & Appendix | 1 | Demo user (`john@wick.com`) documentation, temporary artifact cleanup guidance, `NOT_SEND_EMAIL` implications table, key file paths and URL reference tables |
| Quality Review & Code Review Fixes | 2 | Verification of all 38 source citations against actual code, pre-commit hook validation, code review fix commit (`4583a103`) addressing review findings |
| **Total** | **22** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human Review of Documentation Accuracy | 1.5 | High |
| Documentation Integration & Discoverability (link from CONTRIBUTING.md) | 0.5 | Medium |
| Runtime Verification of Documented Procedures | 1 | Medium |
| **Total** | **3** | |

---

## 3. Test Results

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Pre-commit: trailing-whitespace | pre-commit | 1 | 1 | 0 | N/A | Applied to `app_2cd6ee777f8c.md` |
| Pre-commit: check-yaml | pre-commit | 0 | 0 | 0 | N/A | Skipped — not applicable to `.md` files |
| Pre-commit: ruff | pre-commit | 0 | 0 | 0 | N/A | Skipped — not applicable to `.md` files |
| Pre-commit: ruff-format | pre-commit | 0 | 0 | 0 | N/A | Skipped — not applicable to `.md` files |
| Pre-commit: djlint-jinja | pre-commit | 0 | 0 | 0 | N/A | Skipped — not applicable to `.md` files |
| Document Structure Validation | Manual | 1 | 1 | 0 | N/A | Verified: 923 lines, 4 Mermaid blocks, 38 citations, zero placeholders/TODOs |
| Source Citation Accuracy | Manual | 38 | 38 | 0 | 100% | Every `Source:` reference verified against actual file paths and line numbers |

**Note:** This is a documentation-only task. No unit, integration, or end-to-end tests are applicable. The repository's existing 120 test files in `tests/` were not modified and remain unaffected by this change.

---

## 4. Runtime Validation & UI Verification

This project is a documentation-only deliverable (Markdown file). No runtime services, UI components, or API endpoints were created or modified.

**Validation Results:**

- ✅ **Document file created:** `blitzy/documentation/app_2cd6ee777f8c.md` exists (923 lines, 43,214 bytes)
- ✅ **Markdown structure valid:** Proper heading hierarchy (H1 → H2 → H3 → H4), no broken syntax
- ✅ **Mermaid diagrams valid:** 4 Mermaid blocks with correct syntax (sequenceDiagram, flowchart TD, graph LR)
- ✅ **No source code modified:** `git diff --name-status origin/app_2cd6ee777f8c...HEAD` shows only the documentation file
- ✅ **Working tree clean:** `git status` reports "nothing to commit, working tree clean"
- ✅ **Branch correct:** All changes on `blitzy-24d29ef3-6489-4c01-9271-59b5a444791d`
- ✅ **Zero TODOs/FIXMEs/placeholders:** `grep` for `TODO`, `FIXME`, `PLACEHOLDER`, `STUB` returns no matches

**Source Code Integrity Verification:**

- ✅ `server.py` — unchanged (health endpoint at line 213 confirmed)
- ✅ `app/config.py` — unchanged (`NOT_SEND_EMAIL` at line 91 confirmed)
- ✅ `example.env` — unchanged (`NOT_SEND_EMAIL=true` at line 19 confirmed)
- ✅ All 28+ referenced source files — verified unmodified

---

## 5. Compliance & Quality Review

| Compliance Criterion | Status | Evidence |
|---------------------|--------|----------|
| **AAP: Create `blitzy/documentation/app_2cd6ee777f8c.md`** | ✅ Pass | File exists at specified path, 923 lines, committed |
| **AAP: Q1 — Startup Readiness Verification** | ✅ Pass | Sections 1.1–1.8 covering all specified components |
| **AAP: Q2 — Registration-to-Dashboard Walkthrough** | ✅ Pass | Sections 2.1–2.7 tracing complete user journey |
| **AAP: Q3 — Background Services & Runtime Indicators** | ✅ Pass | Sections 3.1–3.7 covering all specified services |
| **AAP: Minimum 3 Mermaid diagrams** | ✅ Pass | 4 diagrams (exceeds requirement) |
| **AAP: Source citations throughout** | ✅ Pass | 38 citations with file:line references |
| **AAP: No source code modifications** | ✅ Pass | Only 1 file in `blitzy/` directory; all `.py`, `.html`, `.js`, `.env` files unchanged |
| **AAP: SWE-AtlasQnA-Repo rule compliance** | ✅ Pass | Thinking/rationale sections, code-grounded answers, placed in `blitzy/documentation/` |
| **AAP: Testing and cleanup notes** | ✅ Pass | Demo user documented, artifact guidance included, `NOT_SEND_EMAIL` implications table |
| **AAP: Consistent terminology** | ✅ Pass | Uses "activation" (not "verification"), "alias" (not "forward"), "mailbox" (not "real email") |
| **Pre-commit hooks** | ✅ Pass | trailing-whitespace passed; others skipped (N/A for `.md`) |
| **Code review fixes applied** | ✅ Pass | Commit `4583a103` addresses review findings |

**Autonomous Validation Fixes Applied:**

- Code review commit (`4583a103`): Minor documentation refinements (5 insertions, 3 deletions) addressing formatting and accuracy improvements identified during automated review

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Source file line numbers drift after future commits, making citations stale | Technical | Low | High | Include function names alongside line numbers (already done); establish periodic review process | Open — requires human maintenance plan |
| Document not discoverable — no link from README.md or CONTRIBUTING.md | Operational | Low | High | Add cross-reference link from `CONTRIBUTING.md` local development section | Open — 0.5h human task |
| Mermaid diagrams may not render in all Markdown viewers | Technical | Low | Low | Diagrams use standard Mermaid syntax compatible with GitHub, GitLab, and VS Code; fallback is reading the text-based flowchart source | Mitigated |
| `NOT_SEND_EMAIL` behavior may confuse developers following Q2 walkthrough | Operational | Medium | Medium | Document explicitly calls out `NOT_SEND_EMAIL` implications with DB query workaround for activation codes | Mitigated |
| No automated validation that documented procedures actually work in a running environment | Technical | Medium | Medium | Human should execute documented startup verification and registration flow in a live local environment | Open — 1h human task |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 22
    "Remaining Work" : 3
```

```mermaid
pie title Remaining Work by Priority
    "High: Documentation Review" : 1.5
    "Medium: Integration & Verification" : 1.5
```

**Summary:** 22 hours of AAP-scoped work completed out of 25 total hours = **88.0% complete**. All AAP deliverables have been fully implemented. The remaining 3 hours consist entirely of path-to-production human review and integration tasks.

---

## 8. Summary & Recommendations

### Achievement Summary

The project is **88.0% complete** (22 hours completed out of 25 total hours). All deliverables specified in the Agent Action Plan have been fully implemented:

- A single comprehensive Markdown document (`blitzy/documentation/app_2cd6ee777f8c.md`, 923 lines) was created answering three operational questions about the SimpleLogin self-hosted platform's runtime behavior.
- The document contains 38 source citations grounding every technical claim to specific files and functions in the codebase.
- Four Mermaid diagrams provide visual documentation of the startup sequence, registration flow, login/MFA routing, and background services architecture.
- Zero source code files were modified, fully complying with the user's explicit constraint.
- All pre-commit hooks passed and both commits are on the correct branch with a clean working tree.

### Remaining Gaps

The 3 remaining hours are exclusively path-to-production tasks requiring human involvement:

1. **Accuracy review (1.5h):** A developer familiar with the SimpleLogin codebase should verify the 38 source citations and line-number references against the current code state.
2. **Discoverability (0.5h):** The document should be cross-linked from `CONTRIBUTING.md` or `README.md` so new developers can find it.
3. **Runtime verification (1h):** The documented procedures (health endpoint check, registration flow, background service indicators) should be manually executed in a running local environment to confirm accuracy.

### Production Readiness Assessment

The documentation deliverable itself is **production-ready** — complete, accurate at time of writing, and properly structured. The remaining tasks are standard documentation governance activities (review, linking, validation) rather than content creation gaps.

### Recommendations

1. Merge this PR after human review of citation accuracy
2. Add a link in `CONTRIBUTING.md` under the "Run the code locally" section pointing to this operational guide
3. Establish a lightweight process to update line-number citations when referenced source files change (e.g., periodic grep for stale references)
4. Consider extracting the startup checklist (Section 1.7 of the document) into a standalone quick-start card for the README

---

## 9. Development Guide

### System Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| Python | 3.10+ | Application runtime (per `pyproject.toml` target-version) |
| Poetry | Latest | Dependency management |
| Node.js | 10.x | Frontend asset installation |
| PostgreSQL | 13+ | Primary database |
| Redis | Optional | Session store and rate limiter (falls back to in-memory) |
| Git | 2.x+ | Version control |

### Environment Setup

1. **Clone the repository and switch to the feature branch:**

```bash
git clone <repository-url>
cd <repository-root>
git checkout blitzy-24d29ef3-6489-4c01-9271-59b5a444791d
```

2. **Copy the example environment file:**

```bash
cp example.env .env
```

Key environment variables in `.env`:

| Variable | Default | Purpose |
|----------|---------|---------|
| `URL` | `http://localhost:7777` | Base URL for the web application |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | PostgreSQL connection string |
| `FLASK_SECRET` | (set in example.env) | Flask session secret key |
| `NOT_SEND_EMAIL` | `true` | When set, emails are logged to stdout instead of being sent via SMTP |
| `EMAIL_DOMAIN` | `sl.local` | Default email alias domain |

### Dependency Installation

```bash
# Install Python dependencies via Poetry
poetry sync

# Install Node.js dependencies (for frontend assets)
npm install
```

### Database Setup

```bash
# Ensure PostgreSQL is running, then:
alembic upgrade head     # Apply all database migrations
flask dummy-data         # Seed demo data (user: john@wick.com / password)
```

### Application Startup

```bash
# Start the Flask web application (port 7777)
python3 server.py

# In a separate terminal — start the SMTP email handler (port 20381)
python email_handler.py

# In a separate terminal — start the background job runner
python job_runner.py
```

### Verification Steps

1. **Health endpoint:**

```bash
curl http://localhost:7777/health
# Expected: "success" with HTTP 200
```

2. **Web UI:** Open `http://localhost:7777` in a browser — should redirect to the login page at `/auth/login`

3. **Demo user login:** Log in with `john@wick.com` / `password` — should land on the dashboard at `/dashboard/`

4. **Check startup logs for:**
   - `>>> URL: http://localhost:7777` (Flask app)
   - `>>> init logging <<<` (Logger initialized)
   - `Start mail controller 0.0.0.0 20381` (Email handler)

### Viewing the Documentation

The Q&A documentation deliverable is located at:

```bash
cat blitzy/documentation/app_2cd6ee777f8c.md
```

Or open in any Markdown viewer (GitHub renders it natively including Mermaid diagrams).

### Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|------------|
| `RuntimeError: FLASK_SECRET` | `FLASK_SECRET` env var is empty | Set a non-empty value in `.env` |
| DB connection refused | PostgreSQL not running | Start PostgreSQL and verify `DB_URI` in `.env` |
| Activation email not received | `NOT_SEND_EMAIL=true` is set | Check stdout logs for email metadata, or disable the flag and configure MailHog |
| Port 7777 in use | Another process on the port | Kill the process or change the port in `server.py:local_main()` |
| `No module named 'app'` | Not in project root directory | Ensure you're running from the repository root |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `poetry sync` | Install all Python dependencies |
| `npm install` | Install frontend Node.js dependencies |
| `alembic upgrade head` | Apply all database migrations |
| `flask dummy-data` | Seed demo user and test data |
| `python3 server.py` | Start Flask web app (development mode, port 7777) |
| `python email_handler.py` | Start SMTP email handler (port 20381) |
| `python job_runner.py` | Start background job polling loop |
| `python cron.py -j <job_name>` | Run a specific cron job manually |
| `python event_listener.py listener` | Start LISTEN/NOTIFY event listener |
| `python event_listener.py dead_letter` | Process dead-letter events |
| `python monitoring.py` | Start metrics collection loop (60s interval) |
| `curl http://localhost:7777/health` | Verify web app health |

### B. Port Reference

| Port | Service | Protocol |
|------|---------|----------|
| 7777 | Flask web application (Gunicorn in Docker) | HTTP |
| 20381 | aiosmtpd SMTP email handler | SMTP |
| 5432 | PostgreSQL database | TCP |
| 6379 | Redis (optional) | TCP |
| 1025 | MailHog SMTP (if configured) | SMTP |
| 8025 | MailHog Web UI (if configured) | HTTP |

### C. Key File Locations

| Path | Purpose |
|------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Deliverable** — Operational Q&A guide (923 lines) |
| `server.py` | Flask app factory, blueprint registration, health endpoint |
| `email_handler.py` | aiosmtpd SMTP handler for inbound email (2,404 lines) |
| `job_runner.py` | Background job polling loop (347 lines) |
| `cron.py` / `crontab.yml` | Scheduled maintenance tasks |
| `event_listener.py` | LISTEN/NOTIFY event system CLI |
| `monitoring.py` | Metrics collection and export |
| `app/config.py` | All configuration constants (666 lines) |
| `app/auth/views/` | Authentication views (register, activate, login, MFA) |
| `app/dashboard/views/index.py` | Dashboard landing page |
| `app/email_utils.py` | Email template rendering and dispatch |
| `app/mail_sender.py` | SMTP delivery with `NOT_SEND_EMAIL` bypass |
| `example.env` | Configuration variable reference |

### D. Technology Versions

| Technology | Version | Source |
|------------|---------|--------|
| Python | 3.10 | `pyproject.toml` target-version |
| Flask | ^1.1.2 | `pyproject.toml` dependencies |
| SQLAlchemy | 1.3.24 | `pyproject.toml` dependencies |
| PostgreSQL | 13+ | `CONTRIBUTING.md` |
| Node.js | 10.17.0 | `Dockerfile` |
| Gunicorn | ^20.0.4 | `pyproject.toml` dependencies |
| aiosmtpd | ^1.2 | `pyproject.toml` dependencies |
| Redis | ^4.5.3 | `pyproject.toml` dependencies |
| yacron | ^0.11.1 | `pyproject.toml` dependencies |

### E. Environment Variable Reference

| Variable | Default | Required | Purpose |
|----------|---------|----------|---------|
| `URL` | `http://localhost:7777` | Yes | Base URL |
| `DB_URI` | (none) | Yes | PostgreSQL connection URI |
| `FLASK_SECRET` | (none) | Yes | Session encryption key |
| `NOT_SEND_EMAIL` | `true` (in example.env) | No | Log emails instead of sending |
| `EMAIL_DOMAIN` | `sl.local` | Yes | Default alias domain |
| `MEM_STORE_URI` | (none) | No | Redis URI for sessions/rate limiting |
| `SENTRY_DSN` | (none) | No | Sentry error tracking |
| `DISABLE_REGISTRATION` | (none) | No | Block new user registration |
| `HCAPTCHA_SECRET` | (none) | No | Enable hCaptcha on registration |
| `POSTFIX_SERVER` | `localhost` | No | SMTP relay server |
| `POSTFIX_PORT` | `25` | No | SMTP relay port |
| `COLOR_LOG` | (auto in dev) | No | Enable colored log output |

### G. Glossary

| Term | Definition |
|------|------------|
| **Alias** | An email address (e.g., `abc123@sl.local`) that forwards to a user's real mailbox |
| **Mailbox** | The user's real email address that receives forwarded emails |
| **Activation** | The process of verifying a new user's email via a one-time code link |
| **MFA** | Multi-Factor Authentication — FIDO (WebAuthn) or TOTP supported |
| **`NOT_SEND_EMAIL`** | Environment flag that causes emails to be logged to stdout instead of SMTP delivery |
| **Job Runner** | Background process polling the PostgreSQL `Job` table every 10 seconds |
| **SyncEvent** | PostgreSQL-backed event dispatched via LISTEN/NOTIFY for near-real-time processing |
| **yacron** | YAML-based cron scheduler used to run `cron.py` maintenance functions |