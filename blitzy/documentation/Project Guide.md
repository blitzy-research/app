# Blitzy Project Guide — SimpleLogin Runtime Behavior Q&A (`app_2cd6ee777f8c`)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, **runtime-grounded onboarding document** that explains how the SimpleLogin Flask application actually behaves when it is built and run — answered from real, captured execution output rather than static reading. The target users are engineers onboarding to the SimpleLogin codebase who need verifiable answers to five behavioral questions: the app's bind port, its startup logs, the health-check response, the alias-creation API response plus its database write, and the failure mode when PostgreSQL is unavailable. The technical scope spans the WSGI entrypoint, the health route, the REST alias API, the ORM/persistence layer, the database session bootstrap, and the configuration/logging subsystems — all exercised read-only. The business impact is faster, evidence-based developer onboarding with zero risk to production code.

### 1.2 Completion Status

The completion percentage is computed using the AAP-scoped hours methodology (completed hours / total hours). The entire AAP engineering deliverable is complete and independently validated; only human-side path-to-production (review, merge, and an optional baseline re-capture) remains.

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Completion Status — 91.9% Complete
    "Completed Work (hrs)" : 34
    "Remaining Work (hrs)" : 3
```

> **Color legend (Blitzy brand):** Completed Work = Dark Blue `#5B39F3`; Remaining Work = White `#FFFFFF`.

| Metric | Value |
|--------|-------|
| **Total Hours** | **37** |
| **Completed Hours (AI + Manual)** | **34** (34 AI-autonomous + 0 Manual) |
| **Remaining Hours** | **3** |
| **Percent Complete** | **91.9%** (34 / 37) |

### 1.3 Key Accomplishments

- ✅ **Single mandated deliverable produced:** `blitzy/documentation/app_2cd6ee777f8c.md` (963 lines), named after the source branch, under the new `blitzy/documentation/` directory.
- ✅ **Read-only mandate perfectly honored:** exactly one file added (`git diff base..HEAD` = 1 file, +963/-0); working tree clean; zero source/config/test/migration files modified.
- ✅ **Run-first methodology applied:** every answer captured from live execution inside the pinned Docker stack (Python 3.10, SQLAlchemy 1.3.24, Flask 1.1.2, gunicorn 20.0.4, psycopg2 2.9.3), not from static reading.
- ✅ **All five questions (+ the Q6 "actual output" constraint) answered** with verbatim evidence and ~70 exact `file:line` citations, closed out by a coverage pass.
- ✅ **Independently validated:** the Final Validator reproduced every documented runtime claim against live output with **zero discrepancies**; relevant source unit tests pass **61/61**.
- ✅ **Two review cycles resolved:** 6 code-review findings and 2 QA document-integrity findings addressed across commits `406bf67f` and `8811dda9`.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None — no blocking issues identified._ The deliverable is validated production-ready. | N/A | N/A | N/A |

> The only outstanding items are standard human path-to-production steps (review, merge) and one optional fidelity task — none of which block correctness. See Sections 1.6 and 2.2.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| SimpleLogin source repository | Git read/write | None — branch `blitzy-ac2e5a10-...` accessible; working tree clean | ✅ No issue | — |
| Capture Docker image (`ghcr.io/scaleapi/swe-atlas:...QnA_simple-login_app_1.0`) | Container pull/run | None — image available; stack ran successfully during capture and validation | ✅ No issue | — |
| PostgreSQL / Redis (capture env) | Service connectivity | None — both up during capture (PG 15.13, Redis 7.0.15); schema at head | ✅ No issue | — |

**No access issues identified.** All resources required for the investigation and validation were available.

### 1.6 Recommended Next Steps

1. **[High]** Have an SME review and accept `blitzy/documentation/app_2cd6ee777f8c.md`, confirming each Q1–Q5 answer meets the team's onboarding need (1.5h).
2. **[Medium]** Merge the PR into the main branch and close out the working branch (0.5h).
3. **[Low]** _Optional:_ re-capture Q1/Q2/Q4/Q5 output on the exact deployment baseline (PostgreSQL 13 / Redis 6) for absolute version fidelity — answers are unchanged, so this is polish (1.0h).

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All completed hours are AI-autonomous work by Blitzy agents. Each component traces to an AAP requirement (run-first investigation + document authoring + validation).

