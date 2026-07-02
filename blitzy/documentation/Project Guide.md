# Blitzy Project Guide

**Project:** SimpleLogin — First-Time Local Bring-Up: Observed Runtime Behavior (Q1–Q3)
**Branch:** `blitzy-6fc5fcd0-62bc-40bc-8fab-555a025e84e9` · **HEAD:** `df2df36c` · **Base:** `2cd6ee77`
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`

---

## 1. Executive Summary

### 1.1 Project Overview

This project is a **read-only, evidence-based runtime investigation** of the SimpleLogin Flask monolith. Its objective is to answer three concrete first-time local bring-up questions by *actually running the stack* and capturing verbatim output: (Q1) the exact Python exception raised when logging in against an empty, un-migrated database; (Q2) which Python services must run for the system to work, with startup and port-binding evidence; and (Q3) the exact SMTP status code and rejection logs produced when `init_app.py` is skipped. The target audience is engineers and self-hosters. The sole permanent output is one markdown answer document; no application behavior, source file, or dependency was changed.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2'}}}%%
pie showData title Completion — 89.3% Complete
    "Completed Work" : 25
    "Remaining Work" : 3
```

| Metric | Hours |
|--------|-------|
| **Total Hours** | 28.0 |
| **Completed Hours (AI + Manual)** | 25.0 (AI: 25.0 · Manual: 0.0) |
| **Remaining Hours** | 3.0 |
| **Percent Complete** | **89.3%** |

> Completion is computed with the PA1 AAP-scoped hours method: `25.0 / (25.0 + 3.0) × 100 = 89.3%`. All AAP deliverables are complete; the remaining 3.0h is human path-to-production review of the documentation artifact.

### 1.3 Key Accomplishments

- ✅ Built and ran the full SimpleLogin stack from scratch (Python 3.10.18, PostgreSQL, 255 Alembic migrations → 77 tables, `init_app.py` initialization).
- ✅ **Q1 answered:** captured the full verbatim traceback — `sqlalchemy.exc.ProgrammingError` wrapping `psycopg2.errors.UndefinedTable` (`relation "users" does not exist`) — and pinpointed its trigger at `app/auth/views/login.py:43` → `app/models.py:84`.
- ✅ **Q2 answered:** determined the three required long-lived services (webapp :7777, `email_handler.py` :20381, `job_runner.py` no-port) and proved each is up with startup logs + port-binding evidence.
- ✅ **Q3 answered:** reproduced the `550 SL E515 Email not exist` rejection with a full ordered log trail and precise root-cause analysis (`try_auto_create()` returns `None`).
- ✅ Authored a 691-line answer document with ~121 `file:line` citations, one-claim-one-evidence discipline, and honest disclosure of environment deviations.
- ✅ Upheld the read-only rule absolutely: `git diff` vs base shows exactly one file added, zero source modifications.
- ✅ Independently re-verified every claim and citation; corrected 2 secondary-evidence inaccuracies during final validation.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None — no blocking issues._ All three questions are answered with reproducible, verified evidence; the deliverable is committed and the working tree is clean. | None | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| _None_ | — | No access issues identified. The investigation ran entirely in a self-contained local/container environment with no external service, credential, or third-party API dependencies. | N/A | — |

**No access issues identified.**

### 1.6 Recommended Next Steps

