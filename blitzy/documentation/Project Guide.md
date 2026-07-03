# Blitzy Project Guide — SimpleLogin Runtime-Behavior Q&A

> **Governing rule:** `SWE-AtlasQnA-Repo` (read-first, run-first documentation Q&A).
> **Branch:** `blitzy-33f70bcd-a718-4307-83f4-2e539e81e639` · **Baseline:** `origin/app_2cd6ee777f8c` (commit `2cd6ee77`) · **HEAD:** `1f959d09`
> **Brand color legend:** <span style="color:#5B39F3">■</span> Completed / AI Work = Dark Blue `#5B39F3` · <span style="color:#FFFFFF; background:#333">■</span> Remaining = White `#FFFFFF`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only documentation investigation** of the SimpleLogin open-source email-alias application (Python 3.10 / Flask 1.1.2 / SQLAlchemy 1.3.24 monolith backed by PostgreSQL 13, Redis 6, and an `aiosmtpd` SMTP handler). The objective was to determine — by *actually running* the stack — three developer-environment bring-up behaviors: the exact error hit against an empty (unmigrated) database, which Python services must run for the system to work, and the SMTP rejection returned when the initialization step is skipped. The target audience is SimpleLogin developers who found the startup order and failure modes under-documented. The sole deliverable is one Markdown answer document grounded in verbatim runtime evidence; no source code was changed.

### 1.2 Completion Status

The project is **91.7% complete** (AAP-scoped, hours-based per PA1). All autonomous investigation, reproduction, authoring, and validation work is finished and independently verified; the only remaining work is human governance (SME accuracy review + PR merge).

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 91.7% Complete
    "Completed (AI) : 33h" : 33
    "Remaining : 3h" : 3