| Component | Hours | Description |
|-----------|-------|-------------|
| Runnable-stack standup | 4.0 | Stand up the pinned stack in Docker: PostgreSQL + Redis services, virtualenv with pinned deps, Alembic schema via CLI (`alembic upgrade head`), `CONFIG=tests/test.env` profile. |
| Q1 — bind-port capture | 2.0 | Capture Gunicorn `0.0.0.0:7777` (all-interfaces) and dev-server `127.0.0.1:7777` (loopback); prove Werkzeug "Running on" suppression. |
| Q2 — startup-logs capture | 2.5 | Capture import banners (`load config file`, `>>> URL:`, `>>> init logging <<<`), SL DEBUG logger introspection, werkzeug-disabled proof, Gunicorn boot lines, ordering rationale. |
| Q3 — health-endpoint capture | 1.0 | Capture body `success` (7 bytes via `od`), status `200`, `Content-Type: text/html`, `Content-Length: 7`. |
| Q4a — alias-JSON capture | 4.0 | Seed User+ApiKey+default Mailbox under free-plan cap; `Authentication` header; POST `/api/alias/random/new` -> `201`; enumerate full v2 field set; prove compact-JSON serialization. |
| Q4b — DB-write capture | 3.0 | `psql` alias-row query, `to_regclass` table confirmation, mailbox lookup, column-by-column mapping (incl. `mailbox_id = default_mailbox_id`). |
| Q5 — DB-unavailable capture | 2.5 | Point `DB_URI` at a dead port; capture full 107-line chained traceback; prove import-time break at `app/db.py:L12` before `create_app()`; note libpq phrasing variance. |
| Answer-document authoring | 6.0 | Write the 963-line Markdown with ~70 `file:line` citations and verbatim-evidence organization across per-question sections. |
| Coverage pass + read-only cleanup/verification | 2.0 | Decompose Q1–Q6, confirm each sub-part; remove temp scripts; verify `git status --porcelain` empty. |
| Code-review & QA remediation | 4.0 | Resolve 6 code-review findings (`406bf67f`) + 2 QA document-integrity findings (`8811dda9`) with re-captured runtime evidence. |
| Final runtime re-validation | 3.0 | Reproduce every claim live in Docker (zero discrepancies); relevant source unit tests 61/61. |
| **Total Completed** | **34.0** | |

### 2.2 Remaining Work Detail

All remaining work is human-side path-to-production. No engineering fixes remain (nothing is broken).

| Category | Hours | Priority |
|----------|-------|----------|
| Human/SME review & acceptance of the onboarding doc (Q1–Q5 fitness) | 1.5 | High |
| PR merge to main & branch close-out | 0.5 | Medium |
| _Optional:_ re-capture on exact baseline (PostgreSQL 13 / Redis 6) for version fidelity (answers unchanged) | 1.0 | Low |
| **Total Remaining** | **3.0** | |

### 2.3 Hours Reconciliation

- Completed (2.1) **34.0** + Remaining (2.2) **3.0** = **37.0** Total Hours (matches Section 1.2). OK
- Remaining **3.0** is identical in Section 1.2, Section 2.2, and the Section 7 pie chart. OK
- Percent complete = 34.0 / 37.0 = **91.9%** (matches Section 1.2 and Section 7). OK

---

## 3. Test Results

