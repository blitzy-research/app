# Blitzy Project Guide
### SimpleLogin — Runtime-Grounded Q&A Documentation (`app_2cd6ee777f8c`)

> **Brand color legend** — <span style="color:#5B39F3">**Completed / AI Work = Dark Blue `#5B39F3`**</span> · **Remaining / Not Completed = White `#FFFFFF`** · Headings/Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-grounded **Question-and-Answer documentation file** that confirms a freshly started **local SimpleLogin** instance (a self-hosted email-alias and authentication platform) behaves correctly. Its target user is an operator setting SimpleLogin up locally for the first time, who asks three things: what proves the app is *ready* (Q1), what a *new-user signup* looks like end-to-end (Q2), and what *background services* do behind the scenes (Q3). The technical scope is a **read-only** investigation: the full Python/Flask stack (web app plus standalone SMTP, job, cron, and event processes) was booted and exercised, and the observed runtime behavior was captured verbatim with exact `file:line` grounding. The only artifact produced is the answer document — no source code was changed.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'3px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'3px','pieTitleTextSize':'17px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Completion Status — 91.7% Complete (44h of 48h)
    "Completed Work (AI)" : 44
    "Remaining Work" : 4
```

| Metric | Value |
|--------|-------|
| **Total Hours** | **48.0 h** |
| **Completed Hours (AI + Manual)** | **44.0 h** (AI = 44.0 h, Manual = 0.0 h) |
| **Remaining Hours** | **4.0 h** |
| **Percent Complete** | **91.7 %** — `44 / 48 = 91.7%` |

> The completion percentage is calculated exclusively from AAP-scoped work plus path-to-production, per the AAP-scoped hours methodology. All 10 requirements defined by the Agent Action Plan are complete; the residual 4.0 h is human review, acceptance, and merge.

### 1.3 Key Accomplishments

- [x] **Deliverable authored and committed** — `blitzy/documentation/app_2cd6ee777f8c.md` (503 lines, ~173 `file:line` citations, 42 verbatim code/output blocks), correctly named after the source branch.
- [x] **Full multi-process stack booted for observation** — web app (`server.py`), inbound SMTP (`email_handler.py`), job poller (`job_runner.py`), scheduler (`cron.py`), and event consumer (`event_listener.py`) on Python 3.10.18 + PostgreSQL 15.13 + Redis 7.0.15.
- [x] **Q1 (Readiness) answered with live evidence** — `>>> init logging <<<` marker, the uniform `SL` log format, `/health → 200 success`, the `302 → /auth/login` auth gate, and `sl.local` domain registration.
- [x] **Q2 (New-user walkthrough) exercised end-to-end** — register → verify → login → dashboard with captured HTTP statuses, redirects, flash/page text, and DB side effects (30-char activation code, `activated` flip, default `…@sl.local` alias, welcome email), plus email canonicalization and the negative-login path.
- [x] **Q3 (Behind the scenes) demonstrated** — SMTP listener on `:20381`, the `job_runner` `ready→taken→done` poll cycle, `cron.py` startup, and the Postgres `simplelogin_sync_events` `NOTIFY`/`LISTEN` bus, with the `NOT_SEND_EMAIL` short-circuit that renders emails as log lines.
- [x] **Read-only constraint honored** — baseline-to-HEAD diff contains only the document; `git status` clean; every temporary user/alias/job/script created for observation was removed and the seeded baseline restored.
- [x] **Independently validated** — full pytest suite **637 passed / 0 failed**; 15+ citations re-verified accurate; runtime flows re-exercised — zero discrepancies.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None_ | No code defects and zero deliverable discrepancies were found. The document is complete, accurate, and committed; the repository is byte-for-byte unchanged apart from the deliverable. | — | — |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-------------------|----------------|-------------------|-------------------|-------|
| Source repository | Git read/write | None — branch checked out, deliverable committed at HEAD `c5eab6ac`. | ✅ Resolved | Blitzy Agent |
| PostgreSQL / Redis | Local service | None — both online during observation (PostgreSQL 15.13, Redis 7.0.15). | ✅ Resolved | Blitzy Agent |
| `tests/api/test_apple.py` | External internet (Apple servers) | The network-isolated sandbox cannot reach Apple's Sign-in-with-Apple endpoints, so this one test cannot run. It is **out of scope**, unrelated to the deliverable, and not a code defect. | ⚠ Environmental (accepted) | Human (CI/infra) |

> **No access issues affect the deliverable.** The single environmental limitation above concerns an unrelated external-network test only.

### 1.6 Recommended Next Steps

1. **[High]** Have a subject-matter expert review the Q&A document for technical accuracy and completeness, spot-checking a sample of the ~173 `file:line` citations against the source at branch `app_2cd6ee777f8c`.
2. **[High]** Confirm with the original requester that the document answers their three questions (readiness, walkthrough, behind-the-scenes) to their satisfaction (acceptance sign-off).
3. **[Medium]** Review and merge the pull request — a single additive documentation file with no source-code impact.
4. **[Low, optional / out of scope]** If desired later, extend observation to a full inbound alias-forward (delivering a message to the SMTP listener) and to an actual `NOTIFY` payload (by configuring `EVENT_WEBHOOK`) — both are beyond the current AAP scope.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|------:|-------------|
| Runtime environment provisioning & full-stack bring-up | 6.0 | Python 3.10 virtualenv, locked dependency set (incl. `pyre2 0.3.6`, `cbor2 5.2.0`), PostgreSQL 15 + Redis 7 online, out-of-repo `CONFIG` file, `alembic upgrade head`, `flask dummy-data` seeding. |
| Q1 — Readiness investigation & evidence capture | 4.0 | Booted `server.py`; captured startup banner, `>>> init logging <<<` marker, `SL` log format, `/health` (with headers), unauthenticated route gating, `sl.local` registration (via a throwaway DB to catch the first-run line), and the `after_request` access log. |
| Q2 — New-user walkthrough investigation & evidence capture | 7.0 | Wrote/ran an observation script driving register→activate→login→dashboard; captured HTTP statuses, redirect targets, flash/page text, DB side effects (30-char activation code, `activated` flip, default alias, welcome email), email canonicalization, and the negative-login path. |
| Q3 — Background-services investigation & evidence capture | 7.0 | Booted `email_handler.py` (`:20381`), enqueued a sentinel job and observed the `job_runner` `ready→taken→done` cycle, ran `cron.py` and `event_listener.py`, and captured the `NOTIFY` bus, the webhook-not-configured branch, the NewRelic-vs-Postgres event distinction, and the `NOT_SEND_EMAIL` short-circuit. |
| Deliverable authoring | 8.0 | Wrote the 503-line Q&A document: 3 question groups, ~173 `file:line` citations, 42 verbatim code/output blocks, an explicit coverage pass, and honesty notes for behaviors not runtime-exercised. |
| Review-finding remediation | 4.0 | Remediated 12 review findings (commit `2a3a6acc`) and corrected the dependency-transparency disclosure to match the final runtime (commit `c5eab6ac`). |
| Final autonomous validation | 6.0 | Ran full pytest suite (637 passed on fresh DB), re-exercised runtime flows, re-verified citations, and resolved environmental issues (container-only key/format drift, test-DB reset). |
| Cleanup & read-only verification | 2.0 | Deleted the temporary user/alias/sentinel job, dropped the throwaway probe DB, removed all scripts and captured-output files, restored the seeded baseline, and verified `git status`/diff clean. |
| **Total Completed** | **44.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|------:|----------|
| Human SME review of the Q&A document (technical accuracy & completeness; citation spot-checks) | 2.5 | High |
| User acceptance / sign-off that the document answers the three original questions | 1.0 | High |
| Pull request review & merge to target branch | 0.5 | Medium |
| **Total Remaining** | **4.0** | |

### 2.3 Hours Reconciliation

| Line | Hours |
|------|------:|
| Section 2.1 — Completed | 44.0 |
| Section 2.2 — Remaining | 4.0 |
| **Total Project Hours (matches Section 1.2)** | **48.0** |
| **Completion** | **44 / 48 = 91.7 %** |

> **Integrity note:** all remaining work is human path-to-production (review, acceptance, merge). No engineering, coding, or bug-fix work remains for the AAP-scoped deliverable.

---

## 3. Test Results

All results below originate from Blitzy's autonomous validation logs for this project. Because this is a **read-only documentation** task, no tests were added; the project's existing suite was executed as a regression/health signal to confirm the application described by the document is sound.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Full project suite (unit + integration + API under `tests/`) | pytest | 637 | 637 | 0 | Not measured¹ | Run on a freshly migrated test DB (`CONFIG=tests/test.env`); exit code 0 in ~166 s. |
| Excluded — external-network test | pytest | 1 | 0 | 0 | — | `tests/api/test_apple.py` requires live internet to Apple (Sign-in-with-Apple); unavailable in the isolated sandbox. Environmental, not a code defect. |
| Document citation verification | Manual + agent audit | 15+ (spot-check) | 15+ | 0 | — | Independent spot-check of `file:line` citations (see Section 5) — all resolved accurately. |

> ¹ Coverage was not the objective of a read-only documentation task; no code was changed, so the suite served purely as a regression signal. **100 % of runnable tests pass (637/637).**

---

## 4. Runtime Validation & UI Verification

**Legend:** ✅ Operational · ⚠ Partial (disclosed) · ❌ Failing

**Web application & authentication gate**
- ✅ Web app boots via `create_app()` (`server.py:139`).
- ✅ `GET /health → 200 "success"` — `Content-Length: 7`, `Server: Werkzeug/1.0.1 Python/3.10.18` (`server.py:213-215`).
- ✅ Unauthenticated `GET /` and `GET /dashboard/ → 302 → /auth/login` (auth gate live; `server.py:250-255`, `@login_required`).
- ✅ Authenticated `GET /dashboard/ → 200` (`app/dashboard/views/index.py`).

**Q2 new-user flow (exercised end-to-end)**
- ✅ `POST /auth/register → 200` "waiting" page; `create user` logged (`app/auth/views/register.py:85`); 30-char `ActivationCode` stored; `activated=False`.
- ✅ `GET /auth/activate → 302 → /dashboard/`; `User.activated` flips `False→True` (`app/auth/views/activate.py:49`); welcome email composed; success flash `Your account has been activated` (`app/auth/views/activate.py:56`).
- ✅ `POST /auth/login` — correct password `→ 302 → /dashboard/` with `log user … in` (`app/auth/views/login_utils.py:35`); wrong password stays `200` with `Email or password incorrect` (`app/auth/views/login.py:49`).
- ✅ Dashboard landing `→ 200` with `Show intro to <User …>` (`app/dashboard/views/index.py:172`).

**Q3 background services & event bus**
- ✅ `email_handler.py` — `Start mail controller 0.0.0.0 20381` (`email_handler.py:2386`) and `Listen for port 20381` (`email_handler.py:2403`).
- ✅ `job_runner.py` — `Take job …` and `ready→taken→done` state transitions on the Postgres `job` table (`job_runner.py:333-345`).
- ✅ `cron.py` — `Start running cronjob` (`cron.py:1262-1263`).
- ✅ `event_listener.py` — Postgres event consumer starts.
- ✅ Alias-email readiness — `Add sl.local to SL domain` (`init_app.py:44`).
- ⚠ Full inbound alias-forward (`Forward … -> … -> …`, `email_handler.py:688`) — **not** runtime-triggered (requires delivering an inbound message to the listener); the document discloses this and confirms the listener is up.
- ⚠ Postgres `NOTIFY` payload — **not** emitted locally because no `EVENT_WEBHOOK` is configured; the app logs `Not sending events because webhook is not configured…` (`app/events/event_dispatcher.py:62-64`). Disclosed in the document.

**UI verification (server-rendered)**
- ✅ `register_waiting_activation.html` — title `Activation Email Sent`, body `An email to validate your email is on its way.`
- ✅ Login page rendered for anonymous visitors (`/ → 302 → /auth/login`).
- ✅ Dashboard `index.html` shown on first login (intro walkthrough).
- ✅ Bad-login error flash `Email or password incorrect` rendered.

---

## 5. Compliance & Quality Review

Cross-mapping the AAP deliverables and the governing **SWE-AtlasQnA-Repo** rule to quality/compliance benchmarks:

| Benchmark / AAP Requirement | Status | Progress | Evidence |
|-----------------------------|--------|----------|----------|
| Deliverable created at correct path & branch-derived name | ✅ Pass | 100% | `blitzy/documentation/app_2cd6ee777f8c.md` committed (`c72907a5`). |
| Run-first methodology (observe, then write) | ✅ Pass | 100% | Verbatim runtime output captured; stack + services booted. |
| Verbatim quoting + exact `file:line` grounding | ✅ Pass | 100% | ~173 citations, 42 code/output blocks; 15+ independently re-verified. |
| Every question sub-part answered (coverage pass) | ✅ Pass | 100% | Explicit checkbox coverage for all Q1/Q2/Q3 sub-parts. |
| Honesty — state what cannot be verified | ✅ Pass | 100% | Inbound alias-forward & `NOTIFY` payload flagged as not exercised. |
| Read-only scope (no source file changed) | ✅ Pass | 100% | Baseline-to-HEAD diff = 1 file; `git status` clean. |
| Temporary artifacts created then removed | ✅ Pass | 100% | Temp user/alias/sentinel job deleted; throwaway DB dropped; scripts removed; baseline restored. |
| Secret hygiene (no credentials leaked) | ✅ Pass | 100% | DB password, `FLASK_SECRET`, session id, `PGPASSWORD`, test password all redacted. |
| Correct runtime version (Python 3.10) | ✅ Pass | 100% | Python 3.10.18 used; 3.12/3.13 deliberately avoided (per `pyproject.toml:61`, `Dockerfile`). |
| SME accuracy review | ⬜ Pending | 0% | Human review outstanding (Section 1.6 / 2.2). |
| User acceptance sign-off | ⬜ Pending | 0% | Requester acceptance outstanding. |

**Fixes applied during autonomous validation:** 12 review findings remediated (`2a3a6acc`); dependency-transparency disclosure corrected to match the final runtime (`c5eab6ac`); environmental **container-only** restorations (DKIM key format drift, PGP key drift, test-DB pollution reset) — the host repository was never touched.

**Outstanding items:** none for the deliverable itself; the two pending rows above are human review/acceptance.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Cited `file:line` references drift as source evolves | Technical | Low | Medium | Citations pinned to branch `app_2cd6ee777f8c` (HEAD `2cd6ee77`); document distinguishes run-specific values from invariants. | Mitigated |
| Two behaviors stated but not runtime-exercised (inbound alias-forward; `NOTIFY` payload) | Technical | Low | Low | Transparently disclosed in the coverage pass; full exercise needs inbound SMTP delivery + configured `EVENT_WEBHOOK`. | Disclosed / Accepted |
| Secret leakage in documentation | Security | Low | Low | All secrets redacted; out-of-repo `CONFIG` never committed. Verified. | Mitigated |
| New attack surface | Security | None | None | Read-only task — no code, dependency, or config change to the repo. | N/A |
| Environment reproducibility (needs Python 3.10, PostgreSQL, Redis, specific deps) | Operational | Low-Med | Medium | Exact versions and copy-pasteable commands documented (Section 9 + document setup summary). | Mitigated |
| `tests/api/test_apple.py` external-network dependency | Operational | Low | High (isolated CI) | Exclude the test or provide internet; environmental and out of scope. | Documented / Accepted |
| Production email path (Postfix relay, DNS/MX, S3) not covered | Integration | Low | Low | Document explicitly scoped to local `NOT_SEND_EMAIL` observation; production integration is out of scope per AAP. | Out of scope |

**Overall risk posture: LOW.** No critical or high-severity risks; no code defects; zero deliverable discrepancies.

---

## 7. Visual Project Status

**Project hours — completed vs. remaining** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'3px','pieOuterStrokeColor':'#B23AF2','pieOuterStrokeWidth':'3px','pieTitleTextSize':'17px','pieSectionTextColor':'#B23AF2','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Project Hours Breakdown (Total 48h)
    "Completed Work" : 44
    "Remaining Work" : 4
```

