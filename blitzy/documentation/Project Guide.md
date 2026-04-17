# Blitzy Project Guide — SimpleLogin Runtime Verification Q&A

**Branch:** `blitzy-034c517c-e9a0-43d1-be2c-d8d25b8c6f37`
**Source Branch (base):** `app_2cd6ee777f8c` (commit `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`)
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (1,273 lines)
**Rule Applied:** `SWE-AtlasQnA-Repo` (documentation-only, no source modifications)

---

## 1. Executive Summary

### 1.1 Project Overview

This project produced a single comprehensive markdown Q&A document (`blitzy/documentation/app_2cd6ee777f8c.md`) that answers four operational questions about the self-hosted SimpleLogin application. The target audience is a new self-hoster who needs to understand how to verify that the three core subsystems (web server, email handler, job runner) are functioning, what log messages and dashboard UI states confirm operational health, what to expect when performing typical user actions (registration, alias creation, email receipt), and whether the background components auto-start with the web server. The document is 1,273 lines with every claim anchored by a specific `file_path:line_number` citation or a verbatim runtime log excerpt captured from live processes in a Docker sandbox. No source code was modified per the `SWE-AtlasQnA-Repo` rule.

### 1.2 Completion Status

```mermaid
pie title Project Completion Status (95% Complete)
    "Completed Work" : 19
    "Remaining Work" : 1
```

| Metric | Hours |
|---|---|
| **Total Hours** | **20** |
| Completed Hours (Blitzy AI Agents) | 19 |
| Completed Hours (Manual Work To Date) | 0 |
| **Remaining Hours** | **1** |
| **Completion** | **95%** |

**Calculation:** 19 completed / (19 completed + 1 remaining) = **95.0%**

### 1.3 Key Accomplishments

- [x] Sole AAP-mandated deliverable (`blitzy/documentation/app_2cd6ee777f8c.md`) created, 1,273 lines with zero source code modifications
- [x] All 4 operational questions from the user prompt comprehensively answered with evidence
- [x] Every source reference verified against actual codebase — `server.py`, `wsgi.py`, `Dockerfile`, `email_handler.py`, `job_runner.py`, `app/log.py`, `app/auth/views/login.py`, `app/auth/views/login_utils.py`, `app/dashboard/views/index.py`, `templates/dashboard/index.html`, `app/auth/views/register.py`, `app/api/views/new_random_alias.py`, `app/api/base.py`, `app/fake_data.py`, `app/models.py`, `app/config.py`, `cron.py`, `event_listener.py` — all line numbers match exactly
- [x] Runtime verification executed on 2026-04-17: all three core processes (Gunicorn web server, aiosmtpd email handler, job runner) started, exercised, and verified
- [x] End-to-end email forwarding lifecycle captured: 19-line log trail from `set_message_id` → `_handle()` banner → `New message` → `Forward phase` → `Create or get contact` → `Forward` → `Create EmailLog` → `Replace To/Delete Cc` → `Forward mail from X to Y` → `250 Message accepted for delivery`
- [x] API verification: `POST /api/alias/random/new` created `noodle_gunned315@sl.local` (HTTP 201); `GET /api/v2/aliases?page_id=0` confirmed `nb_forward=1` after test email
- [x] Job runner active dispatcher behavior proven: enqueued probe job (id=4) drained within 10s with `Take job` log output
- [x] 9 code review findings addressed across 2 follow-up commits to enforce verbatim source fidelity
- [x] Complete cleanup verified: alias id=22, contact id=9, email_log id=8, job id=4 all `COUNT(*)=0`; baseline seed data intact (2 users, 11 aliases, 1 contact, 4 mailboxes, 2 API keys)
- [x] All 3 commits authored by `Blitzy Agent <agent@blitzy.com>`

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| No critical unresolved issues | All AAP requirements completed; document verified accurate against source code and live runtime evidence | N/A | N/A |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| No access issues identified | — | The SWE-AtlasQnA-Repo task was fully executable with the provided Docker sandbox image (`andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c...`) and pre-configured PostgreSQL/Redis instances. All runtime verification completed successfully. | N/A | N/A |

### 1.6 Recommended Next Steps

