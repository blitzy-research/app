# Blitzy Project Guide — SimpleLogin First-Time Local-Dev Startup Behavior (Q&A)

> **Brand legend:** Completed / AI Work = Dark Blue `#5B39F3` · Remaining / Not Completed = White `#FFFFFF` · Headings / Accents = Violet-Black `#B23AF2` · Highlight = Mint `#A8FDD9`

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single, evidence-based Markdown document that explains the **first-time local-development startup behavior of SimpleLogin** (an open-source email-alias service). It is consumed by **engineers onboarding to the codebase** who need to know exactly what happens when the stack is brought up from scratch and what breaks when setup steps are skipped. The document answers three scoped questions — the empty-database error, the required Python services and their readiness logs, and the SMTP rejection when initialization is skipped — each grounded in **verbatim runtime output** reconciled to the responsible source `file:line`. Scope is intentionally narrow and additive: exactly **one** new documentation file is committed and **no source code is modified**.

### 1.2 Completion Status

```mermaid
%%{init: {"theme":"base", "themeVariables": {"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeWidth":"2px","pieTitleTextSize":"18px","pieSectionTextSize":"15px","pieLegendTextSize":"14px"}}}%%
pie showData title Completion 89.3% (hours)
    "Completed Work (AI)" : 25
    "Remaining Work" : 3
```

<div align="center"><strong>89.3% Complete</strong></div>

| Metric | Hours |
|--------|-------|
| **Total Hours** | **28** |
| Completed Hours (AI + Manual) | 25 |
| Remaining Hours | 3 |

> **Calculation (PA1, AAP-scoped):** `Completion % = Completed ÷ (Completed + Remaining) = 25 ÷ 28 = 89.3%`. All completed hours were delivered autonomously by Blitzy agents; remaining hours are human path-to-production review/merge.

### 1.3 Key Accomplishments

- ✅ **Single deliverable created** at the exact required path/name: `blitzy/documentation/app_2cd6ee777f8c.md` (= source branch name), with the `blitzy/` and `blitzy/documentation/` directories created.
- ✅ **Q1 (empty-DB error) fully answered** — the complete, verbatim Python traceback (`sqlalchemy.exc.ProgrammingError` ← `psycopg2.errors.UndefinedTable: relation "users" does not exist`) captured, including the full SQL and bind parameters, plus a render-vs-submit analysis.
- ✅ **Q2 (required services) fully answered** — webapp (`127.0.0.1:7777`), email handler (`0.0.0.0:20381`), and job runner (no port) enumerated with **verbatim** startup banners, a `/health → success` probe, and `/proc/net/tcp` LISTEN proof for both ports.
- ✅ **Q3 (email rejection) fully answered** — `550 SL E515 Email not exist` captured with the full verbatim rejection log block, the SPF-override caveat, and a Mermaid decision-path diagram.
- ✅ **Evidence is genuine** — 52 verbatim code/console blocks captured from live runs; 149 `file:line` citations, all resolving to the cited source.
- ✅ **Zero source mutation** — `git diff` against the base commit is the single new file only (782 insertions, 0 deletions); working tree clean.
- ✅ **Independently validated** — all three scenarios were reproduced from scratch in the project's Docker runtime with zero discrepancies; 2/2 SPF corroboration tests pass; entry points compile cleanly.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None_ — no blocking issues identified. The single deliverable is complete, committed, and independently validated. | None | — | — |

