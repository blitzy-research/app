# Blitzy Project Guide

**Project:** SimpleLogin Runtime-Behavior Investigation & Documentation
**Branch:** `blitzy-6b3501ac-ee3f-445a-9307-92c03eeb308c`
**Base:** `app_2cd6ee777f8c` (`2cd6ee777f8c2d3531559588bcfb18627ffb5d2c`)
**Deliverable:** `blitzy/documentation/app_2cd6ee777f8c.md`

> **Brand color legend** — Completed / AI work: **Dark Blue `#5B39F3`** · Remaining / Not completed: **White `#FFFFFF`** · Headings/accents: **Violet-Black `#B23AF2`** · Highlight: **Mint `#A8FDD9`**.

---

## 1. Executive Summary

### 1.1 Project Overview

This project empirically investigates and documents the **actual runtime and initialization behavior** of the open-source SimpleLogin email-alias system (a monolithic Flask + PostgreSQL application). Rather than inferring behavior from source, the system was built and run inside the provided container so that observed output is the source of truth. The deliverable is a single markdown document answering five precise questions — database migration table inventory, web-server readiness timing, email-handler custom-port binding, the registration/login-before-activation flow, and dynamic free-plan alias-limit behavior — each with verbatim observed values, code citations, and rationale. The target audience is engineers needing ground-truth operational facts about SimpleLogin. No application source, configuration, or tests were modified.

### 1.2 Completion Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieTitleTextSize':'16px','pieSectionTextColor':'#B23AF2','pieOpacity':'1'}}}%%
pie showData title Completion — 94.1% Complete (32 of 34 hrs)
    "Completed Work (#5B39F3)" : 32
    "Remaining Work (#FFFFFF)" : 2