```

| Metric | Value |
|---|---|
| **Total Hours** | **36 h** |
| **Completed Hours (AI + Manual)** | **33 h** (33 h AI autonomous + 0 h manual) |
| **Remaining Hours** | **3 h** |
| **Percent Complete** | **91.7 %** (33 / 36) |

### 1.3 Key Accomplishments

- ✅ **Q1 answered with the full verbatim Python traceback** — `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable: relation "users" does not exist`, surfaced on the first DB-touching request (login POST) and rendered as HTTP 500.
- ✅ **Q2 answered with a required-vs-auxiliary service inventory** — two required services (webapp `:7777`, email handler `:20381`), three auxiliary (`job_runner.py`, `event_listener.py`, `cron.py`), each started through its real entry point with verbatim startup output and a confirmed port bind.
- ✅ **Q3 answered with the exact SMTP status code and rejection logs** — `550 SL E515 Email not exist` plus `alias xyz@sl.local cannot be created on-the-fly, return 550`, with the true causal relationship to the empty `SLDomain`/`public_domain` table empirically proven.
- ✅ **Read-only scope preserved** — exactly one file added (`blitzy/documentation/app_2cd6ee777f8c.md`); zero source files modified or deleted; clean working tree.
- ✅ **Full regression suite green** — 639 pytest tests passed (exit 0) confirming the investigation left the codebase healthy.
- ✅ **Reproducibility & citation integrity** — every scenario reproduced ≥2× (Q1 ×2, Q2 ×2, Q3 ×3); ~40+ `file:line` citations verified accurate; coverage checklist 10/10; all four user verbatim phrasings preserved.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None_ | The Final Validator found zero code defects; the deliverable is production-ready as committed at `1f959d09`. No unresolved item blocks release or validation. | — | — |

### 1.5 Access Issues

| System / Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _None_ | — | No access issues identified. The autonomous run had full access to PostgreSQL 13, Redis 6, the in-project virtualenv, all service entry points, and the full test suite; no repository-permission, credential, or third-party-API blockers were encountered. | N/A | — |

### 1.6 Recommended Next Steps

1. **[Medium]** Have a SimpleLogin domain SME read `blitzy/documentation/app_2cd6ee777f8c.md` and spot-check the Q1 traceback, Q2 service classification, and Q3 `550 SL E515` + causal-nuance conclusion against the live codebase.
2. **[Medium]** Review the single-file PR, confirm the read-only scope (`git diff --name-status` shows only one `A` entry), then approve and merge.
3. **[Low]** (Optional, out of AAP scope) Note the observed backend behaviors surfaced by the document — verbose `debug=True` traceback to stdout and the absence of a startup schema/health guard — as potential future hardening backlog items; the current task was to *observe*, not remediate.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed work is autonomous (AI) and traces to AAP-specified deliverables, methodology requirements, or constraints.

| Component | Hours | Description |
|---|---:|---|
| Environment provisioning & canonical-config comprehension | 3 | Stand up PostgreSQL 13 & Redis 6, install deps into in-project `./.venv` (Poetry), create canonical `.env` from `example.env` (`EMAIL_DOMAIN=sl.local`), verify versions. *(AAP R11, R14)* |
| Source-code investigation & citation gathering | 7 | Cross-cutting trace of Q1/Q2/Q3 paths across web tier, SMTP handler, ORM/session, config loader, and migration/seed tooling; ~40+ verified `file:line` citations. *(AAP R9)* |
| Q1 reproduction & capture | 3 | Provision empty DB, run `python server.py`, drive `GET`/`POST` login, capture full verbatim traceback (first failing relation `users`), reproduce ×2. *(AAP R2)* |
| Q2 reproduction & capture (5 entry points) | 4 | Migrate + `init_app.py`, start all five entry points through real processes, capture verbatim startup stdout, confirm actual port binds, classify required vs auxiliary, reproduce ×2. *(AAP R3)* |
| Q3 reproduction & capture | 4 | Migrate-but-not-init, verify empty `public_domain`, inject a real message to `xyz@sl.local` via live `aiosmtpd`, capture `550 SL E515` + rejection logs, empirically prove causal nuance, enumerate E515/E207, reproduce ×3. *(AAP R4, R5)* |
| Answer-document authoring | 6 | Author the 1,018-line / 9,141-word document with ~50 verbatim evidence blocks, cause→effect explanations, read-only scope proof, and coverage-pass checklist. *(AAP R1, R6–R8, R10, R15)* |
| Autonomous validation & review cycles | 6 | Independent re-reproduction of all three scenarios, full citation audit, 639-test regression run, repo-pristine confirmation, and two code-review-fix commits. *(AAP R12, R13, R16, R17)* |
| **Total Completed** | **33** | **Matches Completed Hours in Section 1.2** |

### 2.2 Remaining Work Detail

All remaining work is human governance (path-to-production for a documentation deliverable). There are no blocking or code-defect items.

| Category | Hours | Priority |
|---|---:|---|
| SME technical-accuracy review of the answer document | 2 | Medium |
| PR review, read-only-scope confirmation, approval & merge | 1 | Medium |
| **Total Remaining** | **3** | **Matches Remaining Hours in Section 1.2 and Section 7 pie chart** |

### 2.3 Hours Reconciliation

| Check | Result |
|---|---|
| Section 2.1 total (Completed) | 33 h |
| Section 2.2 total (Remaining) | 3 h |
| Section 2.1 + Section 2.2 | 36 h = Total Hours (Section 1.2) ✓ |
| Completion % | 33 / 36 = **91.7 %** ✓ |
| Cross-section Rule 1 (1.2 ↔ 2.2 ↔ 7 remaining) | 3 h = 3 h = 3 h ✓ |

---

## 3. Test Results

All results below originate from **Blitzy's autonomous validation logs** for this branch. Because this is a read-only documentation task, no new application code or tests were authored; the regression suite was executed to confirm the investigation left the codebase healthy.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---:|---:|---:|---|---|
| Unit + Integration (full regression suite) | pytest 7.3.1 | 639 | 639 | 0 | N/A (documentation task — no new source code to cover) | Executed via `CONFIG=tests/test.env ./.venv/bin/pytest tests -q`; exit 0; 51 warnings. Confirms the read-only investigation left the codebase healthy. |
| **Total** | — | **639** | **639** | **0** | — | **100% pass rate** |

**Runtime behavioral reproductions** (functional evidence for the three questions) are summarized here and detailed in Section 4:

| Scenario | Runs | Result | Consistency |
|---|---:|---|---|
| Q1 — empty-DB login POST | 2 | HTTP 500 · `UndefinedTable: relation "users" does not exist` | Identical across runs |
| Q2 — required-service readiness (webapp + email handler) | 2 each | `/health` → 200 `success`; SMTP banner `220 … Python SMTP 1.4.2` on :20381 | Identical across runs |
| Q3 — skipped-init SMTP rejection | 3 | `550 SL E515 Email not exist` + rejection log lines | Identical across runs |

> **Integrity note:** The 639 tests are the repository's existing suite run by autonomous validation — not tests authored for this task (the AAP explicitly excludes authoring tests). No fabricated or externally-sourced test data is included.

---

## 4. Runtime Validation & UI Verification

All items were exercised through their **real entry points** and captured verbatim. Status legend: ✅ Operational · ⚠ Partial · ❌ Failing.

**Service runtime health (Q2 — migrated + initialized database):**

- ✅ **Webapp** (`python server.py`, port **7777**) — `GET /health` returned `success` / HTTP 200; TCP connect to `:7777` succeeded (`connect_ex = 0`). **REQUIRED.**
- ✅ **Email handler** (`python email_handler.py`, port **20381**) — emitted `Listen for port 20381` and `Start mail controller 0.0.0.0 20381`; live SMTP banner `220 … Python SMTP 1.4.2` on `:20381`. **REQUIRED.**
- ✅ **Job runner** (`python job_runner.py`) — process stays alive polling the `Job` table; binds no port. **AUXILIARY.**
- ✅ **Event listener** (`python event_listener.py <listener|dead_letter>`) — starts a PostgreSQL event consumer; binds no port; requires an explicit subcommand. **AUXILIARY.**
- ✅ **Cron** (`python cron.py`) — one-shot scheduler-driven maintenance (15 jobs enumerated in `crontab.yml`); not a persistent daemon. **AUXILIARY.**

**Failure-mode & rejection validation:**

- ✅ **Q1 — Empty-database error:** `python server.py` starts against an empty-but-existing DB (import-time `engine.connect()` succeeds) and serves `GET /auth/login` (200, no query); the login **POST** raises `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable: relation "users" does not exist` → **HTTP 500** via the global `@app.errorhandler(Exception)`. Reproduced ×2.
- ✅ **Q3 — Skipped-initialization rejection:** with `public_domain` empty, a live message to `xyz@sl.local` returns **`550 SL E515 Email not exist`** with logs `alias xyz@sl.local not exist…` → `Cannot auto-create custom domain alias…` → `Cannot auto-create … no directory separator` → `alias … cannot be created on-the-fly, return 550`. Reproduced ×3. Causal nuance empirically proven (seeding `public_domain` still yields 550 for a plain address).

**API integration outcomes:**

- ✅ **Health endpoint** — `GET http://localhost:7777/health` → `success` (200).
- ✅ **Live SMTP path** — real `aiosmtpd` listener accepted `RCPT TO` (250) and returned the `550` end-of-DATA reply through the genuine handler chain (no bypass).