> The empty-DB 500 (Q1) and the `550` rejection (Q3) are **intended, observed behaviors documented as answers** — they are explicitly out of scope to "fix" and are not defects.

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|---------------|-------------------|-------------------|-------|
| Project Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` | Container registry pull | Reproducing the verbatim evidence requires this image (Python 3.10 + pinned legacy deps). The local sandbox runs Python 3.13 with a PEP 668 marker and **cannot** run the legacy stack. | Resolved for Blitzy (validation ran inside the image); reviewers need registry access to re-run | Reviewer / Platform |

> No repository-permission or service-credential blockers exist for the committed artifact itself. The only "access" consideration is for **optionally** re-running the scenarios.

### 1.6 Recommended Next Steps

1. **[High]** SME/technical review of the three answers (Q1/Q2/Q3) and their code-grounded rationale for accuracy and clarity.
2. **[Medium]** Review and merge the PR (`blitzy/documentation/app_2cd6ee777f8c.md`); confirm the additive single-file diff and zero source changes.
3. **[Low]** Optionally re-run one scenario inside the project Docker image to confirm the evidence reproduces on the reviewer's environment.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|------:|-------------|
| Runtime environment & stack baseline | 3.0 | [AAP §0.3.1] Stood up the documented Docker runtime (Python 3.10 venv, PostgreSQL, Redis), created a temporary `.env` from `example.env`, and created the `blitzy/documentation/` directory tree (R1). |
| Q1 — Empty-DB error investigation & write-up | 4.0 | [AAP O1] Provisioned an un-migrated DB, ran `python server.py`, drove the CSRF-aware login flow, captured the full verbatim `ProgrammingError`/`UndefinedTable` traceback, analyzed render-vs-submit, and wrote the code-grounded rationale. |
| Q2 — Required-services investigation & write-up | 5.0 | [AAP O2] Ran migrations + `init_app.py`, started webapp/email-handler/job-runner, captured verbatim readiness banners, `/health`, `/proc/net/tcp` LISTEN + SMTP `220` + 3-process proofs, and wrote the rationale. |
| Q3 — Email-rejection investigation & write-up | 5.0 | [AAP O3] Migrated but skipped `init_app.py`, injected mail to `anything@sl.local`, captured `RCPT 250`/`DATA 550` + the full rejection log block, documented the SPF-override caveat (2 corroborating tests) and Mermaid decision path, and wrote the rationale. |
| Document authoring & code-grounding | 4.5 | [AAP R6/R8 + Rule] Authored §0 methodology, startup-order overview, TL;DR, Appendices A/B; verified and embedded 149 `file:line` citations; 3-commit refinement including a full verbatim-capture overhaul. |
| Autonomous validation & QA | 3.5 | [AAP R5 build-and-run + validation] Independently re-reproduced all three scenarios from scratch, verified pinned dependency versions, confirmed citation resolution, ran `py_compile` on entry points, verified run commands, ran 2/2 SPF tests, analyzed pre-commit/whitespace, clean teardown, commit hygiene. |
| **Total Completed** | **25.0** | **Sums to Completed Hours in §1.2.** |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|------:|----------|
| SME technical review of Q1/Q2/Q3 answers + rationale (accuracy, clarity, completeness) | 2.0 | High |
| PR review & merge of `blitzy/documentation/app_2cd6ee777f8c.md` to the target branch | 0.5 | Medium |
| Optional reproducibility re-run of one scenario in the project Docker image | 0.5 | Low |
| **Total Remaining** | **3.0** | **Sums to Remaining Hours in §1.2 and §7 pie.** |

### 2.3 Hours Reconciliation

| Check | Result |
|-------|--------|
| §2.1 Completed total | 25.0 h |
| §2.2 Remaining total | 3.0 h |
| §2.1 + §2.2 = §1.2 Total | 25.0 + 3.0 = **28.0 h** ✅ |
| Completion % = 25 ÷ 28 | **89.3%** ✅ (matches §1.2, §7, §8) |

---

## 3. Test Results

All entries below originate from **Blitzy's autonomous validation logs** for this project (independent reproduction in the live Docker runtime). There is no application code change, so "tests" comprise behavioral scenario reproductions, the project's own corroborating unit tests, and static verification checks.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Behavioral Scenario Reproduction (O1, O2, O3) | Live Docker runtime harness | 3 | 3 | 0 | 100% of in-scope scenarios | Frame-for-frame exact matches vs. the documented evidence; only run-specific values (timestamps/PIDs/message-ids) differ |
| Unit — SPF override corroboration | pytest | 2 | 2 | 0 | n/a | `test_prevent_5xx_from_spf`, `test_preserve_5xx_with_valid_spf` |
| Static — entry-point compilation | `py_compile` | 4 | 4 | 0 | n/a | `server.py`, `email_handler.py`, `init_app.py`, `job_runner.py` compile cleanly |
| Static — citation resolution | source inspection | 149 | 149 | 0 | 100% | every `file:line` citation resolves to the cited symbol/line |
| Static — dependency version verification | poetry/pip inspection | 11 | 11 | 0 | n/a | flask 1.1.2, werkzeug 1.0.1, sqlalchemy 1.3.24, psycopg2-binary 2.9.3, aiosmtpd 1.4.2, alembic 1.4.3, gunicorn 20.0.4, flask-login/limiter/migrate/dotenv all match the doc |
| **Totals** | — | **169** | **169** | **0** | — | **100% pass rate** |

> **Functional tests:** 5 (3 scenario reproductions + 2 SPF unit tests), all passing. **Static checks:** 164, all passing.

---

## 4. Runtime Validation & UI Verification

**Service runtime health** (captured during autonomous validation in the Docker runtime):

- ✅ **Webapp (`server.py`)** — Operational. Boots on `127.0.0.1:7777`; `* Serving Flask app` / `* Debug mode: on` banner observed; `GET /health → "success"` (HTTP 200); `GET / → 302` redirect to `/auth/login`; login page renders HTTP 200 even against an empty DB.
- ✅ **Email handler (`email_handler.py`)** — Operational. `aiosmtpd` controller bound to `0.0.0.0:20381`; `Listen for port 20381` + `Start mail controller 0.0.0.0 20381` observed; SMTP `220 … Python SMTP 1.4.2` greeting on connect.
- ✅ **Job runner (`job_runner.py`)** — Operational. Enters its `while True` poll loop (`time.sleep(10)`); binds no port (by design).

**API / protocol integration:**

- ✅ **HTTP `/health`** returns `success` (200) — strongest webapp readiness proof.
- ✅ **SMTP `RCPT`/`DATA`** flow exercised — `RCPT 250` accept, `DATA-FINAL 550 SL E515 Email not exist` final reply (Q3).
- ✅ **Port binding** — `/proc/net/tcp` shows `LISTEN` on `7777` (hex `1E61`) and `20381` (hex `4F9D`); all three processes (plus the Werkzeug reloader child) present.

**UI verification** (this is not a UI-building task; UI observation is limited to what the investigation exercised):

- ✅ **Login page** (`templates/auth/login.html`) renders cleanly (HTTP 200) against the empty DB.
- ⚠ **Branded 500 page** (`error/500.html`) renders on the login **POST** against the empty DB — an **expected, documented** failure mode (Q1), not a defect.

---

## 5. Compliance & Quality Review

AAP deliverables and the "SWE-AtlasQnA-Repo" rule cross-mapped to quality benchmarks. Fixes applied during autonomous validation: **none required** (the document was already accurate); outstanding items are human review/merge.

| AAP Deliverable / Rule | Benchmark | Status | Progress | Notes |
|------------------------|-----------|--------|:--------:|-------|
| Single doc created, correct name (= branch) + location (`blitzy/documentation/`) | Rule: deliverable name/placement | ✅ Pass | 100% | `git diff` = `A blitzy/documentation/app_2cd6ee777f8c.md` |
| Build-and-run; capture **verbatim** evidence (not inferred) | Rule: build/run to analyze | ✅ Pass | 100% | 52 verbatim blocks; validator independently reproduced all 3 |
| Base answers on code as truth | Rule: no assumptions | ✅ Pass | 100% | 149 `file:line` citations, all resolve |
| Provide thinking / rationale per answer | Rule: rationale required | ✅ Pass | 100% | per-question rationale + Q3 decision-path diagram |
| Answer all three questions (O1/O2/O3) | AAP core objective | ✅ Pass | 100% | Q1/Q2/Q3 complete with reproduction + output + rationale |
| Do **not** modify existing files | Rule + AAP scope | ✅ Pass | 100% | 0 source files in diff; working tree clean |
| Do **not** add other code | Rule | ✅ Pass | 100% | only the one Markdown file committed |
| Temporary aids not committed (`.env`, scripts) | AAP §0.8.1 | ✅ Pass | 100% | `.env` gitignored & removed; no throwaway scripts committed |
| Markdown integrity (fences/diagram render) | Quality | ✅ Pass | 100% | 52 fences (balanced); Mermaid diagram well-formed |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Verbatim captures embed run-specific values (timestamps, PIDs, GNUPGHOME temp dirs, message-ids, session cookies) | Technical | Low | High (by design) | Doc explicitly labels these as inherently run-specific; structural output is stable across runs | Mitigated |
| Captures & line numbers depend on pinned deps + Python 3.10 (drift if deps float) | Technical | Low | Low | Versions locked in `poetry.lock`; doc records exact versions + the Docker image reference | Mitigated |
| 149 `file:line` citations could drift if the source branch advances | Technical | Low | Low | Citations are pinned to the frozen source branch `app_2cd6ee777f8c` | Mitigated |
| 13 trailing-whitespace lines inside verbatim code fences could be stripped by a linter/pre-commit, breaking fidelity | Technical | Low | Low | Confirmed genuine and required for fidelity; prose is clean; no markdown pre-commit hook enforced | Mitigated / Accepted |
| Example dev credentials in doc (`john@wick.com`/`password`; `DB_URI myuser:mypassword`) | Security | Low | Low | Documented dev defaults from `example.env`/`CONTRIBUTING.md`, not secrets; temporary `.env` gitignored & not committed | Mitigated |
| Documents `debug=True` empty-DB 500 behavior | Security | Low | Low | Informational only; no source changed, no vuln introduced (production uses gunicorn) | Accepted / Informational |
| Reproducibility depends on availability of the specific Docker image | Operational | Low | Low | Doc supplies exact commands + image ref; in-sandbox 3.13 cannot run the legacy stack (documented) | Mitigated |
| Runtime/operational footprint of the change | Operational | None | — | Doc-only change; no CI/monitoring/deploy impact | N/A |
| Cross-file / interface impact | Integration | None | — | Document imports nothing, exports nothing, referenced by no code; purely additive single-file diff | N/A |

> **Overall risk posture: LOW.** No High/Critical risks. The change is additive, documentation-only, and touches zero source files.

---

## 7. Visual Project Status

**Project hours breakdown** (Completed = `#5B39F3`, Remaining = `#FFFFFF`):