1. **[High]** Have a SimpleLogin-familiar engineer review `blitzy/documentation/app_2cd6ee777f8c.md` for technical accuracy and completeness (Q1/Q2/Q3 answers and exact literals).
2. **[Medium]** Re-verify the ~121 `file:line` citations against the current branch HEAD (line numbers drift if the branch advances beyond `df2df36c`); optionally pin citations to a commit hash.
3. **[Medium]** Approve and merge the additive documentation PR; confirm `git diff` still shows only the single added file.
4. **[Low]** Optionally re-run the three reproductions on the documented `postgres:13` image to confirm the observed behavior is identical to the `15.13` run used during investigation.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Foundation environment bring-up | 4.0 | Python 3.10 isolated venv, `poetry install` (locked deps), PostgreSQL start, `.env` derived from `example.env` (`EMAIL_DOMAIN=sl.local`, `DB_URI`, `FLASK_SECRET`). |
| Three-state database staging | 2.5 | Staged and reset the DB into: (A) empty/un-migrated, (B) migrated + `init_app.py`, (C) migrated-only. Verified State C precondition (`public_domain`=0, `custom_domain`=0, `alias`=0). |
| Q1 investigation — empty-DB login failure | 3.0 | Ran `python server.py` vs empty DB; observed `GET /auth/login` = 200, POST = 500; captured full verbatim traceback (`ProgrammingError`/`UndefinedTable`) and identified trigger at `login.py:43`→`models.py:84`. |
| Q2 investigation — required services + readiness | 4.0 | Determined 3 required services; started each; captured verbatim startup logs; proved port binding (`/health`→`success`/200, `Listen for port 20381`, LISTEN sockets, SMTP banner, `pg_stat_activity` backend counts). |
| Q3 investigation — skipped-init SMTP rejection | 3.5 | Injected `e1@sl.local` from `hey@google.com`; captured SMTP transcript ending `550 SL E515 Email not exist`, full ordered log trail, and precise root cause (`try_auto_create()`→`None`; reply-path vs forward-path distinction; E207 bounce nuance). |
| Authoring the answer document | 5.0 | Wrote `blitzy/documentation/app_2cd6ee777f8c.md` (691 lines, ~121 `file:line` citations) with one-claim-one-evidence pairing, methodology, "Also report", and coverage checklist. |
| Validation & fidelity re-verification | 3.0 | Independently reproduced all three questions; verified every citation; corrected 2 secondary-evidence inaccuracies (non-deterministic GET size; missing deterministic `NoneType: None` job-runner line); confirmed read-only and clean tree. |
| **Total Completed** | **25.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| SME technical accuracy review & sign-off of the answer document | 1.5 | High |
| Citation line-number drift re-verification vs current HEAD (pin to commit hash if branch advances) | 1.0 | Medium |
| Stakeholder acceptance / PR merge approval | 0.5 | Medium |
| **Total Remaining** | **3.0** | |

### 2.3 Hours Methodology & Completion Calculation

- **Scope universe (PA1):** all AAP deliverables + standard path-to-production for a documentation artifact. No out-of-scope work is counted.
- **Completed hours:** `4.0 + 2.5 + 3.0 + 4.0 + 3.5 + 5.0 + 3.0 = 25.0h` (all autonomous; 0 manual).
- **Remaining hours:** `1.5 + 1.0 + 0.5 = 3.0h` (human review path-to-production).
- **Total:** `25.0 + 3.0 = 28.0h`.
- **Completion:** `25.0 / 28.0 × 100 = 89.3%` (capped ≤ 99% per honest-assessment principles).
- All 23 AAP/path-to-production requirements are classified **Completed**; none are Partially Completed or Not Started. Because there is no unresolved AAP work, remaining hours consist entirely of human review activities rather than rework.

---

## 3. Test Results

> **Integrity note:** This is a read-only documentation investigation; **no project pytest suite was in scope** (AAP §0.3). Accordingly, the "tests" below are the **staged runtime reproductions** executed by Blitzy's autonomous validation system — each is a pass/fail behavioral check captured verbatim from the validation logs. Code-coverage % is **N/A** for this task (no instrumented test suite).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|-----------|-------|
| Q1 — Empty-DB login failure | Observed runtime capture (`server.py` + HTTP client) | 3 | 3 | 0 | N/A | `GET /` → 302→`/auth/login`; `GET /auth/login` → 200; POST → 500 with `ProgrammingError`/`UndefinedTable` traceback verified. |
| Q2 — Service readiness & port binding | Observed runtime capture (curl / socket / `pg_stat_activity`) | 5 | 5 | 0 | N/A | `/health`→`success`/200; port 7777 LISTEN; `Listen for port 20381` + SMTP banner `220 ... Python SMTP 1.4.2`; job_runner processes test job; 4 webapp backends cross-service proof. |
| Q3 — SMTP rejection behavior | Observed runtime capture (SMTP inject to :20381) | 4 | 4 | 0 | N/A | RCPT `250 OK`; end-of-DATA → `550 SL E515 Email not exist`; full ordered log trail captured; root-cause precision (`try_auto_create()`→`None`) confirmed. |
| Environment bring-up | Alembic / init_app | 2 | 2 | 0 | N/A | 255 migrations applied (77 tables); `init_app.py` seeded `public_domain` (id=1, `sl.local`). |
| Citation verification | Source cross-check | 121 | 121 | 0 | N/A | Every `file:line` reference across Q1/Q2/Q3 and "Also report" verified against source; zero drift at HEAD `df2df36c`. |
| **Totals** | | **135** | **135** | **0** | **N/A** | 100% of documented behavioral claims reproduced and verified. |

