# Blitzy Project Guide — SimpleLogin Runtime-Behavior Onboarding Q&A

> **Branch:** `blitzy-02f43bb7-b6aa-48e3-a39d-44732e0fa33e` · **HEAD:** `da1f0f5f` · **Base:** `2cd6ee77` (`app_2cd6ee777f8c`)
> **Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md` (387 lines) · **Task type:** Documentation (isolated change, zero source modifications)

---

## 1. Executive Summary

### 1.1 Project Overview

This project delivers a single onboarding document — `blitzy/documentation/app_2cd6ee777f8c.md` — that answers four runtime-behavior questions about the **SimpleLogin** (Flask + PostgreSQL) email-aliasing codebase: the bound port and startup logs, the health-check response, the alias-creation API response and resulting database state, and the PostgreSQL-unavailable startup failure. Unlike typical code-reading docs, every answer is grounded in **actual observed runtime behavior** — the pinned stack was built, run, and exercised, capturing verbatim console output, HTTP responses, database `SELECT`s, and inline code citations. The target audience is new SimpleLogin engineers. The technical scope is deliberately narrow and **isolated**: exactly one new markdown file with **zero source-code modifications**, per the binding `SWE-AtlasQnA-Repo` rule.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'17px','pieSectionTextSize':'15px','pieLegendTextSize':'14px'}}}%%
pie showData title Project Completion — 95% Complete
    "Completed Work" : 19
    "Remaining Work" : 1
```

> **Legend (Blitzy brand colors):** Completed / AI Work = **Dark Blue `#5B39F3`** · Remaining = **White `#FFFFFF`** (violet-black `#B23AF2` outline).

| Metric | Hours |
|---|---|
| **Total Hours** | **20.0** |
| **Completed Hours (AI + Manual)** | **19.0** (AI 19.0 + Manual 0.0) |
| **Remaining Hours** | **1.0** |
| **Percent Complete** | **95.0%** |

> **Calculation:** Completion % = Completed ÷ Total × 100 = 19.0 ÷ 20.0 × 100 = **95.0%**. All 19.0 completed hours were delivered autonomously by Blitzy agents (AI); 0 manual hours were required to date.

### 1.3 Key Accomplishments

- ✅ Authored the complete 387-line onboarding document with the rigorous per-question structure: **command → verbatim output → HTTP response → database `SELECT` → rationale + citations**.
- ✅ **Built and ran the real, pinned stack** (Flask 1.1.2 / Werkzeug 1.0.1 / SQLAlchemy 1.3.24 / psycopg2 2.9.3 / Gunicorn 20.0.4 on Python 3.10 + PostgreSQL) — answers reflect observed behavior, not code reading.
- ✅ **Q1** — Confirmed both bind paths (dev `127.0.0.1:7777`, prod `0.0.0.0:7777`) and captured the *literal* startup logs, including the key finding that the textbook Werkzeug banner/access lines are suppressed by the disabled `werkzeug` logger [app/log.py:L70-71].
- ✅ **Q2** — Captured `GET /health` → body `success` (7 bytes), HTTP `200` on both servers.
- ✅ **Q3** — Exercised the alias-creation API end-to-end: discovered the non-standard `Authentication` header, demonstrated 401/401/201 across three auth cases, captured the verbatim 201 JSON (with the easily-missed extra top-level `alias` key), and recorded the persisted `alias`-table row column-by-column (including the PostgreSQL-computed `ts_vector`).
- ✅ **Q4** — Reproduced the PostgreSQL-unavailable failure: chained `sqlalchemy.exc.OperationalError`/`psycopg2.OperationalError` at import time at `connection = engine.connect()` [app/db.py:L12], with the nuanced transitive import chain that fires before the obvious `from app.db import Session`.
- ✅ Verified **~80 inline `[path:Lnn]` citations** resolve correctly at HEAD; corrected AAP imprecisions in a dedicated QA-fix commit (`da1f0f5f`).
- ✅ Kept the repository **byte-for-byte unmodified** apart from the single new file (clean `git status`; `git diff` = one added file, +387/-0); all investigation scripts were transient and removed.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|---|---|---|---|
| _None_ | No critical or release-blocking issues. The deliverable is complete, accurate, validated, and committed; the only remaining step is human SME acceptance. | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|---|---|---|---|---|
| _None_ | — | No access issues identified. The source repository, PostgreSQL provisioning, and the full runtime stack were all reachable during autonomous validation; all four behaviors were reproduced successfully. | ✅ N/A | — |

