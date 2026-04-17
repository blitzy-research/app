# Blitzy Project Guide — SimpleLogin Runtime Behavior Analysis

**Project:** SimpleLogin `app_2cd6ee777f8c` Runtime Behavior Analysis
**Branch:** `blitzy-80e37e93-fa44-4779-a4db-4740917159a5`
**Generated:** April 17, 2026

---

## 1. Executive Summary

### 1.1 Project Overview

This project is an observational runtime analysis of the SimpleLogin email privacy system (an open-source email alias solution). Per the user's explicit constraint that **no source files may be modified**, the deliverable is a single new Markdown document, `blitzy/documentation/app_2cd6ee777f8c.md`, that answers five categories of runtime-behavior questions: (1) Alembic database initialization, (2) Gunicorn/Flask web-server startup, (3) aiosmtpd email-handler startup with a custom port, (4) the API-driven user register→activate→login lifecycle, and (5) dynamic propagation of the `MAX_NB_EMAIL_FREE_PLAN` environment variable. Every answer is backed by captured runtime output plus source-file citations with exact line numbers.

### 1.2 Completion Status

```mermaid
pie title Project Completion — 92.3% Complete
    "Completed Work (24h)" : 24
    "Remaining Work (2h)" : 2
```

**Legend:** Completed Work = Dark Blue (#5B39F3) · Remaining Work = White (#FFFFFF)

| Metric | Value |
|--------|-------|
| **Total Project Hours** | 26 |
| **Completed Hours (AI + Manual)** | 24 |
| **Remaining Hours** | 2 |
| **Completion Percentage** | 92.3% |

**Calculation:** 24 / (24 + 2) × 100 = **92.3%**

All work in scope per the Agent Action Plan has been completed and autonomously verified. The remaining 2 hours represent the human review-and-acceptance pass required before considering the deliverable formally accepted.

### 1.3 Key Accomplishments

- ✅ **1,193-line Markdown deliverable** created at `blitzy/documentation/app_2cd6ee777f8c.md` with answers, runtime evidence, and source-code rationale for every question in the AAP
- ✅ **Experiment 1 — Alembic migrations**: 255 migration scripts applied cleanly on a freshly created empty PostgreSQL 16 database; final count of **77 tables** verified via `SELECT COUNT(*) FROM pg_tables WHERE schemaname='public'`; last table-creating migration identified as `7d7b84779837` (`user_audit_log`)
- ✅ **Experiment 2 — Gunicorn startup**: Exact ready-message captured (`[INFO] Listening at: http://0.0.0.0:7777 (PID)`); elapsed time between first log and ready log documented as **0 ms at Gunicorn's second-level timestamp resolution**
- ✅ **Experiment 3 — Email handler**: `python email_handler.py -p 25025` accepts the custom port via argparse; **INFO** log `Listen for port 25025` and **DEBUG** log `Start mail controller 0.0.0.0 25025` both captured and verified
- ✅ **Experiment 4 — User lifecycle**: Register → DB inspection (`activated=f, notification=t, flags=1`) → Login **HTTP 422 UNPROCESSABLE ENTITY** with body `{"error":"Account not activated"}` → Activate → successful login with `api_key`
- ✅ **Experiment 5 — Dynamic configuration**: `MAX_NB_EMAIL_FREE_PLAN=10` restart propagates to **both** existing and newly-created users (demonstrating runtime read from `config.MAX_NB_EMAIL_FREE_PLAN`, not per-user DB storage)
- ✅ **All source citations verified**: Every file path and line number in the rationale sections was cross-checked against the actual repository at HEAD
- ✅ **QA finding resolved**: Amended Section 5.4 to correctly describe the `users.flags` default as `1` (not `0`) — the Python ORM default `FLAG_DISABLE_CREATE_CONTACTS` takes precedence over the SQL `server_default="0"` at `User.create(...)` time
- ✅ **Test suite validated**: 634/639 passing (99.2%), matching the documented baseline from the setup instructions
- ✅ **No source files modified**; working tree clean; 2 commits pushed to origin

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Document requires human review for acceptance | Low — deliverable is already runtime-verified; this is a formal acceptance step | Human reviewer | 1.5 hours |
| User Q&A alignment verification | Low — confirms the 14 tabulated answers address the user's intent | Human reviewer | 0.5 hours |

No blocking issues. The 5 pre-existing `tests/test_mail_sender.py` failures documented in the setup instructions are container-level IPv6 infrastructure limitations (Errno 99 on `::1` bind) and are explicitly out of scope — they cannot be fixed without modifying source code (which the AAP prohibits) or reconfiguring the container's IPv6 loopback.

### 1.5 Access Issues

No access issues identified. All resources required for the investigation were accessible:

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| PostgreSQL 16 (port 15432) | Read/Write | None | ✅ Operational | N/A |
| Redis 7.x (port 6379) | Read/Write | None | ✅ Operational | N/A |
| Python 3.10.20 venv (`/opt/sl_venv/`) | Execution | None | ✅ Operational | N/A |
| Source repository | Read-only (per AAP) | None | ✅ As intended | N/A |
| GitHub branch push | Read/Write | None | ✅ Both commits pushed | N/A |

### 1.6 Recommended Next Steps

1. **[High]** Human reviewer opens `blitzy/documentation/app_2cd6ee777f8c.md` and verifies the 14 tabulated answers match the user's question intent (≈1.5 hours)
2. **[High]** Reviewer spot-checks 2–3 source citations (e.g., `app/config.py:120-124`, `email_handler.py:2397-2404`) against the actual repository at HEAD (≈0.5 hours)
3. **[Medium]** If any follow-up questions emerge, run the corresponding experiment section of the document's reproduction commands (already validated in the Development Guide below) and capture a fresh runtime output
4. **[Low]** Optionally, include a project-local `.env` file (derived from `example.env`) in a future contributor onboarding pack — noting that this file is intentionally gitignored
5. **[Low]** If the container's IPv6 loopback is ever repaired, the 5 pre-existing `test_mail_sender.py` failures should clear on their own

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| [AAP] Environment setup & dependency verification | 1.5 | Verified PostgreSQL 16 (port 15432), Redis 7.x (port 6379), Python 3.10.20 venv at `/opt/sl_venv/`, Gunicorn 20.0.4, alembic 1.4.3, aiosmtpd 1.4.2; confirmed all tools on PATH |
| [AAP] Experiment 1 — Alembic migration analysis | 3.0 | Dropped/recreated `test` database; ran `alembic upgrade head` on empty DB; counted 255 `Running upgrade` log lines; verified 77 tables via `pg_tables`; identified `user_audit_log` as last table-creating migration; resolved pg_trgm/rollback edge case documented in Issue Resolution section |
| [AAP] Experiment 2 — Gunicorn startup timing | 1.5 | Launched `gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60`; captured full startup log; confirmed `[INFO] Listening at: http://0.0.0.0:7777 (PID)` as ready indicator; measured 0 ms delta at second-level timestamp resolution |
| [AAP] Experiment 3 — Email handler custom port | 1.0 | Launched `python email_handler.py -p 25025`; confirmed INFO log `Listen for port 25025` at line 2403 and DEBUG log `Start mail controller 0.0.0.0 25025` at line 2386; verified port 25025 listening via socket check |
| [AAP] Experiment 4 — User lifecycle API testing | 3.5 | `POST /api/auth/register` (HTTP 200, `{"msg":"User needs to confirm their account"}`); psql inspection (`activated=f, notification=t, flags=1`); `POST /api/auth/login` pre-activation (HTTP 422, `{"error":"Account not activated"}`); `POST /api/auth/activate` with code from `account_activation` table; successful post-activation login with `api_key`; full verbose curl output captured |
| [AAP] Experiment 5 — Dynamic configuration experiment | 3.5 | Baseline `max_alias_free_plan=5` captured; killed Gunicorn; restarted with `MAX_NB_EMAIL_FREE_PLAN=10`; verified existing user's `/api/user_info` now returns 10 (runtime read, not per-user); created second user and verified 10 for new users; explained via `app/config.py:120-124`, `app/models.py:858-865`, `app/api/views/user_info.py:34` |
| [AAP] Documentation authoring (1,193 lines) | 8.0 | Drafted the Markdown deliverable with 5 experiment sections, command blocks, full captured runtime logs, exact JSON responses, source-code rationale with file:line citations, and a consolidated summary table of 14 verified answers |
| [AAP] QA, re-validation, citation verification | 2.0 | Cross-checked every `file.py:line` citation against HEAD; resolved QA finding #1 (flags default is 1, not 0 — Python default precedence over SQL server_default documented in detail); re-ran all 5 experiments and test suite to confirm parity with documentation |
| **Total Completed Hours** | **24.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| [Path-to-production] Human review of 1,193-line deliverable for correctness and completeness | 1.5 | High |
| [Path-to-production] User-acceptance confirmation that the 14 tabulated answers address the question intent | 0.5 | High |
| **Total Remaining Hours** | **2.0** | |

### 2.3 Notes

- The AAP explicitly scopes the project to a **single documentation deliverable**. There is no application code to ship, integrate, or deploy. Production-readiness here means the Markdown file has been validated and is ready for the user's review.
- Verification: **2.1 Completed (24h) + 2.2 Remaining (2h) = 26h = Total Project Hours in Section 1.2** ✓

---

## 3. Test Results

All test results below originate from Blitzy's autonomous validation logs. The test suite was executed using the command documented in the setup instructions (`GITHUB_ACTIONS_TEST=true pytest --no-cov -q`) and the results match the documented baseline exactly.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Application unit + integration (full suite) | pytest 7.x | 639 | 634 | 5 | Not computed (per `pytest.ci.ini` `--no-cov`) | Executed in 172.19 s. The 5 failures are all in `tests/test_mail_sender.py` and share root cause `OSError: [Errno 99] Cannot assign requested address` — a container-level IPv6 loopback limitation (`localhost` resolves to `::1` which the container cannot bind). This is the documented baseline in the setup instructions, not a regression from this project. |
| Documentation authoring — Experiment 1 (Alembic migration) | psql + alembic | 1 | 1 | 0 | N/A | `alembic upgrade head` applied 255 migrations cleanly; `SELECT COUNT(*) FROM pg_tables` returned **77** — matches documentation claim verbatim |
| Documentation authoring — Experiment 2 (Gunicorn startup) | Manual log capture | 1 | 1 | 0 | N/A | Captured ready message `[INFO] Listening at: http://0.0.0.0:7777 (PID)` at same-second timestamp as `Starting gunicorn 20.0.4` — matches documentation verbatim |
| Documentation authoring — Experiment 3 (Email handler) | Manual log capture + socket probe | 2 | 2 | 0 | N/A | INFO line `Listen for port 25025` and DEBUG line `Start mail controller 0.0.0.0 25025` both captured; port 25025 verified listening via socket check |
| Documentation authoring — Experiment 4 (User lifecycle) | curl + psql | 5 | 5 | 0 | N/A | Register → DB inspect → pre-activation login (HTTP 422) → activate → post-activation login — all 5 steps produced exactly the responses claimed in the document |
| Documentation authoring — Experiment 5 (Dynamic config) | curl + gunicorn restart | 4 | 4 | 0 | N/A | Baseline `max_alias_free_plan=5` confirmed; restart with `MAX_NB_EMAIL_FREE_PLAN=10` confirmed; existing user shows 10; new user shows 10 |
| **Total (all autonomous tests)** | | **652** | **647** | **5** | — | **99.2 % pass rate** — all in-scope claims verified; all 5 failures are out-of-scope container IPv6 infrastructure issues |

**Key integrity rule:** All 5 experiment verifications above were executed live by Blitzy during Final Validator's run and produced output byte-for-byte matching the deliverable document. No mocked or stubbed responses were used.

---

## 4. Runtime Validation & UI Verification

This project has no UI component — it is a documentation deliverable. The runtime validation focuses on the five subsystems documented in the deliverable. All five were re-executed live against the running stack during Final Validator's run and matched the documentation verbatim.

**Subsystem Runtime Status:**

- ✅ **Operational** — PostgreSQL 16.13 on port 15432 (user=`test`, password=`test`, database=`test`). Verified via `psql -c "SELECT COUNT(*) FROM pg_tables..." = 77`.
- ✅ **Operational** — Redis 7.x on port 6379. Verified via `redis-cli PING = PONG`.
- ✅ **Operational** — Gunicorn 20.0.4 serving `wsgi:app` on `0.0.0.0:7777`. Verified via HTTP `200 OK` from `POST /api/auth/register` with response body `{"msg":"User needs to confirm their account"}`.
- ✅ **Operational** — aiosmtpd Controller on `0.0.0.0:25025`. Verified via successful bind and INFO/DEBUG log emission.
- ✅ **Operational** — Alembic 1.4.3 migration runner against PostgreSQL 16. Verified via clean head-revision upgrade of 255 scripts, resulting in 77 tables.
- ✅ **Operational** — Flask API surface (`/api/auth/register`, `/api/auth/login`, `/api/auth/activate`, `/api/user_info`). Every endpoint returned the exact status code and JSON body documented.

**API Integration Outcomes (live responses captured):**

- ✅ `POST /api/auth/register` → HTTP 200, body `{"msg":"User needs to confirm their account"}`
- ✅ `POST /api/auth/login` (before activation) → HTTP 422 `UNPROCESSABLE ENTITY`, body `{"error":"Account not activated"}`
- ✅ `POST /api/auth/activate` → HTTP 200, body `{"msg":"Account is activated, user can login now"}`
- ✅ `POST /api/auth/login` (after activation) → HTTP 200 with `api_key`, `email`, `name`, `mfa_enabled`, `mfa_key`
- ✅ `GET /api/user_info` (default config) → HTTP 200 with `"max_alias_free_plan": 5`
- ✅ `GET /api/user_info` (with `MAX_NB_EMAIL_FREE_PLAN=10`) → HTTP 200 with `"max_alias_free_plan": 10` for **both** existing and new users

**Database Integrity Verification (live `psql` results):**

- ✅ `users.activated` defaults to `f` (False) on fresh registration — matches documentation
- ✅ `users.notification` defaults to `t` (True) via `server_default="1"` — matches documentation
- ✅ `users.flags` defaults to `1` (=`FLAG_DISABLE_CREATE_CONTACTS`) via Python ORM default — matches documentation after QA fix
- ✅ `account_activation` table contains a 6-digit numeric code keyed to the new user's ID — matches documentation

---

## 5. Compliance & Quality Review

The AAP for this project is observational-and-documentation-only, with a single firm constraint: **no source files may be modified**. All compliance checks below are mapped to that constraint and to the standard Blitzy quality benchmarks.

| AAP Requirement / Quality Benchmark | Status | Evidence |
|--------------------------------------|--------|----------|
| **No source files modified** | ✅ Pass | `git log --author="agent@blitzy.com" --stat` shows exactly 2 commits, both modifying only `blitzy/documentation/app_2cd6ee777f8c.md`. `git status` shows working tree clean. |
| **Deliverable at correct path** | ✅ Pass | File exists at `blitzy/documentation/app_2cd6ee777f8c.md` matching the source branch name `app_2cd6ee777f8c` as prescribed in Rule §0.7.1 |
| **No new files outside the single deliverable** | ✅ Pass | Only `blitzy/documentation/app_2cd6ee777f8c.md` was added by agent commits |
| **Every answer backed by runtime output (not code-reading)** | ✅ Pass | Each of the 5 experiment sections includes captured stdout/stderr, curl verbose output, or psql result blocks |
| **Rationale sections with file:line citations** | ✅ Pass | Verified every citation: `app/config.py:120-124`, `app/models.py:339`, `:354-356`, `:358`, `:545-550`, `:858-865`, `app/api/views/auth.py:29-30,75-77,87-88,125-131,141`, `app/api/views/user_info.py:34`, `email_handler.py:2381-2386,2397-2404` |
| **Exact values (not approximations)** | ✅ Pass | All 14 tabulated answers are exact strings, integers, or status codes — no "approximately", "around", or ranged values |
| **Dynamic configuration experiment documents before-and-after** | ✅ Pass | §5.1, §5.2, §5.3 capture pre-restart (5), post-restart existing user (10), and post-restart new user (10) |
| **Full verbose curl output for login attempt (as requested)** | ✅ Pass | §4.3 includes `curl -v` headers, status line, connection info, and body |
| **Setup-instructions-compliant test baseline** | ✅ Pass | 634/639 tests pass = exactly the documented baseline; no new regressions; 5 failures are pre-existing IPv6 limitations |
| **Both commits pushed to origin** | ✅ Pass | `git rev-list HEAD ^origin/blitzy-80e37e93-fa44-4779-a4db-4740917159a5 --count = 0` |
| **QA finding resolution (flags default = 1)** | ✅ Pass | Commit `c34a7a10` amends §5.4 with full explanation of Python-default-wins-over-server_default at ORM INSERT time; runtime verification `flags=1` included |

**Fixes applied during autonomous validation:**

- **QA finding #1 (MINOR)** — Original doc claimed `flags` column is `0` by default; runtime check showed `flags=1`. Root cause documented: `app/models.py:545-550` declares `flags` with Python default `FLAG_DISABLE_CREATE_CONTACTS = 1` AND SQL `server_default="0"`. SQLAlchemy supplies the Python default at `INSERT` time during `User.create(...)`, so the SQL default is not used. Fix amended Section 5.4 to reflect this correctly and preserve the downstream bitwise conclusion (`1 & 4 = 0` → `else` branch taken).
- **Accidental modification** — `bash scripts/generate-build-info.sh "test"` overwrote `app/build_info.py` during setup verification. Setup instructions explicitly state this file is tracked with dummy values and must not be committed with regenerated values. Reverted via `git checkout -- app/build_info.py`; working tree is now clean.

**Outstanding quality items:** None. All in-scope work is complete.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Setup instructions' container IPv6 limitation causes 5 `test_mail_sender.py` failures | Technical | Low | N/A — already present | Documented as expected baseline; AAP prohibits source modification that could repair this; requires container IPv6 loopback configuration outside repo scope | ✅ Accepted (pre-existing, out of scope) |
| pg_trgm extension + migration rollback edge case could destroy schema during DB re-creation | Technical | Medium (if encountered) | Low — only triggers if pg_trgm pre-created before migration | Documented in agent logs: drop and recreate DB **without** pre-creating pg_trgm; let the migration's own `CREATE EXTENSION pg_trgm` handle creation | ✅ Mitigated (procedural note in Development Guide) |
| Source citations could drift if reviewer checks against HEAD after future upstream merges | Technical | Low | Medium over long time periods | Each citation is tied to a specific commit hash (HEAD at time of authoring: `c34a7a10`). Reviewer should check out the branch or that commit before verifying citations | ✅ Documented |
| MAX_NB_EMAIL_FREE_PLAN runtime-only resolution could be surprising to operators who expect per-user storage | Operational | Low | Low — behavior is by design per SimpleLogin's source code | §5.4 of the deliverable explains the architectural choice and its implications explicitly; includes full source trace | ✅ Clearly documented |
| Documentation requires human acceptance review before formal sign-off | Operational | Low | High — standard human gate | Scheduled in §2.2 as 2 hours of remaining path-to-production work | ⚠ Pending human action |
| A `.env` file was created locally for experimentation but is gitignored | Security | Low | Low — secrets in the file are test-only credentials (user=`test`, password=`test`) matching `tests/test.env` | Test credentials are identical to repository-committed `tests/test.env`; local `.env` is gitignored and not in the commit | ✅ No leak |
| No authentication or authorization changes introduced | Security | None | N/A | Project is documentation-only | ✅ No change |
| No new external integrations introduced | Integration | None | N/A | Project is documentation-only | ✅ No change |
| SimpleLogin's `FLAG_DISABLE_CREATE_CONTACTS=1` default on new users may affect contact-creation behavior (unrelated to this project but noted during QA) | Integration | Low | High — all new users are affected | Documented in §5.4 rationale; not in scope to change, and AAP explicitly prohibits code modification | ✅ Out of scope |
| The 5 IPv6 test failures will re-appear on every test run until container networking is fixed | Operational | Low | High (guaranteed every run) | Expected; documented as baseline in the setup instructions | ✅ Documented baseline |

**Risk posture summary:** No high-severity risks. All medium/low risks are either pre-existing or out-of-scope per the AAP. The project's narrow observational scope minimizes the risk surface.

---

## 7. Visual Project Status

### Project Hours Breakdown

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 24
    "Remaining Work" : 2
```

**Legend:** Completed Work = Dark Blue (#5B39F3) · Remaining Work = White (#FFFFFF)

### Remaining Work Distribution

```mermaid
pie title Remaining Work by Category
    "Human document review" : 1.5
    "User-acceptance confirmation" : 0.5
```

### AAP Experiment Completion Status

```mermaid
pie title AAP Experiments — All 5 Completed
    "Experiment 1 — Alembic" : 3
    "Experiment 2 — Gunicorn" : 1.5
    "Experiment 3 — Email Handler" : 1
    "Experiment 4 — User Lifecycle" : 3.5
    "Experiment 5 — Dynamic Config" : 3.5
    "Documentation + QA + Setup" : 11.5
```

**Cross-reference integrity:**
- Section 1.2 Remaining Hours = **2** ✓
- Section 2.2 Hours column sum = 1.5 + 0.5 = **2** ✓
- Section 7 pie chart "Remaining Work" value = **2** ✓
- Section 1.2 Total Project Hours = Section 2.1 (24) + Section 2.2 (2) = **26** ✓

---

## 8. Summary & Recommendations

### Achievements

The project delivered a single, comprehensive 1,193-line Markdown document (`blitzy/documentation/app_2cd6ee777f8c.md`) that answers every question posed in the AAP with runtime-verified evidence and source-code rationale. All five experiments — database migration, web-server startup, email-handler port, user lifecycle, and dynamic configuration — were executed live against a freshly initialized SimpleLogin stack (PostgreSQL 16.13, Redis 7.x, Python 3.10.20, Gunicorn 20.0.4, aiosmtpd 1.4.2). The 14 tabulated answers (77 tables, `user_audit_log` last, `[INFO] Listening at: http://0.0.0.0:7777 (PID)` ready message, 0 ms timestamp delta, `Listen for port 25025` INFO log, `Start mail controller 0.0.0.0 25025` DEBUG log, HTTP 422 with `{"error":"Account not activated"}`, etc.) each carry verbatim-captured runtime evidence plus file-and-line citations to the source code that produces them.

### Remaining Gaps

There are no technical or implementation gaps. The 2.0 hours of remaining work consist entirely of the human review-and-acceptance pass standard for every Blitzy deliverable: (1) a reviewer reads the document and verifies it addresses the user's question intent (~1.5 h), and (2) the reviewer spot-checks 2–3 source citations against HEAD to confirm citation integrity (~0.5 h).

### Critical Path to Production

Because this project's production artefact is a Markdown document rather than executable code, the critical path is short:

1. Human reviewer opens `blitzy/documentation/app_2cd6ee777f8c.md` (~15 min)
2. Reviewer skim-reads the Summary of Findings table and confirms all 14 answers are exact values (~15 min)
3. Reviewer samples 2–3 experiment sections to verify the captured runtime logs match the answer values (~30 min)
4. Reviewer spot-checks 2–3 source citations against HEAD (e.g., `app/config.py:120-124`) (~30 min)
5. Reviewer confirms the user's original questions are all addressed in the Table of Contents (~15 min)

Total critical-path time from current state to acceptance: **~1.75 hours**.

### Success Metrics

- ✅ **100 %** of AAP experiments completed and runtime-verified
- ✅ **14 / 14** tabulated answers match captured runtime output byte-for-byte
- ✅ **0** source files modified (AAP hard constraint satisfied)
- ✅ **634 / 639** tests pass (99.2 %), matching documented baseline exactly
- ✅ **All** source-code citations validated against HEAD
- ✅ **2** commits pushed to origin (`109c5a79`, `c34a7a10`)
- ✅ **1** QA finding identified and resolved (flags default = 1)
- ✅ **0** unresolved errors in the working tree

### Production Readiness Assessment

The deliverable is production-ready per the AAP's narrow observational scope. **Completion: 92.3 %** — the remaining 7.7 % represents the human review-and-acceptance pass that is standard for every Blitzy deliverable and is never autonomously completed. No blocking issues exist. The document can be shipped to the user as-is and is expected to require no additional changes based on the rigor of the autonomous validation already performed.

---

## 9. Development Guide

This section documents how to reproduce every runtime experiment in the deliverable document from a clean container state. All commands were validated during Final Validator's run.

### 9.1 System Prerequisites

| Component | Required Version | Purpose |
|-----------|------------------|---------|
| Operating system | Linux (Ubuntu 22.04+ recommended) | Host for PostgreSQL, Redis, and Python |
| Python | 3.10.x (exactly — Dockerfile and CI matrix pin 3.10) | Application runtime |
| PostgreSQL | 16.x | Primary relational database |
| Redis | 7.x | Cache, rate-limit store, session backend |
| `psql` client | matches server | DB introspection for experiments |
| `curl` | any recent version | API testing in experiments |
| Memory | 2 GB minimum | Gunicorn worker + PostgreSQL + Redis |
| Disk | 500 MB minimum | Code + dependencies + test DB |

The provided container image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0`) already contains all prerequisites pre-installed.

### 9.2 Environment Setup

```bash
# Navigate to the repository root (provided working directory)
cd /tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0

# Activate the pre-built Python 3.10.20 virtual environment with all Poetry-exported dependencies
source /opt/sl_venv/bin/activate

# Verify tool versions
python3 --version         # Expected: Python 3.10.20
gunicorn --version        # Expected: gunicorn (version 20.0.4)
alembic --version         # Expected: alembic 1.4.3
```

Ensure PostgreSQL and Redis are running:

```bash
# PostgreSQL 16 cluster on non-default port 15432 (per project convention)
service postgresql start 2>/dev/null || pg_ctlcluster 16 main start
service postgresql status   # Expected: "16/main (port 15432): online"

# Redis on default port 6379
redis-server --daemonize yes 2>/dev/null
redis-cli PING              # Expected: PONG
```

### 9.3 Dependency Installation

Dependencies are pre-installed in `/opt/sl_venv/`. To re-install from scratch (not normally required):

```bash
# Export Poetry lockfile to pip-compatible requirements
poetry export -f requirements.txt --output /tmp/requirements.txt --without-hashes

# Install into the venv
pip install -r /tmp/requirements.txt

# The aiosmtpd, gunicorn, alembic, flask, sqlalchemy, psycopg2-binary, redis,
# bcrypt, email_validator, and other key packages are all pulled in.
```

### 9.4 Application Startup Sequence

The five experiments can be run in any order, but Experiment 1 (migrations) must run before any API experiments that require DB tables.

#### 9.4.1 Prepare a clean database (required before all experiments)

```bash
# Drop and recreate the test database WITHOUT pre-creating pg_trgm.
# CRITICAL: Do not pre-create the pg_trgm extension. Let migration 424808e1fe49
# create it — pre-creating triggers a rollback path that destroys prior schema.
PGPASSWORD=test psql -h localhost -p 15432 -U test -d postgres \
  -c "DROP DATABASE IF EXISTS test;"
PGPASSWORD=test psql -h localhost -p 15432 -U test -d postgres \
  -c "CREATE DATABASE test OWNER test;"

# Apply all 255 migrations
alembic upgrade head
# Expected tail output:
# INFO  [alembic.runtime.migration] Running upgrade 62afa3a10010 -> 91ed7f46dc81, alias_audit_log
# INFO  [alembic.runtime.migration] Running upgrade 91ed7f46dc81 -> 7d7b84779837, user_audit_log
# INFO  [alembic.runtime.migration] Running upgrade 7d7b84779837 -> 32f25cbf12f6, alias_audit_log_index_created_at

# Verify 77 tables
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test \
  -c "SELECT COUNT(*) FROM pg_tables WHERE schemaname='public';"
# Expected: 77
```

#### 9.4.2 Start the Gunicorn web server

```bash
# Use -w 1 to minimize log noise in experiments; production uses -w 2 per Dockerfile
gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60
# Expected first logs:
# [YYYY-MM-DD HH:MM:SS +0000] [PID] [INFO] Starting gunicorn 20.0.4
# [YYYY-MM-DD HH:MM:SS +0000] [PID] [INFO] Listening at: http://0.0.0.0:7777 (PID)
# [YYYY-MM-DD HH:MM:SS +0000] [PID] [INFO] Using worker: sync
# [YYYY-MM-DD HH:MM:SS +0000] [PID] [INFO] Booting worker with pid: <worker_pid>
```

To start Gunicorn with a custom alias-limit (Experiment 5):

```bash
pkill -f "gunicorn wsgi:app"   # if already running
(MAX_NB_EMAIL_FREE_PLAN=10 gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60 \
  > /tmp/gunicorn_restart.log 2>&1 &)
```

#### 9.4.3 Start the email handler

```bash
python email_handler.py -p 25025
# Expected logs:
# 2026-XX-XX HH:MM:SS,XXX - SL - INFO  - PID - ".../email_handler.py:2403" - <module>() -  - Listen for port 25025
# 2026-XX-XX HH:MM:SS,XXX - SL - DEBUG - PID - ".../email_handler.py:2386" - main() -  - Start mail controller 0.0.0.0 25025
```

### 9.5 Verification Steps

#### 9.5.1 Verify Experiment 4 — User lifecycle

```bash
# Register
curl -v -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123"}'
# Expected: HTTP 200 with body {"msg":"User needs to confirm their account"}

# DB state (activated should be f, notification t, flags 1)
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test \
  -c "SELECT activated, notification, flags FROM users WHERE email='testuser@example.com';"

# Attempt login BEFORE activation
curl -v -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123","device":"test-device"}'
# Expected: HTTP 422 UNPROCESSABLE ENTITY with body {"error":"Account not activated"}

# Fetch activation code
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -t -A \
  -c "SELECT code FROM account_activation WHERE user_id=(SELECT id FROM users WHERE email='testuser@example.com');"

# Activate (replace <code> with result above)
curl -s -X POST http://localhost:7777/api/auth/activate \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","code":"<code>"}'
# Expected: {"msg":"Account is activated, user can login now"}

# Login AFTER activation — capture api_key for Experiment 5
curl -s -X POST http://localhost:7777/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123","device":"test-device"}'
# Expected: HTTP 200 JSON with "api_key" field
```

#### 9.5.2 Verify Experiment 5 — Dynamic configuration

```bash
# Baseline (default config, no MAX_NB_EMAIL_FREE_PLAN env var)
curl -s http://localhost:7777/api/user_info -H "Authentication: <api_key>" | python3 -m json.tool
# Expected: "max_alias_free_plan": 5

# Restart with MAX_NB_EMAIL_FREE_PLAN=10 (see 9.4.2)
# Then re-query:
curl -s http://localhost:7777/api/user_info -H "Authentication: <api_key>" | python3 -m json.tool
# Expected: "max_alias_free_plan": 10 — same api_key, same user, different limit
```

### 9.6 Running the Test Suite

```bash
# Full test suite (expected: 634 pass, 5 IPv6-related failures)
GITHUB_ACTIONS_TEST=true pytest --no-cov -q
# Expected tail:
# FAILED tests/test_mail_sender.py::test_mail_sender_save_unsent_to_disk[closed_dummy_server]
# FAILED tests/test_mail_sender.py::test_mail_sender_save_unsent_to_disk[inner0]
# FAILED tests/test_mail_sender.py::test_mail_sender_save_unsent_to_disk[inner1]
# FAILED tests/test_mail_sender.py::test_send_unsent_email_from_fs
# FAILED tests/test_mail_sender.py::test_failed_resend_does_not_delete_file
# 5 failed, 634 passed, 51 warnings in ~170s
```

### 9.7 Example Usage

Full end-to-end reproduction of all five experiments, top to bottom:

```bash
#!/bin/bash
set -e

# Setup
cd /tmp/blitzy/app/blitzy-80e37e93-fa44-4779-a4db-4740917159a5_f6dbd0
source /opt/sl_venv/bin/activate
service postgresql start 2>/dev/null || pg_ctlcluster 16 main start
redis-server --daemonize yes 2>/dev/null

# Experiment 1 — migrations
PGPASSWORD=test psql -h localhost -p 15432 -U test -d postgres \
  -c "DROP DATABASE IF EXISTS test;"
PGPASSWORD=test psql -h localhost -p 15432 -U test -d postgres \
  -c "CREATE DATABASE test OWNER test;"
alembic upgrade head 2>&1 | tee /tmp/alembic.log
PGPASSWORD=test psql -h localhost -p 15432 -U test -d test \
  -c "SELECT COUNT(*) FROM pg_tables WHERE schemaname='public';"

# Experiment 2 — Gunicorn startup (background; capture log)
(gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60 \
  > /tmp/gunicorn.log 2>&1 &)
sleep 5
head -20 /tmp/gunicorn.log

# Experiment 3 — Email handler (background; capture log)
(python email_handler.py -p 25025 > /tmp/email_handler.log 2>&1 &)
sleep 5
grep -E "Listen for port|Start mail controller" /tmp/email_handler.log

# Experiment 4 — User lifecycle
curl -s -X POST http://localhost:7777/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"testuser@example.com","password":"testpass123"}'

# ... (see 9.5.1 for the full sequence)
```

### 9.8 Troubleshooting

| Problem | Root Cause | Resolution |
|---------|-----------|------------|
| `psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed` | Client defaulting to port 5432; this project uses port 15432 | Always pass `-p 15432` to `psql` |
| `OSError: [Errno 99] Cannot assign requested address` in mail sender tests | Container IPv6 loopback (`::1`) not configured | Pre-existing baseline; documented in setup instructions; not fixable without container changes |
| `alembic` migration fails with `UndefinedTable: relation "alias" does not exist` at migration `424808e1fe49` | pg_trgm was pre-created before running alembic; the migration's rollback path destroyed the earlier schema | Drop and recreate the DB **without** pre-creating pg_trgm — let the migration create it |
| `app/build_info.py` modified after `scripts/generate-build-info.sh` | This script overwrites committed dummy values | `git checkout -- app/build_info.py` to revert; never commit regenerated values |
| `POST /api/auth/login` returns 400 `{"error":"Email or password incorrect"}` instead of 422 | Password does not match (bcrypt verify failed) | Double-check password matches exactly what was sent to `/api/auth/register` |
| `GET /api/user_info` returns 401 | Missing or invalid `Authentication` header | Use exact header name `Authentication` (NOT `Authorization`) per SimpleLogin's convention |
| Gunicorn binding fails with `[CRITICAL] WORKER TIMEOUT` | App worker took longer than `--timeout` to import | Use `--timeout 60` (as in the experiments) instead of production's `--timeout 15` for local dev |

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Activate venv | `source /opt/sl_venv/bin/activate` |
| Start PostgreSQL | `service postgresql start` or `pg_ctlcluster 16 main start` |
| Start Redis | `redis-server --daemonize yes` |
| Drop/recreate test DB | `PGPASSWORD=test psql -h localhost -p 15432 -U test -d postgres -c "DROP DATABASE IF EXISTS test; CREATE DATABASE test OWNER test;"` |
| Apply all migrations | `alembic upgrade head` |
| Count tables | `PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c "SELECT COUNT(*) FROM pg_tables WHERE schemaname='public';"` |
| List tables | `PGPASSWORD=test psql -h localhost -p 15432 -U test -d test -c "\dt"` |
| Start Gunicorn (dev) | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60` |
| Start Gunicorn (prod) | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Start Gunicorn with custom alias limit | `MAX_NB_EMAIL_FREE_PLAN=10 gunicorn wsgi:app -b 0.0.0.0:7777 -w 1 --timeout 60` |
| Start email handler on 25025 | `python email_handler.py -p 25025` |
| Start email handler default | `python email_handler.py` (defaults to port 20381) |
| Run full test suite | `GITHUB_ACTIONS_TEST=true pytest --no-cov -q` |
| Lint (read-only) | `ruff check app/` |
| Format check (read-only) | `black --check app/` |
| Git diff on this branch | `git diff 2cd6ee77...HEAD --stat` |
| Check agent-authored commits | `git log --author="agent@blitzy.com" --oneline` |

### B. Port Reference

| Port | Service | Configured In |
|------|---------|---------------|
| 7777 | Gunicorn (Flask web) | `Dockerfile:44 EXPOSE 7777`; `Dockerfile:47 CMD ["gunicorn","wsgi:app","-b","0.0.0.0:7777",...]` |
| 25025 | aiosmtpd email handler (custom, per AAP §0.1.1) | `email_handler.py -p 25025` on the command line |
| 20381 | aiosmtpd email handler (default) | `email_handler.py:2400 default=20381` |
| 15432 | PostgreSQL 16 (project convention — non-default) | `.env`: `DB_URI=postgresql://test:test@localhost:15432/test`; `tests/test.env` line matches |
| 6379 | Redis 7.x (default) | `.env`: `MEM_STORE_URI=redis://localhost:6379/0` |

### C. Key File Locations

| File / Directory | Purpose |
|------------------|---------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The project's single deliverable** — 1,193-line runtime behavior analysis |
| `wsgi.py` | Gunicorn WSGI entry point (`app = create_app()`) |
| `server.py` | Flask app factory (`create_app()` at line 139) |
| `email_handler.py` | aiosmtpd SMTP handler (`main()` at 2381, `__main__` at 2397) |
| `app/config.py` | `MAX_NB_EMAIL_FREE_PLAN` at lines 120–124 |
| `app/models.py` | `User` model: `notification` (354), `activated` (358), `flags` (545), `max_alias_for_free_account()` (858) |
| `app/api/views/auth.py` | `/auth/register` (87), `/auth/login` (29), `/auth/activate` (187), `"Account not activated"` (77) |
| `app/api/views/user_info.py` | `user_info()` GET/PATCH; `max_alias_free_plan` surfaced at line 34 |
| `alembic.ini` | `script_location = migrations` (line 5) |
| `migrations/env.py` | Alembic env; imports `app.models.Base`, reads `DB_URI` (lines 26–27) |
| `migrations/versions/` | 255 migration scripts; head = `32f25cbf12f6`; last table-creating = `7d7b84779837` (`user_audit_log`) |
| `example.env` | Reference for all env vars (e.g., `MAX_NB_EMAIL_FREE_PLAN` at line ~54) |
| `.env` | Local-only config for experiments (gitignored; not in commit) |
| `pyproject.toml` | Poetry manifest, Python 3.10, all package versions |
| `poetry.lock` | Exact locked versions (including gunicorn 20.0.4, alembic 1.4.3, aiosmtpd 1.4.2) |
| `Dockerfile` | Production build; `CMD` gunicorn on port 7777 with 2 workers, `--timeout 15` |
| `tests/test.env` | Test-fixture env (mirrors the experiment config; port 15432, user `test`) |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10.20 | `/opt/sl_venv/bin/python3 --version` |
| Gunicorn | 20.0.4 | `pyproject.toml`, `poetry.lock` |
| Flask | 1.1.4 | `poetry.lock` |
| SQLAlchemy | 1.3.24 | `pyproject.toml`, `poetry.lock` |
| Alembic | 1.4.3 | `/opt/sl_venv/bin/alembic --version` |
| Flask-Migrate | 2.5.3 | `poetry.lock` |
| psycopg2-binary | 2.9.3 | `poetry.lock` |
| aiosmtpd | 1.4.2 | `poetry.lock` |
| Flask-Login | 0.5.0 | `pyproject.toml` |
| Flask-Limiter | 1.4 | `pyproject.toml` |
| Flask-CORS | 3.0.9 | `pyproject.toml` |
| bcrypt | 3.2.0 | `pyproject.toml` |
| python-dotenv | 0.14.0 | `pyproject.toml` |
| email-validator | 1.1.3 | `poetry.lock` |
| sqlalchemy-utils | 0.36.8 | `pyproject.toml` |
| arrow | 0.16.0 | `pyproject.toml` |
| redis | 4.6.0 | `poetry.lock` |
| sentry-sdk | 2.16.0+ | `pyproject.toml` |
| newrelic | 8.8.0 | `pyproject.toml` |
| yacron | 0.11.2 | `poetry.lock` |
| PostgreSQL | 16.13 | `service postgresql status` |
| Redis | 7.x | `redis-cli PING` |
| pytest | 7.x | `/opt/sl_venv/bin/pytest --version` |

### E. Environment Variable Reference

Variables actively exercised by the five experiments (from `.env` and experiment commands):

| Variable | Value Used | Purpose |
|----------|-----------|---------|
| `URL` | `http://localhost:7777` | Canonical base URL for the app |
| `EMAIL_DOMAIN` | `sl.local` | Domain for alias addresses |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | PostgreSQL connection (non-default port 15432) |
| `FLASK_SECRET` | `secret` (test value only) | Flask session cookie signing key |
| `SUPPORT_EMAIL` | `support@sl.local` | Transactional email from-address |
| `SUPPORT_NAME` | `Son from SimpleLogin` | Transactional email from-name |
| `EMAIL_SERVERS_WITH_PRIORITY` | `[(10, "email.hostname.")]` | MX record for outgoing mail |
| `OPENID_PRIVATE_KEY_PATH` | `local_data/jwtRS256.key` | JWT signing private key |
| `OPENID_PUBLIC_KEY_PATH` | `local_data/jwtRS256.key.pub` | JWT signing public key |
| `WORDS_FILE_PATH` | `local_data/test_words.txt` | Word list for alias generation |
| `NOT_SEND_EMAIL` | `true` | Prevents real SMTP delivery during experiments |
| `MEM_STORE_URI` | `redis://localhost:6379/0` | Redis connection for rate-limit and session store |
| `MAX_NB_EMAIL_FREE_PLAN` | `10` (Experiment 5 only; default is 5) | Max free-plan aliases per user |
| `GITHUB_ACTIONS_TEST` | `true` | Enables CI-mode in pytest (per `pytest.ci.ini`) |

### F. Developer Tools Guide

| Tool | Role in This Project |
|------|----------------------|
| `git` | Branch management, diff analysis, citation verification. Used: `git log --author="agent@blitzy.com"`, `git diff 2cd6ee77...HEAD --stat` |
| `psql` | Direct DB inspection and counting tables, querying users/flags/activation codes. Always pass `-p 15432 -U test -d test` |
| `curl` | API endpoint testing. Used with `-v` for full verbose output in Experiment 4 |
| `alembic` | Migration runner. Key command: `alembic upgrade head` |
| `gunicorn` | WSGI server. Used with `-w 1 --timeout 60` for experiments |
| `pytest` | Test runner with `GITHUB_ACTIONS_TEST=true` and `--no-cov` for CI-mode |
| `pkill` | Process management — `pkill -f "gunicorn wsgi:app"` to restart the server between experiment phases |
| `redis-cli` | Redis health-check — `redis-cli PING` |
| `service postgresql` / `pg_ctlcluster` | PostgreSQL service management |

### G. Glossary

| Term | Definition |
|------|------------|
| **AAP** | Agent Action Plan — the primary directive document defining project scope |
| **alembic_version** | Bookkeeping table auto-created by Alembic to track current head revision (not declared in any migration) |
| **FLAG_DISABLE_CREATE_CONTACTS** | `User.flags` bit 0 (value 1); Python ORM default causing new users to have `flags=1` |
| **FLAG_FREE_OLD_ALIAS_LIMIT** | `User.flags` bit 2 (value 4); when set, user gets `MAX_NB_EMAIL_OLD_FREE_PLAN` (15) instead of `MAX_NB_EMAIL_FREE_PLAN` (5 default, 10 in Experiment 5) |
| **Gunicorn arbiter** | The master Gunicorn process that binds the socket and forks workers. The "Listening at:" log is emitted by the arbiter. |
| **max_alias_for_free_account()** | `User` instance method at `app/models.py:858`; reads `config.MAX_NB_EMAIL_FREE_PLAN` at request time (not cached) |
| **MAX_NB_EMAIL_FREE_PLAN** | Integer env var; default 5; controls free-plan alias limit; read-once at worker startup and referenced at request time from `app.config` module |
| **pg_trgm** | PostgreSQL trigram extension; created by migration `424808e1fe49`; pre-creating it triggers a destructive rollback path |
| **server_default="1"** | SQL column default; applied only when ORM does not supply a value at INSERT time. For `flags`, the Python default wins over `server_default="0"` |
| **yacron** | YAML-based cron scheduler used in `crontab.yml`; defines 13+ recurring background jobs |
| **wsgi:app** | Gunicorn's import target: `wsgi.py` module, `app` name (Flask app instance created by `server.create_app()`) |
| **Path-to-production work** | Standard activities required to formally release the AAP deliverables (reviews, acceptance) beyond the AAP-listed work items |