**Remaining hours by category (Section 2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#B23AF2','pie2':'#A8FDD9','pie3':'#FFFFFF','pieStrokeColor':'#5B39F3','pieStrokeWidth':'2px','pieOuterStrokeColor':'#5B39F3','pieLegendTextColor':'#111111'}}}%%
pie showData
    title Remaining Work — 4h
    "SME doc review (High)" : 2.5
    "User acceptance (High)" : 1
    "PR review & merge (Medium)" : 0.5
```

> **Integrity check:** "Remaining Work" = **4 h** here matches Section 1.2 (Remaining Hours = 4 h) and the Section 2.2 total (2.5 + 1 + 0.5 = 4 h). "Completed Work" = **44 h** matches Section 2.1.

---

## 8. Summary & Recommendations

**Achievements.** The project is **91.7 % complete** (`44 / 48 h`). All ten AAP-scoped requirements are delivered: the runtime was provisioned; the web app and all four standalone services were booted; the three question groups (readiness, new-user walkthrough, behind-the-scenes services) were exercised and answered with verbatim output and ~173 exact `file:line` citations; and the single deliverable — `blitzy/documentation/app_2cd6ee777f8c.md` — was authored, remediated against review findings, validated (637 tests passing, citations re-verified), and committed while leaving the repository byte-for-byte unchanged.

**Remaining gaps.** The residual **4.0 h** is entirely human path-to-production: an SME accuracy review, requester acceptance, and PR merge. No engineering or bug-fix work remains.

**Critical path to production.** SME review → user acceptance → merge. Each step is low-risk given the deliverable is a single additive file with no source-code impact and zero validated discrepancies.

**Success metrics.**

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Runnable tests passing | 100% | 637/637 (100%) | ✅ |
| Source files modified | 0 | 0 | ✅ |
| Question sub-parts answered | All | All (coverage pass) | ✅ |
| Citations verified accurate (spot-check) | 100% | 100% | ✅ |
| Working tree clean | Yes | Yes | ✅ |

**Production readiness assessment.** For a documentation deliverable, "production" means the reviewed document is accepted and merged. The artifact is **ready for human review**: complete, accurate, well-grounded, and safe (secrets redacted, read-only). Recommendation: proceed to SME review and merge.

---

## 9. Development Guide

This guide covers (A) reproducing the runtime observations behind the document and (B) viewing and verifying the deliverable itself. Commands mirror `CONTRIBUTING.md` and the document's own verified setup summary.

### 9.1 System Prerequisites

- **Python 3.10** (pinned; `python = "^3.10"` at `pyproject.toml:61`, `FROM python:3.10` in the `Dockerfile`). Python 3.12/3.13 are **not** supported (`CONTRIBUTING.md` notes 3.12 does not work).
- **PostgreSQL 15** and **Redis 7** running locally.
- **Poetry** (dependency management) and **git**.

### 9.2 Environment Setup (non-mutating)

Keep configuration **outside** the repository so the source tree stays unchanged. The app reads the file named by the `CONFIG` environment variable (`app/config.py:65,69`).

```bash
# Create an out-of-repo config from the template (do NOT commit this file)
cp example.env /root/sl.env

# Ensure these values are set in /root/sl.env:
#   URL=http://localhost:7777          # example.env:6
#   NOT_SEND_EMAIL=true                # example.env:19  (emails become log lines)
#   EMAIL_DOMAIN=sl.local              # example.env:22
#   DISABLE_ONBOARDING=true            # example.env:150
#   DB_URI=postgresql://USER:PASSWORD@localhost:5432/simplelogin
#   MEM_STORE_URI=redis://localhost
#   FLASK_SECRET=<a-local-dev-secret>
```

### 9.3 Dependency Installation

```bash
# From the repository root, into a Python 3.10 environment:
poetry env use 3.10
poetry install         # installs the locked dependency set from poetry.lock
```

### 9.4 Schema, Seed & Application Startup

```bash
# 1) Apply migrations (schema head: 32f25cbf12f6)
CONFIG=/root/sl.env poetry run alembic upgrade head

# 2) Seed baseline data (creates john@wick.com / password and winston@continental.com)
CONFIG=/root/sl.env FLASK_APP=server.py poetry run flask dummy-data

# 3) Start the web app (dev server on http://localhost:7777)
CONFIG=/root/sl.env poetry run python server.py

# 4) (Q3) Start the standalone services, each in its own shell:
CONFIG=/root/sl.env poetry run python email_handler.py   # inbound SMTP on :20381
CONFIG=/root/sl.env poetry run python job_runner.py       # 10-second poll loop
CONFIG=/root/sl.env poetry run python cron.py -j <job>    # scheduled maintenance job
CONFIG=/root/sl.env poetry run python event_listener.py   # Postgres LISTEN consumer
```

### 9.5 Verification Steps

```bash
# Health endpoint — expect: HTTP/1.0 200 OK, body "success", Content-Length: 7
curl -sS -i http://127.0.0.1:7777/health

# Auth gate — anonymous root redirects to the login page (302 -> /auth/login)
curl -sS -i http://127.0.0.1:7777/

# Browser: open http://localhost:7777 and log in with john@wick.com / password
```

Expected startup banner (verbatim, from the document):

```text
load config file /root/sl.env
>>> URL: http://localhost:7777
...
>>> init logging <<<
2026-... - SL - DEBUG - <pid> - "/app/app/utils.py:17" - <module>() -  - load words file: /app/local_data/test_words.txt
```

### 9.6 Viewing & Verifying the Deliverable (tested in this session)

```bash
# Confirm the deliverable exists
test -f blitzy/documentation/app_2cd6ee777f8c.md && echo "present"

# Read it / list its headings
grep -n "^#" blitzy/documentation/app_2cd6ee777f8c.md
sed -n '1,60p'  blitzy/documentation/app_2cd6ee777f8c.md

# Read-only verification — working tree must be clean, diff must show ONLY the doc
git status --porcelain                                        # -> empty
git diff --name-status origin/app_2cd6ee777f8c...HEAD          # -> A blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.7 Troubleshooting

- **`error: externally-managed-environment` / wrong Python** → use Python **3.10** (not 3.12/3.13); create a dedicated venv/poetry env.
- **`tests/api/test_apple.py` hangs or fails** → it needs live internet to Apple; exclude it: `pytest tests/ --ignore=tests/api/test_apple.py`.
- **App seems to edit tracked files** → ensure config is referenced via `CONFIG=/root/sl.env` (out-of-repo), never by editing `.env` inside the tree, to preserve the read-only guarantee.
- **Activation/welcome emails not delivered** → expected: `NOT_SEND_EMAIL=true` short-circuits sending (`app/mail_sender.py:130-137`); the emails appear as `app/mail_sender.py:131` log lines instead.
- **Migrations fail on a dirty DB** → drop/recreate the database and re-run `alembic upgrade head`.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Apply migrations | `CONFIG=/root/sl.env alembic upgrade head` |
| Seed baseline data | `CONFIG=/root/sl.env FLASK_APP=server.py flask dummy-data` |
| Run web app | `CONFIG=/root/sl.env python server.py` |
| Run SMTP handler | `CONFIG=/root/sl.env python email_handler.py` |
| Run job poller | `CONFIG=/root/sl.env python job_runner.py` |
| Run cron job | `CONFIG=/root/sl.env python cron.py -j <job>` |
| Run event listener | `CONFIG=/root/sl.env python event_listener.py` |
| Health check | `curl -sS -i http://127.0.0.1:7777/health` |
| Run test suite (health signal) | `CONFIG=tests/test.env pytest tests/ --ignore=tests/api/test_apple.py` |
| Verify read-only | `git status --porcelain` · `git diff --name-status origin/app_2cd6ee777f8c...HEAD` |

### B. Port Reference

| Port | Service |
|------|---------|
| 7777 | Web application (dev server; `URL=http://localhost:7777`) |
| 20381 | Inbound SMTP (`email_handler.py` aiosmtpd controller) |
| 5432 | PostgreSQL |
| 6379 | Redis (default) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (only file added). |
| `server.py` | Web app factory `create_app()`, `/health`, access log. |
| `app/log.py` | The single `SL` logger (format + `>>> init logging <<<`). |
| `init_app.py` | `add_sl_domains()` registers `sl.local`. |
| `app/auth/views/{register,activate,login,login_utils}.py` | Auth flow (Q2). |
| `app/dashboard/views/index.py` | Dashboard landing (`@login_required`). |
| `app/mail_sender.py` | `NOT_SEND_EMAIL` short-circuit (email observability). |
| `email_handler.py`, `job_runner.py`, `cron.py`, `event_listener.py` | Standalone services (Q3). |
| `app/events/event_dispatcher.py` | Postgres `NOTIFY` `simplelogin_sync_events` bus. |
| `example.env`, `CONTRIBUTING.md`, `pyproject.toml`, `Dockerfile` | Setup references. |

### D. Technology Versions (observed / locked)

| Component | Version |
|-----------|---------|
| Python | 3.10.18 (pinned `^3.10`) |
| PostgreSQL | 15.13 |
| Redis | 7.0.15 |
| Alembic schema head | `32f25cbf12f6` |
| Flask | 1.1.2 |
| SQLAlchemy | 1.3.24 |
| Flask-Login | 0.5.0 |
| aiosmtpd | 1.4.2 |
| redis (client) | 4.6.0 |
| bcrypt | 3.2.0 |
| pyre2 / cbor2 | 0.3.6 / 5.2.0 |

### E. Environment Variable Reference

| Variable | Example | Purpose |
|----------|---------|---------|
| `CONFIG` | `/root/sl.env` | Path to the out-of-repo config file (`app/config.py:65,69`). |
| `URL` | `http://localhost:7777` | Base URL / web port. |
| `NOT_SEND_EMAIL` | `true` | Short-circuits email sending → emails become log lines. |
| `EMAIL_DOMAIN` | `sl.local` | Local alias domain. |
| `DISABLE_ONBOARDING` | `true` | Disables onboarding emails locally. |
| `DB_URI` | `postgresql://USER:PASS@localhost:5432/simplelogin` | PostgreSQL connection. |
| `MEM_STORE_URI` | `redis://localhost` | Redis (sessions / rate limiting). |
| `FLASK_SECRET` | `<local-dev-secret>` | Flask session secret. |
| `FLASK_APP` | `server.py` | Entry point for `flask` CLI commands. |

### F. Developer Tools Guide

- **Git verification:** `git status --porcelain` (expect empty) and `git diff --name-status origin/app_2cd6ee777f8c...HEAD` (expect only the deliverable).
- **Markdown sanity:** check code-fence balance — `python3 -c "print(sum(l.startswith(chr(96)*3) for l in open('blitzy/documentation/app_2cd6ee777f8c.md')))"` (expect an even number; observed 84).
- **Citation audit:** for any `path:line` citation, run `sed -n '<line>p' <path>` to confirm it resolves to the quoted symbol/string.

### G. Glossary

| Term | Meaning |
|------|---------|
| **`SL` logger** | SimpleLogin's single structured logger (`app/log.py`); every runtime line uses its uniform format. |
| **Alias** | A generated email address (e.g. `…@sl.local`) that forwards to a real inbox. |
| **Email canonicalization** | Normalizing an address (stripping dots/`+`-suffixes for gmail/proton) before storage (`app/utils.py:78-94`). |
| **`NOTIFY`/`LISTEN`** | PostgreSQL pub/sub used by the `simplelogin_sync_events` channel to fan events out to `event_listener.py`. |
| **`dummy-data`** | A Flask CLI command that seeds baseline users (`john@wick.com`, `winston@continental.com`). |
| **`NOT_SEND_EMAIL`** | Config flag that makes the app log emails instead of sending them — the key enabler for local email observability. |
| **AAP** | Agent Action Plan — the governing specification for this project. |

---

*Cross-section integrity verified: Remaining hours = 4.0 in Sections 1.2, 2.2, and 7; Section 2.1 (44) + Section 2.2 (4) = 48 = Total in Section 1.2; all test figures originate from Blitzy's autonomous validation logs (637 passed); brand colors applied (Completed = `#5B39F3`, Remaining = `#FFFFFF`).*