```mermaid
%%{init: {"theme":"base", "themeVariables": {"pie1":"#5B39F3","pie2":"#FFFFFF","pieStrokeColor":"#B23AF2","pieStrokeWidth":"2px","pieOuterStrokeWidth":"2px","pieTitleTextSize":"18px","pieSectionTextSize":"15px","pieLegendTextSize":"14px"}}}%%
pie showData title Project Hours Breakdown
    "Completed Work" : 25
    "Remaining Work" : 3
```

**Remaining work by priority** (hours):

| Priority | Hours | Share of remaining |
|----------|------:|-------------------:|
| 🟪 High — SME technical review | 2.0 | 66.7% |
| ⬜ Medium — PR review & merge | 0.5 | 16.7% |
| ⬜ Low — optional reproducibility re-run | 0.5 | 16.7% |
| **Total** | **3.0** | **100%** |

> **Integrity check:** the pie "Remaining Work" (3) equals §1.2 Remaining Hours (3) and the §2.2 Hours total (3). "Completed Work" (25) equals §1.2 Completed Hours (25).

---

## 8. Summary & Recommendations

**Achievements.** The project delivers a complete, code-grounded answer to all three questions about SimpleLogin's first-time local-dev startup. Every claim is backed by **verbatim runtime evidence** and reconciled to the responsible source `file:line`. The deliverable is correctly named and located, was produced without modifying any source file, and was **independently reproduced from scratch with zero discrepancies**.