1. **[High]** Read the final Q&A document at `blitzy/documentation/app_2cd6ee777f8c.md` end-to-end (approx. 15–20 minutes) and confirm it comprehensively addresses your operational questions about SimpleLogin startup verification, log/dashboard signals, user action walkthroughs, and background process auto-startup behavior.
2. **[Medium]** Optionally reproduce the runtime verification in your own deployment environment by running the commands from the Development Guide (Section 9 of this project guide) — starting Gunicorn, `email_handler.py`, and `job_runner.py` — and confirm you observe the exact log patterns documented.
3. **[Low]** If your deployment includes the out-of-scope subsystems (cron/yacron, event listener, payment providers, OAuth, PGP encryption, Rspamd/SpamAssassin), commission follow-up Q&A investigations as separate tasks since those were explicitly out of scope for this `SWE-AtlasQnA-Repo` rule invocation.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|---|---|---|
| AAP Req 1 — Startup Health Verification (Q1) | 3 | Research and documentation of `/health` endpoint (`server.py:213-215`), WSGI entry (`wsgi.py:1-3`), Dockerfile CMD (`Dockerfile:47`), aiosmtpd Controller startup (`email_handler.py:2381-2403`), and job_runner polling loop (`job_runner.py:329-347`). Captured startup logs for all 3 processes. |
| AAP Req 2 — Log and Dashboard Readiness Signals (Q2) | 4 | Analysis of log format (`app/log.py:12-15`), logger name (`app/log.py:79`), werkzeug suppression (`app/log.py:70-71`), `EmailHandlerFilter` class (`app/log.py:28-37`), `after_request` per-request log (`server.py:272-296`), login flow with 5 branches (`app/auth/views/login.py:21-72`), post-login redirect (`app/auth/views/login_utils.py:35-45`), blueprint wiring, and 4 dashboard stats cards (`app/dashboard/views/index.py:32-52` + `templates/dashboard/index.html:125-173`). |
| AAP Req 3 — User Action Walkthrough (Q3) | 5 | Deep-dive walkthroughs of account creation (`app/auth/views/register.py:31-129`), random alias creation (`app/api/views/new_random_alias.py:21-118` + `app/api/base.py:16-43`), and end-to-end email forwarding with full 19-line lifecycle log capture (`email_handler.py` trace from `set_message_id` through `250 Message accepted for delivery`). Includes DB record verification (`contact`, `email_log`, `alias.nb_forward`). |
| AAP Req 4 — Background Component Auto-Startup (Q4) | 2 | Analysis of process independence: only `gunicorn wsgi:app` auto-starts per `Dockerfile:47`; `email_handler.py`, `job_runner.py`, `cron.py`, and `event_listener.py` each have their own `__main__` block. Includes `create_light_app()` context analysis and the 5-row Process Independence Matrix at the document's end. |
| Environment setup & runtime verification | 3 | PostgreSQL 15 + Redis 7 setup, `alembic upgrade head`, seed data via `fake_data()` + `add_sl_domains()`, starting all 3 core processes, capturing startup logs, SMTP session testing, API calls, and database queries for verification. |
| Code review iterations (9 findings) | 1.5 | Follow-up commits `a6960e05` (addressed 8 code review findings including fabricated snippets, paraphrased comments, and line reference corrections) and `6135abfb` (restored verbatim source comment in `_handle()` excerpt). All changes tightened verbatim source fidelity. |
| Test data cleanup & verification | 0.5 | Deleted runtime-created alias id=22, contact id=9, email_log id=8, job id=4; verified `COUNT(*)=0` for each; confirmed baseline seed data (2 users, 11 aliases, 1 contact, 4 mailboxes, 2 API keys) intact. |
| **Total** | **19** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|---|---|---|
| Human final review of Q&A document for content/tone/completeness vs. user expectations | 0.5 | High |
| Potential minor corrections or clarifications based on human review feedback | 0.5 | Medium |
| **Total** | **1** | |

### 2.3 Verification

- Section 2.1 total (19) + Section 2.2 total (1) = 20 = Section 1.2 Total Hours ✓
- Section 2.2 total (1) = Section 1.2 Remaining Hours = Section 7 pie chart "Remaining Work" ✓
- Completion % = 19 / 20 = 95.0% (consistent across Sections 1.2, 7, 8) ✓

---

## 3. Test Results

Because this task is explicitly documentation-only per the `SWE-AtlasQnA-Repo` rule (no source code modifications permitted), Blitzy's autonomous validation focused on **behavioral verification of the deliverable's accuracy** rather than code unit/integration testing. Every claim in the 1,273-line document was cross-checked against the actual codebase and live runtime behavior.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Source reference accuracy | Systematic file:line verification | 18 file categories, ~80 distinct line ranges | 18/18 file categories; all line ranges exact | 0 | 100% of documented references | Every `file_path:line_number` claim in the doc was checked via `sed -n` against the codebase; all matched verbatim (verified by Final Validator) |
| Web server health check | `curl` + manual inspection | 3 | 3 | 0 | N/A | `GET /health` → HTTP 200 `success` (7 bytes); `GET /auth/login` → HTTP 200 HTML (6918 bytes) with CSRF token; gunicorn startup log lines match doc |
| Email handler startup | Process + `ss` + log inspection | 2 | 2 | 0 | N/A | `Listen for port 20381` (`email_handler.py:2403`) and `Start mail controller 0.0.0.0 20381` (`email_handler.py:2386`) log lines verified; SMTP NOOP → `(250, b'OK')` |
| Job runner startup & dispatcher | Process + log + DB inspection | 2 | 2 | 0 | N/A | Silent poll behavior verified (15+ seconds idle with no jobs); enqueued `send_proton_welcome_email` probe job (id=4) drained within 10s with `Take job <Job 4 ...>` log output; job state=2, attempts=1, taken_at populated |
| Login & dashboard flow | `curl` with session cookie | 3 | 3 | 0 | N/A | `POST /auth/login` → HTTP 302 Location `/dashboard/`; after-login-redirect logs match (`app/auth/views/login_utils.py:35,44`); dashboard (99,576 bytes) shows 4 stats cards (Aliases=10, Forwarded=0, Replies=0, Blocked=0) |
| API alias creation | `curl` + JSON inspection + DB | 2 | 2 | 0 | N/A | `POST /api/alias/random/new` → HTTP 201 created `noodle_gunned315@sl.local` (id=22, mailbox=john@wick.com); `GET /api/v2/aliases?page_id=0` after forward shows `nb_forward=1`, `nb_reply=0`, `nb_block=0`, populated `latest_activity` |
| End-to-end email forwarding lifecycle | `smtplib` + log trace + DB | 19-line log trail | 19/19 matched | 0 | 100% of documented lifecycle | Full log trail captured: `set_message_id` (log.py:24) → `_handle()` banner (2342) → `New message` (2343) → `Handle mail_from` (1980) → `Forward phase` (2202) → `Create or get contact` (580) → `contact_utils.py:110` → `dmarc.py:33` → `Forward` (688) → `Create EmailLog` (740) → `missing date header` (857) → `From header` rewrite (867) → `Replace To/Delete Cc` (313, 316) → `unsubscribe_generator.py:36` → `Forward mail from X to Y` (893) → `mail_sender.py:131` → `Finish mail_from ... 0.18660235404968262 seconds with return code '250 Message accepted for delivery'<<===` (2367) — every file:line reference matched runtime exactly |
| Database record verification | PostgreSQL queries | 4 | 4 | 0 | N/A | Post-test: contact id=9, email_log id=8 (is_reply=f, blocked=f, bounced=f); after cleanup: `COUNT(*)=0` for alias id=22, contact id=9, email_log id=8, job id=4; seed data intact |
| Pre-existing test suite (reference only) | pytest | 639 | 634 | 5 | N/A (not re-run) | Per setup status: 5 failures in `tests/test_mail_sender.py` are IPv6 loopback environment limitations in Kubernetes host-network containers (`OSError: [Errno 99] Cannot assign requested address` on `::1`). Not code defects. This task did not introduce any code and therefore did not alter these results. |