All entries below originate from Blitzy's autonomous validation logs for this project. The deliverable is a Markdown document (no unit tests of its own); its acceptance criteria are the runtime-grounded verifications, complemented by the source unit tests that exercise the documented code paths.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Unit — Q4 alias-endpoint paths | pytest | 61 | 61 | 0 | n/a | `test_new_random_alias.py` (exact Q4 endpoint), `test_new_custom_alias.py`, `test_alias.py`, `tests/test_models.py` — directly exercise the Q4a/Q4b create->commit->serialize->persist cycle. Exit 0. |
| Runtime Verification — Q1–Q6 | curl / psql / python (manual capture) | 6 | 6 | 0 | 100% | Each question reproduced against live output; zero discrepancies. Q4a `Content-Length: 448` reproduced first try; Q5 break confirmed at `app/db.py:12` (`CREATE_APP_REACHED=False`). |
| Source-citation validation | shell (`grep` + `test -f`) | 14 | 14 | 0 | 100% | All distinct repository `file:line` anchors resolve to real files; 3 non-repo refs are intentional (temp script, venv library, bare filename). |
| **Deliverable-relevant total** | — | **81** | **81** | **0** | — | 100% pass on all deliverable-relevant checks. |

**Full-suite context (out of scope, informational):** The entire source suite (`pytest tests/`, 639 items) does not reach 100% in the capture container due to **environmental** limits — network-dependent tests (no internet) and a socket-server/`--timeout` interaction — not deliverable issues. The setup baseline was **636 passed / 3 failed (99.5%)**; those 3 pre-existing failures are in subsystems unrelated to Q1–Q5 and, under the strict read-only mandate, are correctly not fixed.

---

## 4. Runtime Validation & UI Verification

**Runtime health (captured live inside the pinned Docker stack):**

- ✅ **Operational** — Gunicorn production server boots and binds `0.0.0.0:7777` (`Listening at: http://0.0.0.0:7777`); serves HTTP 200.
- ✅ **Operational** — Development server (`python server.py`) binds `127.0.0.1:7777` (loopback); `Server: Werkzeug/1.0.1 Python/3.10.18`.
- ✅ **Operational** — `GET /health` -> `200`, body `success` (7 bytes), `Content-Type: text/html; charset=utf-8`, `Content-Length: 7`.
- ✅ **Operational** — `POST /api/alias/random/new` (authenticated) -> `201 CREATED`, `Content-Type: application/json`, `Content-Length: 448`; row persisted to table `alias`.
- ✅ **Operational (expected-failure path correctly reproduced)** — with PostgreSQL unreachable, import raises `sqlalchemy.exc.OperationalError` (wrapping `psycopg2.OperationalError`) at `app/db.py:L12`, at import time before `create_app()` runs.

**API integration:**

- ✅ **Operational** — Health endpoint `/health`.
- ✅ **Operational** — Alias REST endpoint `/api/alias/random/new` with header-based API-key authentication (`Authentication` -> `ApiKey.get_by(code=...)`).

**UI verification:**

- ⚠ **Not applicable** — This is a developer-facing Markdown deliverable with no UI. The SimpleLogin dashboard UI is explicitly out of scope; `/health` returns plain text, not a rendered page. No UI screens were in scope for capture.

---

## 5. Compliance & Quality Review

Cross-mapping of the AAP rule set **"SWE-AtlasQnA-Repo"** and quality benchmarks to observed evidence.

| Requirement / Benchmark | Status | Progress | Evidence |
|-------------------------|--------|----------|----------|
| Deliverable location & name (`blitzy/documentation/app_2cd6ee777f8c.md`) | ✅ Pass | 100% | File present (963 lines), branch-named, new directory created. |
| Run-first methodology (build/run before writing) | ✅ Pass | 100% | Every answer sourced from captured live output in the pinned Docker stack. |
| Verbatim evidence with command provenance | ✅ Pass | 100% | Log lines, HTTP responses+headers, DB rows, tracebacks each shown with the producing command. |
| Answer every sub-part (Q1–Q6, incl. Q4a & Q4b) | ✅ Pass | 100% | Coverage pass checks off all questions and sub-parts. |
| Exact grounding (`file:line` for every literal) | ✅ Pass | 100% | ~70 citations; all repo anchors resolve; validator confirmed exact. |
| Read-only scope (no source modified; no retained scripts) | ✅ Pass | 100% | `git diff base..HEAD` = 1 file added; working tree clean; temp scripts removed. |
| Eager `engine.connect()` documented, not "fixed" | ✅ Pass | 100% | `app/db.py:L12` behavior documented as the Q5 trigger; not altered. |
| Code-review findings resolved | ✅ Pass | 100% | 6 findings (F1–F6) resolved in `406bf67f` with re-captured evidence. |
| QA document-integrity findings resolved | ✅ Pass | 100% | 2 findings resolved in `8811dda9` (non-repo citation removed; reproducible Alembic CLI). |
| Markdown well-formedness | ✅ Pass | 100% | Balanced code fences; final newline; no trailing whitespace. |