> **No access issues identified.**

### 1.6 Recommended Next Steps

1. **[High]** Conduct **SME review of the onboarding document** — read all 387 lines, confirm the four runtime answers (Q1–Q4) are convincing, and spot-check a sample of the ~80 `[path:Lnn]` citations against the source at HEAD `da1f0f5f`. *(0.75h)*
2. **[High]** **Approve and merge the PR** — verify `git diff 2cd6ee77..HEAD --name-status` shows exactly one added file and `git status` is clean, then merge to mainline. *(0.25h)*
3. **[Low]** *(Awareness, 0h — out of scope)* Note the **operational insight** the document surfaces in Q4: SimpleLogin connects to PostgreSQL **eagerly at import** [app/db.py:L12], so an unreachable database is fatal at process startup with no graceful degradation — useful input for ops runbooks and readiness-probe design.
4. **[Low]** *(Awareness, 0h — out of scope)* If running the **full** `pytest` suite, expect 3 environment-caused failures (google-re2 vs. pyre2; Apple IAP requires internet; pre-seeded test-DB ordering). These are **not** deliverable defects and are out of AAP scope — do not block on them.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

All components below trace to AAP requirements and were delivered autonomously by Blitzy agents.

| Component | Hours | Description |
|---|---|---|
| Runtime stack build-and-run (prerequisite) | 3.5 | Provision PostgreSQL (`postgres:12.1` on `127.0.0.1:5432`), set env from `example.env`, apply schema via `alembic upgrade head` (258 migration files), seed with `flask dummy-data`, and start the server on both the dev (`python server.py`) and prod (`gunicorn`) paths. |
| Q1 — Port binding & startup logs | 2.5 | Run both paths; capture verbatim dev + prod startup logs; characterize reloader double-import; establish the key finding that the `werkzeug` logger is disabled [app/log.py:L70-71] so the classic `* Running on`/Debugger-PIN/per-request lines are absent. |
| Q2 — Health-check response | 0.5 | Capture `GET /health` → body `success` (7 bytes), HTTP `200`; note dev `HTTP/1.0` (Werkzeug) vs prod `HTTP/1.1` (Gunicorn). |
| Q3 — Alias creation (API JSON + DB row) | 4.0 | Discover the non-standard `Authentication` header; establish seed prerequisites (User + ApiKey + default Mailbox); exercise three auth cases (401/401/201); capture the verbatim 201 JSON incl. the extra top-level `alias` key; record the `alias`-table row column-by-column incl. the computed `ts_vector`. |
| Q4 — PostgreSQL-unavailable failure | 2.0 | Inject an unreachable `DB_URI`; capture the chained `sqlalchemy.exc.OperationalError`/`psycopg2.OperationalError`; pinpoint the import-time break at `connection = engine.connect()` [app/db.py:L12]; trace the transitive import chain; confirm exit code 1. |
| Document authoring & structure | 4.0 | Compose the 387-line markdown: methodology header, per-question (a)–(e) format, reproducibility appendix, run-specific-value labeling, and ~80 inline `[path:Lnn]` citations with rationale. |
| QA / citation-accuracy fix cycle | 1.5 | Verify every citation against source; correct AAP imprecisions (e.g., `app/db.py:L12`, `app/log.py` line refs, serializer range, alias `id`, Alias columns) — committed as `da1f0f5f`. |
| Cleanup & repository integrity | 1.0 | Remove transient scripts/logs (outside the tree), remove the ephemeral Q3 alias row, confirm correct filename/location, and verify the source tree is byte-for-byte unmodified (clean `git status`). |
| **Total Completed** | **19.0** | Matches Section 1.2 Completed Hours. |

### 2.2 Remaining Work Detail

All remaining work is **path-to-production** (human acceptance) — there is no deployment/CI-CD/integration for a documentation artifact.