**Integrity statement:** All tests in the first 8 categories above were executed by Blitzy's autonomous validation agents during this session. The 9th category (pre-existing pytest suite) is documented for reference but was not re-executed by this task since no source code was modified.

---

## 4. Runtime Validation & UI Verification

### Core Process Startup — All Verified ✅

- ✅ **Web server (Gunicorn on port 7777)** — `Starting gunicorn 20.0.4`, `Listening at: http://0.0.0.0:7777`, `>>> init logging <<<`. Worker booted. `/health` returns `200 success`.
- ✅ **Email handler (aiosmtpd on port 20381)** — `Listen for port 20381` (`email_handler.py:2403`) → `Start mail controller 0.0.0.0 20381` (`email_handler.py:2386`). TCP socket accepts connections; SMTP NOOP returns `(250, b'OK')`.
- ✅ **Job runner (polling every 10s)** — Loaded `/workspace/local_data/test_words.txt`; silent when `job` table has no ready rows (confirms documented silent-when-empty behavior); dispatcher fires within 10s when a ready job is inserted.

### HTTP Endpoint Verification — All Verified ✅

- ✅ **`GET /health`** → HTTP 200 body `success` (7 bytes); `Server: gunicorn/20.0.4` header.
- ✅ **`GET /auth/login`** → HTTP 200, 6,918 bytes HTML containing CSRF token; `after_request` log line emitted.
- ✅ **`POST /auth/login`** (john@wick.com / password) → HTTP 302 Location: `/dashboard/`; `log user <User 1 John Wick john@wick.com> in` logged.
- ✅ **`GET /dashboard/`** (authenticated) → HTTP 200, 99,576 bytes HTML with all 4 stats cards rendered (Aliases: 10, Forwarded: 0, Replies/Sent: 0, Blocked: 0).
- ✅ **`POST /api/alias/random/new`** (with API key) → HTTP 201 JSON with new alias `noodle_gunned315@sl.local` (id=22).
- ✅ **`GET /api/v2/aliases?page_id=0`** (with API key) → HTTP 200 JSON with alias list; after test email, `nb_forward=1`, `latest_activity` populated.

### SMTP / Email Processing Verification — All Verified ✅

- ✅ **SMTP connect + NOOP** → `(250, b'OK')`.
- ✅ **End-to-end forwarding** — 19-line log lifecycle captured verbatim, every file:line citation in the Q&A document matches runtime output exactly.
- ✅ **Database side-effects** — `contact` and `email_log` rows created with expected values (`is_reply=f`, `blocked=f`, `bounced=f`); `alias.nb_forward` incremented from 0 → 1.

### Job Runner Active Behavior — Verified ✅

- ✅ **Probe job** — Inserted `send_proton_welcome_email` with `user_id: 99999`, state=0 (ready).
- ✅ **Dispatch** — Within 10 s, job runner logged `Take job <Job 4 send_proton_welcome_email {'user_id': 99999}>` (`job_runner.py:334`) and (expected) `ERROR - Unknown job name send_proton_welcome_email` (`job_runner.py:304`) because the probe name is intentionally not registered.
- ✅ **State transition** — `job.state` → 2 (taken), `attempts` → 1, `taken_at` populated.

### Cleanup — Verified ✅

- ✅ All runtime-created data deleted: alias id=22 → `COUNT(*)=0`; contact id=9 → `COUNT(*)=0`; email_log id=8 → `COUNT(*)=0`; job id=4 → `COUNT(*)=0`.
- ✅ Baseline seed data intact: 2 users (`john@wick.com`, `winston@continental.com`), 11 aliases (10 seed + 1 remaining seed alias), 1 seed contact, 1 seed email_log, 2 API keys (Chrome `code`, Firefox `codeFF`), 4 mailboxes, 2 public domains (`sl.local`, `premium.com`).
- ✅ All 3 processes stopped (zero `gunicorn`/`email_handler`/`job_runner` processes running).

### Out-of-Scope Components — Not Exercised (Per AAP) ⚠

- ⚠ **Cron scheduler (yacron + `cron.py`)** — Described from source only (`cron.py:1262` `__main__`, 15 jobs in `crontab.yml`). Not exercised.
- ⚠ **Event listener (`event_listener.py`)** — Described from source only (`event_listener.py:29,94`). Requires Proton webhook configuration; not exercised.
- ⚠ **Payment integrations, OAuth/OIDC, PGP encryption, Rspamd/SpamAssassin** — Explicitly listed as out-of-scope in AAP Section 0.6.2.