**Fixes applied during autonomous validation:** provenance corrected (real detached-HEAD git output), dev-server output added with loopback proof, Q4a raw-vs-pretty JSON separated, `psql` made non-interactive, full 107-line Q5 traceback restored, non-repository citation reframed as a labeled third-party observation, and the non-reproducible `flask db upgrade` replaced with working Alembic CLI commands. **Outstanding items:** none affecting the deliverable.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Baseline version drift (captured on PG 15.13/Redis 7.0.15 vs baseline PG 13/Redis 6) | Technical | Low | Low | Doc explicitly notes the variance; answers unchanged; optional baseline re-capture available | Documented / Mitigated |
| Ephemeral runtime values differ per run (timestamps, pids, cookies, alias id, generated email) | Technical | Low | Medium | Doc marks these as per-run; structural values (port 7777, 200/201, `alias` table, column set) are stable | Documented / Mitigated |
| libpq phrasing variance in the Q5 error message | Technical | Low | Low | Doc notes newer-vs-older libpq "Connection refused" phrasing | Documented / Mitigated |
| Credential exposure in captured output (60-char `ApiKey.code`, session cookie) | Security | Low | Low | 60-char code redacted; only `tests/test.env` test credentials used; no production secrets | Mitigated |
| Documentation staleness as source evolves (line numbers/values drift) | Operational | Medium | Medium | Doc pinned to commit `2cd6ee77` in header; recommend re-validation when cited files change materially | Open (inherent to docs) |
| `flask db upgrade` non-reproducibility (`KeyError: 'migrate'`) | Operational | Low | Low | Doc provides working Alembic CLI commands + provenance note | Documented / Mitigated |
| Exact-output reproduction requires pinned image + running PG/Redis + seeded schema | Integration | Low | Low | Doc names exact image + commit and documents full seeding prerequisites | Documented / Mitigated |
| 3 pre-existing full-suite test failures + network-dependent hangs | Integration | Low | Low | Environmental / unrelated subsystems; out of scope under read-only mandate; do not affect deliverable | Out of scope / No action |

**Overall risk posture: LOW.** No blocking risks. The most notable is ordinary documentation staleness over time.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 37h — 91.9% Complete)
    "Completed Work" : 34
    "Remaining Work" : 3
```

> **Color legend (Blitzy brand):** Completed Work = Dark Blue `#5B39F3`; Remaining Work = White `#FFFFFF`.

**Remaining hours by category (from Section 2.2, totaling 3.0h):**

| Category | Hours | Priority | Relative Bar |
|----------|-------|----------|--------------|
| Human/SME review & acceptance | 1.5 | High | +++++++++++++++ |
| Optional baseline re-capture (PG 13/Redis 6) | 1.0 | Low | ++++++++++ |
| PR merge & branch close-out | 0.5 | Medium | +++++ |
| **Total** | **3.0** | | |

> **Integrity check:** Section 7 "Remaining Work" = **3** = Section 1.2 Remaining Hours = Section 2.2 total. OK

---

## 8. Summary & Recommendations

**Achievements.** The project fully delivers its single mandated artifact — a 963-line, evidence-backed runtime Q&A that answers all five behavioral questions (plus the Q6 "actual output" constraint) about the SimpleLogin Flask app. Every value the questions ask for is quoted as a literal and grounded with an exact `file:line` citation, and the whole document was produced run-first from captured live output. The read-only mandate was honored perfectly: exactly one file was added and no source, config, test, or migration file was touched.