**UI verification:**

- ⚠ **Limited by design.** This is a backend/CLI bring-up investigation. The only UI surface exercised is the **login page**: `GET /` → 302 → `/auth/login` → 200 (renders without a DB query); the failure surfaces only on form submission (Q1). No broader UI verification was in scope.

---

## 5. Compliance & Quality Review

Cross-map of AAP deliverables and `SWE-AtlasQnA-Repo` mandates to their validation status. Fixes applied during autonomous validation are noted.

| Benchmark / AAP Requirement | Status | Progress | Evidence / Notes |
|---|---|---|---|
| Deliverable at exact path `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | 100% | File present (1,018 lines); `git status` shows `A` |
| Q1 — full verbatim traceback | ✅ Pass | 100% | §2.5 unabridged `ProgrammingError`/`UndefinedTable`; first failing relation `users` |
| Q2 — required services + startup output + port binds | ✅ Pass | 100% | §3.2–3.3; 2 required / 3 auxiliary; binds confirmed |
| Q3 — SMTP status code + rejection logs + causal nuance | ✅ Pass | 100% | §4.3–4.6; `550 SL E515`; empirical causal proof |
| Run-first methodology (observe → assert) | ✅ Pass | 100% | ~50 verbatim evidence blocks; validator re-ran independently |
| One-claim-one-evidence, verbatim, no paraphrase | ✅ Pass | 100% | Each claim paired with its evidence block |
| Cause → effect explanations | ✅ Pass | 100% | §2.6, §3.3, §4.6 |
| Exact `file:line` citations | ✅ Pass | 100% | ~40+ citations; **1 off-by-one citation fixed** during review (commit `1f959d09`) |
| Exhaustiveness / coverage pass | ✅ Pass | 100% | §7 checklist 10/10 |
| Canonical/default configuration + exact commands | ✅ Pass | 100% | §1.1–1.3; only deviation is DB host port 15432 (documented) |
| Real entry points only (no bypass hooks) | ✅ Pass | 100% | Live `aiosmtpd` injection; real HTTP POST |
| Reproducibility (≥2 stable runs) | ✅ Pass | 100% | Q1 ×2, Q2 ×2, Q3 ×3 |
| Read-only scope (no source modified) | ✅ Pass | 100% | `git diff --name-status` = single `A` entry |
| Temporary artifacts removed; repo pristine | ✅ Pass | 100% | Clean tree; `/tmp/blitzy_val` & `/tmp/blitzy_repro` absent; ports free |
| User verbatim phrasings preserved | ✅ Pass | 100% | `python server.py`, `init_app.py`, `@sl.local`, "SLDomain table should be empty" |
| Manifest-pin casing consistency | ✅ Pass | 100% | **Corrected during review** (commit `1f959d09`) |

**Outstanding compliance items:** none. All autonomous quality gates passed; the two remaining items (Section 2.2) are human governance, not compliance gaps.

---

## 6. Risk Assessment

Risk profile is inherently **Low** — a read-only documentation task with zero source changes and a fully-validated deliverable. There are **no High or Critical risks**.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Environment-specific literals (absolute paths, PIDs, timestamps) in captured traceback/logs differ on re-run | Technical | Low | High | Document explicitly states only timestamp/PID/path/timing vary; the categorical result (error type, status code, log line) is stable and was reproduced 2–3× | Mitigated |
| Citation line numbers drift as the codebase evolves | Technical | Low | Medium | Document is scoped to branch `app_2cd6ee777f8c` at baseline commit `2cd6ee77`; treat as a point-in-time snapshot | Accepted (by design) |
| Dependency version-string drift (e.g., `aiosmtpd 1.4.2` SMTP banner) on re-resolution | Technical | Low | Low | Observed values reported alongside manifest pins; banner text is cosmetic | Mitigated |
| Example-env placeholder secrets quoted verbatim (`FLASK_SECRET=secret`, `myuser:mypassword`) | Security | Low | Low | Verified these are the **shipped public `example.env` dev placeholders**, not real secrets; `.gitignore` covers `.env`/`.venv`; no production secret exposed | Mitigated |
| Documented app behaviors (`debug=True` traceback to stdout; verbose HTTP 500) are observations, not fixes | Security | Informational | N/A | AAP scope is observe-not-remediate; no code changed; flagged for team awareness only | Accepted (out of scope) |
| Document staleness / no automated accuracy gate | Operational | Low | Medium | Fully reproducible commands embedded; independently validated; branch/date-scoped | Mitigated |
| Reproduction depends on external services (PostgreSQL 13, Redis 6) and specific ports (7777/20381/15432) | Integration | Low | Low | Development guide (Section 9) documents exact versions, ports, and commands; validated end-to-end | Mitigated |

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Hours — Completed vs Remaining
    "Completed Work" : 33
    "Remaining Work" : 3
```