---

## 5. Compliance & Quality Review

| Requirement | Benchmark | Status | Progress | Notes |
|---|---|---|---|---|
| `SWE-AtlasQnA-Repo` Rule 1: Create `<source_branch_name>.md` | Exactly one markdown file created in `blitzy/documentation/` | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md`, 1,273 lines |
| `SWE-AtlasQnA-Repo` Rule 2: Build and run source code for analysis | PostgreSQL + Redis + Python deps installed; `alembic upgrade head`; all 3 core processes run | ✅ Pass | 100% | Full runtime captured in log trail |
| `SWE-AtlasQnA-Repo` Rule 3: Base all answers on code, no assumptions | Every claim anchored by `file:line` citation OR verbatim runtime log | ✅ Pass | 100% | 18 file categories + 80+ line ranges verified exact by Final Validator; 9 review findings addressed across 2 follow-up commits to enforce verbatim source fidelity |
| `SWE-AtlasQnA-Repo` Rule 4: Provide thinking/rationale | Each Q has structured subsections with explicit reasoning | ✅ Pass | 100% | Q1 has 4 subsections; Q2 has 7; Q3 has 3; Q4 has 5 + matrix |
| `SWE-AtlasQnA-Repo` Rule 5: Do not modify existing files | Zero source file modifications | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD --name-status` shows only `A blitzy/documentation/app_2cd6ee777f8c.md` |
| `SWE-AtlasQnA-Repo` Rule 6: Do not add any other code | Only the markdown deliverable created | ✅ Pass | 100% | `git diff 2cd6ee77..HEAD --stat` shows 1 file, +1273/-0 |
| `SWE-AtlasQnA-Repo` Rule 7: Place doc in `blitzy/documentation/` | File is at the correct path | ✅ Pass | 100% | Full path: `blitzy/documentation/app_2cd6ee777f8c.md` |
| AAP Requirement — Cleanup of temporary test data | All test users/aliases/contacts/email_logs/jobs deleted | ✅ Pass | 100% | `COUNT(*)=0` on each affected row; baseline seed intact |
| AAP Requirement — Evidence-based answers | Every claim has source citation or runtime log excerpt | ✅ Pass | 100% | Verified by Final Validator |
| AAP Requirement — Environment variable consistency | Runtime testing uses `example.env` values | ✅ Pass | 100% | `example.env` values documented in doc preamble (NOT_SEND_EMAIL=true, EMAIL_DOMAIN=sl.local, DISABLE_ONBOARDING=true, etc.) |
| AAP Requirement — Process isolation awareness | Doc clearly states that email handler, job runner, cron, event listener are independent processes | ✅ Pass | 100% | Q4 dedicates 5 subsections to this + a summary matrix |
| Commit attribution | All commits by `Blitzy Agent <agent@blitzy.com>` | ✅ Pass | 100% | 3 commits on branch, all by Blitzy Agent |
| No forbidden files (progress trackers, status docs) | Only the AAP-mandated deliverable present | ✅ Pass | 100% | No VALIDATION_PROGRESS.md, STATUS.md, TODO.md, or similar files |

**Overall Compliance: 13/13 requirements met (100%).**

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| User may want additional or deeper explanations beyond the 4 questions asked | Operational | Low | Medium | Document is 1,273 lines and covers each question in multiple subsections with concrete evidence; additional Q&A rounds can be commissioned as follow-up tasks | Accepted |
| Out-of-scope subsystems (cron, event listener, payment, OAuth, PGP, spam) not exercised in runtime | Technical | Low | Low | These are explicitly listed in AAP Section 0.6.2 as out-of-scope. The doc covers them with source-only analysis (e.g., Q4.4 explicitly calls out cron and event_listener and gives verification guidance) | Accepted — per AAP |
| Runtime evidence captured on a specific date (2026-04-17); future commits to `app_2cd6ee777f8c` may drift line numbers | Technical | Low | Low | Document explicitly pins to commit hash `2cd6ee777f8c2d3531559588bcfb18627ffb5d2c` in the preamble; any future drift is irrelevant to this pinned analysis | Mitigated |
| 5 pre-existing test failures in `tests/test_mail_sender.py` in the Kubernetes host-network container | Technical | Low | N/A | These are IPv6 loopback environment limitations (`OSError: [Errno 99]` on `::1`), not source defects. This task introduced no code changes and therefore could not affect this pre-existing state | Accepted — environmental, pre-existing |
| Document assumes reader is comfortable with Linux/Docker/Python tooling | Operational | Low | Low | Document targets a self-hoster audience, consistent with the SimpleLogin project's existing README conventions | Accepted |
| No user-facing UI/UX security risk (doc-only delivery) | Security | N/A | N/A | This is a documentation deliverable with zero code execution paths introduced; no security surface is added | N/A |
| Integration risk with upstream SimpleLogin releases | Integration | Low | Low | Document is pinned to a specific commit; future SimpleLogin releases may introduce changes, but that is out of scope for this task | Accepted — version-pinned |
| Information exposure risk in document (e.g., disclosing secrets) | Security | Low | Low | Document contains only public-information source references, example env values from `example.env`, and ephemeral test identifiers (IDs 9, 22, etc.) that are deleted. No API keys, passwords, or secrets are embedded. | Mitigated |
| Operational risk — reader follows doc commands in production | Operational | Medium | Low | Document is clearly framed as a self-hosting verification guide, not production deployment. Section 9 of this project guide reinforces with dev-only commands | Mitigated |