**Remaining gaps.** No engineering work remains. The outstanding 3.0 hours are human path-to-production: SME review/acceptance (High), PR merge (Medium), and an optional exact-baseline re-capture (Low, answers unchanged).

**Critical path to production.** Review -> accept -> merge. This is a short, low-risk path because the deliverable has already been independently validated with zero discrepancies and the relevant source unit tests pass 61/61.

**Success metrics.** All five questions answered and coverage-passed; ~70 citations resolve exactly; runtime claims reproduced live (Q4a `Content-Length 448`, Q5 break at `app/db.py:12`); read-only proof (`git status` clean, 1 file added).

**Production-readiness assessment.** At **91.9% complete (34 of 37 hours)**, the deliverable itself is **production-ready**; the remaining 8.1% is human review/merge and an optional fidelity task. Recommendation: **proceed to SME review and merge.**

| Metric | Value |
|--------|-------|
| Percent complete (AAP-scoped) | 91.9% |
| Completed / Total hours | 34 / 37 |
| Remaining hours | 3 |
| Blocking issues | 0 |
| Deliverable-relevant test pass rate | 81/81 (100%) |
| Overall risk posture | Low |

---

## 9. Development Guide

This guide documents how to build, run, reproduce, and validate the SimpleLogin runtime evidence underpinning the deliverable. Commands marked _(tested)_ were verified in the analysis environment; runtime commands are grounded in the repository's `Dockerfile` and `tests/test.env`.

### 9.1 System Prerequisites

- **OS:** Linux (the capture used the pinned Docker image).
- **Container image (authoritative capture env):** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (equivalently `andrewparkscaleai/coding-agent:simple-login__app__2cd6ee777f8c...`). The repo `Dockerfile` base is `FROM python:3.10`.
- **Language runtime:** Python **3.10** (`pyproject.toml:L61` `python = "^3.10"`).
- **Backing services:** PostgreSQL and Redis reachable at the configured URIs.
- **Key Python deps (pinned):** Flask 1.1.2, SQLAlchemy 1.3.24, psycopg2-binary 2.9.3, gunicorn 20.0.4, redis >=4.5.3.

### 9.2 Environment Setup

Select the config profile via `CONFIG` and ensure the DB/Redis URIs point at running services:

```bash
cd /app                          # repository root inside the container
. /app/venv/bin/activate         # activate the prepared virtualenv
export CONFIG=tests/test.env     # drives app/config.py -> prints "load config file ..."
# tests/test.env declares DB_URI on port 15432; export a working DB_URI on the live PG port:
export DB_URI="postgresql://test:test@localhost:5432/test"   # load_dotenv(override=False) keeps this
```

Relevant `tests/test.env` keys: `URL=http://localhost` (L2), `LOCAL_FILE_UPLOAD=1` (L3), `MAX_NB_EMAIL_FREE_PLAN=3` (L13), `DB_URI=...:15432/test` (L17), `FLASK_SECRET=secret` (L20), `MEM_STORE_URI=redis://localhost` (L78).

### 9.3 Dependency Installation

Dependencies are installed at image-build time via Poetry (see `Dockerfile`):

```bash
pip install -U pip
curl -sSL https://install.python-poetry.org | python3 -
poetry config virtualenvs.create false
poetry install --no-interaction --no-ansi --no-root
```

### 9.4 Schema Setup (IMPORTANT)

Use the **Alembic CLI** — do **not** use `flask db upgrade` (it raises `KeyError: 'migrate'` because Flask-Migrate is never registered):

```bash
CONFIG=tests/test.env alembic upgrade head   # exits 0
CONFIG=tests/test.env alembic current        # -> 32f25cbf12f6 (head)
CONFIG=tests/test.env alembic heads          # -> 32f25cbf12f6 (head)
```

### 9.5 Application Startup

```bash
# Production (all interfaces) — reproduces Q1/Q2/Q3/Q4:
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
#  -> Gunicorn logs: "Listening at: http://0.0.0.0:7777"

# Development (loopback) — reproduces Q1 loopback bind:
CONFIG=tests/test.env python server.py
#  -> app.run(debug=True, port=7777) binds 127.0.0.1:7777
```