> **Integrity check:** "Remaining Work" = **3 h**, identical to Section 1.2 Remaining Hours and the sum of the Section 2.2 Hours column. "Completed Work" = **33 h**, identical to Section 1.2 Completed Hours.

**Remaining hours by category** (from Section 2.2):

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Remaining Hours by Category (total 3h)
    "SME accuracy review" : 2
    "PR review & merge" : 1
```

**Completion gauge:** 33 h of 36 h complete → **91.7%**. All remaining effort is Medium-priority human governance; no High-priority or blocking work remains.

---

## 8. Summary & Recommendations

**Achievements.** The project delivered a rigorously evidence-grounded answer document for all three developer-environment bring-up questions. Q1 captures the full `sqlalchemy.exc.ProgrammingError` / `psycopg2.errors.UndefinedTable: relation "users" does not exist` traceback and pinpoints that it surfaces on the first table-touching request (login POST), not at startup. Q2 establishes that exactly two services are required — the webapp (`:7777`) and the email handler (`:20381`) — corroborated by both observed port binds and SimpleLogin's own "2 main components" statement. Q3 reports the exact `550 SL E515 Email not exist` rejection with its log lines and, notably, proves the true cause is a missing non-auto-creatable alias rather than the empty `SLDomain` table.

**Remaining gaps & critical path.** No technical gaps remain. The critical path to production is purely human governance: (1) SME technical-accuracy review of the document (2 h), then (2) PR review and merge (1 h) — 3 h total.

**Production-readiness assessment.** The deliverable is **production-ready as committed** (`1f959d09`). It is internally consistent, fully reproducible, read-only-compliant, and passed all five autonomous validation gates with zero code defects. The full regression suite (639 tests) is green, confirming the investigation left the codebase healthy.

**Success metrics.**

| Metric | Target | Actual |
|---|---|---|
| AAP-scoped completion | ≥ 90% before human review | **91.7%** |
| Questions answered with verbatim evidence | 3 / 3 | **3 / 3** |
| Source files modified | 0 | **0** |
| Regression suite pass rate | 100% | **100% (639/639)** |
| Scenario reproducibility | ≥ 2 runs each | **Q1 ×2, Q2 ×2, Q3 ×3** |
| Coverage-pass checklist | 10 / 10 | **10 / 10** |

**Overall:** The project is **91.7% complete**, with the remaining 8.3% representing standard, non-blocking human sign-off. Recommendation: proceed directly to SME review and merge.

---

## 9. Development Guide

This guide reproduces the exact environment used for the investigation. All commands were verified in the canonical Docker environment. Use `./.venv/bin/python` directly, or `source .venv/bin/activate` first so `python` resolves to the project interpreter.

### 9.1 System Prerequisites

- **Python 3.10** (observed `3.10.20` via pyenv) — pinned `^3.10` in `pyproject.toml`; base image `FROM python:3.10`.
- **Poetry 1.8.5** — dependency manager.
- **Docker** — to run the backing services.
- **PostgreSQL 13** (observed `13.23`) and **Redis 6** — data infrastructure.

### 9.2 Environment Setup

Start the backing services (as used in this environment — PostgreSQL publishes container `5432` on host `15432`):

```bash
# PostgreSQL 13 (host port 15432 -> container 5432)
docker run -d --name sl-db \
  -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword -e POSTGRES_DB=simplelogin \
  -p 15432:5432 postgres:13