**Risk summary: All identified risks are Low severity. No High or Critical risks remain.**

---

## 7. Visual Project Status

### Project Hours Distribution

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 19
    "Remaining Work" : 1
```

### Completed Work Breakdown by AAP Requirement

```mermaid
pie title Completed Hours by Category
    "Q1: Startup Health (3h)" : 3
    "Q2: Log/Dashboard Signals (4h)" : 4
    "Q3: User Action Walkthrough (5h)" : 5
    "Q4: Background Components (2h)" : 2
    "Env Setup & Runtime Verif (3h)" : 3
    "Code Review Iterations (1.5h)" : 1.5
    "Cleanup & Verification (0.5h)" : 0.5
```

### Remaining Work by Priority

| Priority | Hours | Tasks |
|---|---|---|
| High | 0.5 | Human final review |
| Medium | 0.5 | Minor corrections from review |
| Low | 0 | None |
| **Total** | **1.0** | |

**Color legend:**
- Completed / AI Work: **Dark Blue (#5B39F3)**
- Remaining / Not Completed: **White (#FFFFFF)**

---

## 8. Summary & Recommendations

### Achievements

The project delivered the complete and sole AAP-mandated artifact: a 1,273-line evidence-based Q&A document at `blitzy/documentation/app_2cd6ee777f8c.md` that comprehensively answers all four operational questions posed about SimpleLogin self-hosting. Every claim is anchored either by a specific source-file citation (verified exact by line number against the codebase) or by a verbatim runtime log excerpt captured from live processes on 2026-04-17. The SWE-AtlasQnA-Repo rule was honored without exception — zero source code was modified, zero unauthorized files were created, and all ephemeral test data was cleaned up with COUNT(*)=0 verification.

### Remaining Gaps

The project is **95% complete**. The remaining 5% (1 hour) represents only the final human review cycle and any minor corrections that might result from that review. There are no unresolved technical issues, no failing tests introduced by this task, no broken code paths, no incomplete code sections (there is no code at all — this is documentation-only), and no outstanding AAP requirements.

### Critical Path to Production

For a `SWE-AtlasQnA-Repo` documentation task, "production" simply means the document is accepted by the requestor. The critical path is:

1. **Human reads the document** (15–20 minutes of reading, plus optional live reproduction) — 0.5 hours of effort.
2. **Any feedback-driven refinements** (if requested) — 0.5 hours of effort.
3. **Acceptance** — The PR can be merged or the document can be linked to the requestor's knowledge base.

### Success Metrics

| Metric | Target | Achieved |
|---|---|---|
| Single deliverable file created | 1 | ✅ 1 |
| Zero source code modifications | 0 | ✅ 0 |
| All 4 AAP questions answered | 4 | ✅ 4 |
| Source reference accuracy | 100% | ✅ 100% (Final Validator verified every line reference) |
| Runtime verification | All 3 core processes | ✅ All 3 |
| Test data cleanup | 100% | ✅ 100% (COUNT(*)=0) |
| Commit attribution | Blitzy Agent | ✅ 3/3 commits by `Blitzy Agent <agent@blitzy.com>` |

### Production Readiness Assessment

**95% complete** and ready for human review. The document stands on its own as a comprehensive, self-contained reference for any SimpleLogin self-hoster asking the four questions in the original prompt. No blockers exist. The remaining 1 hour represents only the optional final review-and-refinement cycle.

---

## 9. Development Guide

### 9.1 System Prerequisites

| Component | Version | Source / Notes |
|---|---|---|
| Operating system | Linux (any modern distro) or macOS | Dockerized — host OS is mostly irrelevant |
| Docker | ≥ 20.10 | Only needed if using the provided sandbox image |
| Python | `^3.10` | From `pyproject.toml:60` |
| PostgreSQL | 15+ | From setup status (15 was used in validation) |
| Redis | 7+ | From setup status (7 was used in validation) |
| Node.js | (build-time only) | Per Dockerfile multi-stage build; frontend assets |
| `curl` | any recent | For HTTP verification |
| `python3` CLI with `smtplib` | standard library | For SMTP verification |

### 9.2 Environment Setup

**Option A — Use the provided Docker sandbox image (fastest path to runtime verification):**

```bash
# The provided image has PostgreSQL, Redis, Python deps, venv, and migrations pre-applied
# Image reference: andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c...
# Simply exec into the running container and proceed to Section 9.4
docker exec -it <container_name> bash
```

**Option B — Local setup from scratch:**

```bash
# 1. Clone and checkout the pinned commit
cd /tmp
git clone https://github.com/simple-login/app.git simplelogin
cd simplelogin
git checkout 2cd6ee777f8c2d3531559588bcfb18627ffb5d2c

# 2. Install Python dependencies via Poetry
pip install poetry
poetry install

# 3. Copy example.env to .env (no edits needed for local dev)
cp example.env .env

# 4. Start PostgreSQL and Redis (example for Docker on host)
docker run -d --name sl-pg   -e POSTGRES_PASSWORD=mypassword -e POSTGRES_USER=myuser -e POSTGRES_DB=simplelogin -p 5432:5432 postgres:15
docker run -d --name sl-redis -p 6379:6379 redis:7

# 5. Run database migrations
poetry run alembic upgrade head