### 9.6 Verification Steps

```bash
# Q3 — health endpoint (expect: HTTP 200, body "success", Content-Length 7):
curl -i http://127.0.0.1:7777/health

# Q1 — confirm the bound port/interface:
lsof -i :7777        # or:  curl -sI http://0.0.0.0:7777/health
```

### 9.7 Example Usage — Alias API (Q4) & DB Write

```bash
# Q4a — create a random alias (requires a seeded User + ApiKey + default Mailbox):
curl -sS -X POST http://127.0.0.1:7777/api/alias/random/new \
     -H "Authentication: <API_CODE>" \
     -H "Content-Type: application/json" -d '{}'
#  -> HTTP 201 CREATED, Content-Type: application/json  (compact, alpha-sorted keys)

# Q4b — inspect the persisted row (destination table = "alias"):
PGPASSWORD=test psql -h localhost -U test -d test \
  -c "SELECT id, email, user_id, mailbox_id, enabled, flags FROM alias ORDER BY id DESC LIMIT 1;"
```

### 9.8 Reproducing the Q5 Failure Path

```bash
# Point DB_URI at an unreachable port, then import the app (fails at IMPORT time):
DB_URI="postgresql://test:test@localhost:15432/test" python -c "import wsgi"
#  -> sqlalchemy.exc.OperationalError (wrapping psycopg2.OperationalError, "Connection refused")
#     raised at app/db.py:L12  `connection = engine.connect()`  — before create_app()
```

### 9.9 Validating the Deliverable (tested)

````bash
# Size & existence (tested -> 963):
wc -l blitzy/documentation/app_2cd6ee777f8c.md

# Read-only proof — exactly one file added vs base (tested -> "A  blitzy/documentation/app_2cd6ee777f8c.md"):
git diff --name-status 2cd6ee77..HEAD

# Working tree clean (tested -> empty):
git status --porcelain

# Code-fence balance (tested -> balanced):
grep -c '```' blitzy/documentation/app_2cd6ee777f8c.md

# Agent provenance (tested -> b5e1665e, 406bf67f, 8811dda9):
git log --author="agent@blitzy.com" --oneline 2cd6ee77..HEAD
````

### 9.10 Troubleshooting

- **`flask db upgrade` -> `KeyError: 'migrate'`** — expected; use `alembic upgrade head` instead (Flask-Migrate is declared but never registered).
- **App won't start / `OperationalError` at `app/db.py:12`** — PostgreSQL is unreachable at `DB_URI`; start PostgreSQL and confirm host/port. This is the eager import-time connection (the Q5 behavior).
- **Port 7777 already in use** — `lsof -i :7777`, stop the conflicting process, or bind a different port.
- **Redis errors at boot** — ensure `MEM_STORE_URI` (default `redis://localhost`) points at a running Redis.
- **Captured values differ from the doc** — timestamps, pids, cookies, alias id, and generated email are per-run; structural values (`7777`, `200`/`201`, table `alias`, the column/field set) are stable.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---------|---------|
| Apply schema | `CONFIG=tests/test.env alembic upgrade head` |
| Run production server | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Run dev server | `CONFIG=tests/test.env python server.py` |
| Health check | `curl -i http://127.0.0.1:7777/health` |
| Create alias (Q4a) | `curl -sS -X POST .../api/alias/random/new -H "Authentication: <API_CODE>" -d '{}'` |
| Inspect alias row (Q4b) | `PGPASSWORD=test psql -h localhost -U test -d test -c "SELECT ... FROM alias ...;"` |
| Reproduce Q5 | `DB_URI=...:15432/test python -c "import wsgi"` |
| Read-only proof | `git diff --name-status 2cd6ee77..HEAD` |

### Appendix B — Port Reference