# Redis 6
docker run -d --name sl-redis -p 6379:6379 redis:6-alpine
```

Create the canonical local settings file and point `DB_URI` at the running database:

```bash
cp example.env .env
# In .env, set the DB host port to match the container:
# DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
```

Required environment variables (hard-required at import — the app refuses to start without them): `URL`, `EMAIL_DOMAIN` (`sl.local`), `SUPPORT_EMAIL`, `DB_URI`, `FLASK_SECRET`. All ship with canonical values in `example.env`.

### 9.3 Dependency Installation

```bash
# Install into an in-project virtualenv (./.venv)
poetry sync            # or: poetry install
source .venv/bin/activate

# Verify the toolchain
python --version       # Python 3.10.20
alembic --version      # alembic 1.4.3
pytest --version       # pytest 7.3.1
```

### 9.4 Application Startup (canonical order)

```bash
# 1) Run migrations (creates all 77 tables; head = 32f25cbf12f6)
./.venv/bin/alembic upgrade head

# 2) Seed initial data (populates public_domain / SLDomain from ALIAS_DOMAINS)
./.venv/bin/python init_app.py

# 3) Start the webapp — REQUIRED (dev server on :7777)
./.venv/bin/python server.py &

# 4) Start the email handler — REQUIRED (SMTP on :20381)
./.venv/bin/python email_handler.py &

# 5) Auxiliary services (optional for basic operation)
./.venv/bin/python job_runner.py &
./.venv/bin/python event_listener.py listener &   # subcommand required
# cron.py is scheduler-driven (one process per job), not a persistent daemon
```

> **Production note:** the webapp runs under gunicorn in production — `gunicorn wsgi:app -b 0.0.0.0:7777` (`wsgi.py` exposes `app = create_app()`).

### 9.5 Verification

```bash
# Webapp readiness
curl -s -w " HTTP %{http_code}\n" http://localhost:7777/health
# Expected: success HTTP 200