```

| Metric | Hours |
|--------|------:|
| **Total Hours** | 34.0 |
| **Completed Hours (AI + Manual)** | 32.0  (AI 32.0 + Manual 0.0) |
| **Remaining Hours** | 2.0 |
| **Percent Complete** | **94.1%** |

> Completion is computed per the AAP-scoped (PA1) hours methodology: `Completed / (Completed + Remaining) = 32.0 / 34.0 = 94.1%`.

### 1.3 Key Accomplishments

- ✅ **Runtime environment provisioned** — PostgreSQL 13 + Redis 6 containers, Python 3.10.20 venv, full 179-package SimpleLogin dependency stack (SQLAlchemy 1.3.24, Flask 1.1.2, gunicorn 20.0.4, aiosmtpd 1.4.2, alembic 1.4.3).
- ✅ **All five questions answered empirically** with verbatim observed output (table count + last table, readiness log + ms delta, port-bind log, full `curl -i` output + DB booleans, before/after `max_alias_free_plan` JSON).
- ✅ **Q1 corroborated by AST replay** of all 255 migration `upgrade()` bodies (80 create / 4 drop / 3 rename; single head `32f25cbf12f6`).
- ✅ **355-line deliverable authored** with 39 verified `file:line` citations and rationale for every answer; correctly named and placed.
- ✅ **Full regression gate green** — 639 / 639 tests pass, 0 failures; `pip check` clean; `py_compile`/`compileall` OK.
- ✅ **Strict scope compliance** — zero source/config/test files modified; ephemeral `.env` gitignored and never committed.

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| _None._ All five answers empirically reproduced; full test suite green; deliverable committed and validated. | None | — | — |

### 1.5 Access Issues

| System/Resource | Type of Access | Issue Description | Resolution Status | Owner |
|-----------------|----------------|-------------------|-------------------|-------|
| AAP-cited private Docker image (`ghcr.io/scaleapi/swe-atlas:…simple-login…`) | Container registry pull | Image not pullable in the execution environment (registry auth) | **Resolved via equivalent** — runtime rebuilt from `poetry.lock`/`pyproject.toml`; observed dependency versions recorded verbatim (gunicorn 20.0.4 vs AAP narrative 20.1.0). The `Listening at:` readiness wording is identical across gunicorn 20.x, so **no answer is affected**. | Human reviewer (advisory) |

### 1.6 Recommended Next Steps

1. **[High]** SME technical-accuracy review of the document — confirm the five answers' verbatim values against cited source and spot-check the 39 `file:line` citations (~1.5h).
2. **[Medium]** Approve the PR and merge the single deliverable into the destination default branch — no build/deploy needed (doc-only, zero source changes) (~0.5h).
3. **[Low]** _(Advisory, not counted)_ Re-derive point-in-time values (Q1 table count, Q5 default limit) if migrations or config change in future.

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|------:|-------------|
| [AAP] Runtime environment provisioning + 179-pkg dependency stack | 6.0 | Postgres 13/Redis 6 containers, Python 3.10 venv, Poetry install, ephemeral `.env` from `example.env`; verified live versions |
| [AAP-R1] Migration investigation + AST replay of 255 revisions | 4.0 | Empty-DB `alembic upgrade head`; `information_schema` table count; topological-order reconciliation; head vs last-table analysis |
| [AAP-R2] Web-server readiness + sub-second ms-delta measurement | 3.0 | Gunicorn launch on 7777; high-resolution timestamp capture; first→`Listening at:` delta across 3 runs |
| [AAP-R3] Email-handler custom-port (25025) investigation | 2.0 | `python email_handler.py -p 25025`; captured `Listen for port`/`Start mail controller` lines; proved bind via SMTP banner |
| [AAP-R4] Register/login-before-activation flow investigation | 3.0 | `curl -i` register + login; HTTP 422 capture; `psql` query of `activated`/`notification`; MX edge-case both paths |
| [AAP-R5] Alias-limit dynamic-behavior investigation | 3.0 | `/api/user_info` before/after `MAX_NB_EMAIL_FREE_PLAN=10` + restart; both-users-reflect-new-limit confirmation |
| [AAP] Deliverable authoring (355 lines, 39 citations, rationale) + review-fix | 6.0 | Structured Q&A doc; method/observed/citations/rationale per question; summary table; review finding #1 fix |
| [QA] Empirical re-verification + full 639-test suite + citation/scope audit | 5.0 | End-to-end re-derivation of all 5 answers; pytest 639/0; citation range-check; git scope verification |
| **Total Completed** | **32.0** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|------:|----------|
| [AAP] SME technical-accuracy review of the deliverable (verify 5 answers vs cited source; spot-check 39 citations) | 1.5 | High |
| [Path-to-production] PR approval + merge of the single doc artifact (no build/deploy) | 0.5 | Medium |
| **Total Remaining** | **2.0** | |

> **Reconciliation (cross-section integrity):** §2.1 Completed (32.0h) + §2.2 Remaining (2.0h) = **34.0h Total**, matching §1.2. Remaining (2.0h) is identical across §1.2, §2.2, and the §7 pie chart. Completion = 32.0 / 34.0 = **94.1%**.

---

## 3. Test Results

All tests below originate from **Blitzy's autonomous validation logs** for this project. The task is documentation-only and added **zero** new tests; the figures reflect the project's existing pytest suite executed as a **regression gate** to confirm zero source impact.

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|------------:|-------:|-------:|-----------:|-------|
| Unit + Integration (full suite) | pytest 6.x | 639 | 639 | 0 | Not reported | `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true pytest tests`; exit 0; ~164s; matches setup baseline |
| Regression impact of change | git diff | — | — | 0 | — | 1 file changed (doc only), 0 source/test files modified ⇒ no test could regress |
| **Total** | | **639** | **639** | **0** | **n/a** | Zero failures, zero regressions |

> Coverage % is honestly reported as **not measured** — coverage was not part of the autonomous validation logs, and no new code/tests were introduced that would warrant a coverage target.

---

## 4. Runtime Validation & UI Verification

Runtime health (all subsystems exercised live during autonomous validation):

- ✅ **Operational** — PostgreSQL (empty-DB reset + `alembic upgrade head`): 77 base tables created; last table `user_audit_log`.
- ✅ **Operational** — Gunicorn web server on `0.0.0.0:7777`: `Starting gunicorn 20.0.4` → `Listening at: http://0.0.0.0:7777 (<pid>)`; serving API.
- ✅ **Operational** — aiosmtpd email handler on port 25025: `Listen for port 25025` + `Start mail controller 0.0.0.0 25025`; SMTP banner `220 … Python SMTP 1.4.2` proves bind. Default 20381 confirmed without `-p`.
- ✅ **Operational** — Auth API: register → HTTP 200 `{"msg":"User needs to confirm their account"}`; login-before-activation → HTTP 422 `{"error":"Account not activated"}`; DB `activated=f, notification=t`.
- ✅ **Operational** — `/api/user_info`: `max_alias_free_plan` = 5 (default) → 10 for **both** users after `MAX_NB_EMAIL_FREE_PLAN=10` + restart.
- ⚠ **Partial / Not Applicable** — UI verification: SimpleLogin has a web UI, but **none of the five questions concern UI**; the deliverable is a backend runtime investigation, so no UI rendering was in scope.