| Port | Service | Interface | Source |
|------|---------|-----------|--------|
| 7777 | Gunicorn (production WSGI) | `0.0.0.0` (all interfaces) | `Dockerfile:L44,L47` |
| 7777 | Flask/Werkzeug (dev server) | `127.0.0.1` (loopback) | `server.py:L588` |
| 5432 | PostgreSQL (live capture) | localhost | capture env |
| 6379 | Redis | localhost | `MEM_STORE_URI` |

### Appendix C — Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (runtime Q&A) |
| `wsgi.py`, `server.py`, `Dockerfile` | Web entrypoint & boot (Q1, Q2) |
| `server.py` (`/health`) | Health route (Q3) |
| `app/db.py` | Eager DB connection — Q5 failure point (`:L12`) |
| `app/config.py`, `app/log.py` | Config loading & logging (Q2) |
| `app/api/base.py`, `app/api/views/new_random_alias.py`, `app/api/serializer.py` | Alias REST API (Q4a) |
| `app/models.py` | ORM models incl. `Alias` (`__tablename__="alias"`, Q4b) |
| `migrations/`, `tests/conftest.py`, `tests/test.env` | Schema & seeding prerequisites (Q4) |

### Appendix D — Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10 | `pyproject.toml:L61`; `Dockerfile:L8` |
| Flask | 1.1.2 | `pyproject.toml:L62` |
| SQLAlchemy | 1.3.24 | `pyproject.toml:L116` |
| psycopg2-binary | 2.9.3 | `pyproject.toml:L71` |
| gunicorn | 20.0.4 | `pyproject.toml:L66` |
| redis (client) | >=4.5.3 | `pyproject.toml:L117` |
| PostgreSQL (capture) | 15.13 | capture env (baseline: 13) |
| Redis (capture) | 7.0.15 | capture env (baseline: 6) |

### Appendix E — Environment Variable Reference

| Variable | Value (test profile) | Source | Purpose |
|----------|----------------------|--------|---------|
| `CONFIG` | `tests/test.env` | runtime | Selects the env file |
| `URL` | `http://localhost` | `tests/test.env:L2` | App base URL |
| `DB_URI` | `postgresql://test:test@localhost:15432/test` | `tests/test.env:L17` | Database DSN (export live port for capture) |
| `FLASK_SECRET` | `secret` | `tests/test.env:L20` | Flask session secret |
| `MEM_STORE_URI` | `redis://localhost` | `tests/test.env:L78` | Redis / rate-limiter store |
| `MAX_NB_EMAIL_FREE_PLAN` | `3` | `tests/test.env:L13` | Free-plan alias cap (seeding constraint) |
| `LOCAL_FILE_UPLOAD` | `1` | `tests/test.env:L3` | Local file storage toggle |

### Appendix F — Developer Tools Guide

| Tool | Use |
|------|-----|
| `alembic` | Apply/inspect DB schema (`upgrade head`, `current`, `heads`) |
| `gunicorn` | Production WSGI server (Q1/Q2 capture) |
| `curl -i` / `curl -sS -D` | Capture HTTP status, headers, body (Q3/Q4a) |
| `psql` (`PGPASSWORD=test`) | Inspect persisted rows non-interactively (Q4b) |
| `od -c` | Byte-exact body inspection (Q3 "success" = 7 bytes) |
| `python -c "import wsgi"` | Trigger import-time DB connection (Q5) |
| `git diff` / `git status` | Verify read-only mandate |

### Appendix G — Glossary

| Term | Definition |
|------|------------|
| **AAP** | Agent Action Plan — the authoritative project specification. |
| **Run-first methodology** | Building/running the code and capturing real output *before* writing the answer. |
| **Eager connection** | `app/db.py:L12` calls `engine.connect()` at module import, so DB failures surface at import time (before `create_app()`). |
| **v2 serializer** | `serialize_alias_info_v2` (`app/api/serializer.py`) — builds the Q4a JSON field set. |
| **Free-plan cap** | `MAX_NB_EMAIL_FREE_PLAN=3` — limits aliases per free user (Q4 seeding constraint). |
| **Coverage pass** | Final decomposition check confirming every question sub-part is answered. |