| Category | Hours | Priority |
|---|---|---|
| Documentation SME review & acceptance (read 387 lines; confirm Q1–Q4 answers; spot-check citations) | 0.75 | High |
| PR approval & merge to mainline (confirm single-file diff & clean status) | 0.25 | High |
| **Total Remaining** | **1.0** | Matches Section 1.2 Remaining Hours and Section 7 "Remaining Work". |

> **Out-of-scope awareness items (0h, not counted):** the eager import-time DB-connect operational insight (Q4) and the 3 environment-caused/out-of-scope full-suite test failures are documented for context only and carry **no** hours, because fixing them is outside the AAP and would violate the binding no-code-change rule.

### 2.3 Hours Reconciliation

| Check | Value | Status |
|---|---|---|
| Section 2.1 Completed total | 19.0 | ✓ |
| Section 2.2 Remaining total | 1.0 | ✓ |
| 2.1 + 2.2 = Total Project Hours | 19.0 + 1.0 = 20.0 | ✓ matches Section 1.2 |
| Section 7 pie "Completed Work" | 19 | ✓ matches 2.1 |
| Section 7 pie "Remaining Work" | 1 | ✓ matches 2.2 & 1.2 |
| Completion % | 19.0 ÷ 20.0 × 100 = 95.0% | ✓ consistent in 1.2, 7, 8 |

---

## 3. Test Results

All tests below originate **exclusively from Blitzy's autonomous validation logs** for this project (framework: `pytest 7.0.0` + `pytest-cov 3.0.0`). The alias-creation API suite is the deliverable-relevant subset behind Q3; the runtime-behavior verifications are the CLI/HTTP/DB captures performed during autonomous validation.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---|---|---|---|---|---|---|
| Broader regression suite | pytest 7.0.0 | 613 | 610 | 3 | N/A* | 99.5% pass. The 3 failures (+ related collection errors across ~6 items) are **environment-caused** and in AAP-**out-of-scope** functionality — see notes below. |
| ↳ *of which:* Alias-Creation API (Q3 core; subset of above) | pytest 7.0.0 | 16 | 16 | 0 | N/A* | `tests/api/test_new_random_alias.py` (6) + `tests/api/test_new_custom_alias.py` (10). **100% pass** — the endpoints behind Q3. |
| Runtime behavior verification (Q1–Q4) | Blitzy autonomous run (CLI/curl/`psql`/`/proc/net/tcp`) | 4 | 4 | 0 | N/A | Port bind (dev `127.0.0.1:7777` + prod `0.0.0.0:7777`), `GET /health`→200, alias API→201 + DB row, PG-unavailable→`OperationalError` exit 1 — all reproduced. |
| **Total (distinct; subset not double-counted)** | — | **617** | **614** | **3** | — | 613 pytest checks + 4 runtime verifications. |

> **\*Coverage %:** Not separately reported in the validation logs for these subsets, hence `N/A`. Code coverage is not the operative quality metric for a documentation-only deliverable; the project's configured gate is `fail_under = 55` (`coverage.ini`).

**Detail on the 3 out-of-scope failures (none related to Q1–Q4; none introduced by this work; none fixable under the no-code-change rule):**
- `tests/test_regex_utils.py::test_regex_match` — the image ships `google-re2` instead of the manifest's `pyre2`; RE2 rejects a negative-lookahead pattern `pyre2` would accept.
- `tests/api/test_apple.py::test_apple_process_payment` — network-bound Apple IAP call times out (no internet in the container).
- `tests/dashboard/test_custom_alias.py::test_available_suffixes_domain_order` — data-dependent ordering assertion affected by pre-seeded test-DB SLDomains.
- *(Plus collection errors in `tests/handler/test_encrypt_pgp.py`, `tests/handler/test_preserved_headers.py`, `tests/test_email_handler.py` from the same `re2`≠`pyre2` substitution.)*

---

## 4. Runtime Validation & UI Verification

All four documented behaviors were verified by **actually running the application** during autonomous validation.