**Remaining gaps & critical path to production.** The project is **89.3% complete** (25 of 28 AAP-scoped hours). The remaining 3 hours are entirely **human path-to-production** work: a technical/SME review of the answers (2.0 h), PR review & merge (0.5 h), and an optional reproducibility re-run (0.5 h). There is **no remaining engineering or rework** — no compilation errors, no failing tests, and no in-scope defects.

**Success metrics.**

| Metric | Target | Actual |
|--------|--------|--------|
| Questions answered (O1/O2/O3) | 3 / 3 | ✅ 3 / 3 |
| Source files modified | 0 | ✅ 0 |
| Scenario reproductions passing | 3 / 3 | ✅ 3 / 3 |
| SPF corroboration tests | 2 / 2 | ✅ 2 / 2 |
| `file:line` citations resolving | 100% | ✅ 149 / 149 |
| Deliverable name/location correct | yes | ✅ yes |

**Production-readiness assessment.** **Ready for human review/merge.** As a documentation-only, additive change with no runtime footprint and a uniformly LOW risk profile, the path to "production" (merge) is short. We deliberately stop short of 100% to reserve the mandatory human-in-the-loop review and merge.

---

## 9. Development Guide

> The deliverable is a Markdown document; "building/running" means (a) verifying the document and the additive diff anywhere, and (b) reproducing the three documented scenarios inside the project Docker runtime.