---

## 4. Runtime Validation & UI Verification

**Web application (`python server.py`, port 7777)**
- ✅ **Operational** — `/health` returns body `success` with HTTP 200 (`server.py:213-215`).
- ✅ **Operational** — port 7777 in LISTEN state (`/proc/net/tcp` `0100007F:1E61`); socket `connect_ex` → `0`.
- ✅ **Operational (Q1 login page)** — `GET /auth/login` renders HTTP 200 against an empty DB (no error on render).
- ⚠ **Partial (by design, Q1 fault mode)** — POST login against an empty DB returns HTTP 500 (branded `error/500.html`, 5,744 bytes) because the first ORM query hits a missing `users` table; the generic exception handler runs (not bypassed under `debug=True`) and the full traceback is logged to console.

**SMTP email handler (`python email_handler.py`, port 20381)**
- ✅ **Operational** — startup log `Listen for port 20381` (`email_handler.py:2403`) and `Start mail controller 0.0.0.0 20381`.
- ✅ **Operational** — port 20381 LISTEN on `0.0.0.0` (`00000000:4F9D`); SMTP banner `220 ... Python SMTP 1.4.2`; full EHLO reply (`SIZE`, `8BITMIME`, `SMTPUTF8`, `HELP`); `QUIT` → `221 Bye`.
- ⚠ **Partial (by design, Q3 fault mode)** — inbound mail to `e1@sl.local` is rejected `550 SL E515 Email not exist` when `public_domain` is empty (expected behavior for skipped `init_app.py`).

**Job runner (`python job_runner.py`, no port)**
- ✅ **Operational** — polls every 10s; consumed an inserted test job, logged `Take job`, `Unknown job name test-job` + deterministic `NoneType: None`, and advanced the row (`taken=t, state=2, attempts=1`).

**Database / initialization**
- ✅ **Operational** — `alembic upgrade head` applied 255 revisions (77 tables); `init_app.py` seeded `public_domain` (id=1, `sl.local`, `use_as_reverse_alias=t`).

**UI verification:** No custom UI was built or changed (documentation-only task). The only UI surfaces observed are stock SimpleLogin pages — the login page (HTTP 200) and the branded 500 error page (Q1 fault mode) — both captured as HTTP-level evidence rather than visual regressions.

---

## 5. Compliance & Quality Review

| Governing Requirement (rule / AAP) | Benchmark | Status | Evidence / Notes |
|------------------------------------|-----------|--------|------------------|
| Deliverable at mandated path & name | `blitzy/documentation/app_2cd6ee777f8c.md` | ✅ Pass | File exists, 691 lines, committed `df2df36c`. |
| Read-only source tree (no edits/deletes) | 0 source files changed | ✅ Pass | `git diff 2cd6ee77..HEAD --name-status` → `A` (single doc); 0 `.py/.html/.toml/.env/.yml/.ini` modified. |
| Investigate by RUNNING the code first | All claims from observed output | ✅ Pass | Full stack run; verbatim traceback, startup logs, SMTP transcript captured. |
| One claim → one piece of evidence | Every behavioral statement paired | ✅ Pass | Doc uses claim→evidence pairing throughout. |
| Answer every named item + coverage pass | 8 named items addressed | ✅ Pass | `python server.py`, `init_app.py`, `SLDomain`/`public_domain`, `@sl.local`, login page, email handler, SMTP status code, rejection logs — all ticked in final checklist. |
| Be exact & grounded (`file:line`, exact literals) | Exact strings, no paraphrase | ✅ Pass | ~121 citations; reports `550 SL E515 Email not exist`, not "a rejection code". |
| Report-as-observed fidelity | No adjusting values toward expectation | ✅ Pass (fixes applied) | Corrected error-handler-not-bypassed; disclosed `rctp tos` source typo; added deterministic `NoneType: None` line; replaced non-deterministic GET body size with invariant HTTP 200. |
| Temporary artifacts removed | Repo left unchanged | ✅ Pass | SMTP inject script, logs, cookie jars, env helper removed; ports freed; no lingering processes. |
| Runtime matrix compatibility | Python 3.10, PostgreSQL ≥ 13 | ⚠ Pass w/ disclosed deviation | Ran Python 3.10.18 ✓; PostgreSQL 15.13 (satisfies ≥13, doc discloses vs documented 13). Migrations applied via `alembic upgrade head` because `flask db upgrade` errors `KeyError:'migrate'` — deviation disclosed in doc methodology. |
| Observed doc defects reported (not fixed) | Report-only | ✅ Pass | `CONTRIBUTING.md` `DB_URI` port `35432` vs `15432` mapping inconsistency noted in "Also report". |