---

## 5. Compliance & Quality Review

| Benchmark / AAP Deliverable | Status | Progress | Notes |
|------------------------------|--------|---------:|-------|
| Deliverable named `app_2cd6ee777f8c.md` | ✅ Pass | 100% | Matches source-branch naming rule |
| Placed in `blitzy/documentation/` | ✅ Pass | 100% | Destination-repo location correct |
| Exactly one persistent artifact produced | ✅ Pass | 100% | `git diff` = 1 file, 355 insertions, 0 deletions |
| Zero source/config/test files modified | ✅ Pass | 100% | Git-verified; AAP mandate honored |
| Empirical (build-and-run) methodology | ✅ Pass | 100% | All 5 answers from live output, not inference |
| Verbatim values captured | ✅ Pass | 100% | Table count, log strings, ms delta, `curl -i`, DB booleans, before/after JSON |
| Rationale provided per answer | ✅ Pass | 100% | Method/Observed/Citations/Rationale in all 5 sections |
| Code citations valid & in-range | ✅ Pass | 100% | 39 `file:line` citations verified |
| Ephemeral `.env`/configs not committed | ✅ Pass | 100% | Gitignored & untracked |
| No dependency additions/updates/removals | ✅ Pass | 100% | Stack consumed as-locked |
| Q5 baseline used non-test config | ✅ Pass | 100% | Default 5 (not `tests/test.env`'s 3); config used is stated |
| Review finding #1 resolved | ✅ Pass | 100% | Commit `eb571fb1` corrected runtime-dependency facts |

**Fixes applied during autonomous validation:** one documentation correction (review finding #1 — Environment & Setup runtime-dependency facts). Final Validator outcome: *"No corrections were required"* on re-verification. **Outstanding compliance items:** none.

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Runtime differs from AAP-cited private Docker image (equivalent rebuilt from lockfile; gunicorn 20.0.4 vs narrative 20.1.0) | Technical | Low | Certain | Exact locked versions recorded; `Listening at:` wording identical across gunicorn 20.x ⇒ no answer affected | Mitigated / Documented |
| Q2 ms-delta is hardware-dependent (~0.2ms socket bind) | Technical | Low | Medium | Document frames it qualitatively (sub-millisecond) and explains second-resolution log caveat | Mitigated / Documented |
| Q4 register HTTP status depends on `example.com` MX resolution (200 here; 400 if no MX) | Integration | Low | Medium | Both paths documented; the question's subject (login-before-activation → 422) is identical either way | Mitigated / Documented |
| Document staleness — point-in-time values (table count, default limit) could drift | Operational | Low | Low | Exact revisions/config cited, enabling re-derivation | Accepted / Documented |
| Ephemeral secrets used during investigation | Security | Low | Low | `.env` gitignored & untracked; only redacted values shown; zero secrets committed | Mitigated |
| Human-review bottleneck before merge | Operational | Low | Medium | Self-contained doc with reproducible commands minimizes review (~2h) | Open (planned) |

**Category summary:** Technical 2 · Security 1 · Operational 2 · Integration 1. **No High- or Medium-severity risks.**

---

## 7. Visual Project Status

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#FFFFFF','pieStrokeColor':'#B23AF2','pieStrokeWidth':'2px','pieOuterStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Project Hours Breakdown (Total 34h)
    "Completed Work" : 32
    "Remaining Work" : 2
```

**Remaining hours by category (from §2.2):**

```mermaid
%%{init: {'theme':'base','themeVariables':{'pie1':'#5B39F3','pie2':'#A8FDD9','pieStrokeColor':'#B23AF2','pieSectionTextColor':'#B23AF2'}}}%%
pie showData title Remaining Work by Category (Total 2h)
    "SME technical review [High]" : 1.5
    "PR approval + merge [Medium]" : 0.5
```

> **Integrity:** "Remaining Work" = 2 here equals §1.2 Remaining Hours and the §2.2 "Hours" sum (1.5 + 0.5 = 2.0). "Completed Work" = 32 equals §1.2 Completed Hours and the §2.1 sum.

---

## 8. Summary & Recommendations

**Achievements.** The project is **94.1% complete** (32.0 of 34.0 AAP-scoped hours). All five runtime questions were answered empirically by building and running SimpleLogin, with verbatim observed values, 39 verified code citations, and rationale captured in the single 355-line deliverable. The full 639-test regression suite passes with zero failures, and git confirms strictly additive scope (one documentation file, zero source modifications).

**Remaining gaps.** The remaining **2.0 hours** are entirely human, non-engineering effort: a subject-matter-expert accuracy review (1.5h) and PR approval/merge (0.5h). There is **no remaining build, code, test, or deployment work** — the task is documentation-only and the artifact is already committed and validated.

**Critical path to production.** SME review → PR approval → merge to the destination default branch. No CI/CD, container, or infrastructure change is required because no source changed.

**Success metrics.** (1) All five answers reproduce on re-run — met. (2) Zero source/config/test modifications — met. (3) Full suite green — met (639/0). (4) Deliverable correctly named and placed — met.

**Production-readiness assessment.** **Ready to merge** pending the brief human review. The maximum pre-review completion is intentionally capped below 100% to reserve for that human sign-off; no blockers, no failing tests, no unresolved issues exist.

---

## 9. Development Guide

This guide reproduces the runtime used to derive every answer. All commands were tested in the validation environment. Run from the repository root.

### 9.1 System Prerequisites

- **OS:** Linux (Ubuntu/Debian-class) or macOS.
- **Python:** 3.10.x (project requires `^3.10`; system `python3` may be newer — use the project venv). Verified: `Python 3.10.20`.
- **PostgreSQL:** 12.1 for local per README; **13** used in validation (both work). Run as a container.
- **Redis:** 6.x (container) — used for caching/rate-limit backends.
- **Poetry:** 1.8.x (verified `1.8.5`).
- **Docker:** 28.x (verified `28.5.2`, overlay2).
- **curl:** any modern (verified `8.14.1`); **psql** client (verified `17.10`).

### 9.2 Environment Setup

```bash
# 1) Start datastores (validation used host port 15432 for Postgres)
docker run -d --name sl-db   -e POSTGRES_USER=myuser -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=simplelogin -p 15432:5432 postgres:13
docker run -d --name sl-redis -p 6379:6379 redis:6

# 2) Create an EPHEMERAL .env from the template (NEVER commit this)
cp example.env .env
# Edit .env to point at the running DB and set required keys:
#   URL=http://localhost:7777
#   EMAIL_DOMAIN=sl.local
#   DB_URI=postgresql://myuser:mypassword@localhost:15432/simplelogin
#   FLASK_SECRET=secret
#   # MAX_NB_EMAIL_FREE_PLAN stays commented for the Q5 baseline (default 5)
# Config is loaded via the CONFIG env var, else ./.env  (app/config.py:65-71)
```

### 9.3 Dependency Installation

```bash
poetry env use python3.10      # ensure the 3.10 interpreter
poetry install                 # installs the locked 179-package stack
# Verify key versions:
poetry run python -c "import sqlalchemy, flask, gunicorn, aiosmtpd, alembic; \
print(sqlalchemy.__version__, flask.__version__, gunicorn.__version__, aiosmtpd.__version__, alembic.__version__)"
# Expected: 1.3.24 1.1.2 20.0.4 1.4.2 1.4.3
```

### 9.4 Build / Run & Reproduce Each Answer

```bash
# --- Q1: migrations on an EMPTY database ---
echo 'drop schema public cascade; create schema public;' \
  | psql postgresql://myuser:mypassword@localhost:15432/simplelogin
poetry run alembic upgrade head
psql postgresql://myuser:mypassword@localhost:15432/simplelogin \
  -c "select count(*) from information_schema.tables where table_schema='public';"   # => 77
poetry run alembic heads      # => 32f25cbf12f6 (head)  (index migration, not a table)

# --- Q2: web server readiness + ms delta (needs a sub-second timestamper) ---
poetry run gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15 2>&1 | ts '%H:%M:%.S'
# first line: "Starting gunicorn 20.0.4"; ready line: "Listening at: http://0.0.0.0:7777 (<pid>)"

# --- Q3: email handler on custom port 25025 ---
poetry run python email_handler.py -p 25025
# logs: "Listen for port 25025" + "Start mail controller 0.0.0.0 25025"

# --- Q4: register, then login BEFORE activation, then inspect DB ---
curl -i -X POST http://localhost:7777/api/auth/register -H 'Content-Type: application/json' \
  -d '{"email":"testuser@example.com","password":"testpass123"}'
curl -i -X POST http://localhost:7777/api/auth/login    -H 'Content-Type: application/json' \
  -d '{"email":"testuser@example.com","password":"testpass123"}'     # => HTTP 422 {"error":"Account not activated"}
psql postgresql://myuser:mypassword@localhost:15432/simplelogin \
  -c "select activated, notification from users where email='testuser@example.com';"   # => f , t

# --- Q5: dynamic alias limit (requires a RESTART after config change) ---
curl -s http://localhost:7777/api/user_info -H 'Authentication: <user1_api_key>'       # baseline => max_alias_free_plan: 5
# set MAX_NB_EMAIL_FREE_PLAN=10 in .env, then RESTART gunicorn, create user2:
curl -s http://localhost:7777/api/user_info -H 'Authentication: <user1_api_key>'       # => 10  (pre-existing user too)
curl -s http://localhost:7777/api/user_info -H 'Authentication: <user2_api_key>'       # => 10
```

### 9.5 Verification (full regression gate)

```bash
CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true poetry run pytest tests
# Expected: 639 passed, 0 failed  (~164s)
```

### 9.6 Viewing the Deliverable

```bash
sed -n '1,40p' blitzy/documentation/app_2cd6ee777f8c.md   # preview
wc -l blitzy/documentation/app_2cd6ee777f8c.md            # => 355
```

### 9.7 Troubleshooting

- **Q5 "Connection in use" on restart** — the old gunicorn master (limit=5) still holds port 7777. Terminate the exact spawned PID (`kill <pid>`); do **not** `pkill`. Then restart.
- **Q4 register returns HTTP 400 `{"error":"cannot use … as personal inbox"}`** — `example.com` has no resolvable MX in your network and `SKIP_MX_LOOKUP_ON_CHECK=False`. The login-before-activation result (HTTP 422) is unaffected; seed a non-activated user via `flask shell`/`flask dummy-data` if needed.
- **Q2 delta shows 00 ms** — gunicorn's default log timestamps are second-resolution; pipe through `ts '%H:%M:%.S'` (moreutils) or wrap with wall-clock timing to see the sub-millisecond delta.
- **Wrong Python picked up** — system `python3` may be 3.13; always use `poetry run …` or the project `.venv` (3.10.x).
- **`ts` not found** — install `moreutils` (`apt-get install -y moreutils`).

---

## 10. Appendices

### A. Command Reference

| Purpose | Command |
|---------|---------|
| Reset empty schema | `echo 'drop schema public cascade; create schema public;' \| psql $DB_URI` |
| Apply migrations | `poetry run alembic upgrade head` |
| Show migration head | `poetry run alembic heads` |
| Migration history | `poetry run alembic history` |
| Count tables | `psql $DB_URI -c "select count(*) from information_schema.tables where table_schema='public';"` |
| Start web server | `poetry run gunicorn wsgi:app -b 0.0.0.0:7777 -w 2 --timeout 15` |
| Start email handler | `poetry run python email_handler.py -p 25025` |
| Full test suite | `CONFIG=tests/test.env GITHUB_ACTIONS_TEST=true poetry run pytest tests` |
| Seed data | `poetry run flask dummy-data` |

### B. Port Reference

| Port | Service |
|------|---------|
| 7777 | SimpleLogin web app (Gunicorn / Flask) |
| 25025 | Email handler (custom, via `-p 25025`) |
| 20381 | Email handler **default** (no `-p`) |
| 15432 | PostgreSQL (host-mapped container in validation) |
| 5432 | PostgreSQL (default per `example.env`) |
| 6379 | Redis |

### C. Key File Locations

| File | Role |
|------|------|
| `blitzy/documentation/app_2cd6ee777f8c.md` | **The deliverable** (355 lines) |
| `alembic.ini` | `script_location = migrations` |
| `migrations/versions/*.py` | 255 revision files; head `32f25cbf12f6` |
| `migrations/versions/2024_101611_7d7b84779837_user_audit_log.py` | Last table-creating migration (`user_audit_log`) |
| `wsgi.py` / `server.py` | Gunicorn entry / app factory |
| `app/log.py` | Logger format (UTC, ms `asctime`), werkzeug disabled |
| `email_handler.py` | aiosmtpd Controller; `--port` default 20381 (L2398-2400); startup logs (L2386, L2403) |
| `app/api/views/auth.py` | Login/register; not-activated 422 (L75-77) |
| `app/api/views/user_info.py` | `/api/user_info`; `max_alias_free_plan` (L34, L48-50) |
| `app/models.py` | `users` table; `activated`/`notification` (L354-358); `max_alias_for_free_account()` (L858-865) |
| `app/config.py` | dotenv loader (L65-71); `MAX_NB_EMAIL_FREE_PLAN` (L121-124); `SKIP_MX_LOOKUP_ON_CHECK=False` (L600) |
| `scripts/reset_local_db.sh` / `scripts/run-test.sh` | Clean-DB recipe / Postgres+migrate+pytest recipe |

### D. Technology Versions (observed at runtime)

| Component | Version |
|-----------|---------|
| Python | 3.10.20 (venv); system 3.13.7 |
| Poetry | 1.8.5 |
| SQLAlchemy | 1.3.24 |
| Flask | 1.1.2 |
| gunicorn | 20.0.4 |
| aiosmtpd | 1.4.2 |
| alembic | 1.4.3 |
| psycopg2-binary | 2.9.x |
| cryptography | 37.0.1 |
| protobuf | 5.27.1 |
| pyre2 (`import re2`) | 0.3.6 |
| PostgreSQL | 13 (validation); 12.1 (README local) |
| Redis | 6.x |
| Docker | 28.5.2 (overlay2) |

### E. Environment Variable Reference (ephemeral `.env`)

| Variable | Example | Notes |
|----------|---------|-------|
| `URL` | `http://localhost:7777` | App base URL (`example.env` L6) |
| `EMAIL_DOMAIN` | `sl.local` | Alias domain (`example.env` L22) |
| `DB_URI` | `postgresql://myuser:mypassword@localhost:15432/simplelogin` | Connection string (`example.env` L75) |
| `FLASK_SECRET` | `secret` | Session secret (`example.env` L77) |
| `MAX_NB_EMAIL_FREE_PLAN` | _(unset → 5)_ / `10` | Free-plan alias limit; **commented by default** (L55); set to 10 + restart for Q5 "after" |
| `CONFIG` | `tests/test.env` | Selects dotenv file; else `./.env` (`app/config.py` L65-71) |
| `SKIP_MX_LOOKUP_ON_CHECK` | `False` | Hardcoded false (`app/config.py` L600) → MX gate active in Q4 |

### F. Developer Tools Guide

| Tool | Use |
|------|-----|
| `alembic heads` / `history` | Inspect the migration graph (confirms single head `32f25cbf12f6`) |
| `psql … information_schema.tables` | Authoritative live table count (Q1) |
| `ts '%H:%M:%.S'` (moreutils) | Sub-second timestamps for the Q2 readiness delta |
| `curl -i` | Capture HTTP status line + JSON body (Q4) |
| SMTP banner check | Prove the email handler bound the port (Q3) |
| `flask dummy-data` / `flask shell` | Seed a non-activated user if MX gate rejects registration (Q4 fallback) |
| `pip check` | Dependency consistency (validation: clean, 179 pkgs) |

### G. Glossary

| Term | Meaning |
|------|---------|
| **AAP** | Agent Action Plan — the governing requirements specification for this task |
| **Head (migration)** | The latest Alembic revision; here `32f25cbf12f6`, which creates only an index |
| **Readiness line** | Gunicorn's `Listening at: http://0.0.0.0:7777 (<pid>)` — signals the server accepts connections |
| **MX gate** | `email_can_be_used_as_mailbox` MX-record lookup that can reject registration when no MX exists |
| **Request-time global** | A config value (`MAX_NB_EMAIL_FREE_PLAN`) read per-request from global config, not stored per user — why both users reflect a new limit after restart |
| **Ephemeral config** | A non-committed `.env`/`CONFIG` dotenv used only to exercise the system |

---

*Generated by the Blitzy Platform · Completion measured against AAP scope (PA1) · Colors: Completed `#5B39F3`, Remaining `#FFFFFF`.*