# Email handler readiness (SMTP banner on :20381)
python3 -c "import socket; s=socket.socket(); s.settimeout(5); s.connect(('127.0.0.1',20381)); print(repr(s.recv(1024).decode().strip())); s.close()"
# Expected: '220 <host> Python SMTP 1.4.2'

# Seed applied (after init_app.py)
docker exec sl-db psql -U myuser -d simplelogin -c "SELECT count(*) FROM public_domain;"
# Expected: 1
```

### 9.6 Example Usage — Reproducing the Three Scenarios

```bash
# --- Q1: empty-database error ---
# With a fresh, unmigrated DB (no `alembic upgrade head`):
./.venv/bin/python server.py &                     # starts OK, binds :7777
curl -s -o /dev/null -w "GET  /auth/login -> %{http_code}\n" http://localhost:7777/auth/login   # 200 (renders)
# Submitting the login form (POST) triggers the error -> HTTP 500:
#   sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist

# --- Q3: skipped-initialization rejection ---
# With migrations run but init_app.py SKIPPED (public_domain empty):
./.venv/bin/python email_handler.py &              # binds :20381
# Injecting a real message to xyz@sl.local via SMTP returns:
#   550 SL E515 Email not exist
#   log: alias xyz@sl.local cannot be created on-the-fly, return 550
```

### 9.7 Running the Test Suite

```bash
CONFIG=tests/test.env ./.venv/bin/pytest tests -q
# Expected: 639 passed, 51 warnings (exit 0)
```

### 9.8 Troubleshooting

- **`relation "users" does not exist` (or any table)** → migrations were not run. Fix: `./.venv/bin/alembic upgrade head`.
- **Every inbound email returns `550 SL E515 Email not exist`** → the recipient alias does not exist and cannot be auto-created. For a *plain* `@sl.local` address this is expected and is **independent of** the empty `public_domain` table; running `init_app.py` populates `public_domain` (which gates reverse-alias/domain-validation paths), but a bare address still needs an actual alias or an auto-create rule (custom domain or directory).
- **`Address already in use` on :7777 or :20381** → a previous process is still bound. Inspect with `lsof -i :7777` / `lsof -i :20381` and stop that specific PID.
- **`error: externally-managed-environment` from pip** → do not use system pip; use the in-project `./.venv` (created by Poetry) or `--break-system-packages` only if intentionally installing globally.
- **`psycopg2 … connection refused`** → ensure the `sl-db` container is up and `DB_URI`'s port matches the published host port (`15432` in this environment).
- **Missing `URL`/`EMAIL_DOMAIN`/`DB_URI`/`FLASK_SECRET`** → the app raises at import. Ensure `.env` exists (`cp example.env .env`) and is complete.

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---|---|
| Install dependencies | `poetry sync` (or `poetry install`) |
| Activate virtualenv | `source .venv/bin/activate` |
| Run migrations | `./.venv/bin/alembic upgrade head` |
| Seed data | `./.venv/bin/python init_app.py` |
| Start webapp (dev) | `./.venv/bin/python server.py` |
| Start webapp (prod) | `gunicorn wsgi:app -b 0.0.0.0:7777` |
| Start email handler | `./.venv/bin/python email_handler.py` |
| Start job runner | `./.venv/bin/python job_runner.py` |
| Start event listener | `./.venv/bin/python event_listener.py listener` |
| Health check | `curl -s http://localhost:7777/health` |
| Run tests | `CONFIG=tests/test.env ./.venv/bin/pytest tests -q` |
| Verify read-only scope | `git diff --name-status origin/app_2cd6ee777f8c..HEAD` |

### B. Port Reference

| Port | Service | Notes |
|---|---|---|
| 7777 | Webapp (`server.py` / `wsgi:app`) | Flask dev server (`debug=True`); `/health` endpoint |
| 20381 | Email handler (`email_handler.py`) | `aiosmtpd` SMTP listener (default `--port 20381`) |
| 15432 | PostgreSQL 13 (host) | Published from container `5432`; used in `DB_URI` |
| 6379 | Redis 6 | Used by most tests and rate-limiting |

### C. Key File Locations