**Fixes applied during autonomous validation:** (1) replaced a non-deterministic `GET /auth/login` body-size figure with the invariant HTTP 200 + explanation of Flask-DebugToolbar profiler variability; (2) added the deterministic trailing `NoneType: None` line to the job-runner evidence with an explanation of `LOG.e`/`logging.Logger.exception` outside an active exception context. **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Citation line-number drift — ~121 `file:line` anchors shift if the branch advances beyond `df2df36c` | Technical | Low | Medium | Re-verify at merge; pin citations to a commit hash (AAP §0.6.5 explicitly flags this) | Open |
| Environment deviations — observed on PostgreSQL 15.13 (not documented 13) and via `alembic upgrade head` (`flask db upgrade` → `KeyError:'migrate'`) | Technical | Low | Low | Deviations disclosed in doc methodology; optional re-run on `postgres:13` to confirm identical behavior | Mitigated |
| Observed-value non-determinism — `GET /auth/login` body size varied (DebugToolbar profiler) | Technical | Low | Low | Already replaced with invariant HTTP 200 + explanation | Resolved |
| No security exposure introduced — read-only; zero code/dependency/config changes; `example.env` values (e.g., `FLASK_SECRET=secret`) are the repo's own published placeholders | Security | None | N/A | No secrets committed; no attack surface added | N/A |
| Documentation staleness — point-in-time snapshot may drift if login/email/service behavior later changes | Operational | Low | Medium | Treat as a snapshot; record the commit hash it describes | Open |
| Reproducibility variance outside the provided container — PIDs/timestamps/hostnames vary run-to-run (non-substantive) | Operational | Low | Low | Doc explains the variability; substantive values are invariant | Mitigated |
| No integration risk — no external services, API keys, or network dependencies; SMTP injection used a local ephemeral client | Integration | None | N/A | Fully self-contained local reproduction | N/A |

**Overall risk posture: LOW.** No High or Critical risks. The two Open items (citation drift, doc staleness) are inherent to a point-in-time documentation artifact and are addressed by the remaining human-review tasks.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#B23AF2','pieOuterStrokeColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 28h)
    "Completed Work" : 25
    "Remaining Work" : 3
```

**Remaining work by priority (from Section 2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#B23AF2','pie3':'#A8FDD9','pieStrokeColor':'#B23AF2','pieTitleTextColor':'#B23AF2','pieSectionTextColor':'#111111'}}}%%
pie showData title Remaining 3.0h by Task
    "SME review [High]" : 1.5
    "Citation re-verify [Medium]" : 1.0
    "Acceptance/merge [Medium]" : 0.5
```

> **Integrity:** "Remaining Work" = **3.0h**, identical to Section 1.2 (Remaining Hours) and the Section 2.2 total. "Completed Work" = **25.0h**, identical to Section 2.1 total. Colors: Completed = Dark Blue `#5B39F3`, Remaining = White `#FFFFFF`.

---

## 8. Summary & Recommendations