### 9.1 System Prerequisites

- **Recommended:** the project Docker image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_simple-login_app_1.0` (built on `python:3.10`), which supplies Python 3.10 + PostgreSQL + the Poetry-locked dependency set.
- **Manual alternative:** Python **3.10** (`pyproject.toml` declares `python = "^3.10"`), PostgreSQL **13+** (15 observed), Poetry, and Redis.
- ⚠ **Caveat:** the generic sandbox runs **Python 3.13** with a PEP 668 "externally-managed" marker; the legacy pinned stack (Flask 1.1.2, SQLAlchemy 1.3.24, etc.) **cannot** be installed/run there. Use the Docker image for reproduction.

### 9.2 Verify the Deliverable (works in any environment)

```bash
# From the repository root
test -f blitzy/documentation/app_2cd6ee777f8c.md && echo "deliverable present"
wc -l blitzy/documentation/app_2cd6ee777f8c.md           # expect 782

# Confirm the change is additive and touches no source files
git diff 2cd6ee77 --name-status                          # expect: A  blitzy/documentation/app_2cd6ee777f8c.md
git diff 2cd6ee77 --stat                                 # expect: 1 file changed, 782 insertions(+)

# Sanity-check Markdown fences are balanced (renders cleanly)
python3 - <<'PY'
n = open('blitzy/documentation/app_2cd6ee777f8c.md').read().count('```')
print('fence count =', n, '(balanced)' if n % 2 == 0 else '(UNBALANCED)')
PY
```

### 9.3 Environment Setup (for reproduction, inside the Docker runtime)

```bash
# 1) Local config — copy the shipped template (temporary, gitignored; do NOT commit)
cp example.env .env
# example.env already defines the values the app reads at import time:
#   URL=http://localhost:7777
#   DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
#   EMAIL_DOMAIN=sl.local
#   NOT_SEND_EMAIL=true

# 2) Ensure PostgreSQL is running and the database exists
export DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
```

### 9.4 Dependency Installation

```bash
poetry install        # installs the exact versions pinned in poetry.lock
# (inside the project image, deps are pre-installed in the venv at /app/venv)
```

### 9.5 Reproducing the Three Scenarios

**Q1 — Empty-database error**

```bash
# Drop to an empty (un-migrated) schema, then start the webapp
psql "$DB_URI" -c 'drop schema public cascade; create schema public;'
python server.py &                       # Flask debug server on 127.0.0.1:7777
# Drive: GET / (302 -> /auth/login) -> GET /auth/login (200) -> submit login POST
# Expected: HTTP 500 + branded error/500.html; stderr shows
#   sqlalchemy.exc.ProgrammingError: (psycopg2.errors.UndefinedTable) relation "users" does not exist
```

**Q2 — Required services & readiness**

```bash
alembic upgrade head                     # create the schema (255 revisions)
python init_app.py                       # seed public_domain (adds sl.local)
python server.py &                        # webapp        -> 127.0.0.1:7777
python email_handler.py &                 # email handler -> 0.0.0.0:20381
python job_runner.py &                    # job runner    -> no port (poll loop)