**Runtime health & API integration:**
- ✅ **Operational** — Dev server binds `127.0.0.1:7777` (`app.run(debug=True, port=7777)` [server.py:L588]; verified via `/proc/net/tcp` `0100007F:1E61` LISTEN).
- ✅ **Operational** — Prod Gunicorn binds `0.0.0.0:7777` (`gunicorn wsgi:app -b 0.0.0.0:7777 -w 2` [Dockerfile:L47]; logged `Listening at: http://0.0.0.0:7777`).
- ✅ **Operational** — `GET /health` → body `success` (7 bytes), HTTP `200` (dev `HTTP/1.0`/Werkzeug, prod `HTTP/1.1`/Gunicorn).
- ✅ **Operational** — `POST /api/alias/random/new` with `Authentication: code` → **201**, JSON via `serialize_alias_info_v2` + extra `alias` key; one row written to the `alias` table (verified via `SELECT`).
- ✅ **Operational** — Auth negative cases: missing/wrong `Authentication` header → **401** `{"error":"Wrong api key"}`.
- ✅ **Operational** — PostgreSQL-unavailable: `sqlalchemy.exc.OperationalError` wrapping `psycopg2.OperationalError: ... Connection refused` at `app/db.py:L12`, process exits **1**.
- ✅ **Operational** — Startup-log fidelity: SimpleLogin prints + partial Flask banner present; classic Werkzeug `* Running on`/`* Restarting`/`* Debugger PIN`/per-request access lines confirmed **absent** (logger disabled [app/log.py:L70-71]).

**UI verification:** **Not applicable** — this is a documentation-only deliverable with no user-facing UI changes (AAP §0.5.3). No screens, components, or front-end behavior were added or modified.

---

## 5. Compliance & Quality Review

Cross-mapping of AAP deliverables and the binding `SWE-AtlasQnA-Repo` rules to their validation status.

| AAP / Rule Requirement | Benchmark | Status | Evidence |
|---|---|---|---|
| Create exactly one document, named for the source branch | `app_2cd6ee777f8c.md` | ✅ Pass | File present; name = branch `app_2cd6ee777f8c`. |
| Place document in `blitzy/documentation/` of destination repo | Correct location | ✅ Pass | `blitzy/documentation/app_2cd6ee777f8c.md`; directory created. |
| Build & run the code; observe ACTUAL runtime behavior | Live execution | ✅ Pass | Both run paths executed; Q1–Q4 reproduced. |
| Q1 answered (port + startup logs) | Observed + cited | ✅ Pass | Port 7777 both paths; literal logs captured; werkzeug-disabled finding. |
| Q2 answered (health response) | Observed + cited | ✅ Pass | `success` / `200` captured on both servers. |
| Q3 answered (API JSON + DB state) | Observed + cited | ✅ Pass | 201 JSON + `alias`-table row column-by-column; `Authentication` header. |
| Q4 answered (PG-unavailable error + location) | Observed + cited | ✅ Pass | `OperationalError` at `app/db.py:L12` at import; exit 1; import-chain traced. |
| Base answers on code as source of truth | Every claim traceable | ✅ Pass | ~80 inline `[path:Lnn]` citations. |
| Provide rationale | "(e) Rationale" per question | ✅ Pass | Each section explains *why* the behavior occurs. |
| Cite code as `[path:Lnn]` | Inline citations | ✅ Pass | ~80 citations verified at HEAD, **zero errors**. |
| Do **not** modify any existing source file | Zero edits | ✅ Pass | `git diff 2cd6ee77..HEAD` = single `A` entry, +387/-0. |
| Do **not** add any other code | Single artifact only | ✅ Pass | Only the markdown file committed; temp scripts outside tree. |
| Clean up; verify clean `git status` | Byte-for-byte unmodified | ✅ Pass | `git status` clean; transient scripts/DB row removed. |