**Achievements.** The project is **89.3% complete** (25.0h of 28.0h). All AAP deliverables were autonomously completed: the full SimpleLogin stack was brought up from scratch and all three runtime questions were answered with verbatim, reproducible evidence and precise `file:line` citations. Q1 yields the exact exception (`sqlalchemy.exc.ProgrammingError` → `psycopg2.errors.UndefinedTable`, `relation "users" does not exist`) and its trigger; Q2 yields the definitive three-service list (webapp :7777, `email_handler.py` :20381, `job_runner.py`) each with port-binding proof; Q3 yields the exact SMTP reply `550 SL E515 Email not exist` with a full ordered log trail and correct root cause.

**Remaining gaps.** The outstanding **3.0h** is entirely human path-to-production review of a documentation artifact — not code rework. There is no failing functionality: SME accuracy sign-off (1.5h), citation-drift re-verification (1.0h), and stakeholder acceptance/merge (0.5h).

**Critical path to production.** SME review → citation re-verification against current HEAD → approve & merge the additive doc. The read-only constraint is fully satisfied (single file added, zero source changes), so merge risk is minimal.

**Production readiness assessment.** The deliverable is **production-ready** for its purpose: it is committed, the working tree is clean, every claim is grounded in observed output, environment deviations are transparently disclosed, and all cross-section integrity checks pass. Recommended before merge: a human accuracy pass and a citation re-check if the branch has advanced.

| Success Metric | Target | Actual |
|----------------|--------|--------|
| All three questions answered with verbatim evidence | 3/3 | ✅ 3/3 |
| Named items covered | 8/8 | ✅ 8/8 |
| Source files modified | 0 | ✅ 0 |
| Behavioral claims reproduced | 100% | ✅ 100% |
| Citations verified | 100% | ✅ 121/121 |
| Completion | ≤ 99% | 89.3% |

---

## 9. Development Guide

This guide documents how to reproduce the investigation locally. All commands are grounded in verified `file:line` references from the repository.

### 9.1 System Prerequisites

- **Python 3.10** — required. `pyproject.toml:61` pins `python = "^3.10"`; CI uses `3.10`. The project explicitly notes Python 3.12 does **not** work (`CONTRIBUTING.md:236`).
- **PostgreSQL 13** — CI uses `image: postgres:13` (`.github/workflows/main.yml`). PostgreSQL is a hard dependency (PG-specific features); do **not** substitute SQLite.
- **Poetry** — installs the exact locked dependency versions from `poetry.lock`.
- **swaks** (or a Python `smtplib` script) — SMTP client to inject the Q3 test message.
- OS: Linux/macOS; ~1GB free disk for the venv + database.

### 9.2 Environment Setup

```bash
# From the repository root
cp example.env .env

# Key values already present in example.env (verified):
#   EMAIL_DOMAIN=sl.local                                           # example.env:22
#   DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin # example.env:75
#   FLASK_SECRET=secret                                             # example.env:77
#   LOCAL_FILE_UPLOAD=true                                          # example.env:136
# Ensure DB_URI points at your running PostgreSQL instance/database.
```

### 9.3 Dependency Installation

```bash
# Create an isolated Python 3.10 environment and install locked deps
python3.10 -m venv .venv
source .venv/bin/activate
pip install poetry
poetry install            # installs exact poetry.lock versions
```

### 9.4 Database State Staging (the three reproduction states)

```bash
# State A — EMPTY / un-migrated (for Q1): create the DB/schema but DO NOT migrate.
#   (Leaves 0 tables; login POST will fail against missing 'users' table.)

# State B — MIGRATED + INITIALIZED (for Q2):
alembic upgrade head          # applies 255 revisions -> 77 tables
python init_app.py            # seeds public_domain (SLDomain) with sl.local

# State C — MIGRATED ONLY (for Q3): reset DB, then migrate WITHOUT init.
alembic upgrade head          # DO NOT run init_app.py or `flask dummy-data`
# Verify the empty precondition:
psql "$DB_URI" -c "select count(*) from public_domain;"   # expect 0
psql "$DB_URI" -c "select count(*) from custom_domain;"   # expect 0
psql "$DB_URI" -c "select count(*) from alias;"           # expect 0
```