# Verify readiness
curl -s http://localhost:7777/health      # -> success
ss -ltnp | grep -E '7777|20381'           # -> LISTEN on both ports
```

**Q3 — Email rejection when `init_app.py` is skipped**

```bash
psql "$DB_URI" -c 'drop schema public cascade; create schema public;'
alembic upgrade head                      # migrate, but DO NOT run init_app.py (public_domain stays empty)
python email_handler.py &                 # email handler -> 0.0.0.0:20381
# Inject a message to anything@sl.local WITHOUT an X-Spamd-Result header
# Expected final SMTP reply to sender: 550 SL E515 Email not exist
```

### 9.6 Verification Steps

- Webapp: `curl -s http://localhost:7777/health` returns `success`.
- Ports: `ss -ltnp` (or `/proc/net/tcp`) shows `LISTEN` on `7777` and `20381`.
- Processes: `ps -ef | grep -E 'server.py|email_handler.py|job_runner.py'` shows all three (plus the Werkzeug reloader child).
- Q1/Q3 outcomes match the verbatim blocks in the deliverable (only run-specific timestamps/PIDs/message-ids differ).

### 9.7 Troubleshooting

| Symptom | Cause | Resolution |
|---------|-------|------------|
| `error: externally-managed-environment` on `pip install` | PEP 668 marker on the sandbox Python 3.13 | Use the project Docker image, or a `venv`/`--break-system-packages` (reproduction still requires Python 3.10) |
| Dependency install fails / line numbers differ | Wrong Python (3.13 instead of 3.10) | Use Python 3.10 via the project image; deps are pinned in `poetry.lock` |
| Login POST returns 500 on an empty DB | **Expected** (Q1) — `users` table absent | This is the documented behavior, not a bug; run `alembic upgrade head` to create the schema |
| Q3 returns `250 SL E216` instead of `550` | Message carried an SPF-fail `X-Spamd-Result` header | Inject without SpamAssassin/milter headers so the `5xx` is preserved |
| `/health` not reachable | Webapp not started or port in use | Confirm `python server.py` is running and `7777` is free |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `git diff 2cd6ee77 --name-status` | Confirm the additive single-file change |
| `psql "$DB_URI" -c 'drop schema public cascade; create schema public;'` | Reset to an empty/un-migrated DB (Q1, Q3) |
| `alembic upgrade head` | Apply all 255 migrations |
| `python init_app.py` | Seed `public_domain` (adds `sl.local`) — run for Q2, skipped for Q3 |
| `python server.py` | Start the webapp (`127.0.0.1:7777`, debug) |
| `python email_handler.py` | Start the SMTP handler (`0.0.0.0:20381`) |
| `python job_runner.py` | Start the background job runner (no port) |
| `curl -s http://localhost:7777/health` | Webapp readiness probe (`success`) |
| `ss -ltnp \| grep -E '7777\|20381'` | Confirm port LISTEN state |

### B. Port Reference