| Path | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** — runtime-behavior Q&A |
| `server.py` | Web entry point; `local_main()` (:7777); global `@app.errorhandler(Exception)` |
| `wsgi.py` | Production WSGI callable `app = create_app()` |
| `email_handler.py` | `aiosmtpd` SMTP handler; `handle_forward()` rejection path |
| `app/db.py` | SQLAlchemy engine; import-time `engine.connect()` |
| `app/config.py` | `.env` loading; required env vars; `ALIAS_DOMAINS` |
| `app/auth/views/login.py` | Login view exercised in Q1 |
| `app/email/status.py` | `E515 = "550 SL E515 Email not exist"` |
| `app/alias_utils.py` | `try_auto_create()` and strategies (Q3) |
| `app/email_utils.py` | Directory/domain gates (Q3 causal nuance) |
| `app/models.py` | `SLDomain` model (`__tablename__ = "public_domain"`) |
| `init_app.py` | `add_sl_domains()` seed (run for Q2, skipped for Q3) |
| `migrations/` | 255 Alembic versions (`alembic upgrade head`) |
| `example.env` | Canonical dev configuration |

### D. Technology Versions

| Component | Version | Source |
|---|---|---|
| Python | 3.10.20 | observed / `pyproject.toml ^3.10` |
| Flask | 1.1.2 | `pyproject.toml ^1.1.2` |
| SQLAlchemy | 1.3.24 | `pyproject.toml 1.3.24` |
| psycopg2-binary | 2.9.3 | `pyproject.toml ^2.9.3` |
| Flask-Migrate | 2.5.3 | `pyproject.toml ^2.5.3` |
| alembic | 1.4.3 | observed |
| aiosmtpd | 1.4.2 | observed (manifest pins `^1.2`) |
| python-dotenv | 0.14.0 | `pyproject.toml ^0.14.0` |
| Poetry | 1.8.5 | observed |
| PostgreSQL | 13.23 | observed |
| Redis | 6 | observed |
| pytest | 7.3.1 | observed |

### E. Environment Variable Reference

| Variable | Example Value | Required | Notes |
|---|---|---|---|
| `URL` | `http://localhost:7777` | Yes | App base URL |
| `EMAIL_DOMAIN` | `sl.local` | Yes | Folded into `ALIAS_DOMAINS`; the `@sl.local` domain in Q3 |
| `SUPPORT_EMAIL` | `support@sl.local` | Yes | Support address |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:15432/simplelogin` | Yes | Only deviation from `example.env` is host port `15432` |
| `FLASK_SECRET` | `secret` | Yes | App refuses to start if empty (shipped example placeholder) |

> These are the shipped **public** `example.env` dev placeholders; `.env` and `.venv` are gitignored. No production secrets are involved.

### F. Developer Tools Guide

| Tool | Use |
|---|---|
| Poetry | Dependency management → `./.venv` |
| Alembic | Schema migrations (`upgrade head`, 255 versions) |
| pytest | Test runner (`CONFIG=tests/test.env`, 639 tests) |
| Docker | Backing services (`postgres:13`, `redis:6-alpine`) |
| `psql` / `psycopg2` | Direct DB inspection (e.g., `SELECT count(*) FROM public_domain;`) |
| `lsof` | Port inspection (`lsof -i :7777`, `lsof -i :20381`) |
| `git diff --name-status` | Read-only scope verification |

### G. Glossary

| Term | Definition |
|---|---|
| **AAP** | Agent Action Plan — the governing specification for this task |
| **SLDomain / `public_domain`** | The model/table (`__tablename__ = "public_domain"`) that stores SimpleLogin's public alias domains; seeded by `init_app.py` |
| **`ALIAS_DOMAINS`** | Env-derived list of alias domains (includes `EMAIL_DOMAIN`); distinct from the `SLDomain` table |
| **E515** | SMTP status constant `"550 SL E515 Email not exist"` |
| **E207** | SMTP status constant `"250 SL E207 No bounce report"` (ignore-bounce variant of the rejection) |
| **`try_auto_create()`** | Alias auto-creation entry (custom-domain and directory strategies); returns `None` for a plain non-matching address → 550 |
| **Auxiliary service** | A process (`job_runner`, `event_listener`, `cron`) not required for the core webapp + email-handler function |
| **Head revision** | Latest Alembic migration; here `32f25cbf12f6` (77 tables) |

---

*End of Blitzy Project Guide. Completion: **91.7%** (33 h completed / 3 h remaining / 36 h total). All cross-section integrity rules validated.*