> **Note (verified deviation):** In some containers `flask db upgrade` raises `KeyError: 'migrate'`; use `alembic upgrade head` instead (equivalent migration path). This deviation is disclosed in the answer document's methodology.

### 9.5 Application Startup (the three required services — Q2)

```bash
# 1) Web application  (port 7777)
python server.py                 # dev entry: app.run(debug=True, port=7777)

# 2) SMTP email handler (port 20381)
python email_handler.py          # logs "Listen for port 20381"

# 3) Background job runner (no port)
python job_runner.py             # polls every 10s
```

### 9.6 Verification Steps

```bash
# Webapp readiness — expect body "success" and HTTP 200 (server.py:213-215)
curl -s http://localhost:7777/health && echo

# Webapp port bound
ss -ltn | grep ':7777' || echo "7777 not listening"

# SMTP handler port bound
ss -ltn | grep ':20381' || echo "20381 not listening"

# Login page renders (Q1: 200 even on empty DB)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:7777/auth/login   # expect 200
```

### 9.7 Example Usage — Reproduce Each Question

```bash
# --- Q1: empty-DB login failure -------------------------------------------
# With State A (empty DB), POST the login form; watch the server console for
# the full traceback (sqlalchemy.exc.ProgrammingError / psycopg2 UndefinedTable
# "relation \"users\" does not exist"), triggered at app/auth/views/login.py:43.
# The HTTP response is a branded 500 page (error handler NOT bypassed under debug=True).

# --- Q3: skipped-init SMTP rejection --------------------------------------
# With State C (migrated-only, public_domain empty), inject an inbound email:
swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381
# Expected end-of-DATA reply:  550 SL E515 Email not exist
# Expected log trail: "alias e1@sl.local not exist..." ->
#   "no custom domain for sl.local" -> "no directory separator" ->
#   "cannot be created on-the-fly, return 550" ->
#   "Finish ... return code '550 SL E515 Email not exist'"
```

### 9.8 Reading the Deliverable

```bash
# View the full answer document
sed -n '1,120p' blitzy/documentation/app_2cd6ee777f8c.md    # methodology + Q1
sed -n '273,513p' blitzy/documentation/app_2cd6ee777f8c.md  # Q2
sed -n '514,691p' blitzy/documentation/app_2cd6ee777f8c.md  # Q3 + Also report + checklist

# Confirm the read-only constraint (single added file)
git diff 2cd6ee77..HEAD --name-status   # -> A blitzy/documentation/app_2cd6ee777f8c.md
```

### 9.9 Troubleshooting

- **`flask db upgrade` → `KeyError: 'migrate'`** — use `alembic upgrade head` (documented deviation; same migration set).
- **`relation "users" does not exist` on login** — expected for Q1 (State A). Migrate the DB (`alembic upgrade head`) to resolve.
- **`550 SL E515 Email not exist` for every alias** — expected for Q3 (State C). Run `python init_app.py` (and create an alias / verified domain) so aliases can be auto-created.
- **Python 3.12 errors during `poetry install`** — unsupported; use Python 3.10 (`CONTRIBUTING.md:236`).
- **`CONTRIBUTING.md` DB port confusion** — the guide's `DB_URI` example uses port `35432` while its PostgreSQL run command maps `15432`; following it verbatim points the app at an unpublished port. Known doc defect (reported, not fixed).
- **SMTP handler returns `250 SL E207` instead of `550`** — the sender was treated as a bounce; use an ordinary sender (e.g., `hey@google.com`) to observe the 5xx rejection.

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `poetry install` | Install exact locked dependencies |
| `alembic upgrade head` | Apply all 255 migrations (→ 77 tables) |
| `python init_app.py` | Seed `public_domain` (SLDomain) with `sl.local` |
| `python server.py` | Start webapp on :7777 (`debug=True`) |
| `python email_handler.py` | Start SMTP handler on :20381 |
| `python job_runner.py` | Start background job poller (no port) |
| `curl -s http://localhost:7777/health` | Webapp readiness (`success`/200) |
| `swaks --to e1@sl.local --from hey@google.com --server 127.0.0.1:20381` | Inject Q3 test email |
| `git diff 2cd6ee77..HEAD --name-status` | Prove read-only (single added file) |