# 6. Seed demo data (john@wick.com admin user + 10 aliases + 4 mailboxes)
poetry run python -c "from init_app import add_sl_domains; add_sl_domains()"
poetry run python -c "from app.fake_data import fake_data; fake_data()"
```

### 9.3 Environment Variables (`example.env` reference)

Key variables documented in the Q&A doc's preamble (all from `example.env`):

```bash
URL=http://localhost:7777
NOT_SEND_EMAIL=true                       # example.env:19
EMAIL_DOMAIN=sl.local                     # example.env:22
SUPPORT_EMAIL=support@sl.local
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
FLASK_SECRET=secret
DISABLE_ONBOARDING=true                   # example.env:150
LOCAL_FILE_UPLOAD=true
```

### 9.4 Application Startup (3 core processes)

**Terminal 1 — Web Server (Gunicorn on port 7777):**

```bash
# Production entry point — matches Dockerfile:47
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15

# Expected startup log (captured in validation):
# [timestamp] [pid] [INFO] Starting gunicorn 20.0.4
# [timestamp] [pid] [INFO] Listening at: http://0.0.0.0:7777
# [timestamp] [pid] [INFO] Using worker: sync
# [timestamp] [pid] [INFO] Booting worker with pid: <wpid>
# SL - ... - >>> init logging <<<
```

**Terminal 2 — Email Handler (aiosmtpd on port 20381):**

```bash
# Default port is 20381; override with -p if needed
python3 email_handler.py -p 20381

# Expected startup log (captured in validation):
# SL - INFO - ... "email_handler.py:2403" - <module>() - - Listen for port 20381
# SL - DEBUG - ... "email_handler.py:2386" - main() - - Start mail controller 0.0.0.0 20381
```

**Terminal 3 — Job Runner (10s polling loop):**

```bash
python3 job_runner.py

# Expected startup log (captured in validation):
# SL - ... - load words file: /workspace/local_data/test_words.txt
# [then silent every 10 seconds until a job is enqueued]
```

### 9.5 Verification Steps

**9.5.1 Verify web server:**

```bash
curl -s -v http://localhost:7777/health
# Expected: HTTP 200 body "success" (7 bytes); Server: gunicorn/20.0.4 header

curl -s -o /dev/null -w "%{http_code} %{size_download} bytes\n" http://localhost:7777/auth/login
# Expected: 200 6918 bytes (or similar — HTML with CSRF token)
```

**9.5.2 Verify email handler:**

```bash
# Option A — check that the process is listening
ss -tlnp sport = :20381 || netstat -tlnp | grep 20381

# Option B — SMTP NOOP round-trip
python3 -c "import smtplib; s = smtplib.SMTP('localhost', 20381); print(s.noop()); s.quit()"
# Expected: (250, b'OK')
```

**9.5.3 Verify job runner:**

```bash
# Check process is running
ps -ef | grep 'python3 job_runner.py' | grep -v grep

# Insert a probe job and watch the runner drain it (within 10s)
python3 -c "
from app.models import Job, JobState
from app.db import Session
from init_app import add_sl_domains
from server import create_app
with create_app().app_context():
    j = Job.create(name='probe_test', payload={'user_id': 99999}, state=JobState.ready.value)
    Session.commit()
    print('Inserted job id=', j.id)
"
# Then tail the job_runner log — you should see 'Take job <Job ... probe_test ...>' within 10s.
```

**9.5.4 Verify login flow:**

```bash
# 1. Get CSRF token and session cookie
curl -s -c /tmp/cookies.txt http://localhost:7777/auth/login -o /tmp/login.html
CSRF=$(grep -oP 'name="csrf_token"[^>]*value="\K[^"]+' /tmp/login.html)

# 2. Submit login
curl -s -b /tmp/cookies.txt -c /tmp/cookies.txt   -X POST http://localhost:7777/auth/login   -d "csrf_token=$CSRF&email=john@wick.com&password=password"   -D /tmp/headers.txt -o /dev/null
grep "^HTTP\|^Location" /tmp/headers.txt
# Expected: HTTP/1.1 302 FOUND ... Location: /dashboard/

# 3. Access dashboard
curl -s -b /tmp/cookies.txt -o /tmp/dash.html -w "Size: %{size_download}\n" http://localhost:7777/dashboard/
grep -c 'Aliases\|Forwarded\|Replies\|Blocked' /tmp/dash.html
# Expected: at least 4 (one match per stats card)
```

**9.5.5 Verify API alias creation and activity:**

```bash
# Create an alias via API (API key from seed data: 'code')
curl -s -X POST http://localhost:7777/api/alias/random/new   -H "Authentication: code" -H "Content-Type: application/json"   -d '{}' | python3 -m json.tool
# Expected: HTTP 201, JSON with "email": "<word>_<word><num>@sl.local", "id": <n>

# List aliases with activity
curl -s http://localhost:7777/api/v2/aliases?page_id=0   -H "Authentication: code" | python3 -m json.tool | head -40
# Expected: JSON array with alias objects; each has nb_forward, nb_reply, nb_block, latest_activity
```

### 9.6 Example Usage — End-to-End Email Forwarding Test

```bash
# 1. Ensure email handler is running on 20381

# 2. Send a test email to the alias created above (replace ALIAS_EMAIL)
python3 <<'PY'
import smtplib
from email.mime.text import MIMEText
msg = MIMEText('test body')
msg['Subject'] = 'test forward'
msg['From'] = 'sender@example.com'
msg['To'] = 'ALIAS_EMAIL@sl.local'   # replace with the alias from step 5
with smtplib.SMTP('localhost', 20381) as s:
    s.send_message(msg)
    print('Sent')
PY

# 3. Watch the email_handler log — you should see the full lifecycle:
#    set_message_id → _handle() banner → New message → Handle mail_from →
#    Forward phase → Create or get contact → Forward → Create EmailLog →
#    From header rewrite → Replace To/Delete Cc →
#    Forward mail from X to Y → Finish mail_from ... 250 Message accepted for delivery