**Fixes applied during autonomous validation:** Commit `da1f0f5f` corrected citation/factual imprecisions inherited from the AAP (e.g., `app/db.py:L12` vs. the AAP's approximate `L9-11`; precise `app/log.py` line refs; serializer range `L55-93`; the seeded alias `id`; Alias column line refs). Post-fix re-verification found **zero** remaining citation errors.

**Outstanding compliance items:** None. All binding rules satisfied.

---

## 6. Risk Assessment

Overall risk posture is **Low** — zero code changes, fully validated, repository byte-for-byte clean. No High- or Medium-severity blockers.

| Risk | Category | Severity | Probability | Mitigation | Status |
|---|---|---|---|---|---|
| Citation drift — line-number citations may stale if SimpleLogin source later changes | Technical | Low | Medium (over time) | Document is a snapshot pinned to HEAD `da1f0f5f`; re-verify citations if the base advances. | Accepted / Open |
| Run-specific values (alias local-part, `id`, timestamps, pids, GNUPGHOME paths) misread as fixed | Technical | Low | Low | Document explicitly labels them "observed-in-this-run"; Appendix enumerates which values vary vs. which are stable. | Mitigated |
| Working-directory path prefix (`/code` vs `/work`) confuses reproduction | Technical | Low | Low | Appendix discloses `WORKDIR /code` and that only the prefix changes, not behavior. | Mitigated |
| Seed credentials shown (ApiKey `code`; `john@wick.com`/`password`) | Security | Low (informational) | N/A | These are **public dummy-data fixtures** from `flask dummy-data` (documented in `CONTRIBUTING.md`), not production secrets; doc is for local-dev onboarding. | Accepted |
| New attack surface from code changes | Security | None | N/A | Zero code changes → no new vulnerabilities introduced. | N/A |
| Operational footprint of the deliverable | Operational | None | N/A | Markdown file; no runtime footprint, monitoring, or health requirements. | N/A |
| Integration dependencies of the deliverable | Integration | None | N/A | Standalone markdown; no API keys, external services, or network config. | N/A |
| Full-suite env-caused test failures (re2/pyre2, Apple IAP internet, test-DB ordering) | Integration (awareness) | Low | N/A | Out of AAP scope; not deliverable defects; not fixable under no-code-change rule. | Documented / Accepted |

**Informational finding (property of SimpleLogin, surfaced by the doc — not a risk of this work):** the eager import-time DB connection [app/db.py:L12] means an unreachable PostgreSQL is fatal at startup with no graceful degradation (Q4). Valuable input for the team's operational readiness design.

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeWidth':'2px','pieTitleTextSize':'16px','pieSectionTextSize':'14px','pieLegendTextSize':'13px'}}}%%
pie showData title Project Hours Breakdown (Total 20.0h)
    "Completed Work" : 19
    "Remaining Work" : 1
```

> **Colors:** Completed Work = **Dark Blue `#5B39F3`** · Remaining Work = **White `#FFFFFF`** (outline `#B23AF2`).

**Remaining hours by category (from Section 2.2):**

```mermaid
%%{init: {'theme':'base', 'themeVariables': {'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieTitleTextSize':'15px','pieSectionTextSize':'13px','pieLegendTextSize':'13px'}}}%%
pie showData title Remaining Work — 1.0h (Human Acceptance)
    "SME review & acceptance" : 0.75
    "PR approval & merge" : 0.25
```

> **Integrity:** the pie "Remaining Work" total (1.0h) equals Section 1.2 Remaining Hours and the Section 2.2 sum; "Completed Work" (19h) equals the Section 2.1 total.

---

## 8. Summary & Recommendations

**Achievements.** The project is **95.0% complete (19.0 of 20.0 hours)**. The sole AAP deliverable — a runtime-behavior onboarding document answering four questions about SimpleLogin — has been authored, grounded in **observed** runtime behavior across both run paths, and verified end-to-end. All four answers (port/startup logs, health check, alias-creation API + database state, and PostgreSQL-unavailable failure) are backed by verbatim output, HTTP responses, database `SELECT`s, and ~80 verified inline code citations. The repository remains **byte-for-byte unmodified** apart from the single new file, in full compliance with the binding no-code-change rule.

**Remaining gaps.** The remaining **1.0 hour** is entirely **human acceptance**: an SME review of the document and PR approval/merge. There is no deployment, CI/CD, or integration work, because the deliverable is a documentation artifact.

**Critical path to production.** (1) SME review of the four answers and a citation spot-check → (2) PR approval and merge. That is the complete path.

**Success metrics (all met):** single correctly-named file in the correct location ✅; zero source modifications ✅; all four questions answered from observed behavior ✅; ~80 citations verified ✅; deliverable-relevant tests 16/16 ✅; clean `git status` ✅.

**Production-readiness assessment.** The deliverable is **production-ready pending human acceptance**. It is accurate, complete, evidence-backed, and committed on the correct branch. No defects or blockers remain; risk is uniformly Low/informational. Recommendation: proceed directly to SME review and merge.

| Metric | Value |
|---|---|
| Completion | 95.0% (19.0 / 20.0 h) |
| Remaining | 1.0 h (human acceptance) |
| Source files modified | 0 |
| Files added | 1 (`blitzy/documentation/app_2cd6ee777f8c.md`, 387 lines) |
| Deliverable-relevant tests | 16 / 16 passing |
| Citations verified | ~80 (zero errors) |
| Blockers | None |

---

## 9. Development Guide

This guide reproduces the runtime environment behind the four documented answers. The canonical commands are verified against the repository; the full-stack build/run commands were executed during autonomous validation (in the project venv at `/app/venv`, cwd `/work`).

### 9.1 System Prerequisites

- **Python 3.10** (`pyproject.toml` pins `^3.10`; the image uses `python:3.10`).
- **PostgreSQL** (`postgres:12.1` per `README.md`) reachable at `127.0.0.1:5432`.
- **Redis** (backs Flask-Limiter rate limiting / the parallel lock).
- **Poetry** (dependency management).
- **Docker** (to run PostgreSQL) and **curl** (to exercise the API).
- **Git** + **Git LFS**.

### 9.2 Environment Setup

```bash
# From the repository root. .env is gitignored (it will NOT modify the tracked tree).
cp example.env .env
```

Ensure these variables are set (defaults from `example.env`):

```bash
URL=http://localhost:7777
EMAIL_DOMAIN=sl.local
SUPPORT_EMAIL=support@sl.local
DB_URI=postgresql://myuser:mypassword@localhost:5432/simplelogin
FLASK_SECRET=secret
EMAIL_SERVERS_WITH_PRIORITY=[(10, "email.hostname.")]
```

Start PostgreSQL via Docker:

```bash
docker run -d \
  --name sl-db \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=simplelogin \
  -p 5432:5432 \
  postgres:12.1
```

### 9.3 Dependency Installation

```bash
poetry install            # creates the venv and installs the pinned dependencies
poetry shell              # activate the venv (or: source "$(poetry env info --path)/bin/activate")
```

### 9.4 Application Startup

```bash
# Canonical local recipe (CONTRIBUTING.md:L106)
alembic upgrade head      # apply the schema (materializes the `alias` table, etc.)
flask dummy-data          # seed: john@wick.com/password, ApiKey code="code", default mailbox, ~11 aliases

# Development path → binds 127.0.0.1:7777
python3 server.py

# Production path → binds 0.0.0.0:7777 (two sync workers)
gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15
```

After startup you can log in at `http://localhost:7777` as `john@wick.com / password` (CONTRIBUTING.md:L109).

### 9.5 Verification Steps (reproduce Q1–Q4)

```bash
# Q1 — Port & startup logs: observe the console output of the startup command above.
#   Expect SimpleLogin prints (>>> URL:, >>> init logging <<<) + a PARTIAL Flask banner.
#   The classic "* Running on ..." / Debugger-PIN / per-request access lines are ABSENT
#   (werkzeug logger disabled at app/log.py:L70-71).

# Q2 — Health check
curl -i http://127.0.0.1:7777/health
#   Expect: HTTP 200, body `success` (Content-Length: 7).

# Q3 — Alias creation (note the non-standard `Authentication` header; value is `code`)
curl -i -X POST http://127.0.0.1:7777/api/alias/random/new \
     -H 'Authentication: code' \
     -H 'Content-Type: application/json' \
     -d '{"note":"created via API for Q3"}'
#   Expect: HTTP 201 + JSON (serialize_alias_info_v2 + extra top-level `alias` key).
#   Then inspect the database row (id is the autoincrement value returned above):
psql -h 127.0.0.1 -p 5432 -U myuser -d simplelogin -x -c "SELECT * FROM alias WHERE id = <id>;"

# Q4 — PostgreSQL unavailable (point DB_URI at a port with nothing listening)
DB_URI="postgresql://myuser:mypassword@127.0.0.1:5999/simplelogin" python3 server.py
#   Expect: sqlalchemy.exc.OperationalError (psycopg2.OperationalError: ... Connection refused)
#   at app/db.py:L12 `connection = engine.connect()` during import; process exits 1.
```

### 9.6 Example Usage (Q3 auth behavior — three cases)

```bash
# 1) No header → 401 {"error":"Wrong api key"}
curl -i -X POST http://127.0.0.1:7777/api/alias/random/new

# 2) Wrong key → 401 {"error":"Wrong api key"}
curl -i -X POST http://127.0.0.1:7777/api/alias/random/new -H 'Authentication: WRONGKEY'

# 3) Correct key → 201 (success)
curl -i -X POST http://127.0.0.1:7777/api/alias/random/new \
     -H 'Authentication: code' -H 'Content-Type: application/json' \
     -d '{"note":"created via API for Q3"}'
```

### 9.7 Troubleshooting

- **`401 {"error":"Wrong api key"}`** — Use the **`Authentication`** header (NOT `Authorization`) [app/api/base.py:L17]; the seeded value is `code`. The 401 is emitted at [app/api/base.py:L27].
- **`sqlalchemy.exc.OperationalError` at startup** — PostgreSQL is unreachable. The connection is opened **eagerly at import** at `connection = engine.connect()` [app/db.py:L12], so the process dies before serving. Check `DB_URI` host/port and that the container is up (`docker ps`).
- **`pip ... externally-managed-environment`** — Install via Poetry / a virtualenv, not system pip.
- **Repository-integrity check** — `git status` should be clean, and `git diff 2cd6ee77..HEAD --name-status` should show exactly one added file (`A blitzy/documentation/app_2cd6ee777f8c.md`).
- **Console paths show `/code/...`** — That is the Docker `WORKDIR /code`; running from a different directory only changes the path prefix, not behavior.

---

## 10. Appendices

### Appendix A — Command Reference

| Purpose | Command |
|---|---|
| Apply schema | `alembic upgrade head` |
| Seed data | `flask dummy-data` |
| Run dev server (`127.0.0.1:7777`) | `python3 server.py` |
| Run prod server (`0.0.0.0:7777`) | `gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Health check | `curl -i http://127.0.0.1:7777/health` |
| Create alias (API) | `curl -i -X POST http://127.0.0.1:7777/api/alias/random/new -H 'Authentication: code' -H 'Content-Type: application/json' -d '{"note":"..."}'` |
| Inspect alias row | `psql -h 127.0.0.1 -p 5432 -U myuser -d simplelogin -x -c "SELECT * FROM alias WHERE id=<id>;"` |
| Reproduce Q4 failure | `DB_URI="postgresql://myuser:mypassword@127.0.0.1:5999/simplelogin" python3 server.py` |
| Verify repo integrity | `git status` · `git diff 2cd6ee77..HEAD --name-status` |

### Appendix B — Port Reference

| Port | Bind | Source | Purpose |
|---|---|---|---|
| 7777 | `127.0.0.1:7777` | `app.run(debug=True, port=7777)` [server.py:L588] | Development server (Werkzeug) |
| 7777 | `0.0.0.0:7777` | `gunicorn ... -b 0.0.0.0:7777` [Dockerfile:L47]; `EXPOSE 7777` [Dockerfile:L44] | Production server (Gunicorn, 2 workers) |
| 5432 | `127.0.0.1:5432` | `README.md` PostgreSQL setup | PostgreSQL database |
| 5999 | (nothing listening) | Q4 reproduction | Deliberately unreachable DB endpoint |

### Appendix C — Key File Locations

| File | Role |
|---|---|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (387 lines) |
| `server.py` | App factory `create_app()` [L139]; `/health` [L213-215]; dev `app.run` [L588]; `__main__` guard [L598-599]; `dummy-data` CLI [L490] |
| `wsgi.py` | Gunicorn entry: `from server import create_app; app = create_app()` |
| `Dockerfile` | `WORKDIR /code`; `EXPOSE 7777` [L44]; Gunicorn `CMD` [L47] |
| `app/db.py` | Eager import-time `connection = engine.connect()` [L12] — the Q4 break point |
| `app/models.py` | `Alias` → table `alias` [L1469-1470]; `ModelMixin`; `create_new_random` |
| `app/log.py` | SL logger format [L12-15]; `>>> init logging <<<` [L67]; `werkzeug` logger disabled [L70-71] |
| `app/config.py` | `>>> URL:` print [L80]; required env vars |
| `app/api/base.py` | `Authentication` header [L17]; `require_api_auth`; `/api` prefix |
| `app/api/views/new_random_alias.py` | `POST /api/alias/random/new` [L21] |
| `app/api/serializer.py` | `serialize_alias_info_v2` JSON shape |
| `CONTRIBUTING.md` | Canonical run recipe [L106]; login creds [L109] |
| `example.env` | Environment-variable template |

### Appendix D — Technology Versions

| Component | Version |
|---|---|
| Python | 3.10 (runtime reported `3.10.18`) |
| Flask | 1.1.2 |
| Werkzeug | 1.0.1 |
| SQLAlchemy | 1.3.24 |
| psycopg2-binary | 2.9.3 |
| Gunicorn | 20.0.4 |
| Flask-Migrate | 2.5.3 |
| Flask-Limiter | 1.4 |
| python-dotenv | 0.14.0 |
| arrow | 0.16.0 |
| sqlalchemy-utils | 0.36.8 |
| PostgreSQL | 12.1 (per `README.md`) |
| pytest / pytest-cov | 7.0.0 / 3.0.0 |

### Appendix E — Environment Variable Reference

| Variable | Example Value | Notes |
|---|---|---|
| `URL` | `http://localhost:7777` | Required at import; printed as `>>> URL:` [app/config.py:L80] |
| `EMAIL_DOMAIN` | `sl.local` | Required at import |
| `SUPPORT_EMAIL` | `support@sl.local` | Required at import |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:5432/simplelogin` | Required; consumed by `app/db.py` |
| `FLASK_SECRET` | `secret` | Required at import |
| `EMAIL_SERVERS_WITH_PRIORITY` | `[(10, "email.hostname.")]` | Required at import |

### Appendix F — Developer Tools Guide

- **Reproduce all four answers:** follow §9.5 in order (start PostgreSQL → migrate → seed → run → curl/psql).
- **Confirm port binding without curl:** inspect `/proc/net/tcp` for `0100007F:1E61` (= `127.0.0.1:7777`) in `LISTEN` state, as done during validation.
- **Verify citations:** open the cited file at the cited line, e.g. `sed -n '12p' app/db.py`, `sed -n '213,215p' server.py`.
- **Confirm zero source drift:** `git diff 2cd6ee77..HEAD --stat` must report only the single documentation file.

### Appendix G — Glossary

| Term | Meaning |
|---|---|
| **AAP** | Agent Action Plan — the authoritative scope for this task. |
| **Eager import-time connection** | `app/db.py` opens the DB connection when the module is *imported* (not on first request), so an unreachable DB is fatal at startup (Q4). |
| **`Authentication` header** | SimpleLogin's non-standard API auth header (not `Authorization`) carrying the API key [app/api/base.py:L17]. |
| **`serialize_alias_info_v2`** | The serializer defining the alias JSON response shape [app/api/serializer.py]. |
| **`ts_vector`** | PostgreSQL-computed full-text column on `alias` (`to_tsvector('english', note)`); demonstrates server-side stemming/stopword removal. |
| **Reloader double-import** | In Flask debug mode the Werkzeug reloader re-imports the app in a child process, so SimpleLogin's startup prints appear twice (dev path). |
| **`dummy-data`** | The Flask CLI seeding command [server.py:L490] that creates the test user, API key (`code`), default mailbox, and ~11 baseline aliases. |

---

*Generated by the Blitzy Platform. Completion is measured strictly against AAP-scoped work plus path-to-production. Brand colors: Completed `#5B39F3`, Remaining `#FFFFFF`, accents `#B23AF2` / `#A8FDD9`.*