### B. Port Reference

| Port | Service | Readiness Evidence |
|------|---------|--------------------|
| 7777 | Web application (`server.py` / `gunicorn wsgi:app`) | `/health` → `success` / 200 (`server.py:213-215`) |
| 20381 | SMTP email handler (`email_handler.py`) | Log `Listen for port 20381` (`email_handler.py:2403`); banner `220 ... Python SMTP 1.4.2` |
| — | Job runner (`job_runner.py`) | No port; polls every 10s |

### C. Key File Locations

| Path | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **Deliverable** — the answer document (691 lines) |
| `server.py` | Web app factory + dev entry (`:7777`), `/health`, `/`→`auth.login` |
| `email_handler.py` | SMTP `Controller`; `handle_forward()` returns `E515` (`:555`) |
| `job_runner.py` | Background job poll loop |
| `init_app.py` | `add_sl_domains()` seeds `public_domain` |
| `app/auth/views/login.py` | Login route; `User.get_by(...)` at `:43` (Q1 trigger) |
| `app/models.py` | `Session.query(...).first()` at `:84`; `SLDomain`→`public_domain` at `:3116-3119` |
| `app/email/status.py` | `E515` (`:51`), `E207` (`:12`) reply strings |
| `app/alias_utils.py` | `try_auto_create()` auto-create logic (Q3 root cause) |
| `example.env` | Local config defaults |
| `migrations/` | 255 Alembic revisions |

### D. Technology Versions

| Component | Documented / Pinned | Observed During Investigation |
|-----------|--------------------|-------------------------------|
| Python | `^3.10` (`pyproject.toml:61`); 3.12 unsupported | 3.10.18 |
| PostgreSQL | `postgres:13` (CI) | 15.13 (satisfies ≥13; disclosed) |
| Flask | 1.1.2 (locked) | — |
| SQLAlchemy | 1.3.24 (locked) | error surface for Q1 |
| psycopg2-binary | 2.9.3 (locked) | raised `UndefinedTable` (Q1) |
| aiosmtpd | 1.4.2 (locked) | SMTP banner `Python SMTP 1.4.2` (Q2/Q3) |
| Alembic | 1.4.3 (locked) | 255 revisions applied |

### E. Environment Variable Reference

| Variable | Example (from `example.env`) | Notes |
|----------|------------------------------|-------|
| `EMAIL_DOMAIN` | `sl.local` (`:22`) | Local alias domain; part of `ALIAS_DOMAINS` |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` (`:75`) | Point at your running PostgreSQL |
| `FLASK_SECRET` | `secret` (`:77`) | Published placeholder — not a real secret |
| `LOCAL_FILE_UPLOAD` | `true` (`:136`) | Local file storage |

### F. Developer Tools Guide

- **swaks** — SMTP client used to inject the Q3 test message to port 20381 (`CONTRIBUTING.md:218`).
- **psql** — verify DB state (e.g., `select count(*) from public_domain;`).
- **ss / lsof** — confirm port binding (`:7777`, `:20381`).
- **curl** — probe `/health` and the login page.
- **git** — `git diff 2cd6ee77..HEAD --name-status` to confirm the read-only footprint.

### G. Glossary

| Term | Meaning |
|------|---------|
| **SLDomain / `public_domain`** | The ORM model `SLDomain` maps to the physical table `public_domain` (`app/models.py:3116-3119`); seeded by `init_app.py`. |
| **E515** | SMTP reply `550 SL E515 Email not exist` — alias does not exist and cannot be auto-created. |
| **E207** | SMTP reply `250 SL E207 No bounce report` — returned for bounce-type senders (contrast to E515). |
| **`try_auto_create()`** | Alias auto-creation logic; returns `None` when no custom-domain catch-all/rule or directory applies (Q3 root cause). |
| **State A/B/C** | Empty/un-migrated (Q1), migrated + initialized (Q2), migrated-only (Q3) database staging states. |
| **Forward vs reply path** | The `SLDomain`/`public_domain` lookup lives on the reply path (`is_valid_alias_address_domain`), not the forward path where the Q3 rejection occurs. |