# 4. Re-query activity to confirm nb_forward incremented
curl -s http://localhost:7777/api/v2/aliases?page_id=0 -H "Authentication: code" | python3 -m json.tool
```

### 9.7 Cleanup Commands

```bash
# Stop the 3 core processes
pkill -f 'gunicorn wsgi:app'
pkill -f 'python3 email_handler.py'
pkill -f 'python3 job_runner.py'

# Remove any test aliases / contacts / email_logs / jobs you created
python3 <<'PY'
from app.models import Alias, Contact, EmailLog, Job
from app.db import Session
from server import create_app
with create_app().app_context():
    # Example — delete an alias by email
    # Alias.filter_by(email='test_alias@sl.local').delete()
    # Session.commit()
    pass
PY

# Verify cleanup
psql -U myuser -d simplelogin -c   "SELECT (SELECT COUNT(*) FROM alias WHERE email LIKE 'probe_%') AS leaked_aliases,          (SELECT COUNT(*) FROM job WHERE name = 'probe_test') AS leaked_jobs;"
```

### 9.8 Common Issues and Resolutions

| Symptom | Cause | Resolution |
|---|---|---|
| `curl http://localhost:7777/health` connection refused | Gunicorn not running | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` and check `/tmp/gunicorn.log` |
| SMTP connect refused on 20381 | Email handler not running | `python3 email_handler.py -p 20381` — check for `Start mail controller` log line |
| Dashboard shows 0 for all stats after login | Expected for a fresh seed (no forwards/replies/blocks yet) | Run the end-to-end forwarding test in Section 9.6 and refresh |
| Job runner appears frozen (no log output) | Expected silent-when-empty behavior | Insert a probe job (Section 9.5.3) and watch for `Take job` within 10s |
| Login returns `Email or password incorrect` | Seeded admin password | The seed admin is `john@wick.com` / `password` (per `app/fake_data.py`) |
| `POST /api/alias/random/new` returns 401 | Missing or wrong `Authentication` header | Seed provides API key `code` (Chrome) or `codeFF` (Firefox); header name is literal `Authentication` (not `Authorization`) per `app/api/base.py:16-43` |
| `alembic upgrade head` fails on fresh DB | Empty migrations directory or missing `alembic.ini` | Confirm you are at the repository root; `ls alembic.ini migrations/` should both exist |
| Import errors at startup | Missing Python deps or wrong `cryptography`/`PGPy` versions | `poetry install` — these two are pinned together in `pyproject.toml` for binary compatibility |

---

## 10. Appendices

### A. Command Reference

```bash
# Start the 3 core processes
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
python3 email_handler.py -p 20381
python3 job_runner.py

# Database migrations
alembic upgrade head

# Seed demo data (done once on first install)
python3 -c "from init_app import add_sl_domains; add_sl_domains()"
python3 -c "from app.fake_data import fake_data; fake_data()"

# Verify each process
curl -s http://localhost:7777/health                                  # Expected: 200 "success"
python3 -c "import smtplib; print(smtplib.SMTP('localhost',20381).noop())"  # Expected: (250, b'OK')
ps -ef | grep 'python3 job_runner.py' | grep -v grep

# Stop the 3 core processes
pkill -f 'gunicorn wsgi:app'
pkill -f 'python3 email_handler.py'
pkill -f 'python3 job_runner.py'
```

### B. Port Reference

| Port | Service | Protocol | Source |
|---|---|---|---|
| 7777 | Web server (Gunicorn / Flask) | HTTP | `Dockerfile:44` EXPOSE, `Dockerfile:47` CMD |
| 20381 | Email handler (aiosmtpd) | SMTP | `email_handler.py:2400` default for `-p` / `--port` |
| 5432 | PostgreSQL | TCP | `DB_URI` in `example.env` |
| 6379 | Redis (sessions, rate limiting) | TCP | `MEM_STORE_URI` in `example.env` |

### C. Key File Locations

**Deliverable:**
- `blitzy/documentation/app_2cd6ee777f8c.md` (1,273 lines) — the sole output artifact

**Entry points (unchanged, referenced in deliverable):**
- `server.py` — Flask app factory, `/health`, blueprint wiring, `after_request` logging
- `wsgi.py` — WSGI entry (3 lines)
- `email_handler.py` — SMTP handler with `__main__` at 2396
- `job_runner.py` — Polling loop with `__main__` at 329
- `cron.py` — yacron-driven one-shot with `__main__` at 1262
- `event_listener.py` — Postgres NOTIFY consumer with `__main__` at 94

**Supporting source (referenced in deliverable):**
- `app/log.py` — logger format, werkzeug suppression, message-id filter
- `app/auth/views/login.py` — 5-branch login flow
- `app/auth/views/login_utils.py` — post-login redirect
- `app/auth/views/register.py` — registration + activation
- `app/dashboard/views/index.py` — dashboard homepage with stats
- `app/api/views/new_random_alias.py` — random alias API
- `app/api/base.py` — API key authentication (`Authentication` header)
- `app/fake_data.py` — seed data (john@wick.com, aliases, API keys)
- `app/models.py` — ORM models (User, Alias, Contact, EmailLog, Job, Mailbox, ApiKey)
- `app/config.py` — environment variable parsing
- `templates/dashboard/index.html` — dashboard HTML (stats cards)
- `init_app.py` — `add_sl_domains()`, `add_proton_partner()`
- `Dockerfile` — `EXPOSE 7777`, Gunicorn CMD
- `example.env` — all environment variables documented
- `pyproject.toml` — Poetry dependency manifest
- `crontab.yml` — 15 yacron-scheduled jobs