| Port | Service | Bind address | Source |
|------|---------|--------------|--------|
| 7777 | Webapp (Flask dev server) | `127.0.0.1` | `app.run(debug=True, port=7777)` (`server.py:588`) |
| 20381 | Email handler (aiosmtpd) | `0.0.0.0` | `Controller(MailHandler(), hostname="0.0.0.0", port=20381)` (`email_handler.py:2381-2404`) |
| 5432 | PostgreSQL | `localhost` | `DB_URI` (`example.env`) |
| — | Job runner | none | poll loop (`job_runner.py:329-347`) |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (Q&A document) |
| `server.py` | Webapp entry point (Q1, Q2) |
| `email_handler.py` | SMTP handler (Q2, Q3) |
| `init_app.py` | Initialization / `add_sl_domains()` (Q2; skipped for Q3) |
| `job_runner.py` | Background worker (Q2) |
| `app/models.py` | `User`→`users` (Q1); `SLDomain`→`public_domain` (Q3); `ModelMixin.get_by` |
| `app/email/status.py` | `E515 = "550 SL E515 Email not exist"` (Q3) |
| `app/alias_utils.py` | `try_auto_create` chain (Q3) |
| `app/auth/views/login.py` | Login flow / first `User.get_by` query (Q1) |
| `example.env`, `app/config.py` | Required env vars; `ALIAS_DOMAINS=[sl.local]` |
| `migrations/versions/*` | 255 Alembic revisions |
| `tests/test_email_handler.py` | SPF-override corroboration tests (Q3) |

### D. Technology Versions

| Component | Version | Source |
|-----------|---------|--------|
| Python | 3.10 (3.10.18 observed) | `Dockerfile` (`FROM python:3.10`), `pyproject.toml` (`python = "^3.10"`) |
| PostgreSQL | 13+ (15.13 observed) | `CONTRIBUTING.md` |
| Flask / Werkzeug | 1.1.2 / 1.0.1 | `poetry.lock` |
| SQLAlchemy / psycopg2-binary | 1.3.24 / 2.9.3 | `poetry.lock` |
| aiosmtpd | 1.4.2 | `poetry.lock` |
| alembic / flask-migrate | 1.4.3 / 2.5.3 | `poetry.lock` |
| gunicorn | 20.0.4 | `poetry.lock` |
| flask-login / flask-limiter / python-dotenv | 0.5.0 / 1.4 / 0.14.0 | `poetry.lock` |

### E. Environment Variable Reference

| Variable | Value (dev) | Used by |
|----------|-------------|---------|
| `URL` | `http://localhost:7777` | `app/config.py:79` (read at import) |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | `app/config.py:192` |
| `EMAIL_DOMAIN` | `sl.local` | `ALIAS_DOMAINS` computation (`app/config.py:157-161`) |
| `NOT_SEND_EMAIL` | `true` | local email suppression |

> These come from the shipped `example.env`; the local `.env` is **temporary and gitignored** (not committed).

### F. Developer Tools Guide

| Tool | Use |
|------|-----|
| `git diff <base> --name-status / --stat` | Confirm the change is additive and touches no source files |
| `psql` / `alembic` | Provision the three database states (empty / migrated+init / migrated-only) |
| `curl` | Webapp `/health` readiness probe |
| `ss` / `lsof` / `/proc/net/tcp` | Verify port LISTEN state for `7777` and `20381` |
| `pytest` | Run the SPF-override corroboration tests in `tests/test_email_handler.py` |
| `py_compile` | Confirm entry-point modules compile cleanly |

### G. Glossary

| Term | Definition |
|------|------------|
| **AAP** | Agent Action Plan — the primary directive defining project scope |
| **O1 / O2 / O3** | The three scoped questions (empty-DB error / required services / email rejection) |
| **`public_domain`** | Physical table backing the `SLDomain` model; seeded by `init_app.py` |
| **`E515`** | SMTP status `550 SL E515 Email not exist` returned by `handle_forward` |
| **SPF override** | Logic that rewrites a `5xx` to `250 SL E216` only when an SPF-fail `X-Spamd-Result` header is present |
| **Path-to-production** | Standard activities (here: human review + merge) to ship the AAP deliverable |
| **PA1** | The AAP-scoped, hours-based completion methodology used for §1.2 |

---

> **Cross-section integrity confirmed:** Remaining = **3 h** in §1.2, §2.2 (sum), and §7 (pie). §2.1 (25) + §2.2 (3) = **28 h** Total in §1.2. Completion **89.3%** (25 ÷ 28) consistent across §1.2, §7, and §8. All §3 tests originate from Blitzy's autonomous validation logs. Colors: Completed `#5B39F3`, Remaining `#FFFFFF`.