### D. Technology Versions (from `pyproject.toml`)

| Package | Version | Notes |
|---|---|---|
| Python | `^3.10` | `pyproject.toml:60` |
| Flask | `^1.1.2` | Web framework |
| gunicorn | `^20.0.4` | WSGI server (matches captured `Server: gunicorn/20.0.4` header) |
| SQLAlchemy | `1.3.24` | **Pinned** — compatibility constraint |
| psycopg2-binary | `^2.9.3` | PostgreSQL adapter |
| aiosmtpd | `^1.2` | Async SMTP controller |
| flask-login | `^0.5.0` | Session management |
| Flask-Limiter | `^1.4` | Rate limiting (login, alias creation, API) |
| Flask-WTF | `^0.14.3` | CSRF protection |
| Flask-Migrate | `^2.5.3` | Alembic integration |
| Flask-Admin | `^1.5.6` | Admin panel |
| redis | `^4.5.3` | Session store, rate limiting, locks |
| PGPy | `0.5.4` | **Pinned** — compatible with cryptography 37.0.1 |
| cryptography | `37.0.1` | **Pinned** — binary compatibility with PGPy 0.5.4 |
| dkimpy | `^1.0.5` | DKIM signing/verification |
| yacron | `^0.11.1` | YAML cron scheduler |
| pyre2 | `^0.3.6` | Fast regex (container uses 0.3.10 per validator notes) |
| sentry-sdk | `^2.16.0` | Error tracking |
| newrelic | `8.8.0` | **Pinned** — APM agent |

### E. Environment Variable Reference (from `example.env`)

| Variable | Example Value | Notes |
|---|---|---|
| `URL` | `http://localhost:7777` | Base URL for the app |
| `NOT_SEND_EMAIL` | `true` | Skip actual SMTP delivery in local dev (`example.env:19`) |
| `EMAIL_DOMAIN` | `sl.local` | Default alias domain (`example.env:22`) |
| `SUPPORT_EMAIL` | `support@sl.local` | Support address |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | PostgreSQL connection string |
| `FLASK_SECRET` | `secret` | Flask session secret (change for production) |
| `DISABLE_ONBOARDING` | `true` | Skip onboarding emails (`example.env:150`) |
| `LOCAL_FILE_UPLOAD` | `true` | Store uploads locally instead of S3 |
| `MEM_STORE_URI` | `redis://localhost:6379/0` | Redis URL for sessions + rate limiting |
| `EVENT_LISTENER_DB_URI` | (optional) | Set only if running `event_listener.py` |

### F. Developer Tools Guide

| Tool | Purpose |
|---|---|
| `curl` | HTTP endpoint verification (`/health`, `/auth/login`, `/dashboard/`, API) |
| `python3 -c "import smtplib; ..."` | SMTP round-trip verification |
| `ss` / `netstat` | Port listening checks (`ss -tlnp sport = :20381`) |
| `psql` | PostgreSQL queries (seed data, cleanup verification) |
| `ps -ef` | Process enumeration (gunicorn master+worker, email_handler, job_runner) |
| `grep` | Log filtering (`grep ' - SL - ' gunicorn.log`) |
| `alembic` | Database migrations (`alembic upgrade head`) |

### G. Glossary

| Term | Definition |
|---|---|
| **AAP** | Agent Action Plan — the structured task specification this project implements |
| **SWE-AtlasQnA-Repo** | Rule mandating a single Q&A markdown deliverable at `blitzy/documentation/<source_branch_name>.md` with no source code modifications |
| **aiosmtpd** | Async SMTP server library; provides the `Controller` class used by `email_handler.py` |
| **Alias** | A privacy-protecting forwarding address (e.g., `foo@sl.local`) that forwards to a user's real mailbox |
| **Contact** | Auto-created record for each external sender who contacts an alias |
| **EmailLog** | Per-email record (forward/reply/bounce/block) used for statistics and audit |
| **Job** | Row in the `job` table queued for async processing (account deletion, batch imports, etc.) |
| **Job runner** | The `python3 job_runner.py` process that polls `job` every 10 seconds |
| **Email handler** | The `python3 email_handler.py` process that binds SMTP port 20381 and processes inbound mail |
| **Web server** | Gunicorn running `wsgi:app` on port 7777, serving the dashboard/API/auth |
| **Cron (yacron)** | The `yacron -c crontab.yml` process that drives `cron.py -j <name>` invocations |
| **Event listener** | The `python3 event_listener.py <listener\|dead_letter>` process that consumes Postgres NOTIFY events |
| **`create_light_app()`** | Minimal Flask app factory used by background processes (email handler, job runner, cron) for database-only context — skips HTTP blueprint registration |
| **`create_app()`** | Full Flask app factory used by `wsgi.py`; registers all blueprints, extensions, admin, and the `/health` endpoint |
| **`/health`** | HTTP 200 `success` endpoint at `server.py:213-215` used for uptime checks |
| **NOT_SEND_EMAIL** | Environment flag (`example.env:19`) that short-circuits actual SMTP delivery for local dev |
| **Seed data** | Baseline users/aliases/mailboxes created by `app/fake_data.py` for local testing (`john@wick.com`, `winston@continental.com`) |
| **Verbatim source fidelity** | Code review goal: every code fence in the Q&A document must quote source exactly — no paraphrasing, fabrication, or simplification |

---

**End of Blitzy Project Guide**
